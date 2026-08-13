# Environment Variables in JS/Node.js — Deep Dive (`dotenv`)

> **Scope:** Node.js / backend only (no Vite/Next.js/browser bundler stuff here).
> **Level:** Deep dive — covers the concept, `process.env`, the `dotenv` package, its internals, Node's native `--env-file` support, and security best practices.

---

## 1. ELI5: What is an Environment Variable?

Imagine your JS program is a **chef** working in a kitchen. The chef doesn't grow the ingredients or own the building — someone hands them a **note stuck on the fridge** before they start cooking:

> "Today, use **Kitchen #3**. The **spice level is MILD**. The **delivery API key is XYZ**."

That sticky note isn't part of the chef's recipe (code) — it's information the **outside world** (the kitchen manager / operating system) gives the chef **before** they start working. Tomorrow, a different kitchen might stick a different note: "Use Kitchen #7, spice level HOT."

**An environment variable is exactly that sticky note** — a key-value pair that lives **outside your code**, in the operating system (or shell/process that launches your program), and your program can read it at runtime.

```
KEY=VALUE
PORT=3000
DATABASE_URL=postgres://localhost/mydb
NODE_ENV=production
```

> **Full name:** Environment Variable (often shortened to "env var")
> **Lives in:** The OS process environment (not your source code, not your Git repo)
> **Core promise:** Let the *same code* behave differently depending on *where/how* it's run — without editing the code.

---

## 2. Why Do Environment Variables Exist? (The Problem They Solve)

Without env vars, you'd have to **hardcode** config directly into your source code:

```js
// ❌ Bad: hardcoded
const dbUrl = "postgres://admin:supersecret123@prod-db.company.com/app";
const port = 3000;
```

This causes real problems:

| Problem | Why it hurts |
|---|---|
| **Secrets in code** | Your DB password is now in Git history forever, visible to anyone with repo access. |
| **Different environments need different values** | Your laptop (dev), the CI server (test), and production all need a *different* `DATABASE_URL`, but it's the *same* code. |
| **Can't change config without redeploying** | Want to flip a feature flag or change a port? You'd have to edit code and redeploy. |
| **Team collaboration** | Everyone on the team would need the *same* hardcoded values, or you'd be constantly commenting/uncommenting lines. |

**Environment variables fix this** by separating **config** from **code** — the same principle behind the famous [12-Factor App](https://12factor.net/config) methodology: *"Store config in the environment."*

```mermaid
%%{init: {'themeVariables': { 'primaryTextColor': '#000000', 'textColor': '#000000', 'lineColor': '#000000', 'primaryBorderColor': '#000000'}}}%%
flowchart LR
    Code["Your Code<br/>(same on every machine)"]
    Dev["Dev Machine<br/>PORT=3000<br/>DB=localhost"]
    CI["CI Server<br/>PORT=4000<br/>DB=test-db"]
    Prod["Production<br/>PORT=80<br/>DB=prod-db.company.com"]

    Code -->|reads env vars at runtime| Dev
    Code -->|reads env vars at runtime| CI
    Code -->|reads env vars at runtime| Prod

    style Code fill:#e0f0ff,stroke:#000000,color:#000000
    style Dev fill:#e0ffe0,stroke:#000000,color:#000000
    style CI fill:#fff2cc,stroke:#000000,color:#000000
    style Prod fill:#ffe0e0,stroke:#000000,color:#000000
```

**Key insight:** The code never changes. Only the environment around it changes.

---

## 3. `process.env` — How Node.js Exposes Env Vars

Node.js gives you a global object called `process.env`. Every environment variable set in the OS/shell before your program starts is available here **as a string**.

```js
console.log(process.env.PORT);      // "3000" (string, even if it looks like a number!)
console.log(process.env.NODE_ENV);  // "development"
console.log(process.env.HOME);      // "/home/naman" (OS-level vars are here too)
```

Try it yourself in a terminal:

```bash
PORT=5000 node -e "console.log(process.env.PORT)"
# → 5000
```

### ⚠️ Important gotchas about `process.env`

| Gotcha | Explanation |
|---|---|
| **Everything is a string** | `PORT=3000` → `process.env.PORT === "3000"`, NOT `3000`. You must `Number(process.env.PORT)` or `parseInt(...)` yourself. |
| **Booleans don't exist** | `DEBUG=false` → `process.env.DEBUG === "false"` (a truthy **string**!). `if (process.env.DEBUG)` is **always true** unless the var is literally unset. |
| **Undefined if not set** | Accessing a var that was never set gives `undefined`, not an error. |
| **It's a plain object** | You *can* mutate it (`process.env.FOO = "bar"`), but that only affects the current process and its children — not the real OS. |

---

## 4. Where Do Env Vars Come From? (Without Any Package)

Before touching `dotenv`, understand: **Node didn't invent env vars.** They come from the OS/shell that launches `node`. Three common ways to set them, no package needed:

```bash
# 1. Inline, one-off (only for this command)
PORT=4000 node app.js

# 2. Exported in the shell (persists for the whole terminal session)
export PORT=4000
node app.js

# 3. Passed by whatever launches Node (Docker, systemd, PM2, CI runner, Heroku, etc.)
docker run -e PORT=4000 my-app
```

This works fine for **one or two** variables. But real apps need 10-30+ config values (DB URLs, API keys, feature flags...). Typing `export` for each one every time you open a terminal is painful and error-prone. **That's the exact problem `dotenv` solves.**

---

## 5. The `.env` File & the `dotenv` Package

`dotenv` is a tiny npm package that reads a file named `.env` (sitting in your project root) and loads its key-value pairs **into `process.env`** for you — so you get the `export` behavior automatically, from a file, once per project.

### Step-by-step

```bash
npm install dotenv
```

Create a `.env` file in your project root:

```dotenv
# .env
PORT=3000
DATABASE_URL=postgres://localhost:5432/mydb
API_KEY=abc123xyz
NODE_ENV=development
```

Load it at the **very top** of your entry file:

```js
// index.js
require('dotenv').config();
// or, ESM:
// import 'dotenv/config';

console.log(process.env.PORT);         // "3000"
console.log(process.env.DATABASE_URL); // "postgres://localhost:5432/mydb"
```

That's it — `dotenv.config()` reads `.env`, parses it, and copies each key into `process.env`, as if you'd `export`-ed them all in your shell before running `node index.js`.

```mermaid
%%{init: {'themeVariables': { 'primaryTextColor': '#000000', 'textColor': '#000000', 'lineColor': '#000000', 'primaryBorderColor': '#000000'}}}%%
flowchart LR
    File[".env file<br/>PORT=3000"] -->|"dotenv reads & parses"| Dotenv["dotenv.config()"]
    Dotenv -->|"copies keys in"| Env["process.env<br/>(global object)"]
    Env -->|"your code reads it"| App["Your App Logic"]

    style File fill:#fff2cc,stroke:#000000,color:#000000
    style Dotenv fill:#e0f0ff,stroke:#000000,color:#000000
    style Env fill:#e0ffe0,stroke:#000000,color:#000000
    style App fill:#ffffff,stroke:#000000,color:#000000
```

---

## 6. `.env` File Syntax Rules (the format itself)

```dotenv
# This is a comment
SIMPLE=hello
WITH_SPACES = "hello world"      # quotes preserve leading/trailing spaces
SINGLE_QUOTED='raw $tring'       # single quotes = no variable expansion
EMPTY=
NUMBER_LOOKING=3000              # still becomes the STRING "3000"
MULTILINE="line1\nline2"         # \n inside double quotes becomes a real newline
# NAME=value   <- commented out, ignored
```

| Rule | Detail |
|---|---|
| `KEY=VALUE` | No spaces required around `=`, but spaces *around* it are trimmed unless quoted. |
| `#` starts a comment | Anything after `#` on its own line (or after unquoted value) is ignored. |
| Quotes are optional | Unquoted, single-quoted, and double-quoted values are all valid — but they behave slightly differently. |
| Double quotes expand escapes | `"a\nb"` → real newline. Single quotes keep it literal. |
| No expansion of other vars by default | `URL=$HOST/path` does **NOT** substitute `$HOST` — base `dotenv` treats it as a literal string. (See `dotenv-expand` below.) |
| Keys are case-sensitive | `Port` and `PORT` are different keys. |

---

## 7. How `dotenv` Works Internally (the actual mechanism)

`dotenv` is intentionally tiny (~100 lines of core logic). Conceptually, `.config()` does:

1. **Find the file** — defaults to `.env` in `process.cwd()` (can override with `{ path: '...' }`).
2. **Read it** as plain text (`fs.readFileSync`).
3. **Parse it** line by line with a regex that splits `KEY` from `VALUE`, strips quotes/comments, handles escapes.
4. **Assign to `process.env`** — but **only for keys that don't already exist** in `process.env`.

```js
const result = require('dotenv').config();
// result.parsed = { PORT: '3000', DATABASE_URL: '...' }
// result.error  = set if the file couldn't be read
```

### 🔑 The most misunderstood rule: `dotenv` never overrides existing env vars

If `PORT` is **already set** in the real shell environment (e.g. by Docker, Heroku, or `export PORT=8080`), `dotenv` will **NOT** overwrite it with the `.env` file's value. This is intentional — real deployment env vars should always win over a local `.env` file.

```js
// Force .env values to win instead (rarely what you want in prod):
require('dotenv').config({ override: true });
```

### Bonus: `dotenv-expand` (variable interpolation)

Base `dotenv` doesn't let one var reference another. A companion package, `dotenv-expand`, adds that:

```dotenv
HOST=localhost
PORT=3000
BASE_URL=http://${HOST}:${PORT}
```

```js
const dotenv = require('dotenv');
const dotenvExpand = require('dotenv-expand');
dotenvExpand.expand(dotenv.config());
// process.env.BASE_URL === "http://localhost:3000"
```

---

## 8. Node's Native Env File Support (no package needed!)

Since **Node.js v20.6.0**, Node can load a `.env` file **natively**, without installing `dotenv` at all:

```bash
node --env-file=.env index.js
```

Or programmatically (Node v21.7+ / v20.12+, stable since v22):

```js
process.loadEnvFile();        // loads default .env
process.loadEnvFile('.env.production'); // or a specific file
```

### `dotenv` package vs Node's native support

| Feature | `dotenv` (npm package) | Native `--env-file` / `process.loadEnvFile()` |
|---|---|---|
| Requires install | Yes (`npm install dotenv`) | No — built into Node |
| Min Node version | Any | v20.6+ (flag), v20.12+/v22 (stable API) |
| Multiple files, expansion | Yes (with `dotenv-expand`, `dotenv-flow`) | Basic — one file per flag, no built-in expansion (yet) |
| Ecosystem / docs / battle-tested | Huge (millions of projects, since 2013) | Newer, fewer examples, evolving |
| Fine-grained control (`override`, custom path, error handling) | Rich options object | Minimal, still maturing |
| Works in older Node / other tooling expecting it | Yes | No (needs modern Node) |

**Rule of thumb:** If you're on Node 20.6+ and only need "read one `.env` file, no fancy features" → native is fine and removes a dependency. If you need multi-environment files, variable expansion, or support older Node/broad ecosystem compatibility → use the `dotenv` package. Most real-world projects (and almost all tutorials/boilerplates) still use the `dotenv` package as of today.

---

## 9. Precedence: What Wins When Multiple Sources Set the Same Var?

From **lowest to highest priority** (highest wins):

1. `.env` file loaded via `dotenv.config()`
2. Real OS/shell environment variables (`export FOO=bar`, or set by Docker/PM2/Heroku/CI)
3. Inline vars on the command itself (`PORT=9000 node index.js`)

```mermaid
%%{init: {'themeVariables': { 'primaryTextColor': '#000000', 'textColor': '#000000', 'lineColor': '#000000', 'primaryBorderColor': '#000000'}}}%%
flowchart TD
    A[".env file"] -->|"lowest priority<br/>fills in if not already set"| D["process.env"]
    B["Shell/OS export"] -->|"wins over .env"| D
    C["Inline on command line"] -->|"highest priority"| D

    style A fill:#fff2cc,stroke:#000000,color:#000000
    style B fill:#e0f0ff,stroke:#000000,color:#000000
    style C fill:#e0ffe0,stroke:#000000,color:#000000
    style D fill:#ffffff,stroke:#000000,color:#000000
```

---

## 10. Environment-Specific Files (dev / staging / prod)

A common pattern is having **multiple `.env` files** and loading the right one based on `NODE_ENV`:

```
.env                # shared defaults, always loaded
.env.development     # loaded only in dev
.env.production       # loaded only in prod
.env.local            # personal overrides, git-ignored, loaded everywhere
```

```js
require('dotenv').config({
  path: `.env.${process.env.NODE_ENV || 'development'}`
});
```

For a more robust version of this cascading behavior (merging `.env` + `.env.{NODE_ENV}` + `.env.local` automatically, like Next.js/Create React App do), people use the **`dotenv-flow`** package instead of hand-rolling the path logic.

---

## 11. Security Best Practices (⚠️ where real incidents happen)

| Rule | Why |
|---|---|
| **Never commit `.env` to Git** | Add `.env` to `.gitignore` immediately. A leaked `.env` = leaked DB password/API keys, often scraped by bots within minutes of a public push. |
| **Commit a `.env.example` instead** | A template with keys but no real values (`API_KEY=`) so teammates know what to fill in, without exposing secrets. |
| **Don't `console.log(process.env)`** | Dumps every secret to your logs (which may be shipped to a log aggregator, visible to more people than you think). |
| **Validate required vars at startup** | Fail fast and loud if a required var is missing, instead of crashing mysteriously mid-request: |

```js
// Simple manual check
const required = ['DATABASE_URL', 'API_KEY', 'PORT'];
for (const key of required) {
  if (!process.env[key]) {
    throw new Error(`Missing required env var: ${key}`);
  }
}
```

```js
// Or with a schema validator like zod (nicer errors, type coercion)
import { z } from 'zod';

const envSchema = z.object({
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  NODE_ENV: z.enum(['development', 'production', 'test']),
});

export const env = envSchema.parse(process.env); // throws with a clear message if invalid
```

| Rule | Why |
|---|---|
| **Use a real secrets manager in production** | `.env` files are fine for local dev. In real production, prefer AWS Secrets Manager, Google Secret Manager, HashiCorp Vault, or your platform's built-in env var UI (Heroku/Render/Vercel dashboards) — these are encrypted at rest and access-controlled. |
| **Rotate secrets that leak** | If a `.env` or key ever does leak (e.g. committed by mistake), rotate/regenerate it immediately — deleting the commit from Git history is not enough, assume it's compromised. |
| **Least privilege** | Don't give every service the same all-powerful API key; scope keys to what each service actually needs. |

---

## 12. Common Alternatives / Related Tools

| Tool | What it adds |
|---|---|
| **`dotenv`** | The baseline: load `.env` into `process.env`. |
| **`dotenv-expand`** | Adds `${VAR}` interpolation within `.env` files. |
| **`dotenv-flow`** | Automatically cascades `.env` + `.env.{NODE_ENV}` + `.env.local` files. |
| **`dotenvx`** | Newer, encrypts `.env` files so they *can* be safely committed (`.env.vault`), plus multi-environment support. |
| **`envalid` / `zod`** | Validates and type-casts `process.env` into a typed, safe config object at startup. |
| **`cross-env`** | Lets you write `PORT=3000 node index.js` in `package.json` scripts in a way that works on **both** Windows and Unix shells (Windows `cmd` doesn't support that inline syntax natively). |
| **`direnv`** | Shell-level tool (not JS) that auto-loads/unloads env vars when you `cd` into a project directory — outside Node entirely. |

---

## 13. Quick Cheat Sheet

```bash
npm install dotenv
```

```js
// Load as early as possible in your entry file
require('dotenv').config();
// import 'dotenv/config';           // ESM shorthand

// Read values (always strings!)
const port = Number(process.env.PORT) || 3000;
const isProd = process.env.NODE_ENV === 'production';
```

```dotenv
# .env  (git-ignored, never committed)
PORT=3000
DATABASE_URL=postgres://localhost:5432/mydb
API_KEY=change-me
```

```gitignore
# .gitignore
.env
.env.*.local
```

**Mental model to remember:**
> `.env` file → parsed by `dotenv` (or Node natively) → copied into `process.env` → your code reads `process.env.KEY` → config now varies per machine, without touching code.
