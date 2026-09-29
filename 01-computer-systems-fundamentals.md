# 01 — Computer & Systems Fundamentals

Every server is a computer. To reason about server performance you must know what the CPU, memory and storage actually do, and how slow each one is compared to the others.

---

# Part A — How a Computer Works

## A.1 The Big Picture

```text
            ┌─────────────────────────────┐
            │            CPU              │
            │  ┌─────┐ ┌───────────────┐  │
            │  │ ALU │ │ Control Unit  │  │
            │  └─────┘ └───────────────┘  │
            │      Registers   L1/L2/L3   │
            └──────────────┬──────────────┘
                           │  Bus (address, data, control)
        ┌──────────────────┼──────────────────┐
        │                  │                  │
      RAM             Storage (SSD/HDD)   I/O devices
                                          (NIC, keyboard, GPU...)
```

## A.2 Components

| Component | Role |
| --- | --- |
| **CPU** | Executes instructions. Contains ALU, control unit, registers, caches. |
| **ALU** (Arithmetic Logic Unit) | Performs arithmetic (`+ - × ÷`) and logic (`AND OR NOT`, comparisons). |
| **Control Unit** | Fetches and decodes instructions and directs other parts. |
| **Registers** | Tiny, fastest storage inside the CPU (e.g. 16–32 of them, 64 bits each). Hold operands, results, the program counter. |
| **Cache** | Small, fast memory between CPU and RAM (L1/L2/L3). Holds recently used data. |
| **RAM** | Main memory. Volatile (contents lost without power). Holds running programs and their data. |
| **Storage** | Persistent memory (SSD, HDD). Keeps data without power. Much slower than RAM. |
| **Motherboard** | Circuit board connecting all components. |
| **Bus** | Wires carrying signals between components: **address bus** (where), **data bus** (what), **control bus** (read/write, timing). |
| **I/O devices** | Network card (NIC), disk controller, keyboard, display, GPU. |

Important registers:

- **Program Counter (PC):** address of the next instruction.
- **Instruction Register (IR):** the instruction currently being executed.
- **Stack Pointer (SP):** top of the current stack.
- **General-purpose registers:** temporary values.

## A.3 Binary Representation

Computers store everything as bits (0 or 1).

- $n$ bits can represent $2^n$ different values.
- A **byte** = 8 bits = 256 values.

**Binary to decimal.** The bit string $b_{n-1}\dots b_1 b_0$ has value:

$$\sum_{i=0}^{n-1} b_i \cdot 2^i$$

Example: $1011_2 = 1\cdot 8 + 0\cdot 4 + 1\cdot 2 + 1\cdot 1 = 11$.

**Hexadecimal** (base 16) is a compact way to write binary: one hex digit = 4 bits. `0xFF` = `1111 1111` = 255.

**Negative integers (two's complement).** To negate: invert all bits and add 1. For 8 bits, $-1$ is `1111 1111`. The most significant bit acts as a sign bit.

**Characters.** Text is stored as numbers via an encoding. ASCII maps `'A'` → 65. **UTF-8** encodes every Unicode character using 1–4 bytes and is backward compatible with ASCII.

**Floating point (IEEE 754).** A 64-bit `double` stores a sign bit, 11 exponent bits and 52 fraction bits:

$$\text{value} = (-1)^{s} \times 1.f \times 2^{\,e - 1023}$$

**Endianness.** Byte order of multi-byte numbers. **Little-endian** stores the least significant byte first (x86, most ARM). **Big-endian** stores the most significant byte first (this is also "network byte order" used in network protocols — see doc 03/05).

**Data size units** (used constantly in estimation):

| Unit | Value (binary) | Approx (decimal) |
| --- | --- | --- |
| 1 KB | $2^{10}$ bytes | $10^3$ |
| 1 MB | $2^{20}$ bytes | $10^6$ |
| 1 GB | $2^{30}$ bytes | $10^9$ |
| 1 TB | $2^{40}$ bytes | $10^{12}$ |
| 1 PB | $2^{50}$ bytes | $10^{15}$ |

## A.4 Machine Instructions

The CPU only understands **machine instructions**: binary patterns such as "add register A and B, store in C". Humans write assembly or high-level languages; compilers/interpreters translate them.

Each instruction has:

- an **opcode** (what operation), and
- **operands** (which registers/memory addresses/constants).

Example (illustrative assembly):

```asm
MOV  R1, 5      ; put 5 in register R1
MOV  R2, 7      ; put 7 in register R2
ADD  R3, R1, R2 ; R3 = R1 + R2  -> 12
STORE R3, [0x1000] ; write R3 to memory address 0x1000
```

The set of instructions a CPU understands is its **ISA** (Instruction Set Architecture), e.g. x86-64, ARM.

## A.5 Instruction Execution: Fetch → Decode → Execute

The CPU repeats this cycle billions of times per second:

```text
        ┌───────────────────────────────────────┐
        ▼                                       │
   1. FETCH  ── read instruction at address PC  │
        ▼                                       │
   2. DECODE ── work out opcode and operands    │
        ▼                                       │
   3. EXECUTE ─ ALU computes / memory accessed  │
        ▼                                       │
   4. WRITE-BACK ─ store result; PC advances ───┘
```

**Clock speed** is the number of cycles per second. A 3 GHz CPU does $3\times10^9$ cycles/second, so one cycle lasts:

$$\frac{1}{3\times10^{9}}\ \text{s} \approx 0.33\ \text{ns}$$

Modern CPUs **pipeline** instructions (overlap the stages of different instructions), run several instructions per cycle (**superscalar**), and predict branches to avoid stalls. You do not need the details; you need to know the CPU is extremely fast compared to everything else.

---

# Part B — CPU & Memory

## B.1 CPU Concepts

**Program vs process.** A program is a file of instructions on disk. A **process** is a running instance of it, with its own memory (details in doc 02).

**CPU cores.** Each core is an independent execution unit. A CPU with 8 cores can run 8 instruction streams truly in parallel.

**Threads.** A thread is a sequence of instructions scheduled onto a core. One process can have many threads sharing memory. With **hyper-threading / SMT**, one physical core presents 2 logical threads to the OS to keep its units busy.

**Concurrency vs parallelism**

- *Concurrency:* multiple tasks make progress in overlapping time (may be interleaved on one core).
- *Parallelism:* multiple tasks run at the same instant (needs multiple cores).

**Context switching.** When the OS stops one thread and runs another on the same core, it must:

1. Save the current thread's registers, program counter, stack pointer.
2. Load the next thread's saved state.
3. Possibly flush/invalidate caches and TLB entries (extra cost).

A context switch costs roughly microseconds (and causes cache misses afterwards). Too many threads → the CPU spends more time switching than working.

**CPU scheduling basics.** The OS **scheduler** decides which ready thread runs next and for how long (a **time slice**, typically a few milliseconds). Details: doc 02.

**CPU-bound vs I/O-bound workloads**

| | CPU-bound | I/O-bound |
| --- | --- | --- |
| Limited by | CPU speed | Waiting for disk/network/DB |
| Examples | Video encoding, hashing, sorting, ML | Web APIs calling databases, file serving |
| Threads waiting | Rarely | Mostly waiting |
| Scaling helps | More cores | More concurrency (async/non-blocking) |

Most backend web services are **I/O-bound**. This is the whole reason Node.js's event loop works well (doc 16).

## B.2 The Memory Hierarchy

Faster memory is smaller and more expensive; slower memory is larger and cheaper.

```text
        Smaller / Faster / Costlier
              ▲
              │   Registers        ~ <1 ns        bytes–KB
              │   L1 cache         ~ 1 ns         32–64 KB per core
              │   L2 cache         ~ 3–4 ns       256 KB – few MB per core
              │   L3 cache         ~ 10–20 ns     several–tens of MB, shared
              │   RAM              ~ 100 ns       GBs
              │   SSD              ~ 100 µs       100s of GB – TBs
              │   HDD              ~ 5–10 ms      TBs
              │   Network storage  ~ ms–100s of ms
              ▼
        Larger / Slower / Cheaper
```

### Latency numbers every engineer should know (approximate, order of magnitude)

| Operation | Time |
| --- | --- |
| L1 cache reference | 1 ns |
| Branch mispredict | ~3–5 ns |
| L2 cache reference | ~4 ns |
| Mutex lock/unlock | ~20 ns |
| Main memory reference | ~100 ns |
| Read 1 MB sequentially from RAM | ~10–50 µs |
| SSD random read | ~100 µs |
| Read 1 MB sequentially from SSD | ~200 µs – 1 ms |
| Network round trip within a data center | ~0.5 ms |
| Read 1 MB sequentially from HDD | ~2–20 ms |
| HDD seek | ~5–10 ms |
| Network round trip across continents | ~100–200 ms |

Unit conversions:

$$1\ \text{s} = 10^3\ \text{ms} = 10^6\ \mu\text{s} = 10^9\ \text{ns}$$

Key ratio: RAM is roughly $100\times$ slower than L1; SSD is roughly $1000\times$ slower than RAM; HDD seek is roughly $10^5\times$ slower than RAM. Values vary by hardware but the *orders of magnitude* hold. This is why caches, indexes and in-memory stores exist.

## B.3 Registers, L1/L2/L3 Cache, RAM

- **Registers:** accessed in the same cycle. The compiler decides what lives in registers.
- **L1:** per core, split into instruction and data caches; smallest and fastest.
- **L2:** per core (or per core pair), larger, slower.
- **L3:** shared by all cores; also helps cores exchange data.
- **RAM (DRAM):** main memory; must be refreshed constantly; volatile.

Data moves between levels in fixed-size blocks called **cache lines** (typically **64 bytes**). Reading one byte loads the whole 64-byte line.

## B.4 Cache Hit / Miss

- **Cache hit:** requested data is in the cache → fast.
- **Cache miss:** data is not in the cache → must be fetched from the next slower level (miss penalty).

$$\text{Hit ratio} = \frac{\text{hits}}{\text{hits}+\text{misses}}$$

**Average Memory Access Time (AMAT):**

$$\text{AMAT} = t_{\text{hit}} + (1 - h)\times t_{\text{miss penalty}}$$

where $h$ is the hit ratio.

Worked example: cache access $= 1\ \text{ns}$, RAM access $= 100\ \text{ns}$, hit ratio $h = 0.95$.

$$\text{AMAT} = 1 + 0.05 \times 100 = 6\ \text{ns}$$

If $h$ drops to $0.90$: $\text{AMAT} = 1 + 0.10\times100 = 11\ \text{ns}$ — almost double. Tiny changes in hit ratio have a big effect on performance. **The exact same formula applies to application caches like Redis** (doc 23).

Types of misses: **compulsory** (first access), **capacity** (cache too small), **conflict** (mapping collisions).

**Replacement policies** decide what to evict: LRU (least recently used), FIFO, random, LFU.

**Write policies** for caches: *write-through* (write to cache and memory immediately) vs *write-back* (write to cache, memory later). You will see these again in doc 23.

## B.5 Locality of Reference

Caches work because programs are predictable:

- **Temporal locality:** recently used data will likely be used again soon (loop variables).
- **Spatial locality:** data near recently used data will likely be used soon (array elements).

```js
// Good spatial locality: walks memory sequentially (row-major)
for (let i = 0; i < n; i++)
  for (let j = 0; j < n; j++)
    sum += matrix[i][j];

// Poor spatial locality: jumps across rows (column-wise), many cache misses
for (let j = 0; j < n; j++)
  for (let i = 0; i < n; i++)
    sum += matrix[i][j];
```

Both are $O(n^2)$, but the first can be several times faster on real hardware. Arrays beat linked lists in practice for this reason. The same principle drives database design: sequential disk reads are far faster than random reads.

## B.6 Virtual Memory

Each process believes it has its own large, private, continuous address space. The OS and hardware (the **MMU**) map these **virtual addresses** to **physical addresses** in RAM.

Benefits:

- **Isolation:** one process cannot read another's memory.
- **Simplicity:** each program can use the same address layout.
- **Larger-than-RAM:** unused pages can be moved to disk (**swap**).
- **Sharing:** the same physical page can be mapped into multiple processes (shared libraries).

## B.7 Pages, Memory Addresses, Page Faults

Memory is divided into fixed-size blocks called **pages** (commonly **4 KB**). Physical memory is divided into equally sized **frames**.

A virtual address splits into:

$$\text{virtual address} = (\text{page number},\ \text{offset})$$

For page size $2^{k}$ bytes, the lowest $k$ bits are the **offset** and the remaining upper bits are the **page number**. For 4 KB pages, $k = 12$.

The **page table** maps page number → frame number. The physical address is:

$$\text{physical address} = \text{frame number}\times \text{page size} + \text{offset}$$

Worked example (page size 4096 bytes): virtual address $10000$.

$$\text{page number}=\left\lfloor \frac{10000}{4096}\right\rfloor = 2,\qquad \text{offset}=10000 \bmod 4096 = 1808$$

If page 2 maps to frame 7: physical address $= 7\times4096+1808 = 30480$.

**Page fault.** Happens when a program accesses a page that is not currently mapped to RAM. The OS:

1. Traps the access.
2. Finds the page (on disk/swap or a fresh zero page).
3. Loads it into a free frame (evicting another page if needed).
4. Updates the page table and resumes the instruction.

A page fault that reads from disk costs milliseconds (HDD) or ~100 µs (SSD) — a huge slowdown. If a system constantly swaps pages in and out, it is **thrashing**.

**TLB** (Translation Lookaside Buffer): a small cache of recent page-table entries so most address translations avoid extra memory reads (see doc 02).

## B.8 Performance Metrics from This Section

**CPU utilization**

$$\text{CPU utilization} = \frac{\text{time CPU spent doing work}}{\text{total time}} \times 100\%$$

Consistently near 100% means no headroom. Also distinguish user time, system (kernel) time, I/O wait, and idle.

**Memory latency:** time from issuing a memory request to receiving data (~100 ns for RAM).

**I/O latency:** time to complete a disk or network operation (µs to ms).

**Throughput vs latency**

- **Latency:** time for one operation.
- **Throughput:** number of operations per unit time.

They are related but not identical. Pipelining and parallelism raise throughput without necessarily lowering latency. **Little's Law** (used throughout system design):

$$L = \lambda \times W$$

where $L$ = average number of items in the system, $\lambda$ = arrival rate, $W$ = average time an item spends in the system.

Example: 200 requests/s arrive and each takes 0.05 s on average $\Rightarrow$ $L = 200\times0.05 = 10$ requests in flight on average.

---

# Why This Matters for System Design

| Fact | Consequence |
| --- | --- |
| RAM is ~1000× faster than SSD | Keep hot data in memory (caching, Redis) |
| Sequential access ≫ random access | Append-only logs (Kafka, LSM trees, WAL) are fast |
| Context switches are costly | Avoid thousands of OS threads; use event loops/async |
| Most services are I/O-bound | Concurrency model matters more than CPU speed |
| Cache hit ratio drives performance | Design for high hit ratios |
| Page faults/swapping are disastrous | Size memory correctly; disable swap on latency-sensitive servers |

---

# Key Takeaways

- A CPU repeatedly does **fetch → decode → execute**; it is vastly faster than memory, disk and network.
- The memory hierarchy trades size for speed; caches work thanks to **temporal and spatial locality**.
- $\text{AMAT} = t_{\text{hit}} + (1-h)\,t_{\text{miss}}$: small hit-ratio changes matter a lot.
- Virtual memory gives isolation and flexibility via pages and page tables; page faults to disk are very expensive.
- CPU-bound work needs more cores; I/O-bound work needs better concurrency.
- Know the latency orders of magnitude (ns → µs → ms) — they justify almost every design decision later.
