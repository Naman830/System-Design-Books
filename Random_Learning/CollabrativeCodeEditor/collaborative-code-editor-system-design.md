# CodeSync — Collaborative Code Editor
### Full System Design (HLD + LLD) — Portfolio Reference Guide

> Goal: a real-time collaborative code editor with live cursors, presence, and sandboxed code execution — built to demonstrate distributed-systems thinking to recruiters, not just CRUD skills.

---

## Table of Contents
1. [Critique of Your Original Sketch](#1-critique-of-your-original-sketch)
2. [Recommended Tech Stack](#2-recommended-tech-stack)
3. [High-Level Design](#3-high-level-design)
4. [Core Flows (Sequence Diagrams)](#4-core-flows-sequence-diagrams)
5. [Low-Level Design](#5-low-level-design)
6. [Code Execution Sandbox](#6-code-execution-sandbox-the-hard-part)
7. [Scaling Strategy](#7-scaling-strategy)
8. [Extra Features (Prioritized)](#8-extra-features-prioritized)
9. [Learning Roadmap](#9-learning-roadmap)
10. [Build Order / Milestones](#10-build-order--milestones)

---

## 1. Critique of Your Original Sketch

### What you got right (genuinely good instincts)
- **Room-based model** (Create Room / Join Room / Fill Room ID / Join via link) — correct mental model for isolating collaboration sessions.
- **You identified CRDT as the sync mechanism** and even noted "concurrent, works each other" — that's the core insight most beginners miss entirely.
- **Presence UX details** — random color assignment, short-name avatars, join/disconnect sound effects. These are small but recruiters notice polish like this in a demo.
- **Analytics hook** ("how much live code written") — shows product thinking beyond just "make it work."
- **Link-based join with shortener** — good UX instinct, reduces friction vs manual Room ID entry.

### Mistakes / gaps (in order of severity)

| Issue | Why it matters |
|---|---|
| **No code execution architecture** | You want "run code in browser" — this is the single most dangerous and most impressive part of the system. Arbitrary code execution needs sandboxing, resource limits, and timeouts, or it's a security hole. This wasn't in the sketch at all. |
| **No persistence layer distinction** | "Save that file" is ambiguous — save *where*? Locally (browser) disappears on refresh; you need server-side persistence (Postgres) with versioning for this to be portfolio-credible. |
| **CRDT box floats disconnected from data flow** | Your diagram shows CRDT near Output Bar but doesn't show *which* data structure syncs (the document text) vs what's ephemeral (cursor position). Yjs treats these very differently — `Y.Text` (persistent, synced) vs `Awareness` (ephemeral, not persisted). |
| **No reconnection / conflict handling** | What happens when a user's WebSocket drops mid-edit and reconnects? Your sketch has "User Discover → sound effect" but no state reconciliation logic. |
| **No auth / access control** | Room IDs are guessable strings — anyone with a link can join and edit. Fine for MVP, but you should at least *design* for view-only vs edit permissions to show you thought about it. |
| **No horizontal scaling consideration** | A single WebSocket server caps your "impressive" ceiling. Even a paragraph on Redis pub/sub scaling in your README shows senior-level thinking. |

### Bottom line
Your sketch is a solid **product flow diagram**, not yet a **system design**. The gap between the two — data models, API contracts, sync protocol, sandboxing, scaling — is exactly what separates "I built a project" from "I understand distributed systems," which is what recruiters are actually screening for.

---

## 2. Recommended Tech Stack

Since you're open to recommendations, here's what optimizes for **portfolio impressiveness + your existing skill overlap + reasonable build time**:

| Layer | Choice | Why |
|---|---|---|
| Frontend framework | **Next.js** | Matches your stack; SSR for landing/room pages, CSR for the editor itself |
| Code editor component | **Monaco Editor** | The actual VS Code editor engine — instant credibility, syntax highlighting, IntelliSense for free |
| CRDT sync | **Yjs** + `y-monaco` binding | Industry-standard CRDT lib; `y-monaco` gives you cursor + text sync out of the box |
| Real-time transport | **y-websocket** (Yjs's own protocol) — NOT Socket.IO | Yjs has its own optimized binary sync protocol. Socket.IO is great for chat/notifications but is the *wrong tool* for CRDT sync — don't mix them for the same channel |
| Presence / live cursors | **Yjs Awareness API** | Built for exactly this — ephemeral state (cursor pos, user color, name) that never touches persistent storage |
| Backend API | **Node.js + Express** | Room creation + execution job dispatch only — no auth, no CRUD |
| State storage | **In-memory (server RAM)** — `Map<roomId, RoomState>` | No DB needed. Room = live Y.Doc + participant list, held only while at least one person is connected |
| Code execution | **Piston** (open-source, self-hosted via Docker) | See Section 6 — sandboxes 40+ languages out of the box, no Judge0 setup overhead |
| Deployment | Vercel (frontend) + single Railway/Render instance (WS + API + Piston container) | One box is enough at this scale |

> **Key correction to your instinct:** Don't reuse your Socket.IO knowledge here for the *document sync* channel. Use Socket.IO (or plain events) only for secondary things like join/leave toasts and sound-effect triggers if you want. The CRDT document itself should sync over `y-websocket`.

> **Why no database:** rooms are ephemeral by design here — that's a legitimate, intentional architecture choice, not a shortcut. It means zero data-retention concerns, zero auth complexity, and a much smaller attack surface. Worth stating explicitly in your README as a design decision, not an omission — recruiters read "I chose not to add X" very differently from "I forgot X."

---

## 3. High-Level Design

```mermaid
flowchart TB
    subgraph Client["🖥️ Client (Next.js + Monaco + Yjs)"]
        UI[Editor UI]
        YDoc[Yjs Y.Doc<br/>local CRDT replica]
        Aware[Awareness<br/>cursor / presence]
    end

    subgraph Server["⚡ Single Node.js Server"]
        WS[y-websocket handler]
        Mem["In-memory room store<br/>Map&lt;roomId, RoomState&gt;<br/>— lives only while<br/>someone is connected"]
        ExecAPI[Execution dispatcher]
    end

    subgraph Exec["📦 Piston (Docker)"]
        Sandbox["Ephemeral sandboxed<br/>container per run"]
    end

    UI <--> YDoc
    UI <--> Aware
    YDoc <-->|binary CRDT updates| WS
    Aware <-->|cursor / join / leave| WS
    WS <--> Mem
    UI -->|POST /execute| ExecAPI
    ExecAPI --> Sandbox
    Sandbox -->|stdout / stderr / exit code| ExecAPI
    ExecAPI -->|result| UI
    UI -->|"Download as .cpp/.py/etc"| Local["User's local filesystem<br/>(browser download, not server save)"]

    Mem -.->|last participant leaves| Flush["Room flushed from memory"]

    style Client fill:#e8f0fe,stroke:#1a56db,color:#000
    style Server fill:#fef3e8,stroke:#c2410c,color:#000
    style Exec fill:#fee2e2,stroke:#b91c1c,color:#000
    style Mem fill:#fff7ed,stroke:#c2410c,color:#000
    style Local fill:#f0fdf4,stroke:#166534,color:#000
```

**Read it like this:** everything happens on one server. Two real-time channels leave the browser — the **Yjs document channel** (the actual code text, CRDT-merged) and the **Awareness channel** (ephemeral cursor/presence). Both live in a plain in-memory `Map` on the server, keyed by room ID — no database round-trips for editing. "Save" means the browser downloads the current buffer as a real file (`.cpp`, `.py`, whatever language is selected) via the [File System Access API](https://developer.mozilla.org/en-US/docs/Web/API/File_System_Access_API) or a simple `Blob` download — it never touches your server's disk. When the last participant disconnects, the room's entry is deleted from the `Map` and garbage collected. This is the right level of complexity for what you're building — explain this separation confidently, it's a real architectural choice.

---

## 4. Core Flows (Sequence Diagrams)

### 4.1 Create Room → Join Flow

```mermaid
sequenceDiagram
    actor A as User A (Host)
    participant API as API Service
    participant Mem as In-Memory Room Store
    participant WS as y-websocket Server
    actor B as User B (Joiner)

    A->>API: POST /rooms {language, name}
    API->>Mem: create RoomState {roomId, Y.Doc, participants:[]}
    API-->>A: {room_id, join_link}
    A->>WS: connect(room_id) as provider
    WS-->>A: sync Y.Doc (empty/initial)

    A->>B: shares join_link
    B->>API: GET /rooms/:id (check exists in Map)
    API-->>B: {room metadata} or 404 if flushed
    B->>WS: connect(room_id) as provider
    WS-->>B: sync Y.Doc (current in-memory state)
    WS-->>A: broadcast "user_joined" (awareness)
    Note over A,B: 🔊 join sound effect triggered client-side
```

### 4.2 Live Collaborative Editing (the CRDT core)

```mermaid
sequenceDiagram
    actor A as User A
    participant YA as Y.Doc (A's replica)
    participant WS as y-websocket Server
    participant YB as Y.Doc (B's replica)
    actor B as User B

    A->>YA: types "console.log()"
    YA->>YA: generate CRDT update (binary diff)
    YA->>WS: send update
    WS->>WS: merge into server-held Y.Doc
    WS->>YB: broadcast update
    YB->>YB: apply update (CRDT auto-merges,<br/>no conflict possible)
    YB-->>B: editor re-renders instantly

    par Meanwhile, cursors move independently
        A->>WS: awareness update (cursor pos, color)
        WS-->>B: broadcast awareness
        B->>B: render A's live cursor label
    end

    Note over A,B: No locking. No "last write wins."<br/>Both replicas mathematically converge<br/>to the same state — that's the CRDT guarantee.
```

### 4.3 Run Code (Sandboxed Execution)

```mermaid
sequenceDiagram
    actor U as User
    participant API as Execution API
    participant P as Piston (Docker)

    U->>API: POST /execute {code, language, room_id}
    API->>API: validate size limit + basic rate limit
    API->>P: POST /api/v2/execute {language, files:[{content:code}]}
    P->>P: spin up isolated container,<br/>run with CPU/memory/time caps
    alt success
        P-->>API: {run:{stdout, stderr, code}}
    else timeout or crash
        P-->>API: {signal: "SIGKILL"} or compile error
    end
    API-->>U: stream output to Output Bar
    Note over P: Container destroyed immediately after —<br/>Piston handles all sandboxing internally,<br/>you just call its API
```

### 4.4 Disconnect / Reconnect Handling

```mermaid
sequenceDiagram
    actor B as User B
    participant WS as y-websocket Server
    actor A as User A

    B--xWS: connection drops (network blip)
    WS-->>A: broadcast "user_left" (awareness cleared)
    Note over A: 🔊 disconnect sound + greyed avatar
    B->>WS: reconnect(room_id)
    WS-->>B: full Y.Doc state sync (catch-up)
    Note over B: CRDT guarantees B merges cleanly<br/>even after missing updates —<br/>no manual conflict resolution needed
    WS-->>A: broadcast "user_joined" again
```

---

## 5. Low-Level Design

### 5.1 Data Model (in-memory, TypeScript shape)

No database — this is the actual server-side state shape, held entirely in RAM:

```typescript
interface Participant {
  userId: string;        // random uuid, generated client-side on join
  name: string;           // short name they typed
  color: string;           // assigned randomly by server on join
  role: "HOST" | "EDITOR" | "VIEWER";
  socketId: string;         // current WS connection id
}

interface RoomState {
  roomId: string;             // short nanoid, e.g. "x7f9k2"
  language: string;            // selected on room creation
  yDoc: Y.Doc;                  // the live CRDT document
  awareness: Awareness;          // Yjs awareness instance (cursors etc.)
  participants: Map<string, Participant>;
  createdAt: number;
}

// The entire "database" of the app:
const rooms = new Map<string, RoomState>();
```

**Room lifecycle:** created on `POST /rooms` → lives in `rooms` Map → **deleted the moment `participants.size` hits 0** (on the last disconnect, after a short grace period of ~30s to survive brief refreshes/reconnects). No cron job, no expiry column — just a `setTimeout` cleanup check on disconnect.

**Saving code:** there is no server-side save. The client calls `Y.Text.toString()` on the current buffer and triggers a browser download (`Blob` + `<a download>`, or the File System Access API) named `main.<ext>` based on the selected language. This is genuinely simpler *and* more honest about what "save" means for a tool with no accounts.

### 5.2 REST API Contract

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/rooms` | Create room `{language, hostName}` → `{roomId, joinLink}` |
| `GET` | `/api/rooms/:id` | Check room still exists in memory (404 if flushed) |
| `POST` | `/api/execute` | `{roomId, code, language}` → proxies to Piston, returns `{stdout, stderr, exitCode}` directly (no job polling needed — Piston runs are fast enough to await) |

Saving/loading is entirely client-side (browser download/upload of a file) so it needs no API endpoint at all.

### 5.3 Real-Time Event Contract (y-websocket + Awareness)

| Channel | Payload | Persisted? |
|---|---|---|
| `doc-update` | Binary Yjs update (CRDT delta) | ✅ Yes (via periodic snapshot) |
| `awareness-update` | `{userId, name, color, cursorPos, selectionRange}` | ❌ No — memory only |
| `user-joined` (custom, via awareness) | `{userId, name, color}` | ❌ No |
| `user-left` | `{userId}` | ❌ No |

---

## 6. Code Execution Sandbox (the hard part)

This is what will make recruiters stop scrolling. It's also the part with real security risk if done carelessly.

```mermaid
flowchart LR
    A[User clicks Run] --> B{Rate limit<br/>check}
    B -->|blocked| X[429 Too Many Requests]
    B -->|ok| C[Enqueue job in Redis queue]
    C --> D[Worker picks up job]
    D --> E[Spin ephemeral Docker container]
    E --> F["Apply limits:<br/>• CPU: 0.5 core<br/>• Memory: 128MB<br/>• Timeout: 5s<br/>• No network access<br/>• Read-only filesystem"]
    F --> G[Run code inside container]
    G --> H{Result}
    H -->|success| I[Capture stdout]
    H -->|timeout| J[Kill container]
    H -->|crash| K[Capture stderr + exit code]
    I & J & K --> L[Destroy container<br/>no state persists]
    L --> M[Return result to client]

    style F fill:#fee2e2,stroke:#b91c1c,color:#000
    style E fill:#fef3e8,stroke:#c2410c,color:#000
    style L fill:#f0fdf4,stroke:#166534,color:#000
```

**Non-negotiable rules for this component:**
1. **No network access inside the sandbox** — otherwise someone runs code that calls out to attack other services.
2. **Hard timeout** (e.g. 5–10s) — otherwise someone runs `while(true){}` and exhausts your server.
3. **Memory/CPU caps** via Docker's `--memory` and `--cpus` flags — otherwise one execution starves the whole box.
4. **Ephemeral, single-use containers** — destroy after every run, never reuse a warm container between different users' code.
5. **Read-only root filesystem** — prevents writing malicious files that outlive the container.

**Your call — Piston — is a good one.** [Piston](https://github.com/engineer-man/piston) is purpose-built for exactly this: open source, self-hostable via Docker, supports 40+ languages out of the box, and already implements all five rules above internally (it's what runs code execution for the Engineer Man Discord and several online judges). You send it `{language, version, files: [{content: code}]}` and get back `{stdout, stderr, code}` — all the sandboxing complexity is abstracted away.

**Why this is the right call for your scope:** self-hosting Judge0 requires its own Postgres + Redis + Sidekiq worker setup — exactly the infra you just correctly decided you don't need. Piston runs as a single Docker container with no external dependencies, which matches an ephemeral, no-database app much better. You still get to *explain* container-level sandboxing in interviews (CPU/memory caps, no network, timeouts) — you're just not reinventing it from scratch.

---

## 7. Scaling — Deliberately Out of Scope (and why that's fine)

At this scope (ephemeral rooms, no accounts, in-memory state) a **single server instance is the right answer** — one process handles the WebSocket connections, the in-memory room `Map`, and proxies to Piston. Adding Redis pub/sub or multiple instances here would be solving a problem you don't have, at the cost of real complexity (cross-instance room-state coordination, sticky session config, another moving part to deploy and pay for).

**What to put in your README instead of building it:**

> "This runs on a single server instance, which is sufficient for the expected load. If this needed to scale beyond one instance, the standard pattern is Redis Pub/Sub: each server publishes CRDT doc-updates to a Redis channel keyed by room ID, and all instances subscribe, so users on different servers still see each other's edits in real time — the same pattern used for scaling Socket.IO horizontally."

That one paragraph demonstrates you understand the scaling story *without* you having to build and maintain infrastructure your app doesn't need yet. Recruiters care that you can reason about the tradeoff, not that you over-engineered a toy app.

---

## 8. Extra Features (Prioritized)

**Tier 1 — do these, high impact/effort ratio:**
- Multi-file / tabs support (most real editors aren't single-file)
- View-only vs Editor permission roles (you already modeled this in the schema above)
- Download/export code as file
- Room expiry + auto-cleanup (shows you think about resource hygiene)

**Tier 2 — strong differentiators if time allows:**
- Undo/redo across the session using Yjs's built-in `UndoManager` (in-memory, no DB needed)
- Room analytics dashboard (lines written, active time — you already sketched this! can be computed live from the in-memory Y.Doc, no storage required)
- Syntax-aware AI code assistant panel (ties into your AI agent learning track)

**Tier 3 — nice-to-have, lower priority:**
- Voice chat integration
- GitHub Gist import/export
- Themeable editor (dark/light/custom)

---

## 9. Learning Roadmap

Since you already understand CRDT vs OT conceptually, this roadmap skips theory and goes straight to implementation + production concerns, in build order:

1. **Yjs implementation** — `Y.Doc`, `Y.Text`, shared types, `y-websocket` provider setup, and critically the **Awareness protocol** (separate from doc sync). Build a minimal 2-browser-tab text sync demo before touching Monaco.
2. **Monaco Editor integration** — embedding it in React/Next.js, then wiring `y-monaco` for automatic cursor decoration and text binding. This is mostly configuration, not hard logic.
3. **WebSocket server internals** — write your own minimal `y-websocket` server (or study the reference implementation) so you can *explain* it, not just import it. Also design the in-memory `Map<roomId, RoomState>` and its cleanup-on-disconnect logic.
4. **Docker fundamentals** — containers, resource limits (`--memory`, `--cpus`), `--network none`. You'll need this to self-host Piston and to explain what it's doing under the hood, even though Piston does the heavy lifting for you.
5. **Piston's API** — read its docs, self-host it locally via `docker-compose`, understand its request/response shape and how it manages runtimes per language.
6. **Client-side file handling** — the `Blob` download pattern and/or the File System Access API for the "save as .cpp/.py" flow, plus basic file-upload-to-restore-a-session if you want that later.
7. **Security fundamentals for sandboxing** — resource exhaustion attacks, why `--network none` matters, rate limiting patterns on your `/execute` endpoint so one user can't hammer Piston.
8. **(Optional, conceptual only) Redis Pub/Sub** — you don't need to build this, but understanding *why* it's the standard horizontal-scaling pattern for WebSocket servers is worth 30 minutes of reading so you can speak to it in interviews.

---

## 10. Build Order / Milestones

```mermaid
flowchart LR
    M1["Milestone 1<br/>Room create/join<br/>+ basic Monaco editor"] --> M2["Milestone 2<br/>Yjs sync working<br/>(2 tabs edit same doc)"]
    M2 --> M3["Milestone 3<br/>Live cursors +<br/>presence bar"]
    M3 --> M4["Milestone 4<br/>Room cleanup on<br/>disconnect + save as file"]
    M4 --> M5["Milestone 5<br/>Code execution<br/>via Piston"]
    M5 --> M6["Milestone 6<br/>Polish: sounds,<br/>analytics, permissions"]

    style M1 fill:#e8f0fe,stroke:#1a56db,color:#000
    style M2 fill:#fef3e8,stroke:#c2410c,color:#000
    style M3 fill:#fef3e8,stroke:#c2410c,color:#000
    style M4 fill:#f0fdf4,stroke:#166534,color:#000
    style M5 fill:#fee2e2,stroke:#b91c1c,color:#000
    style M6 fill:#e8f0fe,stroke:#1a56db,color:#000
```

**Why this order:** get sync working with plain text first (Milestone 2) before adding Monaco's complexity, so you can debug CRDT behavior in isolation. Save execution (the riskiest, most complex piece) for after the collaborative core is solid — that way your portfolio demo works end-to-end even if you run out of time before finishing Milestone 5.

---

*This document is designed to be reused as a reference for future real-time collaborative projects, not just this one.*
