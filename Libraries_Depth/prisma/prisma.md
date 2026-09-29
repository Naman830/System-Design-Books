# prisma — Type-Safe SQL Without Writing SQL

> **Scope:** Prisma ORM for Node.js — `schema.prisma`, migrations, the generated client, relations, transactions, connection pooling, and production deployment. PostgreSQL is the example database; MySQL, SQLite and SQL Server work the same way.
> **Level:** Beginner + practical.
> **New to environment variables?** Read [[dotenv]] first — Prisma reads your database URL from a `DATABASE_URL` env var, and nothing works until that is set.

---

## 1. ELI5: What is Prisma?

You wired up a Postgres database and wrote your first query by hand:

```js
const { rows } = await pool.query("SELECT id, email FROM users WHERE id = $1", [id]);
console.log(rows[0].emial); // undefined — no error, no warning, no red squiggle
```

You typed `emial` instead of `email` and nothing complained, because `rows` is `any` and every property access on `any` is legal. Misspell a column *inside* the SQL string and it is barely better — the database rejects it only when that line actually runs, which may well be in production. Multiply that by every column rename, every new table, every `JOIN` you half-remember the syntax for, and every teammate who runs `ALTER TABLE` on their laptop and forgets to tell you. **Prisma** is like a **translator who has memorized the blueprint of your building**: you hand over one blueprint file — every room, every door between rooms — and from it they physically build the rooms in the database (**migrations**) and hand you a phrasebook where every sentence you are allowed to say is already written out correctly (**the generated client**). You cannot ask for a room that is not on the blueprint, because the phrasebook does not contain that sentence. Your editor autocompletes it, or refuses it.

> **Full name:** Prisma ORM — the CLI, the query engine, and the generated client
> **Type:** npm packages — `prisma` (CLI, dev dependency) and `@prisma/client` (runtime)
> **Core promise:** One schema file is the single source of truth; from it Prisma generates both your database tables and a fully typed query client, so column typos and shape mismatches become compile-time errors instead of production incidents.

```mermaid
%%{init: {'themeVariables': { 'primaryTextColor': '#000000', 'textColor': '#000000', 'lineColor': '#000000', 'primaryBorderColor': '#000000'}}}%%
flowchart LR
    S["schema.prisma<br/>your models"] -->|"prisma migrate dev"| DB["Database<br/>real tables"]
    S -->|"prisma generate"| C["Generated client<br/>typed methods"]
    C -->|"prisma.user.findMany"| R["Rows with<br/>exact TS types"]

    style S fill:#fff2cc,stroke:#000000,color:#000000
    style DB fill:#ffffff,stroke:#000000,color:#000000
    style C fill:#e0f0ff,stroke:#000000,color:#000000
    style R fill:#e0ffe0,stroke:#000000,color:#000000
```

---

## 2. Why Does Prisma Exist? (The Problem It Solves)

Here is life without it — a "get a user and their posts" endpoint written with the raw `pg` driver:

```js
// ❌ The old way — correct-looking, quietly fragile
const userRes = await pool.query("SELECT * FROM users WHERE id = $1", [id]);
const user = userRes.rows[0]; // type: any, shape: whoever last edited the table
if (!user) return res.status(404).json({ error: "Not found" });
const postRes = await pool.query("SELECT * FROM posts WHERE author_id = $1", [user.id]);
// manual snake_case -> camelCase mapping, by hand, forever
res.json({ id: user.id, createdAt: user.created_at, posts: postRes.rows });
```

Nothing here is checked by anything. Rename `created_at` in the database and this file keeps compiling and starts returning `undefined`. Add a `NOT NULL` column and you find out when an `INSERT` fails at 2am.

| The old way | With Prisma |
|---|---|
| SQL lives in strings — typos surface at runtime, or never | Queries are method calls — a wrong field name is a **TypeScript error before you save** |
| Result rows are `any` — you guess the shape | Result type is derived from the query you wrote, including nested relations and `select` |
| Schema changes are ad-hoc `ALTER TABLE` runs someone did once | Schema changes are **migration files committed to git**, applied in the same order everywhere |
| Every developer's local database drifts from every other one | `prisma migrate dev` replays the same history — everyone converges |
| Relations mean hand-writing JOINs or firing N+1 queries | `include` and `select` fetch relations in a bounded number of queries |
| Renaming a column means grepping for a string across the repo | Renaming a field breaks compilation everywhere it is used |

Prisma's real trick is that the schema is not documentation *about* your database — it is the thing your database and your code are both **generated from**, so they cannot silently disagree.

---

## 3. Installing & Basic Usage

```bash
npm install prisma --save-dev          # the CLI: migrations, generate, studio
npm install @prisma/client             # the runtime client your app imports
npx prisma init --datasource-provider postgresql
# creates prisma/schema.prisma and adds a DATABASE_URL placeholder to .env
```

Put a real connection string in `.env` — the Prisma CLI reads this file automatically:

```bash
DATABASE_URL="postgresql://user:password@localhost:5432/myapp?schema=public"
```

`prisma init` already wrote the `datasource` and `generator` blocks (see §4), so add the smallest possible model under them in `prisma/schema.prisma`, then run `npx prisma migrate dev --name init` to create the table and generate the client:

```prisma
model User {
  id    String  @id @default(uuid())
  email String  @unique
  name  String? // ? means nullable
}
```

```js
// index.js  (ESM — "type": "module" in package.json)
import { PrismaClient } from "@prisma/client";
const prisma = new PrismaClient(); // creates a client that lazily opens a connection pool
// `data` is checked against your model — a misspelled field fails to compile, not at 2am
const user = await prisma.user.create({ data: { email: "ada@example.com", name: "Ada" } });
// a typed filter object, not a SQL string
const all = await prisma.user.findMany({ where: { email: { contains: "@example.com" } } });
console.log(user, all);
await prisma.$disconnect(); // scripts must disconnect or the process hangs on the open pool
```

### CommonJS version

```js
const { PrismaClient } = require("@prisma/client");
const prisma = new PrismaClient(); // identical API — only the import line differs
```

### Express example

```js
import express from "express";
import { PrismaClient } from "@prisma/client";

const app = express();
const prisma = new PrismaClient(); // ONE instance for the whole process — see §8
app.use(express.json());

app.get("/users/:id", async (req, res) => {
  const user = await prisma.user.findUnique({
    where: { id: req.params.id },      // findUnique only accepts @id / @unique fields
    select: { id: true, email: true }, // never ship columns the client does not need
  });
  if (!user) return res.status(404).json({ error: "Not found" }); // returns null, does not throw
  res.json(user);
});

app.listen(3000);
```

That's the entire mental model — edit `schema.prisma`, run a migrate command so the database and the generated client both catch up, then call typed methods on `prisma.<model>`. Everything below is detail on those three moves, plus one rule that belongs in every handler above: validate `req.body` with [[zod]] before it reaches `create`, because Prisma checks types, not business rules.

---

## 4. schema.prisma — The Single Source of Truth

Every Prisma project has exactly one schema file with three kinds of block: **datasource** (where the database is), **generator** (what code to emit), and **models** (your tables).

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

enum Role {
  USER
  ADMIN
}

model User {
  id        String   @id @default(uuid()) // primary key; uuid() is generated by Prisma, not the DB
  email     String   @unique              // UNIQUE index, and makes findUnique({ where: { email } }) legal
  name      String?                       // optional -> nullable column -> `string | null` in TS
  role      Role     @default(USER)       // enums become a real Postgres enum type
  updatedAt DateTime @updatedAt           // Prisma rewrites this on every update() for you
  posts     Post[]                        // virtual field — no column, this is the relation
  @@map("users")                          // table is `users` in SQL, but `prisma.user` in code
}

model Post {
  id        String   @id @default(uuid())
  title     String   @db.VarChar(200) // native type: VARCHAR(200) instead of Postgres TEXT
  body      String
  published Boolean  @default(false)
  authorId  String                    // the actual foreign-key column
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  tags      Tag[]                     // implicit many-to-many — Prisma manages the join table
  createdAt DateTime @default(now())
  @@index([authorId])                 // foreign keys are NOT auto-indexed in Postgres
  @@index([published, createdAt])     // composite index for "latest published posts" queries
}

model Tag {
  id    String @id @default(uuid())
  name  String @unique
  posts Post[] // the other half of the implicit many-to-many
}
```

Things worth internalizing:

- **A relation always has two halves.** `Post.author` (the side carrying `@relation(fields: ...)`) owns the foreign key; `User.posts` is the mirror. Write only one and Prisma refuses to validate the schema.
- **`onDelete: Cascade`** is enforced by the database, not your code — deleting a user really does delete their posts. The alternatives are `Restrict` (block the delete), `SetNull` (requires an optional relation), `SetDefault`, and `NoAction`.
- **Indexes are your job.** Prisma indexes `@id` and `@unique` automatically and nothing else. Every field you regularly filter or sort on wants an `@@index`, and `@@map` / `@map` let the SQL names stay `snake_case` while your code stays `camelCase`.
- **The generator block matters.** `prisma-client-js` emits into `node_modules/.prisma/client`, which is why you import from `@prisma/client`. Recent Prisma versions warn that an explicit `output` path is becoming required and ship a newer `prisma-client` generator that writes real files into your source tree; if you see that warning, add `output = "../src/generated/prisma"` and import from that path instead. The query API is identical either way.

**Rule of thumb:** if a fact about your data is not in `schema.prisma`, it does not exist. Never `ALTER TABLE` by hand — change the schema and let a migration carry it.

---

## 5. Migrations — Getting the Schema Into a Real Database

This is where beginners do real damage, so learn the three commands as three different tools, not three spellings of the same thing.

```mermaid
%%{init: {'themeVariables': { 'primaryTextColor': '#000000', 'textColor': '#000000', 'lineColor': '#000000', 'primaryBorderColor': '#000000'}}}%%
flowchart TD
    Q{"Where are you<br/>running this?"}
    Q -->|"local dev"| A["prisma migrate dev<br/>writes new SQL file<br/>applies it<br/>regenerates client"]
    Q -->|"CI or production"| B["prisma migrate deploy<br/>applies pending files only<br/>never prompts<br/>never resets"]
    Q -->|"throwaway spike"| C["prisma db push<br/>no migration file<br/>can drop data"]

    style Q fill:#fff2cc,stroke:#000000,color:#000000
    style A fill:#e0f0ff,stroke:#000000,color:#000000
    style B fill:#e0ffe0,stroke:#000000,color:#000000
    style C fill:#ffe0e0,stroke:#000000,color:#000000
```

```bash
# DEVELOPMENT — after every schema edit. Diffs the schema, writes
# prisma/migrations/20260818120000_add_post_tags/migration.sql, applies it, regenerates the client
npx prisma migrate dev --name add_post_tags

# PRODUCTION / CI — the only migrate command allowed near real data.
# Applies migration files that have not run yet. No prompts, no diffing, no reset.
npx prisma migrate deploy
npx prisma db push          # PROTOTYPING ONLY — no history, no migration file
npx prisma migrate status   # what has run, what is pending, has the DB drifted
npx prisma generate         # regenerate the client without touching the database
npx prisma migrate reset    # DROP everything, replay all migrations, run the seed
```

> ⚠️ **Never run `prisma migrate dev` against a production database.** It compares the live schema to your migration history, and if it finds drift — a column someone added by hand, a migration edited after it was applied — its remedy is to **reset the database**: drop every table and replay from scratch. It prompts first, but the prompt is easy to click through, and CI has no human to click. Production uses `migrate deploy`, which has no code path that drops data.

**The `prisma/migrations/` folder is committed to git.** It is your schema's history: an ordered list of `.sql` files any environment can replay to arrive at exactly the schema you have. Treat those files as immutable once they have run anywhere — editing an already-applied migration is what causes the drift that makes `migrate dev` want to reset. Made a mistake? Write a *new* migration that fixes it.

**When the auto-generated SQL is not good enough**, add `--create-only`: `npx prisma migrate dev --name rename_bio --create-only` writes the SQL file without applying it, so you can edit `migration.sql` and then run `npx prisma migrate dev` to apply your version. This is how you rename without data loss — Prisma's diff sees "column `bio` dropped, column `about` added" and generates `DROP` plus `ADD`, so you replace that with `ALTER TABLE "users" RENAME COLUMN "bio" TO "about";`.

---

## 6. The Generated Client — CRUD, select vs include, Nested Writes

Every model becomes `prisma.<camelCaseModel>` with the same method set.

```js
const user = await prisma.user.create({ data: { email: "a@b.com", name: "Ada" } });
// READ one — findUnique takes @id/@unique fields only and returns null when absent;
// findUniqueOrThrow throws P2025 instead, and findFirst is the one that accepts any filter
const byId = await prisma.user.findUnique({ where: { id } });
const byName = await prisma.user.findFirst({ where: { name: "Ada" } });
// READ many — filters compose as objects, they never concatenate into a string
const posts = await prisma.post.findMany({
  where: {
    published: true,
    title: { contains: "prisma", mode: "insensitive" }, // mode is PostgreSQL/MongoDB only
    createdAt: { gte: new Date("2026-01-01") },
    author: { role: "ADMIN" }, // filtering on a RELATION's field — becomes a JOIN
    OR: [{ body: { contains: "sql" } }, { tags: { some: { name: "database" } } }],
  },
});
// WRITE — update/delete throw P2025 when the row is missing; the *Many variants return { count }
await prisma.post.update({ where: { id }, data: { published: true } });
await prisma.post.deleteMany({ where: { published: false } });
// UPSERT — insert or update in one round trip, so there is no read-then-write race
await prisma.user.upsert({
  where: { email: "a@b.com" },
  create: { email: "a@b.com", name: "Ada" },
  update: { name: "Ada L." },
});
```

### select vs include — pick one, and prefer `select`

Both pull in related data. They differ in what they do to the *rest* of the row.

```js
// ❌ include: every scalar column of Post PLUS every column of User — passwordHash included
const withInclude = await prisma.post.findMany({ include: { author: true } });
// ✅ select: only what you name, nothing else crosses the wire. Relations nest select too.
const withSelect = await prisma.post.findMany({
  select: { id: true, title: true, author: { select: { id: true, name: true } } },
});
```

`include` is convenient and quietly expensive: it selects every column of both tables, which means big `TEXT` bodies you never render and — the dangerous part — fields like `passwordHash` that end up in `res.json()` because nobody re-read the handler. `select` makes the response shape explicit, shrinks the SQL, and gives you a TypeScript type with exactly those keys. You cannot use both at the same level; Prisma throws a validation error. **Rule of thumb:** `select` in anything that returns to a client, `include` only in scripts where you genuinely want the whole row.

### Nested writes — one call, one transaction

```js
// Create a user AND their post AND link a tag, atomically, in one call
const author = await prisma.user.create({
  data: {
    email: "grace@example.com",
    posts: {
      create: {
        title: "Hello",
        body: "First post",
        // reuse the tag row if it already exists, otherwise create it
        tags: { connectOrCreate: { where: { name: "intro" }, create: { name: "intro" } } },
      },
    },
  },
  select: { id: true, posts: { select: { id: true, title: true } } },
});
```

A nested write is **implicitly a transaction** — if the post violates a constraint, the user is not created either. The relation verbs you will use: `create` (new related row), `connect` (link an existing row by unique field), `connectOrCreate`, `disconnect`, `set` (replace the whole list), `update`, `delete`.

### Pagination — offset vs cursor

```js
// Offset — simple, fine for small tables and "jump to page 7" admin UIs
await prisma.post.findMany({ orderBy: { createdAt: "desc" }, skip: 40, take: 20 });
// Cursor — constant cost at any depth, correct under concurrent inserts
await prisma.post.findMany({
  take: 20,
  cursor: { id: lastSeenId }, // must be a unique field
  skip: 1,                    // skip the cursor row itself, or you return it twice
  orderBy: { id: "asc" },     // cursor pagination REQUIRES a stable total ordering
});
```

Offset pagination degrades linearly — `skip: 100000` makes Postgres scan and discard 100,000 rows — and it *skips or repeats* items when rows are inserted while a user pages. Cursor pagination says "give me 20 after this exact row," which is one index seek regardless of depth. Use offset for admin tables, cursor for feeds and infinite scroll.

---

## 7. Relations, Transactions and Performance

### The N+1 problem

This is the single biggest performance mistake with any ORM:

```js
// ❌ 1 query for the users, then 1 more query PER user = 101 round trips
const users = await prisma.user.findMany({ take: 100 });
for (const u of users) {
  u.posts = await prisma.post.findMany({ where: { authorId: u.id } });
}
// ✅ one call, bounded query count — Prisma loads the relation for you
const withPosts = await prisma.user.findMany({
  take: 100,
  select: { id: true, name: true, posts: { select: { id: true, title: true } } },
});
```

```mermaid
%%{init: {'themeVariables': { 'primaryTextColor': '#000000', 'textColor': '#000000', 'lineColor': '#000000', 'primaryBorderColor': '#000000'}}}%%
flowchart LR
    A["findMany users<br/>then loop per user"] -->|"1 plus N"| B["101 round trips<br/>latency paid 101 times"]
    C["findMany users<br/>with nested select"] -->|"bounded"| D["A couple of queries<br/>one network wait"]

    style A fill:#ffe0e0,stroke:#000000,color:#000000
    style B fill:#ffe0e0,stroke:#000000,color:#000000
    style C fill:#e0f0ff,stroke:#000000,color:#000000
    style D fill:#e0ffe0,stroke:#000000,color:#000000
```

The loop version is not slow because the queries are slow — each is sub-millisecond. It is slow because you pay network latency 101 times, serially, and a nested `select` or `include` collapses that into one wait. Turn on `log: ["query"]` in development and watch the query count for any endpoint that feels sluggish; N+1 is instantly visible as the same `SELECT` repeated with different parameters.

An *implicit* many-to-many (`tags Tag[]` on both sides, as in §4) makes Prisma create and manage a hidden `_PostToTag` join table. That is perfect until you need a column *on the relationship itself* — who added the tag, when, a quantity. Then you declare the join model yourself with `@@id([postId, tagId])` as its composite primary key and a `@relation` to each side. Converting implicit to explicit later is a real migration with data movement, so if you suspect you will ever want metadata on the link, start explicit.

### Transactions — two shapes

```js
// 1. Array form — independent operations, run in order, all-or-nothing
const [userCount, postCount] = await prisma.$transaction([prisma.user.count(), prisma.post.count()]);
// 2. Interactive form — when a later step depends on an earlier result
await prisma.$transaction(
  async (tx) => {
    // atomic DB-side decrement, not a read-modify-write you could lose a race on
    const from = await tx.account.update({
      where: { id: fromId },
      data: { balance: { decrement: 100 } },
    });
    if (from.balance < 0) throw new Error("Insufficient funds"); // throwing rolls everything back
    await tx.account.update({ where: { id: toId }, data: { balance: { increment: 100 } } });
  },
  { timeout: 10000, isolationLevel: "Serializable" } // default timeout is 5s — raise it deliberately
);
```

Inside the callback you **must** use `tx`, not `prisma`. A stray `prisma.something()` in there runs on a different connection, outside the transaction, and will not roll back. Keep interactive transactions short: they hold a connection from the pool for their whole duration, so an HTTP call or a slow loop inside one starves every other request.

### Connection pooling

`new PrismaClient()` manages a **pool**, not a single connection. The default size is `num_physical_cpus * 2 + 1`, and you tune it in the connection string:

```bash
DATABASE_URL="postgresql://user:pass@host:5432/db?connection_limit=10&pool_timeout=20"
```

The arithmetic that bites people: 4 app instances times 10 connections is 40 connections, and a small managed Postgres often caps out around 100 total. Exceed it and you get `too many clients already` — not a Prisma error, a database error. Serverless makes this much worse, because every function instance is its own process with its own pool, hundreds can exist concurrently, and each holds connections open through the freeze/thaw cycle. A long-running Express server on a VM is fine with a plain pool; serverless needs an external pooler in front — **Prisma Accelerate**, or **PgBouncer** in transaction mode (add `?pgbouncer=true` to the URL so Prisma stops using prepared statements that transaction-mode pooling cannot support).

---

## 8. Client Lifecycle, Studio, Seeding and Raw SQL

`PrismaClient` is designed to be instantiated **once** and shared. The classic beginner bug is instantiating it per request, which opens a new pool per request and exhausts the database in minutes. The subtler bug is dev-only: [[nodemon]] and other hot-reload tools re-execute your module on every save, and each reload builds *another* client while the old pools are still draining.

```js
// src/db.js — import this everywhere instead of calling new PrismaClient() again
import { PrismaClient } from "@prisma/client";

const g = globalThis; // globalThis survives module re-evaluation on hot reload

export const prisma =
  g.prisma ?? // reuse the client from the previous reload if there is one
  new PrismaClient({
    log: process.env.NODE_ENV === "development" ? ["query", "warn", "error"] : ["error"],
  });
// Cache in dev only — in production the module is evaluated exactly once, so this buys nothing
if (process.env.NODE_ENV !== "production") g.prisma = prisma;
```

### Studio and seeding

`npx prisma studio` opens a browser table editor at `http://localhost:5555` that understands your relations — click a post, see its author, edit a row. It connects with the same full-privilege `DATABASE_URL` your app uses, so treat it as a local dev tool and never point it at production from a laptop.

```js
// prisma/seed.js — deterministic starting data for a fresh database (ESM, top-level await)
import { PrismaClient } from "@prisma/client";
const prisma = new PrismaClient();
// upsert, not create, so re-running the seed is safe and idempotent
await prisma.user.upsert({
  where: { email: "admin@example.com" },
  update: {},
  create: { email: "admin@example.com", name: "Admin", role: "ADMIN" },
});
await prisma.$disconnect();
```

Point Prisma at it with a `prisma.seed` key in `package.json` — `{ "prisma": { "seed": "node prisma/seed.js" } }` — then run `npx prisma db seed`. `prisma migrate reset` runs the seed automatically after replaying migrations, which is what makes "blow away my local DB and start clean" a one-command operation.

### Raw SQL, safely

For the rare query Prisma's API cannot express — a window function, a recursive CTE, a database-specific operator:

```js
// ✅ SAFE — tagged template. Every ${...} becomes a real bound parameter, never string concat
const rows = await prisma.$queryRaw`SELECT id, email FROM users WHERE email = ${email}`;
// $executeRaw for statements that return a row count instead of rows
const n = await prisma.$executeRaw`UPDATE posts SET published = true WHERE author_id = ${id}`;
// ❌ UNSAFE — JavaScript builds the string before Prisma ever sees it.
// email = "x' OR '1'='1" now returns every user in the table
await prisma.$queryRawUnsafe(`SELECT * FROM users WHERE email = '${email}'`);
```

The distinction is the whole point: `$queryRaw` as a **tagged template** is parameterized and injection-safe, because the values never become part of the SQL string. `$queryRawUnsafe` takes an already-assembled string and has "unsafe" in its name for a reason — reach for it only when the *structure* of the query is dynamic, such as a column name chosen at runtime, and then validate that fragment against an allowlist you wrote by hand. Raw results are also **not** typed or mapped for you: `BIGINT` columns and count aggregates come back as `BigInt`, and column names are the SQL ones, not your Prisma field names.

---

## 9. TypeScript Version

Prisma is a TypeScript tool that happens to work in JavaScript — this is where it pays off.

```ts
// src/routes/posts.ts
import { Router, type Request, type Response, type NextFunction } from "express";
import { Prisma, type Post } from "@prisma/client";
import { prisma } from "../db.js";
const router = Router();
// Prisma generates a type for the exact shape of any query — no hand-written interface,
// and it breaks at compile time the moment the select below changes.
type PostCard = Prisma.PostGetPayload<{
  select: { id: true; title: true; author: { select: { name: true } } };
}>;

router.get("/", async (req: Request, res: Response<PostCard[]>, next: NextFunction) => {
  try {
    const posts = await prisma.post.findMany({
      where: { published: true },
      select: { id: true, title: true, author: { select: { name: true } } },
      orderBy: { createdAt: "desc" },
      take: 20,
    });
    res.json(posts); // TS verifies this really matches PostCard[]
  } catch (err) {
    next(err);
  }
});

router.post("/", async (req: Request, res: Response, next: NextFunction) => {
  // Prisma.PostCreateInput is generated — required fields stay required, and it changes
  // shape automatically the next time you edit schema.prisma
  const data: Prisma.PostCreateInput = {
    title: req.body.title,
    body: req.body.body,
    author: { connect: { id: req.body.authorId } }, // relations connect, not raw ids
  };
  try {
    const post: Post = await prisma.post.create({ data });
    res.status(201).json(post);
  } catch (err) {
    // Known request errors carry a stable code — map them to real HTTP statuses
    if (err instanceof Prisma.PrismaClientKnownRequestError) {
      if (err.code === "P2002") return res.status(409).json({ error: "Already exists" });
      if (err.code === "P2025") return res.status(404).json({ error: "Author not found" });
    }
    next(err);
  }
});
export default router;
```

The codes worth memorizing: **P2002** unique constraint violated, **P2025** record not found, **P2003** foreign key constraint failed, **P2028** transaction API error (usually a timeout). Handle them in your [[express]] error middleware so a duplicate email returns `409` instead of a `500` stack trace.

---

## 10. Production Setup

```jsonc
// package.json — Docker layers and CI caches have no node_modules/.prisma until postinstall runs
{
  "scripts": {
    "postinstall": "prisma generate",
    "start": "prisma migrate deploy && node dist/server.js" // migrate BEFORE serving traffic
  }
}
```

```prisma
generator client {
  provider      = "prisma-client-js"
  // Alpine and Debian containers need the matching engine binary compiled in. A wrong target
  // means "Query engine binary could not be found" at runtime, and only inside Docker.
  binaryTargets = ["native", "linux-musl-openssl-3.0.x"]
}
```

- **Shut down gracefully.** On `SIGTERM`, stop accepting requests with `server.close()` and then `await prisma.$disconnect()` so in-flight queries finish and pooled connections are released before the container is SIGKILLed.
- **`DATABASE_URL` comes from the platform's secret store, never a committed `.env`.** Note the asymmetry: the Prisma **CLI** loads `.env` automatically, but your **running app does not** — you need [[dotenv]], `node --env-file=.env` (Node 20.6+), or real platform env vars. "Works with `npx prisma studio`, crashes on `node server.js`" is almost always this.
- **Migrations run as a release step**, not inside your app's startup on every replica. Ten replicas booting at once and all running `migrate deploy` is safe — Prisma takes an advisory lock — but slow and noisy; a single release command is cleaner.
- **Two database users if you can**: a migration user with DDL rights used only by `migrate deploy`, and a runtime user that can only `SELECT/INSERT/UPDATE/DELETE`. Then a compromised app cannot `DROP TABLE`.
- **Size the pool deliberately** — pick `?connection_limit=N` so that replicas times N stays under the database's `max_connections`, leaving headroom for migrations and your own `psql` session. Log queries in dev and errors only in prod: `log: ["query"]` in production is a firehose that leaks parameter values into your log aggregator.
- **Commit `prisma/migrations/` and `schema.prisma`; never commit the generated client** — it is build output. Run `prisma migrate status` in CI to catch a schema change that shipped without a migration file.

---

## 11. Common Gotchas & Best Practices

| Gotcha | Fix |
|---|---|
| **`prisma migrate dev` run against production** | It diffs against migration history and offers to **reset** — drop all tables — when it detects drift. Production and CI use `prisma migrate deploy`, which only applies pending files and has no reset path. Keep prod credentials out of your local shell so the accident is impossible. |
| **`@prisma/client did not initialize yet` in Docker or CI** | The generated client lives in `node_modules` and is wiped by `npm ci` or a fresh Docker layer. Add `"postinstall": "prisma generate"`, and make sure `prisma/schema.prisma` is copied into the image *before* `npm ci` runs. |
| **`too many clients already` after a few minutes** | You are calling `new PrismaClient()` per request or per hot reload. Export one instance from a `db.js` module and use the `globalThis` singleton from §8 so [[nodemon]] restarts reuse it. |
| **`update()` throws instead of returning null** | `update` and `delete` on a missing row throw `P2025` — they are not no-ops. Either catch `P2025`, or use `updateMany`/`deleteMany` (which return `{ count: 0 }`), or `upsert` when "create if absent" is what you meant. |
| **Duplicate email returns a 500** | Prisma surfaces the unique-constraint violation as `P2002`, and an uncaught error becomes a 500 with a stack trace. Catch `Prisma.PrismaClientKnownRequestError`, check `err.code`, and map `P2002` to `409`. |
| **`include` leaks `passwordHash` into the API response** | `include: { author: true }` selects **every** column of the related row. Use `select` with an explicit field list on anything that reaches a client — it is shorter SQL and a smaller blast radius. |
| **`TypeError: Do not know how to serialize a BigInt`** | Postgres `BIGINT` columns and count aggregates in raw queries come back as JS `BigInt`, which `JSON.stringify` refuses. Convert with `Number(value)` where the range is safe, or `.toString()` for real 64-bit ids. `Decimal` fields need `.toString()` too. |
| **`P2028: Transaction already closed`** | Interactive `$transaction` callbacks default to a 5-second timeout and hold a pooled connection the whole time. Move HTTP calls, image processing and email sending *outside* the transaction; raise `{ timeout: 15000 }` only for genuinely long database work. |

---

## 12. Alternatives — When Prisma Isn't the Best Fit

| Tool | What it is | Best for | The catch |
|---|---|---|---|
| **Prisma** | Schema-first ORM: own schema language, generated client, own migration system | Teams that want types, migrations and relation queries handled for them with the least ceremony | Its own DSL rather than TypeScript; a heavier runtime than a query builder; serverless needs a pooler |
| **Drizzle** | Schema defined *in TypeScript*, SQL-shaped query builder, generated types | You know SQL and want it to look like SQL, with a tiny runtime and no engine binary — very serverless-friendly | Younger ecosystem; relation queries are more manual; migration tooling is less opinionated |
| **Kysely** | Pure type-safe query builder, no ORM layer, no migrations of its own | Complex analytical SQL where you want full control and still want the compiler checking column names | You write every JOIN yourself; types must be hand-written or generated from the DB; no relation loading |
| **TypeORM** | Traditional decorator and class-based ORM with Active Record and Data Mapper modes | Java or .NET-style entity code, or an existing codebase already using it | Weaker type inference, historically shaky migrations, a lot of decorator magic |
| **raw `pg`** | The Postgres driver, nothing more | Scripts, one-off jobs, or a service with three queries total | No types, no migrations, manual mapping — the §2 problem, in full |
| **[[mongoose]]** | ODM for MongoDB — a document database, not SQL at all | Flexible nested documents, schema that varies per record, no JOINs needed | A different data model entirely; not a Prisma competitor so much as a different answer to "what database" |

```mermaid
%%{init: {'themeVariables': { 'primaryTextColor': '#000000', 'textColor': '#000000', 'lineColor': '#000000', 'primaryBorderColor': '#000000'}}}%%
flowchart TD
    Q1{"Relational data<br/>with real joins?"}
    Q1 -->|"no, nested documents"| M["Use MongoDB<br/>with mongoose"]
    Q1 -->|"yes, SQL"| Q2{"How much SQL<br/>do you want to write?"}
    Q2 -->|"as little as possible"| P["Prisma"]
    Q2 -->|"I like SQL<br/>and want it lean"| D["Drizzle"]
    Q2 -->|"heavy custom SQL<br/>no ORM layer"| K["Kysely"]
    Q2 -->|"three queries<br/>in a small script"| PG["raw pg driver"]

    style Q1 fill:#fff2cc,stroke:#000000,color:#000000
    style Q2 fill:#fff2cc,stroke:#000000,color:#000000
    style P fill:#e0ffe0,stroke:#000000,color:#000000
    style D fill:#e0f0ff,stroke:#000000,color:#000000
    style K fill:#e0f0ff,stroke:#000000,color:#000000
    style M fill:#ffffff,stroke:#000000,color:#000000
    style PG fill:#ffe0e0,stroke:#000000,color:#000000
```

**Rule of thumb:** the first question is never "Prisma or Mongoose?" — it is "does my data have relationships I will need to join across?" Orders, users, invoices and permissions, anything where the same fact must not be duplicated in two documents, means relational, and then Prisma is the friendliest way in. Event logs, CMS content blocks and per-record variable shapes point to MongoDB and [[mongoose]]. Choosing the ORM before choosing the data model is how projects end up modeling foreign keys by hand inside a document store.

---

## 13. Interview Questions

**Q: What is the difference between `prisma migrate dev` and `prisma migrate deploy`?**
A: `migrate dev` is a development command — it diffs `schema.prisma` against the database, generates a new migration SQL file, applies it, and regenerates the client. Crucially it can decide the database needs a **reset**, dropping everything and replaying, if it detects drift. `migrate deploy` only applies migration files that have not run yet: no diffing, no prompts, no reset, no client generation. Production and CI use `deploy` exclusively.

**Q: Why is `select` usually better than `include`?**
A: `include` returns every scalar column of both the parent and the related rows, so you fetch data you never render and risk leaking sensitive fields like `passwordHash` straight into a JSON response. `select` names exactly the fields you want, producing narrower SQL, a smaller payload, and a TypeScript type with only those keys. That last part matters most: adding a sensitive column to a model cannot silently widen an existing API response.

**Q: What is the N+1 problem and how does Prisma help?**
A: N+1 is fetching a list with one query and then firing one additional query per item to load its relation — 100 users becomes 101 round trips, each paying full network latency. Prisma solves it by letting you request the relation as part of the original query with `include` or a nested `select`, which it resolves in a bounded number of queries instead of one per row. Enabling `log: ["query"]` in development makes N+1 obvious, because you see the same statement repeated with different parameters.

**Q: When would you use the interactive form of `$transaction` instead of the array form?**
A: The array form runs a fixed list of independent operations atomically and is right when nothing depends on an earlier result. The interactive form takes an async callback and hands you a `tx` client, so you can read a row, branch on its value, and then write — a balance transfer, or "only insert if the count is still under the limit." You must use `tx` inside the callback rather than the outer `prisma`, and keep the work short, because the callback holds a pooled connection and times out after five seconds by default.

**Q: Is `$queryRaw` vulnerable to SQL injection?**
A: Not when used as a tagged template. Every interpolation is compiled into a real bound parameter, so the value never becomes part of the SQL string the database parses. `$queryRawUnsafe` takes a pre-assembled string and is genuinely unsafe — it should only appear when the query's *structure* is dynamic, with any interpolated identifier validated against a hard-coded allowlist first.

**Q: Why do serverless deployments need Prisma Accelerate or PgBouncer?**
A: Each `PrismaClient` owns a connection pool, and in serverless every concurrent function instance is a separate process with its own pool that stays alive across freeze and thaw. A few hundred concurrent invocations can therefore open far more connections than a managed Postgres allows, producing `too many clients already`. An external pooler multiplexes many short-lived clients onto a small set of real database connections; with PgBouncer in transaction mode you also add `?pgbouncer=true` so Prisma stops relying on prepared statements.

**Q: Should the `prisma/migrations` folder be committed to git?**
A: Yes — it *is* your schema history, and it is what lets any environment replay the exact same sequence of changes and end up with an identical database. Once a migration has been applied anywhere you treat its SQL as immutable, because editing it causes the drift that makes `migrate dev` want to reset. Mistakes are corrected by writing a new migration on top, exactly like fixing a bad commit that has already been pushed.

---

## 14. Quick Cheat Sheet

```bash
npm install prisma --save-dev && npm install @prisma/client
npx prisma init --datasource-provider postgresql

npx prisma migrate dev --name add_something   # dev: create + apply + regenerate
npx prisma generate                           # regenerate client only
npx prisma studio                             # GUI on localhost:5555
npx prisma db seed                            # run prisma/seed.js
npx prisma migrate deploy                     # production: apply pending, never resets
npx prisma migrate status                     # what is pending, has it drifted
```

```js
await prisma.user.create({ data: { email } });
await prisma.user.findUnique({ where: { id } });             // unique fields only, returns null
await prisma.user.findFirst({ where: { name: "Ada" } });     // non-unique filters
await prisma.user.findMany({ where: { role: "ADMIN" }, select: { id: true, email: true } });
await prisma.user.update({ where: { id }, data: { name } }); // throws P2025 if missing
await prisma.user.upsert({ where: { email }, create: { email }, update: { name } });
await prisma.user.delete({ where: { id } });
// Relation query + cursor pagination
await prisma.post.findMany({
  select: { id: true, author: { select: { name: true } } },
  orderBy: { id: "asc" },
  take: 20,
  cursor: { id: lastId },
  skip: 1,
});
// Transaction, and injection-safe raw SQL
await prisma.$transaction(async (tx) => {
  await tx.account.update({ where: { id }, data: { balance: { decrement: 100 } } });
}, { timeout: 10000 });
await prisma.$queryRaw`SELECT id FROM users WHERE email = ${email}`;
```

**Mental model to remember:**
> Prisma = one `schema.prisma` file that generates *both* your database tables and a fully typed client, so your code and your schema physically cannot disagree. Edit the schema, run `migrate dev` locally and `migrate deploy` in production, then query with `select` (not `include`) through a single shared client instance. Pair it with [[express]] for routing, [[zod]] to validate input before it reaches `create`, [[dotenv]] for `DATABASE_URL`, and reach for [[mongoose]] instead when your data is documents rather than rows.
