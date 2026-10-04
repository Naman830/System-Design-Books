# jest + supertest — Proving Your API Works Without Opening Postman

> **Scope:** Automated testing for a Node 20+ / Express / MongoDB API. Jest as the test runner and assertion library, supertest for driving HTTP routes in-process, mocking, and `mongodb-memory-server` for a throwaway test database.
> **Level:** Beginner + practical.
> **New to Express?** Read [[express]] first — everything here assumes you already have routes and middleware you understand.

---

## 1. ELI5: What is jest + supertest?

You built `POST /api/users`. You opened Postman, typed a JSON body, hit Send, saw `201 Created`, and felt good about yourself. Three weeks later you rename a field in your [[mongoose]] schema, tweak a validation rule, and deploy. Signup is now silently broken for anyone who sends a phone number — and nobody finds out until a user emails you. You never re-ran that Postman request, because re-running forty Postman requests by hand after every change is not a thing humans do.

Think about how cars get certified. Nobody proves a car is safe by driving it around the block and saying "felt fine." They put it on a **closed test track**, strap a **crash-test dummy** into the driver's seat, wire it with sensors, and run the same scripted scenarios on every model that comes off the line.

**Jest is the test track and the clipboard** — it runs each scenario, watches what happened, and stamps pass or fail. **supertest is the crash-test dummy** — it sits *inside* your Express app and fires real HTTP requests into it, so you never start a server, pick a port, or click anything.

> **Type:** two npm dev dependencies — `jest` (runner + assertions + mocking + coverage in one) and `supertest` (an HTTP client that drives an Express app directly).
> **Core promise:** Write down the behavior you expect **once, in code**, and re-verify all of it in seconds — every commit, forever, with no browser and no human.

```mermaid
%%{init: {'themeVariables': { 'primaryTextColor': '#000000', 'textColor': '#000000', 'lineColor': '#000000', 'primaryBorderColor': '#000000'}}}%%
flowchart LR
    T["Test file<br/>users.test.js"] -->|"npx jest"| J["Jest<br/>runs each it block"]
    J -->|"request app"| S["supertest<br/>fires POST /api/users"]
    S --> A["Your Express app<br/>same process"]
    A -->|"res.body + res.status"| E{"expect<br/>matches?"}
    E -->|"yes"| P["PASS"]
    E -->|"no"| F["FAIL<br/>with a diff"]

    style T fill:#e0f0ff,stroke:#000000,color:#000000
    style J fill:#fff2cc,stroke:#000000,color:#000000
    style S fill:#fff2cc,stroke:#000000,color:#000000
    style A fill:#ffffff,stroke:#000000,color:#000000
    style E fill:#fff2cc,stroke:#000000,color:#000000
    style P fill:#e0ffe0,stroke:#000000,color:#000000
    style F fill:#ffe0e0,stroke:#000000,color:#000000
```

---

## 2. Why Does jest + supertest Exist? (The Problem It Solves)

Life without them is either clicking around and hoping, or the homegrown script every beginner writes once:

```js
// check.js — what you write before you know testing exists
import app from "./src/app.js";
const server = app.listen(4001);               // pick a port and pray it is free
const res = await fetch("http://localhost:4001/api/users", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ email: "a@b.com", password: "secret123" }),
});
if (res.status !== 201) console.log("BROKEN");  // no diff, no line number, no detail
server.close();                                 // forget this and it hangs forever
// ...and it just wrote a real user into your dev database. Permanently.
```

That script is slow, order-dependent, leaves garbage in your database, tells you *that* something broke but never *what*, and stops at the one thing you remembered to check.

| The old way | With jest + supertest |
|---|---|
| Manual Postman clicking — never re-run after the change that broke it | One `npm test`, every scenario re-run in seconds, on every commit |
| "It failed" | A colored diff: `Expected: 201, Received: 400`, plus the exact file and line |
| Pick a port, start a server, remember to close it | supertest binds an ephemeral port itself and tears it down per request |
| Test runs pollute your dev database | An in-memory Mongo created in `beforeAll`, thrown away in `afterAll` |
| Refactoring is scary, so bad code stays bad | A green suite is permission to rip things apart and rebuild them |
| "What is this endpoint supposed to do?" | The test file *is* the spec — executable, and it cannot go stale silently |

That last row matters more than it looks: tests are the only documentation that **fails loudly when it becomes a lie**.

---

## 3. Installing & Basic Usage

```bash
npm install --save-dev jest supertest
```

Both are dev dependencies — they must never ship to production.

Jest defaults to CommonJS. Since our code is modern ESM (`"type": "module"` in `package.json`), Jest needs two switches: the `--experimental-vm-modules` Node flag, and an empty `transform` so Jest stops trying to compile the ESM away.

```js
// jest.config.js
export default {
  testEnvironment: "node",  // "node", not "jsdom" — there is no browser in an API
  transform: {},            // no Babel: run our ESM natively
};
```

```json
{
  "scripts": {
    "test": "NODE_OPTIONS=--experimental-vm-modules jest",
    "test:watch": "NODE_OPTIONS=--experimental-vm-modules jest --watch"
  }
}
```

Now the smallest test that can pass. Jest picks up any `*.test.js` file, or anything inside a `__tests__` folder:

```js
// tests/money.test.js — src/money.js exports toCents(rupees) => Math.round(rupees * 100)
import { toCents } from "../src/money.js";

describe("toCents", () => {              // describe = a group, purely for readable output
  it("converts whole rupees", () => {    // it = one scenario, one behavior
    expect(toCents(10)).toBe(1000);      // expect(actual).matcher(expected)
  });
  it("rounds instead of truncating", () => {
    expect(toCents(0.615)).toBe(62);     // 61.5 rounds up — truncating would give 61
  });
});
```

`npm test` prints a green tick per `it` alongside the file name. That is the whole setup — no assertion library to pick, no plugin to install, no config marathon.

### CommonJS version

No `"type": "module"`? Drop the Node flag and the `transform` line — Jest's default mode is exactly this:

```js
const { toCents } = require("../src/money");

test("converts whole rupees", () => {
  expect(toCents(10)).toBe(1000);
});
```

### Express example

The one structural change your app needs: **`app.js` builds and exports the app, `server.js` is the only file that calls `listen()`.**

```js
// src/app.js — knows nothing about ports
import express from "express";
import userRoutes from "./routes/users.js";

const app = express();
app.use(express.json());
app.use("/api/users", userRoutes);
export default app;                              // supertest imports THIS

// src/server.js — the only entry point that touches the network
import mongoose from "mongoose";
import app from "./app.js";
await mongoose.connect(process.env.MONGO_URI);
app.listen(process.env.PORT ?? 3000);
```

```js
// tests/users.test.js
import request from "supertest";
import app from "../src/app.js";

it("rejects signup without a password", async () => {
  const res = await request(app)          // supertest wraps the app — no listen() anywhere
    .post("/api/users")
    .send({ email: "a@b.com" });          // sets Content-Type: application/json for you

  expect(res.status).toBe(400);
});
```

That's the entire mental model — `describe`/`it` name the behavior, `expect` asserts it, and `request(app)` lets you speak HTTP to Express without ever starting a server yourself.

---

## 4. Why Test At All — and What to Actually Test

Three honest reasons, in the order they matter day to day:

1. **Refactoring without fear.** Untested code calcifies. You *know* the 300-line route handler is a mess, but touching it might break checkout, so it stays. A green suite converts "I think this still works" into "I know this still works."
2. **Bugs get caught by a machine, not a customer.** The cheapest bug is one caught 90 seconds after you typed it.
3. **Executable documentation.** `it("locks the account after 5 failed logins")` tells the next developer the rule *and* proves it is still true.

```mermaid
%%{init: {'themeVariables': { 'primaryTextColor': '#000000', 'textColor': '#000000', 'lineColor': '#000000', 'primaryBorderColor': '#000000'}}}%%
flowchart TD
    E["End-to-end<br/>a handful<br/>real browser<br/>slow and flaky"] --> I["Integration<br/>dozens<br/>supertest plus test DB<br/>best value per hour"]
    I --> U["Unit<br/>hundreds<br/>pure functions<br/>milliseconds"]

    style E fill:#ffe0e0,stroke:#000000,color:#000000
    style I fill:#e0ffe0,stroke:#000000,color:#000000
    style U fill:#e0f0ff,stroke:#000000,color:#000000
```

**Unit tests** exercise one function with no I/O. **Integration tests** send a request through your real router, middleware and Mongoose models against a real (throwaway) database. **End-to-end tests** drive an actual browser against a deployed stack: highest confidence, highest maintenance, flakiest. Here is the part nobody tells beginners: **for a typical CRUD API, integration tests over your routes give the best value per hour spent.** A unit test of a Mongoose model mostly tests Mongoose. One supertest call to `POST /api/orders` covers routing, body parsing, [[zod]] validation, auth middleware, the database write and the JSON shape — five layers, one test, roughly 40 milliseconds.

**Rule of thumb:** unit-test logic that is tricky and pure; integration-test every route a user can reach; save end-to-end for the two or three flows that would cost you money if they broke.

---

## 5. Jest Basics — describe, it, expect

```js
expect(2 + 2).toBe(4);                                  // Object.is — primitives ONLY
expect({ a: 1 }).toEqual({ a: 1 });                     // deep value equality — objects/arrays
expect(res.body).toMatchObject({ email: "a@b.com" });   // subset — ignores _id, createdAt...
expect(["a", "b"]).toContain("b");                      // array membership or substring
expect(res.body).toHaveProperty("user.email");          // nested path, safe if parents missing
expect(() => parseAge("abc")).toThrow(/not a number/);  // pass a FUNCTION, not a call result
await expect(getUser("nope")).rejects.toThrow();        // async throw
await expect(getUser("id1")).resolves.toMatchObject({ id: "id1" });
```

`toBe` vs `toEqual` is the single most common beginner stumble. `toBe` asks "is this the same object in memory?" — two structurally identical objects are different objects, so `expect({a:1}).toBe({a:1})` **fails**. `toEqual` walks the structure and compares values, which is what you meant. Primitives get `toBe`; everything else gets `toEqual`. And `toMatchObject` is what makes API tests pleasant — assert only the fields you care about, so adding a field to the response later does not break twenty tests. When you care that a generated value merely *exists*, assert its shape: `expect(res.body).toEqual(expect.objectContaining({ _id: expect.any(String) }))`.

### Lifecycle hooks

```js
beforeAll(async () => { /* ONCE — start the test DB, connect Mongoose */ });
afterAll(async () => { /* ONCE — close every connection or Jest will hang */ });
beforeEach(async () => { /* before EVERY it — seed a known fixture */ });
afterEach(async () => { /* after EVERY it — wipe collections, reset mocks */ });
```

Hooks in a `describe` apply only to tests inside it, and outer `beforeEach` hooks run before inner ones. The rule that saves you hours: **every test must pass alone *and* as part of the full suite.** If ordering matters, your cleanup is wrong.

### Table-driven tests with `it.each`

Ten near-identical tests are ten places to update. Write the table instead:

```js
it.each([
  { body: {},                                  field: "email" },
  { body: { email: "not-an-email" },           field: "email" },
  { body: { email: "a@b.com", password: "1" }, field: "password" },
])("rejects signup when $field is invalid", async ({ body }) => {
  const res = await request(app).post("/api/users").send(body);
  expect(res.status).toBe(400);      // $field is interpolated into the test name
});
```

Each row is reported as its own test, so a failure names the exact row. `it.only` runs one test while you debug, `it.skip` parks one, and `it.todo("handles duplicate emails")` records a test you have not written yet so it shows in the output instead of being forgotten.

---

## 6. Mocking — What to Fake and What to Never Fake

A **mock** is a fake stand-in for a real dependency:

```js
import { jest } from "@jest/globals";   // in ESM the jest object is NOT a global — import it

const send = jest.fn();                 // a spy function: records every call it receives
send.mockReturnValue(true);             // control what it returns
send.mockResolvedValue({ id: "x" });    // async version — resolves to this
expect(send).toHaveBeenCalledWith("hello");
const spy = jest.spyOn(Date, "now").mockReturnValue(1_700_000_000_000);  // wrap a real method
spy.mockRestore();                      // put the original back
```

| Dependency | Mock it? | Why |
|---|---|---|
| Third-party HTTP ([[axios]] to Stripe, a weather API) | **Yes** | Slow, rate-limited, needs secrets — their downtime must not fail your build |
| Email / SMS ([[nodemailer]], Twilio) | **Yes** | Nobody wants 400 real emails per test run |
| Payment providers | **Yes** | You cannot charge a real card in CI |
| The clock (`Date.now`, timers) | **Yes** | "Expires in 15 minutes" is untestable while time keeps moving |
| **Your own database** | **No** | Mocking Mongoose tests your mock, not your query — use a real throwaway DB (section 8) |
| Your route handlers and middleware | **No** | That is precisely what you are trying to verify |
| Express itself | **No** | If the framework is broken, your fake will happily hide it |

**Rule of thumb:** mock what you do not own and cannot afford to call. Run everything you *do* own for real.

### Mocking an axios call

In ESM, `jest.mock()` cannot work — `import` statements resolve before any of your code runs, so there is nothing to hoist the mock above. The ESM tool is `jest.unstable_mockModule` plus a **dynamic import** of the module under test, performed *after* the mock is registered:

```js
import { jest } from "@jest/globals";

jest.unstable_mockModule("axios", () => ({ default: { get: jest.fn() } }));  // mock `default`
const axios = (await import("axios")).default;      // dynamic: runs after the mock exists
const { getTemperature } = await import("../src/weather.js");

it("returns the temperature from the upstream API", async () => {
  axios.get.mockResolvedValue({ data: { current: { temp_c: 31 } } });

  expect(await getTemperature("Delhi")).toBe(31);
  // Assert the CONTRACT you send upstream, not only the answer you got back
  expect(axios.get).toHaveBeenCalledWith(expect.stringContaining("q=Delhi"));
});
```

The CommonJS equivalent is the familiar hoisted form: `jest.mock("axios")` at the top of the file, then plain `require`s.

### Mocking a nodemailer send

[[nodemailer]] is two levels deep — `createTransport()` returns an object with `sendMail()` — so the factory must return an object:

```js
import { jest } from "@jest/globals";

const sendMail = jest.fn().mockResolvedValue({ messageId: "fake-id" });
jest.unstable_mockModule("nodemailer", () => ({
  default: { createTransport: jest.fn(() => ({ sendMail })) },
}));
const request = (await import("supertest")).default;
const app = (await import("../src/app.js")).default;

it("emails a welcome message on signup", async () => {
  const body = { email: "a@b.com", password: "secret123" };
  await request(app).post("/api/users").send(body).expect(201);
  expect(sendMail).toHaveBeenCalledTimes(1);
  expect(sendMail.mock.calls[0][0].to).toBe("a@b.com");   // inspect the real argument
});
```

Need the clock frozen? `jest.useFakeTimers().setSystemTime(new Date("2026-01-01"))`, then `jest.useRealTimers()` when done — skip the restore and every later test lives in 2026.

> ⚠️ Mock state leaks between tests by default. Set `clearMocks: true` and `restoreMocks: true` in `jest.config.js` so call counts reset and `spyOn` originals are restored automatically. Without them you will one day spend an hour debugging a phantom extra call.

---

## 7. supertest — Testing Express Without a Port

The idea that makes supertest click: `request(app)` takes your Express app — which is just a request-handler function — starts it on an **ephemeral port** chosen by the OS for the duration of that one request, sends the request, and shuts it down. You never pick a port, never call `listen`, and twenty test files run in parallel without colliding.

This is exactly why `app.js` and `server.js` must be separate. If `app.js` ended with `app.listen(3000)`, merely importing it from a test would bind port 3000 — the second test file would die with `EADDRINUSE`, and Jest would hang afterwards because a live server is an open handle.

```js
import request from "supertest";
import app from "../src/app.js";

describe("POST /api/users", () => {
  it("creates a user and never returns the password", async () => {
    const res = await request(app)
      .post("/api/users")
      .send({ email: "a@b.com", password: "secret123" })
      .expect("Content-Type", /json/)   // regex match on the response header
      .expect(201);                     // supertest can assert the status itself

    expect(res.body).toMatchObject({ email: "a@b.com" });
    expect(res.body.passwordHash).toBeUndefined();   // the assertion that catches real leaks
  });

  it("returns 400 when validation fails", async () => {
    const res = await request(app)
      .post("/api/users").send({ email: "not-an-email", password: "1" }).expect(400);

    // Your zod error handler flattens issues into an array — assert shape, not prose
    expect(res.body.errors).toEqual(
      expect.arrayContaining([expect.objectContaining({ path: ["email"] })]));
  });
});
```

Note the second test: a `400` from [[zod]] is a behavior worth pinning, because a refactor that accidentally makes validation permissive is silent otherwise.

Chaining a real auth flow — sign up, log in, then use the returned [[jsonwebtoken]] on a protected route:

```js
describe("GET /api/me", () => {
  it("returns the profile for a valid token", async () => {
    const creds = { email: "me@b.com", password: "secret123" };
    await request(app).post("/api/users").send(creds).expect(201);

    const login = await request(app).post("/api/auth/login").send(creds).expect(200);

    const res = await request(app)
      .get("/api/me")
      .set("Authorization", `Bearer ${login.body.token}`)  // what a browser would send
      .expect(200);

    expect(res.body.email).toBe("me@b.com");
  });

  it("returns 401 without a token", async () => {
    await request(app).get("/api/me").expect(401);
  });

  it("returns 401 for a garbage token", async () => {
    // Proves the middleware catches the jwt.verify() throw instead of 500-ing
    await request(app).get("/api/me").set("Authorization", "Bearer no.pe").expect(401);
  });
});
```

Other builders you will need: `.query({ page: 2 })` for a query string, `.attach("avatar", "tests/fixtures/pic.png")` plus `.field("title", "x")` for a [[multer]] upload, and `request.agent(app)` when auth uses **cookies** instead of a header — an agent persists cookies across requests the way a browser does.

---

## 8. A Real Test Database

Never point tests at your dev database, and obviously never at production — tests delete rows. You want a database created for the run and destroyed after it. For Mongo that is `mongodb-memory-server`, which downloads a real `mongod` binary once, then starts it on a random port against a throwaway data directory it deletes on shutdown.

```bash
npm install --save-dev mongodb-memory-server
```

```js
// tests/setup.js — wired in via setupFilesAfterEnv, so it applies to every test file
import { MongoMemoryServer } from "mongodb-memory-server";
import mongoose from "mongoose";

let mongo;

beforeAll(async () => {
  mongo = await MongoMemoryServer.create();   // a real mongod, random port, temp data dir
  await mongoose.connect(mongo.getUri());     // your models now talk to it — zero code changes
});

afterEach(async () => {
  const collections = await mongoose.connection.db.collections();
  for (const c of collections) await c.deleteMany({});   // wipe: test order can never matter
});

afterAll(async () => {
  await mongoose.connection.dropDatabase();
  await mongoose.connection.close();          // close the client BEFORE stopping the server
  await mongo.stop();                         // skip this and Jest never exits
});
```

```js
// jest.config.js — the section 3 config, plus four lines
export default {
  testEnvironment: "node",
  transform: {},
  setupFiles: ["<rootDir>/tests/env.js"],            // runs first: environment variables
  setupFilesAfterEnv: ["<rootDir>/tests/setup.js"],  // runs after Jest exists: hooks live here
  clearMocks: true,                                  // reset call counts between tests
  restoreMocks: true,                                // undo every spyOn automatically
};
```

The distinction matters: `setupFiles` runs *before* the test framework is installed, so it is the right place for env vars but you cannot call `beforeAll` there. `setupFilesAfterEnv` runs after, so hooks work.

```js
// tests/env.js
import dotenv from "dotenv";

process.env.NODE_ENV = "test";                         // your app silences logs on this
dotenv.config({ path: ".env.test", override: true });   // override anything a real .env set

if (process.env.MONGO_URI?.includes("mongodb+srv")) {   // a seatbelt that has saved real data
  throw new Error("Refusing to run tests against a remote cluster");
}
```

`.env.test` holds only fake secrets (`JWT_SECRET=test-secret-not-used-anywhere-real`), so committing it is fine — see [[dotenv]] for how config layering works.

### The Postgres / Prisma equivalent

There is no in-memory Postgres worth using, so with [[prisma]] the standard pattern is a throwaway **Docker container** on a non-default port. Point `.env.test` at it, apply the schema with `migrate deploy` (not `migrate dev` — `deploy` never prompts and never regenerates migrations), and `TRUNCATE` your tables in `afterEach` instead of dropping collections:

```bash
docker run -d --name test-pg -e POSTGRES_PASSWORD=test -p 5433:5432 postgres:17
DATABASE_URL="postgresql://postgres:test@localhost:5433/test" npx prisma migrate deploy
```

---

## 9. TypeScript Version

Jest cannot read TypeScript on its own — it needs a transform. `ts-jest` is the standard choice; `@swc/jest` is the faster one once compile time starts hurting. `ts-node` is what lets Jest load a `jest.config.ts` at all, and the `NODE_OPTIONS=--experimental-vm-modules` script from section 3 is still required.

```bash
npm install --save-dev typescript ts-jest ts-node @types/jest @types/supertest
```

```ts
// jest.config.ts
import type { Config } from "jest";

const config: Config = {
  preset: "ts-jest/presets/default-esm",   // TypeScript + native ESM
  testEnvironment: "node",
  extensionsToTreatAsEsm: [".ts"],
  moduleNameMapper: { "^(\\.{1,2}/.*)\\.js$": "$1" },  // ESM imports end .js — map back to .ts
  setupFilesAfterEnv: ["<rootDir>/tests/setup.ts"],
};

export default config;
```

Real Express types, not `any` — the middleware under test, then the test that pins its contract:

```ts
// src/middleware/requireAuth.ts
import type { Request, Response, NextFunction } from "express";
import jwt from "jsonwebtoken";

export interface AuthedRequest extends Request {
  userId?: string;                        // what the middleware attaches on success
}

export function requireAuth(req: AuthedRequest, res: Response, next: NextFunction): void {
  const header = req.headers.authorization;
  if (!header?.startsWith("Bearer ")) {
    res.status(401).json({ error: "Missing token" });
    return;                               // return void — never `return res.json()`
  }
  try {
    const { sub } = jwt.verify(header.slice(7), process.env.JWT_SECRET!) as { sub: string };
    req.userId = sub;
    next();
  } catch {
    res.status(401).json({ error: "Invalid token" });
  }
}
```

```ts
// tests/me.test.ts
import request from "supertest";
import jwt from "jsonwebtoken";
import app from "../src/app.js";
import { User } from "../src/models/User.js";

interface MeResponse { _id: string; email: string }

it("returns the caller's profile with a valid token", async () => {
  const user = await User.create({ email: "me@b.com", passwordHash: "not-read-here" });
  const token = jwt.sign({ sub: user.id }, process.env.JWT_SECRET!, { expiresIn: "1h" });

  const res = await request(app)
    .get("/api/me")
    .set("Authorization", `Bearer ${token}`)
    .expect(200);

  const body = res.body as MeResponse;   // res.body is `any` — pin the type down yourself
  expect(body.email).toBe("me@b.com");
});
```

---

## 10. Production Setup

Add one more script next to the two from section 3 — the one CI will call:

```json
"test:ci": "NODE_OPTIONS=--experimental-vm-modules jest --ci --coverage --runInBand"
```

Inline `VAR=value` prefixes do not work on Windows `cmd` — add `cross-env` if your team is mixed.

| Flag | When you need it |
|---|---|
| `--watch` | Local development. Re-runs only tests affected by files changed since your last commit; `--watchAll` re-runs everything. |
| `--coverage` | Prints a per-file table of which lines actually executed. |
| `--runInBand` (`-i`) | Runs test files **serially** in one process. Required when files share one database, and often faster in a CPU-limited CI container. |
| `--detectOpenHandles` | Diagnoses "Jest did not exit one second after the test run completed" by printing the stack trace of whatever is still open. |
| `--ci` | Fails instead of silently writing new snapshots. Always on in CI. |
| `-t "creates a user"` | Runs only tests whose name matches, while you debug one thing. |
| `--forceExit` | Kills the process regardless of open handles. A **band-aid, not a fix** — you are hiding a leak that also exists in production. |

### Coverage — and why 100% is the wrong goal

```js
collectCoverageFrom: ["src/**/*.js", "!src/server.js"],   // the listen file has nothing to assert
coverageThreshold: { global: { statements: 80, branches: 70, functions: 75, lines: 80 } },
```

Coverage measures which lines *ran*, not which behaviors are *correct* — a test that calls every function and asserts nothing scores 100%. Chasing the last 15% pushes people into brittle tests for getters and unreachable error branches, and those break on every refactor. Use coverage as a **map of untested regions** ("the whole payments module is at 12%") rather than a score. 70-80% on route and service files is a healthy real-world number.

### Running in CI

```yaml
# .github/workflows/test.yml
name: test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }   # cache: npm reuses the npm download cache
      - run: npm ci                              # ci, not install — respects the lockfile
      - run: npm run test:ci
```

Locally, wire the suite into a **pre-push** hook with [[husky_lint_staged]] so broken code never reaches the remote. Keep it on pre-push, not pre-commit — a full suite on every commit makes people reach for `--no-verify`, which defeats the point.

---

## 11. Common Gotchas & Best Practices

| Gotcha | Fix |
|---|---|
| **"Jest did not exit one second after the test run completed"** | Something is still open — almost always a Mongoose connection, an [[ioredis]] client, a `setInterval`, or an `app.listen()` you forgot. Run `--detectOpenHandles` for the stack trace, then close it in `afterAll`. Do **not** reach for `--forceExit`; that same leak is why your container ignores `SIGTERM` in production. |
| **Tests pass alone, fail when run together** | Shared state. Wipe every collection in `afterEach`, never let one `it` depend on data another created, and add `--runInBand` if separate test *files* are fighting over one database. |
| **`EADDRINUSE`, or the suite hangs the moment you import the app** | `app.listen()` is sitting inside `app.js`. Split it: `app.js` exports the app, `server.js` calls `listen`. supertest binds its own ephemeral port. |
| **`expect(res.body).toBe({ email: "a@b.com" })` always fails** | `toBe` is reference identity. Use `toEqual` for deep equality, or `toMatchObject` to assert a subset and ignore `_id` / `createdAt` / `__v`. |
| **A test passes but the assertion never ran** | You forgot `await` on the supertest chain, so Jest finished the function before the request resolved. Always `await request(app)...`. For callback-style tests add `expect.assertions(2)` so Jest fails if that many assertions did not run. |
| **`ReferenceError: jest is not defined`** | In ESM the `jest` object is not injected as a global. Add `import { jest } from "@jest/globals";` at the top of the file. |
| **`jest.mock()` silently does nothing in ESM** | Hoisting does not exist for ES modules. Use `jest.unstable_mockModule("pkg", factory)` **and** load the module under test with a dynamic `await import()` afterwards. |
| **Tests wiped your dev database** | You never isolated the environment. Use `mongodb-memory-server`, load `.env.test` from `setupFiles`, and throw at startup if the URI is not local. |

---

## 12. Alternatives — When jest + supertest Isn't the Best Fit

| Tool | What it is | Best for |
|---|---|---|
| **Jest** | Runner + assertions + mocking + coverage in one package. Huge ecosystem; every tutorial assumes it. | Existing projects, CommonJS codebases, teams that want the safest default and the most Stack Overflow answers. |
| **Vitest** | Nearly identical API (`describe`/`it`/`expect`/`vi.fn`), but ESM- and TypeScript-native with no flags, and far faster in watch mode. | **New projects — recommended.** No `--experimental-vm-modules`, no `transform: {}`, no `unstable_mockModule`; `vi.mock` just works with ESM. Migration is mostly `jest.` → `vi.`. |
| **`node:test`** | Built into Node 20+, zero dependencies. `node --test`, with `assert` for matchers. | Small libraries and CLIs where a dependency-free toolchain matters. Weaker mocking, thinner watch mode, plainer failure output. |
| **Mocha + Chai + Sinon** | The classic three-package split: runner, assertions, mocks. | Legacy codebases already using it. For anything new, three packages to wire up is strictly more work than one. |
| **supertest** | Drives an Express app in-process — no port, no browser. | Every API integration test. This is the default and stays the default. |
| **Playwright / Cypress** | Real browser automation against a deployed app. | True end-to-end: does the user actually *see* the dashboard after logging in? Seconds per test and flakier, but the only thing that tests frontend and backend together. |

```mermaid
%%{init: {'themeVariables': { 'primaryTextColor': '#000000', 'textColor': '#000000', 'lineColor': '#000000', 'primaryBorderColor': '#000000'}}}%%
flowchart TD
    Q1{"What are you<br/>testing?"}
    Q1 -->|"a pure function"| Q2{"New project<br/>or existing?"}
    Q1 -->|"an HTTP route"| ST["supertest<br/>plus Jest or Vitest"]
    Q1 -->|"a user journey<br/>through the UI"| PW["Playwright<br/>or Cypress"]
    Q2 -->|"new, ESM plus TS"| V["Vitest"]
    Q2 -->|"existing Jest setup"| J["Stay on Jest"]
    Q2 -->|"zero dependencies"| N["node:test"]

    style Q1 fill:#e0f0ff,stroke:#000000,color:#000000
    style Q2 fill:#e0f0ff,stroke:#000000,color:#000000
    style ST fill:#e0ffe0,stroke:#000000,color:#000000
    style V fill:#e0ffe0,stroke:#000000,color:#000000
    style J fill:#fff2cc,stroke:#000000,color:#000000
    style N fill:#ffffff,stroke:#000000,color:#000000
    style PW fill:#ffe0e0,stroke:#000000,color:#000000
```

**Rule of thumb:** starting fresh in 2026 with ESM and TypeScript? Use **Vitest** — same API, none of the ESM friction. Inheriting a codebase that already runs Jest? Leave it alone. Either way **supertest is the answer for route tests**, and neither choice changes a line of your test bodies.

---

## 13. Interview Questions

**Q: Why must `app.js` and `server.js` be separate files for testing?**
A: supertest needs the Express app object, not a running server — it binds its own ephemeral port per request. If `app.js` called `app.listen()`, importing it from a test would occupy a fixed port, so a second test file would fail with `EADDRINUSE`, and the live server would keep Jest's process alive after the suite finished. Exporting the app from `app.js` and calling `listen()` only in `server.js` keeps the app importable and free of side effects.

**Q: What is the difference between `toBe` and `toEqual`?**
A: `toBe` uses `Object.is`, so for objects and arrays it asks "is this literally the same reference in memory?" — two structurally identical objects fail it. `toEqual` recursively compares values, which is what you want for objects, arrays and API response bodies. Use `toBe` for primitives, `toEqual` for structures, and `toMatchObject` when you only care about a subset of the fields.

**Q: Should you mock your database in tests?**
A: Not in an integration test. Mocking Mongoose means you are asserting against your own fake, so a broken query, a missing index or a schema rule will pass happily. Use a real throwaway database — `mongodb-memory-server` for Mongo, a Docker container for Postgres — and mock only what you do not own and cannot afford to call: third-party APIs, email, payments, the clock.

**Q: Jest says "did not exit one second after the test run completed". What is happening?**
A: An asynchronous handle is still open — usually a Mongoose or Redis connection, a timer, or a server you started. Run with `--detectOpenHandles` to get the stack trace of the culprit, then close it in `afterAll`. `--forceExit` makes the message disappear but hides a real leak that will also stop your production container from shutting down cleanly.

**Q: What does the testing pyramid recommend, and do you agree for a CRUD API?**
A: It says write many fast unit tests, fewer integration tests and very few end-to-end tests, because cost and flakiness rise as you go up. For a typical CRUD API I would deliberately weight it toward integration tests: one supertest call through a route exercises routing, body parsing, validation, auth middleware, the database write and the response shape at once. Unit tests are still ideal for genuinely tricky pure logic like pricing or permission rules.

**Q: How do you test a route protected by a JWT?**
A: Do it end to end inside the test: sign up, log in through the real login route, take the token out of the response body, and send it as `.set("Authorization", "Bearer " + token)` on the protected request. That proves the whole chain — hashing, signing and verification — rather than just the handler. Then add the negative cases: no header at all, and a malformed token, both expecting 401.

**Q: When would you reach for Playwright instead of supertest?**
A: supertest stops at the HTTP boundary — it never renders a page or runs frontend JavaScript. When the question is "can a real user log in and see their dashboard", you need a real browser, which means Playwright or Cypress. Those tests take seconds each and are flakier, so keep them to the two or three flows that would cost real money if they silently broke.

---

## 14. Quick Cheat Sheet

```bash
npm install --save-dev jest supertest mongodb-memory-server

npx jest                       # everything, once
npx jest --watch               # re-run tests affected by changed files
npx jest tests/users.test.js   # one file
npx jest -t "creates a user"   # one test, by name
npx jest --coverage            # coverage table
npx jest --runInBand           # serial — when tests share one database
npx jest --detectOpenHandles   # diagnose "Jest did not exit"
```

```js
// Matchers
expect(n).toBe(4);                             // primitives
expect(obj).toEqual({ a: 1 });                 // deep equality
expect(res.body).toMatchObject({ ok: true });  // subset
await expect(p).rejects.toThrow();
expect(fn).toHaveBeenCalledWith("arg");

// supertest
const res = await request(app)
  .post("/api/users")
  .set("Authorization", `Bearer ${token}`)
  .send({ email: "a@b.com", password: "secret123" })
  .expect("Content-Type", /json/)
  .expect(201);
```

```js
// Mocking in ESM
import { jest } from "@jest/globals";
jest.unstable_mockModule("axios", () => ({ default: { get: jest.fn() } }));
const axios = (await import("axios")).default;
axios.get.mockResolvedValue({ data: { ok: true } });
```

**Mental model to remember:**
> Jest is the referee that runs your scenarios and prints a diff when reality disagrees with you; supertest is the fake client that fires real HTTP at your Express app with no server to start — which is why `app.js` must export the app and only `server.js` may call `listen()`. Mock what you do not own (network, email, the clock), run what you do own for real against a disposable database, and lean on integration tests over routes because one 40-millisecond call covers [[express]], [[zod]], [[jsonwebtoken]] and [[mongoose]] at once. Wire `npm test` into a pre-push hook with [[husky_lint_staged]] and let the machine do the clicking.
