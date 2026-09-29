# 00 - Prerequisites: Programming & Data Structures/Algorithms

System design decides *which* data structure, *which* algorithm and *how much* it costs at scale. Without these basics, every later topic (indexes, caches, queues, sharding) will feel like magic.

> **Code convention for this series:** examples are shown in **JavaScript** (Node.js) and **C++17** side by side. C++ is included because it exposes what JavaScript hides: memory, pointers, integer sizes and object lifetimes. Compile C++ examples with:
>
> ```bash
> g++ -std=c++17 -Wall -Wextra -O2 file.cpp -o out && ./out
> ```

**JavaScript vs C++ at a glance**

| Aspect | JavaScript | C++ |
| --- | --- | --- |
| Typing | Dynamic | Static (types fixed at compile time) |
| Execution | Interpreted, then JIT-compiled by V8 | Compiled ahead of time to native machine code |
| Numbers | One `Number` type (64-bit float) plus `BigInt` | Many types: `int`, `long long`, `double`, `uint8_t`, ... |
| Memory | Garbage collected | No GC; you control lifetimes (RAII, smart pointers) |
| Strings | Immutable | `std::string` is mutable |
| Out-of-bounds array access | Returns `undefined` | `[]` does **not** check: undefined behavior |
| Common failure | Exceptions / `undefined` | Crashes, corrupted memory, undefined behavior |

**Undefined behavior (UB)** means the C++ standard places no requirements on what happens (crash, wrong result, or seemingly fine). Examples: signed integer overflow, out-of-bounds access, dereferencing `nullptr`, using freed memory. The compiler may assume UB never happens and optimize accordingly, so UB bugs can be very strange.

---

# Part A - Programming Fundamentals

## A.1 Variables, Data Types, Operators

A **variable** is a named location in memory holding a value.

| Category | Examples | Notes |
| --- | --- | --- |
| Integer | `int`, `long` | Fixed size (e.g. 32 or 64 bits) → can **overflow** |
| Floating point | `float`, `double` | Approximate values (see below) |
| Boolean | `true`, `false` | 1 logical bit (usually stored in 1 byte) |
| Character / String | `'a'`, `"hello"` | String = sequence of characters |
| Composite | arrays, objects, structs | Group several values |

**Integer range.** An unsigned $n$-bit integer holds $0$ to $2^n - 1$. A signed (two's complement) $n$-bit integer holds:

$$-2^{n-1} \;\text{to}\; 2^{n-1}-1$$

For $n = 32$: $-2{,}147{,}483{,}648$ to $2{,}147{,}483{,}647$. Going past the maximum is **integer overflow**: in Java it wraps around; in C++ it wraps for *unsigned* types but is **undefined behavior** for *signed* types. JavaScript numbers are 64-bit floats, so they do not overflow this way; they lose exactness beyond $2^{53}-1$ (`Number.MAX_SAFE_INTEGER`).

**C++ (fixed-width integers):**

```cpp
#include <cstdint>
#include <iostream>
#include <limits>

int main() {
    std::cout << sizeof(int) << "\n";                          // usually 4 bytes (not guaranteed by the standard)
    std::cout << std::numeric_limits<int32_t>::max() << "\n";  // 2147483647

    uint8_t x = 255;
    x = x + 1;                 // unsigned wraps around: x becomes 0 (well-defined)

    // int32_t y = INT32_MAX;
    // y = y + 1;              // signed overflow: UNDEFINED BEHAVIOR
}
```

| C++ type | Size | Range |
| --- | --- | --- |
| `int8_t` / `uint8_t` | 1 byte | $-128..127$ / $0..255$ |
| `int16_t` / `uint16_t` | 2 bytes | $-32{,}768..32{,}767$ / $0..65{,}535$ |
| `int32_t` / `uint32_t` | 4 bytes | about $\pm 2.1\times10^9$ / $0..4.29\times10^9$ |
| `int64_t` / `uint64_t` | 8 bytes | about $\pm 9.2\times10^{18}$ / $0..1.8\times10^{19}$ |

Prefer these fixed-width types (from `<cstdint>`) whenever the exact size matters: network protocols, file formats, hashes, IDs.

**Floating-point is approximate.** Decimals like $0.1$ cannot be represented exactly in binary:

```js
console.log(0.1 + 0.2);           // 0.30000000000000004
console.log(0.1 + 0.2 === 0.3);   // false
```

**C++:**

```cpp
#include <cmath>
#include <iostream>

int main() {
    double a = 0.1 + 0.2;
    std::cout << (a == 0.3) << "\n";                    // 0 (false)
    std::cout << (std::fabs(a - 0.3) < 1e-9) << "\n";   // 1: compare floats with a tolerance
}
```

> Rule: never store money in floating point. Use integers (e.g. paise/cents) or a decimal type.

**Operators**

| Type | Operators |
| --- | --- |
| Arithmetic | `+ - * / %` |
| Comparison | `== != < > <= >=` |
| Logical | `&& \|\| !` |
| Bitwise | `& \| ^ ~ << >>` |
| Assignment | `= += -= *=` |

`%` (modulo) is heavily used in system design (hash → bucket: `hash % N`).

**C++ gotchas with operators:**

```cpp
#include <iostream>

int main() {
    int a = 7, b = 2;
    std::cout << a / b << "\n";        // 3   integer division truncates toward zero (JavaScript gives 3.5)
    std::cout << 7 / 2.0 << "\n";      // 3.5 one operand is double, so real division
    std::cout << a % b << "\n";        // 1
    std::cout << -7 % 3 << "\n";       // -1  result takes the sign of the dividend (same as JavaScript)
    std::cout << (a & b) << "\n";      // 2   bitwise AND  (0111 & 0010)
    std::cout << (a | b) << "\n";      // 7   bitwise OR
    std::cout << (a ^ b) << "\n";      // 5   bitwise XOR
    std::cout << (1 << 4) << "\n";     // 16  left shift = multiply by 2^4
}
```

---

## A.2 Control Flow

Control flow decides which instructions run.

```js
// Conditional
if (age >= 18) {
  console.log("adult");
} else {
  console.log("minor");
}

// Loops
for (let i = 0; i < 5; i++) { /* runs 5 times */ }
while (queue.length > 0) { /* runs until empty */ }
```

- `break` leaves a loop; `continue` skips to the next iteration.
- `switch` chooses among many constant cases.

**C++:**

```cpp
#include <iostream>
#include <queue>
#include <vector>

int main() {
    int age = 20;
    if (age >= 18) { std::cout << "adult\n"; }
    else           { std::cout << "minor\n"; }

    for (int i = 0; i < 5; i++) { /* runs 5 times */ }

    std::queue<int> q;
    while (!q.empty()) { /* runs until empty */ }

    std::vector<int> values = {1, 2, 3};
    for (int x : values) { /* range-based for over a container */ }

    int day = 2;
    switch (day) {
        case 1: /* ... */ break;   // without break, execution "falls through" to the next case
        case 2: /* ... */ break;
        default: break;
    }
}
```

---

## A.3 Functions

A **function** packages reusable logic: it takes **parameters**, may **return** a value.

```js
function add(a, b) {
  return a + b;
}
```

**C++:**

```cpp
#include <iostream>

int add(int a, int b);            // declaration (prototype): tells the compiler the signature

int main() {
    std::cout << add(2, 3) << "\n";                // 5
    auto twice = [](int x) { return x * 2; };      // lambda: an anonymous function
    std::cout << twice(4) << "\n";                 // 8
}

int add(int a, int b) {           // definition
    return a + b;
}
```

C++ specifics: a function must be declared before it is used; several functions can share a name with different parameter types (**overloading**); parameters can have **default values** (`int f(int a, int b = 0)`).

Key ideas:

- **Scope**: where a variable is visible (local vs global).
- **Call stack**: each call creates a **stack frame** (parameters, local variables, return address). It is destroyed when the function returns.
- **Pure function**: same input → same output, no side effects. Easy to test and reason about.
- **Higher-order function**: takes or returns another function (`map`, `filter`, callbacks).

---

## A.4 Recursion

A function that calls itself. Every recursion needs:

1. **Base case** - stops the recursion.
2. **Recursive case** - reduces the problem toward the base case.

```js
function factorial(n) {
  if (n <= 1) return 1;          // base case
  return n * factorial(n - 1);   // recursive case
}
```

**C++:**

```cpp
unsigned long long factorial(unsigned int n) {
    if (n <= 1) return 1;                 // base case
    return n * factorial(n - 1);          // recursive case
}
```

Overflow warning: $20! = 2{,}432{,}902{,}008{,}176{,}640{,}000$ is the largest factorial that fits in `unsigned long long`; $21!$ silently wraps around. In JavaScript, use `BigInt` for exact large integers.

$$n! = n \times (n-1)!, \quad 0! = 1$$

**Danger:** each call uses a stack frame. Too-deep recursion causes **stack overflow**. Recursion depth $d$ uses $O(d)$ stack space.

**Fibonacci** shows why naive recursion can be exponentially slow:

```js
function fib(n) {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2); // recomputes the same values many times
}
```

**C++ (naive and memoized):**

```cpp
#include <vector>

long long fibNaive(int n) {
    if (n <= 1) return n;
    return fibNaive(n - 1) + fibNaive(n - 2);              // O(2^n)
}

long long fibMemo(int n, std::vector<long long>& memo) {
    if (n <= 1) return n;
    if (memo[n] != -1) return memo[n];                     // already computed
    return memo[n] = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);   // O(n)
}
// usage: std::vector<long long> memo(n + 1, -1);  fibMemo(n, memo);
```

$F(92)$ is the largest Fibonacci number that fits in a signed 64-bit integer.

**Stack depth in C++:** each call takes a stack frame from a fixed-size stack (commonly 8 MB on Linux, 1 MB on Windows by default). Recursion tens of thousands of levels deep can crash with a stack overflow. For very deep recursion, convert to a loop with an explicit `std::stack` or `std::vector`.

Time: $T(n) = T(n-1) + T(n-2) + O(1) \Rightarrow O(2^n)$. With memoization (caching results) it becomes $O(n)$. This is the same idea as **caching** in system design.

---

## A.5 Arrays and Strings

An **array** stores elements in **contiguous memory**. Element $i$ lives at:

$$\text{address}(i) = \text{base} + i \times \text{elementSize}$$

This gives $O(1)$ random access. Insertion/deletion in the middle is $O(n)$ because elements must shift.

A **string** is an array of characters. In many languages (JavaScript, Java, Python) strings are **immutable**: "modifying" one creates a new string. Repeated concatenation in a loop can therefore cost $O(n^2)$ overall.

**C++:**

```cpp
#include <array>
#include <iostream>
#include <string>
#include <vector>

int main() {
    int raw[5] = {1, 2, 3, 4, 5};          // C-style array: fixed size, no bounds checking
    std::array<int, 3> fixed = {1, 2, 3};  // fixed size, safer wrapper
    std::vector<int> v = {1, 2, 3};        // dynamic array, contiguous memory
    v.push_back(4);                        // amortized O(1)

    int a = v[2];                          // O(1), NO bounds check (out of range = undefined behavior)
    int b = v.at(2);                       // O(1), throws std::out_of_range if invalid

    std::string s = "hello";               // std::string is MUTABLE
    s += " world";                         // appends in place, amortized O(1) per character
    std::cout << s << "\n";
}
```

Reading or writing past the end of a C-style array is a **buffer overflow**, a classic source of crashes and security vulnerabilities.

---

## A.6 Pointers / References

- A **pointer** stores a memory address.
- A **reference** is a safer alias to an object (used in Java, JavaScript, Python).

```cpp
int x = 10;
int *p = &x;   // p holds the address of x
*p = 20;       // writes to x through the pointer -> x is now 20
```

Dereferencing a `NULL`/`null` pointer is a classic crash. Linked lists, trees and graphs are all built with pointers/references.

**C++ pointers, references and smart pointers:**

```cpp
#include <memory>

int x = 10;
int& r = x;                      // reference: another name for x; must be initialized, can never be null
r = 30;                          // x is now 30

int* p = nullptr;                // always use nullptr (not NULL or 0) in modern C++
p = new int(5);                  // allocate an int on the heap
delete p;                        // you must free it yourself
p = nullptr;

auto sp = std::make_unique<int>(5);   // smart pointer: owns the int, frees it automatically
```

**Pointer arithmetic:** for `int arr[5]; int* q = arr;`, the expression `q + i` points to `arr[i]`. The address moves by `i * sizeof(int)` bytes, not by `i` bytes.

Common pointer bugs: **dangling pointer** (points to freed memory), **double free**, **use-after-free**, **null dereference**, **memory leak**. Modern C++ avoids them with references and smart pointers (`std::unique_ptr`, `std::shared_ptr`).

---

## A.7 Memory Allocation, Stack vs Heap

| | Stack | Heap |
| --- | --- | --- |
| Holds | Function frames, local variables | Dynamically allocated objects |
| Allocation | Automatic, very fast (move a pointer) | Manual or garbage-collected, slower |
| Lifetime | Ends when function returns | Until freed / garbage collected |
| Size | Small, fixed limit | Large, limited by RAM |
| Failure | Stack overflow | Out of memory, leaks, fragmentation |

```js
function demo() {
  let n = 5;              // primitive: lives in the stack frame
  let obj = { a: 1 };     // the object lives on the heap; `obj` (a reference) is in the frame
}
```

**C++ (where each thing lives):**

```cpp
#include <memory>
#include <vector>

void demo() {
    int n = 5;                            // stack
    int arr[3] = {1, 2, 3};               // stack
    int* p = new int(5);                  // the int is on the heap; the pointer p is on the stack
    std::vector<int> v(1000);             // the vector object is on the stack; its 1000 ints are on the heap
    auto up = std::make_unique<int>(7);   // heap, freed automatically when `up` goes out of scope

    delete p;                             // forgetting this line = memory leak
}   // stack frame (n, arr, p, v, up) is destroyed here; v and up free their heap memory in their destructors
```

**RAII (Resource Acquisition Is Initialization):** tie a resource's lifetime to an object's lifetime. The destructor releases the resource, so cleanup happens automatically even when an exception is thrown. This is the central C++ idiom for memory, files, locks and sockets.

A huge local array (for example `int big[10'000'000];` inside a function) can overflow the small stack; allocate large data on the heap (`std::vector`).

**Memory leak:** memory that is no longer needed but is still reachable (or never freed), so usage grows until the process crashes. Very common in long-running servers.

---

## A.8 Pass by Value vs Pass by Reference

- **Pass by value:** the function receives a *copy*. Changing it does not affect the caller.
- **Pass by reference:** the function receives an alias to the original.

```js
function change(x) { x = 100; }
let a = 1;
change(a);
console.log(a); // 1  (copy)

function mutate(o) { o.count = 100; }
let obj = { count: 1 };
mutate(obj);
console.log(obj.count); // 100
```

In JavaScript, Java and Python everything is technically **pass by value**, but for objects the *value copied is the reference*. So you can mutate the shared object, but reassigning the parameter does not change the caller's variable.

**C++ supports both explicitly:**

```cpp
#include <string>

void byValue(int x)                { x = 100; }        // gets a copy; caller unaffected
void byRef(int& x)                 { x = 100; }        // alias of the caller's variable; caller changes
void byPtr(int* x)                 { *x = 100; }       // address of the caller's variable
void byConstRef(const std::string& s) { /* read s, cannot modify it */ }

int main() {
    int a = 1;
    byValue(a);   // a is still 1
    byRef(a);     // a is now 100
    a = 1;
    byPtr(&a);    // a is now 100
}
```

Passing a large object (a `std::string`, `std::vector`, or your own class) **by value copies all of it**, which can be very expensive. Rule of thumb: pass small types (`int`, `double`) by value and large types by `const&`.

---

## A.9 Error Handling

Programs fail: bad input, network down, disk full. Handle errors explicitly.

```js
try {
  const data = JSON.parse(input);
} catch (err) {
  console.error("Invalid JSON:", err.message);
} finally {
  cleanup(); // always runs
}
```

**C++:**

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

int main() {
    std::string input = "abc";
    try {
        int v = std::stoi(input);     // throws std::invalid_argument if not a number
        std::cout << v << "\n";
    } catch (const std::invalid_argument& e) {
        std::cerr << "Not a number: " << e.what() << "\n";
    } catch (const std::exception& e) {          // catch by const reference
        std::cerr << "Error: " << e.what() << "\n";
    }
}
```

C++ has **no `finally`**. Cleanup is done by destructors (RAII, see A.7). For example, `std::lock_guard<std::mutex> lock(m);` unlocks the mutex automatically when `lock` goes out of scope, even if an exception is thrown, and `std::ifstream` closes its file automatically. Throw exceptions by value and catch them by `const` reference. Many C++ codebases (and all C APIs) also report errors through return codes instead of exceptions.

Good practice:

- Fail fast on invalid input; validate at boundaries.
- Never swallow errors silently.
- Distinguish **expected** errors (validation) from **unexpected** ones (bugs).
- Always release resources (files, connections) - use `finally` or equivalent.

---

## A.10 File I/O

Reading and writing files goes through the OS (system calls, see doc 02).

```js
const fs = require("fs");

const text = fs.readFileSync("input.txt", "utf8");   // blocking
fs.writeFileSync("output.txt", text.toUpperCase());

fs.readFile("input.txt", "utf8", (err, data) => {    // non-blocking
  if (err) throw err;
  console.log(data);
});
```

**C++:**

```cpp
#include <fstream>
#include <iostream>
#include <string>

int main() {
    std::ifstream in("input.txt");
    if (!in) {
        std::cerr << "cannot open input.txt\n";
        return 1;
    }

    std::ofstream out("output.txt");
    std::string line;
    while (std::getline(in, line)) {      // processes one line at a time: constant memory
        out << line << "\n";
    }
}   // `in` and `out` are closed automatically here (RAII)
```

C++ streams are **buffered**: many small `<<` writes are collected and sent to the OS in fewer, larger `write` system calls (doc 02), which is much faster.

Disk I/O is **orders of magnitude slower** than memory. Large files should be processed in **streams/chunks**, not loaded fully into memory.

---

## A.11 Modules / Packages

A **module** is a file exposing functionality; a **package** is a distributable collection of modules.

```js
// math.js
module.exports = { add: (a, b) => a + b };

// app.js
const { add } = require("./math");
```

**C++ (headers and source files):**

```cpp
// math.h  (declaration: the interface)
#pragma once
int add(int a, int b);

// math.cpp  (definition: the implementation)
#include "math.h"
int add(int a, int b) { return a + b; }

// main.cpp
#include <iostream>
#include "math.h"
int main() { std::cout << add(2, 3) << "\n"; }
```

```bash
g++ -std=c++17 main.cpp math.cpp -o app    # compiles each .cpp to an object file, then links them
```

`#include` makes the preprocessor **copy the header's text** into the file, so headers should contain declarations only. `namespace` groups names to avoid collisions (`std::vector`). C++20 adds real `import` modules, but header/source files are still the norm. Larger projects use **CMake** as the build tool and **vcpkg** or **Conan** for dependencies.

Benefits: reuse, separation of concerns, namespaces, testing. Package managers (npm, pip, Maven) handle **dependencies** and **versions**.

---

## A.12 Basic Debugging

1. **Reproduce** the bug reliably.
2. **Isolate** the smallest failing case.
3. **Inspect** state: logs, breakpoints, stack traces.
4. **Form a hypothesis**, change one thing, re-test.
5. **Add a test** so the bug never returns.

Reading a **stack trace** from top to bottom shows the chain of function calls that led to the error.

**C++ debugging tools:** compile with `-g` (debug symbols) and debug with `gdb` or `lldb`. Catch memory bugs with sanitizers (`-fsanitize=address,undefined` enables AddressSanitizer and UndefinedBehaviorSanitizer) or with Valgrind. A **segmentation fault** means the program touched memory it may not access (null/dangling pointer, buffer overflow).

---

## A.13 Time & Space Complexity

Complexity measures how **cost grows with input size $n$**, ignoring constants.

- **Time complexity:** number of basic operations.
- **Space complexity:** extra memory used.

### Common growth rates (slowest growth → fastest)

| Notation | Name | Example |
| --- | --- | --- |
| $O(1)$ | Constant | Array index, hash lookup |
| $O(\log n)$ | Logarithmic | Binary search, B-tree lookup |
| $O(n)$ | Linear | Scan a list |
| $O(n \log n)$ | Linearithmic | Merge sort |
| $O(n^2)$ | Quadratic | Nested loops |
| $O(2^n)$ | Exponential | Naive Fibonacci, subsets |
| $O(n!)$ | Factorial | All permutations |

For $n = 10^6$: $\log_2 n \approx 20$, $n \log_2 n \approx 2\times10^7$, but $n^2 = 10^{12}$ (unusable).

### How to count

```js
for (let i = 0; i < n; i++) {        // n iterations
  for (let j = 0; j < n; j++) {      // n iterations each
    doWork();                        // -> n * n = n^2
  }
}
```

**C++:**

```cpp
for (int i = 0; i < n; i++)          // n iterations
    for (int j = 0; j < n; j++)      // n iterations each
        doWork();                    // n * n = n^2
```

Rules: drop constants ($3n \to n$), keep the dominant term ($n^2 + n \to n^2$), sequential blocks **add**, nested blocks **multiply**.

---

## A.14 Big-O, Big-Theta, Big-Omega

Formal definitions (for $f, g : \mathbb{N} \to \mathbb{R}^+$):

**Big-O (upper bound)**

$$f(n) = O(g(n)) \iff \exists\, c>0,\, n_0 : \; f(n) \le c\,g(n) \;\; \forall n \ge n_0$$

**Big-Omega (lower bound)**

$$f(n) = \Omega(g(n)) \iff \exists\, c>0,\, n_0 : \; f(n) \ge c\,g(n) \;\; \forall n \ge n_0$$

**Big-Theta (tight bound)**

$$f(n) = \Theta(g(n)) \iff f(n) = O(g(n)) \;\text{and}\; f(n) = \Omega(g(n))$$

Example: $f(n) = 3n^2 + 5n + 2$ is $\Theta(n^2)$.

Informally: $O$ = "at most", $\Omega$ = "at least", $\Theta$ = "exactly this growth". In practice people say "Big-O" and usually mean the worst case.

**Best / average / worst case.** Linear search: best $O(1)$ (first element), worst $O(n)$. Quicksort: average $O(n\log n)$, worst $O(n^2)$.

**Amortized cost.** A dynamic array doubles its capacity when full. One push may cost $O(n)$, but averaged over many pushes the cost is $O(1)$ each.

---

# Part B - Data Structures & Algorithms

## B.1 Arrays

Contiguous memory. Access $O(1)$; search $O(n)$ (unsorted); insert/delete in middle $O(n)$; append amortized $O(1)$ (dynamic array).

**Why it matters:** cache-friendly because neighbouring elements sit together in memory (locality of reference, doc 01).

**C++:**

```cpp
#include <array>
#include <vector>

std::array<int, 3> a = {1, 2, 3};   // fixed size, lives on the stack, no overhead
std::vector<int> v;
v.reserve(1000);                    // pre-allocate capacity to avoid repeated reallocation
v.push_back(1);                     // amortized O(1)
v.insert(v.begin(), 0);             // O(n): shifts every element right
v.erase(v.begin());                 // O(n): shifts every element left
```

When a `std::vector` runs out of capacity it allocates a larger block (growth factor is implementation-defined, commonly 1.5x to 2x), moves all elements, and frees the old block. This invalidates existing pointers, references and iterators into the vector.

## B.2 Linked Lists

Nodes connected by pointers.

```js
class Node {
  constructor(value) {
    this.value = value;
    this.next = null;
  }
}
```

**C++:**

```cpp
struct Node {
    int value;
    Node* next;
    explicit Node(int v) : value(v), next(nullptr) {}
};

Node* pushFront(Node* head, int v) {
    Node* n = new Node(v);
    n->next = head;      // O(1): the new node points at the old head
    return n;            // the new node is the new head
}
// Every `new` needs a matching `delete`. In real code use std::list (doubly linked)
// or std::forward_list (singly linked).
```

| Operation | Singly linked list |
| --- | --- |
| Access by index | $O(n)$ |
| Insert/delete at head | $O(1)$ |
| Insert/delete after a known node | $O(1)$ |
| Search | $O(n)$ |

Used inside LRU caches (doubly linked list), hash-table chaining and queues.

## B.3 Stacks

**LIFO** (last in, first out). Operations `push`, `pop`, `peek`, all $O(1)$.

```js
const stack = [];
stack.push(1); stack.push(2);
stack.pop(); // 2
```

**C++:**

```cpp
#include <stack>

std::stack<int> st;
st.push(1);
st.push(2);
int top = st.top();   // 2: reads the top without removing it
st.pop();             // removes the top; returns void in C++
```

Uses: function call stack, undo, expression parsing, DFS.

## B.4 Queues

**FIFO** (first in, first out). Operations `enqueue`, `dequeue`, $O(1)$ (with a linked list or circular buffer).

Variants: **deque** (both ends), **priority queue** (highest priority first, usually a heap).

Uses: task scheduling, BFS, **message queues** (doc 25), request buffering.

**C++:**

```cpp
#include <deque>
#include <queue>

std::queue<int> q;
q.push(1);
q.push(2);
int front = q.front();   // 1
q.pop();                 // O(1); returns void

std::deque<int> dq;      // double-ended queue: push_front, push_back, pop_front, pop_back are all O(1)
```

## B.5 Hash Tables

Map keys to values using a **hash function** $h(k)$ that picks a bucket:

$$\text{index} = h(k) \bmod m$$

where $m$ is the number of buckets.

- Average lookup/insert/delete: $O(1)$. Worst case: $O(n)$ if many keys collide.
- **Collision** = two keys map to the same bucket. Handled by:
  - **Chaining:** each bucket holds a list.
  - **Open addressing:** probe for another empty slot (linear/quadratic probing, double hashing).
- **Load factor:** $\alpha = \dfrac{n}{m}$ ($n$ stored keys). When $\alpha$ gets too large (commonly $\approx 0.75$), the table **resizes** and rehashes.

```js
const map = new Map();
map.set("user:1", { name: "Asha" });
map.get("user:1"); // O(1) average
```

**C++:**

```cpp
#include <string>
#include <unordered_map>

std::unordered_map<std::string, std::string> m;
m["user:1"] = "Asha";                    // insert or update, average O(1)
auto it = m.find("user:1");              // average O(1)
if (it != m.end()) {
    // it->second == "Asha"
}
m.erase("user:1");
```

Notes: `m[key]` **inserts a default value if the key is missing**, so use `find` or `count` to only check. `std::unordered_map` is a hash table (average $O(1)$); `std::map` is an ordered balanced tree ($O(\log n)$). `std::unordered_map` grows when its load factor exceeds `max_load_factor()` (default $1.0$ in C++, not $0.75$ as in Java).

Hash tables underlie caches, Redis, sharding (`hash(key) % N`), and indexes.

## B.6 Trees

A **tree** is a connected acyclic graph with a root; each node has children.

Terms: **root**, **leaf**, **parent/child**, **depth** (distance from root), **height** (longest path to a leaf), **subtree**.

Tree with $n$ nodes has exactly $n-1$ edges.

## B.7 Binary Trees

Each node has at most 2 children (left, right).

```js
class TreeNode {
  constructor(val) { this.val = val; this.left = null; this.right = null; }
}
```

**Traversals**

| Order | Visit sequence |
| --- | --- |
| Inorder | left, node, right |
| Preorder | node, left, right |
| Postorder | left, right, node |
| Level order | level by level (BFS) |

**C++:**

```cpp
#include <vector>

struct TreeNode {
    int val;
    TreeNode* left = nullptr;
    TreeNode* right = nullptr;
    explicit TreeNode(int v) : val(v) {}
};

void inorder(const TreeNode* node, std::vector<int>& out) {
    if (!node) return;              // base case
    inorder(node->left, out);       // left
    out.push_back(node->val);       // node
    inorder(node->right, out);      // right
}
```

A binary tree of height $h$ has at most $2^{h+1}-1$ nodes; with $n$ nodes the minimum possible height is $\lfloor \log_2 n \rfloor$.

## B.8 Binary Search Tree (BST)

Binary tree where for every node: all values in the left subtree are **smaller**, all in the right subtree are **larger**.

- Search/insert/delete: $O(h)$.
- Balanced: $h = O(\log n)$. Degenerate (sorted input inserted in order): $h = O(n)$.
- **Self-balancing** trees (AVL, Red-Black) keep $h = O(\log n)$.
- Inorder traversal of a BST yields sorted order.

- **In C++**, `std::set` and `std::map` are self-balancing BSTs (typically red-black trees): $O(\log n)$ insert, find and erase, sorted iteration, and `lower_bound` / `upper_bound` for range queries. `std::unordered_set` / `std::unordered_map` are hash tables instead.

Databases use a generalization: **B-trees / B+ trees** (doc 12).

## B.9 Heaps

A **complete binary tree** stored in an array satisfying the heap property:

- **Min-heap:** parent $\le$ children (minimum at root).
- **Max-heap:** parent $\ge$ children.

For a node at index $i$ (0-based):

$$\text{parent}(i)=\left\lfloor \tfrac{i-1}{2} \right\rfloor,\quad \text{left}(i)=2i+1,\quad \text{right}(i)=2i+2$$

| Operation | Cost |
| --- | --- |
| Peek min/max | $O(1)$ |
| Insert | $O(\log n)$ |
| Extract min/max | $O(\log n)$ |
| Build heap from $n$ items | $O(n)$ |

Uses: priority queues, schedulers, top-K, Dijkstra.

**C++:**

```cpp
#include <functional>
#include <queue>
#include <vector>

std::priority_queue<int> maxHeap;                                       // max-heap by default
std::priority_queue<int, std::vector<int>, std::greater<int>> minHeap;  // min-heap

minHeap.push(5);
minHeap.push(1);
minHeap.push(3);
int smallest = minHeap.top();   // 1, O(1)
minHeap.pop();                  // O(log n)
```

JavaScript has no built-in heap; you implement one on an array using the index formulas above.

## B.10 Graphs

$G = (V, E)$: vertices and edges. Edges may be **directed/undirected** and **weighted/unweighted**.

**Representations**

| | Adjacency list | Adjacency matrix |
| --- | --- | --- |
| Space | $O(V+E)$ | $O(V^2)$ |
| Check edge $(u,v)$ | $O(\deg u)$ | $O(1)$ |
| Iterate neighbours | $O(\deg u)$ | $O(V)$ |
| Best for | Sparse graphs | Dense graphs |

```js
const graph = { A: ["B", "C"], B: ["D"], C: ["D"], D: [] };
```

**C++:**

```cpp
#include <utility>
#include <vector>

int n = 4;                                      // vertices 0..3  (A=0, B=1, C=2, D=3)
std::vector<std::vector<int>> adj(n);           // adjacency list
adj[0] = {1, 2};
adj[1] = {3};
adj[2] = {3};

std::vector<std::vector<std::pair<int, int>>> wadj(n);   // weighted: (neighbour, weight)
wadj[0].push_back({1, 7});
```

Graphs model networks, service dependencies, social connections, web links.

## B.11 Tries

A **prefix tree**: each edge is a character; paths from the root spell out stored words.

- Insert/search a word of length $L$: $O(L)$, independent of how many words are stored.
- Great for autocomplete and prefix search.

**C++:**

```cpp
#include <array>
#include <memory>
#include <string>

struct TrieNode {
    std::array<std::unique_ptr<TrieNode>, 26> child{};   // for letters a-z; all null at first
    bool isWord = false;
};

void insert(TrieNode& root, const std::string& word) {
    TrieNode* cur = &root;
    for (char ch : word) {
        int i = ch - 'a';
        if (!cur->child[i]) cur->child[i] = std::make_unique<TrieNode>();
        cur = cur->child[i].get();
    }
    cur->isWord = true;
}

bool contains(const TrieNode& root, const std::string& word) {
    const TrieNode* cur = &root;
    for (char ch : word) {
        cur = cur->child[ch - 'a'].get();
        if (!cur) return false;
    }
    return cur->isWord;
}
```

Using `unique_ptr` means the whole trie frees itself automatically.

## B.12 Sorting

| Algorithm | Best | Average | Worst | Space | Stable |
| --- | --- | --- | --- | --- | --- |
| Bubble / Insertion | $O(n)$ | $O(n^2)$ | $O(n^2)$ | $O(1)$ | Yes |
| Merge sort | $O(n\log n)$ | $O(n\log n)$ | $O(n\log n)$ | $O(n)$ | Yes |
| Quick sort | $O(n\log n)$ | $O(n\log n)$ | $O(n^2)$ | $O(\log n)$ | No |
| Heap sort | $O(n\log n)$ | $O(n\log n)$ | $O(n\log n)$ | $O(1)$ | No |

Any comparison-based sort needs $\Omega(n \log n)$ comparisons in the worst case. **Stable** means equal elements keep their original relative order.

**External sorting** (data larger than RAM) uses merge sort on disk chunks - this shows up in databases and big-data systems.

**C++:**

```cpp
#include <algorithm>
#include <vector>

std::vector<int> v = {5, 2, 9, 1};
std::sort(v.begin(), v.end());                                       // ascending, O(n log n) worst case
std::sort(v.begin(), v.end(), [](int a, int b) { return a > b; });   // descending via a comparator
std::stable_sort(v.begin(), v.end());                                // stable, O(n log n)
```

JavaScript gotcha: `arr.sort()` with no comparator sorts elements **as strings** (`[10, 9, 1].sort()` gives `[1, 10, 9]`). For numbers use `arr.sort((a, b) => a - b)`.

## B.13 Searching

- **Linear search:** $O(n)$, works on anything.
- **Binary search:** requires sorted data; halves the search range each step.

$$T(n) = T(n/2) + O(1) \;\Rightarrow\; O(\log n)$$

```js
function binarySearch(arr, target) {
  let lo = 0, hi = arr.length - 1;
  while (lo <= hi) {
    const mid = lo + Math.floor((hi - lo) / 2); // avoids overflow of (lo+hi)
    if (arr[mid] === target) return mid;
    if (arr[mid] < target) lo = mid + 1;
    else hi = mid - 1;
  }
  return -1;
}
```

**C++:**

```cpp
#include <vector>

int binarySearch(const std::vector<int>& arr, int target) {
    int lo = 0, hi = (int)arr.size() - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;      // NOT (lo + hi) / 2: that sum can overflow a 32-bit int
        if (arr[mid] == target) return mid;
        if (arr[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}
// Standard library: std::binary_search (returns true/false), std::lower_bound (first element >= target),
// std::upper_bound (first element > target). All require sorted input and run in O(log n).
```

Searching 1 billion sorted items takes only about $\log_2(10^9) \approx 30$ steps.

## B.14 Recursion & Backtracking

**Backtracking** builds a solution step by step and **undoes** a step when it leads to a dead end.

```js
// Generate all subsets
function subsets(nums) {
  const result = [], path = [];
  function go(i) {
    if (i === nums.length) { result.push([...path]); return; }
    go(i + 1);                 // skip nums[i]
    path.push(nums[i]);        // choose nums[i]
    go(i + 1);
    path.pop();                // undo (backtrack)
  }
  go(0);
  return result;
}
```

**C++:**

```cpp
#include <vector>

void go(const std::vector<int>& nums, int i,
        std::vector<int>& path, std::vector<std::vector<int>>& result) {
    if (i == (int)nums.size()) { result.push_back(path); return; }
    go(nums, i + 1, path, result);      // skip nums[i]
    path.push_back(nums[i]);            // choose nums[i]
    go(nums, i + 1, path, result);
    path.pop_back();                    // undo (backtrack)
}
```

Number of subsets of $n$ items: $2^n$.

## B.15 BFS and DFS

Both visit every reachable vertex in $O(V+E)$ time.

**BFS** (queue): explores level by level → finds the **shortest path in unweighted graphs**.

```js
function bfs(graph, start) {
  const visited = new Set([start]);
  const queue = [start];
  while (queue.length) {
    const node = queue.shift();
    for (const next of graph[node]) {
      if (!visited.has(next)) {
        visited.add(next);
        queue.push(next);
      }
    }
  }
  return visited;
}
```

**C++ BFS:**

```cpp
#include <queue>
#include <vector>

std::vector<int> bfs(const std::vector<std::vector<int>>& graph, int start) {
    std::vector<bool> visited(graph.size(), false);
    std::vector<int> order;
    std::queue<int> q;

    visited[start] = true;
    q.push(start);
    while (!q.empty()) {
        int node = q.front();
        q.pop();
        order.push_back(node);
        for (int next : graph[node]) {
            if (!visited[next]) {
                visited[next] = true;
                q.push(next);
            }
        }
    }
    return order;
}
```

Performance note: `std::queue::pop()` is $O(1)$. In the JavaScript version, `Array.shift()` can cost $O(n)$, so for large graphs use an index pointer or a real queue implementation.

**DFS** (stack or recursion): goes as deep as possible first. Used for cycle detection, topological sort, connected components.

```js
function dfs(graph, node, visited = new Set()) {
  visited.add(node);
  for (const next of graph[node]) {
    if (!visited.has(next)) dfs(graph, next, visited);
  }
  return visited;
}
```

**C++ DFS:**

```cpp
#include <vector>

void dfs(const std::vector<std::vector<int>>& graph, int node, std::vector<bool>& visited) {
    visited[node] = true;
    for (int next : graph[node])
        if (!visited[next]) dfs(graph, next, visited);
}
```

Very deep graphs can overflow the call stack; use an explicit `std::stack` for an iterative DFS.

**Topological sort** (ordering a DAG by dependencies) is exactly how build systems and this roadmap's learning order work.

## B.16 Greedy Algorithms

Make the **locally best choice** at each step, hoping for a global optimum. Correct only when the problem has the **greedy-choice property** and **optimal substructure**.

Examples: interval scheduling (pick earliest finishing), Huffman coding, Dijkstra's shortest path (non-negative weights).

Greedy fails for some problems, e.g. coin change with coins $\{1, 3, 4\}$ and target $6$: greedy gives $4+1+1$ (3 coins), optimum is $3+3$ (2 coins).

## B.17 Dynamic Programming

Solve a problem by combining answers to **overlapping subproblems**, storing each answer once.

Requirements: **optimal substructure** + **overlapping subproblems**.

```js
// Fibonacci: top-down (memoization)
function fib(n, memo = {}) {
  if (n <= 1) return n;
  if (memo[n] !== undefined) return memo[n];
  return memo[n] = fib(n - 1, memo) + fib(n - 2, memo);
}

// Coin change: bottom-up (tabulation)
function minCoins(coins, amount) {
  const dp = Array(amount + 1).fill(Infinity);
  dp[0] = 0;
  for (let a = 1; a <= amount; a++) {
    for (const c of coins) {
      if (a - c >= 0) dp[a] = Math.min(dp[a], dp[a - c] + 1);
    }
  }
  return dp[amount] === Infinity ? -1 : dp[amount];
}
```

Recurrence for coin change:

$$dp[a] = \min_{c \in \text{coins},\, c \le a} \big(dp[a-c] + 1\big), \quad dp[0]=0$$

Time $O(\text{amount} \times |\text{coins}|)$.

**C++ (coin change, bottom-up):**

```cpp
#include <algorithm>
#include <climits>
#include <vector>

int minCoins(const std::vector<int>& coins, int amount) {
    const int INF = INT_MAX / 2;               // half of max so that INF + 1 never overflows
    std::vector<int> dp(amount + 1, INF);
    dp[0] = 0;
    for (int a = 1; a <= amount; a++)
        for (int c : coins)
            if (a - c >= 0) dp[a] = std::min(dp[a], dp[a - c] + 1);
    return dp[amount] >= INF ? -1 : dp[amount];
}
```

The memoized C++ Fibonacci is in A.4.

## B.18 Basic Algorithmic Complexity - Data Structure Cheat Sheet

| Structure | Access | Search | Insert | Delete |
| --- | --- | --- | --- | --- |
| Array | $O(1)$ | $O(n)$ | $O(n)$ | $O(n)$ |
| Linked list | $O(n)$ | $O(n)$ | $O(1)^*$ | $O(1)^*$ |
| Stack / Queue | - | $O(n)$ | $O(1)$ | $O(1)$ |
| Hash table (avg) | - | $O(1)$ | $O(1)$ | $O(1)$ |
| Balanced BST | - | $O(\log n)$ | $O(\log n)$ | $O(\log n)$ |
| Heap | peek $O(1)$ | $O(n)$ | $O(\log n)$ | $O(\log n)$ |

$^*$ when the position/node is already known.

**C++ standard library containers and their costs:**

| Container | Underlying structure | Notable costs |
| --- | --- | --- |
| `std::vector` | Dynamic array | Index $O(1)$; `push_back` amortized $O(1)$; middle insert/erase $O(n)$ |
| `std::array` | Fixed array | Index $O(1)$ |
| `std::list` / `std::forward_list` | Doubly / singly linked list | Insert/erase at a known position $O(1)$; no random access |
| `std::deque` | Chunked array | Push/pop at both ends $O(1)$; index $O(1)$ |
| `std::stack`, `std::queue` | Adapters over `deque` | Push/pop $O(1)$ |
| `std::priority_queue` | Binary heap in a `vector` | `top` $O(1)$; push/pop $O(\log n)$ |
| `std::map` / `std::set` | Red-black tree | Find/insert/erase $O(\log n)$, sorted |
| `std::unordered_map` / `std::unordered_set` | Hash table | Average $O(1)$, worst $O(n)$ |

Default choice: **`std::vector`**. Its contiguous memory gives the best cache behavior, and it often beats theoretically "faster" structures for realistic sizes.

---

# Why This Comes First

| Concept here | Where it reappears |
| --- | --- |
| Hash table, $h(k) \bmod m$ | Sharding, caches, consistent hashing, Redis |
| Binary search, $O(\log n)$ | B-tree indexes, database lookups |
| Queue | Message queues, task queues, backpressure |
| Heap / priority queue | Schedulers, top-K, timers |
| Graph, BFS/DFS, topological sort | Service dependencies, build pipelines, crawlers |
| Memoization / DP | Caching |
| Stack vs heap, memory leaks | Node.js/JVM tuning, OOM incidents |
| Big-O | Every capacity and performance estimate |

---

# Key Takeaways

- Choose data structures by the operations you need most; cost is measured with Big-O.
- Hash tables give average $O(1)$ lookup; balanced trees give $O(\log n)$ and keep order.
- Recursion needs a base case; memoization/DP removes repeated work.
- Stack memory is automatic and small; heap memory is large and must be managed.

- In C++ you control object lifetimes: prefer RAII, `std::vector`, references and smart pointers over raw `new`/`delete`; avoid undefined behavior (signed overflow, out-of-bounds, dangling pointers).
- Never use floating point for money; never assume unbounded integers.
- Complexity growth, not raw speed, decides whether a system survives at scale.
