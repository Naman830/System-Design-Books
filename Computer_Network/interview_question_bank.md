# Computer Network Interview Question Bank

> A **standalone, self-contained** set of the ~50 most important, high-frequency Computer Network interview questions — spanning fundamentals, IP addressing, TCP/UDP, application protocols, switching/routing, security, and real scenario/coding questions. No prior reading required — every answer stands on its own.
>
> Organized layer-by-layer (the way most interviewers actually ask), moving from fundamentals up through application protocols, then across to security and hands-on scenarios.

---

## Table of Contents

1. [Fundamentals & Models](#1-fundamentals--models)
2. [IP Addressing & Subnetting](#2-ip-addressing--subnetting)
3. [Transport Layer — TCP & UDP](#3-transport-layer--tcp--udp)
4. [Application Layer — HTTP, DNS, Email](#4-application-layer--http-dns-email)
5. [Data Link Layer, Switching & Devices](#5-data-link-layer-switching--devices)
6. [Network Layer & Routing](#6-network-layer--routing)
7. [Network Security](#7-network-security)
8. [Scenario-Based & Practical/Coding Questions](#8-scenario-based--practicalcoding-questions)

---

## 1. Fundamentals & Models

### Q1. What is a computer network, and what problem does it actually solve?
A computer network is a collection of interconnected devices that can exchange data and share resources. It solves the problem of isolated computing — without it, every device would be an island, unable to share files, communicate, or access shared services like the internet.

### Q2. Explain the OSI model — all 7 layers, in order.
The OSI (Open Systems Interconnection) model is a conceptual 7-layer framework describing how data moves from one device to another:

| Layer | Name | Responsibility | Example |
|---|---|---|---|
| 7 | Application | User-facing protocols | HTTP, DNS, FTP |
| 6 | Presentation | Data formatting, encryption, compression | TLS, JPEG encoding |
| 5 | Session | Manages sessions/connections between apps | Login sessions |
| 4 | Transport | Reliable/unreliable end-to-end delivery | TCP, UDP |
| 3 | Network | Logical addressing & routing | IP, ICMP |
| 2 | Data Link | Physical addressing, framing, error detection | Ethernet, MAC, ARP |
| 1 | Physical | Raw bit transmission over media | Cables, radio signals |

**Memory trick:** "All People Seem To Need Data Processing" (top to bottom).

### Q3. How does the TCP/IP model differ from OSI?
TCP/IP is the practical, simplified 4-layer model actually used in real networks: **Application, Transport, Internet, Link**. It merges OSI's top three layers (Application/Presentation/Session) into one "Application" layer, and merges OSI's bottom two (Data Link/Physical) into one "Link" layer. OSI is a teaching/reference model; TCP/IP is what the real internet is built on.

### Q4. What is encapsulation and decapsulation?
**Encapsulation**: as data travels *down* the stack on the sending side, each layer wraps the data from the layer above it with its own header (and sometimes trailer) — e.g., Application data → gets a TCP header (becomes a segment) → gets an IP header (becomes a packet) → gets a MAC header (becomes a frame). **Decapsulation** is the exact reverse process on the receiving side, stripping each header off as data moves *up* the stack.

### Q5. What's the difference between bandwidth, throughput, and latency?
- **Bandwidth**: the maximum theoretical data-carrying capacity of a link (e.g., "100 Mbps connection").
- **Throughput**: the actual achieved data rate in practice — always ≤ bandwidth, reduced by overhead, congestion, or errors.
- **Latency**: the time delay for data to travel from source to destination (measured in ms) — independent of bandwidth; a high-bandwidth link can still have high latency (e.g., satellite internet).

### Q6. What's the difference between a protocol and a standard?
A **protocol** is a specific set of rules governing how devices communicate (e.g., TCP, HTTP). A **standard** is a broader, formally agreed-upon specification (often published by a body like IEEE or IETF) that protocols are built to comply with — e.g., 802.11 is the *standard*; a specific Wi-Fi implementation follows it.

---

## 2. IP Addressing & Subnetting

### Q7. What is an IP address, and how is IPv4 structured?
An IP address is a unique logical identifier assigned to a device on a network, enabling routing across networks. IPv4 is a 32-bit address written as four decimal octets (0-255) separated by dots, e.g., `192.168.1.1` — split into a **network portion** and a **host portion**.

### Q8. What is a subnet mask, and what does /24 mean?
A subnet mask defines how many bits of an IP address represent the network vs the host portion. `/24` (CIDR notation) means the first 24 bits are the network part, leaving 8 bits for hosts — giving `2^8 = 256` addresses, of which 254 are usable (network + broadcast address reserved).

### Q9. Explain public vs private IP addresses.
**Public IPs** are globally unique and routable directly on the internet, assigned by ISPs/regional registries. **Private IPs** (e.g., `192.168.x.x`, `10.x.x.x`, `172.16.x.x–172.31.x.x`) are reserved for internal/local networks only, not routable on the public internet — devices using private IPs rely on **NAT** to communicate externally.

### Q10. What is CIDR, and why was it introduced?
CIDR (Classless Inter-Domain Routing) replaced the old rigid classful addressing system (Class A/B/C) with flexible-length prefixes (`/8`, `/24`, `/27`, etc.), allowing address blocks to be sized to actual need instead of forcing organizations into oversized, wasteful fixed blocks — critical for conserving the limited IPv4 address space.

### Q11. **Practical:** Divide `192.168.10.0/24` into 4 equal subnets. Show your work.
```
Need 4 subnets → 2^n ≥ 4 → borrow n = 2 bits
New prefix: /24 + 2 = /26  (mask: 255.255.255.192)
Block size = 256 - 192 = 64

Subnet 1: 192.168.10.0    – 192.168.10.63    (usable: .1–.62)
Subnet 2: 192.168.10.64   – 192.168.10.127   (usable: .65–.126)
Subnet 3: 192.168.10.128  – 192.168.10.191   (usable: .129–.190)
Subnet 4: 192.168.10.192  – 192.168.10.255   (usable: .193–.254)
```
**Formula to remember:** `usable hosts = 2^(host bits) - 2`.

### Q12. What is NAT, and why is it needed?
NAT (Network Address Translation) translates private IP addresses into a single public IP address (and back), allowing many devices on a local network to share one public IP for internet access. It exists because public IPv4 addresses are scarce — without NAT, every single device worldwide would need its own public address.

### Q13. Key differences between IPv4 and IPv6?

| | IPv4 | IPv6 |
|---|---|---|
| Address length | 32-bit (~4.3 billion addresses) | 128-bit (effectively unlimited) |
| Format | Decimal, dotted (`192.168.1.1`) | Hexadecimal, colon-separated |
| NAT dependency | Heavy (address scarcity) | Largely unnecessary |
| Header | More complex, variable options | Simplified, fixed base header |
| Fragmentation | Routers or sender | Sender only |

---

## 3. Transport Layer — TCP & UDP

### Q14. TCP vs UDP — when would you use each?
**TCP** is connection-oriented, guarantees ordered, reliable delivery via handshakes, acknowledgments, and retransmission — use for web browsing, file transfer, email, anything needing correctness. **UDP** is connectionless, faster, with no delivery guarantee — use for video/voice calls, live streaming, gaming, and DNS, where speed and low latency matter more than occasional data loss.

### Q15. Explain the TCP 3-way handshake in detail.
```
Client                          Server
  │── SYN (seq=x) ─────────────►│      "I want to connect, my starting seq is x"
  │◄── SYN-ACK (seq=y, ack=x+1)─│      "OK, my starting seq is y, got your x"
  │── ACK (ack=y+1) ────────────►│      "Confirmed, connection established"
        [ESTABLISHED — data transfer begins]
```
This exchange synchronizes sequence numbers on both sides, confirms both directions of communication work, and is why TCP is called "connection-oriented" even though the underlying IP network is connectionless.

### Q16. Explain TCP connection termination (4-way close).
```
Client                          Server
  │── FIN ──────────────────────►│   "I'm done sending"
  │◄── ACK ──────────────────────│   "Acknowledged"
  │◄── FIN ──────────────────────│   "I'm done too"
  │── ACK ──────────────────────►│   "Acknowledged" → enters TIME_WAIT
```
It's a 4-step (not 3-step) process because each side must independently signal it's finished sending — a connection is full-duplex, so one side can still be sending data after the other has finished. The side that sends the final ACK enters a `TIME_WAIT` state briefly to handle any delayed duplicate packets before fully closing.

### Q17. What's the difference between TCP flow control and congestion control?
**Flow control** paces the sender to match the *receiver's* buffer capacity (using the TCP receive window) — preventing a fast sender from overwhelming a slow receiver. **Congestion control** paces the sender to match the *network's* capacity (using the congestion window, via algorithms like Slow Start) — preventing the sender from overwhelming routers/links in between. They solve different problems and the sender's actual rate is limited by whichever is smaller.

### Q18. What are ports, and what is a socket?
A **port** is a 16-bit number identifying a specific process/service on a device (e.g., port 443 for HTTPS). A **socket** is the full combination of `(source IP, source port, destination IP, destination port, protocol)` — this 5-tuple is what uniquely identifies a single connection, which is why one server can handle thousands of simultaneous clients all connecting to the same port.

### Q19. What's the difference between a stateful and a stateless protocol?
A **stateful** protocol (like TCP) maintains information about the ongoing connection/session across multiple exchanges. A **stateless** protocol (like UDP, or HTTP by default) treats each request/packet independently, with no memory of prior interactions — which is why applications built on stateless protocols (like web apps on HTTP) need extra mechanisms like cookies or sessions to maintain user state.

### Q20. Explain TCP's sliding window and why it improves throughput.
Instead of sending one segment and waiting for its ACK before sending the next (Stop-and-Wait — very slow on high-latency links), TCP allows multiple segments to be "in flight" simultaneously, up to a **window size**. As ACKs arrive, the window "slides" forward, allowing new data to be sent. This dramatically improves throughput on links where waiting for each individual ACK would waste huge amounts of time.

---

## 4. Application Layer — HTTP, DNS, Email

### Q21. Walk through what happens when you type a URL and press Enter.
```
1. DNS Resolution: browser → OS cache → resolver → root → TLD →
   authoritative server → returns the IP address
2. TCP Handshake: SYN → SYN-ACK → ACK with that IP on port 443
3. TLS Handshake (if HTTPS): ClientHello → ServerHello + certificate
   → key exchange → encrypted channel established
4. HTTP Request sent over the encrypted connection
5. Server processes it (possibly via load balancer, cache, DB)
6. HTTP Response returned
7. Browser parses HTML, discovers more resources (CSS/JS/images),
   repeats steps 2-6 for each (often reusing the same connection)
```
This is one of the most common system-design/networking questions — being able to explain it fluently touches almost every other topic in this bank.

### Q22. What is DNS, and how does resolution work?
DNS (Domain Name System) translates human-readable domain names into IP addresses. Resolution follows a hierarchical chain: your device checks its local cache first, then queries a **recursive resolver** (often your ISP's), which — if not cached — queries a **root server**, then a **TLD server** (e.g., `.com`), then the domain's **authoritative server**, which returns the final IP.

### Q23. HTTP vs HTTPS — what's the actual difference?
HTTP transmits data in plaintext — anyone intercepting the traffic can read or modify it. HTTPS is HTTP layered over **TLS**, which encrypts the entire exchange, verifies the server's identity via a certificate, and ensures data integrity — protecting against eavesdropping and tampering.

### Q24. How does HTTP (a stateless protocol) maintain state via cookies/sessions?
Since HTTP itself has no memory between requests, the server sends a **cookie** (a small piece of data) to the client after the first interaction; the browser automatically attaches that cookie to every subsequent request to the same domain. The server uses it to look up **session** data stored server-side (or encoded in the cookie itself) — this is how login state, shopping carts, etc. persist across otherwise-independent HTTP requests.

### Q25. Briefly explain the TLS/SSL handshake.
```
1. ClientHello: client proposes supported TLS versions/cipher suites
2. ServerHello + Certificate: server picks a cipher suite, sends
   its certificate (signed by a trusted CA) to prove identity
3. Key Exchange: client and server use asymmetric cryptography to
   safely agree on a shared SYMMETRIC session key
4. Both sides switch to fast symmetric encryption for all
   subsequent application data
```
Asymmetric encryption is used only briefly (it's slow) to safely establish a key; the bulk of the actual data transfer uses fast symmetric encryption.

### Q26. Difference between HTTP/1.1, HTTP/2, and HTTP/3?
- **HTTP/1.1**: One request per connection at a time (or limited pipelining) — head-of-line blocking at the application level.
- **HTTP/2**: Multiplexes multiple requests over a *single* TCP connection simultaneously — fixes app-level head-of-line blocking, but a single lost TCP packet still blocks all streams (TCP enforces strict order).
- **HTTP/3**: Runs on **QUIC** (built on UDP) instead of TCP — streams are fully independent, so a lost packet only affects its own stream, eliminating head-of-line blocking entirely.

### Q27. Difference between a REST API and a WebSocket connection?
REST APIs are built on stateless HTTP request/response — the client always initiates, gets a response, and the connection typically closes or is reused for the next independent request. **WebSockets** establish a single persistent, full-duplex connection where *either side* can push data at any time without a new request — essential for real-time features like chat apps, live notifications, or collaborative editing.

### Q28. Difference between SMTP, POP3, and IMAP?
**SMTP** sends/relays mail (client → server, server → server). **POP3** retrieves mail by downloading it to the client and typically deleting it from the server. **IMAP** retrieves mail while keeping it synced on the server, so multiple devices see a consistent mailbox state — which is why IMAP is the modern standard for multi-device email access.

---

## 5. Data Link Layer, Switching & Devices

### Q29. Difference between a Hub, a Switch, and a Router?
- **Hub** (Layer 1): Dumb repeater — blasts every incoming signal to all ports, no intelligence, one shared collision domain.
- **Switch** (Layer 2): Learns MAC addresses and forwards frames only to the correct port — each port gets its own collision domain.
- **Router** (Layer 3): Forwards packets between *different* networks based on IP address, using a routing table.

### Q30. What's the difference between a MAC address and an IP address?
A **MAC address** is a physical, hardware-burned identifier (Layer 2) unique to a network interface, used for delivery *within* a local network. An **IP address** is a logical, software-assigned identifier (Layer 3) used for routing *across* different networks — IP gets you to the right network, MAC gets you to the right device on that local network.

### Q31. What is ARP, and how does it work?
ARP (Address Resolution Protocol) resolves an IP address to its corresponding MAC address on the local network. The device broadcasts "Who has IP X? Tell me, IP Y" to the whole LAN; the device owning that IP replies directly with its MAC address, which gets cached briefly for future use. ARP only works within a local network segment — it can't cross routers.

### Q32. What are collision domains and broadcast domains?
A **collision domain** is a network segment where two devices transmitting simultaneously can cause a signal collision (relevant to hubs/shared media). A **broadcast domain** is the set of devices that receive a Layer 2 broadcast frame sent by any one device in it. Switches create separate collision domains per port but keep one shared broadcast domain (unless VLANs are used); routers separate both.

### Q33. What is a VLAN, and why is it used?
A VLAN (Virtual LAN) logically divides a single physical switch into multiple isolated broadcast domains — as if they were separate physical switches — without needing separate hardware. It's used for security (isolating departments/traffic types) and performance (reducing unnecessary broadcast traffic), and is far more flexible/cheaper than physically separate switches.

### Q34. Explain CSMA/CD vs CSMA/CA.
Both are "listen before you talk" media access strategies for shared mediums. **CSMA/CD** (wired Ethernet) actively detects collisions *while transmitting* and immediately stops, backs off randomly, and retries. **CSMA/CA** (Wi-Fi) can't reliably detect collisions mid-transmission (a transmitting radio can't hear over its own signal), so it instead *avoids* them proactively — waiting a random backoff period before transmitting and requiring an explicit ACK to confirm successful delivery.

---

## 6. Network Layer & Routing

### Q35. What is routing, and static vs dynamic routing?
Routing is the process of selecting a path for packets to travel across networks. **Static routing** uses manually configured, fixed routes — simple and predictable but doesn't adapt to failures. **Dynamic routing** uses protocols (RIP, OSPF, BGP) where routers automatically exchange information and adapt routes as the network topology changes — essential at any real scale.

### Q36. Distance Vector vs Link State routing — what's the difference?
**Distance Vector** (e.g., RIP): each router only knows the distance/direction to each destination, learned secondhand from neighbors (Bellman-Ford based) — simple but slow to converge and prone to routing loops (the "count-to-infinity" problem). **Link State** (e.g., OSPF): every router builds a *complete* map of the network topology by flooding link information to everyone, then independently computes shortest paths (Dijkstra based) — faster convergence, more scalable, but higher memory/CPU overhead.

### Q37. What is BGP, and why does it matter?
BGP (Border Gateway Protocol) is the **Exterior Gateway Protocol** that routes traffic *between* different autonomous systems (separate networks/organizations) — it's the protocol that actually holds the global internet together. Unlike interior protocols (RIP/OSPF) which optimize for shortest path, BGP makes routing decisions based on policy and business agreements between networks, which is why a BGP misconfiguration at a major ISP can cause widespread internet outages.

### Q38. What is a default gateway?
The default gateway is the device (typically a router) that a device on a local network sends traffic to when the destination IP is outside its own local network/subnet. It's the "exit point" of the local network onto the wider internet or other networks.

### Q39. What is ICMP, and what is it used for?
ICMP (Internet Control Message Protocol) is a Network-layer protocol for diagnostic and error messages — not for carrying application data. `ping` uses ICMP Echo Request/Reply to test reachability; `traceroute` uses ICMP "Time Exceeded" messages (triggered by increasing TTL values) to map the path/hops to a destination.

---

## 7. Network Security

### Q40. Symmetric vs asymmetric encryption — and why does TLS use both?
**Symmetric encryption** (e.g., AES) uses one shared secret key — fast, but requires securely sharing that key beforehand, which is hard over an insecure channel. **Asymmetric encryption** (e.g., RSA) uses a public/private key pair — solves the key-distribution problem since the public key can be shared openly, but is much slower. TLS uses asymmetric encryption briefly during the handshake to safely establish a shared symmetric key, then switches to fast symmetric encryption for the actual bulk data transfer — getting the security benefits of asymmetric crypto with the speed of symmetric crypto.

### Q41. What is a firewall, and what are the main types?
A firewall filters network traffic based on defined rules to block unauthorized access. **Packet-filtering** firewalls check individual packets against static rules (IP/port/protocol) with no memory of past traffic. **Stateful** firewalls track active connections and only allow return traffic matching an established session. **Application-layer/proxy firewalls** (and Next-Gen Firewalls) fully inspect application data (like HTTP content) for deeper, smarter filtering.

### Q42. What is a VPN, and how does it work?
A VPN (Virtual Private Network) creates an encrypted tunnel between a device and a remote network over an untrusted network (like the public internet), making remote traffic behave as if it were on a private, local network. It typically uses **IPSec** or **TLS**-based protocols to encrypt and authenticate the traffic end-to-end, hiding it from anyone on the path (including your local ISP).

### Q43. What is a DDoS attack, and how can it be mitigated?
A DDoS (Distributed Denial of Service) attack floods a target with traffic from many distributed sources simultaneously, overwhelming its bandwidth or server resources so legitimate users can't get through. Mitigation includes rate limiting, traffic filtering/scrubbing services, distributing load across a CDN, and techniques like SYN cookies to resist connection-exhaustion variants (SYN floods).

### Q44. What is a Man-in-the-Middle (MITM) attack?
An attacker secretly intercepts and potentially alters communication between two parties who believe they're communicating directly with each other. Common enabling techniques include ARP spoofing (redirecting local traffic through the attacker) and DNS spoofing (redirecting a domain to a malicious IP). Properly implemented HTTPS/TLS with certificate validation is the primary defense, since it authenticates the server's identity and encrypts the channel.

### Q45. What is a digital certificate, and what role does a CA play in HTTPS?
A digital certificate cryptographically binds a public key to an identity (e.g., a website's domain), and is signed by a trusted **Certificate Authority (CA)**. When your browser connects to an HTTPS site, it verifies the certificate's signature chains back to a CA it already trusts (built into the OS/browser) — this **chain of trust** is what lets you verify you're actually talking to the real website and not an impostor.

---

## 8. Scenario-Based & Practical/Coding Questions

### Q46. **Scenario:** A website is loading very slowly. How would you debug it, layer by layer?
```
1. DNS resolution — is the domain resolving quickly? (use `dig`/`nslookup`)
2. Connectivity — can you reach the server at all? (`ping`, though ICMP
   may be blocked — not conclusive alone)
3. TCP handshake — is the SYN/SYN-ACK/ACK completing quickly?
   (check with a packet capture tool like Wireshark)
4. TLS handshake — if HTTPS, is certificate negotiation slow?
5. Server response time — once connected, how long until the first
   byte of the HTTP response? (points to backend/DB slowness if high)
6. Content/resources — is the page loading many large resources
   sequentially instead of in parallel? (browser dev tools Network tab)
```
This systematic, layer-by-layer approach isolates *where* the bottleneck actually is instead of guessing.

### Q47. **Scenario:** Two devices on the same LAN can't reach each other. What do you check?
```
1. Physical connectivity — cables/Wi-Fi actually connected?
2. IP configuration — are both devices actually on the same subnet?
   (check IP + subnet mask on both)
3. ARP — can each device resolve the other's MAC address?
   (`arp -a` to check the ARP cache/table)
4. Firewall — is a host-based firewall blocking the traffic on
   either device?
5. Switch/VLAN config — are both devices actually on the same VLAN?
   (if on different VLANs, they need a router/L3 switch to communicate,
   not just a switch)
```

### Q48. **Scenario:** You restart a server and immediately get "Address already in use." Why, and how do you fix it?
This happens because the previous TCP connection on that port is still sitting in the **TIME_WAIT** state — after closing, the OS holds the socket briefly (2× Maximum Segment Lifetime) to safely absorb any delayed/duplicate packets from the old connection before fully releasing the port. Fix: either wait it out, or set the `SO_REUSEADDR` socket option so the OS allows immediate reuse of an address in `TIME_WAIT`.

### Q49. **Coding:** Write a minimal TCP client-server socket program in Python.
```python
# server.py
import socket

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind(('0.0.0.0', 5000))
server.listen(5)
print("Listening on port 5000...")

conn, addr = server.accept()
print(f"Connected by {addr}")
data = conn.recv(1024)
print("Received:", data.decode())
conn.sendall(b"Hello from server!")
conn.close()
```
```python
# client.py
import socket

client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client.connect(('127.0.0.1', 5000))
client.sendall(b"Hello from client!")
print("Received:", client.recv(1024).decode())
client.close()
```
This demonstrates the core TCP flow: `bind` + `listen` + `accept` on the server, `connect` on the client, then bidirectional `send`/`recv` — the actual application-level use of everything covered in the TCP handshake question above.

### Q50. **Practical:** Given `172.16.0.0/22`, how many usable hosts, and what's the address range?
```
/22 means 22 network bits, leaving 32 - 22 = 10 host bits
Total addresses = 2^10 = 1024
Usable hosts = 1024 - 2 = 1022  (subtract network + broadcast address)

Range: 172.16.0.0 – 172.16.3.255
   Network address:   172.16.0.0
   First usable host: 172.16.0.1
   Last usable host:  172.16.3.254
   Broadcast address: 172.16.3.255
```

---

## Quick Revision Checklist

Before an interview, make sure you can confidently explain, from memory and without notes:
- [ ] The OSI model layers and what each one does
- [ ] TCP 3-way handshake and 4-way termination
- [ ] TCP vs UDP, and *why* each is used where it is
- [ ] Subnetting math (`2^n - 2` for hosts, CIDR block sizes)
- [ ] "What happens when you type a URL" — end to end
- [ ] DNS resolution chain
- [ ] TLS handshake and why it uses both encryption types
- [ ] Switch vs Router vs Hub, and collision vs broadcast domains
- [ ] Distance Vector vs Link State routing
- [ ] Symmetric vs asymmetric encryption
- [ ] At least one full debugging scenario, talked through out loud

If every box above feels comfortable to explain to someone else in plain language, you're genuinely interview-ready on computer networks.
