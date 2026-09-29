# 04 — IP Networking

IP (Internet Protocol) gives every host an address and moves packets across networks toward it. This document covers addressing (IPv4/IPv6, subnets, CIDR), routing (routers, gateways, NAT) and DNS (turning names into addresses).

Depends on: doc 03 (layers, packets, ports).

---

# 4.1 IP Addressing

## What an IP Address Is

An **IP address** identifies a network interface at layer 3. It has two logical parts:

$$\underbrace{\text{network part}}_{\text{which network}} \;\Big|\; \underbrace{\text{host part}}_{\text{which device in that network}}$$

## IPv4

- **32 bits**, written as four decimal numbers (octets) separated by dots: `192.168.10.77`.
- Each octet is 8 bits ($0$–$255$).
- Total addresses: $2^{32}=4{,}294{,}967{,}296$ (about 4.3 billion). This is fewer than the number of internet-connected devices, and the free pool of public addresses ran out (IANA exhausted its pool in 2011). Solutions: **NAT** (4.2) and **IPv6**.

Conversion example: $192.168.10.77$ in binary is `11000000.10101000.00001010.01001101`. As one integer:

$$192\cdot2^{24}+168\cdot2^{16}+10\cdot2^{8}+77=3{,}232{,}238{,}669$$

### IPv4 header (20 bytes minimum)

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
┌───────┬───────┬───────────────┬───────────────────────────────┐
│Version│  IHL  │  DSCP / ECN   │         Total Length          │
├───────┴───────┴───────────────┼───┬───────────────────────────┤
│        Identification         │Flg│      Fragment Offset      │
├───────────────┬───────────────┼───┴───────────────────────────┤
│      TTL      │   Protocol    │        Header Checksum        │
├───────────────┴───────────────┴───────────────────────────────┤
│                       Source Address (32)                     │
├───────────────────────────────────────────────────────────────┤
│                    Destination Address (32)                   │
└───────────────────────────────────────────────────────────────┘
```

| Field | Meaning |
| --- | --- |
| Version | 4 |
| IHL | Header length in 32-bit words (5 means 20 bytes) |
| Total Length | Header + payload, up to 65,535 bytes |
| Identification, Flags, Fragment Offset | Used when a packet is split (**fragmented**) to fit a smaller MTU |
| **TTL** (Time To Live) | Decremented by every router; when it reaches 0 the packet is dropped. Prevents endless loops. `traceroute` exploits this. |
| **Protocol** | What the payload is: 6 = TCP, 17 = UDP, 1 = ICMP |
| Header Checksum | Detects header corruption (recomputed at each hop) |
| Source / Destination | The two IP addresses |

Parsing it in Python:

```python
import socket
import struct

def parse_ipv4(pkt: bytes) -> dict:
    (ver_ihl, tos, total_len, ident, flags_frag,
     ttl, proto, checksum, src, dst) = struct.unpack("!BBHHHBBH4s4s", pkt[:20])
    return {
        "version": ver_ihl >> 4,
        "header_len": (ver_ihl & 0x0F) * 4,        # IHL counts 4-byte words
        "total_len": total_len,
        "ttl": ttl,
        "protocol": proto,                          # 6 = TCP, 17 = UDP, 1 = ICMP
        "src": socket.inet_ntoa(src),
        "dst": socket.inet_ntoa(dst),
    }
```

In C, the system header `<netinet/ip.h>` provides `struct ip` (or `struct iphdr` on Linux). Bit-field layout in C structs is implementation-defined, which is why protocol code often extracts fields with shifts and masks instead.

## IPv6

- **128 bits**, written as **eight groups of four hex digits** separated by colons.
- Total addresses: $2^{128}\approx3.4\times10^{38}$.
- Fixed 40-byte header (simpler than IPv4's variable header); **no broadcast**; routers do **not** fragment packets; built-in autoconfiguration (SLAAC); Neighbor Discovery replaces ARP.

**Shortening rules**

1. Drop leading zeros in each group.
2. Replace **one** run of consecutive all-zero groups with `::` (only once per address).

```text
2001:0db8:0000:0000:0000:ff00:0042:8329
→ 2001:db8::ff00:42:8329
```

| IPv6 address / prefix | Meaning |
| --- | --- |
| `::1` | Loopback |
| `::` | Unspecified |
| `fe80::/10` | Link-local (valid only on the local link) |
| `fc00::/7` (in practice `fd00::/8`) | Unique local (IPv6's "private") |
| `2000::/3` | Global unicast (public) |
| `ff00::/8` | Multicast |
| `2001:db8::/32` | Reserved for documentation |

A typical LAN gets a **/64**, leaving 64 bits for hosts.

## Public vs Private IPs

- **Public IP:** globally unique and routable on the internet. Assigned via ISPs / regional registries or cloud providers.
- **Private IP:** usable inside any private network, **not routable** on the internet. Many networks reuse the same private ranges.

| Range | CIDR | Purpose |
| --- | --- | --- |
| 10.0.0.0 – 10.255.255.255 | `10.0.0.0/8` | Private (about 16.7 million addresses) |
| 172.16.0.0 – 172.31.255.255 | `172.16.0.0/12` | Private (about 1 million) |
| 192.168.0.0 – 192.168.255.255 | `192.168.0.0/16` | Private (65,536) |
| 100.64.0.0 – 100.127.255.255 | `100.64.0.0/10` | Carrier-grade NAT shared space |
| 169.254.0.0 – 169.254.255.255 | `169.254.0.0/16` | Link-local (auto-assigned when DHCP fails; cloud metadata endpoint `169.254.169.254`) |
| 224.0.0.0 – 239.255.255.255 | `224.0.0.0/4` | Multicast |
| 127.0.0.0 – 127.255.255.255 | `127.0.0.0/8` | Loopback |

Documentation-only ranges (safe for examples): `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`.

## Loopback and 0.0.0.0

- **Loopback (`127.0.0.1`, hostname `localhost`; IPv6 `::1`):** traffic to it never leaves the machine; it goes through the OS network stack and comes right back. Used for local testing and local-only services.
- **`0.0.0.0`:** as a *bind* address it means "**all** network interfaces".

This distinction matters for deployment:

```text
Server bound to 127.0.0.1:8080  → reachable only from the same machine
Server bound to 0.0.0.0:8080    → reachable from any interface (other machines, containers)
```

A very common Docker mistake: an app inside a container listens on `127.0.0.1`, so the host cannot reach it. Bind to `0.0.0.0` inside containers (doc 29).

## Subnets, Subnet Masks and CIDR

A **subnet** is a smaller network carved out of a larger address range. Subnetting reduces broadcast traffic, improves security (isolation) and organizes addresses.

**Subnet mask:** 32 bits with 1s marking the network part and 0s marking the host part. Example: `255.255.255.192` = `11111111.11111111.11111111.11000000`.

**CIDR notation** (Classless Inter-Domain Routing) writes the number of network bits after a slash: `192.168.10.0/26` means the first 26 bits are the network and the last $32-26=6$ bits are the host.

For a prefix length $n$:

$$\text{mask}=\underbrace{1\cdots1}_{n}\underbrace{0\cdots0}_{32-n}$$

$$\text{total addresses}=2^{32-n},\qquad \text{usable hosts}=2^{32-n}-2\quad(n\le30)$$

The two reserved addresses are the **network address** (host bits all 0) and the **broadcast address** (host bits all 1). Special cases: `/31` (2 addresses, used for point-to-point links, RFC 3021) and `/32` (one single host).

**Key bitwise formulas** (with `&` = AND, `|` = OR, `~` = NOT):

$$\text{network}=\text{IP}\ \&\ \text{mask},\qquad \text{broadcast}=\text{network}\ |\ (\sim\text{mask})$$

**Common prefixes**

| CIDR | Mask | Total addresses | Usable hosts |
| --- | --- | --- | --- |
| /8 | 255.0.0.0 | 16,777,216 | 16,777,214 |
| /16 | 255.255.0.0 | 65,536 | 65,534 |
| /24 | 255.255.255.0 | 256 | 254 |
| /25 | 255.255.255.128 | 128 | 126 |
| /26 | 255.255.255.192 | 64 | 62 |
| /27 | 255.255.255.224 | 32 | 30 |
| /28 | 255.255.255.240 | 16 | 14 |
| /29 | 255.255.255.248 | 8 | 6 |
| /30 | 255.255.255.252 | 4 | 2 |
| /31 | 255.255.255.254 | 2 | 2 (point-to-point) |
| /32 | 255.255.255.255 | 1 | 1 |

> Cloud providers reserve extra addresses per subnet (AWS reserves 5), so a `/28` on AWS gives 11 usable, not 14.

### Worked example 1: `192.168.10.77/26`

- Host bits: $32-26=6$, block size $2^6=64$.
- The last octet: $77 = 01001101_2$, mask octet $192=11000000_2$.
- Network: $01001101\ \&\ 11000000 = 01000000 = 64$, so **network = 192.168.10.64**.
- Broadcast: $64+64-1=127$, so **broadcast = 192.168.10.127**.
- Usable range: **192.168.10.65 – 192.168.10.126** ($2^6-2=62$ hosts).

### Worked example 2: `10.1.77.200/21`

- Mask: `255.255.248.0`. Host bits: $11$. Usable hosts: $2^{11}-2=2046$.
- The boundary falls in the third octet with block size $256-248=8$. Since $77=9\cdot8+5$, the block starts at $72$.
- Network: **10.1.72.0**; broadcast: **10.1.79.255**; usable **10.1.72.1 – 10.1.79.254**.

### Splitting a network (subnetting)

Split `192.168.1.0/24` into four equal subnets. Each needs 2 more network bits (since $2^2=4$), so the new prefix is /26:

| Subnet | Range | Usable |
| --- | --- | --- |
| 192.168.1.0/26 | .0 – .63 | .1 – .62 |
| 192.168.1.64/26 | .64 – .127 | .65 – .126 |
| 192.168.1.128/26 | .128 – .191 | .129 – .190 |
| 192.168.1.192/26 | .192 – .255 | .193 – .254 |

Sizing rule: for $H$ required hosts you need $h$ host bits with $2^h-2\ge H$. For 100 hosts: $2^7-2=126\ge100$, so $h=7$ and the prefix is $/25$.

Two addresses are on the **same subnet** exactly when $(\text{A}\ \&\ \text{mask})=(\text{B}\ \&\ \text{mask})$.

### Doing it in code

**Python** (standard library):

```python
import ipaddress

net = ipaddress.ip_network("192.168.10.77/26", strict=False)
print(net)                       # 192.168.10.64/26
print(net.netmask)               # 255.255.255.192
print(net.broadcast_address)     # 192.168.10.127
print(net.num_addresses)         # 64
print(ipaddress.ip_address("192.168.10.100") in net)          # True
print(list(ipaddress.ip_network("192.168.1.0/24").subnets(new_prefix=26)))
```

**JavaScript** (bitwise operators work on 32-bit signed integers, so use `>>> 0` to get unsigned values):

```js
const ipToInt = (ip) => ip.split(".").reduce((acc, o) => acc * 256 + Number(o), 0);
const intToIp = (n) => [24, 16, 8, 0].map((s) => (n >>> s) & 255).join(".");

function cidrInfo(cidr) {
  const [ip, p] = cidr.split("/");
  const prefix = Number(p);
  // JS shift counts wrap modulo 32, so a shift by 32 would do nothing: handle /0 separately
  const mask = prefix === 0 ? 0 : (0xFFFFFFFF << (32 - prefix)) >>> 0;
  const network = (ipToInt(ip) & mask) >>> 0;
  const broadcast = (network | ~mask) >>> 0;
  return { network: intToIp(network), broadcast: intToIp(broadcast), total: 2 ** (32 - prefix) };
}
console.log(cidrInfo("192.168.10.77/26"));
// { network: '192.168.10.64', broadcast: '192.168.10.127', total: 64 }
```

**C++** (use `uint32_t`; shifting a 32-bit value by 32 is **undefined behavior**):

```cpp
#include <cstdint>
#include <iostream>
#include <sstream>
#include <string>

uint32_t ipToInt(const std::string& ip) {
    std::istringstream ss(ip);
    std::string part;
    uint32_t r = 0;
    while (std::getline(ss, part, '.'))
        r = (r << 8) | static_cast<uint32_t>(std::stoi(part));
    return r;
}

std::string intToIp(uint32_t n) {
    return std::to_string(n >> 24) + "." + std::to_string((n >> 16) & 0xFF) + "." +
           std::to_string((n >> 8) & 0xFF) + "." + std::to_string(n & 0xFF);
}

int main() {
    uint32_t ip = ipToInt("192.168.10.77");
    int prefix = 26;
    uint32_t mask = (prefix == 0) ? 0 : (0xFFFFFFFFu << (32 - prefix));
    uint32_t network = ip & mask;
    uint32_t broadcast = network | ~mask;
    uint64_t total = 1ull << (32 - prefix);              // 64-bit: 2^32 does not fit in uint32_t
    std::cout << intToIp(network) << " " << intToIp(broadcast) << " " << total << "\n";
    // 192.168.10.64 192.168.10.127 64
}
```

**C** (using the system parsing functions; addresses inside `in_addr` are in **network byte order**):

```c
#include <arpa/inet.h>
#include <stdint.h>
#include <stdio.h>

int main(void) {
    struct in_addr a;
    inet_pton(AF_INET, "192.168.10.77", &a);           // text -> binary (network byte order)
    uint32_t ip = ntohl(a.s_addr);                      // convert to host order for arithmetic

    int prefix = 26;
    uint32_t mask = prefix == 0 ? 0 : 0xFFFFFFFFu << (32 - prefix);
    struct in_addr net = { htonl(ip & mask) };

    char buf[INET_ADDRSTRLEN];
    inet_ntop(AF_INET, &net, buf, sizeof buf);          // binary -> text
    printf("%s/%d\n", buf, prefix);                     // 192.168.10.64/26
    return 0;
}
```

**C#** (.NET):

```csharp
using System.Net;

var ip = IPAddress.Parse("192.168.10.77");
byte[] b = ip.GetAddressBytes();                            // [192, 168, 10, 77], already big-endian
uint ipInt = ((uint)b[0] << 24) | ((uint)b[1] << 16) | ((uint)b[2] << 8) | b[3];

int prefix = 26;
uint mask = prefix == 0 ? 0u : uint.MaxValue << (32 - prefix);   // C# also masks shift counts to 5 bits
uint net = ipInt & mask;
var netIp = new IPAddress(new[] { (byte)(net >> 24), (byte)(net >> 16), (byte)(net >> 8), (byte)net });
Console.WriteLine($"{netIp}/{prefix}");                         // 192.168.10.64/26
```

### Why this matters in system design

- **Cloud VPC planning:** choose a VPC range (e.g. `10.0.0.0/16`), then carve subnets (`10.0.1.0/24` public, `10.0.2.0/24` private). **Overlapping CIDRs cannot be peered or connected by VPN**, so plan address space early.
- **Security groups and firewall rules** are written with CIDRs (`10.0.0.0/8`, `0.0.0.0/0` = "the entire internet").
- **Kubernetes and Docker** allocate pod/container IPs from CIDR pools (docs 29, 44).

Historical note: originally addresses were **classful** (Class A `/8`, B `/16`, C `/24`), which wasted addresses. CIDR (1993) replaced this with arbitrary prefix lengths.

---

# 4.2 Routing

## Router, Routing Table, Default Gateway

A **router** connects different networks and forwards packets between them using **layer 3** information (doc 03).

A **routing table** is a list of rules: *"for destinations matching this prefix, send the packet to this next hop through this interface."* Linux example (`ip route`):

```text
default via 192.168.1.1 dev eth0
10.8.0.0/24 via 192.168.1.50 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.20
```

- `192.168.1.0/24 dev eth0`: this network is **directly attached**; deliver locally.
- `10.8.0.0/24 via 192.168.1.50`: forward to that neighbor router.
- `default via 192.168.1.1` (equal to `0.0.0.0/0`): everything else goes to the **default gateway**, the router that leads out of the local network.

### How a host decides where to send a packet

```text
1. Compute: (destination IP & my mask) == (my network address) ?
2a. Yes → destination is on my subnet: find its MAC with ARP and send the frame directly.
2b. No  → send the frame to the default gateway's MAC (found with ARP);
          the IP destination stays the final destination.
```

### Longest Prefix Match

Several table entries can match one destination. The router uses the **most specific** match: the entry with the **longest prefix**.

| Destination | Matching entries | Chosen |
| --- | --- | --- |
| `10.1.2.9` | `0.0.0.0/0`, `10.0.0.0/8`, `10.1.0.0/16`, `10.1.2.0/24` | `10.1.2.0/24` |
| `10.1.9.9` | `0.0.0.0/0`, `10.0.0.0/8`, `10.1.0.0/16` | `10.1.0.0/16` |
| `8.8.8.8` | `0.0.0.0/0` | default route |

**Python:**

```python
import ipaddress

routes = [
    (ipaddress.ip_network("0.0.0.0/0"),   "192.168.1.1"),   # default
    (ipaddress.ip_network("10.0.0.0/8"),  "10.0.0.1"),
    (ipaddress.ip_network("10.1.0.0/16"), "10.1.0.1"),
    (ipaddress.ip_network("10.1.2.0/24"), "10.1.2.1"),
]

def lookup(dst: str) -> str:
    ip = ipaddress.ip_address(dst)
    matches = [(net, hop) for net, hop in routes if ip in net]
    return max(matches, key=lambda m: m[0].prefixlen)[1]

print(lookup("10.1.2.9"))   # 10.1.2.1
print(lookup("10.1.9.9"))   # 10.1.0.1
print(lookup("8.8.8.8"))    # 192.168.1.1
```

**C++:**

```cpp
#include <cstdint>
#include <string>
#include <vector>

struct Route { uint32_t network; int prefix; std::string nextHop; };

std::string lookup(const std::vector<Route>& table, uint32_t dst) {
    const Route* best = nullptr;
    for (const auto& r : table) {
        uint32_t mask = (r.prefix == 0) ? 0 : (0xFFFFFFFFu << (32 - r.prefix));
        if ((dst & mask) == r.network && (!best || r.prefix > best->prefix))
            best = &r;                                 // keep the longest matching prefix
    }
    return best ? best->nextHop : "unreachable";
}
```

This linear scan is $O(n)$. Real routers use a **binary trie** (lookup cost $O(32)$ bits for IPv4, independent of table size) or hardware **TCAM**, since the internet routing table holds roughly a million prefixes.

**Forwarding at each hop**

1. Receive frame, strip the layer-2 header.
2. Decrement **TTL**; if it becomes 0, drop and send an ICMP "Time Exceeded" message.
3. Longest-prefix-match the destination IP.
4. Build a new layer-2 frame for the next hop and send it.

## NAT (Network Address Translation)

NAT lets many devices with **private** addresses share one **public** address by rewriting addresses in packet headers as they cross a router.

The most common form is **PAT / NAPT** (also called "masquerade" or just "NAT"): the router rewrites the **source IP and source port** and remembers the mapping in a **NAT table**.

```text
 Private LAN                     NAT router                     Internet
 192.168.1.20:51000 ─────►  203.0.113.5:40001  ─────►  93.184.216.34:443
 192.168.1.21:51000 ─────►  203.0.113.5:40002  ─────►  93.184.216.34:443

 NAT table:
   (192.168.1.20, 51000) ⇄ (203.0.113.5, 40001)
   (192.168.1.21, 51000) ⇄ (203.0.113.5, 40002)

 Reply to 203.0.113.5:40001 → router looks up the table → forwards to 192.168.1.20:51000
```

Variants:

| Type | What is rewritten | Use |
| --- | --- | --- |
| **SNAT** (source NAT) | Source address of outgoing packets | Private hosts reaching the internet |
| **DNAT** (destination NAT) / port forwarding | Destination of incoming packets | Expose an internal server: `public:8080 → 192.168.1.20:80` |
| **CGNAT** | ISP-level NAT for many customers | IPv4 shortage (uses `100.64.0.0/10`) |

Consequences:

- Outgoing connections work automatically; **incoming connections** need an explicit port-forward, because the router has no table entry for them.
- NAT breaks the end-to-end principle: two hosts behind different NATs cannot connect directly without tricks (STUN, TURN, hole punching in P2P and WebRTC).
- Each public IP has a limited number of mappings (ports), which is another cause of **port exhaustion** for busy proxies.
- IPv6 has enough addresses to make NAT unnecessary (a firewall is still needed).

In practice: cloud **NAT gateways** let servers in private subnets make outbound calls (updates, third-party APIs) while remaining unreachable from the internet; Docker publishes container ports with DNAT rules and lets containers reach out with SNAT/masquerade (doc 29).

## Static vs Dynamic Routing

| | Static routing | Dynamic routing |
| --- | --- | --- |
| Configured by | Administrator manually | Routers exchange information via a routing protocol |
| Adapts to failures | No | Yes (recomputes routes) |
| Overhead | None | CPU, bandwidth, complexity |
| Best for | Small/simple networks, stub networks, default routes | Large or changing networks |

**Dynamic routing protocols**

| Protocol | Scope | Algorithm | Metric |
| --- | --- | --- | --- |
| **RIP** | Inside an organization (IGP) | Distance vector | Hop count (max 15) |
| **OSPF** | Inside an organization (IGP) | Link state (Dijkstra) | Link cost |
| **BGP** | Between organizations (**autonomous systems**) | Path vector, policy-based | AS path plus policies |

- **Distance vector** (Bellman-Ford): each router tells neighbors its distances; a router updates with

$$d_x(y)=\min_{v\in\text{neighbors}(x)}\big\{\,c(x,v)+d_v(y)\,\big\}$$

- **Link state**: each router floods information about its links, so all routers hold the same map, and each runs **Dijkstra's shortest-path algorithm** (doc 00, B.15/B.16) on it.
- **BGP** glues the internet together. Each network (AS) announces which prefixes it can reach; routes are chosen mainly by policy (business relationships), not just shortest path. BGP mistakes can take large services offline or misdirect traffic, and are a known cause of major outages.

**Useful commands**

```bash
ip addr                      # my interfaces and IPs
ip route                     # routing table
ip route get 8.8.8.8         # which route would be used for this destination
traceroute 8.8.8.8           # hops to a destination (uses increasing TTL)
```

---

# 4.3 DNS (Domain Name System)

Humans use names (`example.com`); routers need IP addresses. **DNS** is a distributed, hierarchical database that maps names to data (mostly IP addresses). It runs at the application layer, mostly over **UDP port 53**.

## Domain Names

A **domain name** is a sequence of **labels** separated by dots, read from right to left going from general to specific:

```text
      www . example . com .
       │       │       │   └─ root (the trailing dot, usually hidden)
       │       │       └───── TLD (top-level domain)
       │       └───────────── second-level domain (registered by you)
       └───────────────────── subdomain / host label
```

Limits: each label at most 63 characters; whole name at most 253 characters. Names are case-insensitive. A name including the trailing dot (`www.example.com.`) is a **fully qualified domain name (FQDN)**.

## The DNS Hierarchy and Servers

```text
                       .  (Root)
            ┌──────────┼──────────┐
           com        org        in ...        ← TLD servers
            │
       example.com                             ← Authoritative servers (owner's zone)
        ┌───┴────┐
   www.example  api.example                    ← records inside the zone
```

| Component | Role |
| --- | --- |
| **DNS resolver** (recursive resolver) | The server that does the lookup work on your behalf and caches results. Run by your ISP or public (Google `8.8.8.8`, Cloudflare `1.1.1.1`). Your device's small built-in client is the **stub resolver**. |
| **Root DNS servers** | Top of the hierarchy. They do not know `example.com`; they know which servers handle each TLD. There are 13 named root server addresses (`a.root-servers.net` to `m.root-servers.net`), served by hundreds of machines worldwide using **anycast**. |
| **TLD servers** | Handle `.com`, `.org`, `.in`, ... They know the authoritative name servers of every domain under that TLD. |
| **Authoritative DNS servers** | Hold the **actual records** for a domain and give the final answer. Managed by you or your provider (Route 53, Cloudflare...). |

## Basic DNS Flow

```text
 Browser wants www.example.com
    │
    ▼
 1. Browser cache → 2. OS cache (and /etc/hosts) → (miss)
    │
    ▼
 3. Recursive resolver (checks its own cache first)
    │
    ├─(a)─► Root server:       "www.example.com?"  →  "ask the .com TLD servers"
    ├─(b)─► .com TLD server:   "www.example.com?"  →  "ask ns1.example.com"
    └─(c)─► Authoritative NS:  "www.example.com?"  →  "203.0.113.10 (TTL 300)"
    │
    ▼
 4. Resolver caches the answer and returns 203.0.113.10 to the client
    │
    ▼
 5. Client opens a TCP connection to 203.0.113.10:443
```

- The client sends a **recursive** query ("give me the final answer").
- The resolver performs **iterative** queries to the servers above (each answers with the answer or a **referral** to the next servers).
- In practice the root and TLD name servers are already cached, so a typical uncached lookup needs only the **one** authoritative query.

**Expected lookup time** with cache hit ratio $h$:

$$E[T]=h\cdot t_{\text{cache}}+(1-h)\cdot t_{\text{resolve}}$$

Example: $t_{\text{cache}}=1$ ms, $t_{\text{resolve}}=80$ ms, $h=0.95$: $E[T]=0.95+4=4.95$ ms.

## DNS Records

| Type | Maps | Example |
| --- | --- | --- |
| **A** | Name → IPv4 address | `example.com. 300 IN A 203.0.113.10` |
| **AAAA** | Name → IPv6 address | `example.com. 300 IN AAAA 2001:db8::10` |
| **CNAME** | Name → another name (alias) | `www.example.com. IN CNAME example.com.` |
| **MX** | Domain → mail server, with priority (lower number = preferred) | `example.com. IN MX 10 mail.example.com.` |
| **TXT** | Arbitrary text: SPF/DKIM email policy, domain-ownership verification | `example.com. IN TXT "v=spf1 -all"` |
| **NS** | Domain → its authoritative name servers (delegation) | `example.com. IN NS ns1.example.com.` |

Also common: **SOA** (zone metadata and negative-cache TTL), **PTR** (reverse lookup IP → name), **SRV** (service host + port, used for service discovery), **CAA** (which CAs may issue certificates).

Zone file excerpt:

```text
example.com.       3600  IN  NS     ns1.example.com.
example.com.        300  IN  A      203.0.113.10
example.com.        300  IN  AAAA   2001:db8::10
www.example.com.    300  IN  CNAME  example.com.
example.com.       3600  IN  MX     10 mail.example.com.
example.com.       3600  IN  TXT    "v=spf1 -all"
```

Rules to remember:

- A **CNAME** cannot coexist with other record types at the same name, and so cannot be placed at the **zone apex** (`example.com`), which must hold NS and SOA. Providers offer ALIAS/ANAME or "CNAME flattening" for that case.
- A resolver following a CNAME does an extra lookup for the target name.

## DNS Caching and TTL

Every record carries a **TTL** (Time To Live, in seconds): how long resolvers and clients may cache it.

- Caching happens at every level: browser, OS, resolver. It cuts latency and load on authoritative servers.
- **Negative caching:** "this name does not exist" (NXDOMAIN) is also cached.
- A change to a record is not visible everywhere at once; stale answers may live up to the old TTL (and a few resolvers ignore or exceed TTLs). This is often called "propagation" but is really cache expiry.

| Low TTL (30–300 s) | High TTL (hours–a day) |
| --- | --- |
| Fast failover and changes | Fewer queries, faster lookups |
| More load on DNS servers, slightly slower first lookups | Slow to change; outages last longer |

Best practice for a migration: **lower the TTL well before the change**, switch, then raise it again.

## DNS Messages in Practice

- Transport: UDP port 53; falls back to **TCP** for large answers (the response sets a **TC** truncation flag) and zone transfers. Classic UDP answers were limited to 512 bytes; **EDNS0** raises that. Encrypted variants: **DoT** (TLS, port 853) and **DoH** (HTTPS, port 443).
- A DNS message starts with a 12-byte header (ID, flags, counts of questions/answers/authority/additional records).

A hand-built DNS query over UDP in Python (this also shows UDP in action, see doc 05):

```python
import random
import socket
import struct

def build_query(name: str) -> bytes:
    tid = random.randint(0, 0xFFFF)
    header = struct.pack("!HHHHHH", tid, 0x0100, 1, 0, 0, 0)  # flags 0x0100 = recursion desired; 1 question
    qname = b"".join(bytes([len(label)]) + label.encode() for label in name.split(".")) + b"\x00"
    question = qname + struct.pack("!HH", 1, 1)               # QTYPE=1 (A), QCLASS=1 (IN)
    return header + question

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.settimeout(3)
sock.sendto(build_query("example.com"), ("8.8.8.8", 53))
data, _ = sock.recvfrom(512)
tid, flags, qd, an, ns, ar = struct.unpack("!HHHHHH", data[:12])
print("answers:", an, "rcode:", flags & 0xF)                   # rcode 0 = success
```

(Reading the answer records fully requires handling DNS **name compression** pointers; use a library in real code.)

## Resolving Names from Code

The normal way is to let the OS resolver do it (it consults `/etc/hosts`, then DNS).

**C** (`getaddrinfo`, blocking):

```c
#include <arpa/inet.h>
#include <netdb.h>
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>

int main(void) {
    struct addrinfo hints, *res, *p;
    memset(&hints, 0, sizeof hints);
    hints.ai_family   = AF_UNSPEC;        // both IPv4 and IPv6
    hints.ai_socktype = SOCK_STREAM;

    int rc = getaddrinfo("example.com", "443", &hints, &res);
    if (rc != 0) { fprintf(stderr, "%s\n", gai_strerror(rc)); return 1; }

    for (p = res; p; p = p->ai_next) {
        char ip[INET6_ADDRSTRLEN];
        void *addr = (p->ai_family == AF_INET)
            ? (void *)&((struct sockaddr_in  *)p->ai_addr)->sin_addr
            : (void *)&((struct sockaddr_in6 *)p->ai_addr)->sin6_addr;
        inet_ntop(p->ai_family, addr, ip, sizeof ip);
        puts(ip);
    }
    freeaddrinfo(res);                    // always free the list
    return 0;
}
```

**C++:** same `getaddrinfo` API as C (wrap `res` in a `std::unique_ptr<addrinfo, decltype(&freeaddrinfo)>` for RAII), or use Boost.Asio's `ip::tcp::resolver`.

**Python:**

```python
import socket

for family, _, _, _, sockaddr in socket.getaddrinfo("example.com", 443, type=socket.SOCK_STREAM):
    print(family.name, sockaddr[0])

# Specific record types need a library:  pip install dnspython
# import dns.resolver
# for r in dns.resolver.resolve("example.com", "MX"): print(r.preference, r.exchange)
```

**Node.js:**

```js
const dns = require("dns").promises;

(async () => {
  console.log(await dns.resolve4("example.com"));                    // A records: a real DNS query
  console.log(await dns.lookup("example.com", { all: true }));       // OS getaddrinfo (respects /etc/hosts)
  console.log(await dns.resolveMx("example.com"));                   // MX records
})();
```

`dns.lookup` calls `getaddrinfo` in libuv's **thread pool** (doc 16), so many slow lookups can starve other thread-pool work; `dns.resolve*` uses the c-ares library and does not.

**C#:**

```csharp
using System.Net;

IPAddress[] addrs = await Dns.GetHostAddressesAsync("example.com");
foreach (var a in addrs) Console.WriteLine(a);
```

**Command line:**

```bash
dig example.com                # A record, with TTL and which server answered
dig MX example.com
dig @8.8.8.8 example.com       # ask a specific resolver
dig +trace example.com         # follow root → TLD → authoritative yourself
nslookup example.com
```

## DNS in System Design

- **DNS load balancing:** return several A records (or rotate them: *round robin*). Simple, but coarse: clients and resolvers cache answers, there is no real health awareness, and long-lived connections stay put.
- **GeoDNS / latency-based routing:** answer with the nearest region or data center (Route 53, Cloudflare).
- **Failover:** health checks remove dead endpoints from answers; the low TTL limits how long clients keep the dead address.
- **CDNs:** `static.example.com` is a CNAME to the CDN's hostname (doc 52).
- **Service discovery:** internal DNS names (`orders.internal`, Kubernetes `service.namespace.svc.cluster.local`) let services find each other (docs 32, 44).
- **Reliability:** DNS is a critical dependency; a DNS outage can make a healthy system unreachable. Use multiple providers or NS sets for critical domains, and cache aggressively in applications where safe.

---

# Key Takeaways

- IPv4 = 32 bits ($\approx4.3$ billion addresses); IPv6 = 128 bits. Private ranges (`10/8`, `172.16/12`, `192.168/16`) are reused everywhere and need NAT to reach the internet.
- CIDR `a.b.c.d/n`: $2^{32-n}$ addresses; network $=\text{IP}\ \&\ \text{mask}$, broadcast $=\text{network}\ |\ \sim\text{mask}$. Bind servers to `0.0.0.0` (not `127.0.0.1`) when they must be reachable from outside.
- Routers forward by **longest prefix match** on the destination IP; the **default gateway** (`0.0.0.0/0`) handles everything not otherwise matched. TTL prevents routing loops.
- NAT rewrites addresses/ports and keeps a mapping table; it enables IPv4 sharing but blocks unsolicited inbound traffic and can exhaust ports.
- Static routes are manual; dynamic protocols (RIP, OSPF inside, BGP between organizations) adapt to changes.
- DNS resolves names through **resolver → root → TLD → authoritative**, with caching at every level controlled by **TTL**. Choose TTLs deliberately; treat DNS as critical infrastructure.
- In shifts, in any language: use unsigned 32-bit types for IPv4 math and never shift by the full width (32).
