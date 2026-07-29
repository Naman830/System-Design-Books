# Five Projects

Five open-source projects, five different domains. Each was chosen because it sits in a gap where the problem is real, the engineering is deep, and the thing genuinely has not been built — not because it is a harder version of something common.

**How these were chosen.** 24 candidates were generated across six domains. Every one was then independently investigated for prior art — existing products, GitHub repos, npm packages, Show HN, academic literature — and audited for feasibility. Nine strong candidates were rejected on evidence. That reasoning is kept at the end of this document, because knowing *why* an idea dies is most of the value.

**Constraints applied.** JavaScript/TypeScript only, no exceptions. Solo developer with AI coding assistance. Open source and genuinely adoptable. Must solve a real problem for real users. Skills to showcase: real-time media & networking, distributed systems & data, low-level systems.

---

## The Five

| # | Project | Domain | Core technology |
|---|---------|--------|-----------------|
| 1 | **Flightbox** | Real-time media | WebRTC `RTCRtpScriptTransform`, SharedArrayBuffer, OPFS, WebCodecs |
| 2 | **Limbo** | Developer tooling | V8 heap snapshots, CDP, columnar graph engine |
| 3 | **Phaselock** | Scientific instrumentation | AudioWorklet, Kalman filtering, DSP, DataChannel |
| 4 | **Firmament** | Security & verifiability | WebSerial, Merkle transparency logs |
| 5 | **Causa** | Distributed data / local-first | CRDT causal DAGs, delta debugging |

Exactly one — **Flightbox** — has WebRTC as its irreducible core. Phaselock uses DataChannels as transport but would work equally over WebTransport, so it does not double-count.

Next.js is the UI surface on all five (Flightbox viewer, Limbo trace explorer, Phaselock dashboard, Firmament fleet view, Causa DAG panel). In every case the *engine* is the portfolio artifact and Next.js is the window onto it. That is the right ratio.

---

## The through-line

Put a version of this in the README of every repo.

**One engineer who builds instruments for systems that cannot currently be questioned.**

A video call that only reports `freezeCount=1`. A Node process that is hung and will not say why. Two devices that cannot agree what time it is. A chip that will not say what code it is running. A CRDT that merged and will not say how.

Each project takes something the industry has quietly agreed is unknowable, builds the apparatus that makes it observable, and then ships an explicit **"what this does not tell you"** section. That last part is the signature. It is what separates a senior engineer from someone with a good demo, and it reads as strength to all three audiences at once — the infra reviewer who will look for the limitation you didn't mention, the CTO who has been burned by tools that overclaim, and the OSS user deciding whether to trust your output.

---

## 1. Flightbox — the missing capture stage for WebRTC media forensics

**Domain:** real-time media · **This is the WebRTC project.**

### The problem

A user reports "the video froze." The engineer opens the dashboard and finds `freezeCount: 1`. That is the entire evidence trail.

libwebrtc ships `video_replay` and `neteq_rtpplay` for *analyzing* a captured RTP stream — mature, trusted tools that media engineers already use. But there is no way to capture that stream from inside a real browser in production. The analysis stage exists. The capture stage has been missing for a decade.

### The framing

Do not pitch this as a "replayer." Pitch it as:

> An always-on, bounded, production-safe rolling recorder of the **encoded** receive path that costs nothing until the user clicks "report."

Its primary output is a **causal decode timeline**:

> *frame 8814 referenced 8809, which never arrived; 43 frames undecodable until the keyframe at 8858.*

Exported as rtpdump/IVF into the tooling media engineers already trust. Pixel replay is a closing flourish, not the claim.

### The hardest problem

**Proving the tap is free.** An allocation-free hot path in a garbage-collected language, touching every encoded frame — with V8 CPU profiles and p99 frame-arrival histograms demonstrating no change to end-to-end latency or drop distribution at 1080p60.

Plus: correct behaviour when `frameId` spaces discontinuously change under simulcast layer switches and mid-call renegotiation, and graceful degradation when OPFS stalls (it does, on Windows) so that a slow disk degrades the *recording*, never the *call*.

### v1 scope

- Receive-side `RTCRtpScriptTransform` only.
- Lock-free SPSC ring buffer in `SharedArrayBuffer`, one copy into a preallocated slab, explicit overrun policy.
- Writer worker appending to a chunk-indexed, CRC'd, append-then-commit `.rtcbb` container in OPFS, with a fixed byte budget and O(1) eviction.
- Offline analyzer building the causal decode timeline from `frameId` gaps and the dependency descriptor.
- Exporter to rtpdump / IVF.
- Viewer with a software `WebCodecs` decoder and a pinned stats timeline.
- **Ship the overhead benchmark harness as part of the repo.**

### What makes it adoptable

A ~20-line integration that works against unmodified mediasoup, LiveKit, and Janus. Bounded, budgeted, published cost — that is what gets it through the production review. Encrypted at rest with a user-held key, so a privacy team can sign off. And because it emits artifacts the existing `video_replay` toolchain already consumes, it is additive to an RTC team's workflow rather than a rival viewer they must switch to.

### Biggest risk

"Pixel-for-pixel identical" is the shakiest claim in the pitch and is probably false: `VideoDecoder` generally errors and closes on chunks with missing references rather than emitting matching artifacts, and hardware decoders conceal errors differently from software ones.

**Mitigation:** make the causal timeline the guaranteed, always-correct headline. Ship pixel replay clearly labelled best-effort against a software decoder you control. Demo artifact reproduction, not a claim of visual identity.

---

## 2. Limbo — `why-is-node-running` for the async era

**Domain:** developer tooling

### The problem

A Node process is hung in production. `why-is-node-running` tells you which *handles* are open — but not which `await` is unsettled, and it must be loaded before the process starts, which means you needed to predict the outage.

Node core closed the `--report-pending-promises` request as not-planned. So for the runtime whose defining feature is asynchrony, "why is this hanging?" remains unanswerable.

### The framing

Attach to an **already-running, un-instrumented** process and get the complete **await-DAG** of every unsettled promise, with a stated fidelity tier per Node version. Zero cost until the moment you need it.

Be precise about the contribution: it is exclusively the *promise-graph semantic layer*. The streaming columnar heapsnapshot parser is plumbing that memlab and others have already built, and the README should say so plainly.

### The hardest problem

Reconstructing "promise P is awaited by async function F" from an undocumented, version-drifting V8 promise-reaction and async-continuation edge layout — then resolving each pending promise to a source position through `SharedFunctionInfo` → script → source map, when optimized and inlined async frames may have lost the association entirely.

### v1 scope

**Build the self-validating corpus first, before the parser.** Programs with analytically known await topology, snapshotted, with an assertion that the reconstructed graph matches exactly. This ordering is not optional — it is the only thing that keeps the project honest.

Then a hybrid acquisition path:
- CDP `Runtime.queryObjects(Promise)` + `getProperties` for enumeration and `[[PromiseState]]` — documented, stable, works on every Node version. This is the always-works floor.
- Heap-snapshot edge mining used **only** for the awaiter/awaitee edges.
- Pin support to two Node LTS lines.
- **Refuse to run loudly** on an unrecognized layout, degrading to enumeration-only mode rather than emitting a wrong graph.
- Cycle detection and root ranking. `npx limbo attach <pid>`.

### What makes it adoptable

The install cost is literally zero — nothing to add in advance, no flags, no restart, no permanent overhead. That is the entire pitch and it is unbeatable against every existing option. It slots into the emotional position `why-is-node-running` already occupies, with a large installed base and a well-known gap.

### Biggest risk

Partial recoverability. A 70%-complete graph that confidently points at the wrong promise is strictly worse than no tool at all, and every Node minor threatens the layout.

**Mitigation:** the corpus-first discipline above, plus explicit fidelity tiers printed on every run ("enumeration: exact; await edges: verified on this V8 build"), plus `queryObjects` as the always-works floor. This converts a project-ending risk into a documented degradation ladder — and the honesty is itself the hiring signal.

---

## 3. Phaselock — "browser LSL": a zero-install coherent-sampling fabric

**Domain:** scientific instrumentation

### The problem

Multi-device synchronized recording — acoustics, biomechanics, EEG and behavioural labs, field measurement — requires either expensive hardware-clocked gear or Lab Streaming Layer. LSL needs installation, and explicitly only *timestamps* streams, leaving the researcher to re-align them afterwards. Two ordinary laptops cannot be made into one instrument.

### The framing

Weld N ordinary devices into one **rigidly sample-aligned** instrument, solving the piece LSL explicitly punts on: **continuous drift absorption, so every stream lands on one virtual sample grid.**

The headline contribution is the **trustworthiness monitor**. Every alignment carries a covariance and can *hard-invalidate itself* when the OS changes the audio route underneath you. Every prior work reports accuracy on curated data; none ships self-invalidation. That is the unoccupied ground.

### The hardest problem

**Identifiability.** Network offset, sample-clock skew, and constant hardware I/O latency are entangled in every observation. The latency term is large (5–80 ms), device-specific, and changes mid-session when the OS switches audio paths.

A reciprocal chirp exchange breaks the entanglement — but only with sub-sample chirp arrival detection in a reverberant room, and only if you can detect the invalidating event *before* it produces a confidently wrong number.

### v1 scope

Two devices.

- Kalman offset/skew estimator over min-filtered RTT probes.
- Sample-clock ppm regression against AudioWorklet render-quantum arrival.
- **Farrow fractional-delay polyphase resampler** absorbing drift with no sample slip.
- Reciprocal chirp handshake for latency identification and pairwise distance.
- **GCC-PHAT** with phase-slope sub-sample interpolation.
- Zero-allocation AudioWorklet discipline with SharedArrayBuffer rings.
- Defer WebGPU. Eight channel-pairs at 48 kHz does not need it, and it would drag in WGSL.

**Ship three standalone npm packages:** `gcc-phat`, a Farrow resampler, and a sample-rate-offset estimator. The JS DSP ecosystem verifiably has none of them. They are citable output independent of whether the product itself lands.

### What makes it adoptable

Bill of materials: two laptops and a tape measure. Zero install, cross-platform, and it reaches LSL's 150-device ecosystem without asking anyone to compile anything.

### Biggest risk

`getUserMedia` applies AGC, echo cancellation, and noise suppression that destroy phase coherence — and the constraints to disable them are silently ignored or only partially applied on several platforms. The exact signal the project depends on may be unrecoverable on consumer microphones.

**Mitigation: run that measurement in week one, not week ten.** Publish the per-device/per-browser accuracy envelope as a dataset regardless of the result. **Verify — do not assume — that processing is disabled, and refuse to report when you cannot confirm it.** If phones turn out to be hopeless, laptops plus cheap USB audio interfaces still make the instrument real. Having measured and published the failure is a stronger artifact than having quietly avoided it.

---

## 4. Firmament — fleet integrity for open-firmware deployments

**Domain:** security & verifiability

### The problem

Deliberately reframed away from anti-implant paranoia, which is a threat model this tool cannot honestly serve:

> You have 200 ESPHome / Tasmota / Meshtastic nodes in the field. Prove every one is running the build you think you shipped, and show me the three that drifted.

Today there is no dignified answer. This is a weekly, non-adversarial, genuinely felt pain.

### The framing

Plug the device in, open a tab, read the firmware back off the silicon over **WebSerial**, canonicalize away the per-device regions, and check the hash against a public, **rebuilder-attested transparency log** — with inclusion and consistency proofs verified in-page.

### The hardest problem

**Canonicalization.** Two honest devices running an identical release must hash identically despite NVS/config/calibration blobs, per-device serials, wear-levelling, A/B OTA slot layout, and bootloader-written metadata.

The learning-by-diffing approach has a nasty property: mask too much and an attacker hides inside the mask. **Every mask is a security decision and must be reviewable as one.** Build the review affordance into the tool, not the docs.

### v1 scope

One device family — ESP32 over WebSerial, reusing transport patterns proven by `esptool-js`.

- Image-format parsing.
- A region-mask DSL compiled into a deterministic hashing plan.
- Streaming hash of multi-MB reads with proper buffer pooling.
- A tile-based TypeScript transparency log, **pre-populated from public CI artifacts** of projects that already publish reproducible releases.
- In-browser inclusion and consistency proof verification.
- A byte-range divergence map, so "red" arrives with *"sector 0x3E1000–0x3E2000 differs, and this image appears in no log."*
- Fleet view second.
- **Cut the signed-attestation stub for readout-protected parts entirely** — no such artifact exists for any real device family, and building one means writing C.

### What makes it adoptable

No install, no vendor cooperation, no account. The log ships non-empty because you populate it yourself from public CI, which kills the cold-start problem that would otherwise make the whole thing verify nothing on day one.

### Biggest risk

The threat-model hole: on the devices where compromise is most plausible, read-back is fused off, or is answered by the very firmware under test.

**Mitigation:** put that in the first section of the README, above the features, in plain language — *"against secure-boot/RDP-fused parts, and against an implant that replays a clean image, this tool proves nothing."* Then scope the product to the fleet-drift case, where the adversary is a bad OTA rather than an attacker, and the guarantee is fully sound. A tool that is precise about its own limits reads as senior engineering; one that overclaims reads as the opposite, and infra interviewers test exactly this.

---

## 5. Causa — "CRDT blame"

**Domain:** distributed data / local-first

*(Named `Causa` rather than `Antichain` — that term is load-bearing in the timely/differential-dataflow world and would collide in search with Materialize's ecosystem.)*

### The problem

A collaborative document merges into something wrong. The library insists the merge was correct. Nobody can explain *why the text ended up like that*, so the bug report dies in an argument between the app team and the CRDT maintainer.

### The framing

**Lead with the explainer, not the minimizer.**

Point at a character or a field in a broken document and get the **ordering decision that produced it**: the two competing operations, their origins and timestamps, and the concrete tie-break rule that chose the winner.

Then causal diff between two disagreeing replicas. Then — and only if the performance work lands — the lattice minimizer.

### The hardest problem

Dependency-aware delta debugging generalized from sequences to the **lattice of downward-closed subsets of a causal DAG**, with 1-minimality under a non-monotone predicate.

Every candidate cut must be causally closed, which invalidates ddmin's complement step. Combine that with an incremental merge cache keyed on frontier hash, able to derive `state(cut)` from a *neighbouring* cut instead of replaying from zero.

### v1 scope

**Yjs only.**

- `causa explain` as a devtools panel and a CLI, deriving explanations from Yjs's *actual* integration path rather than a reimplementation of YATA.
- Causal diff between two replica states.
- A canvas DAG viewer over causal history extracted from update binaries.
- Export any reduced case as a byte-sized regression test.
- Automerge and Loro are explicit stretch goals — say why in the README.

### What makes it adoptable

It runs on an artifact — a `.bin` from a bug report — with no SDK, no instrumentation, and no cooperation from the app. That is near-zero friction for every team shipping Yjs.

And it directly serves its own worst case. The most common honest answer is *"the merge was correct; your `insertAt` index was stale."* The explainer is precisely the tool that **proves** that, specifically and satisfyingly, instead of leaving the maintainer to argue it.

### Biggest risk

The explanation silently diverging from what the library actually did — fatal for a tool whose entire product is "trust my explanation." Plus the minimizer failing to terminate on the large histories that justify its existence.

**Mitigation:** derive explanations by instrumenting Yjs's real ordering decisions rather than reimplementing YATA. Differential-test every explanation against the actually materialized document. Ship explain + causal diff first, so the project is useful before the minimizer is fast, and treat the concurrency-directed heuristic as an experiment with published numbers rather than a promise.

---

## Suggested build order

**1. Flightbox.** The WebRTC flagship and the best demo. Build the overhead benchmark harness *before* the optimization — the numbers are the deliverable, and you cannot optimize what you have not instrumented.

**2. Causa.** The smallest credible v1 of the five, and it converts CRDT/OT study into a tool. Take this first instead if you want an early shipped win before committing to Flightbox.

**3. Limbo.** Highest real-world value in the set. Corpus first, parser second, no exceptions.

**4. Phaselock.** Run the `getUserMedia` phase-coherence measurement in week one. It is a genuine go/no-go, and finding out in week ten would be expensive.

**5. Firmament.** Needs a ~$5 ESP32. Write the threat-model limitations section before writing any feature.

---

## What makes these portfolio-grade rather than merely large

Each ships an artifact a skeptic can check **without trusting you**:

| Project | The verifiable claim |
|---|---|
| **Flightbox** | Published p99 frame-arrival histograms and V8 CPU profiles showing the tap is free at 1080p60 |
| **Limbo** | A self-validating corpus with analytically known await topology, asserted exactly |
| **Phaselock** | A per-device/per-browser accuracy envelope published as a dataset, including the failures |
| **Firmament** | A transparency-log inclusion proof anyone can verify independently, in-page |
| **Causa** | Every explanation differential-tested against the actually materialized document |

Write a short design document for each *before* the code, and keep a decision log as you go. For an audience of infra reviewers, founders, and OSS users equally, the design doc is usually what gets read first — and it is the cheapest thing on this list to do well.

Extract reusable pieces as separate npm packages with their own tests and READMEs: a lock-free SPSC ring, `gcc-phat`, a Farrow resampler, a sample-rate-offset estimator, a tile-based transparency-log verifier. Five small packages people actually depend on is a stronger signal than five large repositories people only star.

---

## Notable rejections

Nine strong candidates were dropped. The reasoning is worth keeping, because "I considered this and here is the evidence against it" is itself a portfolio signal.

**Leverage** — causal profiling for Node. The highest raw novelty score in the entire set: `coz`'s own README states it cannot support JIT or interpreted languages, which is unusually strong evidence of an unoccupied niche. Rejected anyway, for three compounding reasons. For CPU-bound single-threaded JS, throughput leverage ≈ self-time fraction, so the tool *reproduces the flame graph by construction* in the common case. JS has no preemption, so delay injection is only possible at yield points, making the virtual-speedup model approximate in a way `coz`'s is not. And it requires a sustained representative load rig that almost no OSS user has.

**Lockstep / Groundhog** — deterministic simulation testing for JavaScript. **Rejected on direct evidence.** The space was colonized during 2026 by `chronos` and `glideapps/determined`, and Antithesis auto-instruments Node at the hypervisor level, which is strictly stronger than anything shimmable from userland. Building the fourth or fifth framework and landing behind on features while still being 98% deterministic is worse than useless, since flaky replay destroys trust in the one property being sold. (The salvageable piece — a linearizability checker, which genuinely does not exist in JS — is a component, not a project.)

**Understudy** — capture a real user's network personality from the browser and replay it deterministically as a TURN relay. A good idea, but only one WebRTC slot exists and Flightbox is stronger. Its capture half is academically mature (Mahimahi 2015, ERRANT 2021, NeuralEmu 2026), the getStats-inversion story is an identifiability problem you lose to anyone holding a packet capture, and its fidelity validator — "replay reproduces the original getStats distributions" — is circular in a way a reviewer spots in ninety seconds.

**Revenant** — CI for web archives. Browsertrix QA shipped the cheap 80% (screenshot, text, and resource-status diffing on re-replay) in 2024. More seriously, "deterministic replay" must be honestly downgraded to "entropy-pinned replay with a tolerance relation," because you cannot control fetch settlement, font shaping, GC, or JIT timing from JS — and once downgraded, the headline is much weaker than it reads.

**Retrace** — vector-accurate magnification for canvas/WebGL content. Painful to drop: the crossing between graphics debugging and accessibility is genuinely unoccupied and the demo is the most visually arresting of the set. Rejected because inferring render-pass roles from framebuffer bindings and texture aliasing with no ground truth is an open research problem, and the failure mode is catastrophic rather than graceful — an accessibility tool that renders a *wrong* map is more dangerous than a blurry one.

**Parity** — bundle differential testing. Genuinely unoccupied technique and excellent ddmin/AST-reduction engineering, but the product fires roughly once a year per team and gets uninstalled in between.

**Accessibility as a whole domain is deliberately absent.** This is a considered call, not an oversight. The two candidates scored 9 on portfolio signal and 4 on real-world value, and in both cases the low value is structural rather than fixable by scoping: switch users are locked into insurance-funded Windows AAC suites that `npm install` cannot reach, and multiline tactile displays exist in the low thousands of units at $15k each, behind gated SDKs a solo developer will not obtain. Building either would mean shipping simulator-only work to a population that cannot run it — which reviewers correctly read as unvalidated, and which fails the "solves a real problem for real users" bar no matter how good the engineering is.

If accessibility matters to you personally, the honest path is to contribute to NVDA add-ons or Orca rather than to build a new tool nobody in the target population can install.

---

## Before you commit

Spot-check the prior-art claims yourself before sinking months into any of these. Two worth verifying first:

- `chronos deterministic simulation testing node` — confirms why the DST idea was dropped.
- `webrtc encoded transform recorder` — confirms Flightbox's gap is still open.

Prior art moves. A claim that was true when this document was written is a starting point for your own search, not a substitute for it.
