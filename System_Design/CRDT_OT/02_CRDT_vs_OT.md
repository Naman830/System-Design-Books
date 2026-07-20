# OT vs CRDT, Part 2: The Concepts We Skipped

This is a follow-up to *"OT vs CRDT: A Complete, Beginner-Friendly Guide."* Part 1 covered the core problem, the basic "why not just X" questions, and how each algorithm works at a high level. This part goes one layer deeper — into the machinery that makes both algorithms provably correct, and a few practical concerns you'll hit once you actually build a real-time app.

You don't need everything in this document to *use* a library like Yjs. But understanding it will make you a much stronger engineer when things go wrong, or when you need to explain a design decision to someone else.

---

## 1. Causality: how do you know what happened "before" what?

Both OT and CRDT need to answer a subtle question: when two edits arrive, were they truly simultaneous ("concurrent"), or did one happen *after* the person had already seen the other?

This matters because the correct merge behavior is different in each case. If your friend's edit came after seeing yours, it should probably be treated as building on top of yours. If it happened without ever seeing yours, both should be treated as independent and preserved.

### Logical clocks / vector clocks

Instead of using real-world timestamps (which are unreliable across different devices and clocks), collaborative systems use **logical clocks** — a small counter per user that tracks "how many of my own edits, plus how many of yours, have I seen so far."

Each user keeps a pair like `(my_count, their_count)`. When an edit's clock fully contains another edit's clock, you know it happened causally after it. When neither clock contains the other, the edits are concurrent.

This is the mechanism illustrated in the causality diagram above: edit A `(1,0)` and edit B `(0,1)` are concurrent — neither user had seen the other's edit. Edit C `(2,1)` came *after* seeing B, because its clock already includes B's count.

**Why this matters practically:** this is the underlying reason CRDT and OT systems can say with certainty "these two edits definitely conflict" vs "these two edits are unrelated," instead of guessing based on arrival time at the server (which can be misleading due to network delays).

---

## 2. Tombstones: what actually happens when you delete something

A natural question: if CRDT assigns every character a permanent, unique ID, what happens when you *delete* a character? Wouldn't removing it break the chain for anyone still referencing it?

This is exactly the problem, and CRDT solves it by **never actually removing the node** — it's just marked dead, called a **tombstone**. The structure diagram above shows this: the letter "l" is deleted, but instead of vanishing, it stays in place as an invisible, inactive node. This matters because another user's edit might reference "insert this new character right after the 'l'" — if the node were truly gone, that reference would have nowhere to attach.

### The tradeoff: memory growth

Because tombstones stick around forever by default, a document that has been heavily edited and deleted over months or years can accumulate a lot of dead nodes, bloating memory usage even though the visible text is short.

Modern CRDT libraries (Yjs, Automerge) have specific strategies to deal with this — for example, periodically compacting tombstones once every connected device has confirmed it no longer needs them, or using more memory-efficient internal representations than a naive linked list. As an app builder, you generally don't have to implement this yourself, but it's worth knowing it's happening under the hood, especially if you're ever debugging unexpected memory growth in a long-lived document.

---

## 3. Two flavors of CRDT you'll see mentioned: state-based vs operation-based

When reading about CRDTs, you'll often see the terms **CvRDT** and **CmRDT**. They're two different strategies for syncing changes between devices:

- **State-based (CvRDT — "Convergent" RDT):** Each device periodically sends its *entire current state* to others, and a merge function combines any two states into one. Simple to reason about, but can be wasteful for large documents since you're sending the whole thing repeatedly.
- **Operation-based (CmRDT — "Commutative" RDT):** Each device sends only the *individual operations* (like "insert this character here"), and those operations are designed so that applying them in any order gives the same result. This is far more bandwidth-efficient for text editing, and it's what most real-time text-editing CRDT libraries (like Yjs) actually use.

**Practical takeaway:** you don't need to choose between these yourself — the library you pick (Yjs, Automerge, etc.) has already made this decision. But if you read a CRDT paper or a library's documentation and see "CvRDT" or "CmRDT," now you know what it means.

---

## 4. OT's formal correctness rules (good to recognize, not to memorize)

Part 1 mentioned that OT's transform functions are notoriously hard to get right. The formal computer science literature defines two properties any correct transform function must satisfy:

- **TP1 (Transformation Property 1):** Applying operation A then the transformed operation B must give the same result as applying operation B then the transformed operation A — no matter which order the two operations actually arrive in.
- **TP2 (Transformation Property 2):** This matters when there are *three or more* concurrent operations and a third operation needs to be transformed against two others — the result must be consistent no matter what order those transformations happen in.

You will likely never need to implement or prove these yourself if you're using an existing library. But if you ever read about an OT bug in a real system, it's almost always a violation of TP1 or TP2 that someone missed in an edge case — which is precisely why OT libraries are harder to trust than well-tested CRDT libraries today.

---

## 5. Presence and awareness: showing who's editing what

This isn't strictly a conflict-resolution problem, but it's something every real-time collaborative app needs and is usually bundled with the same libraries: showing **other users' live cursors, selections, and "who's currently online."**

This is typically called **awareness** (Yjs has a dedicated `Awareness` API for exactly this). It works differently from the actual document data — awareness information is usually treated as *ephemeral* (temporary, not persisted), since you don't need a permanent history of where someone's cursor was five minutes ago, only where it is right now.

**Practical takeaway:** when you're evaluating a CRDT library, check whether it includes awareness/presence support out of the box — Yjs does, which is part of why it's a popular default choice.

---

## 6. Undo/redo is harder than it looks

In a single-user app, undo/redo is simple: keep a stack of past states, pop off the top to undo. In a collaborative app, this breaks down, because:

- If you undo your last edit, but your friend has since typed *after* your edit, what should happen? Undoing your edit might shift text your friend typed on top of it.
- A naive "undo the last operation" can end up undoing someone else's work if operations are interleaved in a shared history.

The generally accepted solution is **selective undo** — instead of "undo the last thing that happened," it's "undo the last thing *I* did, regardless of what anyone else did in between," achieved by tracking each operation's origin and applying an inverse operation targeted specifically at your own edits. This is a solved problem in mature libraries (Yjs has a dedicated `UndoManager`), but it's a common surprise for engineers who assume undo will "just work" the same way it does in a single-user text editor.

---

## Summary

| Concept | What it solves |
|---|---|
| Logical / vector clocks | Knowing whether two edits are concurrent or causally related |
| Tombstones | Letting deleted content stay referenceable without literally existing |
| CvRDT vs CmRDT | Two strategies for how devices sync state (whole-state vs operations-only) |
| TP1 / TP2 | The formal rules an OT transform function must satisfy to be correct |
| Awareness / presence | Showing live cursors and who's currently online, separate from document data |
| Selective undo | Undoing only your own edits without breaking or reverting a collaborator's work |

Together with Part 1, this gives you both the intuition for *why* OT and CRDT exist, and the deeper mechanics of *how* they're actually implemented in production-grade libraries like Yjs and Automerge.
