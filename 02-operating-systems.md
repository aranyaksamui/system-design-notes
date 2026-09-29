# 02 - Operating Systems

The OS sits between your program and the hardware. Every backend server is a process (or several) managed by an OS, so nearly every performance and reliability problem you will meet has an OS explanation.

```text
   Applications (Node.js, Nginx, PostgreSQL ...)
─────────────────────────────────────────────────
   System call interface   (read, write, fork ...)
─────────────────────────────────────────────────
   Kernel: process mgmt · memory mgmt · file systems
           device drivers · networking stack
─────────────────────────────────────────────────
   Hardware: CPU · RAM · disk · NIC
```

---

# 2.1 Processes

## Program vs Process

- **Program:** passive file of instructions on disk (e.g. `/usr/bin/node`).
- **Process:** an **active, running instance** of a program, with its own resources.

Running the same program twice creates two independent processes.

## What a Process Contains

```text
High addresses ┌────────────────────┐
               │       Stack        │  local variables, call frames (grows down)
               │         ↓          │
               │                    │
               │         ↑          │
               │        Heap        │  dynamic memory (grows up)
               ├────────────────────┤
               │   Data (globals)   │
               ├────────────────────┤
Low addresses  │   Code (text)      │  the instructions
               └────────────────────┘
```

Plus OS-side resources: open file descriptors, environment variables, a PID, user ID, current working directory.

## Process States

```text
   New ──► Ready ◄──────────────┐
             │  ▲               │ I/O or event completes
   scheduler │  │ preempted     │
   dispatch  ▼  │               │
           Running ───────► Waiting (Blocked)
             │          waits for I/O / lock
             ▼
         Terminated
```

| State | Meaning |
| --- | --- |
| **New** | Being created |
| **Ready** | Can run, waiting for a CPU |
| **Running** | Currently executing on a CPU |
| **Waiting/Blocked** | Waiting for an event (disk read, network data, lock) |
| **Terminated** | Finished; may still exist as a **zombie** until the parent collects its exit status |

## Process Control Block (PCB)

The kernel's data structure describing a process:

- Process ID (PID), parent PID
- State
- Program counter and saved CPU registers
- Scheduling info (priority, time used)
- Memory info (page table pointers)
- Open files / I/O status
- Accounting info

During a context switch the kernel saves the running process's state into its PCB and loads another PCB.

## Process Creation and Termination

On Unix-like systems a new process is created with **`fork()`**, which duplicates the calling process; the child usually then calls **`exec()`** to load a different program.

```c
#include <unistd.h>
#include <sys/wait.h>
#include <stdio.h>

int main() {
    pid_t pid = fork();          // creates a child process
    if (pid == 0) {
        // child
        execlp("ls", "ls", "-l", NULL);   // replace child with `ls`
    } else if (pid > 0) {
        // parent
        int status;
        waitpid(pid, &status, 0);         // wait for the child to finish
        printf("child finished\n");
    }
    return 0;
}
```

`fork()` returns: `0` in the child, the child's PID in the parent, `-1` on failure. Modern OSes use **copy-on-write**: parent and child share memory pages until one writes, then that page is copied.

**Termination** happens via `exit()`, returning from `main`, an unhandled error, or a signal (`SIGTERM` = polite request to stop, `SIGKILL` = forced kill that cannot be caught).

- **Zombie process:** finished, but the parent has not yet called `wait()`.
- **Orphan process:** parent died first; adopted by the init process (PID 1).

## Parent/Child Processes

Processes form a tree. Docker containers, process managers (PM2, systemd) and Node's `cluster` module all rely on this. Signals sent to a parent (e.g. `SIGTERM`) are not automatically forwarded to children, which matters in containers (doc 29).

**Inter-process communication (IPC):** processes are isolated, so they communicate via pipes, sockets, shared memory, message queues or files.

---

# 2.2 Threads

## Thread vs Process

A **thread** is a unit of execution *inside* a process. Threads of one process **share** code, heap, globals and open files, but each has its **own** stack, registers and program counter.

| | Process | Thread |
| --- | --- | --- |
| Memory | Separate address space | Shared address space |
| Creation cost | Heavy | Lightweight |
| Context switch | Expensive | Cheaper |
| Communication | IPC needed | Direct via shared memory |
| Isolation | Strong (crash contained) | Weak (one bad thread can crash all) |

## User Threads vs Kernel Threads

- **Kernel threads:** managed and scheduled by the OS; can run on different cores; a blocking call blocks only that thread.
- **User (green) threads:** managed by a runtime library in user space; very cheap, but the kernel sees only one thread, so blocking calls can stall all of them unless handled carefully.
- **Threading models:** many-to-one, one-to-one (Linux/Windows/Java), many-to-many (Go goroutines are multiplexed onto OS threads).

## Multithreading

Benefits: use multiple cores; overlap I/O waiting with computation. Risks: race conditions, deadlocks, harder debugging (see 2.4, 2.5).

## Thread Pools

Creating a thread per request is costly and unbounded. A **thread pool** keeps a fixed number of worker threads and a task queue.

```text
  tasks ──► [ queue ] ──► Worker 1
                     ├──► Worker 2
                     └──► Worker N
```

Pool sizing rules of thumb:

- CPU-bound tasks: about the number of cores $N_{\text{cores}}$ (or $N_{\text{cores}}+1$).
- I/O-bound tasks: larger, commonly estimated as

$$N_{\text{threads}} \approx N_{\text{cores}} \times \left(1 + \frac{W}{C}\right)$$

where $W$ is average wait time and $C$ is average compute time per task.

Example: 8 cores, each task waits 90 ms and computes 10 ms $\Rightarrow$ $8\times(1+9)=80$ threads.

## Context Switching (Threads)

Switching threads in the same process is cheaper than switching processes (no address-space change, so the TLB need not be flushed), but still costs time and cache warmth.

---

# 2.3 CPU Scheduling

The scheduler picks which **ready** process/thread runs next.

**Terms**

- **Arrival time:** when the process becomes ready.
- **Burst time:** CPU time needed.
- **Completion time (CT).**
- **Turnaround time:** $TAT = CT - \text{arrival}$.
- **Waiting time:** $WT = TAT - \text{burst}$.
- **Response time:** time from arrival until first getting the CPU.

**Preemptive vs non-preemptive**

- **Non-preemptive:** a process keeps the CPU until it finishes or blocks.
- **Preemptive:** the OS can take the CPU away (timer interrupt, higher-priority process arrives). All modern general-purpose OSes are preemptive.

## FCFS (First-Come, First-Served)

Run in arrival order. Simple, non-preemptive. Suffers from the **convoy effect**: short jobs wait behind a long one.

## SJF (Shortest Job First)

Run the process with the smallest burst time. **Provably minimizes average waiting time** among non-preemptive schedules. Problems: needs burst time prediction; long jobs can **starve**. Preemptive version: **SRTF** (Shortest Remaining Time First).

## Round Robin (RR)

Each process gets a fixed **time quantum** $q$, then goes to the back of the ready queue. Preemptive and fair.

- $q$ too large → behaves like FCFS.
- $q$ too small → too many context switches.

## Priority Scheduling

Each process has a priority; the highest runs first. Can be preemptive or not. **Starvation** of low-priority processes is fixed by **aging** (gradually increasing priority of waiting processes).

## Worked Example

Processes (all arrive at time 0): P1 burst 6, P2 burst 8, P3 burst 3.

**FCFS** order P1, P2, P3:

| Process | Completion | TAT | WT |
| --- | --- | --- | --- |
| P1 | 6 | 6 | 0 |
| P2 | 14 | 14 | 6 |
| P3 | 17 | 17 | 14 |

$$\text{Avg WT}=\frac{0+6+14}{3}=6.67$$

**SJF** order P3, P1, P2:

| Process | Completion | TAT | WT |
| --- | --- | --- | --- |
| P3 | 3 | 3 | 0 |
| P1 | 9 | 9 | 3 |
| P2 | 17 | 17 | 9 |

$$\text{Avg WT}=\frac{0+3+9}{3}=4$$

SJF is better on average waiting time, as expected.

Real OS schedulers (e.g. Linux CFS/EEVDF) use fair-share ideas with priorities and are more complex than these textbook algorithms.

---

# 2.4 Concurrency

## Race Condition

The result depends on the **timing/order** of concurrent operations on shared data.

```js
// Two threads run this at the same time on a shared counter
counter = counter + 1;
```

Under the hood this is three steps: **read → add → write**.

```text
counter = 5
Thread A: read 5
Thread B: read 5
Thread A: write 6
Thread B: write 6      ← expected 7, got 6 (lost update)
```

## Critical Section

A code region that accesses shared data and must not run in more than one thread at once. A correct solution needs:

1. **Mutual exclusion:** at most one thread inside.
2. **Progress:** if nobody is inside, a waiting thread can enter.
3. **Bounded waiting:** no thread waits forever.

## Mutex (Lock)

Only the thread that locked it can unlock it.

```c
pthread_mutex_lock(&m);
counter++;                 // critical section
pthread_mutex_unlock(&m);
```

## Semaphore

An integer counter with two atomic operations:

- `wait()` / `P()`: decrement; block if the value would go below 0.
- `signal()` / `V()`: increment; wake a waiter.

A **binary semaphore** ($0/1$) behaves like a lock; a **counting semaphore** limits access to $N$ identical resources (e.g. a pool of $N$ database connections).

Unlike a mutex, any thread can `signal()` - semaphores are also used for signalling between threads.

## Monitor

A higher-level construct that bundles shared data, the operations on it, and automatic mutual exclusion, plus **condition variables** (`wait`, `notify`) for waiting until a condition holds. Java's `synchronized` with `wait/notify` is a monitor.

## Locks (Kinds)

- **Spinlock:** busy-waits in a loop; good only for very short critical sections.
- **Blocking lock:** the waiting thread sleeps (context switch cost).
- **Reentrant lock:** the same thread may lock it repeatedly.

## Read/Write Locks

Many readers **or** one writer.

- Multiple threads can hold the **read** lock simultaneously.
- The **write** lock is exclusive.

Great when reads greatly outnumber writes. Watch for **writer starvation**.

## Atomic Operations

Indivisible operations performed by the CPU in a single step, e.g.:

- **Test-and-set**
- **Fetch-and-add**
- **Compare-and-swap (CAS):** "set to `new` only if the current value is `expected`".

```text
CAS(address, expected, new):
    if *address == expected:
        *address = new
        return true
    else:
        return false
```

CAS enables **lock-free** data structures and optimistic concurrency. The same idea reappears in databases (optimistic locking with version numbers) and Redis.

---

# 2.5 Deadlocks

A set of processes are **deadlocked** when each waits for a resource held by another in the set, so none can proceed.

```text
Thread 1: holds Lock A, wants Lock B
Thread 2: holds Lock B, wants Lock A
          → both wait forever
```

## Four Necessary Conditions (Coffman conditions)

**All four** must hold simultaneously:

1. **Mutual exclusion:** a resource can be held by only one process.
2. **Hold and wait:** a process holds resources while waiting for others.
3. **No preemption:** resources cannot be forcibly taken away.
4. **Circular wait:** a cycle of processes, each waiting for a resource held by the next.

## Handling Deadlocks

**Prevention** - break at least one condition:

| Break | How |
| --- | --- |
| Hold and wait | Request all resources at once, or release everything before requesting more |
| No preemption | Allow the OS to take resources back |
| Circular wait | Impose a **global ordering** of resources; always lock in that order |
| Mutual exclusion | Usually impossible for inherently non-shareable resources |

Lock ordering example (always lock the lower id first):

```js
function transfer(from, to, amount) {
  const [first, second] = from.id < to.id ? [from, to] : [to, from];
  first.lock();
  second.lock();
  // ... move money ...
  second.unlock();
  first.unlock();
}
```

**Avoidance** - the OS only grants a request if the system stays in a **safe state** (a safe order of completion exists). Classic algorithm: **Banker's algorithm**. Requires advance knowledge of maximum needs, so it is rarely used in practice.

**Detection and recovery** - allow deadlocks, detect them by finding cycles in a **wait-for graph**, then recover (kill a process, roll back, preempt). Databases do exactly this (doc 14).

**Ignoring** ("ostrich algorithm") - common in general-purpose OSes; combine with timeouts.

Related problems: **livelock** (threads keep reacting to each other without progress) and **starvation** (a thread never gets the resource).

---

# 2.6 Memory Management

## Stack and Heap

Covered in doc 00. In a process: the **stack** holds call frames (fast, automatic), the **heap** holds dynamically allocated memory.

## Virtual Memory

Each process gets its own virtual address space (doc 01). The OS maps virtual pages to physical frames.

## Paging

Memory is split into fixed-size **pages** (virtual) and **frames** (physical). Because all units are equal in size, there is **no external fragmentation**.

Translation:

$$\text{physical address} = \text{PageTable}[\text{page number}] \times \text{page size} + \text{offset}$$

**Page table size.** With 48-bit virtual addresses and 4 KB pages there are $2^{48}/2^{12}=2^{36}$ pages. A flat table would be enormous, so real systems use **multi-level page tables** (x86-64 uses 4 levels), allocating only parts that are used.

**Demand paging:** pages are loaded into RAM only when first accessed (via page faults).

**Page replacement** when RAM is full: FIFO, **LRU**, Clock (an LRU approximation used in practice), optimal (theoretical). A poor policy or too little RAM leads to **thrashing**.

## Segmentation

Divides memory into variable-size **logical segments** (code, data, stack, heap). An address is $(\text{segment}, \text{offset})$. Matches the program's logical structure but suffers from external fragmentation. Modern systems mainly use paging (sometimes with light segmentation support).

## Page Tables

Per-process structure mapping virtual page → physical frame, plus flags: **present**, **read/write**, **user/kernel**, **dirty** (modified), **accessed** (recently used).

## TLB (Translation Lookaside Buffer)

A small, very fast hardware cache of recent page table entries. A translation without the TLB requires several extra memory reads (one per page-table level).

**Effective Access Time (EAT)** with TLB hit ratio $h$, TLB lookup time $t_{\text{tlb}}$ and memory access time $t_{\text{mem}}$ (single-level page table):

$$\text{EAT} = h\,(t_{\text{tlb}}+t_{\text{mem}}) + (1-h)\,(t_{\text{tlb}}+2\,t_{\text{mem}})$$

Example: $t_{\text{tlb}}=1$ ns, $t_{\text{mem}}=100$ ns, $h=0.98$:

$$\text{EAT}=0.98(101)+0.02(201)=98.98+4.02=103\ \text{ns}$$

## Memory Allocation

Allocators (`malloc`, `new`, the JVM/V8 heap) hand out chunks of the heap.

Placement strategies: **first fit**, **best fit**, **worst fit**. Garbage-collected runtimes (V8, JVM) reclaim unreachable objects automatically; GC pauses can hurt tail latency.

## Fragmentation

- **Internal fragmentation:** wasted space *inside* an allocated block (e.g. a 4 KB page holding only 1 byte).
- **External fragmentation:** free memory exists in total but is split into pieces too small to use.

Paging eliminates external fragmentation but has internal fragmentation (on average about half a page per allocation).

---

# 2.7 I/O

## Blocking vs Non-blocking

- **Blocking I/O:** the call does not return until the operation is done; the thread sleeps.
- **Non-blocking I/O:** the call returns immediately; if not ready it reports "would block" (e.g. `EAGAIN`), and the program tries later.

## Synchronous vs Asynchronous

- **Synchronous:** the caller waits for the result (blocking, or repeatedly polling).
- **Asynchronous:** the caller starts the operation and is **notified** when it completes (callback, event, completion queue, e.g. Linux `io_uring`, Windows IOCP).

Note: *non-blocking* means "don't wait inside the call"; *asynchronous* means "completion is signalled later". Non-blocking + readiness notification is how most servers work.

## I/O Multiplexing

One thread monitors **many** file descriptors and learns which are ready:

| Mechanism | Notes |
| --- | --- |
| `select` | Old, limited to ~1024 descriptors, $O(n)$ scan |
| `poll` | No fixed limit, still $O(n)$ |
| `epoll` (Linux) | Scales to very large numbers of connections; cost proportional to *ready* descriptors |
| `kqueue` (BSD/macOS) | Similar to epoll |

```text
Thread ──► epoll_wait([fd1, fd2, ... fd10000])
                   │ returns only the ready ones
                   ▼
           handle fd7, fd2 ... then wait again
```

This is the foundation of Nginx and Node.js (libuv), letting one thread serve thousands of connections (the "C10K problem").

## File Descriptors

A **file descriptor (FD)** is a small integer the kernel gives a process to refer to an open file, socket, pipe or device.

- `0` = stdin, `1` = stdout, `2` = stderr.
- Each process has a limit on open FDs (`ulimit -n`). A busy server holds one FD per connection, so "**too many open files**" is a classic production error.

---

# 2.8 System Calls

## User Mode vs Kernel Mode

The CPU runs in two privilege levels:

- **User mode:** restricted; cannot touch hardware or other processes' memory directly.
- **Kernel mode:** full privileges.

Protection exists so a buggy application cannot crash the whole machine.

## System Calls

A **system call** is how a program asks the kernel to do something privileged. The CPU switches to kernel mode, runs the kernel routine, then returns to user mode. This transition has a cost (hundreds of nanoseconds to microseconds), so batching operations (buffered I/O) helps.

```text
User program ── read(fd, buf, n) ──► [trap into kernel] ──► kernel reads disk/network
             ◄──── returns bytes ────  [return to user mode]
```

| Call | Purpose |
| --- | --- |
| `open(path, flags)` | Open a file, returns an FD |
| `read(fd, buf, n)` | Read up to $n$ bytes |
| `write(fd, buf, n)` | Write up to $n$ bytes |
| `close(fd)` | Release the descriptor |
| `fork()` | Create a child process |
| `exec()` | Replace the current program image |

Example:

```c
#include <fcntl.h>
#include <unistd.h>

int main() {
    int fd = open("data.txt", O_RDONLY);
    char buf[100];
    ssize_t n = read(fd, buf, sizeof(buf));   // may return fewer than 100 bytes
    write(1, buf, n);                          // write to stdout (FD 1)
    close(fd);
    return 0;
}
```

Sockets use related calls: `socket`, `bind`, `listen`, `accept`, `connect`, `send`/`recv` (see doc 03/05).

Tools: `strace` (Linux) shows a process's system calls - very useful for debugging.

---

# 2.9 OS-Level Performance

## CPU Utilization

Percent of time the CPU is busy. Split into **user**, **system**, **iowait** (idle while waiting for I/O), **idle**. High `iowait` suggests a disk/network bottleneck rather than a CPU one.

## Memory Utilization

Track used, free, **cached/buffers** (file cache; the OS reclaims it when needed, so "low free memory" is often fine), **swap usage** (heavy swapping = trouble), and per-process **resident set size (RSS)**. Watch for leaks (steady growth) and the OOM killer terminating processes.

## Disk I/O

Metrics: **IOPS** (operations/s), throughput (MB/s), latency (ms), queue depth, utilization. Relationship:

$$\text{throughput} = \text{IOPS}\times\text{I/O size}$$

Example: 10,000 IOPS with 4 KB I/O $\Rightarrow$ $10{,}000\times4\ \text{KB}=40\ \text{MB/s}$.

## Network I/O

Bandwidth (bits/s), packets/s, error/drop counts, connection counts. Note: bandwidth is usually quoted in **bits** per second; divide by 8 for bytes per second (1 Gbps $\approx$ 125 MB/s).

## Load Average

On Linux, the average number of processes that are **running or waiting to run** (also uninterruptible I/O wait) over 1, 5 and 15 minutes.

Rule of thumb: compare to the core count $N_{\text{cores}}$.

- Load $\approx N_{\text{cores}}$: fully used.
- Load $\gg N_{\text{cores}}$: work is queuing.

Example: 4 cores and a load average of 12 means about 3 tasks per core competing.

## Throughput and Latency

- **Throughput:** work completed per unit time (requests/s).
- **Latency:** time per request.

Latency is best described with **percentiles** ($p50$, $p95$, $p99$), not averages, because a few slow requests hide inside the mean.

**Utilization and queueing.** As utilization $\rho$ approaches 1, waiting time grows sharply. For a simple M/M/1 queue with service rate $\mu$ and arrival rate $\lambda$ ($\rho=\lambda/\mu<1$):

$$W = \frac{1}{\mu-\lambda}$$

Example: $\mu=100$ req/s. At $\lambda=50$: $W=20$ ms. At $\lambda=90$: $W=100$ ms. At $\lambda=99$: $W=1000$ ms. So keep production systems well below 100% utilization.

**Useful Linux tools:** `top`/`htop`, `vmstat`, `iostat`, `free -m`, `ss`/`netstat`, `strace`, `lsof`.

---

# Key Takeaways

- A **process** has its own memory; **threads** share memory inside a process and are cheaper but riskier.
- Race conditions come from unsynchronized access to shared data; fix with mutexes, semaphores, atomics.
- Deadlock needs all four Coffman conditions; break one (usually circular wait via lock ordering).
- Paging + page tables + TLB implement virtual memory; page faults to disk are costly.
- Servers scale by using **non-blocking I/O + multiplexing** (`epoll`) instead of one thread per connection.
- User→kernel transitions (system calls) have a cost; file descriptors are a limited resource.
- Watch utilization, load average and percentile latency; queueing delay explodes as utilization nears 100%.
