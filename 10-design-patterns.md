# 10 - Design Patterns

A **design pattern** is a named, reusable solution to a **recurring design problem** in a particular context. It is not a library or code to copy; it is a **template for structuring classes and objects** that experienced engineers have found to work. The value is as much in the **shared vocabulary** ("use a Decorator here") as in the solution.

The classic catalogue is **Design Patterns: Elements of Reusable Object-Oriented Software** (Gamma, Helm, Johnson, Vlissides, 1994, the "Gang of Four" or **GoF**), which describes 23 patterns in three categories:

| Category | Question it answers | Patterns covered here |
| --- | --- | --- |
| **Creational** | How do I create objects without hard-wiring concrete classes? | Singleton, Factory, Abstract Factory, Builder, Prototype |
| **Structural** | How do I compose classes and objects into larger structures? | Adapter, Decorator, Facade, Proxy, Composite, Bridge |
| **Behavioral** | How do objects communicate and share responsibility? | Strategy, Observer, Command, State, Template Method, Chain of Responsibility, Iterator |
| **Backend-relevant** (not GoF) | How do I structure a server application? | Repository, Service Layer, Dependency Injection, Unit of Work, MVC, Middleware, Event-driven architecture |

Depends on: doc 08 (interfaces, polymorphism, composition) and doc 09 (SOLID, coupling and cohesion). Almost every pattern below is an application of "program to an interface" (9.7), "composition over inheritance" (9.6), or the Open/Closed and Dependency Inversion principles (9.5).

## How to Read Each Pattern

Every pattern is described as: **Intent** (one line), **Problem**, **Solution**, **Structure** (diagram), **Code**, **Use when**, **Trade-offs**.

## Patterns Are a Means, Not a Goal

- Use a pattern when you feel the **specific pain** it solves. Adding patterns "in case" violates YAGNI (9.3) and KISS (9.2).
- Language features can absorb patterns. In languages with **first-class functions**, Strategy, Command, Template Method and Observer often shrink to "pass a function". Python and JavaScript modules replace most Singleton classes. C# has events built in (Observer). Every mainstream language has built-in iteration (Iterator).
- The same structure can serve different **intents** (Strategy and State look alike). The intent is what distinguishes patterns.

> **Code in this document:** JavaScript (Node.js) and Python in most examples; C++17 and C# where language mechanics matter (ownership, virtual dispatch, events); C for a few patterns where it exposes the mechanism.

---

# 10.1 Creational Patterns

Creational patterns control **how and where objects are created**, so the rest of the code does not depend on concrete classes (Dependency Inversion, 9.5).

---

## 10.1.1 Singleton

**Intent:** guarantee that a class has **exactly one instance** and provide a global access point to it.

**Problem:** some things should exist once per process: a configuration object, a connection pool, a logger, an in-memory cache.

**Structure:**

```text
 ┌───────────────────────────┐
 │ Singleton                 │
 │ - instance: Singleton     │  ← one static field
 │ - Singleton()             │  ← private constructor: nobody else can create one
 │ + getInstance(): Singleton│  ← creates it on first use, then always returns it
 └───────────────────────────┘
```

**JavaScript (Node.js).** A module is evaluated once and cached, so exporting an instance is already a singleton:

```js
// config.js
class Config {
  constructor() {
    this.values = { port: Number(process.env.PORT ?? 3000) };
  }
  get(key) { return this.values[key]; }
}
module.exports = new Config();

// anywhere else:  const config = require("./config");   // always the same object
```

(Node caches by resolved file path. Two copies of the same package installed in different `node_modules` folders are two modules, so two instances.)

**Python.** A module-level object is the simplest singleton. An explicit class version:

```python
class Config:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.values = {"port": 3000}
        return cls._instance

assert Config() is Config()          # same object every time
print(Config().values)
```

**C++ (Meyers singleton).** A function-local `static` is initialized exactly once, and since C++11 that initialization is **thread-safe**:

```cpp
#include <iostream>
#include <string>

class Logger {
public:
    static Logger& instance() {
        static Logger inst;                       // created on first call; thread-safe initialization
        return inst;
    }
    Logger(const Logger&) = delete;               // no copies
    Logger& operator=(const Logger&) = delete;

    void log(const std::string& msg) { std::cout << msg << "\n"; }   // NOT synchronized: add a mutex if threads share it
private:
    Logger() = default;                           // private constructor
};

int main() {
    Logger::instance().log("started");
    return &Logger::instance() == &Logger::instance() ? 0 : 1;      // always the same address
}
```

**C#.** `Lazy<T>` gives thread-safe lazy creation:

```csharp
public sealed class Config
{
    private static readonly Lazy<Config> _instance = new(() => new Config());
    public static Config Instance => _instance.Value;
    private Config() { }
    public int Port { get; } = 3000;
}
```

In ASP.NET Core you normally let the DI container manage this: `services.AddSingleton<IConfig, Config>();` (see Dependency Injection, 10.4.3).

**C.** A file-scope `static` plus an accessor:

```c
typedef struct { int level; } Logger;
static Logger g_logger = { 1 };                  /* exactly one instance, lives for the whole program */
Logger *logger_get(void) { return &g_logger; }
```

**Use when:** one instance is genuinely required by the problem (a single hardware resource, a process-wide pool).

**Trade-offs and pitfalls:**

| Problem | Why it hurts |
| --- | --- |
| **Global state** | Any code can change it; behavior becomes hard to trace |
| **Hidden dependency** | Classes call `Config.instance()` internally, so their dependencies are not visible in constructors (violates DIP) |
| **Hard to test** | State leaks between tests; you cannot substitute a fake |
| **Thread safety** | Creation must be safe; the instance's own methods may also need locking |
| **"Singleton" is per process** | With 10 server instances there are 10 "singletons". A counter that must be globally unique or a rate limit shared by all servers must live in a **shared store** (Redis, a database), not in a singleton object (docs 21, 24) |

**Better default:** create the object **once** at startup (the composition root) and **inject** it where needed. You keep "one instance" without global access.

---

## 10.1.2 Factory (Simple Factory and Factory Method)

**Intent:** move the decision of *which concrete class to instantiate* out of the calling code.

**Problem:** `new EmailNotifier()` scattered through the code ties every caller to one concrete class. Adding `SmsNotifier` means editing every place that decides.

### Simple Factory (an idiom, not a GoF pattern)

One function chooses and creates the object:

```python
from abc import ABC, abstractmethod

class Notifier(ABC):
    @abstractmethod
    def send(self, to: str, message: str) -> str: ...

class EmailNotifier(Notifier):
    def send(self, to, message): return f"email to {to}: {message}"

class SmsNotifier(Notifier):
    def send(self, to, message): return f"sms to {to}: {message}"

_REGISTRY = {"email": EmailNotifier, "sms": SmsNotifier}

def create_notifier(kind: str) -> Notifier:
    try:
        return _REGISTRY[kind]()
    except KeyError:
        raise ValueError(f"unknown notifier kind: {kind}") from None

print(create_notifier("sms").send("user-1", "your OTP is 4821"))
```

Callers depend only on `Notifier`. A **registry** (a dictionary from name to class) lets you add a new kind without editing the factory function, which respects the Open/Closed Principle (9.5).

```js
// JavaScript: same idea, with plain functions as creators
const creators = {
  email: () => ({ send: (to, msg) => `email to ${to}: ${msg}` }),
  sms:   () => ({ send: (to, msg) => `sms to ${to}: ${msg}` }),
};
function createNotifier(kind) {
  const create = creators[kind];
  if (!create) throw new Error(`unknown notifier kind: ${kind}`);
  return create();
}
console.log(createNotifier("email").send("user-1", "welcome"));
```

### Factory Method (GoF)

**Intent:** define an interface for creating an object, but let **subclasses** decide which class to instantiate.

```text
        Exporter (creator)                      Document (product)
   ┌──────────────────────────┐              ┌────────────────┐
   │ + create(): Document  ◄──┼─ factory     │ + render()     │
   │ + exportReport()         │   method     └───────▲────────┘
   │     doc = create()       │                  ┌───┴─────┐
   │     return doc.render()  │             PdfDocument  HtmlDocument
   └────────────▲─────────────┘
          ┌─────┴──────┐
     PdfExporter   HtmlExporter     ← each overrides create()
```

The creator's own logic (`exportReport`) works with whatever product `create()` returns, never with a concrete class.

```cpp
#include <iostream>
#include <memory>
#include <string>

struct Document {
    virtual ~Document() = default;
    virtual std::string render() const = 0;
};
struct PdfDocument : Document {
    std::string render() const override { return "PDF"; }
};
struct HtmlDocument : Document {
    std::string render() const override { return "HTML"; }
};

class Exporter {                                              // the creator
public:
    virtual ~Exporter() = default;
    virtual std::unique_ptr<Document> create() const = 0;     // the factory method
    std::string exportReport() const {
        auto doc = create();                                  // uses the product through its interface
        return "exported as " + doc->render();
    }
};
struct PdfExporter : Exporter {
    std::unique_ptr<Document> create() const override { return std::make_unique<PdfDocument>(); }
};
struct HtmlExporter : Exporter {
    std::unique_ptr<Document> create() const override { return std::make_unique<HtmlDocument>(); }
};

int main() {
    PdfExporter pdf;
    HtmlExporter html;
    std::cout << pdf.exportReport() << "\n" << html.exportReport() << "\n";
}
```

Returning `std::unique_ptr` makes ownership explicit: the caller owns the new object and it is freed automatically.

```csharp
// C#: a switch expression is the idiomatic simple factory
public static INotifier Create(string kind) => kind switch
{
    "email" => new EmailNotifier(),
    "sms"   => new SmsNotifier(),
    _       => throw new ArgumentException($"unknown notifier kind: {kind}")
};
```

**Use when:** the exact class depends on configuration or input (database driver, payment provider, file format), or a framework must let users plug in their own product types.

**Trade-offs:** an extra layer of indirection. If you only ever have one concrete class, a factory is pointless (YAGNI).

---

## 10.1.3 Abstract Factory

**Intent:** create **families of related objects** without specifying their concrete classes, and ensure the objects of one family are used together.

**Problem:** your app needs a storage service **and** a queue service. In production both come from AWS; in local development both are local fakes. Mixing an S3 storage with an in-memory queue by accident would be a bug. An Abstract Factory returns a **consistent set**.

```text
            InfraFactory (abstract)
         ┌─ storage(): Storage ─┐
         └─ queue():   Queue   ─┘
              ▲              ▲
        AwsFactory      LocalFactory
        S3Storage       LocalStorage
        SqsQueue        InMemoryQueue
```

```python
from abc import ABC, abstractmethod

class Storage(ABC):
    @abstractmethod
    def put(self, key: str, data: bytes) -> str: ...

class Queue(ABC):
    @abstractmethod
    def push(self, msg: str) -> str: ...

class S3Storage(Storage):
    def put(self, key, data): return f"s3://bucket/{key}"
class SqsQueue(Queue):
    def push(self, msg): return f"sqs <- {msg}"

class LocalStorage(Storage):
    def put(self, key, data): return f"/tmp/{key}"
class InMemoryQueue(Queue):
    def __init__(self): self.items = []
    def push(self, msg):
        self.items.append(msg)
        return f"memory <- {msg}"

class InfraFactory(ABC):
    @abstractmethod
    def storage(self) -> Storage: ...
    @abstractmethod
    def queue(self) -> Queue: ...

class AwsFactory(InfraFactory):
    def storage(self): return S3Storage()
    def queue(self): return SqsQueue()

class LocalFactory(InfraFactory):
    def storage(self): return LocalStorage()
    def queue(self): return InMemoryQueue()

def run(factory: InfraFactory):                      # application code: no concrete class anywhere
    print(factory.storage().put("report.txt", b"data"))
    print(factory.queue().push("job-1"))

run(LocalFactory())      # development
run(AwsFactory())        # production
```

| | Factory Method | Abstract Factory |
| --- | --- | --- |
| Creates | **One** product | A **family** of products |
| Mechanism | Inheritance (subclass overrides the method) | Composition (you pass a factory object) |
| Typical use | Pick a class | Switch a whole set (UI theme, cloud vendor, database dialect) |

**Trade-offs:** adding a **new product type** (say `Cache`) means changing the abstract factory and **every** concrete factory. Adding a new family is easy; adding a new product kind is not.

---

## 10.1.4 Builder

**Intent:** construct a complex object **step by step**, separating construction from representation, so the same process can build different results and optional parts do not need huge constructors.

**Problem: the telescoping constructor.** With $n$ optional parameters, offering a constructor for every useful combination needs up to $2^{n}$ overloads, and calls like `new Server("h", 8080, true, false, null, 30, true)` are unreadable. A builder names every step.

```text
 client ──► builder.host("a").port(8080).tls(true) ──► build() ──► finished, validated object
```

**JavaScript: a query builder** (fluent interface: each method returns `this`):

```js
class QueryBuilder {
  #table = null;
  #conditions = [];
  #params = [];
  #order = null;
  #limit = null;

  from(table) { this.#table = table; return this; }

  where(column, value) {
    this.#params.push(value);
    this.#conditions.push(`${column} = $${this.#params.length}`);   // $1, $2 ... placeholders
    return this;
  }
  orderBy(column, direction = "ASC") {
    if (!["ASC", "DESC"].includes(direction)) throw new Error("bad direction");
    this.#order = `${column} ${direction}`;
    return this;
  }
  limit(n) {
    if (!Number.isInteger(n) || n <= 0) throw new Error("bad limit");
    this.#limit = n;
    return this;
  }
  build() {
    if (!this.#table) throw new Error("table is required");     // validation happens once, here
    let sql = `SELECT * FROM ${this.#table}`;
    if (this.#conditions.length) sql += ` WHERE ${this.#conditions.join(" AND ")}`;
    if (this.#order) sql += ` ORDER BY ${this.#order}`;
    if (this.#limit) sql += ` LIMIT ${this.#limit}`;
    return { sql, params: [...this.#params] };
  }
}

const query = new QueryBuilder()
  .from("users")
  .where("country", "IN")
  .where("active", true)
  .orderBy("created_at", "DESC")
  .limit(10)
  .build();
console.log(query);
// { sql: 'SELECT * FROM users WHERE country = $1 AND active = $2 ORDER BY created_at DESC LIMIT 10',
//   params: [ 'IN', true ] }
```

Security note: **values** go into `params` (parameterized, so safe from SQL injection), but **table and column names** are pasted into the SQL text. They must come from your code or a whitelist, never from user input (doc 18).

**C#: an immutable object with a builder:**

```csharp
public sealed class ServerOptions
{
    public string Host { get; }
    public int Port { get; }
    public bool UseTls { get; }
    public TimeSpan Timeout { get; }

    internal ServerOptions(string host, int port, bool useTls, TimeSpan timeout)
        => (Host, Port, UseTls, Timeout) = (host, port, useTls, timeout);
}

public sealed class ServerOptionsBuilder
{
    private string _host = "localhost";
    private int _port = 8080;
    private bool _tls;
    private TimeSpan _timeout = TimeSpan.FromSeconds(30);

    public ServerOptionsBuilder Host(string host) { _host = host; return this; }
    public ServerOptionsBuilder Port(int port) { _port = port; return this; }
    public ServerOptionsBuilder UseTls() { _tls = true; return this; }
    public ServerOptionsBuilder Timeout(TimeSpan t) { _timeout = t; return this; }

    public ServerOptions Build()
    {
        if (_port is < 1 or > 65535) throw new InvalidOperationException("port out of range");
        return new ServerOptions(_host, _port, _tls, _timeout);   // the product is immutable once built
    }
}

// var opts = new ServerOptionsBuilder().Host("api.example.com").Port(443).UseTls().Build();
```

**C++:** the same fluent style returns `*this` by reference:

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

struct ServerOptions { std::string host; int port; bool tls; };

class ServerOptionsBuilder {
    std::string host_ = "localhost";
    int port_ = 8080;
    bool tls_ = false;
public:
    ServerOptionsBuilder& host(std::string h) { host_ = std::move(h); return *this; }
    ServerOptionsBuilder& port(int p) { port_ = p; return *this; }
    ServerOptionsBuilder& tls(bool t = true) { tls_ = t; return *this; }
    ServerOptions build() const {
        if (port_ < 1 || port_ > 65535) throw std::invalid_argument("port out of range");
        return ServerOptions{host_, port_, tls_};
    }
};

int main() {
    auto o = ServerOptionsBuilder().host("api.example.com").port(443).tls().build();
    std::cout << o.host << ":" << o.port << (o.tls ? " (tls)" : "") << "\n";
}
```

**Python** often needs no builder: **keyword arguments with defaults** and `@dataclass(frozen=True)` give the same readability.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class ServerOptions:
    host: str = "localhost"
    port: int = 8080
    tls: bool = False

    def __post_init__(self):
        if not 1 <= self.port <= 65535:
            raise ValueError("port out of range")

print(ServerOptions(host="api.example.com", port=443, tls=True))
```

**Use when:** objects have many optional parts, need validation before they are usable, or should be immutable once built. Real examples: query builders, HTTP request builders, test-data builders, `StringBuilder` / `std::ostringstream` (build a string efficiently: repeated `+` on immutable strings is $O(n^2)$, doc 00).

**Trade-offs:** more code than a constructor; unnecessary for objects with two or three required fields.

---

## 10.1.5 Prototype

**Intent:** create new objects by **copying (cloning) an existing object** instead of building from scratch.

**Problem:** construction is expensive (loads a template, parses a file, queries a database) or you only have a base-class reference and do not know the concrete type to copy. Clone a ready-made **prototype** and change only what differs.

```text
 prototype ──clone()──► copy ──(tweak a few fields)──► new object
```

**Python:** `copy.deepcopy` (doc 08, deep vs shallow copy):

```python
import copy

default_settings = {"theme": "dark", "notifications": {"email": True, "sms": False}}

user_a = copy.deepcopy(default_settings)
user_a["notifications"]["sms"] = True            # does not affect the template or other users

print(default_settings["notifications"]["sms"])  # False
print(user_a["notifications"]["sms"])            # True
```

**JavaScript.** `structuredClone` makes a deep copy of plain data but **drops the prototype**: a cloned class instance becomes a plain object without its methods. For class instances, give the class its own `clone()`:

```js
class Enemy {
  constructor(name, hp, loot) { this.name = name; this.hp = hp; this.loot = loot; }
  clone() { return new Enemy(this.name, this.hp, [...this.loot]); }   // copy the array: deep enough
  describe() { return `${this.name} (${this.hp} hp)`; }
}

const goblinTemplate = new Enemy("Goblin", 30, ["coin"]);
const boss = goblinTemplate.clone();
boss.name = "Goblin King";
boss.hp = 300;
boss.loot.push("crown");

console.log(goblinTemplate.describe(), goblinTemplate.loot);   // Goblin (30 hp) [ 'coin' ]
console.log(boss.describe(), boss.loot);                       // Goblin King (300 hp) [ 'coin', 'crown' ]
console.log(structuredClone(goblinTemplate).describe);         // undefined: the methods were lost
```

(Game engines use this idea at scale: spawning an enemy clones a ready-made template, such as a prefab, rather than building it field by field.)

**C++: a virtual `clone()` copies through a base pointer** without knowing the concrete type. This is the standard fix for the **slicing** problem from doc 08:

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

struct Shape {
    virtual ~Shape() = default;
    virtual std::unique_ptr<Shape> clone() const = 0;     // the "virtual copy constructor"
    virtual std::string name() const = 0;
};
struct Circle : Shape {
    double r;
    explicit Circle(double r) : r(r) {}
    std::unique_ptr<Shape> clone() const override { return std::make_unique<Circle>(*this); }
    std::string name() const override { return "circle r=" + std::to_string(r); }
};
struct Square : Shape {
    double side;
    explicit Square(double s) : side(s) {}
    std::unique_ptr<Shape> clone() const override { return std::make_unique<Square>(*this); }
    std::string name() const override { return "square s=" + std::to_string(side); }
};

int main() {
    std::vector<std::unique_ptr<Shape>> shapes;
    shapes.push_back(std::make_unique<Circle>(1.5));
    shapes.push_back(std::make_unique<Square>(2.0));

    std::vector<std::unique_ptr<Shape>> copies;
    for (const auto& s : shapes) copies.push_back(s->clone());     // correct dynamic type is copied

    for (const auto& s : copies) std::cout << s->name() << "\n";
}
```

**C#:** implement your own `Clone()`; `MemberwiseClone()` is shallow (doc 08). The built-in `ICloneable` interface does not say whether a clone is deep or shallow, so it is usually avoided in new code.

**Use when:** creating by copying is cheaper or simpler than creating from scratch (templates, default configurations, game entities, undo snapshots).

**Trade-offs:** deep-copying object graphs with **cycles or shared references** is tricky; `deepcopy` handles cycles, hand-written `clone()` methods must be written with care.

---

# 10.2 Structural Patterns

Structural patterns describe **how to assemble objects and classes** into larger structures while keeping them flexible. Several of them look almost the same in code (they all wrap another object); they differ by **intent**:

| Pattern | Wraps another object to... | Interface of the wrapper |
| --- | --- | --- |
| **Adapter** | **Convert** an incompatible interface into the expected one | Different from the wrapped object |
| **Decorator** | **Add** behavior (stackable) | Same as the wrapped object |
| **Proxy** | **Control access** (lazy, cache, protect, remote) | Same as the wrapped object |
| **Facade** | **Simplify** a whole subsystem | New, simpler interface over many objects |

---

## 10.2.1 Adapter

**Intent:** convert the interface of a class into the one clients expect, so classes with incompatible interfaces can work together.

**Problem:** your application expects `PaymentGateway.charge(cents, token)`. A third-party SDK you cannot modify offers `make_payment(amount_in_rupees, card_no)`. Instead of changing your application (or the SDK), write a small **adapter**.

```text
 Application ──► PaymentGateway (target interface)
                       ▲
                 LegacyPaymentAdapter ───has-a──► LegacyPaymentSdk (adaptee)
                  charge(cents, token)  translates to  make_payment(rupees, card_no)
```

```python
class LegacyPaymentSdk:                           # third-party code we cannot change
    def make_payment(self, amount_in_rupees: float, card_no: str) -> dict:
        return {"ok": True, "txn": "T123"}

class PaymentGateway:                             # the interface our application expects
    def charge(self, cents: int, token: str) -> bool:
        raise NotImplementedError

class LegacyPaymentAdapter(PaymentGateway):
    def __init__(self, sdk: LegacyPaymentSdk):
        self._sdk = sdk
    def charge(self, cents, token):
        # the SDK wants floating-point rupees; the conversion is confined to this one place
        result = self._sdk.make_payment(cents / 100, token)
        return result["ok"]

gateway: PaymentGateway = LegacyPaymentAdapter(LegacyPaymentSdk())
print(gateway.charge(49900, "tok_abc"))            # True
```

```cpp
// C++ object adapter: hold the adaptee by composition
#include <iostream>
#include <string>

class LegacyLogger {                                   // adaptee, fixed
public:
    void writeLine(const char* text, int severityCode) {
        std::cout << "[" << severityCode << "] " << text << "\n";
    }
};

class Logger {                                         // target interface
public:
    virtual ~Logger() = default;
    virtual void info(const std::string& msg) = 0;
    virtual void error(const std::string& msg) = 0;
};

class LegacyLoggerAdapter : public Logger {
    LegacyLogger& legacy_;
public:
    explicit LegacyLoggerAdapter(LegacyLogger& l) : legacy_(l) {}
    void info(const std::string& msg) override  { legacy_.writeLine(msg.c_str(), 1); }
    void error(const std::string& msg) override { legacy_.writeLine(msg.c_str(), 3); }
};

int main() {
    LegacyLogger legacy;
    LegacyLoggerAdapter logger(legacy);
    logger.info("service started");
    logger.error("disk almost full");
}
```

(A **class adapter** inherits from the adaptee instead of holding it; it needs multiple inheritance and is rarer. Prefer the composition-based **object adapter** above.)

**Where you meet it:** database drivers behind a common interface, wrapping each cloud vendor's SDK behind your own interface, translating between an external API's data model and yours. At architecture scale this is the **anti-corruption layer** between a legacy system and a new service (doc 31).

**Trade-offs:** one more class per adapted thing; hides but does not remove the mismatch (watch for lossy conversions like the float above).

---

## 10.2.2 Decorator

**Intent:** attach **additional responsibilities** to an object dynamically by wrapping it in objects with the **same interface**. A flexible alternative to subclassing.

**Problem:** a data sink may need compression, encryption, logging, or any combination. With inheritance, every combination is a class (up to $2^{n}$ for $n$ features, 9.6). With decorators, each feature is **one** class and you stack them at runtime.

```text
 client ──► Compression ──► Encryption ──► FileSource
              write(data)    write(data)     write(data)
   each layer does its own work, then calls inner.write(...)
```

```js
class FileSource {
  write(data) { return `file<${data}>`; }
}
class EncryptionDecorator {
  constructor(inner) { this.inner = inner; }
  write(data) { return this.inner.write(`enc(${data})`); }
}
class CompressionDecorator {
  constructor(inner) { this.inner = inner; }
  write(data) { return this.inner.write(`gzip(${data})`); }
}

// Order matters: compress first, then encrypt (encrypted data looks random and does not compress)
const sink = new CompressionDecorator(new EncryptionDecorator(new FileSource()));
console.log(sink.write("hello"));      // file<enc(gzip(hello))>
```

**Python** has two related things with the same name:

1. The **Decorator pattern** (object wrapping, as above).
2. The `@decorator` **syntax**: a function that wraps another function. It applies the same idea to functions.

```python
import functools
import time

def timed(fn):
    @functools.wraps(fn)                       # keeps the original name and docstring
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        try:
            return fn(*args, **kwargs)
        finally:
            elapsed_ms = (time.perf_counter() - start) * 1000
            print(f"{fn.__name__} took {elapsed_ms:.1f} ms")
    return wrapper

@timed
def slow_query():
    time.sleep(0.01)
    return "rows"

print(slow_query())
```

**C#: a logging decorator over an interface** (the same shape works for caching, retries, metrics):

```csharp
public interface IMessageSender { Task SendAsync(string to, string body); }

public sealed class SmtpSender : IMessageSender
{
    public Task SendAsync(string to, string body) { /* talk to the SMTP server */ return Task.CompletedTask; }
}

public sealed class LoggingSender : IMessageSender
{
    private readonly IMessageSender _inner;
    private readonly ILogger<LoggingSender> _log;
    public LoggingSender(IMessageSender inner, ILogger<LoggingSender> log) { _inner = inner; _log = log; }

    public async Task SendAsync(string to, string body)
    {
        _log.LogInformation("sending to {To}", to);
        await _inner.SendAsync(to, body);                  // delegate to the wrapped object
        _log.LogInformation("sent to {To}", to);
    }
}
```

**C++:** the same wrapper idea with `std::unique_ptr<Interface>` as the inner member.

**Where you meet it:** I/O stream wrappers (`BufferedReader(InputStreamReader(...))` in Java, Python's `io` module layers), middleware stacks (10.4.6), caching and retry wrappers around repositories (9.6), `functools.lru_cache`.

**Trade-offs:** many small objects; **order of wrapping matters**; debugging means walking through several layers; identity checks (`obj is concrete`) stop working because you hold the wrapper.

---

## 10.2.3 Facade

**Intent:** provide a **single simple interface** to a complex subsystem.

**Problem:** placing an order requires calling inventory, payment and shipping, in the right order, with the right arguments. Making every caller know that is repetitive and brittle.

```text
              ┌────────────────┐        ┌─ InventoryService
 Client ────► │ CheckoutFacade │ ─────► ├─ PaymentService
              │ placeOrder()   │        └─ ShippingService
              └────────────────┘
```

```js
class InventoryService { reserve(sku, qty) { return true; } }
class PaymentService   { charge(userId, cents) { return "pay_1"; } }
class ShippingService  { schedule(userId, sku) { return "ship_1"; } }

class CheckoutFacade {
  constructor(inventory, payment, shipping) {
    this.inventory = inventory;
    this.payment = payment;
    this.shipping = shipping;
  }
  placeOrder(userId, sku, qty, cents) {
    if (!this.inventory.reserve(sku, qty)) throw new Error("out of stock");
    const paymentId = this.payment.charge(userId, cents);
    const shipmentId = this.shipping.schedule(userId, sku);
    return { paymentId, shipmentId };
  }
}

const checkout = new CheckoutFacade(new InventoryService(), new PaymentService(), new ShippingService());
console.log(checkout.placeOrder("u1", "book-42", 1, 49900));
```

This sketch ignores failure: if payment fails after stock was reserved, the reservation must be released. Handling that across services is the subject of distributed transactions and the Saga pattern (doc 35).

**Where you meet it:** a service layer (10.4.2) is a facade over domain objects and repositories; an **API gateway** is a facade over many microservices (doc 39); a library's high-level client (`requests.get`) hides sockets, TLS and HTTP parsing.

**Facade vs Adapter:** an adapter makes **one** incompatible interface fit an expected one; a facade offers a **new, simpler** interface over **many** objects. Neither hides the subsystem from people who need it: the subsystem stays directly usable.

**Trade-offs:** can become a **god object** that knows too much; keep it thin and delegate real work to the subsystem.

---

## 10.2.4 Proxy

**Intent:** provide a **surrogate** that controls access to another object, using the same interface.

| Proxy kind | What it does | Example |
| --- | --- | --- |
| **Virtual (lazy)** | Creates the real, expensive object only when first used | ORM lazy loading of related rows |
| **Caching** | Remembers results of earlier calls | Caching wrapper around a remote API client |
| **Protection** | Checks permissions before delegating | Only admins may call `delete()` |
| **Remote** | Represents an object in another process or machine | RPC/gRPC client stubs |
| **Smart reference** | Adds bookkeeping on access | `std::shared_ptr` reference counting |

```text
 client ──► Proxy ──(only if needed)──► RealSubject
            same interface as RealSubject
```

**Python: a lazy + caching proxy:**

```python
class ReportService:                                   # the real subject
    def __init__(self):
        print("connecting to the data warehouse ...")  # expensive
    def monthly_total(self, month: str) -> int:
        print(f"computing {month} ...")
        return 42_000

class ReportServiceProxy:
    def __init__(self):
        self._real = None
        self._cache = {}
    def monthly_total(self, month):
        if month in self._cache:                        # caching proxy
            return self._cache[month]
        if self._real is None:                          # virtual proxy: create on first real use
            self._real = ReportService()
        self._cache[month] = self._real.monthly_total(month)
        return self._cache[month]

service = ReportServiceProxy()                          # nothing expensive happened yet
print(service.monthly_total("2026-09"))                 # connects and computes
print(service.monthly_total("2026-09"))                 # served from the proxy's cache
```

**JavaScript has a built-in `Proxy` object** that intercepts reads, writes, and calls:

```js
const user = { name: "Asha", role: "viewer" };

const guarded = new Proxy(user, {
  get(target, prop, receiver) {
    console.log("read", String(prop));
    return Reflect.get(target, prop, receiver);
  },
  set(target, prop, value) {
    if (prop === "role") throw new Error("role is read-only");   // protection proxy
    target[prop] = value;
    return true;
  },
});

console.log(guarded.name);          // logs "read name", then prints Asha
guarded.name = "Asha K";            // allowed
try { guarded.role = "admin"; } catch (e) { console.log(e.message); }   // role is read-only
```

(Vue 3 uses this mechanism for reactivity: reads and writes of state are intercepted to track dependencies.)

**At infrastructure scale:** a **reverse proxy** such as Nginx stands in front of your servers and exposes the same HTTP interface (docs 28, 30); a **CDN** is a giant caching proxy (doc 52).

**Proxy vs Decorator:** structurally identical. A decorator **adds behavior** the client asked for and is usually composed by the client; a proxy **controls access** to the subject and often manages the subject's lifecycle itself.

**Trade-offs:** adds a layer (small latency, more code); a proxy that caches must also solve **cache invalidation** (doc 23).

---

## 10.2.5 Composite

**Intent:** compose objects into **tree structures** and let clients treat **individual objects and groups of objects uniformly**.

**Problem:** a file system has files and folders; a folder contains files and other folders. You want `size()` to work on either without checking which one you have.

```text
         Node (interface)  size()
          ▲          ▲
        File      Directory ◇──────► children: list of Node   (recursive!)
```

The size of a node $n$ is defined recursively:

$$S(n)=\begin{cases}\text{bytes}(n) & n \text{ is a file}\\[4pt] \displaystyle\sum_{c\,\in\,\text{children}(n)} S(c) & n \text{ is a directory}\end{cases}$$

Computing it visits every node once, so it takes $O(N)$ time for $N$ nodes, and the recursion depth equals the height of the tree.

```python
from abc import ABC, abstractmethod

class Node(ABC):
    @abstractmethod
    def size(self) -> int: ...

class File(Node):
    def __init__(self, name: str, size: int):
        self.name, self._size = name, size
    def size(self) -> int:
        return self._size

class Directory(Node):
    def __init__(self, name: str):
        self.name = name
        self.children: list[Node] = []
    def add(self, node: Node) -> "Directory":
        self.children.append(node)
        return self
    def size(self) -> int:
        return sum(child.size() for child in self.children)    # works for files AND directories

root = (Directory("root")
        .add(File("a.txt", 100))
        .add(Directory("src").add(File("b.py", 50)).add(File("c.py", 25))))
print(root.size())      # 175
```

```cpp
// C++: the composite owns its children
#include <iostream>
#include <memory>
#include <string>
#include <vector>

struct Node {
    virtual ~Node() = default;
    virtual long long size() const = 0;
};
struct File : Node {
    long long bytes;
    explicit File(long long b) : bytes(b) {}
    long long size() const override { return bytes; }
};
struct Directory : Node {
    std::vector<std::unique_ptr<Node>> children;
    Directory& add(std::unique_ptr<Node> n) { children.push_back(std::move(n)); return *this; }
    long long size() const override {
        long long total = 0;
        for (const auto& c : children) total += c->size();     // uniform call
        return total;
    }
};

int main() {
    Directory root;
    root.add(std::make_unique<File>(100));
    auto src = std::make_unique<Directory>();
    src->add(std::make_unique<File>(50)).add(std::make_unique<File>(25));
    root.add(std::move(src));
    std::cout << root.size() << "\n";    // 175
}
```

**Where you meet it:** the DOM and React component trees, UI widget hierarchies, expression trees (`(a + b) * c`), org charts, menus, and **role/permission groups** where a group can contain users or other groups (doc 18).

**Trade-offs:** a design question is whether child-management methods (`add`, `remove`) belong on the common interface (uniform but unsafe for leaves) or only on the composite (safe but clients must know the type). Cycles (a folder containing itself) must be prevented or the recursion never ends.

---

## 10.2.6 Bridge

**Intent:** **decouple an abstraction from its implementation** so the two can vary independently.

**Problem:** notifications have **kinds** (alert, reminder, receipt) and **channels** (email, SMS, push, Slack). Subclassing every combination gives $m\times n$ classes (3 kinds × 4 channels = 12). With a bridge you keep **two small hierarchies** and combine them by composition: $m+n$ classes (3 + 4 = 7). Adding a fifth channel adds **one** class instead of three.

```text
   Notification (abstraction)  ───has-a───►  Channel (implementor interface)
     ▲          ▲                                ▲          ▲
   Alert     Reminder                        EmailChannel  SmsChannel
```

```python
from abc import ABC, abstractmethod

class Channel(ABC):                                     # implementation hierarchy
    @abstractmethod
    def deliver(self, to: str, text: str) -> str: ...

class EmailChannel(Channel):
    def deliver(self, to, text): return f"EMAIL {to}: {text}"

class SmsChannel(Channel):
    def deliver(self, to, text): return f"SMS {to}: {text}"

class Notification:                                     # abstraction hierarchy
    def __init__(self, channel: Channel):
        self.channel = channel                          # the "bridge"
    def send(self, to: str) -> str:
        raise NotImplementedError

class Alert(Notification):
    def send(self, to): return self.channel.deliver(to, "ALERT: server down")

class Reminder(Notification):
    def send(self, to): return self.channel.deliver(to, "Reminder: meeting at 5")

print(Alert(SmsChannel()).send("user-1"))               # any kind works with any channel
print(Reminder(EmailChannel()).send("user-2"))
```

**Where you meet it:** drivers (your code talks to a `Connection` abstraction; a PostgreSQL or MySQL driver implements it), logging facades with pluggable back ends, graphics APIs (shapes × renderers), and the C++ **pimpl idiom** ("pointer to implementation") that hides a class's private members behind a pointer so changing them does not recompile every user.

**Bridge vs Adapter vs Strategy:** an **adapter** fixes an incompatibility **after** the fact; a **bridge** is designed **up front** so two dimensions can evolve separately; a **Strategy** (10.3.1) has the same structure but its intent is to swap an **algorithm** inside one object.

**Trade-offs:** more indirection and more interfaces; only worth it when there really are two independent axes of variation.

---

# 10.3 Behavioral Patterns

Behavioral patterns are about **how objects interact and how responsibility is divided** between them.

---

## 10.3.1 Strategy

**Intent:** define a family of **interchangeable algorithms**, encapsulate each one, and let the client choose which to use at runtime.

**Problem:** a load balancer must choose a server. Round robin, least connections and random are all valid; hard-coding one with `if/else` means editing tested code whenever a policy is added (the Open/Closed Principle, 9.5).

```text
 LoadBalancer ──has-a──► Strategy (pick(servers))
                            ▲        ▲          ▲
                     RoundRobin  LeastConn   Random
```

```python
import itertools
from typing import Callable, Sequence

Server = dict                                    # {"name": str, "connections": int}
Strategy = Callable[[Sequence[Server]], Server]  # a strategy is just a function here

def round_robin() -> Strategy:
    counter = itertools.count()
    def pick(servers):
        return servers[next(counter) % len(servers)]
    return pick

def least_connections() -> Strategy:
    def pick(servers):
        return min(servers, key=lambda s: s["connections"])
    return pick

class LoadBalancer:
    def __init__(self, servers, strategy: Strategy):
        self.servers = servers
        self.strategy = strategy                 # can be swapped at runtime
    def route(self) -> str:
        return self.strategy(self.servers)["name"]

servers = [
    {"name": "A", "connections": 5},
    {"name": "B", "connections": 1},
    {"name": "C", "connections": 3},
]
lb = LoadBalancer(servers, round_robin())
print([lb.route() for _ in range(4)])           # ['A', 'B', 'C', 'A']
lb.strategy = least_connections()
print(lb.route())                               # 'B'
```

In languages with first-class functions the strategy is just a function. In class-based style (C#, C++) it is an **interface**:

```csharp
public interface IPricingStrategy { decimal Price(decimal basePrice); }

public sealed class RegularPricing : IPricingStrategy
{
    public decimal Price(decimal basePrice) => basePrice;
}
public sealed class FestivalSalePricing : IPricingStrategy
{
    public decimal Price(decimal basePrice) => basePrice * 0.8m;      // 20% off
}

public sealed class Cart
{
    private readonly IPricingStrategy _pricing;
    public Cart(IPricingStrategy pricing) => _pricing = pricing;
    public decimal Total(IEnumerable<decimal> items) => items.Sum(p => _pricing.Price(p));
}
```

```cpp
// C++: std::function lets any callable (lambda, function pointer, functor) be a strategy
#include <algorithm>
#include <functional>
#include <iostream>
#include <vector>

void sortWith(std::vector<int>& v, std::function<bool(int, int)> before) {
    std::sort(v.begin(), v.end(), before);
}

int main() {
    std::vector<int> v = {3, 1, 2};
    sortWith(v, [](int a, int b) { return a < b; });     // ascending strategy
    for (int x : v) std::cout << x << " ";               // 1 2 3
    sortWith(v, [](int a, int b) { return a > b; });     // descending strategy
    for (int x : v) std::cout << x << " ";               // 3 2 1
    std::cout << "\n";
}
```

`std::sort`'s comparator is itself the Strategy pattern.

**Where you meet it in this roadmap:** load-balancing algorithms (doc 22), rate-limiting algorithms (doc 38), cache eviction policies (LRU, LFU; doc 23), retry/backoff policies (doc 36), password hashing algorithms (doc 18), payment providers.

**Strategy vs State:** a strategy is **chosen by the client** and does not usually change on its own; states **switch themselves** as the object runs (10.3.4).

**Trade-offs:** clients must know the strategies exist to choose between them; for two trivial variants a plain `if` is simpler.

---

## 10.3.2 Observer

**Intent:** define a **one-to-many dependency** so that when one object (the **subject**, or publisher) changes state, all its **observers** (subscribers) are notified automatically.

**Problem:** when an order is created, the system must send an email, update analytics and reserve stock. If `OrderService` calls all three directly, it knows about every consumer and must change whenever a new one is added. With Observer it only announces "an order was created".

```text
              ┌──► EmailObserver
 Subject ─────┼──► AnalyticsObserver
 emit(event)  └──► InventoryObserver
```

**JavaScript: a minimal event bus:**

```js
class EventBus {
  #handlers = new Map();                       // event name -> Set of functions

  on(event, handler) {
    if (!this.#handlers.has(event)) this.#handlers.set(event, new Set());
    this.#handlers.get(event).add(handler);
    return () => this.#handlers.get(event).delete(handler);       // returns an "unsubscribe" function
  }
  emit(event, payload) {
    for (const handler of this.#handlers.get(event) ?? []) handler(payload);
  }
}

const bus = new EventBus();
const unsubscribeEmail = bus.on("order.created", (o) => console.log("email for order", o.id));
bus.on("order.created", (o) => console.log("analytics for order", o.id));

bus.emit("order.created", { id: 1 });          // both observers run
unsubscribeEmail();
bus.emit("order.created", { id: 2 });          // only analytics runs
```

Node.js ships this pattern as `EventEmitter`:

```js
const { EventEmitter } = require("events");

const emitter = new EventEmitter();
emitter.on("order.created", (o) => console.log("handler 1:", o.id));
emitter.on("order.created", (o) => console.log("handler 2:", o.id));
emitter.emit("order.created", { id: 7 });      // handlers run synchronously, in registration order
```

Facts worth knowing: `emit` calls listeners **synchronously**; emitting an `'error'` event with no `'error'` listener **throws**; Node warns when more than 10 listeners are added to one event (a sign of a leak).

**C#: events are Observer built into the language:**

```csharp
public class OrderService
{
    public event Action<int>? OrderCreated;                  // the subject's event

    public void Create(int id)
    {
        // ... save the order ...
        OrderCreated?.Invoke(id);                            // notify all subscribers
    }
}

// var svc = new OrderService();
// svc.OrderCreated += id => Console.WriteLine($"email for {id}");
// svc.OrderCreated += id => Console.WriteLine($"analytics for {id}");
// svc.Create(1);
```

**C++:**

```cpp
#include <functional>
#include <iostream>
#include <vector>

class OrderService {
    std::vector<std::function<void(int)>> observers_;
public:
    void subscribe(std::function<void(int)> f) { observers_.push_back(std::move(f)); }
    void create(int id) {
        for (auto& f : observers_) f(id);                    // notify everyone
    }
};

int main() {
    OrderService svc;
    svc.subscribe([](int id) { std::cout << "email for " << id << "\n"; });
    svc.subscribe([](int id) { std::cout << "analytics for " << id << "\n"; });
    svc.create(1);
}
```

**Pitfalls**

| Problem | Explanation |
| --- | --- |
| **Lapsed listener leak** | An observer that is never unsubscribed stays reachable from the subject, so it is never garbage collected |
| **Synchronous slow observer** | One slow handler delays the whole `emit` and the code after it |
| **Exceptions** | One failing handler may stop later handlers from running, unless you catch per handler |
| **Ordering and cascades** | Handlers that emit more events can cause hard-to-follow chains |

**In-process Observer vs distributed publish/subscribe.** The pattern above lives inside one process: delivery is immediate, nothing is stored, and if the process crashes pending notifications are lost. **Pub/sub through a message broker** (docs 25, 26, 27) gives the same decoupling **between services**, plus durability, retries, ordering guarantees and independent consumer speeds, at the price of network latency and failure handling.

---

## 10.3.3 Command

**Intent:** encapsulate a **request as an object**, so it can be queued, logged, retried, passed around, and **undone**.

**Problem:** a text editor needs undo/redo; a worker needs to run jobs later; an audit trail needs a record of every action. If actions are only direct method calls, none of this is possible. A command object holds **what to do** (and how to reverse it).

```text
 Invoker ──► Command.execute() ──► Receiver (does the real work)
 (editor)    Command.undo()        (document)
```

```python
class Document:
    def __init__(self):
        self.text = ""

class AppendText:                                   # a concrete command
    def __init__(self, doc: Document, s: str):
        self.doc, self.s = doc, s
    def execute(self):
        self.doc.text += self.s
    def undo(self):
        self.doc.text = self.doc.text[: len(self.doc.text) - len(self.s)]

class Editor:                                       # the invoker keeps the history
    def __init__(self):
        self.doc = Document()
        self._undo, self._redo = [], []
    def run(self, command):
        command.execute()
        self._undo.append(command)
        self._redo.clear()                          # a new action invalidates the redo history
    def undo(self):
        if self._undo:
            command = self._undo.pop()
            command.undo()
            self._redo.append(command)
    def redo(self):
        if self._redo:
            command = self._redo.pop()
            command.execute()
            self._undo.append(command)

editor = Editor()
editor.run(AppendText(editor.doc, "Hello"))
editor.run(AppendText(editor.doc, ", world"))
print(editor.doc.text)      # Hello, world
editor.undo()
print(editor.doc.text)      # Hello
editor.redo()
print(editor.doc.text)      # Hello, world
```

In languages with lambdas, a simple command is just a function (`std::function<void()>`, a JavaScript closure); you need a class only when you need **undo**, **serialization** or extra data.

```js
// JavaScript: a task queue where each task is a command (a closure)
const queue = [];
queue.push(() => console.log("send welcome email"));
queue.push(() => console.log("resize avatar"));
while (queue.length) queue.shift()();               // execute later, in order
```

**Where you meet it in this roadmap:** **job queues** (a job message is a serialized command; docs 25, 27), the **write-ahead log** of a database (a log of changes that can be replayed, docs 12, 49), CQRS "commands" and **event sourcing** (doc 26), and compensating actions in Sagas (doc 35).

**Trade-offs:** one class per action; undo for actions with external side effects (an email already sent) is impossible or needs a compensating action.

---

## 10.3.4 State

**Intent:** let an object **alter its behavior when its internal state changes**; it appears to change its class.

**Problem:** an order can be created, paid, shipped, delivered or cancelled. What `ship()` means depends on the state: allowed after payment, forbidden before. Scattered `if (status == ...)` checks all over the code are fragile.

State diagram:

```text
                 pay            ship            deliver
   CREATED ───────────► PAID ───────────► SHIPPED ───────────► DELIVERED
      │                  │
      │ cancel           │ refund
      ▼                  ▼
  CANCELLED          REFUNDED
```

**Table-driven (simple, very common in backends):**

```js
const transitions = {
  CREATED:   { pay: "PAID", cancel: "CANCELLED" },
  PAID:      { ship: "SHIPPED", refund: "REFUNDED" },
  SHIPPED:   { deliver: "DELIVERED" },
  DELIVERED: {},
  CANCELLED: {},
  REFUNDED:  {},
};

class Order {
  state = "CREATED";

  apply(event) {
    const allowed = transitions[this.state];
    if (!Object.hasOwn(allowed, event)) {
      throw new Error(`cannot "${event}" when order is ${this.state}`);
    }
    this.state = allowed[event];
    return this;
  }
}

const order = new Order();
console.log(order.apply("pay").apply("ship").state);      // SHIPPED
try { order.apply("cancel"); } catch (e) { console.log(e.message); }   // cannot "cancel" when order is SHIPPED
```

**Object-based (the GoF form): each state is a class that knows what is allowed and which state comes next:**

```csharp
public interface IOrderState
{
    IOrderState Pay();
    IOrderState Ship();
}

public sealed class Created : IOrderState
{
    public IOrderState Pay()  => new Paid();
    public IOrderState Ship() => throw new InvalidOperationException("pay first");
}
public sealed class Paid : IOrderState
{
    public IOrderState Pay()  => throw new InvalidOperationException("already paid");
    public IOrderState Ship() => new Shipped();
}
public sealed class Shipped : IOrderState
{
    public IOrderState Pay()  => throw new InvalidOperationException("already paid");
    public IOrderState Ship() => throw new InvalidOperationException("already shipped");
}

public sealed class Order
{
    private IOrderState _state = new Created();
    public void Pay()  => _state = _state.Pay();
    public void Ship() => _state = _state.Ship();
}
```

**Backend notes**

- Store the state in a database column and validate transitions in **one** place.
- Make the transition **atomic** with a conditional update so two concurrent requests cannot both move the same order (optimistic concurrency, doc 14):

```sql
UPDATE orders SET status = 'PAID' WHERE id = 42 AND status = 'CREATED';
-- if 0 rows were updated, someone else already changed it: reject or retry
```

**Where you meet it:** order and payment lifecycles, TCP connection states (doc 05), circuit breakers (closed/open/half-open; doc 36), Saga progress (doc 35), workflow engines.

**State vs Strategy:** same structure (an object delegates to a replaceable helper). In State the helpers **replace themselves** and know about each other; in Strategy the client picks one and they are independent.

**Trade-offs:** many small classes for a simple machine; the table-driven form is often enough.

---

## 10.3.5 Template Method

**Intent:** define the **skeleton of an algorithm** in a base class, deferring some steps to subclasses. Subclasses redefine steps without changing the algorithm's structure.

**Problem:** every importer (CSV, JSON, API) does the same thing in the same order: read, parse, validate, save. Only some steps differ. Put the fixed order in one place.

```text
 Importer.run():            ← the template (not overridden)
   raw  = read(source)
   rows = parse(raw)        ← abstract: every subclass must implement
   good = validate(rows)    ← hook: default behavior, may be overridden
   save(good)               ← abstract
```

```python
from abc import ABC, abstractmethod

class Importer(ABC):
    def run(self, source):                              # the template method
        raw = self.read(source)
        rows = self.parse(raw)
        valid = [row for row in rows if self.validate(row)]
        self.save(valid)
        return len(valid)

    def read(self, source):                             # default step
        return source
    @abstractmethod
    def parse(self, raw): ...                           # required step
    def validate(self, row):                            # hook with a default
        return True
    @abstractmethod
    def save(self, rows): ...                           # required step

class CsvUserImporter(Importer):
    def __init__(self):
        self.saved = []
    def parse(self, raw):
        return [line.split(",") for line in raw.strip().splitlines()]
    def validate(self, row):
        return len(row) == 2 and "@" in row[1]          # name,email
    def save(self, rows):
        self.saved.extend(rows)

importer = CsvUserImporter()
count = importer.run("asha,asha@example.com\nbad-row\nravi,ravi@example.com")
print(count, importer.saved)      # 2 [['asha', 'asha@example.com'], ['ravi', 'ravi@example.com']]
```

**C++ idiom: Non-Virtual Interface (NVI).** The public function is non-virtual and fixed; the steps are private virtual functions:

```cpp
#include <iostream>

class Job {
public:
    virtual ~Job() = default;
    void run() {                                  // the template: public, non-virtual
        setup();
        execute();
        teardown();
    }
private:
    virtual void setup() {}                       // optional hook
    virtual void execute() = 0;                   // required step
    virtual void teardown() {}                    // optional hook
};

class BackupJob : public Job {
    void execute() override { std::cout << "backing up\n"; }
};

int main() { BackupJob().run(); }
```

This is the **Hollywood principle**: "don't call us, we'll call you". The framework (base class) calls your code. Frameworks use it widely: test frameworks' `setUp`/`tearDown`, ASP.NET Core's `BackgroundService.ExecuteAsync`, servlet `doGet`.

**Trade-offs:** it uses **inheritance**, so subclasses are tightly coupled to the base class's step order. If the steps vary independently, prefer Strategy objects (composition, 9.6).

---

## 10.3.6 Chain of Responsibility

**Intent:** pass a request along a **chain of handlers**; each handler decides to **process it**, **reject it**, or **pass it on**.

**Problem:** an incoming request must be authenticated, rate limited and validated. Each check is separate, the set and order can change, and any check can stop the request.

```text
 request ─► Auth ─► RateLimit ─► Validation ─► (final handler) ─► response
              │         │             │
              └─────────┴─────────────┴──► any link can stop the chain and answer immediately
```

```python
class Handler:
    def __init__(self, next_handler=None):
        self.next = next_handler
    def handle(self, request: dict) -> str:
        if self.next:
            return self.next.handle(request)             # pass it on
        return "200 OK"                                  # end of the chain

class AuthHandler(Handler):
    def handle(self, request):
        if not request.get("user"):
            return "401 unauthenticated"                 # stop here
        return super().handle(request)

class RateLimitHandler(Handler):
    def handle(self, request):
        if request.get("calls_this_minute", 0) > 100:
            return "429 too many requests"
        return super().handle(request)

class ValidationHandler(Handler):
    def handle(self, request):
        if not request.get("body"):
            return "400 empty body"
        return super().handle(request)

chain = AuthHandler(RateLimitHandler(ValidationHandler()))
print(chain.handle({"user": "asha", "body": "x"}))       # 200 OK
print(chain.handle({"body": "x"}))                       # 401 unauthenticated
print(chain.handle({"user": "asha", "calls_this_minute": 500, "body": "x"}))   # 429 too many requests
```

**Where you meet it:** **HTTP middleware** (10.4.6) is this pattern; also logging frameworks (levels), event bubbling in the browser DOM, exception handling (`catch` blocks tried in order), approval workflows.

**Chain of Responsibility vs Decorator:** in a decorator chain **every** layer adds something and passes the call on; in a chain of responsibility a handler may **end** the chain.

**Trade-offs:** a request can fall off the end **unhandled** if you forget a terminal handler; **order matters** and can be hard to see; long chains are harder to debug.

---

## 10.3.7 Iterator

**Intent:** provide a way to access the elements of a collection **sequentially without exposing its internal structure**.

**Problem:** callers should be able to loop over an array, a tree, a database result, or an infinite sequence with the same `for` loop, without knowing how it is stored.

Every mainstream language builds this in:

| Language | Protocol |
| --- | --- |
| JavaScript | `[Symbol.iterator]()` returning `{ next() }`; generators with `function*` |
| Python | `__iter__` and `__next__`; generators with `yield` |
| C++ | iterators with `begin()`/`end()`; range-based `for` |
| C# | `IEnumerable<T>` / `IEnumerator<T>`; `yield return` |
| C | none; use a cursor struct and a `next()` function |

**Generators** make writing iterators trivial and **lazy** (values are produced on demand, so memory stays $O(1)$ instead of $O(n)$):

```python
def fibonacci():
    a, b = 0, 1
    while True:                       # an infinite sequence is fine: values are made on demand
        yield a
        a, b = b, a + b

import itertools
print(list(itertools.islice(fibonacci(), 10)))     # [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

**A backend example: iterate over paginated API results as one stream** (cursor pagination, doc 17):

```js
async function* fetchAll(fetchPage) {
  let cursor = null;
  do {
    const { items, nextCursor } = await fetchPage(cursor);
    yield* items;                       // hand out one item at a time
    cursor = nextCursor;
  } while (cursor);
}

// fake paginated API for the demo: 2 items per page
const data = ["a", "b", "c", "d", "e"];
async function fetchPage(cursor) {
  const start = cursor ?? 0;
  const nextStart = start + 2;
  return { items: data.slice(start, nextStart), nextCursor: nextStart < data.length ? nextStart : null };
}

(async () => {
  for await (const item of fetchAll(fetchPage)) console.log(item);   // a b c d e
})();
```

The caller sees one stream; pages are fetched lazily as the loop needs them.

**C++: a minimal iterable class** (a range-based `for` needs `begin()`/`end()` returning objects with `*`, `++` and `!=`):

```cpp
#include <iostream>

class Countdown {
    int start_;
public:
    explicit Countdown(int s) : start_(s) {}

    class Iterator {
        int cur_;
    public:
        explicit Iterator(int c) : cur_(c) {}
        int operator*() const { return cur_; }
        Iterator& operator++() { --cur_; return *this; }
        bool operator!=(const Iterator& other) const { return cur_ != other.cur_; }
    };

    Iterator begin() const { return Iterator(start_); }
    Iterator end() const { return Iterator(0); }
};

int main() {
    for (int x : Countdown(3)) std::cout << x << " ";      // 3 2 1
    std::cout << "\n";
}
```

**C#:**

```csharp
public static IEnumerable<int> Evens(int max)
{
    for (int i = 0; i <= max; i += 2)
        yield return i;                  // the compiler builds the iterator state machine
}
// foreach (var n in Evens(10)) Console.WriteLine(n);
```

**Pitfalls:** modifying a collection while iterating over it can skip elements or crash. In C++ `push_back` on a `std::vector` may reallocate and **invalidate** every iterator; in Python changing a dict's size during iteration raises `RuntimeError`.

---

# 10.4 Backend-Relevant Patterns

These are not all in the GoF book, but they are the patterns you will use in almost every server application. Together they produce the layered structure covered in doc 19:

```text
 HTTP request
     ↓
 Middleware pipeline          (cross-cutting: auth, logging, errors)
     ↓
 Controller                   (HTTP in/out)               ← MVC
     ↓
 Service layer                (business use cases)        ← Unit of Work boundary
     ↓
 Repository                   (data access)
     ↓
 Database
 (dependencies wired together by Dependency Injection)
```

---

## 10.4.1 Repository

**Intent:** mediate between the domain and the data store with a **collection-like interface**, hiding queries and persistence details.

**Problem:** SQL scattered through business code ties it to one database and makes it impossible to test without one. A repository offers `add`, `get`, `find_by_email` and keeps all the SQL in one place.

```text
 Service ──► UserRepository (interface) ◄── SqlUserRepository   (production)
                                         ◄── InMemoryUserRepository   (tests)
```

```python
import sqlite3
from dataclasses import dataclass
from typing import Optional, Protocol

@dataclass
class User:
    id: Optional[int]
    email: str

class UserRepository(Protocol):                          # the abstraction the service depends on
    def add(self, user: User) -> User: ...
    def get(self, user_id: int) -> Optional[User]: ...
    def find_by_email(self, email: str) -> Optional[User]: ...

class SqliteUserRepository:                              # one implementation: SQL lives only here
    def __init__(self, conn: sqlite3.Connection):
        self.conn = conn
    def add(self, user):
        cur = self.conn.execute("INSERT INTO users(email) VALUES (?)", (user.email,))
        return User(cur.lastrowid, user.email)
    def get(self, user_id):
        row = self.conn.execute("SELECT id, email FROM users WHERE id = ?", (user_id,)).fetchone()
        return User(*row) if row else None
    def find_by_email(self, email):
        row = self.conn.execute("SELECT id, email FROM users WHERE email = ?", (email,)).fetchone()
        return User(*row) if row else None

class InMemoryUserRepository:                            # another implementation: instant, no database
    def __init__(self):
        self._users, self._next_id = {}, 1
    def add(self, user):
        saved = User(self._next_id, user.email)
        self._users[saved.id] = saved
        self._next_id += 1
        return saved
    def get(self, user_id):
        return self._users.get(user_id)
    def find_by_email(self, email):
        return next((u for u in self._users.values() if u.email == email), None)

class SignupService:                                     # depends only on the abstraction
    def __init__(self, users: UserRepository):
        self.users = users
    def register(self, email: str) -> User:
        if self.users.find_by_email(email):
            raise ValueError("email already registered")
        return self.users.add(User(None, email))

# the same service runs against either implementation
conn = sqlite3.connect(":memory:")
conn.execute("CREATE TABLE users(id INTEGER PRIMARY KEY AUTOINCREMENT, email TEXT UNIQUE NOT NULL)")
for repo in (SqliteUserRepository(conn), InMemoryUserRepository()):
    service = SignupService(repo)
    print(service.register("asha@example.com"))
    try:
        service.register("asha@example.com")
    except ValueError as e:
        print("rejected:", e)
```

**Guidelines**

- Expose **domain-meaningful** methods (`find_overdue_invoices`), not generic `execute(sql)`.
- Return domain objects, not raw rows.
- A repository is not the same as an ORM. Entity Framework's `DbSet<T>` and TypeORM repositories are already repository-like; an extra layer on top is worthwhile only if you need the isolation or testing benefits.
- Beware the **N+1 query** problem: loading a list and then querying once per item. Provide methods that load what the use case needs in one query (a JOIN or `IN` query; doc 11).

---

## 10.4.2 Service Layer

**Intent:** define the application's **use cases** in one layer that coordinates domain objects and repositories, and marks the **transaction boundary**.

| Layer | Responsible for | Must not |
| --- | --- | --- |
| **Controller** | Parse the HTTP request, call one service method, map the result or error to a status code and body | Contain business rules or SQL |
| **Service** | Orchestrate a use case (`placeOrder`), enforce business rules, own the transaction | Know about HTTP (`req`, `res`) or SQL |
| **Repository** | Read and write persistent data | Contain business decisions |

```js
// Express-style controller + service (illustrative: app, orderService, uowFactory, ConflictError are assumed to exist)
// controller: thin
app.post("/orders", async (req, res, next) => {
  try {
    const order = await orderService.placeOrder(req.user.id, req.body.items);
    res.status(201).json(order);
  } catch (err) {
    next(err);                                         // errors handled in one central place
  }
});

// service: one method = one use case
class OrderService {
  constructor(uowFactory) { this.uowFactory = uowFactory; }

  async placeOrder(userId, items) {
    return this.uowFactory.run(async (uow) => {        // one transaction around the whole use case
      for (const item of items) {
        const product = await uow.products.get(item.productId);
        if (!product || product.stock < item.qty) throw new ConflictError("insufficient stock");
        product.stock -= item.qty;
        await uow.products.save(product);
      }
      return uow.orders.add({ userId, items });
    });
  }
}
```

Because the service knows nothing about HTTP, the same use case can be triggered from a REST endpoint, a message consumer, a cron job or a test.

**Trade-offs:** a service layer that merely forwards calls to repositories adds nothing. If all business logic lives in services and the domain objects are only data bags, you have an **anemic domain model**; move rules into the objects that own the data when they naturally belong there.

---

## 10.4.3 Dependency Injection (DI)

**Intent:** a class **receives** the objects it depends on instead of creating them itself. It is the practical technique for Dependency Inversion (9.5).

```text
 Without DI:   class OrderService { repo = new SqlOrderRepository(); }       ← hard-wired
 With DI:      class OrderService { constructor(repo) { this.repo = repo; } } ← supplied from outside
```

Three injection styles:

| Style | How | Notes |
| --- | --- | --- |
| **Constructor injection** | Dependencies are constructor parameters | Preferred: the object is always fully initialized, dependencies are visible and can be `readonly` |
| **Setter / property injection** | Dependencies set after construction | For optional dependencies; the object can exist half-configured |
| **Method injection** | Passed to the one method that needs them | For per-call dependencies |

**Composition root.** All wiring happens in **one place near the program's entry point**, where the object graph is built. No other code creates its collaborators.

```js
// plain JavaScript: manual DI
class UserRepository {
  constructor(db) { this.db = db; }
}
class UserService {
  constructor(userRepo, mailer) { this.userRepo = userRepo; this.mailer = mailer; }
}

// composition root
const db = { name: "pg-pool" };
const mailer = { send: () => {} };
const userService = new UserService(new UserRepository(db), mailer);
console.log(userService.userRepo.db.name);     // pg-pool
```

**A tiny DI container** shows what frameworks do for you:

```js
class Container {
  #factories = new Map();
  #singletons = new Map();

  register(name, factory, { singleton = true } = {}) {
    this.#factories.set(name, { factory, singleton });
  }
  resolve(name) {
    const entry = this.#factories.get(name);
    if (!entry) throw new Error(`no registration for "${name}"`);
    if (!entry.singleton) return entry.factory(this);          // a new object every time ("transient")
    if (!this.#singletons.has(name)) this.#singletons.set(name, entry.factory(this));
    return this.#singletons.get(name);                         // one shared object
  }
}

class Repo { constructor(db) { this.db = db; } }
class Service { constructor(repo) { this.repo = repo; } }

const c = new Container();
c.register("db", () => ({ name: "pg-pool" }));
c.register("repo", (c) => new Repo(c.resolve("db")));
c.register("service", (c) => new Service(c.resolve("repo")));

console.log(c.resolve("service").repo.db.name);                // pg-pool
console.log(c.resolve("service") === c.resolve("service"));    // true: singleton lifetime
```

A circular dependency (A needs B, B needs A) makes this recurse forever; real containers detect it and throw an error. It is also a design smell: extract the shared part into a third class.

**C# (ASP.NET Core has a built-in container):**

```csharp
builder.Services.AddSingleton<IClock, SystemClock>();           // one instance for the whole app
builder.Services.AddScoped<IUserRepository, SqlUserRepository>(); // one per HTTP request
builder.Services.AddTransient<IReportBuilder, ReportBuilder>();    // new each time it is requested

public class UserService
{
    private readonly IUserRepository _repo;
    public UserService(IUserRepository repo) => _repo = repo;   // injected automatically
}
```

| Lifetime | Meaning | Typical use |
| --- | --- | --- |
| **Singleton** | One instance per application | Config, caches, HTTP client factories |
| **Scoped** | One instance per request/scope | Database context, unit of work |
| **Transient** | New instance each time | Lightweight stateless helpers |

Pitfall: **captive dependency**, a singleton holding a scoped object, which then lives far longer than intended (a database context shared across requests).

**Python and C++:** pass collaborators through constructors (as in doc 9, 9.5). Python usually needs no container; C++ uses constructor-injected references or `std::unique_ptr`/`std::shared_ptr` with interfaces.

**Service Locator (an anti-pattern):** code that calls `container.resolve("repo")` **inside** its methods hides its dependencies and is hard to test. Resolve dependencies only at the composition root.

**Benefit:** swap implementations (real vs fake gateway) without touching the class: the foundation of testable code.

---

## 10.4.4 Unit of Work

**Intent:** keep track of everything changed during a business transaction and write all changes **together, atomically** (all succeed or none do).

**Problem:** placing an order decrements stock **and** inserts an order. If the second step fails after the first succeeded, the data is inconsistent. A Unit of Work gives one transaction to all repositories used by a use case. This is the ACID *atomicity* property in application code (doc 13).

```text
 Service ─► UnitOfWork ─┬─► ProductRepository ─┐
                        └─► OrderRepository  ──┴─► same connection / one transaction
            commit() on success, rollback() on any exception
```

```python
import sqlite3
from dataclasses import dataclass

@dataclass
class Product:
    id: int
    stock: int

class ProductRepository:
    def __init__(self, conn): self.conn = conn
    def get(self, product_id):
        row = self.conn.execute("SELECT id, stock FROM products WHERE id = ?", (product_id,)).fetchone()
        return Product(*row) if row else None
    def save(self, product):
        self.conn.execute("UPDATE products SET stock = ? WHERE id = ?", (product.stock, product.id))

class OrderRepository:
    def __init__(self, conn): self.conn = conn
    def add(self, product_id, qty):
        cur = self.conn.execute("INSERT INTO orders(product_id, qty) VALUES (?, ?)", (product_id, qty))
        return cur.lastrowid

class UnitOfWork:
    """One database transaction shared by all repositories."""
    def __init__(self, conn): self.conn = conn
    def __enter__(self):
        self.products = ProductRepository(self.conn)
        self.orders = OrderRepository(self.conn)
        return self
    def __exit__(self, exc_type, exc, tb):
        if exc_type is None:
            self.conn.commit()
        else:
            self.conn.rollback()
        return False                                  # never swallow the exception

class OrderService:                                   # service layer: one use case = one unit of work
    def __init__(self, uow_factory): self.uow_factory = uow_factory
    def place_order(self, product_id, qty):
        with self.uow_factory() as uow:
            product = uow.products.get(product_id)
            if product is None:
                raise LookupError("no such product")
            if product.stock < qty:
                raise ValueError("insufficient stock")
            product.stock -= qty
            uow.products.save(product)
            return uow.orders.add(product_id, qty)

conn = sqlite3.connect(":memory:")
conn.executescript("""
CREATE TABLE products(id INTEGER PRIMARY KEY, stock INTEGER NOT NULL CHECK (stock >= 0));
CREATE TABLE orders(id INTEGER PRIMARY KEY AUTOINCREMENT, product_id INTEGER NOT NULL, qty INTEGER NOT NULL);
INSERT INTO products VALUES (1, 5);
""")
service = OrderService(lambda: UnitOfWork(conn))

print("order id:", service.place_order(1, 2))                      # order id: 1
try:
    service.place_order(1, 10)
except ValueError as e:
    print("rejected:", e)                                          # rejected: insufficient stock

# a crash in the middle of a use case: the stock change is rolled back
try:
    with UnitOfWork(conn) as uow:
        p = uow.products.get(1)
        p.stock -= 1
        uow.products.save(p)
        raise RuntimeError("crash before commit")
except RuntimeError:
    pass
print("stock:", conn.execute("SELECT stock FROM products WHERE id = 1").fetchone()[0])   # stock: 3
```

Note: reading the stock and then writing it back is a **read-modify-write** sequence. Two concurrent requests can both read the same stock and oversell. The database must enforce it, using a row lock (`SELECT ... FOR UPDATE`) or a conditional update such as `UPDATE products SET stock = stock - 2 WHERE id = 1 AND stock >= 2` and checking the affected row count (isolation and locking, doc 14).

**Node.js with PostgreSQL (`pg`):**

```js
async function withTransaction(pool, work) {
  const client = await pool.connect();
  try {
    await client.query("BEGIN");
    const result = await work(client);              // all repositories use this one client
    await client.query("COMMIT");
    return result;
  } catch (err) {
    await client.query("ROLLBACK");
    throw err;
  } finally {
    client.release();                               // return the connection to the pool
  }
}
```

**C#:** Entity Framework's `DbContext` **is** a Unit of Work: it tracks changed entities and `SaveChanges()` writes them in one transaction. Its `DbSet<T>` properties act as repositories.

**Trade-offs:** long-lived units of work hold database locks and connections; keep them as short as the use case allows, and never wrap a slow external call (an HTTP request) inside one. A unit of work covers **one database**; across several services you need Sagas (doc 35).

---

## 10.4.5 MVC (Model-View-Controller)

**Intent:** separate an application into three roles so the data, its presentation, and the input handling can change independently (separation of concerns, 9.4).

| Role | Responsibility |
| --- | --- |
| **Model** | The data and business rules; knows nothing about the UI or HTTP |
| **View** | Renders the model for the user (an HTML template, or the JSON representation in an API) |
| **Controller** | Receives input, calls the model, chooses the view |

```text
 request ─► Controller ─► Model ─► Database
               │            │
               │ ◄── data ──┘
               ▼
              View ─► response (HTML or JSON)
```

```js
// Express-style (illustrative; requires `npm install express`)
const express = require("express");
const app = express();

// Model: data + rules
const Book = {
  async findById(id) { /* query the database */ return { id, title: "Dune" }; },
};

// View: how the data is presented
const bookView = (b) => ({ id: b.id, title: b.title });

// Controller: wires them together
app.get("/books/:id", async (req, res) => {
  const book = await Book.findById(req.params.id);
  if (!book) return res.sendStatus(404);
  res.json(bookView(book));
});
```

**Variants:** **MVP** (a Presenter mediates and the view is passive), **MVVM** (a bindable view-model; common in WPF, Angular, Vue), and the SPA split where the browser app is the view and the server exposes only a JSON API. Frameworks built around MVC: ASP.NET Core MVC, Spring MVC, Rails, Django (called "MTV": model, template, view).

**Pitfall:** the **fat controller**: controllers that accumulate business logic. Keep them thin and move rules into services and the model (10.4.2). The full architecture comparison is in doc 19.

---

## 10.4.6 Middleware

**Intent:** process a request through a **pipeline of functions**, each able to act **before** and **after** the next one, handle cross-cutting concerns (logging, authentication, compression, error handling, CORS), or stop the request.

It combines **Chain of Responsibility** (a step may end the request) and **Decorator** (steps wrap the next step).

```text
 request  ──►  logger ──►  auth ──►  rate limit ──►  handler
 response ◄──  logger ◄──  auth ◄──  rate limit ◄──  handler      (the "onion" model)
```

**Express** middleware has the signature `(req, res, next)`; calling `next()` continues the chain and an error-handling middleware takes four arguments:

```js
// Express (illustrative)
app.use((req, res, next) => {                      // runs for every request, in registration order
  console.log(req.method, req.url);
  next();
});
app.use(authenticate);                             // may respond 401 and NOT call next()
app.get("/profile", profileHandler);
app.use((err, req, res, next) => {                 // 4 parameters: this is the error handler
  res.status(500).json({ error: "internal error" });
});
```

(Express 5 forwards a rejected promise from an async handler to the error handler; Express 4 does not, so async handlers there must catch errors and call `next(err)`.)

**The mechanism in a few lines (this is the idea behind Koa's `compose`):**

```js
function compose(middlewares) {
  return function run(ctx) {
    let index = -1;
    function dispatch(i) {
      if (i <= index) return Promise.reject(new Error("next() called multiple times"));
      index = i;
      const fn = middlewares[i];
      if (!fn) return Promise.resolve();               // end of the pipeline
      try {
        return Promise.resolve(fn(ctx, () => dispatch(i + 1)));
      } catch (err) {
        return Promise.reject(err);
      }
    }
    return dispatch(0);
  };
}

const logger = async (ctx, next) => {
  console.log("-> request", ctx.path);
  await next();                                        // everything after this runs on the way back
  console.log("<- response", ctx.status);
};
const auth = async (ctx, next) => {
  if (!ctx.user) { ctx.status = 401; return; }         // stop: do not call next()
  await next();
};
const handler = async (ctx) => { ctx.status = 200; };

const app = compose([logger, auth, handler]);
(async () => {
  await app({ path: "/profile", user: "asha" });       // -> request /profile, then <- response 200
  await app({ path: "/profile" });                     // -> request /profile, then <- response 401
})();
```

**ASP.NET Core:**

```csharp
app.Use(async (context, next) =>
{
    var sw = System.Diagnostics.Stopwatch.StartNew();
    await next(context);                               // call the rest of the pipeline
    Console.WriteLine($"{context.Request.Path} took {sw.ElapsedMilliseconds} ms");
});
```

**Ordering rules of thumb:** error handling and logging first (outermost, so they see everything), then security headers, CORS, authentication, authorization, rate limiting, body parsing, and finally routes. Putting authentication **after** the route it should protect is a classic security bug.

Reverse proxies and API gateways apply the same pipeline idea at the network level (docs 28, 39).

---

## 10.4.7 Event-Driven Architecture (as a pattern)

**Intent:** components communicate by **publishing and reacting to events** (facts about something that happened) instead of calling each other directly.

At the **code level** this is the Observer pattern (10.3.2): `orderService` emits `order.created` and independent handlers react. At the **system level** the same idea is implemented with a message broker between services, which brings durability, retries, replay and independent scaling, but also eventual consistency, duplicate delivery and harder debugging.

| | In-process (Observer) | Across services (event-driven architecture) |
| --- | --- | --- |
| Transport | Function calls in one process | Broker: Kafka, RabbitMQ, SQS (docs 25, 27) |
| Delivery | Immediate, synchronous by default | Asynchronous, at-least-once typically |
| If a consumer is down | Not applicable (same process) | Messages wait in the broker |
| Failure of the process | Pending events lost | Events survive in the broker |
| Consistency | Same transaction possible | Eventual (doc 33) |

Event sourcing, CQRS, delivery guarantees and idempotent consumers are covered in docs 26, 27 and 37.

---

# 10.5 Choosing and Comparing Patterns

## Choose by the Problem You Have

| Problem | Pattern |
| --- | --- |
| Decision about **which class** to create is spread everywhere | Factory / Factory Method |
| Need a **matching set** of objects (all AWS or all local) | Abstract Factory |
| Object has **many optional parts** or needs validation before use | Builder |
| Creating by **copying** a template is cheaper | Prototype |
| Exactly **one** shared instance is needed (rare; prefer injection) | Singleton |
| Third-party interface **does not match** what you need | Adapter |
| Add features **dynamically**, stackable (compression, retries, logging) | Decorator |
| Hide a **messy subsystem** behind a simple interface | Facade |
| **Control access** (lazy, cache, permissions, remote) | Proxy |
| **Tree** of parts and groups treated alike | Composite |
| Two independent **dimensions** of variation | Bridge |
| Swap an **algorithm** at runtime | Strategy |
| Many objects must be **notified** of a change | Observer |
| Need **undo**, queueing, logging of actions | Command |
| Behavior depends on a **lifecycle state** | State |
| Fixed **skeleton**, varying steps | Template Method |
| Request passes through **optional checks** | Chain of Responsibility / Middleware |
| Traverse a collection **without exposing it** | Iterator |
| Keep SQL **out of business code** | Repository |
| Several changes must **commit together** | Unit of Work |
| Business logic needs a place **independent of HTTP** | Service Layer |
| Classes create their own collaborators, **hard to test** | Dependency Injection |

## Patterns That Look Alike

| Group | How to tell them apart |
| --- | --- |
| **Adapter, Decorator, Proxy, Facade** | Adapter **changes** the interface; Decorator **adds** behavior with the same interface; Proxy **controls access** with the same interface; Facade creates a **simpler** interface over many objects |
| **Strategy, State, Bridge** | Same structure (delegation to a swappable object). Strategy: **client picks** an algorithm. State: object **switches itself**. Bridge: **designed up front** to separate abstraction and implementation |
| **Factory Method, Abstract Factory, Builder** | Factory Method: **one product**, subclass decides. Abstract Factory: **a family** of products. Builder: **one complex product** assembled in steps |
| **Chain of Responsibility, Decorator, Middleware** | Decorator: every layer adds and forwards. Chain: a layer may **stop** the request. Middleware: both |
| **Observer vs pub/sub broker** | Observer: subject knows its observers directly, in one process. Broker pub/sub: publisher and subscribers **never know each other** |

## Where These Patterns Reappear in the Roadmap

| Pattern | Larger-scale form | Doc |
| --- | --- | --- |
| Strategy | Load-balancing algorithms, rate-limit algorithms, cache eviction, retry policies | 22, 38, 23, 36 |
| Proxy | Reverse proxy, CDN, service mesh sidecar | 28, 52 |
| Facade | API gateway | 39 |
| Adapter | Anti-corruption layer, ports and adapters (hexagonal architecture) | 19, 31 |
| Observer | Publish/subscribe, event streaming | 25, 26, 27 |
| Command | Job queues, event sourcing, write-ahead log | 25, 26, 49 |
| State | Saga progress, circuit breaker states, order lifecycle | 35, 36 |
| Chain / Decorator | Middleware, filters, gateway plugins | 17, 39 |
| Repository / Unit of Work | Data-access layer, transactions | 13, 19 |
| Singleton (per process) | Connection pools, in-memory caches (and why shared state needs Redis) | 24 |
| Composite | Role hierarchies, permission groups | 18 |
| Builder | Query builders, ORMs | 11, 17 |
| Abstract Factory | Cloud-provider abstraction, dev/prod environment switching | 43, 46 |
| Iterator | Cursor pagination, streaming results | 17 |

## Common Misuse (Anti-Patterns)

| Anti-pattern | Description | Better |
| --- | --- | --- |
| **Singleton everywhere** | Global state in disguise | One instance created at startup and injected |
| **Service Locator** | Classes pull dependencies from a global registry | Constructor injection |
| **God object / fat controller** | One class knows and does everything | Separation of concerns, SRP |
| **Golden hammer** | Applying a favorite pattern to every problem | Choose by the problem |
| **Speculative generality** | Factories and strategies for variations that never come | YAGNI; refactor when the second variant appears |
| **Pattern soup** | Several layers of indirection for trivial logic | KISS |

---

# Key Takeaways

- A pattern is a **named solution to a recurring problem**; it gives teams shared vocabulary. Apply a pattern when you feel its specific pain, not in advance.
- **Creational:** Factory hides *which class*, Abstract Factory yields *consistent families*, Builder assembles *complex objects stepwise*, Prototype *copies*, Singleton gives *one instance* (use sparingly; it is per process, not per system).
- **Structural:** Adapter (**convert**), Decorator (**add**), Proxy (**control access**), Facade (**simplify**), Composite (**trees treated uniformly**), Bridge (**$m+n$ classes instead of $m\times n$**).
- **Behavioral:** Strategy (swap algorithms), Observer (notify many), Command (requests as objects, undo, queues), State (behavior follows state), Template Method (fixed skeleton), Chain of Responsibility (pipeline that can stop), Iterator (uniform traversal, lazily).
- **Backend:** Repository isolates data access, Service Layer owns use cases, **Unit of Work** makes several writes atomic, **Dependency Injection** wires objects at one composition root, MVC separates input, rules and presentation, **Middleware** is a pipeline for cross-cutting concerns, and event-driven architecture scales Observer across services.
- Many patterns collapse into language features (first-class functions, events, generators, modules); use the simplest form that expresses the intent.
