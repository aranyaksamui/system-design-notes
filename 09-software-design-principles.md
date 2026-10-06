# 09 - Software Design Principles

Design patterns (doc 10) are specific solutions. **Design principles** are the reasons they work. They are **heuristics**, not laws: each trades something (simplicity, speed, flexibility) for something else, and applying them without judgement produces over-engineered code.

The goal behind all of them is the same: code that is **easy to change**, **easy to test** and **hard to break by accident**. The two measurable ideas underneath are **coupling** (how much modules depend on each other) and **cohesion** (how well the contents of one module belong together). Section 9.9 makes them precise.

Depends on: doc 08 (classes, interfaces, polymorphism, relationships).

---

# 9.1 DRY: Don't Repeat Yourself

> Every piece of **knowledge** should have a single, authoritative representation in the system.

DRY is about duplicated **knowledge** (a business rule, a format, a constant), not about identical-looking lines. When the rule changes, you should have to change it in **one** place.

**Violation:** the same email rule in two places; one gets fixed, the other does not.

```js
// signup.js
if (!/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(email)) throw new Error("invalid email");
// profile.js
if (!/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(email)) throw new Error("invalid email");
```

**Fix:** one definition.

```js
// validators.js
const EMAIL_RE = /^[^@\s]+@[^@\s]+\.[^@\s]+$/;
function assertValidEmail(email) {
  if (!EMAIL_RE.test(email)) throw new Error("invalid email");
}
module.exports = { assertValidEmail };
```

**Caveats**

- **Rule of three:** tolerate duplication twice; abstract on the third occurrence, when the real shared shape is visible.
- **The wrong abstraction is worse than duplication.** Two pieces of code that look alike but change for different reasons should stay separate; merging them couples unrelated things (and leads to functions full of boolean flags).
- Across services, sharing code creates deployment coupling; sometimes duplication is the right call in microservices (doc 31).

---

# 9.2 KISS: Keep It Simple

> Prefer the simplest design that solves the actual problem.

Complexity is measured, not just felt. **Cyclomatic complexity** counts independent paths through a function. For a control-flow graph with $E$ edges, $N$ nodes and $P$ connected components:

$$M=E-N+2P$$

For a single function this equals the number of decision points (`if`, `for`, `while`, `case`, `&&`, `||`, `catch`) plus one. A function with 3 `if`s has $M=4$, and needs at least 4 test cases to cover its paths. A common guideline is to keep $M\le10$.

**Over-engineered:**

```python
class AbstractNumberAdderFactory: ...
class NumberAdderStrategy: ...
result = AbstractNumberAdderFactory().create().get_strategy().execute(a, b)
```

**Simple:**

```python
result = a + b
```

Techniques: small functions, early returns instead of nested `if`s, clear names, standard library over custom machinery, and deleting code.

```python
def shipping_cost(order):
    if order.total >= 5000:
        return 0                      # early return: no nesting
    if order.country != "IN":
        return 1500
    return 500
```

---

# 9.3 YAGNI: You Aren't Gonna Need It

> Do not build functionality until it is actually needed.

Speculative features cost time to write, test and maintain, add surface for bugs, and are usually guessed wrong. The cost of building feature $f$ now is certain; the probability it is needed later is often low.

- Do not add plugin systems, config flags, or extra database columns "just in case".
- Do keep the design **easy to extend** (good boundaries and tests), because that makes adding the feature later cheap.

YAGNI does not mean "ignore design". It means do not **pay now** for needs that may never arrive. Decisions that are **expensive to reverse** (public API shapes, data models, security boundaries) deserve more up-front thought than internal code that can be refactored freely.

---

# 9.4 Separation of Concerns (SoC)

> Split a program into parts, each addressing one distinct concern.

A **concern** is an aspect of the system: handling HTTP, business rules, persistence, formatting, logging, authentication. When concerns mix, changing one risks breaking the others.

**Violation: one function does everything.**

```js
app.post("/orders", async (req, res) => {
  const { userId, items } = JSON.parse(req.body);                        // HTTP parsing
  if (!items.length) return res.status(400).send("empty");               // validation
  const total = items.reduce((s, i) => s + i.price * i.qty, 0);          // business rule
  const discount = total > 10000 ? total * 0.1 : 0;                      // business rule
  await db.query("INSERT INTO orders ...", [userId, total - discount]);  // persistence
  res.status(201).json({ total: total - discount });                    // presentation
});
```

**Separated** (the layering reappears in doc 19):

```js
// controller: HTTP in, HTTP out
app.post("/orders", async (req, res) => {
  const order = await orderService.place(req.body.userId, req.body.items);
  res.status(201).json(order);
});

// service: business rules, no HTTP and no SQL
class OrderService {
  constructor(orderRepo) { this.orderRepo = orderRepo; }
  async place(userId, items) {
    if (!items.length) throw new ValidationError("empty order");
    const total = items.reduce((s, i) => s + i.price * i.qty, 0);
    const discount = total > 10000 ? total * 0.1 : 0;
    return this.orderRepo.save({ userId, total: total - discount });
  }
}

// repository: persistence only
class OrderRepository {
  save(order) { return db.query("INSERT INTO orders ...", [order.userId, order.total]); }
}
```

Now the business rule can be tested without a web server or database, and the database can change without touching the rule.

---

# 9.5 SOLID

Five principles for class and module design, named by Robert C. Martin.

| Letter | Principle | One line |
| --- | --- | --- |
| **S** | Single Responsibility | A class has one reason to change |
| **O** | Open/Closed | Open for extension, closed for modification |
| **L** | Liskov Substitution | Subtypes must be usable wherever the base type is |
| **I** | Interface Segregation | Many small interfaces beat one fat one |
| **D** | Dependency Inversion | Depend on abstractions, not concretions |

## S: Single Responsibility Principle

> A class should have only **one reason to change**: it serves one actor/concern.

**Violation:** `Report` changes when the calculation changes, when the HTML layout changes, and when the storage location changes.

```python
class Report:
    def __init__(self, rows): self.rows = rows
    def total(self): return sum(r["amount"] for r in self.rows)           # calculation
    def to_html(self): return f"<h1>Total: {self.total()}</h1>"           # presentation
    def save(self, path): open(path, "w").write(self.to_html())           # storage
```

**Fix:** one class per reason to change.

```python
class ReportCalculator:
    def total(self, rows): return sum(r["amount"] for r in rows)

class HtmlReportFormatter:
    def render(self, total): return f"<h1>Total: {total}</h1>"

class FileStorage:
    def save(self, path, content):
        with open(path, "w") as f:
            f.write(content)
```

SRP does **not** mean "one method per class". It means the things inside a class change **together and for the same reason**.

## O: Open/Closed Principle

> Software entities should be **open for extension** but **closed for modification**: add new behavior by adding new code, not by editing tested code.

**Violation:** every new customer type edits the same function (and risks the old branches).

```python
def discount(order):
    if order.kind == "regular":  return 0
    elif order.kind == "premium": return order.total * 0.10
    elif order.kind == "vip":     return order.total * 0.20
    # a new kind means editing this function again
```

**Fix:** polymorphism. A new rule is a new class.

```python
from abc import ABC, abstractmethod

class DiscountPolicy(ABC):
    @abstractmethod
    def discount(self, total: float) -> float: ...

class NoDiscount(DiscountPolicy):
    def discount(self, total): return 0
class PremiumDiscount(DiscountPolicy):
    def discount(self, total): return total * 0.10
class VipDiscount(DiscountPolicy):
    def discount(self, total): return total * 0.20

def price(total: float, policy: DiscountPolicy) -> float:
    return total - policy.discount(total)           # never changes when a policy is added
```

The same in C++ with a virtual interface:

```cpp
struct DiscountPolicy {
    virtual ~DiscountPolicy() = default;
    virtual double discount(double total) const = 0;
};
struct VipDiscount : DiscountPolicy {
    double discount(double total) const override { return total * 0.20; }
};
double price(double total, const DiscountPolicy& p) { return total - p.discount(total); }
```

This is the **Strategy** pattern (doc 10). Do not pre-build extension points for changes you cannot foresee (YAGNI); introduce the abstraction when the second or third variation appears.

## L: Liskov Substitution Principle

> If $S$ is a subtype of $T$, objects of type $T$ can be replaced with objects of type $S$ **without breaking the correctness** of the program.

A subclass must honor the **contract** of its parent:

| Rule | Meaning |
| --- | --- |
| Preconditions cannot be strengthened | The subclass must not demand more from callers |
| Postconditions cannot be weakened | The subclass must still deliver everything the parent promised |
| Invariants are preserved | The parent's guarantees still hold |
| No new unexpected exceptions | Callers handle what the parent could throw |

**Classic violation:** a `Square` that "is a" `Rectangle` in geometry, but not in behavior.

```python
class Rectangle:
    def __init__(self, w, h): self._w, self._h = w, h
    def set_width(self, w):  self._w = w
    def set_height(self, h): self._h = h
    def area(self): return self._w * self._h

class Square(Rectangle):
    def set_width(self, w):  self._w = self._h = w      # keeps it square...
    def set_height(self, h): self._w = self._h = h      # ...but breaks the Rectangle contract

def stretch(r: Rectangle):
    r.set_width(5)
    r.set_height(2)
    assert r.area() == 10        # true for Rectangle; for Square the area is 4
```

`stretch` works for `Rectangle` and fails for `Square`: the subtype is **not substitutable**. Fixes: make shapes immutable, or give them a common `Shape` base with `area()` and no setters, rather than forcing `Square` to inherit `Rectangle`.

Other smells: an override that throws `NotImplementedError`, or one that silently does nothing, or code that checks `if isinstance(x, Special)` before calling a method.

## I: Interface Segregation Principle

> Clients should not be forced to depend on methods they do not use.

**Violation:** a fat interface forces simple devices to implement operations they cannot do.

```csharp
public interface IMultiFunctionDevice
{
    void Print(Document d);
    void Scan(Document d);
    void Fax(Document d);
}
public class BasicPrinter : IMultiFunctionDevice
{
    public void Print(Document d) { /* ok */ }
    public void Scan(Document d)  { throw new NotSupportedException(); }   // forced to fake it
    public void Fax(Document d)   { throw new NotSupportedException(); }
}
```

**Fix:** small, role-based interfaces.

```csharp
public interface IPrinter { void Print(Document d); }
public interface IScanner { void Scan(Document d); }
public interface IFax     { void Fax(Document d); }

public class BasicPrinter : IPrinter { public void Print(Document d) { /* ok */ } }
public class OfficeMachine : IPrinter, IScanner, IFax { /* implements all three */ }
```

Benefits: fewer reasons for a change in one method to ripple to unrelated implementers, and easier fakes in tests. In Go and TypeScript, small interfaces defined **by the consumer** are idiomatic.

## D: Dependency Inversion Principle

> 1. High-level modules should not depend on low-level modules. **Both** should depend on abstractions.
> 2. Abstractions should not depend on details; details should depend on abstractions.

**Without inversion:** business logic is welded to one vendor.

```text
 OrderService ───────────► StripeClient          (policy depends on detail)
```

**With inversion:**

```text
 OrderService ──► PaymentGateway (interface) ◄── StripeGateway   (both depend on the abstraction)
                                             ◄── FakeGateway     (tests)
```

The *interface is owned by the high-level code* (it states what the business needs), so the dependency arrow of the low-level code points **toward** the abstraction: it is "inverted".

**C++:**

```cpp
class PaymentGateway {
public:
    virtual ~PaymentGateway() = default;
    virtual bool charge(long long cents) = 0;
};

class StripeGateway : public PaymentGateway {
public:
    bool charge(long long cents) override { /* call Stripe's API */ return true; }
};
class FakeGateway : public PaymentGateway {                   // for tests: no network
public:
    bool charge(long long) override { return true; }
};

class OrderService {
    PaymentGateway& gateway_;                                 // depends on the abstraction
public:
    explicit OrderService(PaymentGateway& g) : gateway_(g) {} // dependency injected
    bool checkout(long long cents) { return gateway_.charge(cents); }
};
```

**C#:**

```csharp
public interface IPaymentGateway { bool Charge(long cents); }

public class OrderService
{
    private readonly IPaymentGateway _gateway;
    public OrderService(IPaymentGateway gateway) => _gateway = gateway;   // constructor injection
    public bool Checkout(long cents) => _gateway.Charge(cents);
}

// ASP.NET Core wiring (composition root):
// builder.Services.AddScoped<IPaymentGateway, StripeGateway>();
```

**Python / JavaScript** use duck typing: the "abstraction" is just the expected method set.

```python
class OrderService:
    def __init__(self, gateway):                  # anything with .charge(cents)
        self.gateway = gateway
    def checkout(self, cents):
        return self.gateway.charge(cents)

class FakeGateway:
    def __init__(self): self.charged = []
    def charge(self, cents): self.charged.append(cents); return True

def test_checkout_charges_gateway():
    fake = FakeGateway()
    assert OrderService(fake).checkout(500)
    assert fake.charged == [500]
```

```js
class OrderService {
  constructor(gateway) { this.gateway = gateway; }
  checkout(cents) { return this.gateway.charge(cents); }
}
```

**Three related terms, often confused:**

| Term | Meaning |
| --- | --- |
| **Dependency Inversion (DIP)** | The *principle*: depend on abstractions |
| **Dependency Injection (DI)** | A *technique*: pass dependencies in (constructor, setter) instead of creating them inside |
| **IoC container** | A *tool* that builds object graphs and injects dependencies automatically (ASP.NET Core DI, Spring, NestJS) |

---

# 9.6 Composition over Inheritance

> Prefer building behavior by **combining objects** (has-a) over **inheriting** it (is-a).

Inheritance is the **strongest coupling** (doc 08): a subclass depends on its parent's implementation, and parent changes can silently break it (the **fragile base class** problem). It is also fixed at compile time and limited to one dimension of variation.

**Combinatorial explosion.** Suppose a repository can optionally add logging, caching and retries. With inheritance each combination is a class: $n$ independent features can require up to $2^{n}$ classes. With composition each feature is **one** wrapper that can be stacked: $n$ classes.

```python
from typing import Protocol

class UserRepository(Protocol):
    def get(self, user_id: int) -> dict: ...

class SqlUserRepository:
    def get(self, user_id): return {"id": user_id}          # pretend: SELECT ...

class CachedUserRepository:
    def __init__(self, inner: UserRepository, cache: dict):
        self.inner, self.cache = inner, cache
    def get(self, user_id):
        if user_id not in self.cache:
            self.cache[user_id] = self.inner.get(user_id)
        return self.cache[user_id]

class LoggedUserRepository:
    def __init__(self, inner: UserRepository):
        self.inner = inner
    def get(self, user_id):
        print(f"get({user_id})")
        return self.inner.get(user_id)

repo = LoggedUserRepository(CachedUserRepository(SqlUserRepository(), {}))   # stack at runtime
```

Use inheritance when the relationship is a true **is-a** that satisfies LSP and you want polymorphism; otherwise compose.

---

# 9.7 Program to an Interface, Not an Implementation

> Depend on the **minimum contract** you need, not on a concrete type.

If a function only iterates, accept something iterable, not a specific container. Callers stay free to pass anything compatible, and you can change the implementation without touching callers.

```csharp
public double Average(IEnumerable<double> values) { ... }     // not List<double>
```

```python
from typing import Iterable
def average(values: Iterable[float]) -> float:
    items = list(values)
    return sum(items) / len(items)
```

```cpp
#include <span>                                  // C++20: a view over any contiguous sequence
double average(std::span<const double> values);  // accepts vector, array, C array
```

At component scale this is the same idea as DIP: callers know the **interface** (`IPaymentGateway`), never the class behind it.

---

# 9.8 Encapsulation

Encapsulation (doc 08) as a design principle means: **keep data and the rules for changing it together, and hide the representation.**

**Tell, don't ask.** Do not pull an object's data out to make decisions for it; tell the object what to do.

```js
// Ask (logic leaks out; every caller must repeat the rule)
if (order.status === "PAID" && order.items.length > 0) { order.status = "SHIPPED"; }

// Tell (the object enforces its own rules)
order.ship();     // throws if the order cannot be shipped
```

**Law of Demeter** (principle of least knowledge): a method should talk only to its **immediate collaborators**: itself, its parameters, objects it creates, its own fields. Long chains expose the internal structure of other objects.

```js
customer.getWallet().getCard().charge(amount);   // knows how Customer stores payment details
customer.pay(amount);                            // knows only Customer
```

Do not mechanically add getters and setters for every field; that exposes the data and defeats encapsulation. Expose **behavior**, keep fields private, and maintain invariants in the constructor and mutators.

---

# 9.9 Loose Coupling and High Cohesion

## Coupling: how much modules depend on each other

**Loose coupling** means a change in one module rarely forces changes in another. From worst to best:

| Coupling type | Description |
| --- | --- |
| **Content** | One module reaches into another's internals (worst) |
| **Common** | Modules share global state |
| **Control** | One passes a flag that steers the other's logic |
| **Stamp** | Passing a whole data structure when only a field is needed |
| **Data** | Passing only the needed values (good) |
| **Message** | Communicating through events or interfaces (best) |

Ways to loosen: interfaces (DIP), dependency injection, events and queues (doc 25, 26), passing only what is needed.

## Cohesion: how well one module's parts belong together

**High cohesion** means everything in a module serves one clear purpose.

| Cohesion type | Example | Quality |
| --- | --- | --- |
| **Functional** | `PasswordHasher`: only hashing | Best |
| Sequential / communicational | Steps on the same data | Good |
| Temporal | `initEverythingAtStartup()` | Weak |
| Logical | `Utils` with unrelated helpers chosen by "type" | Poor |
| Coincidental | Random leftovers | Worst |

A module named `Utils`, `Helpers` or `Manager` with many unrelated functions is the usual sign of low cohesion.

## Measuring coupling: afferent, efferent, instability

For a module $m$:

- **Afferent coupling** $C_a$: number of modules that **depend on** $m$ (incoming).
- **Efferent coupling** $C_e$: number of modules $m$ **depends on** (outgoing).

$$I=\frac{C_e}{C_a+C_e}\qquad(0\le I\le1)$$

$I=0$ is **maximally stable** (many depend on it, it depends on nothing, so it is hard to change). $I=1$ is **maximally unstable** (nothing depends on it, it depends on everything, so it is easy to change).

Example: module `billing` is used by 3 modules ($C_a=3$) and uses 1 ($C_e=1$): $I=1/4=0.25$ (stable). A `report-generator` used by none and using 4: $I=1$ (unstable, free to change).

**Stable Dependencies Principle:** depend in the direction of stability, that is, toward modules with lower $I$. Stable modules should be **abstract** (interfaces), so their stability does not block extension. This is DIP expressed as a metric, and it is why domain code (stable, abstract) must not depend on frameworks and databases (unstable details), the core idea of clean and hexagonal architecture (doc 19).

---

# 9.10 Using the Principles Wisely

| Symptom | Principle to reach for |
| --- | --- |
| Same rule copy-pasted in several places | DRY |
| A change in one place breaks unrelated features | Separation of concerns, low coupling |
| A class with "and" in its description | SRP |
| `if/else` chain grows with every new type | OCP (polymorphism / Strategy) |
| Subclass throws "not supported" or special-cased by callers | LSP (rethink the hierarchy) |
| Classes implement methods they cannot support | ISP |
| Cannot test without a database or network | DIP, dependency injection |
| Inheritance tree keeps growing in several directions | Composition over inheritance |
| `a.b().c().d()` chains | Law of Demeter, tell-don't-ask |
| Speculative features and unused hooks | YAGNI, KISS |

Principles conflict. DRY can couple things that should vary independently; OCP and DIP add indirection that costs readability (KISS); YAGNI says wait while "design for change" says prepare. The practical rule: **start simple, keep boundaries clean, and add abstraction when change actually shows up (typically at the second or third variation).**

---

# Key Takeaways

- **DRY:** one source of truth for each piece of knowledge, but do not merge code that only looks similar. **KISS / YAGNI:** the simplest thing that works; build only what is needed now.
- **Separation of concerns:** split HTTP, business rules, persistence and presentation; each can then change and be tested alone.
- **SOLID:** one reason to change (SRP); extend by adding code (OCP); subtypes honor the parent's contract (LSP); small interfaces (ISP); depend on abstractions owned by the high-level code (DIP).
- **Composition over inheritance:** wrappers scale as $n$ classes, inheritance of combinations as up to $2^n$.
- **Program to interfaces**, tell objects what to do instead of asking for their data, and follow the Law of Demeter.
- Aim for **loose coupling** and **high cohesion**; instability $I=\dfrac{C_e}{C_a+C_e}$ tells you which direction dependencies should point (toward stable abstractions).
- Principles are trade-offs: apply them where change is real, not everywhere.
