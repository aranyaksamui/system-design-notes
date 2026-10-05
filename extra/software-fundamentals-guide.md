# The Fundamentals Behind Every Tech Stack

> **The tools change. The fundamentals stay.**

Frameworks, languages, and cloud services come and go. But every serious software system, whether it's a banking app, a chat app, or an AI assistant, must answer the same set of questions. If you learn those questions, you can understand *any* stack, because you'll recognize which question each new tool is trying to answer.

This guide walks through those fundamentals in order. For each one you get:

1. **What it is**: a plain-language definition
2. **A diagram**: a generalized picture (no code)
3. **An example**: something concrete
4. **The fundamental question**: the question every builder must answer
5. **Gaps filled in**: the important ideas that usually get skipped
6. **Connects to**: how it links to the other fundamentals

---

## Map of this guide

| Part | Sections | What it covers |
|---|---|---|
| **Part 1: The Core Eight** | 1. Data · 2. State · 3. Communication · 4. Storage · 5. Computation · 6. Concurrency · 7. Failure · 8. Security | The building blocks every system is made of |
| **Part 2: What Makes a System Good** | 9. Scalability · 10. Performance · 11. Observability · 12. Consistency · 13. Abstraction & Layering · 14. Interfaces & API Design · 15. Deployment & Environments · 16. Testing · 17. Cost · 18. Trade-offs | The qualities, structures, and practices that decide whether those building blocks work well in the real world |
| **Part 3: Putting It Together** | 19. The Connection · 20. The Bigger Idea · 21. Glossary · 22. Study Path | How everything fits, and how to keep learning |

---

## How to use this guide

Think of building software like designing a **city**:

| City | Software |
|---|---|
| Residents, goods, mail | **Data** |
| What each resident currently remembers or owns | **State** |
| Roads, phone lines, postal service | **Communication** |
| Warehouses, archives, filing cabinets | **Storage** |
| Factories and workshops | **Computation** |
| Rush-hour traffic | **Concurrency** |
| Power outages, floods, accidents | **Failure** |
| Locks, ID cards, police | **Security** |
| The city growing from a town to a metropolis | **Scalability** |
| How quickly a bus gets you across town | **Performance** |
| Traffic cameras and the control-room dashboard | **Observability** |
| Every map and clock in the city showing the same thing | **Consistency** |
| Flipping a light switch without knowing how the power grid works | **Abstraction & Layering** |
| Standard plugs and sockets that fit everywhere | **Interfaces & API Design** |
| Building and opening new roads without shutting down the city | **Deployment & Environments** |
| Inspecting a bridge before cars drive on it | **Testing** |
| The city budget | **Cost** |
| Choosing a park *or* a parking lot for the same plot of land | **Trade-offs** |

Keep this city picture in mind. Each section below will refer back to it.

### One running example

To make everything concrete, we'll keep returning to a simple app: **an online photo-sharing app** where people sign up, upload photos, like each other's photos, and get AI-generated captions.

---

## 0. The Big Picture First

Before the details, here is the structure of a system in one view:

```
                         ┌───────────┐
                         │   USER    │
                         └─────┬─────┘
                               │
     ┌─────────────────────────┼─────────────────────────┐
     │  CROSS-CUTTING CONCERNS (touch every layer):      │
     │  SECURITY  ·  FAILURE  ·  CONCURRENCY             │
     │                         │                         │
     │                  ┌──────▼──────┐                  │
     │                  │  FRONTEND   │  what the user sees
     │                  └──────┬──────┘                  │
     │                         │                         │
     │                  ┌──────▼──────┐                  │
     │                  │COMMUNICATION│  how parts talk   │
     │                  │ (interfaces)│                  │
     │                  └──────┬──────┘                  │
     │                         │                         │
     │                  ┌──────▼──────┐                  │
     │                  │   BACKEND   │  the "brain"      │
     │                  └───┬─────┬───┘                  │
     │                      │     │                      │
     │              ┌───────▼┐   ┌▼────────────┐         │
     │              │ STATE  │   │ COMPUTATION │         │
     │              └───┬────┘   └──────┬──────┘         │
     │                  │               │                │
     │              ┌───▼────┐     ┌────▼───┐            │
     │              │STORAGE │     │ RESULT │            │
     │              └────────┘     └────────┘            │
     └───────────────────────────────────────────────────┘
```

Read it top to bottom: a **user** does something → the **frontend** captures it → it travels via **communication** → the **backend** receives it → the backend either **remembers** something (state → storage) or **works something out** (computation → result). Wrapped around all of it are three concerns that apply everywhere: **security, failure, and concurrency**.

That diagram shows what a system is *made of*. A second view shows how a system is *judged, shaped, and run*:

```
   ┌─────────────────────────────────────────────────────────────┐
   │  DECISIONS:   TRADE-OFFS  (every choice gives something up) │
   ├─────────────────────────────────────────────────────────────┤
   │  QUALITIES:   SCALABILITY · PERFORMANCE · CONSISTENCY · COST │
   │               (how well does the system behave?)            │
   ├─────────────────────────────────────────────────────────────┤
   │  STRUCTURE:   ABSTRACTION & LAYERING · INTERFACES / APIs    │
   │               (how is the system organized?)                │
   ├─────────────────────────────────────────────────────────────┤
   │  THE CORE:    DATA · STATE · COMMUNICATION · STORAGE ·      │
   │               COMPUTATION · CONCURRENCY · FAILURE · SECURITY│
   ├─────────────────────────────────────────────────────────────┤
   │  PRACTICES:   TESTING ──▶ DEPLOYMENT ──▶ OBSERVABILITY      │
   │               (how do we change and run it safely?)         │
   │                     ▲                          │            │
   │                     └──────── feedback ◀───────┘            │
   └─────────────────────────────────────────────────────────────┘
```

The core eight sit in the middle. Structure organizes them, qualities measure them, practices keep them running safely, and trade-offs are the decisions you make along the way. Now let's zoom into each piece.

---

# PART 1: THE CORE EIGHT

---

## 1. DATA

### What it is

**Data is the information your application cares about.** Every app is, at heart, a machine that *creates, transforms, retrieves, and stores* information. If you remove the data, there's nothing left for the software to do.

Data comes in many shapes:

- **User profiles**: name, email, preferences
- **Transactions**: payments, orders, transfers
- **Messages**: chats, emails, notifications
- **Images / files / media**: photos, videos, documents
- **AI inputs and outputs**: prompts, model responses, embeddings

### Diagram

```
          User Profiles        Transactions
                 \                /
                  \              /
   Messages ─────────  DATA  ─────────  Images
                  /              \
                 /                \
            AI Inputs          AI Outputs

        (create · transform · retrieve · store)
```

### Example

In our photo app, the data includes:

- A **user profile** (username, email, bio)
- A **photo** (the image file plus its title, upload time, and owner)
- A **like** (which user liked which photo, and when)
- An **AI caption** (text the model generated for a photo)

Notice that even in this tiny app there are different *kinds* of data, with different sizes, shapes, and lifetimes. A like is tiny. A photo is large. A profile changes rarely. A like count changes constantly.

### The fundamental question

> **What data does the system need, and how should it be represented?**

"Represented" means: what fields does it have, how are the pieces related to each other, and what format is it in? Designing this is called **data modeling**, and it is one of the most important and most underrated skills in software. A badly modeled piece of data causes problems everywhere downstream.

### Gaps filled in

- **Data has relationships.** A photo *belongs to* a user. A like *connects* a user to a photo. Thinking in terms of "things and how they connect" is the heart of data modeling.
- **Data has a shape (structure).** Some data is neat and tabular (rows and columns, like a spreadsheet). Some is flexible and nested (like a form with optional sections). Some is unstructured (a photo, a PDF, free text). The shape influences where you store it (see Storage).
- **Data has a lifecycle.** It's created, read, updated, and eventually deleted (often shortened to **CRUD**). Ask: how long should this data live? Who is allowed to delete it? Do laws require you to delete it on request?
- **Data has a source of truth.** When the same fact exists in several places, one of them must be the official version. More on this in State.
- **Data has a volume.** Ten thousand records and ten billion records are different problems, even if the data model is identical.

### Connects to

- **Interfaces & API Design (14):** the shape of your data becomes the shape of your API contracts.
- **Scalability (9) and Cost (17):** the amount of data you keep drives how big and how expensive the system gets.
- **Testing (16):** realistic test data is one of the hardest and most valuable parts of testing.

---

## 2. STATE

### What it is

**State is information a system needs to *remember* in order to behave correctly right now.**

Data is *what exists*. State is *what is currently true and must be remembered*. A useful way to feel the difference:

- Data: "A shopping cart can hold items."
- State: "*This* shopping cart, at this moment, holds 3 items."

If a system had no state, it would treat every interaction as if it had never met you before.

### Diagram: where can state live?

```
   ┌──────────────────────────────────────────┐
   │ BROWSER                                  │  closest to the user
   │ localStorage · cookies · UI/React state  │  disappears easily
   └───────────────────┬──────────────────────┘
                       ▼
   ┌──────────────────────────────────────────┐
   │ APPLICATION                              │
   │ in-memory variables · cache              │  lost if the app restarts
   └───────────────────┬──────────────────────┘
                       ▼
   ┌──────────────────────────────────────────┐
   │ SERVER                                   │
   │ sessions · server-side state             │  shared across requests
   └───────────────────┬──────────────────────┘
                       ▼
   ┌──────────────────────────────────────────┐
   │ DATABASE                                 │  survives restarts
   │ persistent · source of truth             │  the "official record"
   └──────────────────────────────────────────┘

   Higher up = faster and closer, but more fragile.
   Lower down = slower and farther, but more durable and trustworthy.
```

### Example

In our photo app:

- Whether a **menu is open or closed** is state living in the **browser** (nobody else cares, and it's fine to lose it).
- The fact that **you're logged in** is state tracked by the **server** (a session) and remembered by the browser (a cookie).
- The **list of photos you've liked** is state that must live in the **database**, because you'd be furious if it vanished.

Same app, three pieces of state, three different homes.

### The fundamental question

> **Where should this state live, and who owns it?**

"Who owns it" means: which part of the system is the one authoritative keeper of this information? If two places both think they own the same fact, they will eventually disagree, and that causes bugs that are very hard to track down.

### Gaps filled in

- **Stateful vs. stateless.** A *stateless* component remembers nothing between requests. Each request carries everything needed. Stateless parts are easy to copy and replace. A *stateful* component remembers things, so it is harder to scale and recover. Good designs push state into a small number of well-chosen places.
- **Single source of truth.** For any piece of state, pick one official copy. Other copies (caches, browser copies) are just *convenient shortcuts* that can be out of date.
- **Staleness.** Copies of state can get out of date. If you "like" a photo and the like count on another person's screen doesn't update for 10 seconds, that's a stale copy. Sometimes that's acceptable; sometimes (your bank balance) it isn't.
- **State vs. Storage.** State is *the concept* (what must be remembered). Storage (section 4) is *the mechanism* (where and how it's physically kept).

### Connects to

- **Scalability (9):** stateless components are the easiest to multiply; state is what makes scaling hard.
- **Consistency (12):** the moment state has more than one copy, you must decide how closely the copies must agree.
- **Testing (16):** state is what makes tests tricky, because a test must set up a known starting state and check the resulting one.
- **Deployment (15):** state in memory is lost when you restart or replace a server, which matters every time you release a change.

---

## 3. COMMUNICATION

### What it is

**Communication is how the different parts of a system send information to each other.** A modern app is almost never one block. It's a frontend, a backend, a database, maybe several smaller services, and external providers (payments, email, AI models). These parts are often on different machines, so they need agreed-upon ways to talk.

### Diagram

```
   Synchronous (ask and wait for the answer):

   ┌──────────┐    ┌─────┐    ┌─────────┐    ┌──────────┐
   │ Frontend │ ─▶ │ API │ ─▶ │ Backend │ ─▶ │ Database │
   └──────────┘    └─────┘    └─────────┘    └──────────┘
        ◀──────────────── response comes back ───────────

   Asynchronous (drop off a message and move on):

   ┌───────────┐    ┌─────────┐    ┌───────────┐
   │ Service A │ ─▶ │  QUEUE  │ ─▶ │ Service B │
   └───────────┘    └─────────┘    └───────────┘
   (A doesn't wait. B picks the message up whenever it's ready.)
```

### Common mechanisms (in plain words)

| Mechanism | Plain-language picture | Typical use |
|---|---|---|
| **HTTP** | Send a request, wait for one response, like ordering at a counter | Loading a page, calling an API |
| **WebSockets** | A phone call that stays open so either side can speak any time | Live chat, live scores, multiplayer |
| **gRPC** | A fast, strict, efficient way for services to call each other | Internal service-to-service calls |
| **Queues** | A mailbox: leave a message, the receiver reads it later | Background jobs, smoothing out traffic spikes |
| **Events** | A broadcast: "something happened," and anyone interested reacts | Notifying many parts of the system at once |

### Example

You upload a photo in our app:

1. The **frontend** sends the image to the **API** (synchronous: you wait to see "uploaded!").
2. The backend saves it, then drops a message in a **queue**: "please generate a caption for photo #123."
3. A separate **caption service** picks that message up later, runs the AI model, and saves the result.

You didn't have to wait for the slow AI step. That's the benefit of asynchronous communication.

### The fundamental question

> **How should these components communicate?**

### Gaps filled in

- **Synchronous vs. asynchronous** is the single most important distinction here. Synchronous = simple but you wait, and if the other side is down, you're stuck. Asynchronous = more flexible and resilient but harder to reason about.
- **Request/response vs. push.** Sometimes the client asks (pull). Sometimes the server tells the client unprompted (push, as with WebSockets).
- **Networks are slow and unreliable compared with talking inside one program.** Every time you cross a network boundary, you pay a cost (time) and take a risk (failure). This is why Communication and Failure are deeply linked.
- **Latency and bandwidth.** *Latency* is how long one message takes. *Bandwidth* is how much can be sent per second. Different problems (see Performance).
- **Communication needs an agreed contract.** Both sides must agree on what a message means. That agreement is an API (see Interfaces & API Design, section 14).

### Connects to

- **Failure (7):** every message across a network can be lost, delayed, or duplicated.
- **Performance (10):** network hops are often the biggest chunk of response time.
- **Consistency (12):** messages arriving late or out of order are a main source of inconsistent data.
- **Interfaces & API Design (14):** the contract that makes communication understandable and safe to change.
- **Observability (11):** following one request as it hops across many components requires tracing.

---

## 4. STORAGE

### What it is

**Storage is where data is physically kept, and for how long.** Different kinds of data have different needs: speed, size, durability, cost, and structure. So systems use *several kinds* of storage side by side, rather than one place for everything.

### Diagram: the storage ladder

```
   FASTEST / MOST EXPENSIVE / LEAST DURABLE
   ▲
   │   ┌────────────────────────────────────────────┐
   │   │ MEMORY        RAM, in-process variables    │  temporary; lost
   │   │               (blazing fast)               │  on restart
   │   ├────────────────────────────────────────────┤
   │   │ CACHE         Redis, Memcached             │  frequently used
   │   │               (fast reads for hot data)    │  data, kept nearby
   │   ├────────────────────────────────────────────┤
   │   │ DATABASE      PostgreSQL, MongoDB, MySQL   │  structured, durable
   │   │               (the main application data)  │  application data
   │   ├────────────────────────────────────────────┤
   │   │ OBJECT        S3, GCS, Cloudflare R2       │  files, media,
   │   │ STORAGE       (huge and cheap)             │  large blobs
   ▼   └────────────────────────────────────────────┘
   SLOWEST / CHEAPEST / MOST DURABLE
```

### Example

In our photo app:

- The **user profile** (username, email) lives in a **database** (structured, must be accurate and durable).
- The **uploaded image file** lives in **object storage** (large, cheap, doesn't need rich querying).
- The **most-viewed photos' info** is kept in a **cache** so the home page loads instantly without bothering the database every time.
- The **current request's temporary calculations** live in **memory** and vanish when finished.

### The fundamental question

> **What should be stored, where, and for how long?**

### Gaps filled in

- **The trade-off triangle: speed, cost, durability.** You can't have all three at the maximum. Memory is fast but forgets. Object storage remembers cheaply but is slower. Pick per data type.
- **Durability** means "will it still be there after a crash or power cut?" Memory: no. Database: yes (when set up properly). Object storage: extremely yes.
- **Backups and replication.** A single copy of anything important is a risk. *Replication* keeps live copies on several machines; *backups* are historical snapshots you can restore from. They solve different problems.
- **Caches can lie.** Because a cache is a copy, it can become outdated. Deciding *when to refresh or throw away cached data* ("cache invalidation") is famously one of the hard problems in computing.
- **Different database styles exist.** Broadly: *relational* (tables with strict structure and relationships) vs. *non-relational/document* (flexible, nested records). Neither is "better," they fit different data shapes.
- **Retention.** "For how long?" matters legally and financially. Storing everything forever costs money and can create privacy risk.

### Connects to

- **Scalability (9):** when one database machine can't cope, you split or copy the data across several, which brings new complexity.
- **Performance (10):** where data lives largely determines how fast it can be read.
- **Consistency (12):** every copy (replica, cache) is a chance for disagreement.
- **Cost (17):** storage is billed by amount, by speed tier, and by how often you read or move the data.
- **Failure (7):** backups and replicas are your safety net when storage breaks.

---

## 5. COMPUTATION

### What it is

**Computation is the work the software does *with* data**: the actual thinking and processing. Data sitting still is just a pile of information; computation is what turns it into something useful.

Typical kinds of computation:

- **Calculate**: totals, taxes, scores
- **Search**: find matching items
- **Sort**: order results (newest first, cheapest first)
- **Transform**: convert from one form to another (resize an image, reformat a date)
- **Predict**: make an estimate (recommendations, AI models)
- **Render**: produce what the user sees (draw a web page, a game frame)

### Diagram

```
   Six kinds of work:
   ┌───────────┐ ┌────────┐ ┌──────┐
   │ Calculate │ │ Search │ │ Sort │
   └───────────┘ └────────┘ └──────┘
   ┌───────────┐ ┌─────────┐ ┌────────┐
   │ Transform │ │ Predict │ │ Render │
   └───────────┘ └─────────┘ └────────┘

   A typical AI application's flow:

   ┌─────────┐      ┌──────────────────────┐      ┌─────────┐
   │  INPUT  │ ───▶ │ MODEL / COMPUTATION  │ ───▶ │ OUTPUT  │
   └─────────┘      └──────────────────────┘      └─────────┘
```

### Example

When a user uploads a photo and the app writes a caption:

- **Input**: the image
- **Computation**: the app first *transforms* it (shrinks it to a standard size), then an AI model *predicts* a caption
- **Output**: a sentence of text, saved and shown to the user

### The fundamental question

> **Where should the computation happen, and how expensive is it?**

### Gaps filled in

- **"Where" has real options.** Computation can happen on the *user's device* (browser or phone), on *your server*, or on *specialized machines* (e.g., machines with GPUs for AI). Doing it on the user's device is free for you and instant for them, but you can't trust it (see Security) and weak devices struggle. Doing it on your server is controlled but costs you money.
- **"Expensive" means several things:** time (how long it takes), memory (how much it uses), money (cloud bills, especially for AI), and energy.
- **Cheap now vs. cheap later.** Sometimes you can do expensive work *ahead of time* (precompute) or *once and reuse* (cache the result), instead of redoing it for every user.
- **Efficiency matters more as scale grows.** A slow method that's fine for 10 users can collapse at 10 million. This is why algorithms and data structures are a fundamental skill: they're about choosing *cheaper ways to compute*.
- **Foreground vs. background work.** Quick work happens while the user waits. Slow work should be pushed to the background (via the queues from Communication).

### Connects to

- **Performance (10):** computation time is a major part of how fast the system feels.
- **Scalability (9):** more users means more computation; you either make it cheaper or add more machines to do it.
- **Cost (17):** compute is usually billed by time and power used, and AI computation can be the largest bill of all.
- **Concurrency (6):** doing computation for many users at once is exactly the concurrency challenge.

---

## 6. CONCURRENCY

### What it is

**Concurrency is what happens when many things occur at overlapping times.** Real systems never serve just one person politely in sequence. Thousands of users, background jobs, and scheduled tasks all hit the system at once.

Doing many things at once is powerful, but it creates a unique family of problems.

### Diagram

```
   User A ──┐
   User B ──┤
   User C ──┼──▶  BACKEND  (handling all of them simultaneously)
   User D ──┘

   Which can cause:
   ┌──────────────────┐  ┌──────────────┐
   │ Race conditions  │  │  Deadlocks   │
   └──────────────────┘  └──────────────┘
   ┌───────────────────┐ ┌───────────────┐
   │Resource contention│ │ Ordering issues│
   └───────────────────┘ └───────────────┘

   Tools for taming it:
   locks · queues · atomic operations · transactions
```

### The four classic problems, in plain words

**Race condition.** Two operations "race" to change the same thing, and the final result depends on who happens to win.
> *Example:* A concert has **1 ticket left**. Two people click "Buy" at the exact same moment. Both see "1 available." Both purchases go through. You've sold one ticket twice.

**Deadlock.** Two operations each hold something the other needs, and both wait forever.
> *Example:* Person A holds the fork and needs the knife. Person B holds the knife and needs the fork. Neither will let go. Nobody eats.

**Resource contention.** Many operations compete for the same limited resource, so they slow each other down.
> *Example:* A hundred people trying to use one checkout counter.

**Ordering issues.** Things happen in a different order than intended.
> *Example:* In a chat app, the reply "Yes!" shows up *before* the question "Want pizza?"

### Example

In our photo app, a popular photo is "liked" by 500 people in the same second. If the like counter is handled carelessly, two people might both read "count = 100", both add one, and both write back "101", so one like is lost. The fix is to make the update **atomic**: it happens as one indivisible step that nobody can interrupt halfway.

### The fundamental question

> **What happens when multiple operations interact at the same time?**

### The toolbox, in plain words

| Tool | Plain picture |
|---|---|
| **Lock** | A "do not disturb" sign on the bathroom door: only one at a time |
| **Queue** | A line: everyone gets served, one by one, in order |
| **Atomic operation** | An action that is all-or-nothing and can't be interrupted midway |
| **Transaction** | A bundle of steps treated as one: either every step succeeds, or none of them count (like a bank transfer: money leaves one account *and* arrives in the other, or neither happens) |

### Gaps filled in

- **Concurrency vs. parallelism.** *Concurrency* is about *managing* many tasks that overlap (like one chef juggling several dishes). *Parallelism* is about *literally doing* many at the same moment (several chefs). You can have one without the other.
- **Shared mutable state is the root of most pain.** If many things can *change* the same piece of data, you get trouble. Reducing sharing (or making data unchangeable) removes whole categories of bugs.
- **Concurrency bugs are the sneakiest bugs.** They may appear once in a million runs, only under heavy load, and vanish when you try to reproduce them.
- **Idempotency.** An operation is *idempotent* if doing it twice has the same effect as doing it once ("set the light to ON" is idempotent; "flip the light switch" is not). This is a lifesaver when messages get retried or duplicated.

### Connects to

- **Consistency (12):** concurrency is what puts data's agreement at risk; consistency is the set of promises that keep it safe.
- **Scalability (9):** adding more machines multiplies the number of things happening at once.
- **Testing (16):** concurrency bugs are among the hardest to catch, which calls for special testing techniques.
- **Performance (10):** locks and queues protect correctness, but they can also slow things down.

---

## 7. FAILURE

### What it is

**Failure is the guarantee that things will go wrong.** Not *if*, but *when*. A reliable system isn't one that never breaks. It's one designed so that breakage is expected, detected, and recovered from.

Things that will fail:

- Networks drop
- Servers crash
- Dependencies (other services) time out
- Databases become unavailable
- Users send unexpected, weird, or broken input

### Diagram

```
   Things that go wrong:
   [Network fails] [Server crash] [Timeout] [DB unavailable] [Bad input]
                          │
                          ▼
                 ┌──────────────────┐
                 │  What can fail?  │      (anticipate)
                 └────────┬─────────┘
                          ▼
                 ┌──────────────────────┐
                 │ How will we detect   │   (notice quickly)
                 │ it?                  │
                 └────────┬─────────────┘
                          ▼
                 ┌──────────────────────┐
                 │ How will we recover? │   (respond well)
                 └──────────────────────┘

   Recovery tools: retry · fallback · circuit breaker · graceful degradation
```

### Recovery tools, in plain words

| Tool | Plain picture |
|---|---|
| **Retry** | The call dropped, so try dialing again (ideally waiting a bit longer each time) |
| **Fallback** | The main option failed, so use a backup (show saved data instead of live data) |
| **Circuit breaker** | Like a home fuse: if a service keeps failing, *stop calling it for a while* so it doesn't drag everything down |
| **Graceful degradation** | Lose a feature, not the whole product (if the AI caption service is down, uploading still works, and captions come later) |

### Example

In our photo app, the AI caption service goes down:

- **Bad design:** The upload fails with a scary error, and users can't post at all.
- **Good design:** The upload succeeds. The caption step is retried in the background. If it still fails, the photo appears with no caption and a quiet note, and the team is alerted.

The user's core goal still works. That is graceful degradation.

### The fundamental question

> **What should the system do when something goes wrong?**

### Gaps filled in

- **Retries can cause harm.** If a "charge the credit card" request times out and you retry, you might charge twice. This is why *idempotency* (from Concurrency) matters so much for failure handling.
- **Timeouts are essential.** Without a limit on how long you'll wait, one slow dependency can freeze everything waiting on it.
- **Partial failure is the hardest kind.** It's easy to handle "everything works" or "everything is dead." The nasty case is when *some* parts work and some don't, or you don't know whether an action succeeded.
- **Redundancy.** Having more than one of something important (servers, databases) so one failure isn't fatal. Avoid "single points of failure."
- **Failing safely.** When unsure, fail in the *safe* direction (a door lock that fails closed vs. a fire exit that fails open depends on what's being protected).
- **Detecting failure needs visibility.** "How will we detect it?" is answered by Observability (section 11): logs, metrics, and alerts.

### Connects to

- **Observability (11):** you can't recover from what you can't see.
- **Testing (16):** failure handling that has never been tested usually doesn't work when it's finally needed.
- **Deployment (15):** a bad release is a failure too, which is why safe rollouts and rollbacks exist.
- **Scalability (9):** more machines means more things that can break, so failures become routine, not rare.
- **Consistency (12):** during a failure (like a network split), a system must choose between staying available and staying perfectly in agreement.

---

## 8. SECURITY

### What it is

**Security is making sure only the right people can do only the right things with the right data, and that bad actors can't break or abuse the system.** The guiding mindset: **never trust input by default.**

Data arrives from many places: users, browsers, APIs, other services, third parties. Any of these might be mistaken, malicious, or compromised. So everything coming in must be checked.

### Diagram

```
   Users      Browsers      APIs      3rd Parties
      \           |           |           /
       \          |           |          /
        ▼         ▼           ▼         ▼
   ┌────────────────────────────────────────────────┐
   │              SECURITY LAYER                    │
   │ validate · sanitize · authenticate · authorize │
   └───────────────────────┬────────────────────────┘
                           ▼
        ┌──────────┐ ┌──────────┐ ┌──────────────┐
        │  Authn   │ │  Authz   │ │  Encryption  │
        │who are   │ │what can  │ │data in       │
        │you?      │ │you do?   │ │transit+rest  │
        └──────────┘ └──────────┘ └──────────────┘

        ┌─────────────────────┐   ┌────────────────────────┐
        │ Frontend validation │   │ Backend validation     │
        │ improves UX         │   │ protects the system    │
        └─────────────────────┘   └────────────────────────┘
```

### Core concerns, in plain words

| Concern | Plain meaning | Analogy |
|---|---|---|
| **Authentication (Authn)** | *Who are you?* Proving identity | Showing your ID at the door |
| **Authorization (Authz)** | *What are you allowed to do?* | Your ID says you're a guest, so you can enter the lobby but not the vault |
| **Validation** | Checking input is well-formed and safe | Checking a package isn't ticking before you accept it |
| **Encryption** | Scrambling data so only the intended reader can understand it, both while traveling ("in transit") and while sitting stored ("at rest") | Sending a letter in a locked box instead of a postcard |
| **Secrets** | Passwords, API keys, tokens that must be protected and never exposed | The master key: never taped to the front door |

> **Authn vs. Authz** is the pair people confuse most. Authn = *proving who you are*. Authz = *what that identity may do.* You need both.

### Example

In our photo app, the signup form checks that your email *looks* like an email. That's **frontend validation**, and it exists to give you quick, friendly feedback. But a malicious person can skip your form entirely and send fake requests directly to the server. So the **backend must validate again**. Frontend validation is for *user experience*. Backend validation is for *protection*. Never rely on the first alone.

Another example: you can edit **your own** profile but not someone else's. The system must check *every time* that the person making the request owns that profile (authorization).

### The fundamental question

> **Who can access this, and what are they allowed to do?**

### Gaps filled in

- **Anything running on the user's device can be tampered with.** Their browser, their phone, their rules. This is *why* the backend must be the final judge.
- **Principle of least privilege.** Give every person and every component the *minimum* access it needs. If it gets compromised, the damage stays small.
- **Defense in depth.** Don't rely on one wall. Layer several protections so one failure isn't catastrophic.
- **Common attack ideas (conceptually):** tricking the system into treating input as instructions (injection), stealing a logged-in session, guessing weak passwords, and overwhelming a service with traffic. You don't need exploit details to understand the lesson: *treat all outside input as untrusted.*
- **Privacy and compliance.** Security also includes respecting what data you're allowed to collect, keep, and share (and, increasingly, what you may feed into AI systems).
- **AI-specific note.** AI features add a new kind of untrusted input: text a user (or a web page) gives the model may try to manipulate its behavior. The same mindset applies: validate, limit permissions, and don't let untrusted text control sensitive actions.
- **Security is a design habit, not a final step.** It's much cheaper to build in from the start than to bolt on later.

### Connects to

- **Interfaces & API Design (14):** every API is a doorway, so each one needs authentication, authorization, and validation.
- **Deployment (15):** secrets such as passwords and keys must be handled differently in each environment and never stored in plain view.
- **Observability (11):** security logs ("who did what, and when") let you detect and investigate abuse.
- **Testing (16):** security should be tested deliberately, including attempts to misuse the system.
- **Scalability (9):** a popular system attracts more attacks, and an overwhelming flood of traffic is itself a security problem.

---

# PART 2: WHAT MAKES A SYSTEM GOOD

The core eight describe what a system is *made of*. This part describes how to make it work well in the real world: how it grows, how fast it feels, how you see inside it, how data stays in agreement, how it's organized, how it's released and checked, what it costs, and how you decide between competing options.

---

## 9. SCALABILITY

### What it is

**Scalability is a system's ability to keep working well as demand grows.** Maybe your app has 100 users today and 1,000,000 next year. A scalable system can handle that growth without being rebuilt from scratch.

There are two basic ways to grow:

- **Vertical scaling ("scale up"):** buy a bigger, stronger machine.
- **Horizontal scaling ("scale out"):** add more machines and share the work among them.

### Diagram

```
   VERTICAL (scale up)              HORIZONTAL (scale out)

   ┌──────────┐                                  ┌──────────┐
   │          │                              ┌─▶ │ Server 1 │ ─┐
   │  BIGGER  │          Users ─▶ [LOAD  ] ──┼─▶ │ Server 2 │ ─┼─▶ shared
   │  SERVER  │                   [BALANCER]  └─▶ │ Server 3 │ ─┘   database
   │          │                                  └──────────┘     + cache
   └──────────┘
   Simple, but has a ceiling       More complex, but can keep growing
   and one machine = one failure   and survives losing a machine
```

### Example

Our photo app goes viral. Traffic jumps from 100 to 100,000 people at once.

- A **load balancer** (a traffic director) spreads incoming requests across many identical copies of the app server.
- Heavy work, like captioning, is placed on a **queue** so a pool of workers can chew through it at their own pace.
- Popular photos are served from a **cache** so the database isn't asked the same question a million times.
- Eventually, the **database** becomes the thing that struggles most, because many servers all depend on it.

### The fundamental question

> **What happens when usage grows 10× or 1,000×, and what breaks first?**

### Gaps filled in

- **Bottlenecks.** A system is only as fast as its slowest part. Scaling means finding the current bottleneck, fixing it, and discovering the next one. The database is very often the hardest to scale.
- **Stateless parts scale easily.** If a server remembers nothing between requests, you can add or remove copies freely. That's why State design matters so much here.
- **Common scaling tools:** load balancing (spread traffic), caching (avoid repeated work), queues (smooth out spikes), replication (copy data for more reads), and sharding/partitioning (split data across machines by some rule, like "users A–M here, N–Z there").
- **Autoscaling.** Machines are added automatically when load rises and removed when it falls, which saves money.
- **Don't scale too early.** Building for a million users when you have a hundred adds cost and complexity for no benefit. Scale in response to real measurements.
- **Scaling has a price: complexity.** More machines means more communication, more failure points, and more consistency questions.

### Connects to

State (stateless is easy to scale), Storage (databases are the hard part), Concurrency (more things at once), Failure (more machines, more breakage), Performance (scaling should maintain speed, not just capacity), Cost (growth costs money), Trade-offs (simplicity now vs. headroom later).

---

## 10. PERFORMANCE (Latency & Throughput)

### What it is

**Performance is how fast and how efficiently a system does its work.** It has two main faces that people often mix up:

- **Latency:** how long *one* operation takes ("how long until my photo appears?").
- **Throughput:** how *many* operations can be completed per unit of time ("how many photos per second can we handle?").

A helpful picture is a water pipe: its **length** is like latency (how long a drop takes to travel), and its **width** is like throughput (how much water flows per second). A long, wide pipe can be slow for a single drop but still carry a lot.

### Diagram

```
   Where does the time go in one request?

   User taps ─▶ [network to server] ─▶ [server computes] ─▶ [database read]
                      40 ms                  30 ms               80 ms
                                                                   │
   User sees ◀─ [network back] ◀──────── [render] ◀────────────────┘
                      40 ms                10 ms

   Total latency = the sum of all the steps.
   To speed it up, find the BIGGEST step first (here: the database read).

   Throughput = how many of these journeys the system finishes per second.
```

### Example

On our photo app's home page:

- The page takes 3 seconds to load, and users leave.
- Measurement shows most time goes to fetching the same popular photo data from the database again and again.
- Adding a **cache** drops this to 0.5 seconds. Resizing images to smaller versions before sending them drops it further.

Nothing about the features changed, only where time was being spent.

### The fundamental question

> **How fast must this feel to the user, and how much work must the system handle per second?**

### Gaps filled in

- **Measure before you optimize.** Guessing where the slowness is leads to wasted effort. Find the real bottleneck first.
- **Averages hide pain.** If most requests take 0.1 seconds but 1 in 100 takes 10 seconds, the average looks fine while some users are miserable. That's why engineers look at **percentiles** (e.g., "99% of requests finish within X seconds") and the slow "tail latency."
- **Perceived performance matters.** A loading skeleton, a progress bar, or showing a result instantly while saving in the background can make a system *feel* faster without being faster.
- **Performance budgets.** Decide in advance how fast something must be (for example, "the page must be usable in under 2 seconds") so you know when you're done.
- **Common speed-ups:** caching, doing less work, doing work in the background, reading less data, putting data closer to the user, and choosing better algorithms.
- **Speed vs. correctness vs. cost.** Faster often costs more money or gives up some accuracy or freshness (see Trade-offs).

### Connects to

Communication (network hops add latency), Storage (where data lives affects read speed), Computation (algorithm choice), Concurrency (waiting on locks), Scalability (capacity growth vs. speed), Observability (you need measurements), Cost (faster usually costs more).

---

## 11. OBSERVABILITY

### What it is

**Observability is your ability to see what's happening inside a running system and to figure out *why*.** Monitoring tells you something is wrong. Observability helps you ask new questions to find out what and why, even for problems you didn't predict.

You can't fix what you can't see, and in a distributed system, you can't see much unless you deliberately build the windows in.

### The three kinds of signals

| Signal | Plain meaning | Analogy |
|---|---|---|
| **Logs** | A diary of events: "10:42:03, user 77 uploaded photo 123" | A ship's logbook |
| **Metrics** | Numbers measured over time: requests per second, error rate, response time | A car's dashboard gauges |
| **Traces** | The path of *one* request as it travels through every component, with timing for each step | Tracking a parcel across every depot |

### Diagram

```
   ┌──────────────────────────────────────────┐
   │              THE RUNNING SYSTEM          │
   │  Frontend ─▶ API ─▶ Backend ─▶ Database  │
   └───────┬───────────────┬─────────────┬────┘
           │ logs          │ metrics     │ traces
           ▼               ▼             ▼
   ┌──────────────────────────────────────────┐
   │      COLLECTION AND DASHBOARDS           │
   │  "What is happening right now?"          │
   └───────────────────┬──────────────────────┘
                       ▼
              ┌────────────────┐
              │    ALERTS      │  "Something looks wrong"
              └───────┬────────┘
                      ▼
              ┌────────────────┐
              │  A HUMAN (or   │  investigate ─▶ fix ─▶ learn
              │  automation)   │
              └────────────────┘
```

### Example

Users complain that "uploads are failing sometimes." With observability:

- **Metrics** show the upload error rate jumped from 0.1% to 8% at 3:00 PM.
- **Logs** show a "storage timeout" message appearing at the same time.
- **Traces** show that the slow step is the call to object storage.
- An **alert** had already notified the team at 3:02 PM, before most users noticed.

Without these signals, the team would be guessing.

### The fundamental question

> **How will we know what the system is doing, and why it's doing it?**

### Gaps filled in

- **Observability is how "How will we detect it?" (from Failure) gets answered.** Failure tells you *to* detect problems; observability gives you the *means*.
- **Good alerts are rare and meaningful.** If an alert fires constantly, people ignore it. Alert on things that need a human, and make each alert actionable.
- **Correlation IDs.** Stamp every request with a unique ID, and pass it along to every component. Then all the logs for one user's problem can be found together.
- **Log carefully.** Logs are useful but can accidentally capture passwords or personal information. Treat logs as sensitive data (see Security).
- **Business metrics count too.** Not just "is the server healthy?" but "are people actually completing signup?" Technical health and user outcomes can diverge.
- **Observability costs money and storage.** More data means a bigger bill, so you choose what's worth keeping and for how long.
- **Learn from incidents.** After something breaks, a blameless review ("what happened, why, and how do we prevent it?") turns pain into improvement.

### Connects to

Failure (detection), Performance (measurement), Security (audit trails), Deployment (watching a new release), Scalability (seeing bottlenecks), Cost (data volume), Testing (monitoring is a form of testing in the real world).

---

## 12. CONSISTENCY

### What it is

**Consistency is about whether different copies of the same data agree with each other, and how quickly.** The moment data exists in more than one place (a database and a cache, a primary and a replica, two regions), copies can drift apart.

Two broad styles:

- **Strong consistency:** after a change, *everyone* sees the new value immediately. Safe and simple to reason about, but slower and less available when things go wrong.
- **Eventual consistency:** after a change, copies catch up *shortly*, but for a moment some people may see the old value. Faster and more scalable, but briefly stale.

### Diagram

```
   You "like" a photo (a write):

                    ┌─────────────────┐
   You ──write────▶ │  PRIMARY COPY   │  count = 101   (updated now)
                    └────────┬────────┘
                             │ copying takes time...
                  ┌──────────┴──────────┐
                  ▼                     ▼
          ┌──────────────┐      ┌──────────────┐
          │  REPLICA 1   │      │  REPLICA 2   │
          │ count = 100  │      │ count = 100  │   (stale for a moment)
          └──────────────┘      └──────────────┘
                  ▲
   Friend ──read──┘   sees 100 for a short time, then 101

   STRONG: the friend would be made to wait until the copies agree.
   EVENTUAL: the friend sees the old value now and the new one soon.
```

### Example

- **Like counts** in our photo app: it's perfectly fine if one person sees 100 likes and another sees 101 for a few seconds. Eventual consistency is a good, cheap choice.
- **A payment or account balance:** showing an outdated balance, or allowing money to be spent twice, is unacceptable. Use strong consistency.
- **Your own actions:** if you post a photo and it doesn't appear on *your* screen, you'll think it failed. Systems usually guarantee you see your own changes right away ("read your own writes"), even when others see them a little later.

### The fundamental question

> **When data exists in more than one place, how quickly and how strictly must the copies agree?**

### Gaps filled in

- **A famous tension.** When a network splits so parts of the system can't talk to each other, you must choose between *staying available* (keep answering, maybe with stale data) and *staying consistent* (refuse to answer until things agree). You can't fully have both during the split. This idea is often called the **CAP theorem**.
- **Choose per kind of data, not per system.** Most real apps mix both: strict where mistakes are costly, relaxed where a short delay is harmless.
- **Where it appears:** caches, database replicas, multiple regions, offline apps that sync later, and messages arriving out of order.
- **Conflicts.** If two people change the same thing while disconnected, the system needs a rule for resolving it (last change wins, merge both, or ask a human).
- **Transactions give strong guarantees in one place.** Spread across many services, they get much harder. Systems often use compensating steps ("undo the earlier steps if a later one fails") instead.
- **Consistency vs. speed.** Waiting for all copies to agree costs time. This is a direct link to Performance.

### Connects to

State (which copy is the truth), Storage (replicas and caches), Concurrency (overlapping changes), Communication (delayed or reordered messages), Failure (network splits), Performance and Scalability (agreeing costs time and limits growth), Trade-offs (the classic example of one).

---

## 13. ABSTRACTION & LAYERING

### What it is

**Abstraction means hiding complicated details behind a simpler surface so you can use something without knowing how it works inside.** **Layering** means stacking these simplifications so each level only needs to understand the one directly below it.

You do this every day: you drive a car without understanding combustion, and you flip a light switch without understanding the power grid.

### Diagram

```
   Each layer uses the one below it and hides the details from the one above.

   ┌─────────────────────────────────────────────┐
   │  USER INTERFACE        "Tap 'Upload'"       │  ← users think here
   ├─────────────────────────────────────────────┤
   │  APPLICATION LOGIC     "Photos belong to    │  ← feature developers
   │                         users; limit 20 MB" │    think here
   ├─────────────────────────────────────────────┤
   │  SERVICES / LIBRARIES  "Save this file",    │  ← shared building
   │                         "Send this message" │    blocks
   ├─────────────────────────────────────────────┤
   │  PLATFORM / OS / CLOUD "Run this program",  │  ← infrastructure
   │                         "Store these bytes" │    engineers think here
   ├─────────────────────────────────────────────┤
   │  HARDWARE              CPUs, disks, network │
   └─────────────────────────────────────────────┘
```

### Example

In our photo app, a developer writes "save this photo." They don't need to know *how* object storage splits the file across disks, keeps backup copies, or recovers from a failed drive. The storage service hides all of that behind one simple instruction. That's abstraction, and it's the reason a small team can build something powerful.

### The fundamental question

> **What can this layer hide, so people and components above it don't need to know the details?**

### Gaps filled in

- **Abstraction is why the other fundamentals can be understood separately.** It's the reason you can study Storage without learning every detail of Security.
- **Separation of concerns.** Give each part one clear job. The part that stores photos shouldn't also decide who may see them.
- **Coupling and cohesion.** Aim for parts that are *loosely coupled* (they depend on each other as little as possible, so changing one doesn't break others) and *highly cohesive* (everything inside a part belongs together).
- **Leaky abstractions.** Abstractions sometimes fail to hide the reality below. A "simple" file-saving function still gets slow when the network is slow, and your code must cope. Great engineers know enough about the layer below to understand when the abstraction leaks.
- **Don't abstract too early.** Over-engineering layers you don't need makes software harder to understand. Abstract when you see real repeated patterns, not imagined ones.
- **Frameworks are abstractions.** That's the core of "frameworks change, fundamentals stay": a framework is a particular abstraction, and the fundamentals are what's underneath.

### Connects to

Communication and Interfaces (a layer's surface *is* its interface), Computation (choosing where work happens at each layer), Failure (layers can hide failures you still must handle), Testing (each layer can be tested in isolation), Deployment (layers can be changed independently).

---

## 14. INTERFACES & API DESIGN

### What it is

**An interface is the agreed boundary between two parts of a system. An API (Application Programming Interface) is an interface that programs use to talk to each other.** It's a **contract**: "if you send me *this*, I promise to give you back *that*."

An API doesn't expose how something works inside, only what you can ask and what you'll get. This is abstraction applied to the connection between two components.

### Diagram

```
   ┌──────────────┐         THE CONTRACT          ┌──────────────┐
   │   CALLER     │  ┌─────────────────────────┐  │   PROVIDER   │
   │ (frontend or │  │ You may ask for:        │  │ (backend or  │
   │  other       │─▶│  · "list photos"        │─▶│  service)    │
   │  service)    │  │  · "upload photo"       │  │              │
   │              │◀─│ You must send: a valid  │◀─│ Inside can   │
   └──────────────┘  │  login, a file under    │  │ change freely│
                     │  20 MB                  │  └──────────────┘
                     │ You will get back: the  │
                     │  photo's ID, OR a clear │
                     │  error explaining why   │
                     └─────────────────────────┘

   As long as the contract stays the same, either side can be rebuilt
   without the other noticing.
```

### Example

In our photo app, the "get this user's photos" API promises:

- **What you send:** which user, and how many photos you want per page.
- **What you get back (success):** a list of photos, each with an ID, a URL, a caption, and a like count.
- **What you get back (failure):** a clear error: "not logged in," "user not found," or "too many requests, try again shortly."

The mobile app, the website, and a partner's app can all use the same contract.

### The fundamental question

> **What is the contract between these two parts, and how can it change safely?**

### Gaps filled in

- **A contract must cover failures, not just success.** Errors, limits, and edge cases are part of the interface. A good API makes failure clear and actionable.
- **Backward compatibility.** Once others depend on your API, you can't casually change it. Adding is usually safe; removing or renaming breaks people.
- **Versioning.** When a breaking change is unavoidable, provide a new version and let old and new run side by side for a while.
- **Small, clear, consistent.** Predictable naming and behavior make an API easy to learn. Surprising behavior is the enemy.
- **Idempotency and retries.** Design operations so that repeating them (after a timeout) is safe (see Concurrency and Failure).
- **Pagination and limits.** Don't let a single request ask for a million items. Return data in manageable pages, and apply **rate limits** (a cap on how often someone may call) to protect the system.
- **Documentation is part of the interface.** An API nobody can understand might as well not exist.
- **Different shapes of API exist.** Common styles include request/response APIs over HTTP, strictly typed service-to-service calls, and event-based interfaces. You'll recognize them as the mechanisms from Communication.
- **Interfaces appear at every scale:** between a button and its handler, between frontend and backend, between teams, and between your company and the world.

### Connects to

Communication (an API is the *meaning* carried by the mechanism), Abstraction (an interface is a layer's face), Security (every API is a doorway), Data (the shape of data in the contract), Deployment and Testing (contracts let each side be changed and verified independently), Failure (error semantics), Scalability (rate limits, pagination).

---

## 15. DEPLOYMENT & ENVIRONMENTS

### What it is

**Deployment is the process of getting working software from a developer's machine into the hands of real users, safely and repeatably.** An **environment** is a separate copy of the whole setup where software runs for a particular purpose.

Typical environments:

- **Development:** where developers build and experiment. Mistakes are cheap.
- **Testing / Staging:** a realistic rehearsal space that resembles production.
- **Production:** the real system, with real users and real data. Mistakes are expensive.

### Diagram

```
   Developer's    ┌─────────┐   ┌─────────┐   ┌──────────┐   ┌────────────┐
   machine   ───▶ │  BUILD  │──▶│  TEST   │──▶│ STAGING  │──▶│ PRODUCTION │
   (new code)     │ package │   │automatic│   │ rehearse │   │ real users │
                  └─────────┘   │ checks  │   └──────────┘   └─────┬──────┘
                                └────┬────┘                        │
                                     │ fails? STOP                 │ watch (observability)
                                     ▼                             │
                               fix and try again          problem? ▼
                                                           ROLL BACK to the
                                                           previous version

   This automated path is often called CI/CD
   (continuous integration / continuous delivery).
```

### Example

We add a new "AI caption" feature to our photo app:

1. It's built and tested automatically.
2. It's rehearsed in staging.
3. It's released to just **5% of users** first (a *canary release*), while the team watches error rates.
4. If metrics look healthy, it rolls out to everyone. If errors spike, it's switched off or rolled back in minutes.

### The fundamental question

> **How does a change get from a developer's machine to real users safely and reversibly?**

### Gaps filled in

- **Configuration is separate from code.** Settings that differ per environment (database address, keys, feature limits) should live outside the program, so the same software can run anywhere.
- **Secrets need special handling.** Passwords and keys must never be stored openly alongside code (see Security).
- **Rollback must be easy.** If you can't quickly undo a bad release, every release is frightening, so teams release rarely and in big, risky chunks. Easy rollback encourages small, frequent, safer releases.
- **Gradual rollouts.** Canary releases (to a small group first), and **feature flags** (switches that turn a feature on or off without redeploying) limit the blast radius of mistakes.
- **Infrastructure as code.** Describe servers and networks in files so environments can be recreated reliably, rather than set up by hand and forgotten.
- **Containers (conceptually).** A way to package software together with everything it needs, so it behaves the same on a laptop and in the cloud. This helps keep environments consistent.
- **Environments should resemble each other.** "It worked on my machine" happens when development and production differ. The more alike they are, the fewer surprises.
- **Deployments touch state.** Replacing servers loses anything stored only in memory, and changing a database's shape (a *migration*) must be done carefully so old and new software versions both keep working during the transition.
- **Never use real user data carelessly in lower environments.** It creates privacy risk.

### Connects to

Failure (a bad release is a failure; rollback is recovery), Testing (the gate every release passes through), Observability (watching a release), Security (secrets and access to production), State (migrations, restarts), Scalability (deploying to many machines), Interfaces (compatibility between old and new versions during a rollout).

---

## 16. TESTING

### What it is

**Testing is how you gain confidence that software does what it should, including when things go wrong.** It's the practice of checking behavior deliberately and repeatedly, ideally automatically, before users find the problems for you.

### Diagram: the testing pyramid

```
                      /\
                     /  \          END-TO-END TESTS
                    /    \         "Does the whole app work for a real
                   /──────\         user journey?"  Few. Slow. Realistic.
                  /        \
                 /          \      INTEGRATION TESTS
                /            \     "Do the parts work together?"
               /──────────────\     (app + database, service + service)
              /                \
             /                  \  UNIT TESTS
            /                    \ "Does each small piece work alone?"
           /──────────────────────\ Many. Fast. Cheap.

   More tests at the bottom (fast and cheap), fewer at the top
   (slow and expensive but closest to reality).
```

### Example

For our photo app, tests would check things like:

- **Small piece:** "Given a 25 MB file, does the size check reject it?"
- **Parts together:** "When a photo is uploaded, does a record appear in the database and a caption job in the queue?"
- **Whole journey:** "Can a new user sign up, upload a photo, and see it on their profile?"
- **Failure and concurrency:** "If the caption service is down, does the upload still succeed?" and "If 500 people like a photo at once, is the final count exactly right? If the same like arrives twice, is it only counted once?"

### The fundamental question

> **How do we know it works, including when things go wrong?**

### Gaps filled in

- **Test the unhappy paths.** The happy path (everything goes right) is easy. Real confidence comes from testing bad input, timeouts, duplicates, and outages.
- **Regression tests.** When you fix a bug, write a test that would have caught it, so it can never silently return.
- **Tests are a safety net for change.** The point is not perfection but freedom: with good tests, you can improve code without fear.
- **Automated beats manual.** Tests that run on every change (as part of deployment) catch problems early, when they're cheapest to fix.
- **Test data.** Realistic, safe test data (not real user data) is often the hardest part.
- **Testing concurrency and failure on purpose.** These bugs appear rarely, so teams deliberately simulate slow networks, crashed servers, and heavy load (load testing, and sometimes "chaos testing," where failures are injected on purpose).
- **Testing doesn't prove the absence of bugs.** It reduces risk; it doesn't remove it. That's why Observability continues the job in production.
- **Test the contract.** Checks that a service still keeps its promises to the things depending on it prevent accidental breakage (see Interfaces).
- **Security testing.** Deliberately trying to misuse the system finds weaknesses before attackers do.

### Connects to

Failure and Concurrency (the hardest things to test, and the most important), Deployment (tests gate releases), Interfaces (contract tests), Observability (testing continues in production through monitoring), Abstraction (layers allow isolated tests), Security (misuse testing), Cost (more testing costs time, but cheaper than outages).

---

## 17. COST

### What it is

**Cost is what it takes, in money, time, and effort, to build, run, and maintain the system.** Every design choice has a price, and in the cloud era, you often pay for exactly what you use, down to each request, each gigabyte, and each AI call.

### Diagram: where the money goes

```
   TOTAL COST OF A SYSTEM
   ┌──────────────────────────────────────────────────────────┐
   │ COMPUTE         servers, functions, GPUs for AI          │
   ├──────────────────────────────────────────────────────────┤
   │ STORAGE         databases, caches, files, backups        │
   ├──────────────────────────────────────────────────────────┤
   │ NETWORK         data moved between systems and to users  │
   ├──────────────────────────────────────────────────────────┤
   │ THIRD-PARTY     payments, email, AI model APIs, tools    │
   ├──────────────────────────────────────────────────────────┤
   │ OBSERVABILITY   logs, metrics, traces (more data = more $)│
   ├──────────────────────────────────────────────────────────┤
   │ PEOPLE & TIME   building, maintaining, being on call     │
   └──────────────────────────────────────────────────────────┘
   Often the largest hidden cost: the human time to keep it running.
```

### Example

In our photo app, each AI caption costs a small amount. At 100 uploads a day, nobody notices. At 10 million uploads a day, it's a major expense. So the team:

- **Resizes images first** so the AI model processes fewer pixels (less computation).
- **Caches** captions so the same photo is never captioned twice.
- Decides to caption **only photos that get shared publicly**, not every private upload.
- Moves old, rarely viewed photos to **cheaper storage** tiers.

### The fundamental question

> **What does this design cost to build and run, and is it worth it?**

### Gaps filled in

- **Cost is a design constraint, just like speed or security.** The "best" technical solution may be unaffordable or not worth its price.
- **Fixed vs. variable costs.** Some costs stay the same (a reserved server). Others grow with usage (per-request pricing). Know which is which.
- **Unit economics.** Work out the cost *per user* or *per action*. If each user costs more than they bring in, growth makes things worse.
- **Hidden costs.** Moving data out of a cloud provider, storing logs forever, idle machines nobody turned off, and the human cost of maintaining a complicated system.
- **Cost of complexity.** Each extra component is something that must be learned, monitored, secured, and fixed. Simple systems are cheaper in the long run.
- **Build vs. buy.** Use an existing service or build your own? Buying is faster to start and has an ongoing fee; building gives control but takes time and ongoing maintenance.
- **Set budgets and alerts.** Cloud bills can balloon silently, so monitor spending like any other metric (see Observability).
- **Cost shapes the other fundamentals.** Faster (Performance), more reliable (Failure), more scalable, and more heavily logged systems all cost more.

### Connects to

Computation (compute bills), Storage (storage tiers), Communication (data transfer), Scalability (growth multiplies cost), Performance (speed costs), Observability (data volume), Failure (redundancy costs money), Trade-offs (cost is usually one side of the scale).

---

## 18. TRADE-OFFS

### What it is

**A trade-off is a choice where gaining one good thing means giving up another.** In software there is rarely a "best" option, only options that fit your particular constraints. Almost every fundamental you've learned so far is secretly a trade-off.

### Diagram: the tensions

```
   You want...                  but it costs you...

   Speed (Performance)  ◀──────────▶  Money (Cost) / Freshness (Consistency)
   Strict agreement     ◀──────────▶  Speed and availability
     (Consistency)
   Simplicity           ◀──────────▶  Flexibility and future growth
   Safety (Security)    ◀──────────▶  Convenience for users and developers
   Reliability          ◀──────────▶  Money and complexity (redundancy)
   Move fast (shipping) ◀──────────▶  Thoroughness (testing, polish)
   Build it yourself    ◀──────────▶  Buy it (speed to start vs. control)
   Detail (Observability) ◀────────▶  Storage cost and noise

   Good engineering = choosing deliberately, not pretending the
   other side of the scale doesn't exist.
```

### Example

Our photo app must decide how to caption photos:

| Option | Gains | Gives up |
|---|---|---|
| **Caption while the user waits** | Simple; caption is there instantly | Slow uploads; fails if the AI is down; costly spikes |
| **Caption in the background (queue)** | Fast uploads; survives AI outages; smooths cost | More moving parts; caption appears a bit later; extra things to monitor |

For a small prototype, the first is a fine choice. As the app grows, the second is usually worth its complexity. Neither is "right" in general, only right for a situation.

### The fundamental question

> **What are we giving up to get what we want, and is that acceptable?**

### Gaps filled in

- **Start from requirements and constraints.** What *must* be true (it can never lose a payment)? What would be *nice* (captions in under a second)? What are the limits (budget, deadline, team size)? Those determine which trade is right.
- **Reversible vs. irreversible decisions.** Easy-to-undo choices can be made quickly. Hard-to-undo ones (a data model, a public API) deserve more thought.
- **Write down the "why."** Brief decision records ("we chose X over Y because Z") spare future teammates, and future you, from wondering why things are the way they are.
- **Good enough is a skill.** Perfect on every axis is impossible and expensive. Decide what must be excellent and what can be merely adequate.
- **Today's right answer may not be tomorrow's.** As usage, budget, and understanding change, revisit decisions.
- **Beware of hype.** Any tool that claims no downsides is hiding them. Ask: *what is this trading away?*
- **Trade-offs are how all the fundamentals connect.** The eight core questions each have more than one defensible answer, and the sections of Part 2 tell you how to judge between them.

### Connects to

Everything. This is the thinking skill that turns knowledge of the fundamentals into good decisions.

---

# PART 3: PUTTING IT TOGETHER

---

## 19. THE CONNECTION: how they all work together

These fundamentals are **not isolated topics**. They are connected pieces of one system.

### A full journey through the system

Let's follow one action, **"user uploads a photo"**, through every fundamental:

```
 1. USER taps "Upload".
        │
        ▼
 2. FRONTEND  checks the file looks reasonable (quick UX validation)
        │               and keeps temporary UI STATE ("uploading... 40%")
        ▼
 3. COMMUNICATION  sends it over the network (HTTP) through an API
        │              (an INTERFACE with a clear contract, including errors)
        ▼
 4. BACKEND  ── SECURITY: Is this user logged in? (authn)
        │                 Are they allowed to upload? (authz)
        │                 Is the file safe and valid? (backend validation)
        │
        ├──▶ STATE / STORAGE
        │       · Image file  → object storage
        │       · Photo record (owner, time, title) → database
        │       · "Recently uploaded" list → cache for fast loading
        │       · CONSISTENCY: the cache and replicas catch up shortly;
        │         the uploader sees their own photo immediately
        │
        └──▶ COMPUTATION
                · Resize the image
                · Queue an AI caption job (async COMMUNICATION)
                · AI model predicts a caption
        │
        ▼
 5. RESULT  returned to the user: "Photo posted!"

 Throughout all of it:
   · CONCURRENCY:   thousands of other uploads and likes are happening too
   · FAILURE:       what if storage is slow? the AI is down? the network drops?
   · OBSERVABILITY: logs, metrics, and traces record what happened and how long
   · PERFORMANCE:   each step adds to how long the user waits
   · SCALABILITY:   the same journey must work for 100 users or 100,000
   · COST:          every step (compute, storage, AI call) has a price

 Around the journey:
   · ABSTRACTION & LAYERING:  each step only needs to know the layer next to it
   · TESTING + DEPLOYMENT:    this whole path was tested, and was released
                              safely, and can be rolled back
   · TRADE-OFFS:              every design choice above gave something up
```

### The wrapping concerns

Look again at the big-picture diagram: **Security, Failure, and Concurrency** wrap around everything. They're called **cross-cutting concerns** because they aren't a single box you place on a diagram. They *cut across* every layer. Part 2 adds more of the same kind: **Observability, Performance, Cost, and Consistency** also affect every layer, and **Trade-offs** govern the choices you make about all of them.

### How the fundamentals influence each other

- **Storage ↔ State:** state decides *what* must be remembered; storage decides *where*.
- **Communication ↔ Failure:** every message across a network can be lost, delayed, or duplicated.
- **Concurrency ↔ State:** trouble starts when many actors change the same state.
- **Computation ↔ Communication:** you can move the work to the data, or move the data to the work, and each has costs.
- **Security ↔ everything:** every arrow in the diagram is a place where trust must be checked.
- **Scalability ↔ State, Storage, Concurrency:** growth is easy for stateless parts and hard where data must be shared.
- **Consistency ↔ Concurrency, Storage, Failure:** agreement between copies is threatened by overlapping changes, and by breakage.
- **Observability ↔ Failure, Performance, Deployment:** you can only fix, speed up, or safely release what you can see.
- **Interfaces ↔ Communication, Abstraction, Security:** the contract is where meaning, hiding, and trust meet.
- **Testing ↔ Deployment ↔ Observability:** a loop. Test before release, deploy gradually, watch in production, learn, and test again.
- **Cost ↔ Performance, Scalability, Failure:** nearly every improvement has a price.
- **Trade-offs ↔ all of the above:** the decision-making layer over everything.

---

## 20. THE BIGGER IDEA

> **Frameworks change. Fundamentals stay.**

React changes. Python evolves. AI frameworks come and go. Cloud services change names and features every year.

But every serious system, no matter what tools it uses, must still answer the same questions.

### The core eight

| # | Fundamental | The question |
|---|---|---|
| 1 | **Data** | What data do we have, and how should it be represented? |
| 2 | **State** | Where does state live, and who owns it? |
| 3 | **Communication** | How do components communicate? |
| 4 | **Storage** | Where is data stored, and for how long? |
| 5 | **Computation** | Where does computation happen, and how expensive is it? |
| 6 | **Concurrency** | What happens when operations overlap? |
| 7 | **Failure** | What happens when something fails? |
| 8 | **Security** | Who is allowed to access what? |

### What makes a system good

| # | Fundamental | The question |
|---|---|---|
| 9 | **Scalability** | What happens when usage grows 10× or 1,000×, and what breaks first? |
| 10 | **Performance** | How fast must this feel, and how much work must it handle per second? |
| 11 | **Observability** | How will we know what the system is doing, and why? |
| 12 | **Consistency** | When data lives in several places, how quickly and strictly must the copies agree? |
| 13 | **Abstraction & Layering** | What can this layer hide from the layers around it? |
| 14 | **Interfaces & API Design** | What is the contract between these parts, and how can it change safely? |
| 15 | **Deployment & Environments** | How does a change reach real users safely and reversibly? |
| 16 | **Testing** | How do we know it works, even when things go wrong? |
| 17 | **Cost** | What does this cost to build and run, and is it worth it? |
| 18 | **Trade-offs** | What are we giving up to get what we want? |

### How to use this when you meet a new tool

Whenever you encounter a new framework, database, or cloud service, don't ask *"how do I use it?"* first. Ask:

1. **Which fundamental does this tool help with?** (A cache helps Storage and Performance. A queue helps Communication, Concurrency, and Scalability. A login library helps Security. A dashboard tool helps Observability.)
2. **What trade-off is it making?** (Faster but less durable? Simpler but less flexible? Cheaper but less consistent?)
3. **What does it assume about failure and concurrency?**
4. **Where does it want state to live, and who owns it?**
5. **What is its interface, and how stable is the contract?**
6. **How do I test it, deploy it, and observe it?**
7. **What will it cost when I use it a lot?**

Once you do this, new tools stop feeling like brand-new mountains to climb. They become *new answers to questions you already know.*

---

## 21. QUICK GLOSSARY

| Term | Simple meaning |
|---|---|
| **Abstraction** | Hiding complexity behind a simpler surface |
| **API** | A defined menu of requests one program can make of another |
| **Async** | "Send it and move on; I'll handle the reply later" |
| **Atomic** | All-or-nothing; cannot be interrupted halfway |
| **Authn / Authz** | Proving who you are / what you're allowed to do |
| **Availability** | The system keeps answering requests, even during problems |
| **Backend** | The behind-the-scenes part that holds rules, logic, and data access |
| **Backward compatibility** | New versions keep working with things built for old versions |
| **Bottleneck** | The slowest part, which limits the whole system |
| **Cache** | A fast, temporary copy of data for quick reuse |
| **Canary release** | Releasing a change to a small group first to catch problems early |
| **CAP theorem** | During a network split, you can't fully have both consistency and availability |
| **CI/CD** | Automated building, testing, and delivering of changes |
| **Circuit breaker** | A safety switch that stops calling a failing service for a while |
| **Concurrency** | Many things in progress at overlapping times |
| **Consistency** | Whether copies of data agree with each other |
| **Coupling** | How much one part depends on another (lower is usually better) |
| **Deadlock** | Two or more things waiting on each other forever |
| **Encryption** | Scrambling data so only the intended party can read it |
| **Environment** | A separate copy of the system for a purpose (development, staging, production) |
| **Eventual consistency** | Copies disagree briefly, then catch up |
| **Feature flag** | A switch that turns a feature on or off without redeploying |
| **Frontend** | The part the user sees and interacts with |
| **Horizontal scaling** | Adding more machines |
| **Idempotent** | Doing it twice has the same effect as doing it once |
| **Latency** | How long one operation takes |
| **Load balancer** | Spreads incoming traffic across several servers |
| **Lock** | A mechanism letting only one operation touch something at a time |
| **Logs / Metrics / Traces** | A diary of events / numbers over time / the path of one request |
| **Migration** | A careful change to the shape of stored data |
| **Object storage** | Cheap, huge storage for files and media |
| **Observability** | Being able to see inside a running system and understand why |
| **Percentile** | "99% of requests were faster than this" (reveals slow outliers) |
| **Persistent** | Survives restarts and crashes |
| **Queue** | A waiting line of messages or jobs |
| **Race condition** | Result depends on which of two overlapping operations wins |
| **Rate limit** | A cap on how often someone may call something |
| **Replica** | A live copy of data kept on another machine |
| **Rollback** | Returning to the previous working version |
| **Scalability** | Staying effective as demand grows |
| **Session** | The server's memory of "this person is logged in" |
| **Sharding** | Splitting data across machines by some rule |
| **State** | What the system currently needs to remember |
| **Stateless** | Remembers nothing between requests |
| **Strong consistency** | Everyone sees the latest value immediately |
| **Throughput** | How much work can be completed per unit of time |
| **Trade-off** | Gaining something by giving up something else |
| **Transaction** | A group of steps that either all succeed or all fail together |
| **Versioning** | Labeling versions of an interface so changes don't break users |
| **Vertical scaling** | Using a bigger machine |

---

## 22. A SIMPLE STUDY PATH

If you want to learn these in a sensible order:

**Part 1: The core eight**

1. **Data**: practice modeling things and their relationships on paper (a library, a school, a shop).
2. **State**: for any app you use, list what it must remember and where that memory probably lives.
3. **Communication**: understand a request and a response; then understand why queues exist.
4. **Storage**: learn how a database differs from a cache and from memory; trace where each piece of an app's information lives.
5. **Computation**: study basic algorithms and data structures and learn to reason about cost.
6. **Concurrency**: learn the four classic problems with simple everyday stories (ticket booking, shared counters).
7. **Failure**: for any app you use, list ten ways it could break and how it should respond.
8. **Security**: learn the Authn/Authz distinction first, then validation, then encryption.

**Part 2: What makes a system good**

9. **Scalability**: learn vertical vs. horizontal scaling, then how load balancers, caches, and queues help.
10. **Performance**: practice finding where time goes in one request; learn latency vs. throughput and percentiles.
11. **Observability**: learn what logs, metrics, and traces each tell you; imagine debugging a failure using only them.
12. **Consistency**: learn strong vs. eventual consistency with the "like count vs. bank balance" contrast.
13. **Abstraction & Layering**: draw the layers of any app from user interface down to hardware.
14. **Interfaces & API design**: write (in plain words) the contract for three simple operations, including their errors.
15. **Deployment & environments**: learn dev → staging → production, rollbacks, and gradual rollouts.
16. **Testing**: learn the testing pyramid, then think of three failure cases to test.
17. **Cost**: estimate the cost per user of a simple app you imagine.
18. **Trade-offs**: for any design choice, write what it gains and what it gives up.

**Part 3: Putting it together**

19. **The connection**: take any app you use and *draw its diagram*, labeling where each fundamental shows up.

> **Best exercise:** Pick any app you use daily. Ask all eighteen fundamental questions about it. Draw the answers. Do this a few times, and you will start seeing the same structure everywhere.

---

*The tools change. The fundamentals stay.*
