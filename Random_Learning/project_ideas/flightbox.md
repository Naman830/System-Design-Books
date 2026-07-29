# Flightbox — Explained Like You're New To This

## 1. The problem, in plain English

Imagine you're on a video call (Zoom, Google Meet, Discord — anything that uses WebRTC, which is the technology browsers use for live video/audio). Suddenly the video freezes for a second.

You report it: "the video froze for a moment around 2pm."

The engineer on the other end opens their monitoring dashboard and sees... one number:

```
freezeCount: 1
```

That's it. That's the entire evidence they have. They don't know:
- Which video frame froze
- Why it froze (packet lost? decoder crashed? network glitch?)
- What the video actually looked like at that moment
- Whether it will happen again

It's like reporting a car crash and the only evidence the police have is "a crash happened once." No photos, no skid marks, no black box recording. You can't debug what you can't see.

**Why does this gap exist?** Because recording live video traffic inside a real browser, during a real call, without slowing down the call or eating up the user's disk/CPU, is genuinely hard. Tools exist to *analyze* a recorded stream after the fact (`video_replay`, `neteq_rtpplay` — tools real video engineers already trust). But nothing captures that stream from inside a live browser tab in production. The "record it" half of the pipeline has never been built. Flightbox is that missing half.

## 2. What Flightbox actually does (the simple version)

Think of it like a **flight data recorder ("black box") for video calls**, or a car's dash-cam that's always quietly recording a rolling buffer, but only saves the footage if there's an incident.

1. While the video call is running, Flightbox quietly taps into the incoming video stream (before it's decoded into pixels — while it's still compressed data).
2. It keeps a short rolling recording of that stream in the browser (like a dash-cam's loop buffer) — old stuff gets thrown away automatically to save space.
3. If something goes wrong and the user clicks "Report a problem," Flightbox saves that rolling buffer instead of throwing it away.
4. Later, an engineer opens that saved recording in a viewer and sees exactly what happened, frame by frame: *"frame 8814 depended on frame 8809, which never arrived. That's why the next 43 frames couldn't be shown until the next full picture (keyframe) arrived at frame 8858."*

That sentence — the **causal decode timeline** — is the actual product. Not a fancy video player. A precise explanation of *why* the freeze happened, expressed as cause and effect between frames.

## 3. Why this is technically hard (the "boss fight" of the project)

The single biggest challenge: **the recorder must be free.** It cannot slow down the call, add lag, or use noticeably more battery/CPU — because if it does, no real company will ever turn it on in production. So most of the engineering effort goes into:

- Copying video data with as few extra memory copies as possible (ideally zero wasted copies).
- Never triggering garbage collection pauses in JavaScript (which can cause its own freezes — ironic, since you'd be the thing causing the bug you're trying to catch).
- Proving all this with real measurements (CPU graphs, timing histograms), not just claiming it's fast.

The secondary hard problem: figuring out *which frame depends on which* when frames can arrive out of order, get dropped, or when the video quality level switches mid-call (this is called "simulcast layer switching"). It's a puzzle of cause-and-effect bookkeeping.

## 4. How you'd actually build it (v1, the realistic starting scope)

Don't build the whole thing at once. Here's the order that keeps you honest and unblocked:

1. **Build the speed-measuring harness first.** Before writing a single line of the recorder, build the tool that measures "did this slow the call down?" You need the ruler before you build the thing you're measuring — otherwise you're guessing.
2. **Tap the incoming video stream.** Use a WebRTC browser feature called `RTCRtpScriptTransform` — it lets you see the compressed video chunks as they arrive, before they're turned into a picture.
3. **Store them in a rolling buffer.** Keep the last N seconds in memory using a `SharedArrayBuffer` (a fast, shared memory block two parts of your code can both read/write to instantly, no copying).
4. **Save to disk without blocking anything.** Write the buffer to a file system that lives inside the browser tab (called OPFS — Origin Private File System) using a background worker so the main call never waits on disk.
5. **Build the "why it froze" analyzer.** Walk through the saved frames offline and reconstruct which frame depended on which, and where the chain broke.
6. **Build the viewer.** A simple web page (Next.js) where you load a saved recording and see the timeline plus, best-effort, the actual video played back using `WebCodecs` (the browser's built-in video decoder).
7. **Export to formats real engineers already use** (`rtpdump`/IVF file formats) so your tool plugs into existing debugging tools instead of forcing people to learn a new one.

## 5. Tools & technologies you'll need to learn

| Tool / Concept | What it's for | Beginner-friendly note |
|---|---|---|
| **WebRTC** | The browser tech for live audio/video calls | You don't need to build a video call app — you're plugging into existing ones like a wiretap |
| **`RTCRtpScriptTransform`** | Lets you intercept encoded video/audio frames mid-flight | This is the actual "tap point" — the core hook of the whole project |
| **`SharedArrayBuffer`** | Fast shared memory between browser threads | Think of it as a shared whiteboard two workers can write on without passing notes back and forth |
| **Web Workers** | Background threads in the browser | Keeps the recording work off the main thread so the call doesn't lag |
| **OPFS (Origin Private File System)** | A private, fast file system inside the browser tab | This is where your rolling recording gets saved to disk |
| **`WebCodecs`** | Browser API to decode video frames into actual pictures | Used in the viewer, to play back what was recorded |
| **`rtpdump` / IVF file formats** | Standard formats real video engineers already use | Exporting to these means your tool is useful on day one, not a rival ecosystem |
| **Next.js** | For the viewer/dashboard UI | This is just the "window" into your recordings — not the hard part of the project |
| **Chrome DevTools / V8 CPU profiler** | Measuring whether your recorder is actually "free" | This produces the evidence a skeptical engineer would demand before trusting your tool |

You'll be testing against real WebRTC servers like **mediasoup**, **LiveKit**, or **Janus** (open-source video-call backends) to prove your integration works on real infrastructure, not just a toy demo.

## 6. What success looks like

A GitHub repo where:
- A ~20-line code snippet plugs Flightbox into an existing WebRTC app.
- A benchmark report proves the recording adds no measurable slowdown at high resolution/frame rate.
- A saved recording, when opened, tells you in plain language exactly which frame broke and why.
- The README has an honest **"what this does not tell you"** section (e.g., pixel-perfect replay is best-effort, not guaranteed) — because a tool that's honest about its limits is trusted more than one that overclaims.

## 7. What it deliberately does NOT try to do (v1)

- It does not try to fix the network or the call — only to explain what happened.
- It does not guarantee the replayed video looks pixel-identical to the original — that claim is fragile and clearly labeled as best-effort.
- It only records the *incoming* (receive-side) video for now — not the outgoing side, not audio-only edge cases.

---
*Companion doc to `5_ideas.md` — this is the deep-dive on project #1, Flightbox.*
