# OT vs CRDT: A Complete, Beginner-Friendly Guide

This document explains why real-time collaborative apps (like Google Docs, Figma, or Notion) need special algorithms to handle multiple people editing at once — and walks through the two main solutions: **Operational Transformation (OT)** and **Conflict-free Replicated Data Types (CRDT)**.

By the end, you'll understand the problem, why "obvious" solutions don't work, how each real solution works, and which one to pick for your own project.

---

## 1. The problem we're actually solving

Imagine a shared document. Two people are editing it at the same time:

- You type "X" at position 5.
- Your friend types "Y" at position 5.
- Neither of you knows about the other's edit until half a second later.

Now both of your apps have a different idea of what the document looks like. Someone — or something — has to decide the final, agreed-upon result, and every device involved (yours, your friend's, and the server) must end up with the **exact same document**.

This is called the **conflict resolution problem** in real-time collaboration. Everything in this document is about solving it.

---

## 2. "Why can't we just use X?" — basic questions answered

### Why not just use WebSockets?

WebSockets are a **transport mechanism** — they let messages travel between client and server instantly, in both directions, without the delay of repeatedly opening new connections (like regular HTTP requests would require).

But a WebSocket is just a pipe. It doesn't know anything about:
- What order edits *should* be applied in
- How to merge two edits that touched the same part of a document
- What to do when two people type at the same position at the same time

**WebSockets solve the "how do I send data instantly" problem. OT and CRDT solve the "what do I do when two edits collide" problem.** You typically use WebSockets (or WebRTC) *as the delivery pipe*, and OT/CRDT logic *on top of it* to actually merge the data correctly.

### Why not just "last write wins" (whoever's edit arrives last overwrites the rest)?

Simple to build, but it silently **destroys data**. If you and your friend both typed different things, one of you would just lose your work with no warning. Fine for something like a single settings toggle, terrible for collaborative text or documents.

### Why not lock the document while someone is editing (like old-school file locking)?

This is called **pessimistic locking** — only one person can edit at a time, everyone else is blocked or read-only. It technically avoids conflicts entirely, but it kills the entire point of *real-time* collaboration. Nobody wants to wait for their coworker to "give up the pen." This is basically how old shared Word documents on a network drive worked, and it's exactly the pain point Google Docs was invented to fix.

### Why not just diff and merge, like Git does?

Git's merge model works because it assumes changes happen **offline, in batches**, and a human is available to manually resolve conflicts when they overlap. That's fine for code commits made hours apart. It falls apart for:
- **Real-time character-by-character editing** — you can't ask a human to resolve a merge conflict every time two people type in the same sentence.
- **No acceptable "conflict marker" experience** — nobody wants `<<<<<<< HEAD` markers appearing mid-sentence in a live document.

Git-style merging is great for infrequent, coarse-grained, human-reviewed changes. OT and CRDT are built for continuous, fine-grained, automatic changes.

### So what do OT and CRDT actually add that these don't?

They give you a **mathematical guarantee of convergence** — a formal promise that no matter what order edits arrive in, no matter who's offline, no matter how many people are editing at once, everyone's copy of the document will end up **identical**, automatically, with no data loss and no human intervention.

That guarantee is the entire value proposition. Everything below is about *how* each approach delivers it.

---

## 3. Operational Transformation (OT)

### The intuition

Think of OT like a translator sitting between two people passing notes. When two edits collide, OT doesn't discard either one — it **mathematically rewrites (transforms)** one operation so it still makes sense after the other has already been applied.

### Walkthrough

1. You insert "X" at position 5.
2. Your friend, at the same moment, inserts "Y" at position 5.
3. The server receives your edit first and applies it. The document has now shifted — position 5 doesn't mean what it used to.
4. Before applying your friend's edit, the server **transforms** it: "insert Y at position 5" becomes "insert Y at position 6," because your "X" now occupies the old position 5.
5. Both documents converge to the same final text.

### Key properties

- Needs a **central server** to decide the official order of operations.
- Every possible pair of operation types (insert-insert, insert-delete, delete-delete, etc.) needs its own carefully proven transform function.
- Historically the algorithm behind early Google Docs and Google Wave.

### Good-to-know (not essential, but useful context)

Writing correct transform functions is famously difficult — there are many subtle edge cases, and getting them wrong causes documents to silently diverge between users (a bug class that plagued early collaborative editors for years). This difficulty is a big part of why OT has fallen out of favor for new projects.

---

## 4. CRDT (Conflict-free Replicated Data Type)

### The intuition

CRDT takes the opposite philosophy: **never transform anything, never argue about "position 5."** Instead, every single character (or piece of data) gets a permanent, unique ID the moment it's created — something like a timestamp combined with a user ID.

Edits don't say "insert at position 5." They say "insert this character right after the character with ID `u1-104`." Because that anchor never changes, the edit never needs to be rewritten later — no matter what order it's applied in.

### Walkthrough

1. You insert "X" with ID `u1-104`.
2. Your friend, at the same time, inserts "Y" with ID `u2-098`.
3. Both edits get applied **whenever they arrive**, in any order, on any device.
4. Because the IDs are unique and the anchors don't move, there's a well-defined mathematical rule for how to order same-position insertions consistently everywhere.
5. Every device ends up with the identical document — with no server required to referee.

### Key properties

- No central server needed to resolve conflicts (though you may still use one to relay messages).
- Works naturally offline — edits made while disconnected merge in cleanly once you reconnect.
- Powers modern tools like **Figma**, **Linear**, and libraries like **Yjs** and **Automerge**.

### Good-to-know (not essential, but useful context)

Deleted content is usually kept internally as an invisible "tombstone" (marked dead, not actually removed) rather than deleted outright — this is part of why naive CRDT implementations can bloat in memory over a long-lived document. Modern CRDT libraries have specific optimizations to keep this bloat under control, so you generally don't need to worry about it as an app builder.

---

## 5. Side-by-side comparison

| | OT | CRDT |
|---|---|---|
| Who resolves conflicts | A central server, using transform functions | Math built directly into the data structure |
| Needs a server to resolve conflicts? | Yes | No — can work peer-to-peer |
| Offline editing support | Difficult to get right | Naturally excellent |
| Where the complexity lives | Transform functions (many edge cases) | The data structure and merge algorithm |
| Real-world examples | Early Google Docs, Google Wave | Figma, Linear, Yjs, Automerge |
| Maturity of available libraries today | Rare, harder to find well-maintained | Common, actively maintained, well-documented |

---

## 6. Which one should you actually use?

**For almost any new real-time collaborative app being built today, CRDT is the practical choice.** Here's why:

- Nobody hand-writes OT or CRDT algorithms from scratch anymore — both are research-level hard to get perfectly correct. You use a battle-tested library either way.
- CRDT libraries (**Yjs**, **Automerge**) are actively maintained, well-documented, and battle-tested in production by companies like Figma and Linear.
- CRDT gives you offline support and peer-to-peer sync almost for free — valuable even if you don't need it on day one.
- Good OT libraries are much rarer today and harder to find well-maintained.

**Practical setup:** use a WebSocket (or WebRTC) connection as your transport layer to send changes instantly between clients, and use a CRDT library like Yjs on top of it to actually merge those changes correctly. The WebSocket is the pipe; the CRDT is the brain that knows how to merge what flows through it.

---

## 7. One more concept worth knowing (optional, deeper)

Both OT and CRDT ultimately depend on tracking **causality** — knowing whether one edit happened *before* another, or whether they were truly simultaneous ("concurrent"). This is often implemented using **vector clocks** or **logical clocks**, a way of timestamping operations that captures cause-and-effect relationships, not just wall-clock time.

You don't need to implement this yourself if you're using Yjs or Automerge — it's handled internally — but knowing it exists helps you understand *why* convergence is a provable guarantee rather than something that just "usually works."

---

## Summary in one sentence

**WebSockets deliver the messages instantly; OT and CRDT decide how to merge them correctly — and for a real-time app built today, CRDT (via a library like Yjs) is almost always the right choice.**
