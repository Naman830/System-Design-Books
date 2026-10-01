# ioredis — The In-Memory Store That Makes Your Backend Fast

> **Scope:** The `ioredis` npm client for Redis in Node.js — caching, sessions, rate-limit counters, pub/sub, locks, and the connection handling that keeps it stable in production.
> **Level:** Beginner + practical.
> **New to Express?** Read [[express]] first — every runnable example here plugs into an Express route.

---

## 1. ELI5: What is ioredis?

You built a `/api/leaderboard` route. It sorts 200,000 documents in MongoDB, takes 40 ms, and returns the exact same 50 rows to every visitor. Traffic grows, the route is hit 300 times a second, and your database CPU sits at 95% running the *same query over and over* for data that changes maybe once a minute. Nothing is broken. Everything is just slow and expensive.

Think of a pharmacy. Behind the counter is a small shelf holding the fifty medicines people ask for all day — the pharmacist turns around, grabs one, done in a second. Everything else lives in the stockroom at the back: enormous, complete, authoritative, and a two-minute walk each way. The front shelf is not a replacement for the stockroom. It is small, it holds copies, and if it burned down you could restock it from the back. It exists for one reason: most requests are for the same few things, so keep those close.

**Redis** is the front shelf — an in-memory key-value store that answers in microseconds. **ioredis** is the phone line your Node app uses to talk to it. Your database ([[mongoose]] / MongoDB, Postgres, whatever) stays the stockroom: the source of truth.

> **Full name:** `ioredis` — a Redis client for Node.js, maintained under the official `redis` GitHub org
> **Type:** npm package (runtime dependency), a TCP client speaking the Redis protocol
> **Core promise:** A promise-based, auto-reconnecting Redis client supporting every Redis command, Cluster and Sentinel — and the one the rest of the Node ecosystem ([[bullmq]], the [[socket_io]] adapter) expects you to use.

```mermaid
%%{init: {'themeVariables': { 'primaryTextColor': '#000000', 'textColor': '#000000', 'lineColor': '#000000', 'primaryBorderColor': '#000000'}}}%%
flowchart LR
    App["Node app<br/>GET /users/42"] -->|"redis.get"| R{"Key in Redis?"}
    R -->|"hit"| Fast["Return cached JSON<br/>about 0.3 ms"]
    R -->|"miss"| DB["Query MongoDB<br/>about 30 ms"]
    DB --> Save["redis.set<br/>with TTL"]
    Save --> Fast

    style App fill:#e0f0ff,stroke:#000000,color:#000000
    style R fill:#fff2cc,stroke:#000000,color:#000000
    style Fast fill:#e0ffe0,stroke:#000000,color:#000000
    style DB fill:#ffe0e0,stroke:#000000,color:#000000
    style Save fill:#ffffff,stroke:#000000,color:#000000
```

---

## 2. Why Does ioredis Exist? (The Problem It Solves)

Here is life without Redis, and the dead end every beginner walks into:

```js
const cache = new Map(); // "I'll just cache it in a JavaScript variable" — and logins too, why not

app.get("/api/leaderboard", async (req, res) => {
  if (cache.has("top")) return res.json(cache.get("top")); // never expires — serves stale rows forever
  const top = await Score.find().sort({ points: -1 }).limit(50).lean(); // 40 ms of DB work, on every miss
  cache.set("top", top); // grows without bound: a Map has no TTL and no memory limit
  res.json(top);
});
```

A `Map` is a cache only *one* Node process can see. Run two instances behind a load balancer — or PM2 cluster mode, see [[pm2]] — and you have two disagreeing caches and two disagreeing session stores. Deploy, and both are wiped. Nothing expires, so memory climbs until the process is killed.

| Without Redis | With Redis + ioredis |
|---|---|
| Cache lives in one process — instance A's cache is invisible to instance B | One shared store every instance reads and writes |
| Cache dies on every restart or deploy | Survives app restarts, and optionally Redis restarts too via RDB/AOF |
| No expiry — you hand-roll `setTimeout` cleanup and leak memory | `EX`/`TTL` is built in; keys delete themselves |
| Sessions in memory: the user is logged out whenever the load balancer picks the other server | Sessions in Redis: every instance sees the same login |
| Rate-limit counters per process — 3 instances means 3x your intended limit | One atomic `INCR` counter shared by the whole fleet |
| Background jobs need a DB table and a polling loop you wrote yourself | [[bullmq]] uses Redis as a real queue with retries and delays |
| A WebSocket broadcast only reaches clients on *that* server | Pub/sub fans the message out to every instance |

**Rule of thumb:** the moment your app runs as more than one process, anything you were keeping in a module-level variable belongs in Redis.

---

## 3. Installing & Basic Usage

```bash
npm install ioredis
docker run -d --name redis -p 6379:6379 redis:8-alpine   # redis:7-alpine works identically
```

The smallest possible working example:

```js
import { Redis } from "ioredis";

// One client for the whole process. Constructing it already opens a TCP connection.
const redis = new Redis(process.env.REDIS_URL ?? "redis://127.0.0.1:6379");
// ALWAYS attach this. An unhandled "error" event on an EventEmitter crashes Node.
redis.on("error", (err) => console.error("[redis]", err.message));

await redis.set("greeting", "hello", "EX", 60);  // "EX" 60 = self-destruct after 60 seconds
const value = await redis.get("greeting");       // -> "hello" (always a string, or null on a miss)
console.log(value, await redis.ttl("greeting")); // -> hello 60
```

Every Redis command is a lowercase method, and every argument after the key is passed exactly as you would type it in `redis-cli`: `SET key val EX 60` becomes `redis.set("key", "val", "EX", 60)`. That is the entire API surface.

### CommonJS version

```js
const { Redis } = require("ioredis"); // older code used: const Redis = require("ioredis")

const redis = new Redis(process.env.REDIS_URL);
redis.on("error", (err) => console.error("[redis]", err.message));
module.exports = redis;
```

### Express example

```js
import express from "express";
import { redis } from "./redis.js";          // the shared singleton client
import { Score } from "./models/Score.js";   // your Mongoose model

const app = express();

app.get("/api/leaderboard", async (req, res) => {
  const cached = await redis.get("leaderboard:top50");
  // Redis hands back a string — parse it into the shape the client expects.
  if (cached !== null) return res.json({ source: "cache", data: JSON.parse(cached) });

  const top = await Score.find().sort({ points: -1 }).limit(50).lean(); // .lean() = plain objects
  await redis.set("leaderboard:top50", JSON.stringify(top), "EX", 60);  // worst staleness: 60s
  res.json({ source: "db", data: top });
});

app.listen(3000);
```

That's the entire mental model — **read from Redis first, fall back to the database, write what you learned back with an expiry**. Everything below is a refinement of those three lines.

---

## 4. What Redis Actually Is (And What It Isn't)

Redis keeps its whole dataset **in RAM**. No disk seek, no B-tree walk, no query planner — a `GET` is a hash-table lookup in a process already holding your data, which costs microseconds, so the 0.2–1 ms your app measures is almost entirely network round trip. Redis also executes commands on a **single thread**, one at a time, and that has two consequences you must internalize:

- Every individual command is **atomic** for free. Two instances running `INCR` on the same key can never lose an increment — no locks, no transactions needed.
- One slow command blocks *everybody*. `KEYS *` over a million keys, or an `LRANGE` over a huge list, freezes the entire server for its full duration.

Because that data lives in RAM, it is both finite and losable:

- **Memory is capped.** Set `maxmemory` plus an eviction policy or Redis grows until the machine OOM-kills it. For a cache you want `maxmemory-policy allkeys-lru` — "when full, discard whatever was used least recently." The *default* policy is `noeviction`, which instead rejects writes with an OOM error: correct for a database, disastrous for a cache.
- **Persistence is a safety net, not a guarantee.** **RDB** takes periodic point-in-time snapshots — compact and fast, but a crash loses everything since the last one. **AOF** appends every write to a log and replays it on boot; at the default `appendfsync everysec` you lose at most about a second. You can run both.
- **Therefore:** never let Redis hold the *only* copy of anything you would be sad to lose. A lost cache entry is a cache miss and a lost session is one re-login — both cheap. Your users' orders belong in [[mongoose]].

---

## 5. The Cache-Aside Pattern

The single most important pattern in this file. "Cache-aside" (also called lazy loading) means the **application** owns the caching logic: it asks the cache, and on a miss it fills the cache itself. Redis never talks to your database.

```mermaid
%%{init: {'themeVariables': { 'primaryTextColor': '#000000', 'textColor': '#000000', 'lineColor': '#000000', 'primaryBorderColor': '#000000'}}}%%
flowchart TD
    Read["GET /users/42"] --> C{"redis.get<br/>user:42:profile"}
    C -->|"string"| Hit["JSON.parse<br/>and respond"]
    C -->|"null"| Miss["User.findById<br/>in MongoDB"]
    Miss --> Fill["redis.set key json EX 300"]
    Fill --> Hit
    Write["PATCH /users/42"] --> Mongo["Update MongoDB first"]
    Mongo --> Del["redis.del user:42:profile"]
    Del --> Note["Next read misses<br/>and refills from DB"]

    style Read fill:#e0f0ff,stroke:#000000,color:#000000
    style Write fill:#e0f0ff,stroke:#000000,color:#000000
    style C fill:#fff2cc,stroke:#000000,color:#000000
    style Hit fill:#e0ffe0,stroke:#000000,color:#000000
    style Miss fill:#ffe0e0,stroke:#000000,color:#000000
    style Fill fill:#ffffff,stroke:#000000,color:#000000
    style Mongo fill:#ffffff,stroke:#000000,color:#000000
    style Del fill:#fff2cc,stroke:#000000,color:#000000
    style Note fill:#ffffff,stroke:#000000,color:#000000
```

```js
import { redis } from "./redis.js";
import { User } from "./models/User.js";

export async function getUser(id) {
  const key = `user:${id}:profile`;
  const cached = await redis.get(key);
  if (cached !== null) return JSON.parse(cached); // compare to null: "" and "0" are valid but falsy
  const user = await User.findById(id).lean(); // .lean() drops Mongoose wrappers — JSON-safe
  // Negative caching: a missing id is cached for 30s too, so a bot hammering /users/99999999
  // cannot turn every request into a database query.
  await redis.set(key, JSON.stringify(user ?? null), "EX", user ? 300 : 30);
  return user ?? null;
}

export async function updateUser(id, patch) {
  // Database first — it is the source of truth; if this throws, nothing else happened.
  const user = await User.findByIdAndUpdate(id, patch, { new: true }).lean();
  await redis.del(`user:${id}:profile`); // DELETE the key — never write the new value into it
  return user;
}
```

Why delete instead of overwriting with the fresh object? Because two concurrent updates can reach Redis in the *opposite* order they reached MongoDB, leaving the cache permanently disagreeing with the database. A delete has no such race — whoever reads next re-derives the truth from the DB. **Delete is idempotent; a stale write is forever.** Order matters too: update the DB, *then* delete. Delete first and another request can refill the cache with the old value before your write lands.

### Choosing a TTL, and naming keys

The TTL is your honest answer to "how stale is acceptable?" — and your safety net for every invalidation bug you have not found yet. A profile you also delete on write can sit at 5–15 minutes, since the TTL is only insurance. A leaderboard nobody can date-check is fine at 30–60 seconds, and that alone collapses thousands of heavy queries into one. Anything you cannot invalidate reliably gets a deliberately short TTL, because a short TTL turns a correctness bug into a brief cosmetic one.

**Rule of thumb:** every cache key gets a TTL. A key without one is a memory leak with extra steps.

Redis has no tables, so the key name *is* your schema. Everyone uses colon-separated segments running general to specific — `user:42:profile`, `post:1337:comments`, `sess:9f3a...`, `rl:203.0.113.9:/login` — which is `entity : id : aspect` plus a namespace for each job (`sess:`, `rl:`). One extra trick: prefix cached objects with a schema version, as in `v2:user:42:profile`. When you later change the *shape* of what you cache, don't hunt down the old keys — bump the prefix and let the whole previous generation expire on its own.

### The stampede problem

Your leaderboard key expires at exactly 12:00:00. Three hundred concurrent requests call `redis.get`, all receive `null`, and all run that 40 ms query at once. Your cache just amplified load instead of reducing it. That is a **cache stampede**, and it lands at the worst possible moment, when you are busiest. Fix one is TTL jitter — `"EX", 300 + Math.floor(Math.random() * 60)` — so a whole family of keys never expires in the same instant. Fix two is a recompute lock, letting exactly one caller rebuild:

```js
import { redis } from "./redis.js";
import { Score } from "./models/Score.js";

export async function getLeaderboard() {
  const cached = await redis.get("leaderboard:top50");
  if (cached !== null) return JSON.parse(cached);
  // SET ... NX succeeds for exactly one caller across the entire fleet.
  const gotLock = await redis.set("lock:leaderboard", "1", "PX", 5000, "NX");
  if (!gotLock) {
    await new Promise((resolve) => setTimeout(resolve, 100)); // someone else is rebuilding — wait
    const retry = await redis.get("leaderboard:top50");
    if (retry !== null) return JSON.parse(retry);
  }
  try {
    const top = await Score.find().sort({ points: -1 }).limit(50).lean();
    await redis.set("leaderboard:top50", JSON.stringify(top), "EX", 60);
    return top;
  } finally {
    if (gotLock) await redis.del("lock:leaderboard"); // only ever release a lock you own
  }
}
```

Fix three is **refresh-ahead**: a repeatable [[bullmq]] job recomputes the hot keys on a schedule, so the cache is never empty and no user request ever pays for the rebuild. That is the right answer for a handful of genuinely expensive, genuinely hot keys.

---

## 6. Data Types — And What Each One Is Actually For

Redis is not "a big JSON object." Each value type exists because it makes one specific operation cheap.

| Type | Commands | Use it for | Why not just a String? |
|---|---|---|---|
| **String** | `SET` `GET` `INCR` | Cached JSON, counters, flags, locks | Nothing — this is the default and most of your usage |
| **Hash** | `HSET` `HGET` `HGETALL` | Objects where you update one field | Bumping `lastSeen` doesn't mean reading and rewriting the whole blob |
| **List** | `LPUSH` `RPOP` `BRPOP` | Simple FIFO queues, capped activity feeds | O(1) push and pop at both ends, and `BRPOP` *blocks* until work arrives instead of polling |
| **Set** | `SADD` `SISMEMBER` `SCARD` | Unique membership: "has this user voted?", online-user ids | O(1) membership test with automatic de-duplication |
| **Sorted Set** | `ZADD` `ZRANGE` `ZREMRANGEBYSCORE` | Leaderboards, sliding-window rate limits, time-ordered indexes | Every member carries a score and stays sorted — range queries are O(log N) |
| **Stream** | `XADD` `XREADGROUP` | Append-only event logs with consumer groups and replay | The durable log type reliable event pipelines are built on |

```js
await redis.hset("user:42", { name: "Naman", plan: "pro" }); // Hash — object form of HSET
await redis.hset("user:42", "lastSeen", Date.now());         // touches one field, no read-modify-write
const profile = await redis.hgetall("user:42");              // -> { name: "Naman", plan: "pro", ... }
// Careful: HGETALL on a missing key returns {} — not null. Check Object.keys(profile).length.

await redis.sadd("poll:7:voters", "user:42");                            // Set — membership
const voted = (await redis.sismember("poll:7:voters", "user:42")) === 1; // sismember returns 1 or 0

await redis.zadd("leaderboard", 1500, "user:42");                          // Sorted Set — sorted for free
const top10 = await redis.zrange("leaderboard", 0, 9, "REV", "WITHSCORES"); // ZREVRANGE is deprecated
```

### The atomic counter — why rate limiting lives in Redis

Across every type, a handful of commands do most of the work: `incr`/`incrby` for counters, `expire` to attach a TTL to a key that already exists, `ttl` to read one back (`-1` means no expiry was set, `-2` means the key is gone), `set(key, val, "NX")` to write only if absent, and a variadic `del`. `INCR` returning the new value **atomically** is the whole reason Redis is the default rate-limit backend: two instances cannot both read `9`, both write `10`, and lose a request.

```js
import { redis } from "./redis.js";

export async function hitLimit(ip, windowSeconds = 60, max = 100) {
  const key = `rl:${ip}`;
  // MULTI/EXEC runs both commands back-to-back with nothing interleaved. "NX" on EXPIRE (Redis 7+)
  // means "only if there is no TTL yet", so the window starts on the FIRST request of the window.
  const results = await redis.multi().incr(key).expire(key, windowSeconds, "NX").exec();
  const count = results[0][1]; // ioredis returns [[err, value], [err, value]]
  return { count, limited: count > max };
}
```

Forgetting that `NX` is the classic bug: refresh the TTL on every request and a continuously busy client keeps resetting its own window, so it is never actually limited. On Redis 6 and older, set the TTL only when `INCR` returned `1`. For a true **sliding window**, keep request timestamps in a sorted set — `ZREMRANGEBYSCORE` to drop entries older than the window, `ZADD` the current one, `ZCARD` to count what remains — exact to the millisecond, at the cost of one key per client. In real apps you rarely hand-roll any of this; you point [[express_rate_limit]] at Redis with the `rate-limit-redis` store — `new RedisStore({ sendCommand: (...args) => redis.call(...args) })` — so one limit is shared fleet-wide instead of being multiplied by your instance count.

### SET NX PX as a simple distributed lock

```js
import { randomUUID } from "node:crypto";
import { redis } from "./redis.js";
import { sendNightlyEmails } from "./mailer.js";

const token = randomUUID(); // proves *we* own this lock, not some later holder
if (await redis.set("lock:nightly-email", token, "PX", 30_000, "NX")) {
  try {
    await sendNightlyEmails();
  } finally {
    // Compare-and-delete in Lua, so the check and the delete are one atomic step. Without it a slow
    // task whose lock already expired would delete the NEXT owner's lock.
    await redis.eval(
      "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end",
      1, "lock:nightly-email", token,
    );
  }
}
```

> ⚠️ A Redis lock is **best-effort**, not a correctness guarantee. If the TTL expires while your task is still running — a long GC pause, a slow query, a network stall — two workers run at once and neither knows. On a primary/replica failover an unreplicated lock can be handed out twice. Use it to *avoid duplicate work*: one instance sends the nightly emails, one rebuilds the cache. Never use it as the only thing standing between two workers and a double refund; that guarantee belongs in your database, as a unique index or a transaction.

---

## 7. Sessions and Pub/Sub — Surviving More Than One Server

One process and one `Map` of sessions is fine on your laptop. Now you deploy two instances. The login POST goes to server A, which stores the session in *its* memory. The next request lands on server B, which has never heard of that session id, and the user is bounced to the login page. Refresh, it works. Refresh, it doesn't. This is the most confusing bug in a beginner's first real deployment, and the fix is to keep sessions in the one store both instances share.

```js
// npm install express-session connect-redis
import session from "express-session";
import { RedisStore } from "connect-redis"; // connect-redis v8+; v7 used a default export
import { app } from "./app.js";
import { redis } from "./redis.js";

app.use(session({
  store: new RedisStore({ client: redis, prefix: "sess:" }), // reuses your one shared client
  secret: process.env.SESSION_SECRET, // signs the cookie so a session id cannot be forged
  resave: false,                      // don't rewrite the session on every request
  saveUninitialized: false,           // don't create a Redis key for anonymous visitors
  // httpOnly keeps page JavaScript out; connect-redis derives the Redis key TTL from maxAge.
  cookie: { httpOnly: true, secure: process.env.NODE_ENV === "production", sameSite: "lax", maxAge: 1000 * 60 * 60 * 24 * 7 },
}));
```

Sessions now survive deploys, expire on their own, and — a genuine advantage over stateless [[jsonwebtoken]] tokens — can be **revoked** instantly. Logging a user out everywhere is `redis.del("sess:" + id)`.

### Pub/sub: broadcasting between instances

Any client can `PUBLISH` to a channel, and every client currently `SUBSCRIBE`d to it receives the message. That is exactly what you need when instance A handles an event that must reach a user connected to instance B.

```js
import { redis } from "./redis.js";

const sub = redis.duplicate(); // a SEPARATE connection, cloned from your config
sub.on("error", (err) => console.error("[redis:sub]", err.message));
await sub.subscribe("user-events");
sub.on("message", (channel, message) => {
  const event = JSON.parse(message); // messages are always strings — serialize yourself
  console.log("received on", channel, event);
});

// Publishing uses the NORMAL client, never the subscriber.
await redis.publish("user-events", JSON.stringify({ type: "profile-updated", userId: 42 }));
```

> ⚠️ **A connection in subscriber mode cannot run normal commands.** After `subscribe()`, that client rejects `get`, `set` and `incr` — everything except subscribe-family commands. This is a Redis protocol rule, not an ioredis quirk, so you always need **two clients**: one for commands, one from `duplicate()` for subscribing. The error reads "Connection in subscriber mode, only subscriber commands may be used" — now you know exactly what it means.

This is the precise mechanism behind the [[socket_io]] Redis adapter. No magic — two ioredis clients and a channel:

```js
import { createAdapter } from "@socket.io/redis-adapter";
import { io } from "./io.js";
import { redis } from "./redis.js";

io.adapter(createAdapter(redis, redis.duplicate())); // pub client, sub client
// Now io.emit() on ANY instance reaches sockets connected to EVERY instance.
```

**Pub/sub is fire-and-forget**: no persistence, no acknowledgement, no replay. A subscriber that is disconnected when a message is published never sees it. Perfect for "broadcast this live update"; wrong for "process this payment" — that needs a durable queue, which is [[bullmq]].

---

## 8. Connection Handling in ioredis

More production Redis incidents come from connection handling than from anything you do with the data.

```js
// redis.js — imported everywhere, constructed exactly once
import { Redis } from "ioredis";

export const redis = new Redis(process.env.REDIS_URL, {
  maxRetriesPerRequest: 3,
  retryStrategy: (times) => Math.min(times * 200, 5000), // backoff, capped at 5s
});

redis.on("error", (err) => console.error("[redis] error:", err.message));
redis.on("ready", () => console.log("[redis] ready"));
```

Node's module cache guarantees this file runs once, so every importer shares one TCP connection. `new Redis()` inside a request handler opens a fresh connection *per request* — you will exhaust `maxclients` under load and see `ERR max number of clients reached`. Redis handles thousands of commands per second on a single connection, so you do not need a pool.

| Option | What it does | When you need it |
|---|---|---|
| `lazyConnect: true` | Don't open the socket until the first command | Tests, CLI scripts and serverless — so importing the module doesn't hang a process that never uses Redis |
| `retryStrategy: (times) => ms` | Delay before reconnect attempt N; return `null` to give up for good | Always — capping the backoff keeps your logs sane during an outage |
| `maxRetriesPerRequest` | How many times one command retries before rejecting (default 20) | Lower it to 3 so a request fails fast instead of hanging an HTTP handler. **Must be `null` for [[bullmq]]** |
| `enableOfflineQueue` | Buffer commands issued while disconnected, flush on reconnect (default `true`) | Set `false` for a cache client, so an outage fails instantly and you fall through to the DB |
| `keyPrefix` | Silently prepends a string to every key | Sharing one Redis between staging and prod, or between two apps |
| `tls: {}` | Enable TLS | Managed Redis — though a `rediss://` URL turns this on for you |

`Redis` is an `EventEmitter`, and Node rethrows an `"error"` event with **no listener** as an uncaught exception, which kills your process. Redis *will* disconnect at some point — a deploy, a failover, a network blip — so a server missing that one line is a server that will crash. ioredis reconnects by itself; the listener exists to log, not to fix. Add it to every client, including every `duplicate()`. On the way out, `await redis.quit()` inside your `SIGTERM` handler finishes in-flight commands before closing (`disconnect()` is the rude version).

### Pipelining and MULTI — killing round trips

Fifty sequential `await redis.get(...)` calls means fifty network round trips. At 0.5 ms each that is 25 ms of pure waiting. A **pipeline** sends all fifty in one packet:

```js
// Pipeline: batching only — other clients' commands may interleave.
const userIds = [42, 43, 44];
const pipeline = redis.pipeline();
for (const id of userIds) pipeline.get(`user:${id}:profile`);
const results = await pipeline.exec(); // [[err, value], [err, value], ...]
const users = results.map(([err, value]) => (err || value === null ? null : JSON.parse(value)));

// MULTI: batching PLUS atomicity — nothing from any other client runs in between.
await redis.multi().incr("stats:signups").sadd("signups:today", "user:42").expire("signups:today", 86_400).exec();
```

**Rule of thumb:** use `pipeline()` when you only want fewer round trips, and `multi()` when the commands must not be interleaved with anyone else's. A Redis transaction has **no rollback** — if one command fails at runtime the others still ran. It means "run these together", not "all or nothing." For genuine all-or-nothing logic write a small Lua script and `eval` it; Lua runs atomically on that single thread.

```js
// ❌ KEYS blocks the entire server while it walks every key
const keys = await redis.keys("user:*");

// ✅ SCAN in cursor-sized chunks — non-blocking, safe on a live server
for await (const batch of redis.scanStream({ match: "user:*:profile", count: 100 })) {
  if (batch.length) await redis.del(...batch);
}
```

---

## 9. TypeScript Version

ioredis ships its own type definitions — do **not** install `@types/ioredis`, a deprecated v4 stub that will fight the real types.

```ts
// redis.ts
import { Redis, type RedisOptions } from "ioredis";

const options: RedisOptions = {
  maxRetriesPerRequest: 3,
  enableOfflineQueue: false, // fail fast during an outage; the database is our fallback
  retryStrategy: (times: number): number | null => (times > 20 ? null : Math.min(times * 200, 5000)),
};

export const redis = new Redis(process.env.REDIS_URL!, options);
redis.on("error", (err: Error) => console.error("[redis]", err.message));
```

```ts
// cache.ts — one generic helper that types the *parsed* value, not the raw string
import { redis } from "./redis.js";

export async function cached<T>(key: string, ttlSeconds: number, loader: () => Promise<T | null>): Promise<T | null> {
  const hit: string | null = await redis.get(key); // ioredis types GET as string | null
  if (hit !== null) return JSON.parse(hit) as T;   // a cast is unavoidable: Redis has no schema

  const fresh = await loader();
  // Cache the miss too, briefly, so bogus ids can't stampede the database.
  await redis.set(key, JSON.stringify(fresh), "EX", fresh === null ? 30 : ttlSeconds);
  return fresh;
}
```

```ts
// routes/users.ts
import type { Request, Response, NextFunction } from "express";
import { User } from "../models/User.js";
import { cached } from "../cache.js";

interface UserProfile { _id: string; email: string; name: string }

// Request<{ id: string }> types the params, so req.params.id is a string.
export async function getUserHandler(req: Request<{ id: string }>, res: Response, next: NextFunction): Promise<void> {
  try {
    const user = await cached<UserProfile>(`user:${req.params.id}:profile`, 300, () =>
      User.findById(req.params.id).lean<UserProfile>().exec());
    if (!user) {
      res.status(404).json({ error: "Not found" });
      return; // return void, never `return res...` — Express 5 typings expect void
    }
    res.json(user);
  } catch (err) {
    next(err); // a Redis outage must not fail the route silently
  }
}
```

If you want a stronger guarantee than `as T`, validate on the way out of the cache with [[zod]] — a schema change that leaves incompatible JSON in Redis is a real and very confusing production bug.

---

## 10. Production Setup

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:8-alpine
    ports: ["6379:6379"]
    # AOF persistence, plus cache behaviour: evict the coldest keys instead of erroring when full.
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes: ["redis-data:/data"]
volumes:
  redis-data:
```

```bash
docker compose up -d
docker exec -it $(docker compose ps -q redis) redis-cli   # a REPL against your local Redis

# .env — loaded by [[dotenv]], or by `node --env-file=.env` on Node 20.6+
REDIS_URL=redis://127.0.0.1:6379
# Managed provider: note the double-s. rediss:// makes ioredis enable TLS for you.
# REDIS_URL=rediss://default:SUPERSECRET@eu1-xyz.upstash.io:6379
```

| Server setting | Value | Why |
|---|---|---|
| `maxmemory` | Around 70% of the box's RAM | Redis needs headroom for copy-on-write during snapshots |
| `maxmemory-policy` | `allkeys-lru` for a pure cache; `volatile-lru` if the same instance also holds sessions | The default `noeviction` starts *rejecting writes* when full |
| `appendonly` | `yes` for sessions and queues; `no` is fine for a throwaway cache | AOF loses at most about a second of writes in a crash |
| `requirepass` or an ACL user | Always | An unauthenticated Redis reachable from the internet is compromised within minutes |
| Bind address | Private network only | Redis has no business listening on `0.0.0.0` |

Managed Redis — Upstash, Redis Cloud, AWS ElastiCache, Railway, Fly — is plain Redis over TLS, so ioredis connects with nothing but a `rediss://` URL. Serverless is the one place ioredis is genuinely the wrong tool: Lambda and Vercel functions are short-lived and massively parallel, so every cold start opens a new TCP connection and a traffic spike becomes a connection storm that exhausts `maxclients`. There, use `@upstash/redis`, which speaks HTTP and holds no connections at all. On a long-lived server — container, VPS, Render or Railway service — ioredis over TCP is faster and strictly better.

Your `/healthz` route should `await redis.ping()` inside a `try/catch` and return 503 on failure; `PING` is O(1) and proves the connection is genuinely usable rather than merely "open". Then watch four numbers from `redis-cli INFO`: **`keyspace_hits` vs `keyspace_misses`** (your hit rate — below roughly 80% means your keys or TTLs are wrong), **`evicted_keys`** (climbing means `maxmemory` is too low and Redis is discarding data you wanted), **`used_memory`**, and **`connected_clients`** (climbing steadily means you are creating clients per request).

Finally, decide deliberately how you degrade. A cache read should be wrapped so a Redis failure is treated as a miss and falls through to the database — slow, not down. Sessions and rate limits need a conscious choice between failing open (unlimited traffic gets through) and failing closed (503). Both are defensible; picking one by accident is not.

---

## 11. Common Gotchas & Best Practices

| Gotcha | Fix |
|---|---|
| **Process crashes on an unhandled `"error"` event** | Every ioredis client needs `redis.on("error", ...)`. Node rethrows an unlistened `error` event as an uncaught exception, so a routine reconnect blip kills your server. One line, on every client — including every `duplicate()`. |
| **`redis.set(key, someObject)` stores `"[object Object]"`** | Redis stores strings and binary only. `JSON.stringify` on the way in, `JSON.parse` on the way out — or use a Hash if you need per-field updates. |
| **`if (cached)` skips a valid cached value** | `GET` returns a string, so cached `""`, `"0"` and `"false"` are all falsy. Test `if (cached !== null)`. Same trap with `HGETALL`, which returns `{}` (truthy) for a missing key — check `Object.keys(obj).length`. |
| **Keys set without a TTL; memory climbs until Redis OOMs** | Every cache key gets `"EX", seconds`. Belt and braces: set `maxmemory` plus `maxmemory-policy allkeys-lru` so Redis evicts instead of erroring. |
| **`KEYS user:*` freezes production for a second** | Redis is single-threaded, so `KEYS` blocks every other client. Use `redis.scanStream({ match, count })`, or design keys so you never need a wildcard — keep an index in a Set. |
| **Rate limiter never triggers** | You called `EXPIRE` after every `INCR`, so steady traffic keeps pushing the window forward. Use `expire(key, secs, "NX")` (Redis 7+), or set the TTL only when `INCR` returned `1`. |
| **[[bullmq]] throws "maxRetriesPerRequest must be null"** | BullMQ needs blocking commands that can wait indefinitely. Give it its own client built with `{ maxRetriesPerRequest: null }` — don't reuse your cache client's options. |
| **"Connection in subscriber mode" on a `get` call** | A subscribed client only accepts subscribe-family commands. Create the subscriber with `redis.duplicate()` and keep the original for everything else. |

---

## 12. Alternatives — When ioredis Isn't the Best Fit

| Option | What it is | Best for |
|---|---|---|
| **ioredis** | Full-featured Node client: Cluster, Sentinel, Lua, streams, pipelining. Maintained under the official `redis` GitHub org. | **The default for any long-lived Node server.** [[bullmq]] and the [[socket_io]] adapter are both written against it. |
| **node-redis** (the `redis` package) | The other official client. Modern promise API, but needs an explicit `await client.connect()` and uses camelCase commands (`client.setEx`). | Teams who prefer its API or want its RedisJSON/RediSearch helpers. Perfectly good — just less ecosystem gravity in Node job and socket libraries. |
| **`@upstash/redis`** | Redis over HTTP/REST, no persistent connection. | Serverless and edge runtimes (Lambda, Vercel, Cloudflare Workers) where TCP connections are a liability. |
| **`lru-cache` (in-process)** | A bounded, TTL-aware `Map` inside your Node process. | Tiny, extremely hot, rarely-changing data. Zero network latency — but per-instance, lost on restart, and impossible to invalidate fleet-wide. Often the right *first* layer in front of Redis. |
| **Your existing database** | A Postgres table or a Mongo collection with a TTL index. | Low traffic, where an extra piece of infrastructure costs more in ops and money than it saves. |
| **Valkey / Dragonfly** | Protocol-compatible Redis forks born after the 2024 licence change (Redis re-added an open AGPL option in 2025). | Drop-in *server* replacements — ioredis talks to them unchanged. A hosting decision, not a client decision. |

```mermaid
%%{init: {'themeVariables': { 'primaryTextColor': '#000000', 'textColor': '#000000', 'lineColor': '#000000', 'primaryBorderColor': '#000000'}}}%%
flowchart TD
    Q1{"More than one<br/>process or instance?"}
    Q1 -->|"no, and traffic is tiny"| Local["lru-cache in-process<br/>or just your database"]
    Q1 -->|"yes"| Q2{"Serverless or edge<br/>runtime?"}
    Q2 -->|"yes, short-lived functions"| Up["@upstash/redis<br/>over HTTP"]
    Q2 -->|"no, long-lived server"| Q3{"Need BullMQ, the<br/>Socket.IO adapter,<br/>Cluster or Sentinel?"}
    Q3 -->|"yes"| IO["ioredis"]
    Q3 -->|"no strong preference"| Either["ioredis or node-redis<br/>both are official"]

    style Q1 fill:#fff2cc,stroke:#000000,color:#000000
    style Q2 fill:#fff2cc,stroke:#000000,color:#000000
    style Q3 fill:#fff2cc,stroke:#000000,color:#000000
    style IO fill:#e0ffe0,stroke:#000000,color:#000000
    style Either fill:#e0f0ff,stroke:#000000,color:#000000
    style Up fill:#e0f0ff,stroke:#000000,color:#000000
    style Local fill:#ffe0e0,stroke:#000000,color:#000000
```

---

## 13. Interview Questions

**Q: Why is Redis so much faster than a normal database?**
A: The whole dataset lives in RAM, so a read is a hash-table lookup with no disk seek, no B-tree traversal and no query planner — microseconds of real work, where the millisecond your app measures is mostly network round trip. Its single-threaded execution model also removes lock contention entirely. The trade is that your data is bounded by RAM and not durable by default.

**Q: What is cache-aside, and why delete the cache key on a write instead of updating it?**
A: Cache-aside means the application checks Redis first, falls back to the database on a miss, and writes the result back with a TTL — Redis never talks to the database itself. You delete rather than overwrite because two concurrent updates can reach Redis in the opposite order they reached the database, leaving the cache permanently wrong; deleting is idempotent, so whoever reads next re-derives the truth. You also update the database first and delete second, so nothing can refill the cache with the old value mid-write.

**Q: What happens when Redis runs out of memory?**
A: It depends entirely on `maxmemory-policy`. The default is `noeviction`, which returns OOM errors on every write while reads keep working — correct for a datastore, catastrophic for a cache. For a cache you want `allkeys-lru` so Redis silently evicts the least recently used keys. If `maxmemory` is unset, Redis grows until the OS kills the process.

**Q: Redis is single-threaded — isn't that a bottleneck?**
A: Rarely, because each command is a few microseconds of in-memory work, so one thread comfortably handles well over 100,000 operations per second. It is also a feature: single-threaded execution makes every command atomic for free, which is why `INCR` is a safe fleet-wide counter with no locking. The real danger is that one slow command blocks everyone — `KEYS *` on a big keyspace stalls every other client for its full duration.

**Q: Why do you need two Redis connections for pub/sub?**
A: Once a connection issues `SUBSCRIBE`, the Redis protocol puts it in subscriber mode where it may only run subscribe-family commands — `GET` and `SET` on that connection are rejected. So you keep your normal client for commands and create a second one with `duplicate()` for subscribing. The Socket.IO Redis adapter makes this explicit by requiring a pub client and a sub client as separate arguments.

**Q: Is Redis pub/sub reliable enough for background jobs?**
A: No. Pub/sub is fire-and-forget with no persistence and no acknowledgement — a subscriber that is disconnected when a message is published never receives it, and there is no replay. It is the right tool for broadcasting live updates between instances. For work that must actually happen, use a durable queue like BullMQ, which is built on Redis streams and sorted sets and gives you retries, delays and acknowledgement.

**Q: How would you implement rate limiting with Redis?**
A: The simple fixed window is `INCR` on a key like `rl:<ip>` inside a `MULTI` with `EXPIRE key 60 NX`, rejecting when the returned count exceeds your limit — `INCR` is atomic, so it is correct across every instance. The `NX` matters: refreshing the TTL on every request means a continuously busy client never resets its window and is never limited. For a precise sliding window, use a sorted set of request timestamps, trimming with `ZREMRANGEBYSCORE` and counting with `ZCARD`.

**Q: Is `SET key value NX PX 30000` a safe distributed lock?**
A: It is a good best-effort lock and genuinely atomic to acquire, but it is not a correctness guarantee. If the TTL expires while your task is still running — a long GC pause, a slow query — a second worker acquires the same lock and both run. Releasing also needs a Lua compare-and-delete against a random token you stored as the value, or you can delete a lock a later holder now owns. Use it to prevent duplicate work; enforce real invariants with a unique index or a transaction in your actual database.

---

## 14. Quick Cheat Sheet

```bash
npm install ioredis
docker run -d --name redis -p 6379:6379 redis:8-alpine
redis-cli ping    # -> PONG
```

```js
// One shared client — redis.js
import { Redis } from "ioredis";
export const redis = new Redis(process.env.REDIS_URL, { maxRetriesPerRequest: 3 });
redis.on("error", (err) => console.error("[redis]", err.message)); // mandatory
```

```js
// Core commands — read the cache first, always set a TTL, delete (never overwrite) on a write
await redis.set("k", "v", "EX", 60);                  // set with a 60s TTL
await redis.set("lock:x", "token", "PX", 5000, "NX"); // set only if absent — a lock
await redis.get("k");                                 // string | null, so test with !== null
await redis.incr("views:1337");                       // atomic counter
await redis.expire("k", 60, "NX");                    // TTL only if none set (Redis 7+)
await redis.ttl("k");                                 // -1 no expiry, -2 missing
await redis.del("k1", "k2");                          // variadic delete

// Batching, pub/sub (TWO clients), scanning
await redis.pipeline().get("a").get("b").exec();            // [[err, val], ...]
await redis.multi().incr("c").expire("c", 60, "NX").exec(); // same, but atomic
const sub = redis.duplicate();                              // a subscriber needs its own connection
await sub.subscribe("events");
sub.on("message", (ch, msg) => console.log(ch, JSON.parse(msg)));
await redis.publish("events", JSON.stringify({ hi: true }));
for await (const batch of redis.scanStream({ match: "user:*", count: 100 })) {
  if (batch.length) await redis.del(...batch);              // never KEYS in production
}
```

**Mental model to remember:**
> Redis is a shared, in-memory front shelf sitting in front of your real database — read it first, fall back to [[mongoose]] on a miss, write back with a TTL, and delete the key whenever the underlying row changes. Because every instance sees the same shelf, it is also where sessions, fleet-wide rate-limit counters ([[express_rate_limit]]), cross-instance broadcasts ([[socket_io]]) and background jobs ([[bullmq]]) belong. Keep exactly one ioredis client per process, always attach an `error` listener, always set a TTL — and never let Redis hold the only copy of anything you would hate to lose.
