# 05 - TCP, UDP & Network Communication

IP (doc 04) delivers packets on a best-effort basis: they may be lost, duplicated, delayed or reordered. The **transport layer** turns that into something programs can use. **TCP** builds a reliable, ordered byte stream; **UDP** exposes raw datagram delivery with almost nothing added.

Depends on: doc 03 (ports, sockets, layers), doc 04 (IP).

---

# 5.1 TCP

**TCP (Transmission Control Protocol)** provides:

- **Connection-oriented** communication: a connection is set up before data flows and torn down afterwards.
- **Reliable** delivery: lost data is retransmitted; corrupted data is detected (checksum).
- **Ordered** delivery: bytes arrive in the order sent.
- A **byte stream**: TCP has **no message boundaries**. Ten `send()` calls may arrive as one `recv()`, or one `send()` may arrive as several `recv()`s.
- **Full duplex**: both sides send at the same time over one connection.
- **Flow control** (do not overwhelm the receiver) and **congestion control** (do not overwhelm the network).

## The TCP Header (20 bytes minimum)

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
┌───────────────────────────────┬───────────────────────────────┐
│          Source Port          │       Destination Port        │
├───────────────────────────────┴───────────────────────────────┤
│                        Sequence Number                        │
├───────────────────────────────────────────────────────────────┤
│                    Acknowledgment Number                      │
├───────┬─────────┬─────────────┬───────────────────────────────┤
│Offset │ Reserved│ Flags       │          Window Size          │
├───────┴─────────┴─────────────┼───────────────────────────────┤
│           Checksum            │         Urgent Pointer        │
├───────────────────────────────┴───────────────────────────────┤
│                    Options (MSS, SACK, ...)                   │
└───────────────────────────────────────────────────────────────┘
```

| Field | Meaning |
| --- | --- |
| Source / Destination Port | 16 bits each; with the IPs they form the 5-tuple (doc 03) |
| **Sequence Number** | 32 bits: position in the byte stream of the first data byte of this segment |
| **Acknowledgment Number** | 32 bits: the **next byte the receiver expects** (valid when the ACK flag is set) |
| Data Offset | Header length in 32-bit words |
| **Flags** | `SYN` (open), `ACK` (acknowledges), `FIN` (close), `RST` (abort), `PSH` (deliver now), `URG`, plus `ECE`/`CWR` for congestion signalling |
| **Window Size** | 16 bits: how many more bytes the receiver can buffer (flow control) |
| Checksum | Detects corruption of header and data |
| Options | MSS, window scaling, SACK, timestamps |

## Three-Way Handshake

Before data flows, both sides agree on starting sequence numbers.

```text
   Client                                   Server
     │                                        │  (LISTEN)
     │ ── SYN, seq = x ─────────────────────► │
     │ (SYN_SENT)                             │
     │ ◄──────── SYN+ACK, seq = y, ack = x+1 ─│
     │                                        │ (SYN_RECEIVED)
     │ ── ACK, seq = x+1, ack = y+1 ────────► │
     │ (ESTABLISHED)                          │ (ESTABLISHED)
     │ ══════════ data flows ═══════════════ │
```

- `x` and `y` are **initial sequence numbers (ISNs)**, chosen randomly (this hampers spoofing and confusion with old packets).
- A `SYN` (and a `FIN`) consumes **one** sequence number, which is why the acknowledgment is $x+1$.
- The client can start sending data with (or right after) the third packet. **Cost: 1 RTT** before the first data byte.
- The SYNs also negotiate options: **MSS** (max segment size), **window scaling**, **SACK**.

## Sequence Numbers and ACKs

Sequence numbers count **bytes**, not segments. The receiver replies with **cumulative ACKs**: "I have received everything up to (but not including) byte $N$."

```text
Client sends 3 segments of 500 bytes, starting at seq = 1001:

 seq=1001 len=500  ───►                      ◄─── ack=1501
 seq=1501 len=500  ───►                      ◄─── ack=2001
 seq=2001 len=500  ───► (lost)
 seq=2501 len=500  ───►                      ◄─── ack=2001   (duplicate ACK: still waiting for 2001)
```

$$\text{ack} = \text{seq} + \text{payload length}\quad(\text{for in-order data})$$

The receiver reassembles data by sequence number, so out-of-order arrival is fixed before the application sees anything.

## Retransmission

TCP retransmits a segment when it is not acknowledged. Two triggers:

1. **Timeout (RTO).** The sender keeps a timer. If no ACK arrives before it expires, the segment is resent and the timeout is **doubled** (exponential backoff).
2. **Fast retransmit.** Three **duplicate ACKs** for the same number signal that one segment was lost but later ones arrived, so the sender resends immediately without waiting for the timer.

**SACK** (Selective Acknowledgment) lets the receiver report exactly which ranges arrived, so only the missing bytes are resent.

**Choosing the timeout (RFC 6298).** The sender measures the round-trip time $R$ of each acknowledged segment and maintains a smoothed estimate $\text{SRTT}$ and a variance $\text{RTTVAR}$:

$$\text{RTTVAR}\leftarrow(1-\beta)\,\text{RTTVAR}+\beta\,|\text{SRTT}-R|,\qquad \beta=\tfrac14$$

$$\text{SRTT}\leftarrow(1-\alpha)\,\text{SRTT}+\alpha\,R,\qquad \alpha=\tfrac18$$

$$\text{RTO}=\text{SRTT}+\max(G,\,4\cdot\text{RTTVAR})$$

where $G$ is the clock granularity. A timeout too short causes needless retransmissions; too long makes recovery slow.

## Flow Control (Receiver Window)

The receiver advertises a **window** ($\text{rwnd}$): the free space in its receive buffer. The sender never has more than $\text{rwnd}$ unacknowledged bytes in flight. If the receiver's application reads slowly, the window shrinks (down to 0, when the sender must pause and probe).

The 16-bit field alone caps the window at 65,535 bytes; the **window scale** option multiplies it by $2^{s}$ ($s\le14$), allowing windows up to about 1 GiB.

**Throughput limit from the window.** At most one window can be delivered per round trip:

$$\text{throughput}\le\frac{\text{window}}{\text{RTT}}$$

**Bandwidth-delay product (BDP)** is the amount of data "in the pipe" needed to keep a link full:

$$\text{BDP}=\text{bandwidth}\times\text{RTT}$$

Example: a 1 Gbps link with 50 ms RTT: $\text{BDP}=\dfrac{10^9}{8}\ \text{B/s}\times0.05\ \text{s}=6.25\ \text{MB}$. With an unscaled 65,535-byte window the limit is $\dfrac{65{,}535}{0.05}\approx1.31$ MB/s $\approx10.5$ Mbps, about 1% of the link. This is why window scaling and large buffers matter on long, fast paths.

## Congestion Control

Flow control protects the **receiver**; congestion control protects the **network**. The sender keeps a second window, the **congestion window** $\text{cwnd}$, and sends at most

$$\text{in flight}\le\min(\text{cwnd},\ \text{rwnd})$$

Since the sender cannot see the network, it **infers** congestion from packet loss (or delay) and adapts.

```text
 cwnd
  ▲                         loss (3 dup ACKs)
  │                   /|   ──►  cwnd/2
  │        slow start / |         /|
  │           ___/'''   |   ___  / |  ← congestion avoidance:
  │       __/     ssthresh|/    \/   |    +1 MSS per RTT (linear)
  │    __/                |          |
  └──────────────────────────────────────► time
       exponential growth
```

| Phase | Behavior |
| --- | --- |
| **Slow start** | Start with a small cwnd (initial window is 10 MSS on modern systems). Each ACK increases cwnd by 1 MSS, so cwnd **doubles every RTT** until it reaches `ssthresh` or a loss occurs. |
| **Congestion avoidance** | Above `ssthresh`, grow **linearly**: about +1 MSS per RTT. |
| **On loss (3 dup ACKs)** | Set `ssthresh = cwnd/2`, cut cwnd to that value (fast recovery), continue in avoidance. |
| **On timeout** | Treated as severe: `ssthresh = cwnd/2`, cwnd restarts near 1 MSS, back to slow start. |

This "add a little, halve on loss" rule is **AIMD** (additive increase, multiplicative decrease); it makes competing flows converge to fair shares.

Algorithms: **Reno** (classic), **CUBIC** (Linux default; window grows as a cubic function of time since the last loss), **BBR** (Google; models bandwidth and RTT instead of reacting to loss).

**Slow-start example.** Send 1 MB with MSS = 1460 B, i.e. about 685 segments, starting at cwnd = 10 and ignoring loss and rwnd. Segments per round trip: $10, 20, 40, 80, 160, 320, 640$. The cumulative total first exceeds 685 in the 7th round trip. So a fresh connection needs about 7 RTTs to send 1 MB; at RTT = 50 ms that is about 350 ms, on top of the handshake. **Reusing connections (keep-alive) avoids paying slow start again**, and this is why HTTP connection reuse matters (doc 06).

**Throughput under loss (Mathis approximation).** For loss probability $p$:

$$\text{throughput}\approx\frac{\text{MSS}}{\text{RTT}}\cdot\frac{1.22}{\sqrt{p}}$$

Example: MSS = 1460 B, RTT = 50 ms, $p=10^{-4}$: $\dfrac{1460}{0.05}\cdot\dfrac{1.22}{0.01}=29{,}200\times122\approx3.56$ MB/s $\approx28.5$ Mbps. A high-RTT, lossy path is slow **no matter how much bandwidth it has**.

## Connection Termination

Each direction is closed independently, using `FIN`:

```text
   Client                                   Server
     │ ── FIN ──────────────────────────────► │   client is done sending
     │ (FIN_WAIT_1)                           │
     │ ◄───────────────────────────── ACK ─── │   (server may still send data: "half-close")
     │ (FIN_WAIT_2)                           │ (CLOSE_WAIT)
     │ ◄───────────────────────────── FIN ─── │   server is done too
     │ ── ACK ──────────────────────────────► │ (LAST_ACK → CLOSED)
     │ (TIME_WAIT, then CLOSED)               │
```

An **`RST`** aborts a connection immediately (for example when sending to a closed port, or when an application closes with unread data).

**TCP states you will meet in `ss -tan` output:**

| State | Meaning |
| --- | --- |
| `LISTEN` | Server socket waiting for connections |
| `SYN_SENT` / `SYN_RECV` | Handshake in progress |
| `ESTABLISHED` | Normal data transfer |
| `FIN_WAIT_1/2` | Local side sent FIN, waiting for the peer to finish |
| `CLOSE_WAIT` | Peer closed; **our application has not yet called `close()`** |
| `LAST_ACK` | Waiting for the final ACK after our FIN |
| `TIME_WAIT` | We closed first; waiting so stray packets die out |
| `CLOSED` | Done |

**TIME_WAIT.** The side that closes first stays in `TIME_WAIT` for $2\times\text{MSL}$ (Linux uses a fixed 60 s). It makes sure the last ACK arrives and old duplicates of the connection cannot corrupt a new connection with the same 5-tuple.

## TCP in Practice: What Backend Engineers Hit

**1. TCP is a byte stream, so you must frame messages.** `recv()` can return half a message or two messages. Common framing: a **length prefix** (used below), delimiters (HTTP headers end at a blank line), or fixed size. HTTP itself uses `Content-Length` or chunked encoding (doc 06).

**Python:**

```python
import struct

def recv_exact(sock, n: int) -> bytes:
    buf = bytearray()
    while len(buf) < n:
        chunk = sock.recv(n - len(buf))
        if not chunk:
            raise ConnectionError("peer closed mid-message")
        buf += chunk
    return bytes(buf)

def send_msg(sock, payload: bytes) -> None:
    sock.sendall(struct.pack("!I", len(payload)) + payload)   # 4-byte big-endian length

def recv_msg(sock) -> bytes:
    (length,) = struct.unpack("!I", recv_exact(sock, 4))
    return recv_exact(sock, length)     # in real code, reject lengths above a maximum first
```

**C** (C++ uses the same calls):

```c
#include <arpa/inet.h>
#include <errno.h>
#include <stdint.h>
#include <sys/socket.h>

int recv_all(int fd, void *buf, size_t n) {
    char *p = buf;
    while (n > 0) {
        ssize_t r = recv(fd, p, n, 0);
        if (r == 0) return -1;                       // peer closed
        if (r < 0) { if (errno == EINTR) continue; return -1; }
        p += r;  n -= (size_t)r;                     // recv may return fewer bytes than asked
    }
    return 0;
}

int send_all(int fd, const void *buf, size_t n) {
    const char *p = buf;
    while (n > 0) {
        ssize_t w = send(fd, p, n, MSG_NOSIGNAL);    // MSG_NOSIGNAL: no SIGPIPE if peer is gone
        if (w < 0) { if (errno == EINTR) continue; return -1; }
        p += w;  n -= (size_t)w;                     // send may accept fewer bytes than asked
    }
    return 0;
}

int send_msg(int fd, const void *payload, uint32_t len) {
    uint32_t be = htonl(len);                        // length in network byte order
    if (send_all(fd, &be, sizeof be) < 0) return -1;
    return send_all(fd, payload, len);
}
```

**Node.js** (data arrives as arbitrary chunks):

```js
function frameParser(onMessage) {
  let buf = Buffer.alloc(0);
  return (chunk) => {
    buf = Buffer.concat([buf, chunk]);
    while (buf.length >= 4) {
      const len = buf.readUInt32BE(0);
      if (buf.length < 4 + len) break;               // wait for the rest of this message
      onMessage(buf.subarray(4, 4 + len));
      buf = buf.subarray(4 + len);
    }
  };
}
// socket.on("data", frameParser((msg) => console.log(msg.toString())));
```

**C#:**

```csharp
using System.Buffers.Binary;
using System.Net.Sockets;

static async Task<byte[]> ReadMessageAsync(NetworkStream s)
{
    var header = new byte[4];
    await s.ReadExactlyAsync(header);                          // loops until all 4 bytes are read
    int len = BinaryPrimitives.ReadInt32BigEndian(header);
    var body = new byte[len];
    await s.ReadExactlyAsync(body);
    return body;
}
```

**2. Nagle's algorithm and `TCP_NODELAY`.** Nagle's algorithm holds back small writes until earlier data is acknowledged, to avoid sending many tiny packets. Combined with the receiver's **delayed ACK**, this can add tens of milliseconds to request/response protocols. Latency-sensitive services disable it:

```python
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)     # Python
```

```js
socket.setNoDelay(true);                                       // Node.js
```

```c
int one = 1;
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &one, sizeof one);    // C / C++  (<netinet/tcp.h>)
```

```csharp
client.NoDelay = true;                                         // C# TcpClient
```

**3. Listen backlog and SYN floods.** The kernel keeps a **SYN queue** (half-open connections) and an **accept queue** (finished handshakes waiting for `accept()`); `listen(fd, backlog)` sizes the accept queue, capped by `net.core.somaxconn` on Linux. If the app accepts too slowly the queue fills and new connections are dropped or time out. A **SYN flood** sends many SYNs without completing handshakes; **SYN cookies** let the server survive it without storing state per SYN.

**4. Port exhaustion from `TIME_WAIT`.** A client (or proxy) opening a new connection per request to one destination consumes an ephemeral port for about 60 s after each close. With the Linux default range 32768–60999 ($28{,}232$ ports):

$$\frac{28{,}232\ \text{ports}}{60\ \text{s}}\approx470\ \text{new connections/s per (source IP, destination IP:port)}$$

Fixes: **reuse connections** (keep-alive, connection pools), widen the port range, add source IPs.

**5. `CLOSE_WAIT` piles up = a bug.** It means the peer closed but your code never called `close()`: a leaked socket that will eventually hit the file-descriptor limit (doc 02).

**6. Keep-alive probes.** An idle connection can silently die (a firewall or NAT drops its state). TCP keep-alive sends periodic probes to detect this; application-level heartbeats are more reliable.

**7. Head-of-line blocking.** If one segment is lost, all later bytes wait, even if they arrived. This is a cost of in-order delivery and is a main motivation for QUIC (doc 06).

**Tools:**

```bash
ss -tan                         # all TCP sockets with their states
ss -s                           # summary counts by state
sudo tcpdump -i any port 8080   # capture packets (add -X to see payloads)
```

---

# 5.2 UDP

**UDP (User Datagram Protocol)** adds only **ports** and an optional checksum on top of IP.

- **Connectionless:** no handshake; just send.
- **Datagram:** each `sendto()` becomes exactly one packet, and each `recvfrom()` returns exactly one whole datagram: **message boundaries are preserved**.
- **No guaranteed delivery:** datagrams can be lost, duplicated or reordered, and nothing is retransmitted.
- **No congestion or flow control:** an application can flood the network.
- **Low overhead:** 8-byte header, no connection state.

## The UDP Header (8 bytes)

```text
 0                   1                   2                   3
┌───────────────────────────────┬───────────────────────────────┐
│          Source Port          │       Destination Port        │
├───────────────────────────────┼───────────────────────────────┤
│      Length (header+data)     │           Checksum            │
└───────────────────────────────┴───────────────────────────────┘
```

**Size limits.** The Length field allows 65,535 bytes in total, so the maximum IPv4 UDP payload is:

$$65{,}535-20\ (\text{IP})-8\ (\text{UDP})=65{,}507\ \text{bytes}$$

A datagram larger than the path MTU gets **fragmented** by IP, and losing any fragment loses the whole datagram. To avoid this, keep payloads within the MTU: on Ethernet $1500-20-8=1472$ bytes for IPv4 ($1500-40-8=1452$ for IPv6). This is why classic DNS answers were limited to 512 bytes.

## UDP Servers and Clients in Five Languages

**Python:**

```python
import socket

# server
srv = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
srv.bind(("0.0.0.0", 9000))
while True:
    data, addr = srv.recvfrom(2048)     # one call = exactly one datagram
    srv.sendto(data, addr)              # reply to whoever sent it

# client
c = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
c.settimeout(2)
c.sendto(b"ping", ("127.0.0.1", 9000))
try:
    print(c.recvfrom(2048)[0])
except socket.timeout:
    print("no reply: lost, and UDP will not retry for you")
```

**Node.js:**

```js
const dgram = require("dgram");

const server = dgram.createSocket("udp4");
server.on("message", (msg, rinfo) => server.send(msg, rinfo.port, rinfo.address));   // echo
server.bind(9000);
```

**C** (C++ uses the same calls):

```c
#include <arpa/inet.h>
#include <netinet/in.h>
#include <string.h>
#include <sys/socket.h>

int main(void) {
    int fd = socket(AF_INET, SOCK_DGRAM, 0);
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof addr);
    addr.sin_family      = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port        = htons(9000);
    bind(fd, (struct sockaddr *)&addr, sizeof addr);      // no listen()/accept(): there are no connections

    char buf[2048];
    for (;;) {
        struct sockaddr_in peer;
        socklen_t plen = sizeof peer;                      // reset before each call
        ssize_t n = recvfrom(fd, buf, sizeof buf, 0, (struct sockaddr *)&peer, &plen);
        if (n >= 0) sendto(fd, buf, (size_t)n, 0, (struct sockaddr *)&peer, plen);
    }
}
```

**C#:**

```csharp
using System.Net.Sockets;

using var udp = new UdpClient(9000);
while (true)
{
    UdpReceiveResult r = await udp.ReceiveAsync();                         // one datagram
    await udp.SendAsync(r.Buffer, r.Buffer.Length, r.RemoteEndPoint);      // echo
}
```

Note what is **missing** compared with TCP: no `listen`, no `accept`, no `connect` required, no per-client connection object. One socket serves everyone.

## Reliability on Top of UDP

If an application needs some reliability over UDP, it builds it itself:

- sequence numbers and acknowledgements,
- timeouts and retransmission,
- or, for real-time media, **forward error correction** and simply tolerating loss.

Retransmitting old data is harmful for live audio or video: a late packet is useless. With independent loss probability $p$, a fully reliable protocol needs on average

$$E[\text{transmissions}]=\frac{1}{1-p}$$

sends per packet; for $p=5\%$ that is $\approx1.053$. The cost is mostly **waiting time** (each retry costs at least an RTT), not bandwidth.

**Typical uses of UDP:** DNS queries, DHCP, NTP, VoIP and video calls, live streaming, online games, IoT telemetry, syslog, VPNs such as WireGuard, and **QUIC** (HTTP/3).

UDP also supports **broadcast** and **multicast** (one sender, many receivers), which TCP cannot.

---

# 5.3 TCP vs UDP

| | TCP | UDP |
| --- | --- | --- |
| Connection | Yes (handshake, state) | No |
| Delivery guarantee | Yes (retransmission) | No |
| Ordering | Yes | No |
| Message boundaries | No (byte stream) | Yes (datagrams) |
| Flow control | Yes | No |
| Congestion control | Yes | No (application's job) |
| Header size | 20+ bytes | 8 bytes |
| Startup latency | 1 RTT handshake | None |
| Multicast/broadcast | No | Yes |
| Per-connection kernel state | Yes | None |
| Typical use | Web, APIs, databases, email, SSH, file transfer | DNS, real-time media, games, telemetry, QUIC |

## How to Choose

```text
Must every byte arrive, in order?            → TCP
Is stale data worthless (live audio/video,
  game position updates)?                    → UDP
Tiny request/response, latency critical,
  loss is easy to retry (DNS)?               → UDP
Need many independent streams without one
  lost packet stalling the others?           → QUIC (UDP-based)
Need broadcast/multicast?                    → UDP
```

- **HTTP/1.1 and HTTP/2** run on TCP. **HTTP/3** runs on **QUIC**, which runs on UDP and re-implements reliability, congestion control and encryption in user space, with independent streams so a lost packet blocks only its own stream (doc 06).
- Databases and message brokers (PostgreSQL, Redis, Kafka) use TCP because correctness matters more than the last millisecond.
- Building on UDP means **you** own retransmission, ordering, congestion control and security, which is why most applications use TCP or QUIC unless they have a strong reason not to.

---

# Key Takeaways

- **TCP** = reliable, ordered, full-duplex **byte stream** with a 3-way handshake (1 RTT), cumulative ACKs, retransmission, flow control (`rwnd`) and congestion control (`cwnd`). Sending is limited to $\min(\text{cwnd},\text{rwnd})$.
- Throughput is bounded by $\text{window}/\text{RTT}$; keeping a link full needs a window of at least the bandwidth-delay product.
- New connections start in **slow start**, so short-lived connections are slow: reuse connections.
- TCP has **no message boundaries**: always frame messages and loop until all bytes are read or written.
- Watch for `TIME_WAIT` (port exhaustion), `CLOSE_WAIT` (leaked sockets), full accept queues, and Nagle's algorithm (`TCP_NODELAY`).
- **UDP** = 8-byte header, no connection, no reliability, message boundaries preserved. Ideal when speed or timeliness beats completeness, or when you build your own protocol on top (QUIC).
