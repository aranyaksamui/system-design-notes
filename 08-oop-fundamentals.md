# 08 — OOP Fundamentals

**Object-oriented programming (OOP)** organizes a program as a set of **objects**, each bundling **data** (state) with the **operations** (behavior) that act on that data. OOP is the main prerequisite for low-level design (doc 20) and design patterns (doc 10).

Code here is shown in **JavaScript, Python, C++, C#** and, where it explains the machinery, **C**.

## OOP Across Languages at a Glance

| Feature | C | C++ | C# | Python | JavaScript |
|---|---|---|---|---|---|
| Classes | No (structs + functions) | Yes | Yes | Yes | Yes (`class` is syntax over prototypes) |
| Access control | Opaque pointers, `static` | `public` / `protected` / `private` | `public` / `protected` / `private` / `internal` | Convention (`_x`), name mangling (`__x`) | `#private` fields, convention |
| Inheritance | Manual (embed a struct) | Multiple | Single class + many interfaces | Multiple (MRO) | Single (prototype chain) |
| Method dispatch default | n/a | **Static** unless `virtual` | **Static** unless `virtual` | Dynamic (always) | Dynamic (always) |
| Interfaces | Structs of function pointers | Pure abstract classes | `interface` | `abc` / `Protocol` | Duck typing (TypeScript `interface`) |
| Method overloading | No | Yes | Yes | No | No |
| Memory | Manual | Manual / RAII | Garbage collected | Ref counting + cycle GC | Garbage collected |
| Cleanup | `free` | Destructors (deterministic) | `IDisposable` / finalizers | Context managers / `__del__` | `try/finally` |

---

# 8.1 Core OOP

## Class and Object

- A **class** is a blueprint: it defines the fields (data) and methods (behavior).
- An **object** is one **instance** of a class, with its own field values.

## Encapsulation

**Encapsulation** = keep an object's data private and expose it only through methods, so the object can **enforce its own rules (invariants)**. Below, a balance can never become negative because the only way to change it is through checked methods. (Money is stored as integer cents, never floating point; doc 00.)

**JavaScript:**

```js
class BankAccount {
  #balanceCents = 0;                         // private field: not accessible outside the class

  constructor(openingCents = 0) {
    if (openingCents < 0) throw new RangeError("negative opening balance");
    this.#balanceCents = openingCents;
  }
  deposit(cents) {
    if (cents <= 0) throw new RangeError("deposit must be positive");
    this.#balanceCents += cents;
  }
  withdraw(cents) {
    if (cents > this.#balanceCents) throw new Error("insufficient funds");
    this.#balanceCents -= cents;
  }
  get balanceCents() { return this.#balanceCents; }   // read-only view
}

const acct = new BankAccount(1000);
acct.deposit(500);
console.log(acct.balanceCents);   // 1500
// acct.#balanceCents;            // SyntaxError: private field
```

**Python:**

```python
class BankAccount:
    def __init__(self, opening_cents: int = 0):
        if opening_cents < 0:
            raise ValueError("negative opening balance")
        self._balance_cents = opening_cents      # underscore = "internal"; enforced only by convention

    def deposit(self, cents: int) -> None:
        if cents <= 0:
            raise ValueError("deposit must be positive")
        self._balance_cents += cents

    def withdraw(self, cents: int) -> None:
        if cents > self._balance_cents:
            raise ValueError("insufficient funds")
        self._balance_cents -= cents

    @property
    def balance_cents(self) -> int:              # read-only attribute
        return self._balance_cents
```

**C++:**

```cpp
#include <stdexcept>

class BankAccount {
    long long balanceCents_;                     // private by default in a class
public:
    explicit BankAccount(long long opening = 0) : balanceCents_(opening) {
        if (opening < 0) throw std::invalid_argument("negative opening balance");
    }
    void deposit(long long cents) {
        if (cents <= 0) throw std::invalid_argument("deposit must be positive");
        balanceCents_ += cents;
    }
    void withdraw(long long cents) {
        if (cents > balanceCents_) throw std::runtime_error("insufficient funds");
        balanceCents_ -= cents;
    }
    long long balanceCents() const { return balanceCents_; }   // const: promises not to modify the object
};
```

**C#:**

```csharp
public class BankAccount
{
    private long _balanceCents;

    public BankAccount(long openingCents = 0)
    {
        if (openingCents < 0) throw new ArgumentOutOfRangeException(nameof(openingCents));
        _balanceCents = openingCents;
    }
    public long BalanceCents => _balanceCents;                  // read-only property

    public void Deposit(long cents)
    {
        if (cents <= 0) throw new ArgumentOutOfRangeException(nameof(cents));
        _balanceCents += cents;
    }
    public void Withdraw(long cents)
    {
        if (cents > _balanceCents) throw new InvalidOperationException("insufficient funds");
        _balanceCents -= cents;
    }
}
```

**C** has no `private` keyword, but you get the same effect with an **opaque type**: the header declares the struct without defining it, so callers can only use pointers and the provided functions.

```c
/* account.h */
typedef struct Account Account;               /* incomplete type: layout hidden */
Account *account_create(long long opening_cents);
int      account_withdraw(Account *a, long long cents);   /* returns 0 on success */
long long account_balance(const Account *a);
void     account_destroy(Account *a);

/* account.c */
#include <stdlib.h>
#include "account.h"
struct Account { long long balance_cents; };

Account *account_create(long long opening_cents) {
    if (opening_cents < 0) return NULL;
    Account *a = malloc(sizeof *a);
    if (a) a->balance_cents = opening_cents;
    return a;
}
int account_withdraw(Account *a, long long cents) {
    if (cents > a->balance_cents) return -1;
    a->balance_cents -= cents;
    return 0;
}
long long account_balance(const Account *a) { return a->balance_cents; }
void account_destroy(Account *a) { free(a); }
```

## Abstraction

**Abstraction** = expose *what* something does, hide *how*. Callers depend on a simple contract (e.g. `gateway.charge(amount)`), not on the details (HTTP calls, retries, vendor SDK). Encapsulation hides the **data**; abstraction hides the **complexity**. Interfaces and abstract classes (8.3) are its main tools.

## Inheritance

**Inheritance** lets a class (**subclass / derived class**) reuse and extend another (**superclass / base class**), modelling an **is-a** relationship ("a Circle is a Shape").

```text
        Shape            ← base class: defines what all shapes can do
       ┌──┴───┐
    Circle   Rect        ← derived classes
```

Inheritance gives code reuse and enables polymorphism, but it also **tightly couples** the subclass to its parent; doc 09 explains why composition is often better.

## Polymorphism

**Polymorphism** ("many forms") = the same call behaves differently depending on the object's actual type. Code written against `Shape` works for every present and future shape.

For a circle of radius $r$ and a rectangle of width $w$ and height $h$:

$$A_{\text{circle}}=\pi r^{2},\qquad A_{\text{rect}}=w\cdot h$$

**JavaScript:**

```js
class Shape { area() { throw new Error("not implemented"); } }
class Circle extends Shape {
  constructor(r) { super(); this.r = r; }
  area() { return Math.PI * this.r ** 2; }
}
class Rect extends Shape {
  constructor(w, h) { super(); this.w = w; this.h = h; }
  area() { return this.w * this.h; }
}
const shapes = [new Circle(1), new Rect(2, 3)];
console.log(shapes.reduce((sum, s) => sum + s.area(), 0));   // π + 6 ≈ 9.14
```

**Python:**

```python
import math

class Shape:
    def area(self) -> float:
        raise NotImplementedError

class Circle(Shape):
    def __init__(self, r): self.r = r
    def area(self): return math.pi * self.r ** 2

class Rect(Shape):
    def __init__(self, w, h): self.w, self.h = w, h
    def area(self): return self.w * self.h

print(sum(s.area() for s in [Circle(1), Rect(2, 3)]))   # 9.14...
```

**C++** (polymorphism needs a pointer or reference to the base type):

```cpp
#include <cmath>
#include <memory>
#include <vector>

class Shape {
public:
    virtual ~Shape() = default;               // see 8.3: virtual destructor
    virtual double area() const = 0;          // pure virtual: derived classes must implement
};
class Circle : public Shape {
    double r_;
public:
    explicit Circle(double r) : r_(r) {}
    double area() const override { return M_PI * r_ * r_; }
};
class Rect : public Shape {
    double w_, h_;
public:
    Rect(double w, double h) : w_(w), h_(h) {}
    double area() const override { return w_ * h_; }
};

double total(const std::vector<std::unique_ptr<Shape>>& shapes) {
    double sum = 0;
    for (const auto& s : shapes) sum += s->area();   // runtime dispatch to Circle::area or Rect::area
    return sum;
}
```

**C#:**

```csharp
using System.Linq;

public abstract class Shape { public abstract double Area(); }

public class Circle : Shape
{
    private readonly double _r;
    public Circle(double r) => _r = r;
    public override double Area() => Math.PI * _r * _r;
}
public class Rect : Shape
{
    private readonly double _w, _h;
    public Rect(double w, double h) { _w = w; _h = h; }
    public override double Area() => _w * _h;
}

Shape[] shapes = { new Circle(1), new Rect(2, 3) };
double total = shapes.Sum(s => s.Area());
```

**Kinds of polymorphism**

| Kind | Meaning | Example |
|---|---|---|
| **Subtype** (runtime) | Base reference, derived behavior | `virtual` / `override` above |
| **Parametric** | One definition works for many types | C++ templates, C# generics, `List<T>` |
| **Ad hoc** | Same name, different implementations chosen by argument types | Overloading, operator overloading |

## The Four Pillars

| Pillar | One-line meaning | Benefit |
|---|---|---|
| Encapsulation | Data + rules together, internals hidden | Invariants enforced in one place |
| Abstraction | Expose a simple contract | Callers need not know details |
| Inheritance | Derive new classes from existing ones | Reuse, is-a modelling |
| Polymorphism | One interface, many behaviors | Extend without changing callers |

---

# 8.2 Relationships

Classes rarely stand alone. The relationship between two classes determines **coupling** (how much a change in one forces a change in the other) and **lifetime** (who creates and destroys whom). From weakest to strongest coupling:

| Relationship | Meaning | Lifetime | UML symbol | Example |
|---|---|---|---|---|
| **Dependency** | One class **uses** another briefly (parameter, local variable) | None | dashed arrow `- - ->` | `OrderService.confirm(order, emailClient)` |
| **Association** | One class **holds a reference** to another | Independent | solid line `———` | Teacher ↔ Student |
| **Aggregation** | "Has-a", whole-part; parts **can outlive** the whole | Independent | hollow diamond `◇——` | Team ◇— Player |
| **Composition** | "Owns-a", whole-part; parts **die with** the whole | Bound together | filled diamond `◆——` | House ◆— Room |
| **Inheritance** | "Is-a" | n/a | hollow triangle `——▷` | Dog ▷ Animal |
| **Realization** | Class **implements** an interface | n/a | dashed + triangle `- -▷` | Circle - -▷ Shape |

```text
 Car ◆────── Engine        composition: no Car, no Engine
 Team ◇───── Player        aggregation: players exist without the team
 Teacher ─── Student       association
 OrderService - - -> EmailClient   dependency (used only inside a method)
```

**C++** makes ownership visible in the type:

```cpp
#include <vector>

class Engine {};
class Player {};
class Student {};
class EmailClient {};
class Order {};

class Car {                          // COMPOSITION: Engine is a member object, destroyed with the Car
    Engine engine_;
};

class Team {                         // AGGREGATION: Team refers to Players it does not own
    std::vector<Player*> players_;   // non-owning pointers
};

class Teacher {                      // ASSOCIATION: Teacher knows its Students
    std::vector<Student*> students_;
};

class OrderService {                 // DEPENDENCY: EmailClient appears only as a parameter
public:
    void confirm(const Order& order, EmailClient& mail);
};
```

**Python / JavaScript / C#** use garbage collection, so ownership is expressed by **who creates the part and who can reach it**:

```python
class Engine: ...
class Player: ...

class Car:
    def __init__(self):
        self._engine = Engine()          # composition: Car creates and exclusively owns its Engine

class Team:
    def __init__(self, players: list[Player]):
        self._players = players          # aggregation: players are created elsewhere and passed in
```

```js
class Car { #engine = new Engine(); }                        // composition
class Team { constructor(players) { this.players = players; } }   // aggregation
```

```csharp
public class Car  { private readonly Engine _engine = new(); }          // composition
public class Team { private readonly List<Player> _players; public Team(List<Player> p) => _players = p; }  // aggregation
```

**Rule of thumb:** prefer the weakest relationship that works (dependency over association, composition over inheritance). Weaker links mean fewer ripple effects when code changes (doc 09).

---

# 8.3 Advanced OOP

## Constructors

A **constructor** runs when an object is created and must leave it in a **valid state**.

| Language | Constructor |
|---|---|
| C++ | `ClassName(...)`, with a **member initializer list**: `Account(long long c) : balance_(c) {}` |
| C# | `public ClassName(...)`; chain with `: this(...)` or `: base(...)` |
| Python | `__init__(self, ...)` initializes; `__new__` actually creates the object |
| JavaScript | `constructor(...)`; derived classes must call `super(...)` before using `this` |
| C | A plain function such as `account_create()` |

Variants: **default** (no arguments), **parameterized**, **copy**, and (C++) **move** constructors. A **static factory method** (`Account.fromJson(...)`) is useful when construction can fail or has several modes. A **private constructor** blocks outside creation (used by Singleton, doc 10).

## Destructors and Cleanup

Resources (memory, files, sockets, locks) must be released. Languages differ:

**C++:** the **destructor** runs deterministically when the object goes out of scope. This enables **RAII** (doc 00): tie a resource to an object's lifetime.

```cpp
#include <cstdio>

class File {
    std::FILE* f_;
public:
    explicit File(const char* path) : f_(std::fopen(path, "r")) {}
    ~File() { if (f_) std::fclose(f_); }       // runs automatically, even if an exception is thrown
    File(const File&) = delete;                // a file handle must have exactly one owner
    File& operator=(const File&) = delete;
};
```

**C#:** the garbage collector decides *when* memory is freed, so non-memory resources use `IDisposable` and `using`:

```csharp
class Conn : IDisposable
{
    public void Dispose() { /* close the socket / file */ }
}

using (var c = new Conn()) { /* use c */ }    // Dispose() is called even on exceptions
// or, C# 8+:   using var c = new Conn();
```

**Python:** context managers (`with`). `__del__` exists but its timing is not guaranteed; do not rely on it.

```python
class Conn:
    def __enter__(self): return self
    def __exit__(self, exc_type, exc, tb): self.close()   # always runs
    def close(self): ...

with Conn() as c:
    ...
```

**JavaScript:** no deterministic destructors; use `try { ... } finally { resource.close(); }`.

**C:** you call `free` / `fclose` yourself. Forgetting is a **leak**; doing it twice is a **double free**.

## Copy Constructor, Deep vs Shallow Copy

A **copy** creates a second object from an existing one.

- **Shallow copy:** copies the fields as they are. If a field is a **pointer or reference**, both objects now share the same target.
- **Deep copy:** also duplicates the objects the fields point to, so the copies are fully independent.

```text
 shallow:   A.data ──┐                deep:   A.data ──► [1,2,3]
                     ├─► [1,2,3]             B.data ──► [1,2,3]   (separate copy)
            B.data ──┘
```

**C++**: the compiler-generated copy constructor is **shallow** for raw pointers. A class that owns memory must define its own copy operations, otherwise both copies `delete` the same block (**double free**). This is the **Rule of Five**: if you define any of destructor, copy constructor, copy assignment, move constructor or move assignment, you probably need all five. The **Rule of Zero** says: avoid the problem by using `std::vector` / `std::unique_ptr` members so the defaults are correct.

```cpp
#include <algorithm>
#include <cstddef>

class Buffer {
    std::size_t n_;
    int* data_;
public:
    explicit Buffer(std::size_t n) : n_(n), data_(new int[n]()) {}
    ~Buffer() { delete[] data_; }

    Buffer(const Buffer& o) : n_(o.n_), data_(new int[o.n_]) {       // copy ctor: DEEP copy
        std::copy(o.data_, o.data_ + n_, data_);
    }
    Buffer& operator=(const Buffer& o) {                              // copy assignment
        if (this != &o) {
            int* p = new int[o.n_];
            std::copy(o.data_, o.data_ + o.n_, p);
            delete[] data_;
            data_ = p;
            n_ = o.n_;
        }
        return *this;
    }
    Buffer(Buffer&& o) noexcept : n_(o.n_), data_(o.data_) {          // move ctor: steal, O(1)
        o.n_ = 0;
        o.data_ = nullptr;
    }
    Buffer& operator=(Buffer&& o) noexcept {                          // move assignment
        if (this != &o) {
            delete[] data_;
            data_ = o.data_;
            n_ = o.n_;
            o.data_ = nullptr;
            o.n_ = 0;
        }
        return *this;
    }
};
```

**Python:**

```python
import copy

a = {"tags": ["x"]}
shallow = copy.copy(a)          # new dict, SAME inner list
deep = copy.deepcopy(a)         # fully independent

shallow["tags"].append("y")
print(a["tags"])                # ['x', 'y']  <- changed through the shallow copy
print(deep["tags"])             # ['x']
```

**JavaScript:**

```js
const a = { tags: ["x"] };
const shallow = { ...a };              // spread / Object.assign: shallow
const deep = structuredClone(a);       // deep (Node 17+, modern browsers); handles nested objects and Dates
shallow.tags.push("y");
console.log(a.tags);                   // ["x", "y"]
```

**C#:**

```csharp
public class Order
{
    public List<string> Tags = new();
    public Order ShallowCopy() => (Order)MemberwiseClone();               // shares the Tags list
    public Order DeepCopy() => new Order { Tags = new List<string>(Tags) };  // own list
}
```

**C:**

```c
#include <string.h>
typedef struct { char *name; } User;

User shallow = *u;                          /* copies the pointer: both share one string */
User deep    = { strdup(u->name) };         /* independent string; free it separately */
```

## Method Overloading

**Overloading** = several methods with the same name but **different parameter types or counts**, chosen at **compile time** from the arguments.

```cpp
int    add(int a, int b)       { return a + b; }
double add(double a, double b) { return a + b; }
// add(1, 2) calls the int version; add(1.5, 2.5) calls the double version
```

C# works the same way. **Python and JavaScript have no overloading**: a later definition replaces the earlier one. Use default parameters, `*args`, runtime type checks, or `functools.singledispatch`:

```python
from functools import singledispatch

@singledispatch
def describe(x): return "something"
@describe.register
def _(x: int): return "an int"
@describe.register
def _(x: str): return "a string"
```

## Method Overriding

**Overriding** = a subclass provides its own version of an inherited method with the **same signature**. The version run is chosen at **runtime** from the object's actual type.

| | Overloading | Overriding |
|---|---|---|
| Where | Same class | Subclass vs parent |
| Signature | Different parameters | Identical |
| Resolved | Compile time | Runtime |
| Purpose | Convenience | Specialized behavior |

C++ and C# add keywords that catch mistakes:

```cpp
struct Base    { virtual void run() const; };
struct Derived : Base {
    void run() const override;       // 'override': compile error if it does not match a base virtual
};
struct Leaf final : Derived {};      // 'final': no further derivation
```

```csharp
public class Base    { public virtual void Run() { } }
public class Derived : Base { public override void Run() { base.Run(); /* then extend */ } }
public sealed class Leaf : Derived { }      // sealed = final
```

Python uses `super().run()`, JavaScript `super.run()`, to call the parent's version.

## Virtual Functions and Dynamic Dispatch

- **Static (early) binding:** the function to call is fixed at compile time from the **declared type**.
- **Dynamic (late) binding:** it is chosen at runtime from the **actual object type**.

In **C++ and C#, methods are non-virtual by default**, which means static binding unless you write `virtual`. Python and JavaScript always dispatch dynamically.

```cpp
#include <iostream>
#include <memory>

struct Base {
    void hi()          { std::cout << "Base\n"; }       // non-virtual: static binding
    virtual void vhi() { std::cout << "Base\n"; }       // virtual: dynamic binding
    virtual ~Base() = default;                          // virtual destructor (see below)
};
struct Derived : Base {
    void hi()  { std::cout << "Derived\n"; }
    void vhi() override { std::cout << "Derived\n"; }
};

int main() {
    std::unique_ptr<Base> p = std::make_unique<Derived>();
    p->hi();     // prints "Base"     (declared type Base decides)
    p->vhi();    // prints "Derived"  (actual type decides)
}
```

Two C++ pitfalls:

1. **Virtual destructor.** If a class has virtual functions and is deleted through a base pointer, its destructor **must be virtual**; otherwise only `Base`'s destructor runs, which is undefined behavior and leaks the derived part.
2. **Object slicing.** `Base b = Derived();` copies only the `Base` part into `b`; polymorphism is lost. Use pointers or references to the base type.

## Abstract Classes and Interfaces

- An **abstract class** cannot be instantiated; it may contain **state and partial implementation** and declares methods that subclasses must implement.
- An **interface** is a pure **contract**: method signatures only (no state). A class can implement many interfaces.

Use an **interface** to describe a *capability* ("can be charged", "can be serialized"); use an **abstract class** to share common code among closely related classes.

**C++** (no `interface` keyword; use a class of only pure virtual functions):

```cpp
class PaymentGateway {                           // interface
public:
    virtual ~PaymentGateway() = default;
    virtual bool charge(long long cents) = 0;
};
```

**C#:**

```csharp
public interface IPaymentGateway { bool Charge(long cents); }

public abstract class Animal                     // abstract class: shared state + partial implementation
{
    public string Name { get; }
    protected Animal(string name) => Name = name;
    public abstract string Sound();
    public string Describe() => $"{Name} says {Sound()}";
}
```

**Python:**

```python
from abc import ABC, abstractmethod
from typing import Protocol

class PaymentGateway(ABC):                       # nominal: subclasses must inherit and implement
    @abstractmethod
    def charge(self, cents: int) -> bool: ...

class Chargeable(Protocol):                      # structural: anything with a matching method qualifies
    def charge(self, cents: int) -> bool: ...
```

**JavaScript** has no abstract keyword or interfaces; it relies on **duck typing** ("if it has `charge()`, it works"). TypeScript adds `interface` and `abstract class` checked at compile time.

**C** builds an interface from a struct of function pointers (next section).

## Multiple Inheritance and the Diamond Problem

A class with **more than one parent**.

```text
        A            D inherits B and C, which both inherit A.
       / \           Does D contain one A or two? Which A::f does d.f() mean?
      B   C
       \ /
        D
```

| Language | Approach |
|---|---|
| **C++** | Allowed. Without `virtual` inheritance, `D` contains **two** `A` subobjects and `d.x` is ambiguous. Declaring `B : virtual A` and `C : virtual A` makes `D` share **one** `A`. |
| **C#** | A class has **one** base class but may implement **many interfaces**, which avoids the problem. |
| **Python** | Allowed. Method lookup order is the **MRO** (computed by C3 linearization). |
| **JavaScript** | One prototype chain only; combine behaviors with **mixins**. |

```cpp
struct A { int x = 0; };
struct B : virtual A {};
struct C : virtual A {};
struct D : B, C {};          // exactly one shared A subobject; d.x is unambiguous
```

```python
class A:
    def who(self): return "A"
class B(A):
    def who(self): return "B"
class C(A):
    def who(self): return "C"
class D(B, C): pass

print([k.__name__ for k in D.__mro__])   # ['D', 'B', 'C', 'A', 'object']
print(D().who())                          # 'B'  (first match in the MRO)
```

Multiple inheritance of **implementation** is a frequent source of confusion. Most designs inherit from at most one class and implement several interfaces.

## vtable and vptr: How Dynamic Dispatch Works

C++ (and C#, Java) implement virtual calls with a per-class table of function pointers.

- **vtable:** one table per class that has virtual functions; entry $i$ holds the address of the most-derived implementation of virtual function $i$.
- **vptr:** a hidden pointer **inside every object** pointing to its class's vtable.

```text
  Circle object              vtable for Circle (one, shared)
 ┌─────────────┐            ┌───────────────────────────┐
 │ vptr  ──────┼──────────► │ [0] &Circle::~Circle      │
 │ r = 1.0     │            │ [1] &Circle::area         │
 └─────────────┘            └───────────────────────────┘

 s->area()   compiles to:   load  vt = s->vptr
                            load  f  = vt[1]
                            call  f(s)            (an indirect call)
```

Consequences:

- **Memory cost:** one pointer per object (8 bytes on 64-bit). `struct A { int x; };` is typically 4 bytes; `struct B { int x; virtual void f(); };` is typically **16 bytes** (8-byte vptr + 4-byte `x` + 4 bytes padding).
- **Time cost:** one extra memory load plus an indirect call, roughly a few nanoseconds, and the compiler usually **cannot inline** the call. Negligible for most code; relevant in tight loops.
- A constructor sets the vptr to the class being constructed, which is why calling a virtual function inside a constructor does **not** reach the derived override.

**The same mechanism written by hand in C** shows there is no magic:

```c
#include <stdlib.h>

typedef struct Shape Shape;

struct ShapeVTable {                         /* the "interface" */
    double (*area)(const Shape *);
    void   (*destroy)(Shape *);
};
struct Shape { const struct ShapeVTable *vt; };   /* every object starts with its vptr */

typedef struct { Shape base; double r; } Circle;  /* "inheritance": base is the FIRST member */

static double circle_area(const Shape *s) {
    const Circle *c = (const Circle *)s;      /* valid: a pointer to a struct equals a pointer to its first member */
    return 3.141592653589793 * c->r * c->r;
}
static void circle_destroy(Shape *s) { free(s); }
static const struct ShapeVTable circle_vt = { circle_area, circle_destroy };

Shape *circle_new(double r) {
    Circle *c = malloc(sizeof *c);
    c->base.vt = &circle_vt;                  /* set the vptr */
    c->r = r;
    return &c->base;
}

/* usage:
   Shape *s = circle_new(1.0);
   double a = s->vt->area(s);                 // "virtual call"
   s->vt->destroy(s);
*/
```

**Python and JavaScript** achieve the same effect with **dictionary / prototype lookup at runtime**: `obj.method()` searches the object, then its class (Python: along the MRO; JavaScript: along the prototype chain). It is more flexible and slower than a vtable.

---

# Key Takeaways

- **Encapsulation** protects invariants, **abstraction** hides complexity, **inheritance** models is-a and reuses code, **polymorphism** lets one call work on many types.
- Relationships from weakest to strongest coupling: **dependency, association, aggregation, composition, inheritance**. Prefer the weakest that works.
- **Copying:** shallow copies share nested objects; deep copies do not. In C++, classes owning resources need the Rule of Five (or follow the Rule of Zero with standard containers and smart pointers).
- **Overloading** is compile-time (same name, different parameters); **overriding** is runtime (same signature in a subclass). Use `override` / `final` in C++ and C#.
- C++ and C# methods are **non-virtual by default**; use `virtual` for polymorphism and give polymorphic C++ base classes a **virtual destructor**.
- Use **interfaces** for capabilities and **abstract classes** for shared implementation. Avoid inheriting implementation from multiple parents (diamond problem).
- Virtual dispatch = **vptr → vtable → function pointer**: one extra indirection and 8 bytes per object.
- Cleanup differs by language: C++ destructors (RAII), C# `IDisposable`/`using`, Python `with`, JavaScript `try/finally`, C manual `free`.
