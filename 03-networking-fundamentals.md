# 03 - Networking Fundamentals

Almost every backend system is a set of programs talking over a network. This document explains what a network is, the vocabulary (host, port, socket), and the layered models (OSI and TCP/IP) that organize how data travels.

> Code in this document uses **C, C++, Python, JavaScript (Node.js) and C#**, because the socket API is where the language differences show most clearly.

---

# 3.1 Network Fundamentals

## Core Vocabulary

| Term | Meaning |
| --- | --- |
| **Network** | Two or more devices connected so they can exchange data. |
| **LAN** (Local Area Network) | A network in a small area (home, office, one data center): high speed, low latency, usually one owner. Typical technologies: Ethernet, Wi-Fi. |
| **WAN** (Wide Area Network) | A network spanning cities, countries or continents, usually built by leasing links from carriers. Higher latency. Example: a company linking its offices. |
| **Internet** | The global "network of networks": thousands of independent networks (called **autonomous systems**) interconnected using the IP protocol. |
| **Host** | Any device attached to a network that has an address (laptop, phone, server, router's management interface). |
| **Client** | A program (or host) that **initiates** a request. |
| **Server** | A program (or host) that **waits for and answers** requests. |
| **Port** | A 16-bit number identifying a specific program/service on a host. |
| **Socket** | The OS endpoint through which a program sends and receives network data. |

"Client" and "server" are **roles**, not hardware types. One program can be both (a web server acts as a client when it calls a database).

```text
   LAN A                    WAN / Internet                  LAN B
 ┌──────────┐            ┌───────────────┐             ┌──────────┐
 │ Laptop   │──┐         │               │         ┌───│ Server   │
 │ Phone    │──┼─Router──┤   Many        ├──Router─┤   │ Database │
 │ Printer  │──┘         │   networks    │         └───│ ...      │
 └──────────┘            └───────────────┘             └──────────┘
```

## Packet Switching

The internet uses **packet switching**: data is split into small chunks called **packets**. Each packet carries source and destination addresses and is routed independently; packets of one message may take different paths and arrive out of order.

Compare with **circuit switching** (old telephone network), where a dedicated path is reserved for the whole call. Packet switching shares links efficiently between many users, but packets can be **delayed, lost, duplicated or reordered**. Making this reliable is the job of protocols such as TCP (doc 05).

## Bandwidth, Latency, Throughput

- **Bandwidth:** the maximum rate a link can carry, measured in bits per second (bps, Mbps, Gbps).
- **Latency:** the time for data to travel from one point to another (or round trip: **RTT**).
- **Throughput:** the rate actually achieved. Always $\le$ bandwidth.

Network speeds use **bits**; file sizes use **bytes**. $1\ \text{Gbps}=125\ \text{MB/s}$.

### Where delay comes from

For each hop, total delay is:

$$d_{\text{nodal}} = d_{\text{proc}} + d_{\text{queue}} + d_{\text{trans}} + d_{\text{prop}}$$

- $d_{\text{proc}}$: time to examine the packet header (microseconds).
- $d_{\text{queue}}$: time waiting in a router buffer (varies; grows sharply with congestion).
- $d_{\text{trans}}$: time to push all bits onto the link.
- $d_{\text{prop}}$: time for the signal to travel the distance.

$$d_{\text{trans}}=\frac{L}{R}\qquad d_{\text{prop}}=\frac{D}{s}$$

where $L$ = packet size in bits, $R$ = link rate in bits/s, $D$ = link length, and $s$ = propagation speed (about $2\times10^{8}\ \text{m/s}$ in optical fiber, roughly two-thirds the speed of light).

**Worked example.** A 1500-byte packet on a 100 Mbps link over 3000 km of fiber:

$$d_{\text{trans}}=\frac{1500\times8}{100\times10^{6}}=0.12\ \text{ms},\qquad d_{\text{prop}}=\frac{3\times10^{6}}{2\times10^{8}}=15\ \text{ms}$$

Propagation dominates over long distances and **cannot be improved with more bandwidth**. The physical lower bound on RTT between two points 3000 km apart is about $2\times15=30$ ms. This is why CDNs and regional deployments exist (docs 52 and 43).

## Ports

A **port** is a 16-bit unsigned integer ($0$ to $65{,}535$, so $2^{16}=65{,}536$ values). An IP address finds the **machine**; the port finds the **program** on it.

| Range | Name | Use |
| --- | --- | --- |
| 0 – 1023 | Well-known ports | Standard services; binding usually needs elevated privileges on Unix |
| 1024 – 49151 | Registered ports | Applications (e.g. 5432 PostgreSQL) |
| 49152 – 65535 | Dynamic / ephemeral | Temporary client-side ports assigned by the OS (Linux default range is 32768–60999) |

Common ports:

| Port | Protocol | Port | Protocol |
| --- | --- | --- | --- |
| 22 | SSH | 443 | HTTPS |
| 25 | SMTP | 3306 | MySQL |
| 53 | DNS | 5432 | PostgreSQL |
| 80 | HTTP | 6379 | Redis |
| 123 | NTP | 27017 | MongoDB |

## Sockets

A **socket** is the programming interface to the network. On Unix a socket is a **file descriptor** (doc 02), so you `read` and `write` it like a file.

A socket is identified by: $(\text{protocol},\ \text{local IP},\ \text{local port})$. A **connection** is uniquely identified by the **5-tuple**:

$$(\text{protocol},\ \text{src IP},\ \text{src port},\ \text{dst IP},\ \text{dst port})$$

Example: `(TCP, 192.168.1.5, 52344, 93.184.216.34, 443)`.

Important consequences:

- A server listening on port 443 can hold **millions** of connections, because each client's tuple is different. It is **not** limited to 65,535 connections.
- A single client IP connecting to one `serverIP:port` can have at most about 28k–64k simultaneous connections (the number of ephemeral source ports). A reverse proxy or load balancer talking to one backend can hit this limit: **ephemeral port exhaustion**.

### Socket API call sequence (TCP)

```text
   Server                              Client
   ──────                              ──────
   socket()                            socket()
   bind(ip, port)
   listen(backlog)
   accept()  ◄──── connection ────────  connect(server ip, port)
   read()/recv()  ◄──── data ─────────  write()/send()
   write()/send() ──── data ────────►   read()/recv()
   close()                             close()
```

- `socket()` creates the endpoint.
- `bind()` attaches it to a local address and port.
- `listen()` marks it as passive and sets the queue of pending connections.
- `accept()` returns a **new** socket for each incoming client; the original keeps listening.

**Byte order.** Network protocols use **big-endian** ("network byte order"), while x86 CPUs are little-endian (doc 01). Convert multi-byte numbers with `htons` / `htonl` (host to network) and `ntohs` / `ntohl` (network to host).

### A TCP echo server and client in five languages

The server sends back whatever it receives.

**C (POSIX):**

```c
#include <arpa/inet.h>
#include <netinet/in.h>
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>

int main(void) {
    int srv = socket(AF_INET, SOCK_STREAM, 0);
    if (srv < 0) { perror("socket"); return 1; }

    int yes = 1;
    setsockopt(srv, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof yes);   // allow quick restart

    struct sockaddr_in addr;
    memset(&addr, 0, sizeof addr);
    addr.sin_family      = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);   // all local interfaces
    addr.sin_port        = htons(8080);         // host -> network byte order

    if (bind(srv, (struct sockaddr *)&addr, sizeof addr) < 0) { perror("bind"); return 1; }
    if (listen(srv, 128) < 0) { perror("listen"); return 1; }

    for (;;) {
        int c = accept(srv, NULL, NULL);        // blocks until a client connects
        if (c < 0) { perror("accept"); continue; }
        char buf[1024];
        ssize_t n;
        while ((n = read(c, buf, sizeof buf)) > 0)
            write(c, buf, (size_t)n);           // echo (real code must handle partial writes)
        close(c);
    }
}
```

**C client:**

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in a = {0};
a.sin_family = AF_INET;
a.sin_port   = htons(8080);
inet_pton(AF_INET, "127.0.0.1", &a.sin_addr);      // text -> binary address
connect(fd, (struct sockaddr *)&a, sizeof a);
write(fd, "hello", 5);
char buf[64];
ssize_t n = read(fd, buf, sizeof buf);
close(fd);
```

**C++:** the standard library has **no networking**. C++ programs use the same POSIX calls as C (or libraries such as Boost.Asio). The idiomatic improvement is wrapping the descriptor in an RAII class (doc 00, A.7) so it can never leak:

```cpp
#include <unistd.h>

class Fd {
    int fd_;
public:
    explicit Fd(int fd) : fd_(fd) {}
    ~Fd() { if (fd_ >= 0) ::close(fd_); }       // closed automatically, even on exceptions
    Fd(const Fd&) = delete;                     // no copies: exactly one owner
    Fd& operator=(const Fd&) = delete;
    Fd(Fd&& o) noexcept : fd_(o.fd_) { o.fd_ = -1; }   // ownership can be moved
    int get() const { return fd_; }
};

// Fd srv(::socket(AF_INET, SOCK_STREAM, 0));   // then bind/listen/accept as in the C version
```

**Python:**

```python
import socket

# server
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as srv:
    srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    srv.bind(("0.0.0.0", 8080))
    srv.listen(128)
    while True:
        conn, addr = srv.accept()
        with conn:
            while data := conn.recv(1024):     # empty bytes means the peer closed
                conn.sendall(data)             # sendall retries until everything is sent

# client
with socket.create_connection(("127.0.0.1", 8080)) as s:
    s.sendall(b"hello")
    print(s.recv(1024))
```

**Node.js (JavaScript):**

```js
const net = require("net");

// server: event-driven, one thread handles all connections (doc 16)
const server = net.createServer((socket) => {
  socket.pipe(socket);                   // echo: copy incoming bytes back
});
server.listen(8080, "0.0.0.0");

// client
const client = net.connect(8080, "127.0.0.1", () => client.write("hello"));
client.on("data", (d) => { console.log(d.toString()); client.end(); });
```

**C# (.NET):**

```csharp
using System.Net;
using System.Net.Sockets;

var listener = new TcpListener(IPAddress.Any, 8080);
listener.Start();

while (true)
{
    using TcpClient client = await listener.AcceptTcpClientAsync();
    using NetworkStream stream = client.GetStream();
    var buf = new byte[1024];
    int n;
    while ((n = await stream.ReadAsync(buf)) > 0)
        await stream.WriteAsync(buf.AsMemory(0, n));
}
```

Design comparison:

| Language | Style |
| --- | --- |
| C / C++ | Manual, explicit system calls; you handle errors, partial reads/writes, cleanup |
| Python | Thin wrapper over the same calls (`sendall` hides partial writes) |
| Node.js | Event-driven; the runtime uses `epoll`/`kqueue` internally (doc 02) |
| C# | `async/await` over the OS's asynchronous I/O |

All the servers above handle **one client at a time**. Real servers handle many concurrently with a thread per connection, a thread pool, or (better for many idle connections) non-blocking I/O with `epoll` (doc 02, 16).

**Useful tools:**

```bash
nc -l 8080                # netcat: quick TCP listener
nc 127.0.0.1 8080         # netcat: quick TCP client
ss -tlnp                  # listening TCP sockets and their processes
ping example.com          # reachability and RTT (ICMP)
traceroute example.com    # the path packets take, hop by hop
curl -v https://example.com
```

---

# 3.2 The OSI Model

The **OSI (Open Systems Interconnection) model** splits network communication into **7 layers**. Each layer has one job, uses the service of the layer below, and offers a service to the layer above. It is a **reference model** used for learning and for naming things ("Layer 4 load balancer", "Layer 7 firewall").

```text
 Layer  Name           Data unit (PDU)   Job
 ─────  ─────────────  ────────────────  ─────────────────────────────────────
   7    Application    Data / message    Protocols apps use (HTTP, DNS, SMTP)
   6    Presentation   Data              Encoding, compression, encryption
   5    Session        Data              Establish/maintain/end sessions
   4    Transport      Segment/Datagram  End-to-end delivery between processes (ports)
   3    Network        Packet            Addressing and routing between networks (IP)
   2    Data Link      Frame             Delivery between neighbouring devices (MAC)
   1    Physical       Bits              Signals on wire, fiber, radio
```

Memory aid (top to bottom): **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing.

## Layer 1 - Physical

Transmits raw bits as electrical, optical or radio signals. Concerns: cables (copper, fiber), connectors, voltage/light levels, frequencies, bit rate.

- Examples: Ethernet cables, fiber optics, Wi-Fi radio, hubs, repeaters.

## Layer 2 - Data Link

Delivers **frames** between two devices on the **same local network**. Adds a header with **MAC addresses** and a trailer with an error-detection check (CRC, called the FCS).

- A **MAC address** is a 48-bit hardware address, written like `3c:22:fb:8a:1e:07`.
- Protocols: Ethernet (IEEE 802.3), Wi-Fi (IEEE 802.11), PPP. **ARP** (Address Resolution Protocol) maps an IPv4 address to a MAC address on the local network (it is often placed between layers 2 and 3).
- Devices: **switches** (forward frames by MAC address), network cards.

**Ethernet frame:**

```text
┌────────────┬────────────┬───────────┬──────────────────┬─────────┐
│ Dest MAC   │ Source MAC │ EtherType │ Payload          │ FCS     │
│ 6 bytes    │ 6 bytes    │ 2 bytes   │ 46–1500 bytes    │ 4 bytes │
└────────────┴────────────┴───────────┴──────────────────┴─────────┘
  EtherType: 0x0800 = IPv4, 0x86DD = IPv6, 0x0806 = ARP
```

Parsing the 14-byte header in Python and C:

```python
import struct

def parse_ethernet(frame: bytes):
    dst, src, ethertype = struct.unpack("!6s6sH", frame[:14])   # "!" = network byte order
    mac = lambda b: ":".join(f"{x:02x}" for x in b)
    return mac(dst), mac(src), hex(ethertype)
```

```c
#include <stdint.h>

struct eth_header {
    uint8_t  dst[6];
    uint8_t  src[6];
    uint16_t ethertype;             /* big-endian on the wire: use ntohs() */
} __attribute__((packed));          /* GCC/Clang: no padding between fields */
```

## Layer 3 - Network

Moves **packets** across **multiple networks** from source host to destination host. Provides logical addressing (IP addresses) and **routing** (choosing the path).

- Protocols: **IP** (IPv4, IPv6), **ICMP** (errors and `ping`), IPsec.
- Devices: **routers**.

Details in doc 04.

## Layer 4 - Transport

Delivers data between **processes** (not just hosts) using **ports**. Provides multiplexing of many conversations over one IP address, and optionally reliability, ordering, flow control and congestion control.

- **TCP:** reliable, ordered byte stream (unit: **segment**).
- **UDP:** unreliable, message-oriented (unit: **datagram**).
- **QUIC** (used by HTTP/3) runs over UDP and implements reliability itself.

Details in doc 05.

## Layer 5 - Session

Establishes, manages and terminates **sessions** (long-lived conversations) between applications, including checkpointing and resuming. In practice this layer is not a separate layer in real protocol stacks; its functions are handled by the application or by protocols such as TLS session resumption, RPC frameworks or cookies/session IDs.

## Layer 6 - Presentation

Translates data between the application's format and the network format: **character encoding** (UTF-8), **serialization** (JSON, Protocol Buffers), **compression** (gzip), and **encryption** (TLS is often placed here, although it is really implemented between the application and transport layers).

## Layer 7 - Application

The layer closest to the user program; defines protocols for specific tasks.

- **HTTP/HTTPS** (web), **DNS** (name lookup), **SMTP/IMAP/POP3** (email), **FTP/SFTP** (files), **SSH** (remote shell), **gRPC**, **WebSocket**, **MQTT**.

> "Layer 7" in load balancing means the balancer understands HTTP (URLs, headers). "Layer 4" means it only looks at IPs and ports. You will see this in docs 22, 28 and 39.

## How Data Moves Through the Layers: Encapsulation

The sender goes **down** the stack; each layer **wraps** the data from above with its own header (and sometimes a trailer). This is **encapsulation**. The receiver goes **up**; each layer strips its header (**decapsulation**).

```text
 SENDER                                                  RECEIVER
 Application  [        HTTP request data        ]        Application
 Transport    [TCP hdr][      HTTP data          ]  ◄──  Transport
 Network      [IP hdr][TCP hdr][   HTTP data     ]        Network
 Data Link    [Eth hdr][IP hdr][TCP hdr][data][FCS]       Data Link
 Physical     101100111010101000110101...                 Physical
                       ─────── wire ───────►
```

Typical header sizes:

| Layer | Header | Size |
| --- | --- | --- |
| Ethernet (L2) | 14-byte header + 4-byte FCS | 18 bytes |
| IPv4 (L3) | minimum | 20 bytes |
| TCP (L4) | minimum | 20 bytes |
| UDP (L4) | fixed | 8 bytes |

**MTU and MSS.** The **MTU** (Maximum Transmission Unit) is the largest layer-3 packet a link can carry; on standard Ethernet it is **1500 bytes**. The **MSS** (Maximum Segment Size) is the largest TCP payload:

$$\text{MSS}=\text{MTU}-\text{IP header}-\text{TCP header}=1500-20-20=1460\ \text{bytes}$$

**Efficiency** of a full-size frame (ignoring preamble and inter-frame gap):

$$\frac{1460}{1460+20+20+14+4}=\frac{1460}{1518}\approx 96.2\%$$

For a tiny 10-byte payload the same headers give $10/68\approx15\%$ efficiency, which is why many small messages are wasteful and batching helps.

**What happens at each device**

| Device | Looks at | Decides using |
| --- | --- | --- |
| Switch (L2) | Ethernet header | Destination MAC |
| Router (L3) | IP header | Destination IP and its routing table |
| Firewall / L4 load balancer | IP + TCP/UDP headers | IPs and ports |
| Reverse proxy / L7 load balancer | Application data | URL, headers, cookies |

When a packet crosses a router, the router **removes the old Ethernet frame and builds a new one** for the next hop (new MAC addresses), but the IP source and destination stay the same (except with NAT, doc 04).

---

# 3.3 The TCP/IP Model

The **TCP/IP model** (also called the Internet protocol suite) is the model **actually implemented** on the Internet. It has **4 layers** and is more practical than OSI.

| TCP/IP layer | What it does | Protocols |
| --- | --- | --- |
| **Application** | Application protocols, data formatting, encryption, sessions | HTTP, HTTPS, DNS, SMTP, SSH, FTP, TLS |
| **Transport** | Process-to-process delivery using ports | TCP, UDP, QUIC |
| **Internet** | Addressing and routing between networks | IP (v4/v6), ICMP, IPsec |
| **Network Access** (Link) | Delivery on the local link, including hardware | Ethernet, Wi-Fi, ARP |

## Mapping to OSI

| OSI layer | TCP/IP layer |
| --- | --- |
| 7 Application | **Application** |
| 6 Presentation | **Application** |
| 5 Session | **Application** |
| 4 Transport | **Transport** |
| 3 Network | **Internet** |
| 2 Data Link | **Network Access** |
| 1 Physical | **Network Access** |

Many books use a 5-layer hybrid (Application, Transport, Network, Link, Physical), splitting Network Access into two.

## OSI vs TCP/IP

| | OSI | TCP/IP |
| --- | --- | --- |
| Layers | 7 | 4 (or 5) |
| Origin | Standards bodies (ISO), designed first | Built and deployed first (ARPANET) |
| Role today | Teaching and terminology | The real protocol stack |
| Session/Presentation | Separate layers | Folded into Application |

## Where You Work as a Backend Engineer

```text
 Your code:      Express / Spring / ASP.NET  ── HTTP, JSON, auth      (Application)
 OS kernel:      TCP/UDP + IP stack          ── ports, connections    (Transport, Internet)
 Hardware/driver: NIC, Ethernet/Wi-Fi                                 (Network Access)
```

Your application talks to the kernel through the **socket API**; the kernel implements TCP, UDP and IP; the NIC driver handles the link layer. Most problems fall into one of these buckets:

| Symptom | Likely layer |
| --- | --- |
| Cable unplugged, no link | Physical / link |
| Wrong subnet, no route, cannot ping | Network (IP) |
| Connection refused / timeout / resets | Transport (TCP, ports, firewall) |
| 4xx/5xx errors, wrong headers | Application (HTTP) |

---

# Key Takeaways

- A **host** has an IP address; a **port** selects the program; a **socket** is the OS endpoint; a **connection** is identified by the 5-tuple.
- Network delay = processing + queuing + transmission + propagation. Propagation is bounded by physics, so distance matters more than bandwidth.
- The internet is **packet-switched**: packets can be lost, delayed, duplicated or reordered, so reliability must be added above IP.
- **OSI (7 layers)** is the reference vocabulary; **TCP/IP (4 layers)** is what runs. Know which layer each protocol and device belongs to.
- Each layer **encapsulates** the data of the layer above with its own header. Headers cost bytes (Ethernet + IP + TCP = 58 bytes per full frame); MSS on Ethernet is 1460 bytes.
- "L4" vs "L7" load balancers differ by what they inspect: ports vs HTTP content.
- Sockets are file descriptors; in every language the same sequence applies: `socket → bind → listen → accept` (server), `socket → connect` (client).
