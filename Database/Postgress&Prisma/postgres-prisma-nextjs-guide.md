# PostgreSQL + Prisma + Next.js — Copy-Paste Setup Guide

A reusable setup you can drop into **any** Next.js project to connect PostgreSQL through Prisma ORM, plus the core concepts you need to actually understand what you're doing.

---

## 1. The Mental Model

Think of it like a restaurant:

- **Browser (frontend)** = the customer ordering food
- **Next.js API route** = the waiter taking the order
- **Prisma Client** = the waiter translating your order into kitchen language
- **PostgreSQL** = the kitchen, where the food (data) actually lives

```
Browser  →  Next.js API Route  →  Prisma Client  →  PostgreSQL
   ↑                                                     |
   └─────────────────── response flows back ─────────────┘
```

One file — `schema.prisma` — is the single source of truth. It defines your data shape, and both Prisma Client and your actual database are generated/synced from it.

---

## 2. PostgreSQL Basics (Explain Like I'm 5, then the real depth)

### What is PostgreSQL?
It's a filing cabinet that never forgets. You put data in labeled drawers (**tables**), each drawer has folders (**rows**), and each folder has labeled tabs (**columns**).

| Concept | ELI5 | Real definition |
|---|---|---|
| Database | A whole filing cabinet | A container that holds all your tables |
| Table | One drawer | A collection of rows with the same structure |
| Row | One folder | A single record/entry |
| Column | A labeled tab on the folder | A field that every row has (name, price, etc.) |
| Primary Key | The folder's unique serial number | A column that uniquely identifies each row |
| Foreign Key | A sticky note saying "see drawer B, folder 7" | A column that points to a row in another table |

### The 4 commands you'll use 90% of the time (CRUD)

```sql
-- CREATE
INSERT INTO users (name, email) VALUES ('Naman', 'naman@example.com');

-- READ
SELECT * FROM users WHERE email = 'naman@example.com';

-- UPDATE
UPDATE users SET name = 'Naman Singla' WHERE id = 1;

-- DELETE
DELETE FROM users WHERE id = 1;
```

### JOIN — combining drawers
Imagine `users` and `posts` are two separate drawers, linked by a sticky note (`user_id` in `posts` pointing to `id` in `users`).

```sql
SELECT users.name, posts.title
FROM posts
JOIN users ON posts.user_id = users.id;
```
This says: "For every post, go find the matching user, and show me both."

### Indexes — the book's table of contents
**ELI5:** Without an index, finding a word in a book means reading every page. An index lets you jump straight to it.

**Real definition:** An index is a separate, sorted data structure PostgreSQL builds on a column so it doesn't have to scan every row (`sequential scan`) to find matches. It speeds up `SELECT ... WHERE` and `JOIN` massively, but slightly slows down `INSERT`/`UPDATE` (because the index has to update too).

```sql
CREATE INDEX idx_users_email ON users(email);
```

Rule of thumb: index columns you frequently `WHERE`, `JOIN`, or `ORDER BY` on — not every column.

### Transactions — the "all or nothing" promise
**ELI5:** Transferring money between two friends: take $10 from A, give $10 to B. If the system crashes after taking from A but before giving to B, that $10 vanishes. A transaction makes sure **both steps happen, or neither does.**

**Real definition:** A transaction groups multiple SQL statements so they succeed or fail together, following **ACID**:
- **A**tomicity — all steps happen, or none do
- **C**onsistency — the database stays in a valid state
- **I**solation — concurrent transactions don't corrupt each other
- **D**urability — once committed, it survives a crash

```sql
BEGIN;
UPDATE accounts SET balance = balance - 10 WHERE id = 1;
UPDATE accounts SET balance = balance + 10 WHERE id = 2;
COMMIT;  -- or ROLLBACK; if something went wrong
```

---

## 3. Prisma ORM Basics

### What is an ORM? (ELI5)
Normally you'd write raw SQL, like typing a very precise letter in a foreign language. An **ORM (Object-Relational Mapper)** is a translator — you write JavaScript/TypeScript, and it converts it into SQL for you, and converts the SQL results back into JS objects.

### Why Prisma specifically?
- **Type-safe**: autocomplete for your database, and TypeScript errors if you misspell a field
- **schema.prisma**: one readable file defines your entire data model
- **Migrations**: tracks every change to your database structure over time, like Git for your schema

### The 3 core pieces

1. **`schema.prisma`** — defines your models (tables) in a simple syntax
2. **Prisma Migrate** — turns your schema into actual SQL tables in PostgreSQL
3. **Prisma Client** — the auto-generated, type-safe JS/TS library you import and query with

```
schema.prisma  --(migrate)-->  PostgreSQL tables
schema.prisma  --(generate)-->  Prisma Client (typed JS functions)
```

---

## 4. Full Copy-Paste Setup (Next.js + Prisma + PostgreSQL)

### Step 1 — Install packages
```bash
npm install prisma @prisma/client
npx prisma init
```
This creates a `prisma/schema.prisma` file and a `.env` file.

### Step 2 — Set your database connection
Get a free Postgres URL from **Neon**, **Supabase**, or **Railway** (all have a free tier and give you a connection string instantly — no local install needed).

`.env`
```
DATABASE_URL="postgresql://user:password@host:5432/dbname?sslmode=require"
```

### Step 3 — Define your schema
`prisma/schema.prisma`
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  posts     Post[]
  createdAt DateTime @default(now())
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  author    User     @relation(fields: [authorId], references: [id])
  authorId  Int
  createdAt DateTime @default(now())
}
```

### Step 4 — Push schema to your database
```bash
npx prisma migrate dev --name init
```
This creates real tables in PostgreSQL matching your schema, and generates Prisma Client.

### Step 5 — Create a reusable Prisma Client (avoids connection leaks in dev)
`lib/prisma.ts`
```ts
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const prisma = globalForPrisma.prisma || new PrismaClient();

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

### Step 6 — Use it in a Next.js API route
`app/api/users/route.ts`
```ts
import { prisma } from "@/lib/prisma";
import { NextResponse } from "next/server";

// GET /api/users
export async function GET() {
  const users = await prisma.user.findMany();
  return NextResponse.json(users);
}

// POST /api/users
export async function POST(req: Request) {
  const body = await req.json();
  const user = await prisma.user.create({
    data: { name: body.name, email: body.email },
  });
  return NextResponse.json(user, { status: 201 });
}
```

### That's it — you're connected.
Every time you change `schema.prisma`, run:
```bash
npx prisma migrate dev --name describe_your_change
```

---

## 5. Prisma Query Cheatsheet (must-know)

```ts
// Find all
await prisma.user.findMany();

// Find one
await prisma.user.findUnique({ where: { id: 1 } });

// Find with a filter
await prisma.user.findMany({ where: { email: { contains: "@gmail.com" } } });

// Create
await prisma.user.create({ data: { name: "Naman", email: "n@x.com" } });

// Update
await prisma.user.update({ where: { id: 1 }, data: { name: "New Name" } });

// Delete
await prisma.user.delete({ where: { id: 1 } });

// Include related data (like a JOIN)
await prisma.user.findMany({ include: { posts: true } });

// Transaction (all-or-nothing, like SQL BEGIN/COMMIT)
await prisma.$transaction([
  prisma.user.update({ where: { id: 1 }, data: { name: "A" } }),
  prisma.post.create({ data: { title: "New", authorId: 1 } }),
]);
```

---

## 6. Common Pitfalls to Avoid

- **Forgetting `npx prisma generate`** after pulling code with schema changes — Prisma Client goes out of sync.
- **Creating a `new PrismaClient()` on every request** — exhausts your database's connection limit. Always use the singleton pattern from Step 5.
- **No index on frequently filtered columns** — queries get slow as data grows.
- **Skipping transactions for multi-step writes** — leaves your data half-updated if something fails midway.
- **Committing `.env` to Git** — always add it to `.gitignore`.

---

## 7. Quick Reference: Must-Know Before You're "Comfortable"

**PostgreSQL:**
- SELECT, INSERT, UPDATE, DELETE, WHERE, JOIN
- Primary key vs foreign key
- Indexes and when to add them
- Transactions (BEGIN / COMMIT / ROLLBACK)

**Prisma:**
- schema.prisma syntax (models, fields, relations)
- `npx prisma migrate dev` vs `npx prisma generate`
- Prisma Client CRUD methods
- `include` for relations, `$transaction` for atomic operations

---

## 8. Resources
- Prisma docs: https://www.prisma.io/docs
- PostgreSQL official docs: https://www.postgresql.org/docs
- Free Postgres hosting: https://neon.tech, https://supabase.com, https://railway.app
