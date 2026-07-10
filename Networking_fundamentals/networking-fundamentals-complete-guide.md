# Networking Fundamentals — Zero to Advanced (The Only Guide You'll Need)

This assumes you know **nothing**. Every concept builds on the previous one. By the end, you'll be able to walk into any networking/system-design interview round without flinching.

---

## PART 0: What Even IS a Network?

A **network** is just multiple computers able to talk to each other. That's it. Everything else in this guide is just "how do we make that talking reliable, fast, findable, and secure."

To talk, computers need to agree on:
1. **Where** to send data (addressing — IP addresses)
2. **How** to physically move bits (cables, WiFi, switches)
3. **How** to make sure it arrives correctly (TCP)
4. **What language** to speak once connected (HTTP, DNS, etc.)

This guide walks through exactly these layers, bottom to top.

---

## PART 1: The OSI Model (The Map You'll Reference Forever)

The OSI (Open Systems Interconnection) model is a **7-layer conceptual framework** describing how data travels from one machine to another. You don't need to memorize it like a poem — you need to understand *why* each layer exists.

```
Layer 7 - Application     (HTTP, DNS, FTP, SMTP)         "What does the user/app want to say?"
Layer 6 - Presentation    (encryption, compression)       "Format/translate the data"
Layer 5 - Session         (session management)            "Keep track of the conversation"
Layer 4 - Transport       (TCP, UDP)                       "Reliable or fast delivery?"
Layer 3 - Network         (IP, routers)                    "Which path across networks?"
Layer 2 - Data Link       (MAC addresses, switches)        "Which device on THIS local network?"
Layer 1 - Physical        (cables, radio waves, voltage)   "Actual bits as electricity/light/radio"
```

**Real-world analogy — sending a letter internationally:**
- Layer 7 (Application) = the actual letter content you wrote
- Layer 6 (Presentation) = translating it to the recipient's language, sealing it in an envelope
- Layer 5 (Session) = keeping track that this is "conversation #4" with this pen pal
- Layer 4 (Transport) = choosing courier service that guarantees delivery (TCP) vs regular mail that doesn't (UDP)
- Layer 3 (Network) = the postal routing system deciding which country → city → route
- Layer 2 (Data Link) = the local delivery truck getting it to the exact street
- Layer 1 (Physical) = the literal truck driving on literal roads

**In practice, most real-world discussions collapse this into 4 layers (the TCP/IP model)**, which is what you'll actually use day-to-day:

```
Application   (HTTP, DNS, WebSocket, FTP...)     <- combines OSI layers 5,6,7
Transport     (TCP, UDP)                          <- OSI layer 4
Internet      (IP, routing)                       <- OSI layer 3
Link          (Ethernet, WiFi, MAC addresses)      <- OSI layers 1,2
```

**Interview tip:** When asked "explain OSI model," give the 7 layers, but immediately mention the 4-layer TCP/IP model is what's actually used in practice — this shows real understanding, not memorization.

---

## PART 2: Physical & Data Link Layer (Layers 1–2) — The Local Neighborhood

### 2.1 MAC Address

Every network device (your laptop's WiFi card, your phone) has a **MAC address** — a unique hardware identifier burned into the device, like `A4:C3:F0:85:AC:2D`. It identifies a device **on its local network** (like your home WiFi).

### 2.2 Switches

A **switch** connects multiple devices within the same local network (e.g., all computers in an office) and forwards data using MAC addresses. It only knows about devices directly connected to it — it has no concept of "the internet."

### 2.3 Why This Layer Alone Isn't Enough

MAC addresses only work for local networks. They don't scale globally — there's no efficient way to look up "which physical wire leads to this MAC address" across millions of networks worldwide. This is exactly why we need **IP addresses** (Layer 3) — a addressing system designed for routing across many interconnected networks.

---

## PART 3: The Network Layer — IP Addressing (Layer 3)

### 3.1 What Is an IP Address?

An IP address is a numeric address that identifies a device **globally**, across networks (unlike MAC, which is only meaningful locally).

**IPv4** format: four numbers 0–255, separated by dots. Example: `192.168.1.5`
Each number is 8 bits (a "byte"/octet), so IPv4 = 32 bits total = about 4.3 billion possible addresses.

**IPv6** exists because IPv4 addresses ran out (4.3 billion isn't enough for billions of phones, IoT devices, servers). IPv6 format: `2001:0db8:85a3:0000:0000:8a2e:0370:7334` — 128 bits, effectively unlimited addresses.

### 3.2 Public vs Private IP Addresses

- **Public IP** = unique across the entire internet, assigned by your ISP.
- **Private IP** = only unique within your local network (home/office), reused across millions of different private networks worldwide. Reserved private ranges:
  - `10.0.0.0 – 10.255.255.255`
  - `172.16.0.0 – 172.31.255.255`
  - `192.168.0.0 – 192.168.255.255`

Your laptop at home has a private IP like `192.168.1.5`. Your home router has ONE public IP that the entire outside internet sees. This translation is called **NAT** (covered in Part 8).

### 3.3 Subnetting & CIDR Notation (The Math Part)

This is where most people get scared, but it's simple arithmetic once broken down.

**CIDR notation** looks like `192.168.1.0/24`. The `/24` tells you how many bits (out of 32 total in IPv4) are reserved for the **network portion** — the rest are for individual **hosts** (devices) within that network.

```
192.168.1.0/24
              ^-- /24 means: first 24 bits = network, remaining 8 bits = hosts

32 bits total - 24 network bits = 8 host bits
2^8 = 256 possible addresses in this network
(minus 2 reserved: one for "network address" itself, one for "broadcast address")
= 254 usable host addresses
```

**Common CIDR sizes cheat table:**

| CIDR | Subnet Mask | Total Addresses | Usable Hosts |
|---|---|---|---|
| /24 | 255.255.255.0 | 256 | 254 |
| /25 | 255.255.255.128 | 128 | 126 |
| /26 | 255.255.255.192 | 64 | 62 |
| /16 | 255.255.0.0 | 65,536 | 65,534 |
| /8 | 255.0.0.0 | 16,777,216 | 16,777,214 |

**Quick formula to remember:**
```
Usable hosts = 2^(32 - CIDR number) - 2
```
Example for /26: 2^(32-26) - 2 = 2^6 - 2 = 64 - 2 = 62 ✓

**Why subnetting exists:** Instead of one giant network with millions of devices (slow, insecure, hard to manage broadcast traffic), organizations split their IP range into smaller subnets — e.g., one subnet for the engineering department, one for finance, one for servers — improving security (isolate traffic) and performance (smaller broadcast domains).

### 3.4 Routers & Routing

A **router** connects *different* networks together (unlike a switch, which connects devices *within* one network) and decides the best path to forward a packet toward its destination, hopping through multiple routers across the internet until it reaches the destination network.

```
Your Laptop -> Home Router -> ISP Router -> Backbone Routers -> Destination's ISP -> Destination Server
```

Each router only knows "which direction to send this packet next" (via routing tables), not the full path — this is called **hop-by-hop routing**, similar to asking for directions at each intersection rather than having the entire route memorized upfront.

---

## PART 4: The Transport Layer — TCP vs UDP (Layer 4)

This is one of the most important interview topics. Both TCP and UDP sit on top of IP and add the concept of **ports** (so a computer with one IP address can run many different network programs simultaneously).

### 4.1 TCP (Transmission Control Protocol) — Reliable but Slower

**Guarantees:**
- Data arrives **in order**
- Data arrives **complete** (no missing pieces) — retransmits lost packets automatically
- Detects and corrects errors

**How? The 3-Way Handshake (connection setup):**
```
Client                          Server
  |------ SYN -------------->|   "I want to connect, my starting sequence number is X"
  |<----- SYN-ACK -----------|   "OK, acknowledged X+1, my starting sequence number is Y"
  |------ ACK -------------->|   "Acknowledged Y+1, let's talk"
  |                          |
  |     [connection open]    |
```

**4-Way handshake (connection teardown):**
```
Client                          Server
  |------ FIN -------------->|   "I'm done sending"
  |<----- ACK ----------------|   "OK, acknowledged"
  |<----- FIN ----------------|   "I'm done too"
  |------ ACK -------------->|   "OK, acknowledged, closing now"
```

**Use cases:** Web browsing (HTTP), file transfer, email, WebSocket — anything where correctness matters more than raw speed.

### 4.2 UDP (User Datagram Protocol) — Fast but Unreliable

**No handshake, no guarantee of delivery, no guarantee of order.** Just fires packets ("datagrams") and hopes they arrive. Much lower overhead than TCP.

```
Client ----datagram----> Server   (fire and forget, no confirmation)
Client ----datagram----> Server
Client ----datagram----> Server   (this one might get lost — nobody retries it)
```

**Use cases:** Video calls, live streaming, online gaming, DNS lookups — situations where a little data loss is acceptable, but *delay* waiting for retransmission would be worse (imagine your video call freezing to wait for one lost frame to be resent — better to just skip it and move on).

### 4.3 TCP vs UDP — Interview Comparison Table

| Feature | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented (handshake required) | Connectionless |
| Reliability | Guaranteed delivery, retransmits lost packets | No guarantee |
| Order | Guaranteed in-order delivery | No ordering guarantee |
| Speed | Slower (overhead of guarantees) | Faster (minimal overhead) |
| Header size | Larger (20+ bytes) | Smaller (8 bytes) |
| Use cases | Web, email, file transfer, WebSocket | Video calls, gaming, DNS, streaming |

### 4.4 Ports — How One IP Runs Many Services

A **port** is a number (0–65535) identifying a specific application/service on a device.

**Well-known ports (memorize these — very commonly asked in interviews):**

| Port | Protocol/Service |
|---|---|
| 20/21 | FTP (file transfer) |
| 22 | SSH (secure remote login) |
| 25 | SMTP (sending email) |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 27017 | MongoDB |

### 4.5 Congestion Control (Advanced TCP Behavior)

TCP doesn't just blast data at full speed — it dynamically adjusts sending rate to avoid overwhelming the network. This matters a lot for system design interviews.

**Core mechanisms:**

1. **Slow Start** — TCP begins sending a small amount of data, and **doubles** its sending rate (congestion window) with each successful round-trip, until it either fills the available bandwidth or detects packet loss.

```
Round 1: send 1 packet   -> success
Round 2: send 2 packets  -> success
Round 3: send 4 packets  -> success
Round 4: send 8 packets  -> success
... (exponential growth until loss detected)
```

2. **Congestion Avoidance** — once a threshold is reached (or loss detected), TCP switches from doubling to **linear** growth (add 1 packet per round-trip) — being more cautious to avoid re-triggering congestion.

3. **Fast Retransmit / Fast Recovery** — if TCP detects a lost packet (via duplicate acknowledgments), it retransmits immediately rather than waiting for a full timeout, and reduces its sending rate (but not all the way back to the start) to recover gracefully.

**Why this matters:** This is literally why the internet doesn't collapse under load — every TCP connection self-regulates its speed based on real-time network conditions, backing off cooperatively when congestion is detected rather than continuing to blast data.

---

## PART 5: DNS — How Domain Names Become IP Addresses

### 5.1 The Problem

Computers use IP addresses (`142.250.183.14`), but humans want to type `google.com`. DNS (Domain Name System) is the "phonebook of the internet" — translating human-readable names into IP addresses.

### 5.2 The DNS Lookup Process (Step by Step)

```
You type: google.com

1. Browser checks its own cache -> not found (first time)
2. OS checks its cache -> not found
3. Query sent to a Recursive Resolver (usually your ISP or 8.8.8.8/1.1.1.1)
4. Resolver asks a Root Server: "who handles .com?"
5. Root Server replies: "ask the .com TLD server"
6. Resolver asks TLD Server: "who handles google.com?"
7. TLD Server replies: "ask google's Authoritative Nameserver"
8. Resolver asks Authoritative Nameserver: "what's the IP for google.com?"
9. Authoritative Nameserver replies: "142.250.183.14"
10. Resolver caches this answer and returns it to your browser
11. Browser now connects directly to 142.250.183.14
```

```
   Browser -> Resolver -> Root Server -> TLD Server (.com) -> Authoritative Server -> IP returned
              (this whole chain is usually cached and skipped after the first lookup)
```

### 5.3 Common DNS Record Types

| Record | Purpose |
|---|---|
| A | Maps a domain to an IPv4 address |
| AAAA | Maps a domain to an IPv6 address |
| CNAME | Alias — points one domain to another domain name |
| MX | Mail server for the domain |
| TXT | Arbitrary text (often used for verification, SPF/security records) |
| NS | Which nameservers are authoritative for this domain |

### 5.4 Why DNS Uses UDP (mostly)

DNS queries are small and need to be fast — using UDP avoids the overhead of a TCP handshake for a simple lookup. (DNS does fall back to TCP for larger responses, like zone transfers.)

---

## PART 6: HTTP & TLS — The Application Layer You Actually Build On

### 6.1 HTTP Recap

HTTP (HyperText Transfer Protocol) — request/response protocol, runs on top of TCP (traditionally port 80).

```
Client Request:
GET /page HTTP/1.1
Host: example.com

Server Response:
HTTP/1.1 200 OK
Content-Type: text/html

<html>...</html>
```

### 6.2 Common HTTP Status Codes (Interview Must-Know)

| Code | Meaning |
|---|---|
| 200 | OK — success |
| 201 | Created |
| 301/302 | Redirect (permanent/temporary) |
| 400 | Bad Request — client sent malformed data |
| 401 | Unauthorized — not authenticated |
| 403 | Forbidden — authenticated but not allowed |
| 404 | Not Found |
| 429 | Too Many Requests — rate limited |
| 500 | Internal Server Error |
| 502 | Bad Gateway — upstream server sent invalid response |
| 503 | Service Unavailable |
| 504 | Gateway Timeout |

### 6.3 HTTP/1.1 vs HTTP/2 vs HTTP/3

| Version | Key Change |
|---|---|
| HTTP/1.1 | One request per connection at a time (unless pipelined, rarely used); text-based |
| HTTP/2 | **Multiplexing** — many requests/responses over ONE connection simultaneously, binary framing, header compression |
| HTTP/3 | Runs over **QUIC** (built on UDP, not TCP!) — avoids TCP's head-of-line blocking, faster connection setup |

**Head-of-line blocking** (important interview concept): In HTTP/1.1, if one request is slow, it blocks others queued behind it on the same connection. HTTP/2 fixes this at the HTTP layer via multiplexing, but since HTTP/2 still runs on TCP, a single **lost packet** still blocks all multiplexed streams (because TCP guarantees strict order). HTTP/3 solves this at the transport layer by using QUIC/UDP, where streams are independent — a lost packet in one stream doesn't block others.

### 6.4 TLS/SSL — How HTTPS Actually Secures Data

TLS (Transport Layer Security, successor to SSL) sits between TCP and HTTP, encrypting everything. This is what turns `http://` into `https://`.

**Simplified TLS Handshake:**
```
Client                                    Server
  |----- ClientHello (supported ciphers) ->|
  |<---- ServerHello + Certificate ---------|
  |    (client verifies cert is valid,      |
  |     signed by trusted Certificate       |
  |     Authority)                          |
  |----- Key exchange (agree on shared -->  |
  |       symmetric encryption key)         |
  |<===== Encrypted connection established =>|
```

**Key concepts:**
- **Certificate Authority (CA)** — a trusted third party (like DigiCert, Let's Encrypt) that verifies a server's identity and issues a signed certificate. Your browser trusts a pre-installed list of CAs.
- **Asymmetric encryption** (public/private key pair) is used briefly during the handshake to safely agree on a shared secret.
- **Symmetric encryption** (using that shared secret) is used for the actual data transfer afterward — because it's much faster than asymmetric encryption for bulk data.

**Why this order?** Asymmetric encryption is secure but computationally expensive; symmetric is fast but requires both sides to already share a secret key. TLS uses asymmetric encryption just once, to safely exchange a symmetric key, then switches to fast symmetric encryption for everything else.

---

## PART 7: Load Balancing

### 7.1 Why Load Balancers Exist

A single server can only handle so much traffic. Load balancers distribute incoming requests across multiple backend servers, improving availability (if one server dies, traffic reroutes) and performance (spreads load).

```
                     +---> Server 1
Client --> Load ----+---> Server 2
           Balancer +---> Server 3
```

### 7.2 Common Load Balancing Algorithms

| Algorithm | How it works |
|---|---|
| Round Robin | Requests distributed sequentially, one to each server in turn |
| Least Connections | Send to whichever server currently has the fewest active connections |
| IP Hash | Same client IP always routed to the same server (useful for session consistency) |
| Weighted Round Robin | Servers with more capacity get proportionally more requests |

### 7.3 Layer 4 vs Layer 7 Load Balancing

- **Layer 4 (Transport)** — routes based on IP/port only, doesn't inspect actual content. Faster, less flexible.
- **Layer 7 (Application)** — inspects actual HTTP content (URL path, headers, cookies) to make smarter routing decisions (e.g., route `/api/*` to one server pool, `/images/*` to another). Slightly slower but far more flexible.

### 7.4 Sticky Sessions

As covered in the WebSocket guide — some connections (like WebSocket, or apps storing session state locally on one server) need the SAME client to always hit the SAME backend server. Load balancers support this via cookies or IP-based affinity.

---

## PART 8: NAT (Network Address Translation)

### 8.1 The Problem NAT Solves

Private IP addresses (like `192.168.1.5`) aren't routable on the public internet — there are millions of devices worldwide reusing these same private ranges. NAT lets many devices on a private network share ONE public IP address.

### 8.2 How It Works

```
Laptop (192.168.1.5:54321) ----\
Phone  (192.168.1.6:54322) -----> Router (NAT) ----> Public IP: 203.0.113.5 ----> Internet
Tablet (192.168.1.7:54323) ----/

The router keeps a translation table:
192.168.1.5:54321  <-->  203.0.113.5:61001
192.168.1.6:54322  <-->  203.0.113.5:61002

So when a response comes back to 203.0.113.5:61001, the router knows
to forward it specifically to the laptop at 192.168.1.5:54321.
```

This is also a big reason NAT provides a side benefit of basic security — devices behind NAT aren't directly reachable from the internet unless explicitly configured (port forwarding).

---

## PART 9: CDN (Content Delivery Network)

### 9.1 What It Is

A CDN is a globally distributed network of servers that cache and serve content (images, videos, static files, sometimes full pages) from a location **physically close to the user**, rather than every user hitting one origin server far away.

```
Without CDN:
User in India ------------------------> Origin Server in USA  (slow, high latency)

With CDN:
User in India --> Nearby CDN Edge Server in India  (fast!)
                       |
                       | (only if not cached)
                       v
                  Origin Server in USA (fetched once, then cached)
```

### 9.2 Why It Matters

- **Reduced latency** — data travels a shorter physical distance.
- **Reduced origin server load** — most requests served from cache, never reaching your actual server.
- **Better reliability** — traffic spikes absorbed by the CDN's distributed capacity.

---

## PART 10: Practical Command-Line Tools

These are genuinely used daily by engineers to debug real network issues — worth actually running these yourself.

### 10.1 `ping` — Is the host reachable?

```bash
ping google.com
```
Sends ICMP Echo Request packets and measures round-trip time. Tells you: is the host alive, and roughly how much latency exists.

### 10.2 `traceroute` (Linux/Mac) / `tracert` (Windows) — What path does my traffic take?

```bash
traceroute google.com
```
Shows every router ("hop") your packet passes through on the way to the destination, with the round-trip time to each hop. Extremely useful for diagnosing WHERE a slowdown or failure is happening along the route.

### 10.3 `nslookup` / `dig` — Manual DNS lookups

```bash
nslookup google.com
dig google.com
```
Shows you exactly what IP a domain resolves to, and (with `dig`) detailed DNS record info — useful for debugging DNS misconfigurations.

### 10.4 `netstat` — What connections/ports are active on my machine?

```bash
netstat -an
```
Lists all active network connections and listening ports on your machine — useful for checking "is my server actually listening on port 3000?" or spotting unexpected open connections.

### 10.5 `curl` — Manually make HTTP requests

```bash
curl -v https://example.com
```
Sends an HTTP request from the command line and shows you the full request/response including headers (`-v` = verbose). Extremely useful for testing APIs without needing a browser or Postman.

### 10.6 `ss` (modern Linux replacement for netstat)

```bash
ss -tuln
```
Shows listening TCP (`t`) and UDP (`u`) sockets, faster and more detailed than the older `netstat`.

### 10.7 `ipconfig` (Windows) / `ifconfig` or `ip addr` (Linux/Mac) — Your own network config

```bash
ip addr show
```
Shows your own device's IP address, subnet mask, and network interfaces.

---

## PART 11: Full Journey — What Actually Happens When You Type a URL

This is one of THE most common system-design/networking interview questions. Here's the complete flow, tying everything in this guide together:

```
1. You type "example.com" in the browser and hit Enter

2. DNS RESOLUTION
   Browser checks cache -> OS cache -> Router cache -> ISP Resolver
   -> Root Server -> TLD Server -> Authoritative Server
   -> Returns IP: 93.184.216.34

3. TCP CONNECTION
   Browser initiates 3-way handshake with 93.184.216.34:443
   SYN -> SYN-ACK -> ACK

4. TLS HANDSHAKE (since it's HTTPS)
   ClientHello -> ServerHello + Certificate -> Key exchange
   -> Encrypted channel established

5. HTTP REQUEST SENT
   GET / HTTP/1.1
   Host: example.com
   (over the now-encrypted TCP connection)

6. SERVER PROCESSES REQUEST
   Possibly hits a Load Balancer first, which routes to an
   available backend server, which may query a database,
   check a cache (like Redis), etc.

7. HTTP RESPONSE SENT BACK
   HTTP/1.1 200 OK
   <html>...</html>

8. BROWSER RENDERS THE PAGE
   Parses HTML -> discovers more resources (CSS, JS, images)
   -> repeats steps 2-7 for each (often reusing the same TCP/TLS
      connection thanks to HTTP/1.1 keep-alive or HTTP/2 multiplexing)

9. CONNECTION EVENTUALLY CLOSES OR STAYS ALIVE
   (kept alive briefly for further requests, or closed via
    the TCP 4-way handshake)
```

**This single question can lead into almost every topic in this guide** — DNS, TCP, TLS, HTTP, load balancing, caching, CDNs — so being able to walk through it fluently is one of the highest-leverage things you can prepare.

---

## PART 12: Interview Question Bank (Sharp, Direct Answers)

**Q: Explain the OSI model.**
A: 7 conceptual layers describing how data moves between devices — Physical, Data Link, Network, Transport, Session, Presentation, Application. In practice, most engineers use the simplified 4-layer TCP/IP model: Link, Internet, Transport, Application.

**Q: TCP vs UDP — when would you use each?**
A: TCP guarantees ordered, reliable delivery via handshake and retransmission — use for web traffic, file transfer, anything needing correctness. UDP is connectionless and faster with no delivery guarantee — use for video calls, gaming, DNS, live streaming, where speed matters more than occasional data loss.

**Q: What happens during a TCP handshake?**
A: 3-way handshake: SYN (client requests connection) → SYN-ACK (server acknowledges and responds) → ACK (client confirms) — then the connection is open.

**Q: What is a subnet mask, and what does /24 mean?**
A: A subnet mask defines how many bits of an IP address represent the network vs the host. /24 means the first 24 bits are the network portion, leaving 8 bits (256 addresses, 254 usable) for hosts.

**Q: Explain what happens when you type a URL into a browser.**
A: DNS resolution → TCP handshake → TLS handshake (if HTTPS) → HTTP request sent → server processes (possibly via load balancer/cache/DB) → HTTP response returned → browser renders and fetches additional resources.

**Q: What's the difference between a forward proxy and a reverse proxy?**
A: A forward proxy sits in front of clients, hiding/managing outgoing client requests (e.g., a company proxy controlling employee internet access). A reverse proxy sits in front of servers, managing incoming requests on behalf of the backend (e.g., Nginx routing traffic to backend servers, often paired with load balancing and caching).

**Q: What is NAT and why is it needed?**
A: NAT translates private IP addresses (unique only within a local network) into a single public IP address so multiple devices can share one internet connection, since public IPv4 addresses are scarce.

**Q: How does HTTPS actually secure data?**
A: TLS handshake establishes a trusted, encrypted channel — server presents a certificate signed by a trusted CA, an asymmetric key exchange safely establishes a shared symmetric key, and all subsequent data is encrypted using that fast symmetric key.

**Q: What is head-of-line blocking, and how does HTTP/3 solve it?**
A: When one slow/lost request blocks others queued behind it. HTTP/2 fixed this at the HTTP layer via multiplexing, but a single lost TCP packet still blocks all streams since TCP enforces strict order. HTTP/3 runs on QUIC (UDP-based) where streams are independent, so one lost packet doesn't block unrelated streams.

**Q: Why can't WebSocket connections be load-balanced like normal HTTP requests?**
A: WebSocket connections are stateful and long-lived, pinned to one specific server for their entire duration — unlike stateless HTTP requests where any server can handle any request. This requires sticky sessions and often a Pub/Sub backbone to route messages across server instances.

**Q: What's the difference between Layer 4 and Layer 7 load balancing?**
A: Layer 4 routes based on IP/port only (fast, simple). Layer 7 inspects actual application data like HTTP headers/URL paths, enabling smarter routing (e.g., path-based routing) at the cost of slightly more processing.

**Q: Explain TCP congestion control briefly.**
A: TCP starts sending slowly (Slow Start), doubling its rate each round-trip until it detects congestion/loss, then switches to more cautious linear growth (Congestion Avoidance), and uses Fast Retransmit to recover quickly from isolated packet loss without resetting all the way back to the start.

---

## Quick-Reference Cheat Sheet

1. **OSI 7 layers** (top to bottom): Application, Presentation, Session, Transport, Network, Data Link, Physical. Practical model: Application, Transport, Internet, Link.
2. **TCP** = reliable, ordered, handshake-based, slower. **UDP** = fast, no guarantees, no handshake.
3. **IP address** identifies a device globally; **MAC address** identifies it locally.
4. **CIDR /24** = 256 addresses, 254 usable hosts. Formula: `2^(32-CIDR) - 2`.
5. **DNS** resolves domain names to IPs via a chain: Resolver → Root → TLD → Authoritative server.
6. **HTTP** = app-layer protocol; **TLS** = encrypts it (turns HTTP into HTTPS) via a handshake using asymmetric encryption to exchange a symmetric key.
7. **HTTP/2** = multiplexing over one connection. **HTTP/3** = runs on QUIC/UDP to fix TCP-level head-of-line blocking.
8. **Load balancers** distribute traffic — Layer 4 (fast, IP/port-based) or Layer 7 (smart, content-based).
9. **NAT** lets many private-IP devices share one public IP.
10. **CDN** caches content physically closer to users to reduce latency.
11. Key commands: `ping` (reachability), `traceroute` (path/hops), `dig`/`nslookup` (DNS lookup), `netstat`/`ss` (active connections/ports), `curl` (manual HTTP requests).
12. **"What happens when you type a URL"** ties everything together — practice explaining this fluently.

---
