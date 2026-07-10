# WebSocket — Zero to Advanced (The Only Guide You'll Need)

This guide assumes you know **nothing**. We'll build up every concept brick by brick, so by the end, WebSocket won't feel like a "topic" anymore — it'll feel obvious.

---

## PART 0: The Absolute Basics (Before WebSocket Even Makes Sense)

### 0.1 What is a "Client" and a "Server"?

- **Client** = the one who asks for something (your browser, your phone app).
- **Server** = the one who has the data/service and answers.

Think of it like a restaurant:
- You (client) ask the waiter for food.
- Kitchen (server) prepares and sends it back.

### 0.2 What is an IP Address and a Port?

- **IP address** = the "street address" of a computer on a network. Example: `192.168.1.5`
- **Port** = the "apartment number" inside that building. A computer can run many programs at once (a web server, a game server, etc.), so ports tell the network *which program* the data is for.
- Example: `192.168.1.5:443` means "go to this computer, then to apartment 443" (443 is the standard port for secure web traffic).

### 0.3 What is TCP? (The delivery truck)

TCP (Transmission Control Protocol) is the underlying system that **reliably** delivers data between two computers.

Think of TCP like a phone call:
1. You dial (connect)
2. Both sides say "hello, can you hear me?" (handshake)
3. You talk back and forth (data transfer)
4. You say "bye" and hang up (close)

Key property: TCP guarantees your data arrives **in order** and **complete** (no missing pieces). This matters a lot later.

```
Client                          Server
  |------ SYN (let's talk?) --->|
  |<---- SYN-ACK (ok, go) ------|
  |------ ACK (confirmed) ----->|
  |                             |
  |        [connection open]   |
  |<---- data flows both ways->|
```

This is called the **TCP 3-way handshake**. Remember this — WebSocket builds directly on top of an already-open TCP connection.

### 0.4 What is HTTP? (The language spoken over TCP)

HTTP (HyperText Transfer Protocol) is a set of **rules for how a client asks for something and how a server replies**. It runs *on top of* TCP (TCP is the truck, HTTP is the delivery form you fill out).

A basic HTTP interaction:

```
CLIENT REQUEST:
GET /home.html HTTP/1.1
Host: example.com

SERVER RESPONSE:
HTTP/1.1 200 OK
Content-Type: text/html

<html>...</html>
```

### 0.5 The Critical Limitation of HTTP: It's "Request-Response Only"

This is THE single most important fact to understand before WebSocket makes sense.

**HTTP works like this:**
1. Client asks a question.
2. Server answers.
3. Connection is done (or kept idle briefly).
4. If the client wants new info, it must **ask again**.

**The server can NEVER speak first.** It can only reply when asked. This is called a **half-duplex, client-initiated** protocol.

```
Client: "Any new messages?"    -----> Server: "No."
   (client closes/waits)

Client: "Any new messages?"    -----> Server: "No."
   (client closes/waits)

Client: "Any new messages?"    -----> Server: "Yes, here's 1!"
```

### 0.6 The Problem This Creates: Real-Time Apps

Imagine a chat app, a live stock ticker, a multiplayer game, or a live sports score tracker. You want the **server to instantly push data to the client the moment something happens** — you don't want the client to keep asking "anything new? anything new? anything new?"

But HTTP doesn't let the server push. So historically, people invented **hacks** to fake real-time behavior:

#### Hack 1: Short Polling
Client asks the server every X seconds ("any updates?").

```
Client -----request----> Server
Client <----response---- Server   (wait 3 sec)
Client -----request----> Server
Client <----response---- Server   (wait 3 sec)
...
```
**Problem:** Wasteful (tons of empty requests), and not truly instant (up to 3-sec delay).

#### Hack 2: Long Polling
Client asks the server, but the server **holds the request open** and only responds when it actually has new data (or after a timeout).

```
Client -----request---------> Server
                (server waits... waits... something happens!)
Client <----response(data)---- Server
Client -----request(again)---> Server   (immediately re-asks)
```
**Better**, but still has overhead: every response requires a fresh new HTTP request/connection setup, and each request carries a full set of HTTP headers (cookies, user-agent, etc.) — repeated over and over, wasting bandwidth.

#### Hack 3: Server-Sent Events (SSE)
The server keeps ONE HTTP connection open and can keep streaming data to the client over time.

**Problem:** SSE is **one-directional only** — server → client. The client still can't send data back over that same connection; it needs a separate normal HTTP request. Also SSE only supports text data, not binary.

### 0.7 So... What Do We Actually Need?

We need a connection that:
1. Stays open (no reconnecting every time)
2. Allows **both sides to send data whenever they want**, not just in response to a question
3. Is lightweight (no repeated headers every message)
4. Works over the existing web infrastructure (ports 80/443, browsers, firewalls)

**This is exactly what WebSocket was built to solve.**

---

## PART 1: What Is WebSocket?

**WebSocket is a communication protocol that provides a persistent, full-duplex (two-way) connection between a client and a server over a single TCP connection.**

Let's unpack every word:

- **Persistent** = the connection stays open until either side decides to close it (not one-request-and-done like HTTP).
- **Full-duplex** = both sides can send messages to each other **at any time**, independently, simultaneously. Not "ask then wait."
- **Single TCP connection** = it reuses one underlying pipe instead of opening a new one for every message.

### 1.1 The Restaurant Analogy, Upgraded

- **HTTP** = You call the restaurant every time you want to ask "is my food ready?" New phone call each time.
- **WebSocket** = You stay on one continuous phone call with the kitchen. The kitchen can say "your food is ready!" the *instant* it happens, without you asking. And you can talk back anytime too, on the same call.

### 1.2 Visual: HTTP vs WebSocket

```
HTTP (many short-lived connections):
Client -----REQ 1----> Server
Client <----RES 1----- Server   [connection closes]
Client -----REQ 2----> Server
Client <----RES 2----- Server   [connection closes]
Client -----REQ 3----> Server
Client <----RES 3----- Server   [connection closes]


WebSocket (one long-lived connection):
Client <===========================> Server
        (single connection, open)
Client ---msg---------------------->  Server
Client <----------------msg--------- Server
Client <----------------msg--------- Server
Client ---msg---------------------->  Server
        (either side, any time, no re-asking)
```

---

## PART 2: The WebSocket Handshake (How the Connection Actually Starts)

This is the part most people misunderstand, so pay close attention: **WebSocket does NOT start as a WebSocket.** It **starts as a normal HTTP request** and then "upgrades" itself into a WebSocket connection. This trick is exactly why WebSocket works seamlessly through existing web infrastructure — firewalls, proxies, browsers all already understand HTTP.

### 2.1 Step-by-Step Handshake

**Step 1 — Client sends a special HTTP request asking to "upgrade":**

```
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

What's happening here:
- `Upgrade: websocket` — "Hey server, I don't want plain HTTP anymore, I want to switch protocols."
- `Connection: Upgrade` — confirms this is a protocol-switch request.
- `Sec-WebSocket-Key` — a random value generated by the client, used to prove the server actually understood the WebSocket request (a basic anti-cache / anti-confusion safety check, NOT encryption/security).
- `Sec-WebSocket-Version` — which version of the WebSocket protocol the client speaks (13 is the modern standard, RFC 6455).

**Step 2 — Server agrees and responds:**

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

- `101 Switching Protocols` — special HTTP status code meaning "OK, I'm changing the rules of this connection now."
- `Sec-WebSocket-Accept` — server takes the client's key, combines it with a fixed magic string, hashes it (SHA-1), and sends it back — proving it's a real WebSocket-aware server, not some random HTTP server misinterpreting the request.

**Step 3 — From this point on, the SAME underlying TCP connection is reused, but it's no longer speaking HTTP. It's now speaking the WebSocket protocol.** No new connection was made — it's literally the same pipe, just repurposed.

### 2.2 Full Handshake Diagram

```
   CLIENT                                   SERVER
     |                                         |
     |  GET /chat HTTP/1.1                     |
     |  Upgrade: websocket                     |
     |  Sec-WebSocket-Key: xxxx      --------->|
     |                                         |
     |               <---------  HTTP/1.1 101 Switching Protocols
     |                            Sec-WebSocket-Accept: yyyy
     |                                         |
     |========= WEBSOCKET CONNECTION OPEN =========|
     |                                         |
     |  ---- message: "hello" ---------------->|
     |  <----------------- message: "hi back" -|
     |  <----------------- message: "update!" -|
     |  ---- message: "ack" ------------------>|
     |                                         |
     |========= connection stays open until close =========|
```

### 2.3 The URL Scheme

- `ws://` = WebSocket over plain (unencrypted) connection — equivalent to `http://`
- `wss://` = WebSocket **Secure** over TLS/SSL encryption — equivalent to `https://` (always use this in production)

---

## PART 3: What Happens After the Handshake — The Actual Protocol

Once upgraded, data is sent in **frames**, not raw text blobs. Understanding frames is what separates "I used WebSocket" from "I actually understand WebSocket."

### 3.1 Why Frames? (Not Just Raw Bytes)

If two sides just dumped random bytes into a pipe, how would either side know:
- Where does one message end and the next one begin?
- Is this message text or binary (like an image)?
- Is this actually a "close the connection" signal, or a real message?

**Frames solve this by wrapping every piece of data with a small header describing what it is.**

### 3.2 Frame Structure (Simplified Mental Model)

Every WebSocket frame has:

```
+-----------+---------------+----------------+-------------+
|  FIN bit  |   Opcode      |  Payload Length |   Payload   |
| (1 or 0)  | (what type?)  |  (how much data) |  (the data)|
+-----------+---------------+----------------+-------------+
```

- **FIN bit**: Is this the FINAL piece of the message, or are more fragments coming? (Large messages can be split into multiple frames — this is called **fragmentation**.)
- **Opcode**: Tells the receiver what kind of frame this is:
  - `0x1` = Text frame (UTF-8 text data)
  - `0x2` = Binary frame (images, files, protobuf, etc.)
  - `0x8` = Close frame (either side wants to end the connection)
  - `0x9` = Ping frame (a "are you still alive?" heartbeat check)
  - `0xA` = Pong frame (reply to a ping)
  - `0x0` = Continuation frame (this is a follow-up piece of a fragmented message)
- **Payload length**: How many bytes of actual data follow.
- **Payload**: The actual message content.

### 3.3 Masking (Client → Server Only)

Every frame sent **from client to server MUST be "masked"** — the payload bytes are XOR'd with a random 4-byte key that's included in the frame. The server unmasks it on arrival.

**Why does this exist?** It's a security measure. Without masking, a malicious webpage could craft byte sequences that, if sent unmasked, might trick misconfigured proxies/caches (which don't understand WebSocket) into misinterpreting the traffic as something else — potentially poisoning shared caches. Masking makes the bytes unpredictable so this trick doesn't work.

**Note:** Server → client frames are **NOT masked**. Masking is one-directional (client-to-server only), because the threat model is specifically about untrusted browser code sending crafted payloads.

### 3.4 Ping / Pong — The Heartbeat System

Since a WebSocket connection can sit idle for a long time, both sides need a way to check "are you still there?" — especially because network devices, load balancers, or proxies often auto-close connections that look "idle" or "dead."

```
Server ----PING---->  Client
Client ----PONG---->  Server   (I'm alive!)
```

If a Pong doesn't come back within a timeout, the sender assumes the connection is dead and closes it. This is critical for building reliable production systems (more on this in Part 6).

### 3.5 Closing a Connection

Either side sends a **Close frame** (opcode `0x8`), optionally with a status code and reason (e.g., `1000` = normal closure, `1006` = abnormal closure/connection lost). The other side should reply with its own Close frame to confirm, then the underlying TCP connection is actually torn down.

```
Client ----Close(1000, "done")---->  Server
Client <---Close(1000, "ack")------  Server
        [TCP connection now terminates]
```

---

## PART 4: The WebSocket Lifecycle (From a Developer's Mental Model)

Regardless of language/library, every WebSocket connection conceptually goes through these states:

```
CONNECTING  --->  OPEN  --->  CLOSING  --->  CLOSED
   |                |             |
 handshake      messages      close frame
 in progress    flowing       being exchanged
   |                |             |
   +--- error? ----> [can jump straight to CLOSED]
```

1. **CONNECTING** — the HTTP upgrade handshake is happening.
2. **OPEN** — handshake succeeded, both sides can now send/receive messages freely.
3. **CLOSING** — one side has sent a close frame, waiting for confirmation.
4. **CLOSED** — connection fully terminated (either gracefully or due to an error/network failure).

Every WebSocket client library exposes four core events matching this lifecycle: `open`, `message`, `close`, `error`.

---

## PART 5: Comparing All Real-Time Techniques (Full Picture)

| Technique | Direction | Connection | Overhead | Use Case |
|---|---|---|---|---|
| Short Polling | Client asks repeatedly | New connection each time | High (repeated headers, many idle requests) | Simple, low-frequency updates (e.g., checking order status every few min) |
| Long Polling | Client asks, server holds | New connection per response | Medium | Fallback when WebSocket isn't available |
| SSE (Server-Sent Events) | Server → Client ONLY | One persistent HTTP connection | Low | Live feeds, notifications, stock tickers (no client→server need) |
| WebSocket | Both ways, freely | One persistent TCP connection | Very low | Chat, multiplayer games, collaborative editing, live trading, VoIP signaling |
| WebRTC (bonus, different tool) | Peer-to-peer, both ways | Direct connection (not via server) | Very low but complex setup | Video/audio calls, screen sharing |

**Rule of thumb:** If you need the server to push data AND the client to send data back on the same live channel — WebSocket is almost always the right choice.

---

## PART 6: Advanced — Production-Grade Concerns

This is the part most tutorials skip, but it's what interviewers and real systems actually care about.

### 6.1 Reconnection Strategy

Networks are unreliable — WiFi drops, phones switch towers, servers restart. A production WebSocket client MUST handle disconnects gracefully.

**Exponential Backoff Pattern** (industry standard):
```
Attempt 1: wait 1s   then retry
Attempt 2: wait 2s   then retry
Attempt 3: wait 4s   then retry
Attempt 4: wait 8s   then retry
...
(cap it, e.g., max wait 30s, so it doesn't grow forever)
```
Why exponential and not fixed intervals? If the server just crashed and 10,000 clients all retry every 1 second at the same time, you create a "thundering herd" that can crash the server again the moment it comes back up. Exponential backoff (ideally with slight randomness/"jitter") spreads out reconnection attempts.

**React example — a `useWebSocket` hook with exponential backoff reconnection:**

```javascript
import { useEffect, useRef, useState, useCallback } from "react";

function useWebSocket(url) {
  const [isConnected, setIsConnected] = useState(false);
  const [lastMessage, setLastMessage] = useState(null);
  const wsRef = useRef(null);
  const attemptRef = useRef(0);
  const timeoutRef = useRef(null);

  const connect = useCallback(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onopen = () => {
      setIsConnected(true);
      attemptRef.current = 0; // reset backoff on successful connect
    };

    ws.onmessage = (event) => {
      setLastMessage(JSON.parse(event.data));
    };

    ws.onclose = () => {
      setIsConnected(false);
      const delay = Math.min(1000 * 2 ** attemptRef.current, 30000); // cap at 30s
      const jitter = Math.random() * 500;
      attemptRef.current += 1;
      timeoutRef.current = setTimeout(connect, delay + jitter);
    };

    ws.onerror = () => ws.close(); // triggers onclose -> reconnect logic
  }, [url]);

  useEffect(() => {
    connect();
    return () => {
      clearTimeout(timeoutRef.current);
      wsRef.current?.close();
    };
  }, [connect]);

  const sendMessage = (data) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify(data));
    }
  };

  return { isConnected, lastMessage, sendMessage };
}
```

### 6.2 Heartbeats in Practice

Beyond protocol-level Ping/Pong, many production systems also implement **application-level heartbeats** (a small JSON message like `{"type": "heartbeat"}`) because:
- Some proxies/load balancers don't properly forward low-level Ping/Pong frames.
- It gives the application itself visibility into "when did I last hear from this client" for cleanup logic.

### 6.3 Scaling WebSocket Servers (The Hard Part)

Here's a subtlety that trips up many engineers: **HTTP servers are easy to scale horizontally because each request is independent** — any server behind a load balancer can handle any request. **WebSocket connections are stateful and long-lived** — once Client A connects to Server Instance #2, that connection **stays pinned to Server Instance #2** for its entire lifetime.

This creates a real problem: if Client A (on Server #2) wants to send a message to Client B (connected to Server #5), Server #2 has no direct way to reach Client B.

**Solution: Pub/Sub backbone (commonly Redis Pub/Sub, Kafka, or NATS)**

```
Client A                          Client B
   |                                  |
   v                                  v
Server #2                        Server #5
   |                                  |
   +------------> Redis Pub/Sub <-----+
        (all servers subscribe to
         shared channels; when #2
         publishes, #5 receives it
         and pushes to Client B)
```

Every server instance publishes incoming messages to a shared message broker, and every server instance subscribes to relevant channels — so no matter which physical server a client is attached to, messages can be routed to them.

### 6.4 Load Balancer "Sticky Sessions"

Because a WebSocket connection must stay pinned to one server, load balancers need **sticky sessions** (a.k.a. session affinity) — meaning once a client is routed to Server #2, all future requests/frames from that same client must keep going to Server #2, not get randomly redistributed like normal HTTP traffic.

### 6.5 Backpressure

If a server is pushing messages faster than a slow client (e.g., bad mobile network) can receive them, messages pile up in a send buffer. If unmanaged, this can exhaust server memory (imagine 100,000 slow clients each buffering unsent data). Production systems must monitor buffer sizes and apply strategies like dropping non-critical messages, throttling, or disconnecting unresponsive clients.

### 6.6 Security Considerations

1. **Always use `wss://` (encrypted), never plain `ws://` in production** — otherwise messages (including auth tokens) travel in plaintext, readable by anyone on the network path.

2. **No built-in authentication in the protocol itself.** WebSocket has no concept of "login." Authentication must be handled by the application, typically by:
   - Passing a token in the initial HTTP upgrade request (e.g., as a query parameter or header, since the handshake IS a normal HTTP request), or
   - Sending an auth message as the very first WebSocket message after connecting, and having the server validate it before treating the connection as "trusted."

3. **Origin validation (CSRF-like risk).** Unlike normal AJAX requests, browsers do NOT enforce the Same-Origin Policy for WebSocket connections by default — any website's JavaScript can attempt to open a WebSocket connection to any WebSocket server. This means a malicious website could try to open a WebSocket connection to your server using a victim's existing cookies/session. **The server must explicitly check the `Origin` header** during the handshake and reject connections from untrusted origins.

4. **Rate limiting / flood protection.** Since a WebSocket connection allows unlimited free-form messages after the handshake, servers need to enforce message-rate limits per connection to prevent abuse/DoS (a malicious client blasting thousands of messages per second).

5. **Message size limits.** Without limits, a client could send an enormous single frame/message designed to exhaust server memory. Production servers cap max message size.

6. **Input validation, always.** Just because it's WebSocket doesn't mean you skip validating/sanitizing incoming data — treat every message like you would any other untrusted user input (never trust the client).

### 6.7 Fragmentation in Practice

Large messages (say, a big JSON payload or a file chunk) can be split across multiple frames using the FIN bit + continuation opcode (`0x0`) we discussed in Part 3.2. This lets a sender start transmitting a large message before the whole thing is even ready (useful for streaming), and lets receivers process data incrementally instead of waiting for the entire payload to buffer in memory.

---

## PART 7: Hands-On — Node.js and React Code Examples

Everything above was theory. Here's how it actually looks in code, using the `ws` library on the server (the standard, minimal WebSocket library for Node.js) and plain browser `WebSocket` API on the client (no extra library needed — it's built into every browser).

### 7.1 Setup

```bash
mkdir ws-demo && cd ws-demo
npm init -y
npm install ws
```

### 7.2 Basic Server (Node.js) — Matches Part 1 & Part 4 (Lifecycle)

```javascript
// server.js
const { WebSocketServer } = require("ws");

const wss = new WebSocketServer({ port: 8080 });

wss.on("connection", (socket, request) => {
  console.log("Client connected from:", request.socket.remoteAddress);

  // This fires for every message received from this client
  socket.on("message", (data) => {
    console.log("Received:", data.toString());

    // Broadcast to ALL connected clients (including sender)
    wss.clients.forEach((client) => {
      if (client.readyState === client.OPEN) {
        client.send(data.toString());
      }
    });
  });

  socket.on("close", (code, reason) => {
    console.log(`Client disconnected: ${code} ${reason}`);
  });

  socket.on("error", (err) => {
    console.error("Socket error:", err);
  });

  socket.send(JSON.stringify({ type: "welcome", message: "Connected!" }));
});

console.log("WebSocket server running on ws://localhost:8080");
```

Run it: `node server.js`

### 7.3 Basic Client (React) — Matches Part 4 (Lifecycle Events)

This uses the browser's **native** `WebSocket` API — no npm package needed on the client for raw WebSocket.

```jsx
import { useEffect, useRef, useState } from "react";

function ChatWindow() {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState("");
  const [status, setStatus] = useState("CONNECTING");
  const socketRef = useRef(null);

  useEffect(() => {
    const socket = new WebSocket("ws://localhost:8080");
    socketRef.current = socket;

    socket.onopen = () => setStatus("OPEN");

    socket.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setMessages((prev) => [...prev, data]);
    };

    socket.onclose = () => setStatus("CLOSED");

    socket.onerror = (err) => console.error("WebSocket error:", err);

    // Cleanup: close the connection when the component unmounts
    return () => socket.close();
  }, []);

  const sendMessage = () => {
    if (socketRef.current?.readyState === WebSocket.OPEN) {
      socketRef.current.send(JSON.stringify({ type: "chat", text: input }));
      setInput("");
    }
  };

  return (
    <div>
      <p>Status: {status}</p>
      {messages.map((msg, i) => (
        <p key={i}>{msg.type === "welcome" ? msg.message : msg.text}</p>
      ))}
      <input value={input} onChange={(e) => setInput(e.target.value)} />
      <button onClick={sendMessage}>Send</button>
    </div>
  );
}

export default ChatWindow;
```

**Note the direct mapping to Part 4's lifecycle:** `onopen` → CONNECTING finished, now OPEN. `onmessage` → data flowing while OPEN. `onclose` → CLOSING/CLOSED. `onerror` → error state. This is exactly the four-event pattern described earlier, now as real code.

### 7.4 Server-Side Ping/Pong Heartbeat (Matches Part 3.4 and Part 6.2)

The `ws` library handles low-level ping/pong automatically, but you still need to actively check for dead connections yourself:

```javascript
// Add this to server.js to detect and terminate dead connections
function heartbeat() {
  this.isAlive = true;
}

wss.on("connection", (socket) => {
  socket.isAlive = true;
  socket.on("pong", heartbeat); // browser auto-responds to pings with pongs

  // ...rest of connection handling from 7.2...
});

// Every 30 seconds, ping all clients and terminate any that didn't respond last time
const interval = setInterval(() => {
  wss.clients.forEach((socket) => {
    if (socket.isAlive === false) return socket.terminate(); // no pong = dead

    socket.isAlive = false;
    socket.ping();
  });
}, 30000);

wss.on("close", () => clearInterval(interval));
```

### 7.5 Sending Binary Data (Matches Part 3.2 — Opcode `0x2`)

```javascript
// Server: sending a binary buffer instead of text
socket.send(Buffer.from([1, 2, 3, 4]));

// Client: receiving binary data
socket.onmessage = (event) => {
  if (event.data instanceof Blob) {
    event.data.arrayBuffer().then((buffer) => {
      console.log(new Uint8Array(buffer)); // [1, 2, 3, 4]
    });
  }
};
```

### 7.6 Authenticated Connection (Matches Part 6.6 — Security)

Since WebSocket has no built-in auth, pass a token via query string during the handshake and validate it before accepting the connection:

```javascript
// Server — validate token BEFORE upgrading the connection
const { createServer } = require("http");
const { WebSocketServer } = require("ws");
const url = require("url");

const server = createServer();
const wss = new WebSocketServer({ noServer: true });

server.on("upgrade", (request, socket, head) => {
  const { query } = url.parse(request.url, true);
  const token = query.token;

  if (!isValidToken(token)) {
    socket.write("HTTP/1.1 401 Unauthorized\r\n\r\n");
    socket.destroy();
    return;
  }

  wss.handleUpgrade(request, socket, head, (ws) => {
    wss.emit("connection", ws, request);
  });
});

server.listen(8080);
```

```javascript
// Client — pass the token as a query param
const socket = new WebSocket(`ws://localhost:8080?token=${userToken}`);
```

---

## PART 8: Common Interview Questions (With Short, Sharp Answers)

**Q: What problem does WebSocket solve that HTTP can't?**
A: HTTP is request-response only — the server can never push data on its own. WebSocket provides a persistent, full-duplex connection so either side can send data anytime.

**Q: Why does WebSocket start with an HTTP request?**
A: So it can pass through existing web infrastructure (firewalls, proxies, standard ports 80/443) without special network configuration — it "upgrades" an existing HTTP connection rather than needing an entirely new protocol/port.

**Q: What HTTP status code confirms a successful WebSocket handshake?**
A: `101 Switching Protocols`.

**Q: Is WebSocket traffic masked in both directions?**
A: No — only client-to-server frames are masked. Server-to-client frames are unmasked.

**Q: How would you scale WebSocket servers horizontally?**
A: Use a shared Pub/Sub backbone (like Redis) so any server instance can broadcast a message to clients connected to other instances, combined with sticky sessions at the load balancer level.

**Q: WebSocket vs SSE — when would you pick which?**
A: SSE if you only need server → client push (simpler, works over plain HTTP, auto-reconnect built in). WebSocket if you need true bidirectional, low-latency communication (chat, gaming, live collaboration).

**Q: How do you detect a dead/broken WebSocket connection?**
A: Ping/Pong heartbeats at the protocol level, often supplemented by application-level heartbeats, with a timeout that triggers reconnection logic if no Pong/heartbeat is received.

**Q: Does WebSocket have built-in authentication?**
A: No. Auth must be handled by the app — typically a token passed during the handshake (query param/header) or as the first message after connecting.

---

## PART 9: Real-World Use Cases (So You Know Where This Applies)

- **Chat applications** (WhatsApp Web, Slack, Discord) — instant bidirectional messaging.
- **Live collaborative tools** (Google Docs-style cursors/edits, Figma) — every keystroke/change pushed instantly to all collaborators.
- **Live sports/stock tickers** — server pushes updates the moment they happen.
- **Multiplayer games** — low-latency state sync between players.
- **Notifications systems** — instant "someone liked your post" style pushes.
- **IoT device dashboards** — live sensor data streaming.
- **Trading platforms** — real-time price feeds, order book updates.

---

## Quick-Reference Cheat Sheet (For Fast Recall)

1. WebSocket = persistent, full-duplex connection over a single TCP connection.
2. Starts as HTTP, upgrades via `101 Switching Protocols`.
3. `ws://` = unencrypted, `wss://` = encrypted (always use in production).
4. Data travels in **frames** — has FIN bit, opcode (text/binary/close/ping/pong), payload.
5. Client→server frames are masked; server→client are not.
6. Ping/Pong = heartbeat to detect dead connections.
7. No built-in auth — you must implement it yourself (token in handshake or first message).
8. Browsers don't enforce Same-Origin for WebSocket — server MUST validate `Origin` header manually.
9. Scaling requires sticky sessions + a Pub/Sub backbone (Redis/Kafka) since connections are stateful and pinned to one server.
10. Reconnection should use exponential backoff with jitter, not fixed-interval retries.
11. Working code: Node.js server uses the `ws` library; React client uses the browser's **native** `WebSocket` API (no library needed) — see Part 7.

---

