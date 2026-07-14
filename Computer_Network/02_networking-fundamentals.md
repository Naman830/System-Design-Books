# Computer Networking — Part 2: Data Link, Network Layer Internals, Routing & Security

> **Continuing from Part 1.** Part 1 covered the OSI/TCP-IP models, TCP vs UDP, IP addressing/CIDR basics, DNS, HTTP/HTTPS/TLS, HTTP/2 & HTTP/3, load balancing, NAT, CDNs, diagnostic tools, and the "what happens when you type a URL" walkthrough.
>
> Part 2 goes **one layer deeper** — into the actual mechanics that make TCP/IP and HTTP possible in the first place: how data gets switched across a network, how the Data Link layer frames and error-checks raw bits, how devices discover each other, how routers actually choose paths, IPv6, TCP's internal state machine, and the security fundamentals that protect all of it.
>
> These are the classic **CS-core "Computer Networks" subject** topics — the ones GATE, campus placements, and SDE interviews at every level test directly.

---

## Table of Contents

- [Part 13: Switching Techniques](#part-13-switching-techniques)
- [Part 14: Data Link Layer — Framing & Error Detection](#part-14-data-link-layer--framing--error-detection)
- [Part 15: Multiple Access Protocols (MAC Sublayer)](#part-15-multiple-access-protocols-mac-sublayer)
- [Part 16: Flow Control & Sliding Window Protocols](#part-16-flow-control--sliding-window-protocols)
- [Part 17: ARP, ICMP, Fragmentation & MTU](#part-17-arp-icmp-fragmentation--mtu)
- [Part 18: Routing Algorithms — Distance Vector vs Link State](#part-18-routing-algorithms--distance-vector-vs-link-state)
- [Part 19: IPv6 — The Future-Proofed Addressing](#part-19-ipv6--the-future-proofed-addressing)
- [Part 20: TCP Deep Dive — Connection Lifecycle & Congestion Control](#part-20-tcp-deep-dive--connection-lifecycle--congestion-control)
- [Part 21: Network Security Fundamentals](#part-21-network-security-fundamentals)
- [Quick-Reference Cheat Sheet — Part 2](#quick-reference-cheat-sheet--part-2)
- [Interview Question Bank — Part 2](#interview-question-bank--part-2)

---

## PART 13: Switching Techniques

Before any of Part 1's protocols can run, the network needs a way to actually move data from one point to another across shared physical links. That's **switching**.

### 13.1 What It Is

Switching is the method by which data is moved through intermediate nodes (switches/routers) between a source and a destination, when a direct dedicated link between every pair of devices isn't feasible.

### 13.2 Why It Exists

You can't wire every device directly to every other device (that's O(n²) cables). Switching lets many devices share the same physical infrastructure efficiently.

### 13.3 The Three Types

```
CIRCUIT SWITCHING          PACKET SWITCHING           MESSAGE SWITCHING
(old telephone network)    (modern internet)          (old telegraph/store-forward)

A ──dedicated path──► B    A ──[pkt1][pkt2][pkt3]──► B  A ──[full msg]──► node ──► node ──► B
Path reserved for         Each packet routed          Entire message stored
entire call duration      independently, may take     fully at each hop before
                           different routes            forwarding
```

| Aspect | Circuit Switching | Packet Switching | Message Switching |
|---|---|---|---|
| Path | Fixed, dedicated for session | Dynamic, per-packet | Dynamic, per-message |
| Setup | Requires connection setup (like a phone call) | No setup, or virtual-circuit setup | No dedicated setup |
| Resource use | Reserved even if idle → wasteful | Shared, efficient (statistical multiplexing) | Shared, but stores whole message |
| Delay | Constant once connected | Variable (jitter) | High (store-and-forward per hop) |
| Failure handling | Entire call drops if link fails | Can reroute around failure | Can reroute, but slow |
| Real-world example | Traditional PSTN telephone calls | The Internet (IP packets) | Old email relay systems, telegrams |

### 13.4 Packet Switching: Datagram vs Virtual Circuit

This is the detail interviewers actually probe:

- **Datagram approach** (what IP does): Every packet is routed independently and may arrive out of order. No "call setup." This is why IP is called **connectionless**.
- **Virtual-circuit approach** (e.g., MPLS, ATM, and logically TCP): A logical path is set up first, all packets follow the same path and arrive in order, but the underlying network is still physically packet-switched.

> 💡 **Key insight interviewers look for:** TCP isn't circuit-switched — it *simulates* a reliable, ordered connection on top of a connectionless, packet-switched IP network. That illusion of a "connection" is entirely built by TCP's sequence numbers, ACKs, and retransmission — the network itself has no idea a "TCP connection" exists.

### 13.5 Advantages & Disadvantages

| | Advantages | Disadvantages |
|---|---|---|
| Circuit | Guaranteed bandwidth, constant low latency | Wasteful when idle, slow setup, doesn't scale |
| Packet | Efficient resource sharing, resilient to failure, scales massively | Variable delay (jitter), possible out-of-order/lost packets, needs reassembly logic |

### 13.6 Common Misconceptions

- ❌ "The internet has dedicated wires per connection." No — packets from thousands of unrelated connections share the same physical link, interleaved.
- ❌ "Packet switching is always faster." Not always — under light, predictable load, circuit switching can have lower, more consistent latency (which is why voice networks used it for so long).

### 13.7 Interview Questions

**Q: Why did the internet choose packet switching over circuit switching?**
A: Efficiency and resilience — many users can share the same links (statistical multiplexing), and if a link fails, packets can reroute. A dedicated circuit per user would not scale to billions of devices.

**Q: Is TCP circuit-switched?**
A: No. TCP is a connection-*oriented* protocol running over a connectionless, packet-switched network (IP). The "connection" is a logical state tracked by the two endpoints, not a physical dedicated path.

### Key Takeaways
- Circuit switching = dedicated path, packet switching = shared/dynamic path.
- The modern internet is packet-switched using the **datagram** model at the IP layer.
- TCP adds connection-oriented *behavior* on top without changing the underlying switching model.

---

## PART 14: Data Link Layer — Framing & Error Detection

Once bits are transmitted physically (Layer 1), the **Data Link layer (Layer 2)** organizes them into meaningful units and catches transmission errors before they propagate upward.

### 14.1 What It Is

The Data Link layer takes a raw stream of bits from the physical layer and packages it into **frames**, then checks each frame for errors introduced during transmission (noise, interference, faulty cabling).

### 14.2 Why It Exists

Physical transmission is never perfect — electrical noise, signal attenuation, and interference can flip bits. Without error detection at this layer, corrupted data would silently flow up to the Network and Transport layers, and you'd have no idea your data was wrong until much later (or never).

### 14.3 Framing — How Data Gets Packaged

A frame typically looks like:

```
┌─────────┬─────────┬──────────────┬──────────┬─────────┐
│ Header  │ Src/Dst │   Payload    │   FCS    │ Trailer │
│ (sync)  │  MAC    │   (Data)     │ (CRC)    │  (end)  │
└─────────┴─────────┴──────────────┴──────────┴─────────┘
```

**The core problem framing solves:** how does the receiver know where one frame ends and the next begins, in a continuous stream of bits?

Two classic solutions:
- **Byte stuffing**: Insert a special escape byte before any byte in the payload that accidentally matches the frame's delimiter, so it isn't misread as "end of frame."
- **Bit stuffing**: Used in protocols like HDLC — if the payload naturally contains the flag pattern `01111110`, the sender inserts an extra `0` after five consecutive `1`s to break the pattern, and the receiver removes it.

### 14.4 Error Detection Techniques

| Technique | How it works | Detects | Overhead |
|---|---|---|---|
| **Parity bit** | Add 1 bit so total number of 1s is even (or odd) | Single-bit errors only | Very low |
| **Checksum** | Sum data in fixed-size blocks, send the complement | Most burst errors, not all | Low |
| **CRC (Cyclic Redundancy Check)** | Treat data as a polynomial, divide by a fixed generator polynomial, send the remainder | Virtually all common error patterns (burst errors especially) | Moderate, but very strong — used in Ethernet, Wi-Fi, ZIP files |

**CRC step-by-step:**
1. Sender and receiver agree on a generator polynomial (e.g., `1101`).
2. Sender appends zeros to data, divides by the generator using XOR (not real division), and the remainder becomes the **CRC checksum**.
3. Sender transmits `data + CRC`.
4. Receiver divides the received bits by the same generator — if the remainder is `0`, the frame is (very likely) error-free.

```python
# Simplified CRC calculation (XOR-based binary division)
def crc_remainder(data_bits: str, generator: str) -> str:
    data = data_bits + '0' * (len(generator) - 1)  # append zeros
    data = list(data)
    gen_len = len(generator)

    for i in range(len(data_bits)):
        if data[i] == '1':
            for j in range(gen_len):
                data[i + j] = str(int(data[i + j]) ^ int(generator[j]))

    return ''.join(data[-(gen_len - 1):])  # remainder = CRC

frame = "11010011101100"
generator = "1011"
print("CRC remainder:", crc_remainder(frame, generator))
```

### 14.5 Error Detection vs Error Correction

- **Detection** (parity, checksum, CRC): "Something's wrong, discard/request resend." Used almost everywhere because it's cheap.
- **Correction** (e.g., **Hamming code**): Can actually pinpoint and fix a single-bit error without retransmission, at the cost of extra redundant bits. Used where retransmission is expensive or impossible — e.g., RAM (ECC memory), deep space communication.

> 💡 Most everyday networking (Ethernet, Wi-Fi, TCP) uses **detection + retransmission**, not correction — it's cheaper to ask the sender to resend a bad frame than to build correction into every layer.

### 14.6 Real-World Examples

- Ethernet frames end with a **Frame Check Sequence (FCS)** — a CRC-32 checksum.
- TCP and IP headers each carry their own (much weaker) checksum field.
- ZIP/PNG files use CRC-32 to detect file corruption.

### 14.7 Common Misconceptions

- ❌ "TCP's checksum makes CRC unnecessary." No — TCP's checksum is a weak 16-bit sum, mainly a last-resort sanity check. The Data Link layer's CRC is what catches almost all real transmission errors, *before* the frame is even accepted.
- ❌ "Checksums fix errors." They only detect them (with the exception of Hamming-style codes designed specifically for correction).

### 14.8 Interview Questions

**Q: Why is CRC preferred over simple checksums?**
A: CRC is mathematically far better at catching burst errors (consecutive bit flips), which are common in real transmission media, whereas simple checksums can miss patterns that cancel out.

**Q: If a frame fails a CRC check, what happens?**
A: The Data Link layer drops the frame silently (it never reaches upper layers). Depending on the protocol, either the link layer requests retransmission (e.g., in some Wi-Fi implementations) or it's left to a higher layer like TCP to notice the missing data and retransmit.

### Key Takeaways
- Framing solves "where does one unit of data end and the next begin."
- CRC is the industry-standard error-detection method because it reliably catches burst errors.
- Detection + retransmission is cheaper and more common than active error correction.

---

## PART 15: Multiple Access Protocols (MAC Sublayer)

### 15.1 What It Is & Why It Exists

When multiple devices share the *same* physical medium (a coax cable, a Wi-Fi channel), you need a rule for **who gets to transmit and when** — otherwise signals collide and corrupt each other. This is the job of the **Medium Access Control (MAC) sublayer**.

### 15.2 ALOHA — The Original (Simplest) Approach

- **Pure ALOHA**: Any device transmits whenever it has data. If a collision happens, both sides wait a random time and retry. Simple, but wasteful — max theoretical efficiency ≈ 18%.
- **Slotted ALOHA**: Time is divided into fixed slots; devices may only start transmitting at the beginning of a slot. Cuts collision windows in half, raising efficiency to ≈ 37%.

### 15.3 CSMA — "Listen Before You Talk"

**Carrier Sense Multiple Access**: before transmitting, a device *listens* to check if the medium is already busy.

- **1-persistent CSMA**: If busy, keep listening continuously and transmit the instant it's free (greedy — still collides often).
- **Non-persistent CSMA**: If busy, wait a random backoff time before checking again (less greedy, adds delay).
- **p-persistent CSMA**: If free, transmit with probability `p` (used in slotted systems, balances the two above).

### 15.4 CSMA/CD — Collision *Detection* (Wired Ethernet's Classic Method)

```
1. Listen to the medium (Carrier Sense)
2. If free → start transmitting
3. WHILE transmitting, keep listening
4. If a collision is detected (signal doesn't match what was sent)
      → immediately stop, send a jam signal
      → wait a random backoff time (exponential backoff)
      → retry from step 1
```

This works because on a wired bus, a device *can* detect a collision while transmitting (voltage on the line looks abnormal).

> ⚠️ **Note:** Modern switched Ethernet with full-duplex links has effectively made CSMA/CD obsolete — each device has its own dedicated link to the switch, so there's no shared medium to collide on. CSMA/CD still matters for interviews because it explains *why* switches replaced hubs.

### 15.5 CSMA/CA — Collision *Avoidance* (Wi-Fi's Method)

Wi-Fi (wireless) **can't** reliably detect collisions the way wired Ethernet can — a transmitting radio can't "hear" a collision over its own outgoing signal. So instead of detecting collisions, Wi-Fi tries to **avoid** them:

```
1. Listen to the medium
2. If busy → wait, then wait an additional RANDOM backoff (contention window)
3. If free → transmit
4. Wait for an explicit ACK from the receiver
5. No ACK received → assume collision/loss → retransmit with a larger backoff
```

Wi-Fi also uses an optional **RTS/CTS (Request to Send / Clear to Send)** handshake to reserve the medium before big transmissions — reducing the classic **"hidden node problem,"** where two devices can each hear the access point but not each other.

### 15.6 Comparison Table

| | CSMA/CD | CSMA/CA |
|---|---|---|
| Used in | Wired Ethernet (legacy, shared-medium) | Wi-Fi (802.11) |
| Detects collisions? | Yes, while transmitting | No — can't detect while sending |
| Strategy | Detect & recover fast | Avoid proactively (backoff + ACK) |
| Extra mechanism | Jam signal | RTS/CTS, ACK requirement |

### 15.7 Real-World Example

Your laptop's Wi-Fi card constantly performs CSMA/CA under the hood every time it sends a packet — that random micro-delay before transmission you never notice is the contention window backoff at work.

### 15.8 Interview Questions

**Q: Why does Wi-Fi use CSMA/CA instead of CSMA/CD?**
A: Wireless radios can't transmit and listen for collisions on the same frequency simultaneously with enough sensitivity (a transmitting radio's own signal overwhelms its receiver) — so Wi-Fi avoids collisions proactively via random backoff and confirms delivery via ACKs, rather than detecting collisions mid-transmission like wired Ethernet.

**Q: What is the hidden node problem?**
A: Two wireless devices (A and C) can each reach the access point but not each other. Both may sense the medium as "free" and transmit simultaneously, causing a collision at the AP that neither device could detect. RTS/CTS handshaking mitigates this.

### Key Takeaways
- MAC protocols exist because a shared medium needs a "who talks when" rule.
- CSMA/CD = detect and recover (wired), CSMA/CA = avoid and confirm (wireless).
- Modern switched Ethernet mostly sidesteps this whole problem via dedicated full-duplex links.

---

## PART 16: Flow Control & Sliding Window Protocols

### 16.1 What It Is & Why It Exists

Even with error-free, well-framed data, there's another problem: what if the **sender is faster than the receiver** can process data? Without a control mechanism, the receiver's buffer overflows and data is lost. **Flow control** paces the sender to match the receiver's capacity.

This is distinct from *congestion control* (Part 20) — flow control is about the **receiver's** capacity; congestion control is about the **network's** capacity.

### 16.2 Stop-and-Wait Protocol

```
Sender                          Receiver
  │──── Frame 1 ────────────────►│
  │◄──────────────── ACK 1 ──────│
  │──── Frame 2 ────────────────►│
  │◄──────────────── ACK 2 ──────│
```

Send one frame, wait for its ACK before sending the next. Simple and reliable — but **terribly inefficient** on high-latency links, because the sender sits idle waiting for each ACK.

### 16.3 Sliding Window — The Fix

Instead of waiting for each ACK individually, the sender is allowed to have **multiple frames "in flight"** at once, up to a **window size (W)**. As ACKs come back, the window "slides" forward, allowing new frames to be sent.

```
Window size = 4

Sent, unACKed: [1][2][3][4]      ← can send these before any ACK arrives
Waiting to send: [5][6][7]...
Already ACKed: [0]

ACK for frame 1 arrives → window slides →
Sent, unACKed: [2][3][4][5]
```

### 16.4 Go-Back-N (GBN)

- Receiver only accepts frames **in order**. If frame 3 is lost, frames 4, 5, 6... arriving after it are **discarded even if received correctly**.
- Sender must resend frame 3 *and everything after it* once a timeout occurs.
- Simple for the receiver (only needs one buffer slot), but wasteful on lossy links.

### 16.5 Selective Repeat (SR)

- Receiver **buffers out-of-order frames** and only requests retransmission of the *specific* missing frame.
- Much more efficient on lossy/high-latency links, but requires more buffer memory and bookkeeping on the receiver side.

### 16.6 Comparison Table

| | Stop-and-Wait | Go-Back-N | Selective Repeat |
|---|---|---|---|
| Frames in flight | 1 | Up to window size W | Up to window size W |
| On single frame loss | Resend that frame | Resend that frame + all after it | Resend only that frame |
| Receiver buffering | Minimal | Minimal (discards out-of-order) | Higher (must buffer out-of-order frames) |
| Efficiency on high-latency links | Poor | Good | Best |
| Real-world analog | Old modems | TCP (in simplified form, older implementations) | Modern TCP with Selective ACK (SACK) |

### 16.7 Connecting to Part 1: TCP Uses a Hybrid of This

Modern TCP essentially runs sliding window flow control (advertised via the **Window Size** field in the TCP header) combined with **Selective Acknowledgment (SACK)** — a TCP option that lets a receiver tell the sender exactly which byte ranges arrived, so only truly missing segments get retransmitted, avoiding old GBN-style waste.

### 16.8 Window Size Formula (Common Exam/Interview Calculation)

For Selective Repeat, the maximum window size to avoid ambiguity with a sequence number space of size `2^n` is:

```
Max window size = 2^(n-1)
```

For Go-Back-N, it's:

```
Max window size = 2^n - 1
```

### 16.9 Interview Questions

**Q: What's the difference between flow control and congestion control?**
A: Flow control matches the sender's rate to the *receiver's* buffer capacity (an endpoint-to-endpoint concern). Congestion control matches the sender's rate to the *network's* capacity to avoid overwhelming routers/links in between.

**Q: Why is Selective Repeat generally preferred over Go-Back-N in modern implementations?**
A: It avoids unnecessary retransmission of already-correctly-received frames, which matters a lot on high-bandwidth, high-latency, or lossy links (e.g., satellite, mobile networks) — GBN wastes significant bandwidth resending data the receiver already has.

### Key Takeaways
- Flow control prevents a fast sender from overwhelming a slow receiver.
- Sliding window lets multiple frames be in flight, dramatically improving throughput over stop-and-wait.
- Go-Back-N is simple but wasteful on loss; Selective Repeat is efficient but more complex — modern TCP behaves like a smarter hybrid via SACK.

---

## PART 17: ARP, ICMP, Fragmentation & MTU

### 17.1 ARP (Address Resolution Protocol) — "Whose MAC address is this IP?"

**What it is:** IP addresses (Layer 3) get you *logically* to the right network segment, but physical delivery on a local network (Ethernet/Wi-Fi) requires a **MAC address** (Layer 2). ARP bridges that gap.

**Why it exists:** Your computer knows the destination's IP (e.g., from DNS), but Ethernet frames need a destination MAC address to actually be delivered on the local network.

**How it works, step by step:**
```
1. Device A wants to send to IP 192.168.1.50, but doesn't know its MAC.
2. A broadcasts an ARP Request to the whole LAN:
   "Who has 192.168.1.50? Tell 192.168.1.10 (me)."
3. Every device on the LAN receives it, but only 192.168.1.50 replies:
   "192.168.1.50 is at MAC AA:BB:CC:DD:EE:FF"
4. A caches this mapping in its ARP table (for a short time) and sends the frame.
```

```bash
# View your ARP cache
arp -a
```

**Security note:** ARP has no authentication — this is exploited in **ARP spoofing/poisoning** attacks (covered in Part 21), where an attacker sends fake ARP replies to redirect traffic through themselves (a classic MITM technique).

### 17.2 ICMP (Internet Control Message Protocol) — The Network's "Error Reporting" Protocol

**What it is:** A Network-layer protocol used for diagnostic and error messages — *not* for carrying application data.

**Why it exists:** IP itself is a "best effort," fire-and-forget protocol with no built-in way to report problems (unreachable host, TTL expired, etc.). ICMP fills that gap.

**Real-world connection to Part 1's tools:**
- `ping` = ICMP Echo Request / Echo Reply.
- `traceroute` = sends packets with increasing TTL and reads the ICMP "Time Exceeded" messages that come back from each hop.

| ICMP Message Type | Meaning |
|---|---|
| Echo Request/Reply | Used by `ping` to test reachability |
| Destination Unreachable | Host/network/port couldn't be reached |
| Time Exceeded | TTL hit 0 — used by `traceroute` |
| Redirect | Tells a host to use a better route |

### 17.3 Fragmentation & MTU

**What it is:** The **Maximum Transmission Unit (MTU)** is the largest packet size a given network link can carry (commonly 1500 bytes for Ethernet). If a packet is larger than the MTU of a link it needs to cross, it must be **fragmented** into smaller pieces.

**Why it matters:** Different network technologies have different MTUs. A packet built for a large-MTU network may hit a smaller-MTU link along its path and need to be split up — or dropped, if fragmentation isn't allowed.

**How it works:**
```
Original packet: 4000 bytes, needs to cross a link with MTU 1500

Router fragments it:
Fragment 1: 1480 bytes of data + IP header, flag "More Fragments = 1"
Fragment 2: 1480 bytes of data + IP header, flag "More Fragments = 1"
Fragment 3: 1040 bytes of data + IP header, flag "More Fragments = 0"

Destination reassembles all fragments using the Identification field
and Fragment Offset before passing data up to the Transport layer.
```

**Path MTU Discovery (PMTUD):** Instead of relying on routers to fragment (slow, resource-intensive), modern systems send packets with the **"Don't Fragment" (DF)** bit set. If a packet is too big for a link, the router drops it and sends back an ICMP "Fragmentation Needed" message — the sender then reduces its packet size and retries, effectively "discovering" the smallest MTU along the path without ever fragmenting in-flight.

```bash
# Test path MTU manually on Linux
ping -M do -s 1472 example.com   # -M do = set DF bit, -s = payload size
```

### 17.4 Common Misconceptions

- ❌ "ARP works across the internet." No — ARP is strictly a **local network (LAN)** protocol. Across different networks, routers handle delivery via IP routing, and ARP only resolves the *next hop's* MAC address.
- ❌ "Ping failing always means the host is down." It could also mean ICMP is simply blocked by a firewall — very common in production environments for security reasons.

### 17.5 Interview Questions

**Q: What's the difference between ARP and DNS?**
A: DNS resolves a domain name to an IP address (works globally, application-layer). ARP resolves an IP address to a MAC address (works only on the local network segment, link-layer).

**Q: What happens if fragmentation is disabled and a packet is too big for a link?**
A: The router drops the packet and sends an ICMP "Fragmentation Needed" (Destination Unreachable, code 4) message back to the sender, which is the basis of Path MTU Discovery.

**Q: Why is ARP considered insecure?**
A: It has no authentication mechanism — any device on the LAN can send a fake ARP reply claiming to own an IP address it doesn't, enabling ARP spoofing/MITM attacks.

### Key Takeaways
- ARP maps IP → MAC on the local network; it's what makes actual frame delivery possible.
- ICMP is the network's error-reporting and diagnostics protocol — it powers `ping` and `traceroute` directly.
- MTU limits force fragmentation or Path MTU Discovery to avoid oversized packets breaking on smaller-MTU links.

---

## PART 18: Routing Algorithms — Distance Vector vs Link State

Part 1 covered NAT and load balancing at the edges. Now: **how do routers in the middle of the network actually decide which path to send a packet down?**

### 18.1 What It Is & Why It Exists

A router connected to multiple networks needs a **routing table** — a map of "to reach network X, send via this next-hop." Building and maintaining that table automatically (as topology and link costs change) is the job of a **routing algorithm/protocol**.

### 18.2 Static vs Dynamic Routing

- **Static routing**: Routes are manually configured by an admin. Simple, predictable, but doesn't adapt to failures — used in small/stable networks.
- **Dynamic routing**: Routers exchange information automatically and adapt routes as the network changes — used everywhere at scale.

### 18.3 Distance Vector Routing (e.g., RIP)

**Core idea:** "Tell your neighbors what you know, and trust them." Each router only knows the *distance* (hop count) to each destination and the *direction* (which neighbor to send through) — not the full topology.

```
Bellman-Ford-based algorithm:

1. Each router starts knowing only directly connected networks (distance 0/1).
2. Periodically, each router sends its ENTIRE routing table to its
   directly connected neighbors.
3. Each router updates its table if a neighbor offers a shorter path:
   new_distance = neighbor's_distance_to_X + cost_to_reach_that_neighbor
4. Repeat until the network converges (no more updates happen).
```

**Famous problem — Count to Infinity:**
```
A --- B --- C     (all connected, cost 1 each)

If the link between B and C breaks:
B thinks: "C is unreachable directly, but A said it could reach C in 2 hops!"
B updates: "I can reach C in 3 hops (via A)"
A then updates based on B's WRONG new info: "I can reach C in 4 hops (via B)"
... this loop of misinformation keeps incrementing ("counting to infinity")
until a maximum hop limit is hit.
```
Fixes: **Split Horizon** (don't advertise a route back to the neighbor you learned it from), **Route Poisoning** (advertise a broken route as infinite cost immediately).

### 18.4 Link State Routing (e.g., OSPF)

**Core idea:** "Don't trust secondhand info — build the full map yourself." Every router floods information about its **direct links only** to the *entire* network, and each router independently builds a complete topology map, then computes shortest paths itself.

```
Dijkstra's-algorithm-based approach:

1. Each router discovers its direct neighbors and link costs.
2. Each router floods this info (Link State Advertisements) to ALL
   routers in the network (not just neighbors).
3. Every router now has the SAME complete map of the network.
4. Each router independently runs Dijkstra's shortest-path algorithm
   from itself to compute the best route to every destination.
```

```python
import heapq

def dijkstra(graph, start):
    distances = {node: float('inf') for node in graph}
    distances[start] = 0
    pq = [(0, start)]

    while pq:
        current_dist, node = heapq.heappop(pq)
        if current_dist > distances[node]:
            continue
        for neighbor, weight in graph[node].items():
            distance = current_dist + weight
            if distance < distances[neighbor]:
                distances[neighbor] = distance
                heapq.heappush(pq, (distance, neighbor))
    return distances

network = {
    'A': {'B': 1, 'C': 4},
    'B': {'A': 1, 'C': 2, 'D': 5},
    'C': {'A': 4, 'B': 2, 'D': 1},
    'D': {'B': 5, 'C': 1}
}
print(dijkstra(network, 'A'))  # shortest cost from A to every router
```

### 18.5 Comparison Table

| | Distance Vector (RIP) | Link State (OSPF) |
|---|---|---|
| Algorithm | Bellman-Ford | Dijkstra |
| What's shared | Entire routing table, to neighbors only | Only direct links, flooded to everyone |
| Router's view | Only "distance + direction" — no full map | Full topology map |
| Convergence speed | Slow, prone to loops (count-to-infinity) | Fast, more stable |
| Scalability | Poor (small networks) | Good (large networks, ISPs) |
| Overhead | Lower CPU/memory, higher bandwidth (periodic full tables) | Higher CPU/memory, lower steady-state bandwidth |

### 18.6 BGP — Routing *Between* Networks (Briefly)

RIP and OSPF are **Interior Gateway Protocols (IGP)** — used *within* a single organization's network (an "Autonomous System"). **BGP (Border Gateway Protocol)** is the **Exterior Gateway Protocol** that routes traffic *between* different Autonomous Systems — it's literally the protocol that holds the global internet together, making routing decisions based on policy and AS-path rather than pure shortest-path.

> 💡 This is why a BGP misconfiguration at a single major ISP can cause internet-wide outages — it's the protocol responsible for "how does my packet get from my ISP to Google's network."

### 18.7 Interview Questions

**Q: Why does OSPF converge faster than RIP?**
A: OSPF gives every router a complete, identical topology map via flooding, so each router computes routes independently and consistently. RIP relies on secondhand, incremental info exchanged only between neighbors, which propagates slowly and can create temporary routing loops.

**Q: Explain the count-to-infinity problem.**
A: In distance-vector routing, when a link fails, routers can keep advertising stale/incorrect routes to each other, each incrementally increasing the "distance," because they trust their neighbor's claim of reachability without knowing the whole topology. This loop continues until a maximum metric is reached. Split horizon and route poisoning mitigate it.

**Q: What's the difference between an IGP and BGP?**
A: IGPs (RIP, OSPF) route *within* a single autonomous system based on metrics like hop count or link cost. BGP routes *between* autonomous systems based on policy, business agreements, and AS-path — it's not purely shortest-path.

### Key Takeaways
- Distance Vector = trust your neighbor's summary (simple, but slow/loop-prone).
- Link State = build your own full map and compute it yourself (fast, scalable, more overhead).
- BGP is the protocol that actually connects separate networks into "the internet."

---

## PART 19: IPv6 — The Future-Proofed Addressing

### 19.1 What It Is & Why It Exists

Part 1 covered IPv4 and CIDR. IPv4 has only **~4.3 billion addresses** (`2^32`) — long exhausted given billions of connected devices worldwide. **IPv6** was designed with a vastly larger address space and several protocol-level improvements.

### 19.2 Address Structure

```
IPv4: 192.168.1.1                                    (32 bits, 4 octets)
IPv6: 2001:0db8:85a3:0000:0000:8a2e:0370:7334         (128 bits, 8 groups of 16 bits, hex)

Shortened form (rules: drop leading zeros, "::" replaces ONE run of
consecutive all-zero groups, only once per address):
2001:db8:85a3::8a2e:370:7334
```

`2^128` addresses — enough to assign trillions of addresses to every human on Earth without running out.

### 19.3 Key Differences from IPv4

| Feature | IPv4 | IPv6 |
|---|---|---|
| Address length | 32-bit | 128-bit |
| Address exhaustion | Yes (needs NAT to cope) | No practical exhaustion |
| Header | Complex, variable options | Simplified, fixed 40-byte base header |
| Fragmentation | Done by routers *or* sender | Only by the **sender** (routers never fragment) |
| Broadcast | Yes | Removed — replaced by multicast |
| Built-in security | Optional (IPSec add-on) | IPSec support built into the spec |
| Address auto-config | Manual/DHCP only | Supports **SLAAC** (Stateless Address Autoconfiguration) |
| Header checksum | Present | Removed (relies on upper-layer/link-layer checks instead, for speed) |

### 19.4 Why NAT Becomes (Mostly) Unnecessary

Part 1 explained NAT exists because IPv4 addresses are scarce, so private devices share one public IP. With IPv6's enormous address space, **every device can get its own globally unique address**, removing the core reason NAT was invented (though NAT66/firewalling concepts still exist for other reasons like privacy and network segmentation).

### 19.5 IPv4 → IPv6 Transition Mechanisms

The world can't switch overnight, so several coexistence strategies exist:

- **Dual Stack**: Devices/routers run both IPv4 and IPv6 simultaneously, choosing whichever is available for a given destination. Most common approach today.
- **Tunneling**: IPv6 packets are encapsulated inside IPv4 packets to cross IPv4-only infrastructure (e.g., 6to4, Teredo).
- **NAT64/DNS64**: Lets IPv6-only clients communicate with IPv4-only servers by translating between the two at the network edge.

### 19.6 Common Misconceptions

- ❌ "IPv6 is just IPv4 with more digits." The header structure, fragmentation rules, and address assignment (SLAAC) are fundamentally different, not just longer.
- ❌ "IPv6 adoption is complete/optional now." Adoption is significant but still partial globally — dual stack remains the practical norm for most production systems.

### 19.7 Interview Questions

**Q: Why was IPv6 necessary?**
A: IPv4's 32-bit address space (~4.3 billion addresses) was insufficient for the number of internet-connected devices worldwide. IPv6's 128-bit space effectively removes address exhaustion as a concern.

**Q: Does IPv6 eliminate the need for NAT?**
A: Largely yes, for the original scarcity reason — but NAT-like mechanisms (NAT66) and firewalls are still used for network segmentation, privacy (address hiding), and policy control, just not out of necessity for address conservation.

**Q: What is SLAAC?**
A: Stateless Address Autoconfiguration — lets an IPv6 device generate its own address automatically using the network prefix advertised by a router plus its own interface identifier, without needing a DHCP server.

### Key Takeaways
- IPv6's 128-bit space effectively solves address exhaustion permanently.
- Simplified headers + sender-only fragmentation improve routing efficiency.
- Dual stack is the dominant real-world transition strategy today.

---

## PART 20: TCP Deep Dive — Connection Lifecycle & Congestion Control

Part 1 introduced the TCP 3-way handshake and a brief mention of Slow Start/Congestion Avoidance. Here's the full internal picture.

### 20.1 TCP Connection State Machine

Every TCP socket internally moves through a well-defined set of states:

```
CLIENT                                          SERVER
CLOSED                                          LISTEN
  │── SYN ─────────────────────────────────────►│
SYN_SENT                                    SYN_RCVD
  │◄──────────────────────────── SYN, ACK ──────│
ESTABLISHED                                       │
  │── ACK ─────────────────────────────────────►│
                                              ESTABLISHED
        ... data transfer happens here ...

  │── FIN ─────────────────────────────────────►│
FIN_WAIT_1                                  CLOSE_WAIT
  │◄─────────────────────────────────── ACK ────│
FIN_WAIT_2                                       │
  │◄─────────────────────────────────── FIN ────│
TIME_WAIT                                    LAST_ACK
  │── ACK ─────────────────────────────────────►│
  │ (waits 2×MSL, then closes)              CLOSED
CLOSED
```

**Why TIME_WAIT matters (classic interview trap):** After closing, a socket lingers in `TIME_WAIT` for a period (2× Maximum Segment Lifetime) to guarantee any delayed/duplicate packets from the old connection are absorbed instead of confusing a *new* connection reusing the same port. This is why rapidly restarting a server on the same port can sometimes fail with "address already in use" — the OS is protecting against exactly this scenario. (`SO_REUSEADDR` is the common fix.)

```bash
# See sockets currently stuck in TIME_WAIT
ss -tan state time-wait
```

### 20.2 Congestion Control — The Full Picture

Part 1 briefly mentioned Slow Start and Congestion Avoidance. Here's the complete algorithm evolution:

| Algorithm | Core idea | Era/Use |
|---|---|---|
| **TCP Tahoe** | Slow Start + Congestion Avoidance + Fast Retransmit; treats any loss as full congestion, resets window to 1 | Original (1988) |
| **TCP Reno** | Adds Fast Recovery — avoids resetting all the way to 1 after an isolated loss, halves the window instead | Widely deployed for decades |
| **TCP Cubic** | Uses a cubic function (not linear) to grow the window, optimized for high-bandwidth, high-latency modern links | Default on Linux today |
| **TCP BBR** (Google) | Models actual bottleneck bandwidth & round-trip time directly instead of reacting to packet loss | Used by Google, YouTube, and increasingly common |

**Slow Start + Congestion Avoidance visualized:**

```
Window
Size
  │                    ╱╲  ← loss detected, window cut in half
  │                  ╱    ╲___
  │                ╱          ╲___  (Congestion Avoidance: +1 per RTT)
  │              ╱
  │            ╱   (Slow Start: doubles every RTT)
  │          ╱
  │________╱____________________________________► Time
```

### 20.3 Sockets & Ports — Connecting the Dots

A **socket** is the OS-level abstraction combining `(source IP, source port, destination IP, destination port, protocol)` that uniquely identifies a connection. This is why a server can handle thousands of simultaneous connections from different clients all hitting port 443 — each is a *distinct socket* because the client side of that tuple differs.

```python
# Minimal TCP socket example — server side
import socket

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind(('0.0.0.0', 5000))
server.listen(5)
print("Listening on port 5000...")

conn, addr = server.accept()   # blocks until a client connects
print(f"Connected by {addr}")
data = conn.recv(1024)
conn.sendall(b"Hello, client!")
conn.close()
```

```python
# Minimal TCP socket example — client side
import socket

client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client.connect(('127.0.0.1', 5000))
client.sendall(b"Hello, server!")
print(client.recv(1024))
client.close()
```

### 20.4 Common Misconceptions

- ❌ "TCP congestion control and flow control are the same thing." They solve different problems (network capacity vs receiver capacity — see Part 16) and use different mechanisms (congestion window vs advertised receive window).
- ❌ "A closed TCP connection frees the port instantly." `TIME_WAIT` deliberately delays full cleanup for correctness.

### 20.5 Interview Questions

**Q: Why does TCP use both a congestion window (cwnd) and a receive window (rwnd)?**
A: `rwnd` reflects what the *receiver's buffer* can handle (flow control); `cwnd` reflects what the *network* can handle without congestion (congestion control). The sender's actual send rate is limited by `min(cwnd, rwnd)`.

**Q: Why is TIME_WAIT necessary, and what problem can it cause?**
A: It ensures old, delayed packets from a closed connection don't get misinterpreted as part of a new connection reusing the same port/tuple. It can cause "address already in use" errors when rapidly restarting a server on the same port under heavy connection churn.

**Q: How does TCP Cubic differ from older Reno-style congestion control?**
A: Cubic grows its congestion window using a cubic (not purely linear) function of time since the last loss event, allowing it to more aggressively and efficiently utilize high-bandwidth, high-latency ("long fat") networks common today, instead of growing too conservatively.

### Key Takeaways
- TCP's state machine (especially TIME_WAIT) explains real production issues like "port already in use."
- Congestion control has evolved significantly beyond the basic Slow Start/Avoidance model from Part 1 — Cubic and BBR are what's actually running today.
- A socket is uniquely identified by the full 5-tuple, which is how one port serves many simultaneous clients.

---

## PART 21: Network Security Fundamentals

Part 1 covered TLS/HTTPS at the application level. Here's the broader security picture across the whole stack.

### 21.1 Symmetric vs Asymmetric Encryption (Deeper Dive)

| | Symmetric | Asymmetric |
|---|---|---|
| Keys | One shared secret key | Public/private key pair |
| Speed | Fast | Much slower |
| Key distribution problem | Hard — how do you share the secret safely? | Solved — public key can be shared openly |
| Example algorithms | AES, ChaCha20 | RSA, ECC |
| Used for | Bulk data encryption | Key exchange, digital signatures |

This is exactly why TLS (Part 1) uses **both**: asymmetric encryption to safely exchange a session key, then symmetric encryption (fast) for the actual bulk data.

### 21.2 Digital Signatures & Certificate Chains

**What it is:** A way to prove a message genuinely came from a specific sender and wasn't altered.

**How it works:**
```
1. Sender hashes the message (e.g., using SHA-256).
2. Sender encrypts that hash with their PRIVATE key → this is the "signature."
3. Sender sends: message + signature.
4. Receiver hashes the received message independently, then decrypts
   the signature using the sender's PUBLIC key.
5. If both hashes match → message is authentic and untampered.
```

**Certificate chains** (relevant to TLS from Part 1): A browser trusts a website's certificate because it's signed by a **Certificate Authority (CA)**, whose own certificate is signed by a **Root CA** already trusted by the OS/browser — forming a **chain of trust** back to a small set of universally trusted roots.

### 21.3 Firewalls

| Type | How it works | Layer |
|---|---|---|
| **Packet-filtering** | Allows/blocks based on IP, port, protocol rules — no memory of past packets | L3/L4 |
| **Stateful** | Tracks active connections; only allows return traffic that matches an established session | L3/L4 |
| **Proxy/Application-layer** | Fully inspects application data (e.g., HTTP content) before forwarding | L7 |
| **Next-Gen Firewall (NGFW)** | Combines stateful inspection + deep packet inspection + intrusion prevention | L3-L7 |

```bash
# Simple stateful firewall rule example (iptables, Linux)
# Allow established/related connections back in
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# Allow new SSH connections in
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
# Drop everything else by default
iptables -P INPUT DROP
```

### 21.4 VPN & IPSec

**What it is:** A VPN creates an encrypted tunnel between two points over an untrusted network (like the public internet), making remote traffic behave as if it's on a private network.

**IPSec** is a suite of protocols that provides this at the **Network layer** (as opposed to TLS, which secures at the Transport/Application boundary):
- **AH (Authentication Header):** Ensures integrity and authenticity, but *not* encryption.
- **ESP (Encapsulating Security Payload):** Provides actual encryption plus authentication — this is what most VPNs actually use.

### 21.5 Common Network Attacks (Connecting Back to Earlier Parts)

| Attack | How it works | Related to |
|---|---|---|
| **ARP Spoofing** | Attacker sends fake ARP replies to redirect LAN traffic through themselves | Part 17 (ARP) |
| **DNS Spoofing/Poisoning** | Attacker injects fake DNS responses to redirect a domain to a malicious IP | Part 1 (DNS) |
| **Man-in-the-Middle (MITM)** | Attacker secretly intercepts/relays traffic between two parties who think they're talking directly | ARP spoofing, weak/absent TLS |
| **SYN Flood (a DoS attack)** | Attacker sends many SYN packets without completing the handshake, exhausting server connection resources | Part 1 (TCP handshake) |
| **DDoS** | Same idea as above, but from many distributed sources simultaneously, overwhelming bandwidth or server resources | — |

### 21.6 Common Misconceptions

- ❌ "A VPN makes you anonymous." It hides your traffic from your local network/ISP and encrypts it in transit — but the VPN provider itself can see your traffic, and it doesn't anonymize you from the destination website if you log in with identifying info.
- ❌ "HTTPS alone protects against all attacks." It protects data in transit (confidentiality + integrity) but does nothing against, say, a compromised endpoint, weak passwords, or application-layer vulnerabilities like SQL injection.

### 21.7 Interview Questions

**Q: How does a SYN flood attack work, and how is it mitigated?**
A: The attacker sends many TCP SYN packets (often with spoofed source IPs) without completing the handshake, filling the server's connection backlog queue and preventing legitimate connections. Mitigations include SYN cookies (not allocating full resources until the handshake completes) and rate limiting.

**Q: What's the practical difference between IPSec and TLS?**
A: IPSec secures traffic at the Network layer, protecting *all* traffic between two hosts/networks transparently (application-agnostic) — commonly used for VPNs. TLS secures at the Transport/Application boundary, protecting a specific application connection (e.g., one HTTPS session), and requires the application to be TLS-aware.

**Q: Explain the role of a Certificate Authority in HTTPS.**
A: A CA is a trusted third party that verifies a website's identity and signs its certificate. Browsers trust that signature because the CA's own certificate is trusted (built into the OS/browser), forming a chain of trust that lets users verify they're talking to the real website, not an impostor.

### Key Takeaways
- Symmetric = fast bulk encryption; asymmetric = safe key exchange and identity verification — TLS uses both.
- Firewalls range from simple packet filters to full application-aware inspection.
- Most classic network attacks (ARP spoofing, DNS spoofing, SYN floods) directly exploit the *lack of authentication* in protocols covered earlier in this guide.

---

## Quick-Reference Cheat Sheet — Part 2

1. **Switching**: Circuit = dedicated path (telephone). Packet = shared, dynamic (internet). TCP simulates a "connection" on top of connectionless IP.
2. **Framing & CRC**: CRC is the industry standard for error *detection* (not correction) — used in Ethernet FCS. Detection + retransmission is far more common than correction.
3. **MAC protocols**: CSMA/CD (wired, detects collisions) vs CSMA/CA (Wi-Fi, avoids collisions via backoff + ACK, since it can't detect them mid-transmission).
4. **Flow control**: Stop-and-Wait (simple, slow) → Sliding Window (multiple frames in flight) → Go-Back-N (resend everything after a loss) → Selective Repeat (resend only the lost frame).
5. **ARP** resolves IP → MAC on the *local* network only. **ICMP** carries error/diagnostic messages (`ping`, `traceroute`).
6. **MTU/Fragmentation**: Oversized packets get fragmented or rejected via Path MTU Discovery (DF bit + ICMP "Fragmentation Needed").
7. **Routing**: Distance Vector (RIP) = trust neighbor's summary, Bellman-Ford, prone to count-to-infinity. Link State (OSPF) = full topology map, Dijkstra, faster convergence. BGP connects separate networks (the "internet" glue).
8. **IPv6**: 128-bit addresses solve exhaustion; simplified header; sender-only fragmentation; SLAAC for autoconfiguration; dual stack is the common transition strategy.
9. **TCP internals**: Full state machine (`SYN_SENT → ESTABLISHED → TIME_WAIT → CLOSED`); congestion control evolved Tahoe → Reno → Cubic → BBR; a socket = 5-tuple, not just a port.
10. **Security**: Symmetric (fast, shared key) + Asymmetric (safe exchange, signatures) = how TLS works. Firewalls: packet-filtering → stateful → application-layer/NGFW. IPSec secures at Network layer (VPNs); TLS secures at Transport/App boundary.
11. **Classic attacks tie back to protocol weaknesses**: ARP spoofing (no auth in ARP), DNS spoofing (no auth in classic DNS), SYN flood (exploits handshake resource allocation).

---

## Interview Question Bank — Part 2 (Sharp, Direct Answers)

**Q: What's the difference between flow control and congestion control?**
A: Flow control paces the sender to match the *receiver's* buffer capacity. Congestion control paces the sender to match the *network's* capacity. TCP uses both simultaneously — actual send rate is `min(cwnd, rwnd)`.

**Q: Why can't Wi-Fi use CSMA/CD like wired Ethernet?**
A: A transmitting radio can't simultaneously listen for collisions on the same frequency with enough sensitivity — its own strong outgoing signal drowns out any incoming collision signal. So Wi-Fi avoids collisions proactively (CSMA/CA: random backoff + explicit ACKs) rather than detecting them mid-transmission.

**Q: Explain the TCP TIME_WAIT state and why it exists.**
A: After a connection closes, the side that initiated closure holds the socket in TIME_WAIT for 2×MSL to absorb any late-arriving duplicate packets from the old connection, preventing them from corrupting a new connection that might reuse the same address/port tuple.

**Q: Distance Vector vs Link State routing — which converges faster and why?**
A: Link State (e.g., OSPF) converges faster because every router builds an identical, complete topology map via flooding and computes routes independently with Dijkstra's algorithm. Distance Vector (e.g., RIP) relies on iterative, secondhand updates between neighbors only, which is slower and prone to routing loops like count-to-infinity.

**Q: What problem does ARP solve, and why is it considered a security weakness?**
A: ARP resolves a local IP address to its corresponding MAC address so Ethernet frames can be delivered. It's insecure because it has no built-in authentication — any device can send a forged ARP reply claiming ownership of an IP, enabling ARP spoofing/MITM attacks.

**Q: Why does IPv6 remove the need for routers to fragment packets?**
A: IPv6 pushes all fragmentation responsibility to the *sending host* — routers along the path simply drop oversized packets and rely on Path MTU Discovery (via ICMPv6) so the sender can adjust packet size proactively, improving router performance and simplifying the header.

**Q: What's the difference between symmetric and asymmetric encryption, and why does TLS use both?**
A: Symmetric encryption (e.g., AES) is fast but requires securely sharing a secret key beforehand. Asymmetric encryption (e.g., RSA) solves the key-distribution problem using public/private key pairs but is much slower. TLS uses asymmetric encryption briefly during the handshake to safely establish a shared symmetric session key, then switches to fast symmetric encryption for the actual data.

**Q: How does a SYN flood attack exploit the TCP handshake?**
A: The attacker sends many SYN packets (often with spoofed source IPs) and never completes the handshake with the final ACK. The server allocates connection state for each half-open request, exhausting its backlog queue and blocking legitimate connections. SYN cookies mitigate this by deferring resource allocation until the handshake is verified.

**Q: What's the difference between a stateful firewall and a packet-filtering firewall?**
A: A packet-filtering firewall evaluates each packet in isolation against static rules (IP/port/protocol). A stateful firewall tracks active connections and only permits return traffic that matches an already-established, legitimate session — significantly reducing the attack surface for spoofed or unsolicited packets.

**Q: Explain Go-Back-N vs Selective Repeat with a concrete example.**
A: With a window of 5 frames and frame 3 lost: Go-Back-N discards frames 4 and 5 even though they arrived correctly, and the sender must resend frames 3, 4, and 5 entirely. Selective Repeat buffers frames 4 and 5 as-is and only requests retransmission of frame 3 — far more bandwidth-efficient on lossy or high-latency links.

---

*End of Part 2. Continues from `networking-fundamentals-complete-guide.md` (Part 1).*
