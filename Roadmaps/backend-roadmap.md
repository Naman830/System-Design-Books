# The Backend Roadmap (JavaScript → TypeScript)

**From "I know JavaScript" → "I can design, build, ship and scale a real backend on my own."**

This is a **map, not a textbook.** It does not teach topics — it tells you *what to learn, in what order, why it matters at that moment, what to build to prove it, and when you are allowed to move on.*

Every time a topic already has a deep-dive note in this repo, it is linked. Read the note **when the phase tells you to**, not before. Reading notes out of order is the single most common way people stall.

---

## How to read this roadmap

| Section | What it means |
|---|---|
| **Learn** | The concepts to understand in this phase. |
| **Why now** | Why this phase comes *here* and not earlier or later. |
| **Build** | The project that forces you to actually learn it. Non-negotiable. |
| **Move on when** | The honest checklist. If you can't tick every box, stay. |
| **Skip / Don't** | Traps that waste months. |

**Three rules that decide whether you finish this roadmap or abandon it:**

1. **Build > watch.** A phase is not complete because you understood the video. It is complete when the checklist passes.
2. **One project carries you across many phases.** Do not start a new project every phase. You will grow *one* app from Phase 2 all the way to Phase 11. That single evolving codebase is what makes you senior — not 12 tutorials.
3. **No time estimates on purpose.** People move at different speeds and honest speed varies 5x. The checklists are the clock.

---

## The shape of the journey

```
        ┌──────────────────────────────── FOUNDATION ────────────────────────────────┐
Phase 0   Prerequisites — HTTP, terminal, git, async JS
Phase 1   Node itself — the runtime, not the framework
Phase 2   Your first real HTTP server — Express
        └────────────────────────────────────────────────────────────────────────────┘

        ┌──────────────────────────────── COMPETENT ─────────────────────────────────┐
Phase 3   Persistence #1 — MongoDB + Mongoose
Phase 4   Auth, sessions, and not getting hacked
Phase 5   TypeScript + a codebase that doesn't rot
Phase 6   Persistence #2 — PostgreSQL + Prisma, and real data modeling
Phase 7   Testing — the line between hobbyist and professional
        └────────────────────────────────────────────────────────────────────────────┘

        ┌──────────────────────────── PROFESSIONAL / HIRED ──────────────────────────┐
Phase 8   Async work — Redis, caching, queues, background jobs
Phase 9   Realtime — WebSockets and stateful connections
Phase 10  Shipping — deployment, process management, CI/CD, observability
        └────────────────────────────────────────────────────────────────────────────┘

        ┌──────────────────────────────── RESPECTED ─────────────────────────────────┐
Phase 11  Scale & distributed systems — where system design becomes real
Phase 12  The senior bar — judgment, tradeoffs, ownership
        └────────────────────────────────────────────────────────────────────────────┘
```

**Where does "backend" end?** Honest answer up front, so you can pace yourself:

- **End of Phase 7** → you are hireable. You can be dropped onto a team and be useful.
- **End of Phase 10** → you are independently valuable. You can own a service end to end, alone, in production.
- **End of Phase 12** → you are the person others ask. This is the "great developer" bar — earned through judgment, not through knowing more libraries.

**Phase 10 is "sufficient" for almost every job and every product you will ever build alone.** Phases 11–12 are what separate *employed* from *respected*. Read the [Am I done yet?](#am-i-done-yet) section at the bottom before you start, so you know what you're aiming at.

---

# FOUNDATION

## Phase 0 — Prerequisites

You have JavaScript. You are missing the ground under it.

**Learn**
- **How the web actually works:** client → DNS → TCP → HTTP request → server → response. Request methods, status codes, headers, body, query vs params vs body.
- **The terminal:** navigating, `cd/ls/mkdir/rm`, environment variables, killing a process on a port, reading a stack trace without panicking.
- **git:** branch, commit, push, merge, resolve a conflict, read `git log`. Not optional — you will use it every single day for the rest of your career.
- **Async JavaScript, properly:** the event loop, callbacks → promises → `async/await`, `Promise.all` vs sequential awaits, error handling in async code, why a `try/catch` around a non-awaited promise catches nothing.

📖 Repo note: **[Networking fundamentals — complete guide](../Computer_Network/01_networking-fundamentals-complete-guide.md)** — read the HTTP and TCP sections now, skim the rest, come back in Phase 11.

**Why now**
Every bug you hit for the next year is either an async bug or an HTTP bug. If async JS is shaky, everything after this feels like magic and you'll never debug confidently.

**Build**
A script that fetches from a public API, transforms the data, and writes it to a JSON file. Then rewrite it to fetch 10 URLs *in parallel* and handle one of them failing without killing the rest.

**Move on when**
- [ ] You can explain what happens between typing a URL and seeing a page, without hand-waving.
- [ ] You can say what 200 / 201 / 400 / 401 / 403 / 404 / 409 / 500 mean and when *you* would return each.
- [ ] You know why `forEach` with an `async` callback doesn't wait, and what to use instead.
- [ ] You can recover a git repo you've messed up, without deleting the folder.

**Skip / Don't**
- Don't learn a framework yet.
- Don't learn TypeScript yet — Phase 5. Learning TS and backend simultaneously doubles the confusion and you won't know which one is fighting you.

---

## Phase 1 — Node itself

Most people skip straight to Express and spend years never understanding the thing Express runs on. Don't be them — this phase is 80% of what makes you look senior later.

**Learn**
- What Node *is*: V8 + libuv + the standard library. Single-threaded JS, multi-threaded I/O.
- The event loop **in Node specifically** — phases, `setTimeout` vs `setImmediate` vs `process.nextTick`, why a heavy `for` loop freezes your entire server for every user.
- Core modules: `fs`, `path`, `http`, `events`, `stream`, `crypto`, `os`, `process`.
- **Streams and buffers** — the concept, at minimum. Why you stream a 2GB file instead of reading it into memory.
- **Modules:** CommonJS vs ESM, `require` vs `import`, why the mix breaks things.
- **npm:** `package.json`, dependencies vs devDependencies, semver (`^` vs `~`), lockfiles, `npm ci` vs `npm install`, what `node_modules` really is.
- `process.env`, `process.argv`, exit codes, `process.on('uncaughtException')`.

**Why now**
Express is ~200 lines of convenience over `http`. Once you've written a server with raw `http`, Express stops being magic and starts being a tool you can debug, extend, and eventually replace.

**Build**
An HTTP server using **only** the `http` module — no Express. Route `GET /users` and `POST /users` by hand. Parse the body yourself. Set your own headers and status codes. Serve a file with a stream.

It will be ugly. That's the point — you're earning the right to appreciate Express.

**Move on when**
- [ ] You wrote a working server with zero dependencies.
- [ ] You can explain why Node is "single-threaded but non-blocking" without saying the word "magic".
- [ ] You can explain what CPU-bound work does to a Node server and what you'd do about it.
- [ ] You understand what a lockfile is for and why you commit it.

**Skip / Don't**
- Don't memorize the whole stdlib. Know it exists, look it up.
- Don't go down the worker_threads / cluster rabbit hole yet — Phase 10.

---

## Phase 2 — Your first real HTTP server (Express)

This is where the project you carry to the end of this roadmap begins.

**Learn**
- Routing, route params, query strings, `express.json()`.
- **Middleware** — the whole model. Order matters, `next()`, `next(err)`, error-handling middleware with 4 args.
- Project structure: routes → controllers → services → models. Why the layers exist.
- Centralized error handling and a consistent error response shape.
- Config via environment variables — never hardcode a secret, not even once, not even locally.
- Input validation at the edge of your system.
- HTTP correctness: right status codes, RESTful resource naming, idempotency of `PUT` vs `POST`.

📖 Repo notes, in this order:
1. **[Express deep dive](../Libraries_Depth/express/express.md)** ← the core of this phase
2. **[nodemon](../Libraries_Depth/nodemon/nodemon.md)** — stop restarting the server by hand
3. **[dotenv](../Libraries_Depth/env/dotenv.md)** — config and secrets
4. **[CORS](../Libraries_Depth/cors/cors.md)** — the moment a frontend calls your API, this stops being theory
5. **[Zod](../Libraries_Depth/zod/zod.md)** — validate every incoming request
6. **[uuid / nanoid](../Libraries_Depth/id_generation/uuid_nanoid.md)** — how to generate IDs before a DB does it for you

**Why now**
You now have a server you can shape. Everything from here is adding capability to *this* codebase.

**Build**
🏗️ **THE PROJECT — start it here and never abandon it.**

Pick something with real domain rules, not another todo app. Good candidates: an expense splitter, a job board, a booking system, a URL shortener with analytics, a small e-commerce backend.

Right now it's in-memory (a plain array). No database yet. Full CRUD, validated input, proper status codes, layered folders, a real error handler.

*(Stuck for an idea? → [project ideas](../Random_Learning/project_ideas/5_ideas.md))*

**Move on when**
- [ ] Your app is in layers, not one 900-line `index.js`.
- [ ] Every route validates its input, and invalid input returns a clean 400 — never a crash, never a 500.
- [ ] One error handler owns all error responses; controllers just `next(err)`.
- [ ] No secret or config value is hardcoded anywhere.
- [ ] You can explain middleware order to someone else.

**Skip / Don't**
- Don't add a database yet. Getting HTTP right first is much easier without Mongo confusing the picture.
- Don't chase Fastify/Nest/Hono. Express first — its concepts transfer to all of them.

---

# COMPETENT

## Phase 3 — Persistence #1: MongoDB + Mongoose

**Learn**
- Documents, collections, BSON. How this differs from tables and rows.
- Schemas, models, and why a "schemaless" DB still needs a schema in your code.
- CRUD, filters, projections, sorting, pagination.
- Relationships: embedding vs referencing, `populate()` and its cost.
- **Indexes** — what they are, when you need one, how to prove you need one.
- Transactions, and the fact that they exist here too.
- Connection lifecycle: connect once at boot, not per request. Connection pooling.

📖 Repo note: **[Mongoose deep dive](../Libraries_Depth/mongoose/mongoose.md)**

**Why now**
In-memory data dies on restart. This is the first time your app becomes *real*.

**Build**
Move THE PROJECT onto MongoDB. Every array becomes a collection. Add pagination and filtering to your list endpoints. Add at least one index and be able to say *why*.

**Move on when**
- [ ] Data survives a restart.
- [ ] Your list endpoint paginates — no endpoint returns an unbounded array.
- [ ] You can explain, for *your* schema, why you embedded one thing and referenced another.
- [ ] You've seen a slow query get fast because of an index you added.
- [ ] There are no DB queries inside your controllers — they live in a service/repository layer.

**Skip / Don't**
- Don't learn the aggregation pipeline in depth yet. Know it exists, use it when a real query demands it.
- Don't argue about SQL vs NoSQL yet — you'll have an informed opinion after Phase 6. Opinions before then are borrowed.

---

## Phase 4 — Auth and not getting hacked

The phase that separates "it works on my machine" from "I'd let strangers use this."

**Learn**
- **Password storage** — hashing vs encryption, salts, why fast hashes are wrong here, work factors.
- **Sessions vs JWT** — the actual tradeoff, not the blog-post version. Statefulness, revocation, and why "JWT is better" is not an answer.
- Access tokens + refresh tokens, rotation, expiry, storage on the client (cookie vs localStorage, and what each is vulnerable to).
- **Authentication vs authorization.** Roles, ownership checks, and why the second one is where most real breaches live.
- OAuth / "Login with Google" at the flow level.
- The **OWASP Top 10** — at least well enough to recognize each one in your own code.
- Rate limiting and brute-force protection.
- Security headers and why CORS is not a security feature for your server.

📖 Repo notes:
1. **[Password hashing — overview](../Libraries_Depth/password_hashing/01_password_hashing.md)** → then **[bcrypt](../Libraries_Depth/password_hashing/bcrypt/bcrypt.md)**, **[argon2](../Libraries_Depth/password_hashing/argon2/argon2.md)**, **[scrypt](../Libraries_Depth/password_hashing/scrypt/scrypt.md)**
2. **[jsonwebtoken](../Libraries_Depth/jsonwebtoken/jsonwebtoken.md)**
3. **[Helmet](../Libraries_Depth/helmet/helmet.md)**
4. **[express-rate-limit](../Libraries_Depth/rate_limiting/express_rate_limit.md)**
5. **[Nodemailer](../Libraries_Depth/nodemailer/nodemailer.md)** — verification and password-reset emails
6. **[Multer](../Libraries_Depth/multer/multer.md)** — file upload, and the security holes that come with it

**Why now**
Auth touches every route you've written. Doing it after you have routes but before you have 60 of them is the cheapest moment.

**Build**
Add to THE PROJECT: register, login, logout, refresh, email verification, forgot/reset password, role-based access, and per-resource ownership checks. Rate-limit the login and reset routes. Add avatar/file upload.

**Move on when**
- [ ] A user cannot read, edit, or delete another user's resource — and you've *tried* it with a real request, not just assumed.
- [ ] You can explain why you chose sessions or JWT for this app, including what you gave up.
- [ ] You can revoke a logged-in user's access. (If you can't, you don't understand your own auth.)
- [ ] Passwords are hashed with a slow, salted algorithm and you can name the work factor you used.
- [ ] You can name every OWASP Top 10 item and point to where your app defends against it.

**Skip / Don't**
- Don't write your own crypto. Ever. Not as an exercise, not "just to learn."
- Don't reach for Auth0/Clerk yet. Build it once by hand — then you'll know what those services are actually doing for you.

---

## Phase 5 — TypeScript and a codebase that doesn't rot

**Learn**
- TS basics: types, interfaces, unions, generics, `unknown` vs `any`, narrowing.
- TS in a **Node/Express** context specifically: typing `req`/`res`, extending `Request` for `req.user`, typing async handlers, `tsconfig`, build vs `ts-node`, ESM/CJS interop pain.
- Sharing one source of truth between validation and types (infer types from your Zod schemas rather than writing both).
- Linting and formatting — as an automated gate, not a preference.
- Pre-commit hooks so bad code physically cannot enter the repo.
- Running multiple dev processes at once.
- Conventions: naming, folder structure, barrel files, absolute imports.

📖 Repo notes:
1. **[ESLint + Prettier](../Libraries_Depth/linting/eslint_prettier.md)**
2. **[Husky + lint-staged](../Libraries_Depth/git_hooks/husky_lint_staged.md)**
3. **[concurrently](../Libraries_Depth/concurrently/concurrently.md)**
4. Revisit **[Zod](../Libraries_Depth/zod/zod.md)** — now for type inference, not just validation

**Why now**
Late enough that you understand what you're typing; early enough that migrating is still a weekend and not a quarter. Your project is now big enough to hurt without types — that pain is the lesson.

**Build**
Migrate THE PROJECT to TypeScript, incrementally, without a rewrite. Turn on `strict`. Get to zero `any` in your own code. Add ESLint + Prettier + a pre-commit hook that blocks a commit failing lint or typecheck.

**Move on when**
- [ ] `strict: true`, zero `any` you wrote yourself.
- [ ] `req.user` is properly typed everywhere, with no casting hacks.
- [ ] Your request types are *derived* from your validation schemas, not duplicated by hand.
- [ ] A commit with a type error or lint error is impossible on your machine.
- [ ] You can explain why TS gives you nothing at runtime, and what you use instead at the boundary.

**Skip / Don't**
- Don't rewrite from scratch. Rename `.js` → `.ts` file by file. Rewrites are where projects go to die.
- Don't over-engineer types. Conditional-type wizardry is not the goal; catching real bugs is.

---

## Phase 6 — Persistence #2: PostgreSQL + Prisma

Now you get an actual opinion about databases.

**Learn**
- **Relational modeling:** tables, keys, 1-1 / 1-N / N-N, join tables, normalization and when to break it deliberately.
- **SQL by hand** — `SELECT`, `JOIN`, `GROUP BY`, `HAVING`, subqueries, `EXPLAIN`. Yes, raw SQL, before the ORM. You cannot debug an ORM you can't out-reason.
- **ACID and transactions** — atomicity in a real multi-step operation, isolation levels, deadlocks.
- Constraints as correctness: `NOT NULL`, `UNIQUE`, foreign keys, `CHECK`. Let the DB enforce truth.
- **Indexes for real:** B-tree, composite index column order, covering indexes, when an index *hurts*.
- **Migrations** — versioned schema change, and why editing prod schema by hand ends careers.
- **The N+1 query problem.** Find it in your own code. It is there.
- Connection pooling, and why serverless breaks it.

📖 Repo notes:
1. **[Postgres + Prisma guide](../Database/Postgress%26Prisma/postgres-prisma-nextjs-guide.md)**
2. **[Prisma deep dive](../Libraries_Depth/prisma/prisma.md)**

**Why now**
You've lived with a document DB long enough to feel its edges. Learning Postgres now means comparing two things you've actually shipped — that's where real architectural judgment comes from.

**Build**
Two options, both valid:
- **(A)** Port THE PROJECT to Postgres + Prisma. Model the relations properly. Write the migrations.
- **(B)** Keep Mongo for one domain, add Postgres for another (e.g. transactional/financial data), and be able to defend the split.

Either way: write the 5 hardest queries in your app as raw SQL first, then as Prisma. Compare.

**Move on when**
- [ ] You can design a normalized schema for a new feature on paper before writing code.
- [ ] You can write a multi-table `JOIN` with aggregation from memory.
- [ ] You've wrapped a real multi-step operation in a transaction and tested that it rolls back.
- [ ] You've found and fixed an N+1 in your own code.
- [ ] You can read an `EXPLAIN` plan enough to tell whether your index is being used.
- [ ] You can argue **both** sides of Postgres vs MongoDB using your own project as evidence.

**Skip / Don't**
- Don't let Prisma be the reason you never learn SQL. The ORM is a convenience, not a replacement.
- Don't chase every ORM. Prisma + raw SQL is plenty.

---

## Phase 7 — Testing

The clearest line between hobbyist and professional. Most self-taught developers stop right before this phase — which is exactly why crossing it puts you ahead.

**Learn**
- The pyramid: unit / integration / e2e, and what belongs at each level.
- Testing HTTP endpoints for real (request in → response out), not mocking your own framework.
- Mocking, stubbing, spies — and the discipline of mocking only what you don't own.
- Test databases, fixtures/factories, isolation between tests, teardown.
- Coverage as a signal, not a target.
- Writing code that is *testable*: dependency injection, pure functions, keeping I/O at the edges.

📖 Repo note: **[Jest + Supertest](../Libraries_Depth/testing/jest_supertest.md)**

**Why now**
You have auth, two databases, and real business rules. Refactoring is now genuinely scary. Tests are what make it stop being scary — and everything after this phase involves heavy refactoring.

**Build**
Test THE PROJECT: unit tests for business logic, integration tests hitting real endpoints against a test DB, full coverage of the auth flows (including the failure paths — wrong password, expired token, forbidden resource). Then deliberately break something and watch a test catch it.

**Move on when**
- [ ] `npm test` runs the whole suite against a clean test DB, every time, with no manual setup.
- [ ] Tests are order-independent and pass on a fresh machine.
- [ ] Every auth failure path is tested, not just the happy path.
- [ ] You've refactored something scary and trusted the suite instead of clicking through by hand.
- [ ] You write the test first at least sometimes, because it's genuinely faster there.

**Skip / Don't**
- Don't chase 100% coverage. Chase confidence.
- Don't unit-test getters and setters to inflate the number.

> ### ✅ CHECKPOINT: You are hireable.
> With Phases 0–7 done and one substantial project proving it, you can pass backend interviews and be productive on a real team. Everything after this is about *ownership* — running it, scaling it, and being trusted with it.

---

# PROFESSIONAL

## Phase 8 — Async work: caching, queues, background jobs

Where your app stops doing everything inside the request.

**Learn**
- **Redis** as: cache, session store, rate-limit counter, pub/sub, lock.
- **Caching strategy:** cache-aside, TTL, and **invalidation** — the genuinely hard part. Stale data bugs, thundering herd, cache stampede.
- Why some work must leave the request: emails, image processing, reports, third-party calls, webhooks.
- **Queues and workers:** producer/consumer, retries with backoff, dead-letter queues, **idempotency**, at-least-once vs exactly-once delivery.
- Scheduled/recurring jobs.
- Graceful shutdown — draining in-flight work instead of dropping it.

📖 Repo notes:
1. **[Redis / ioredis](../Libraries_Depth/redis/ioredis.md)**
2. **[BullMQ](../Libraries_Depth/bullmq/bullmq.md)**
3. **[axios](../Libraries_Depth/axios/axios.md)** — calling other services, with timeouts and retries

**Why now**
This is the first phase that's about *architecture* rather than features. It's also the first time your app has more than one moving process — the mental shift that makes Phase 11 possible.

**Build**
Add to THE PROJECT: cache your heaviest read endpoint in Redis (and invalidate it correctly on write). Move every email to a BullMQ queue with retries. Add one scheduled job. Run the worker as a **separate process** from your API. Then kill the worker mid-job and prove nothing is lost or double-processed.

**Move on when**
- [ ] Your slowest endpoint is measurably faster, and you have the before/after numbers.
- [ ] You can explain your invalidation strategy and name where it could still serve stale data.
- [ ] No HTTP request in your app waits on an email or a slow third party.
- [ ] Your jobs are idempotent — running one twice causes no damage, and you can explain why.
- [ ] Killing a worker mid-job loses nothing.
- [ ] Your app shuts down gracefully on `SIGTERM`.

**Skip / Don't**
- Don't cache before you've measured. Caching the wrong thing adds bugs and zero speed.
- Don't reach for Kafka. BullMQ + Redis is the right tool at your scale, and the concepts transfer.

---

## Phase 9 — Realtime

**Learn**
- WebSockets vs HTTP polling vs SSE — and when *not* to use WebSockets.
- The upgrade handshake, connection lifecycle, heartbeats, reconnection.
- Rooms, namespaces, broadcasting.
- **Authenticating a socket** — different from authenticating a request.
- The scaling problem: sockets are stateful, so a second server instance breaks everything. Pub/sub adapters as the fix.
- Backpressure and what happens when a client can't keep up.

📖 Repo notes, in this order:
1. **[WebSocket — complete guide](../WebSocket/websocket-complete-guide.md)** (protocol first)
2. **[Socket.IO explained](../WebSocket/socket-io-explained.md)**
3. **[socket.io deep dive](../Libraries_Depth/socket_io/socket_io.md)**
4. *Optional, only if you need peer-to-peer media:* **[WebRTC fundamentals](../WebRtc/01_webRtc_fundamentals.md)** → **[signaling](../WebRtc/02_webRtc_Singnaling.md)** → **[media & security](../WebRtc/03_webRtc_Media_Security.md)**

**Why now**
Requires Phase 8: the correct way to scale WebSockets is a Redis pub/sub adapter. Doing realtime before Redis teaches you a pattern you'll have to unlearn.

**Build**
Add a genuinely realtime feature to THE PROJECT — live notifications, presence, a chat, or a live-updating dashboard. Authenticate the socket connection using your existing auth. Then run **two instances** of your server behind a load balancer and make a message sent to instance A reach a user connected to instance B.

That last step is the whole phase.

**Move on when**
- [ ] Unauthenticated sockets are rejected at connect time.
- [ ] Messages cross server instances correctly via a Redis adapter.
- [ ] Clients reconnect cleanly after a network drop without duplicating or losing state.
- [ ] You can explain why you chose WebSockets here instead of polling or SSE.

**Skip / Don't**
- Don't put realtime in an app that doesn't need it just to show off. Judgment is part of the skill.

---

## Phase 10 — Shipping: deployment, ops, observability

Code that isn't running in production is a hobby. This is the phase most self-taught developers never finish — and the one that most changes how people see you.

**Learn**
- **Linux server basics:** SSH, users and permissions, systemd, firewall, `nginx` as a reverse proxy, TLS certificates.
- **Process management:** clustering to use all CPU cores, auto-restart on crash, zero-downtime reloads.
- **Docker:** images vs containers, `Dockerfile`, multi-stage builds, `docker-compose` to run app + Postgres + Redis together, `.dockerignore`.
- **CI/CD:** on push → lint, typecheck, test, build, deploy. Automatically.
- **Environments:** dev / staging / prod, secret management, and *never* the same database.
- **Observability — the big three:**
  - **Logs** — structured, leveled, with a request ID that traces one request across your whole system.
  - **Metrics** — latency (p50/p95/p99, not average), throughput, error rate, saturation.
  - **Traces** — where the time actually went.
- **Health checks** — liveness vs readiness, and what a load balancer does with each.
- **Error tracking** and alerting. Alerts that mean something, so you don't learn to ignore them.
- **Backups**, and the fact that an untested backup is not a backup.

📖 Repo notes:
1. **[Winston + Morgan](../Libraries_Depth/logging/winston_morgan.md)**
2. **[PM2](../Libraries_Depth/pm2/pm2.md)**
3. **[VPS deployment guide](../Random_Learning/JoinSecret/vps-deployment-guide.md)** — an end-to-end real deployment

**Why now**
Everything you've built exists. Now it has to survive contact with the real world, on a machine you don't babysit.

**Build**
Deploy THE PROJECT to a real VPS on a real domain with real HTTPS.
- Dockerize it. `docker-compose` brings up app + Postgres + Redis.
- nginx reverse proxy, TLS via Let's Encrypt.
- PM2 or Docker running it in cluster mode, restarting on crash, surviving a reboot.
- A CI pipeline that tests and deploys on merge to `main`.
- Structured JSON logs with a request ID threaded through every log line of a request.
- A `/health` endpoint your load balancer actually uses.
- Error tracking wired up.
- An automated database backup — **and restore it once, to prove it works.**

**Move on when**
- [ ] Strangers can use your app over HTTPS on your own domain.
- [ ] `git push` to `main` deploys it, with no manual steps.
- [ ] You can trace one user's single request across API and worker logs using one request ID.
- [ ] The server survives a reboot with no human intervention.
- [ ] You know your p95 latency and your error rate — as numbers, not vibes.
- [ ] You have successfully restored from a backup.
- [ ] When something breaks, you go to logs/metrics — not to `console.log`.

**Skip / Don't**
- Don't start with Kubernetes. One VPS you fully understand teaches more than a managed cluster you don't.
- Don't use PaaS-only (Vercel/Render) for this phase. Do it the hard way *once* — then use the easy way forever, knowing what it hides.

> ### ✅ CHECKPOINT: You are independently valuable.
> Phases 0–10 mean you can take an idea to a running, monitored, maintained production system alone. **This is "sufficient" for the overwhelming majority of backend jobs and for any product you'll build yourself.**
>
> If you stop here, you're a genuinely good backend developer. Phases 11–12 are what make you a *respected* one.

---

# RESPECTED

## Phase 11 — Scale and distributed systems

Everything so far assumed one server, one database, and things generally working. Reality assumes none of that.

**Learn**
- **Horizontal vs vertical scaling.** Load balancing, and what "stateless service" really demands of your code.
- **Database scale:** read replicas, replication lag, connection pooling at scale, partitioning, sharding, and the pain each one buys.
- **Caching layers:** app cache → Redis → CDN. Where each belongs.
- **CAP, consistency models,** eventual consistency, and why "eventually" is a product decision, not a technical one.
- **Failure as normal:** timeouts, retries with jitter, circuit breakers, bulkheads, graceful degradation. Cascading failure, and how a retry storm becomes an outage.
- **Monolith → services:** when to split (and the much more important question of when *not* to). Service boundaries, sync vs async communication, the distributed monolith anti-pattern.
- **Event-driven architecture:** event sourcing, CQRS, the outbox pattern, sagas for cross-service transactions.
- **Idempotency and exactly-once delivery** — properly this time. Distributed transactions, two-phase commit and why people avoid it.
- **API design at scale:** versioning, deprecation, pagination strategies (offset vs cursor), REST vs GraphQL vs gRPC.
- **Multi-tenancy**, quotas, noisy-neighbour isolation.
- **Deeper networking:** TCP vs UDP, TLS handshakes, HTTP/1.1 vs HTTP/2 vs HTTP/3, keep-alive, DNS and load balancing, latency numbers every engineer should know.

📖 Repo notes:
- **[Networking fundamentals — complete guide](../Computer_Network/01_networking-fundamentals-complete-guide.md)** — now read it *fully*, plus **[02](../Computer_Network/02_networking-fundamentals.md)** and **[03](../Computer_Network/03_networking-fundamentals.md)**
- **[Networking interview question bank](../Computer_Network/interview_question_bank.md)**
- **[CRDT vs OT — part 1](../System_Design/CRDT_OT/01_CRDT_vs_OT.md)** and **[part 2](../System_Design/CRDT_OT/02_CRDT_vs_OT.md)** — how distributed state converges without a central authority
- Worked designs to study: **[collaborative code editor](../Random_Learning/CollabrativeCodeEditor/collaborative-code-editor-system-design.md)**, **[deals/membership platform](../Random_Learning/JoinSecret/deals-membership-platform-design.md)**

> The `../System_Design/`, `../Database/`, and `../Computer_Network/` folders are your reference material for this entire phase — this roadmap points at the milestones, those notes carry the depth.

**Why now**
These ideas are meaningless without production scars. You now have the scars from Phases 8–10, so each pattern lands as "oh, *that's* how you fix the thing that bit me."

**Build**
Two tracks, do both:

**Track A — break your own project.**
Load-test THE PROJECT until it falls over. Find the *actual* bottleneck with your Phase 10 metrics. Fix it. Repeat 3 times. You'll learn more from three real bottlenecks than from any book — because the bottleneck is almost never where you predicted.

**Track B — design on paper.**
Design 5–10 well-known systems end to end: URL shortener, news feed, chat, rate limiter, notification service, ride-hailing, file storage. Write the tradeoffs down. Be specific about numbers.

Then extract *one* piece into a real service — split your worker out into a standalone service that talks to your API over a queue, with its own deploy.

**Move on when**
- [ ] You've found and fixed 3 real bottlenecks in your own system, with measurements.
- [ ] You can design a system on a whiteboard, defend the tradeoffs, and change the design when someone adds a constraint.
- [ ] You can explain what breaks first when traffic 10×'s in *your* app, specifically.
- [ ] You can name a case where the right answer is "don't split this into services."
- [ ] Your service degrades gracefully when a dependency is down instead of cascading into a full outage.
- [ ] You reach for boring, proven solutions and can justify when you don't.

**Skip / Don't**
- Don't design for scale you don't have. Premature distribution is more damaging than premature optimization.
- Don't confuse *knowing* the patterns with *having judgment about* them. Judgment only comes from Track A.

---

## Phase 12 — The senior bar

There is no new library here. This is what people actually mean when they call someone a great backend developer.

**What "great" actually is**

**Judgment over knowledge.** Everyone can look up how BullMQ works. Knowing whether this problem needs a queue *at all* is the rare part. Great developers are recognized for the complexity they *avoided*.

**You optimize for the reader.** Code is read far more than written. Your naming, structure, and boundaries are chosen so the next person — probably you in eight months — understands it fast.

**You think in tradeoffs, not answers.** "It depends" followed by *what* it depends on, and a clear recommendation. Never a technology preference dressed up as a technical fact.

**You own the whole lifecycle.** Design → build → test → ship → monitor → on-call → fix → document. "Done" means running reliably in production, not merged.

**You write.** Design docs before big changes. Clear PR descriptions. Honest postmortems that fix systems instead of blaming people. Writing is how a senior engineer scales past their own hands.

**You reduce risk.** Incremental rollouts, feature flags, reversible migrations, a rollback plan before deploying. You assume you'll be wrong and make being wrong cheap.

**You make others better.** Reviews that teach. Answers that leave someone able to solve the next one alone. This is the actual multiplier — and it's what "respect" is built on.

**You know what you don't know**, say so plainly, and go find out.

**The habits that get you here**
- Read the source of a library you depend on. At least once. It demystifies everything.
- Run a real thing for real users and stay responsible for it over time. Nothing else teaches operational judgment.
- Do postmortems on your own outages, even solo. Write down the cause and the fix.
- Rebuild something you built a year ago and notice what you'd do differently.
- Teach — write the note, answer the question, explain it to a beginner. (This repo is that habit.)
- Follow the primary sources: RFCs, database docs, release notes. Not just tutorials.

**You are here when**
- [ ] People bring you their designs *before* building.
- [ ] You've said "we don't need that" and been right.
- [ ] You've been on call for something you built, and it held.
- [ ] You can join an unfamiliar codebase and be useful within days.
- [ ] You can explain any part of your system to a junior *and* to a non-technical stakeholder.
- [ ] Your first instinct on a hard bug is to reach for data, not to guess.
- [ ] You've changed your mind publicly because of evidence.

---

## Am I done yet?

| If you can... | You are |
|---|---|
| Build a validated CRUD API with real error handling | **Beginner — Phase 2** |
| Add auth, model data properly, and not get owned | **Junior — Phase 4–6** |
| Test it so refactoring isn't scary | **Hireable — Phase 7** |
| Cache it, queue it, and run it in production with monitoring | **Independent — Phase 10** |
| Find the bottleneck before it finds you, and design for scale you can defend | **Senior — Phase 11** |
| Choose *not* to build the complex thing, and explain why | **The bar — Phase 12** |

**Backend ends at Phase 10 for practical purposes.** It ends at Phase 12 for reputational ones. Everything past that is depth in a specialization you choose — data infrastructure, platform, security, performance — not more "backend."

---

## Things you can safely ignore

- **Every new framework.** Express concepts transfer everywhere. Learn Fastify/Nest/Hono when a job or a real constraint requires it, in an afternoon, not now.
- **Kubernetes**, until you have a team and multiple services that genuinely need it.
- **Microservices**, until a monolith is actually hurting you. Most companies that split early regret it.
- **GraphQL**, until REST is provably the wrong shape for your clients.
- **Kafka**, until BullMQ/Redis genuinely cannot cope.
- **Serverless-first thinking**, until you understand the servers it's hiding.
- **Tutorial hell.** If you've watched more hours this month than you've committed, stop watching and go break something.

---

## Related notes in this repo

- **[Libraries — deep dives](../Libraries_Depth/)** — the per-library reference this roadmap links into
- **[System Design](../System_Design/)** — Phase 11 depth
- **[Database](../Database/)** — Phase 3 & 6 depth
- **[Computer Network](../Computer_Network/)** — Phase 0 & 11 depth
- **[WebSocket](../WebSocket/)** / **[WebRTC](../WebRtc/)** — Phase 9 depth
- **[Random Learning](../Random_Learning/)** — worked system designs, deployment guides, project ideas

---

*Build one thing. Take it all the way to production. Keep it alive. That single act teaches more than the next ten tutorials — and it is the only part of this roadmap nobody can fake.*
