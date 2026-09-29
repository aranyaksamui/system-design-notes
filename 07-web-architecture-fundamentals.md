# 07 - Web Architecture Fundamentals

This document combines the operating-system and networking knowledge from docs 01–06 into one picture: what a web application is made of and what happens, step by step, between a click and a rendered page.

Depends on: docs 02 (OS I/O and concurrency), 04 (DNS), 05 (TCP), 06 (HTTP, TLS).

---

# 7.1 Client-Server Architecture

```text
Client
   ↓
Network
   ↓
Server
   ↓
Database
```

- The **client** (browser, mobile app, another service) initiates requests.
- The **server** listens, does the work, and replies.
- The **database** is the durable store the server reads and writes.

## Tiers

| Tier | Responsibility | Examples |
| --- | --- | --- |
| **Presentation** | UI, input, rendering | Browser + HTML/JS, React, mobile app |
| **Application (logic)** | Business rules, validation, orchestration | Node.js/Express, ASP.NET Core, Django |
| **Data** | Durable storage | PostgreSQL, MySQL, Redis, object storage |

- **2-tier:** client talks directly to the database. Rare today because it exposes the database and puts logic in every client.
- **3-tier:** client → application server → database. This is the standard web architecture. The application server is the only component allowed to touch the database.

## Thin vs Thick Clients

- **Thin client:** the server renders most of the UI (classic server-rendered HTML).
- **Thick client / SPA / mobile app:** the client renders the UI and calls the server's **API** (JSON over HTTP). The server becomes a data and business-logic service.

## Why Servers Should Be Stateless

Because HTTP is stateless (doc 06), the easiest architecture keeps **no per-user state in server memory**. State lives in the database, a cache like Redis, or a token held by the client. Then any server instance can handle any request, so you can add or remove instances freely. This is the basis for horizontal scaling and load balancing (docs 21, 22).

## Communication Styles

| Style | Description |
| --- | --- |
| **Request/response** | Client asks, waits for the answer (HTTP APIs) |
| **Push / real-time** | Server sends data when it has something (WebSocket, SSE; doc 53) |
| **Asynchronous** | Client hands work off and gets the result later (queues; doc 25) |

---

# 7.2 Request Lifecycle

The complete journey when a user opens `https://shop.example.com/items/42`:

```text
Browser
 ↓
DNS
 ↓
TCP/TLS
 ↓
HTTP Request
 ↓
Web Server
 ↓
Application
 ↓
Database
 ↓
Application
 ↓
HTTP Response
 ↓
Client
```

## Step by Step

**1. The browser parses the URL** (doc 06): scheme `https`, host `shop.example.com`, port 443 (default), path `/items/42`. It checks its caches first: an HSTS list (force HTTPS) and the HTTP cache (a fresh `Cache-Control: max-age` copy needs no network at all).

**2. DNS resolution** (doc 04): browser cache → OS cache → recursive resolver → root → TLD → authoritative server. Result: an IP address such as `203.0.113.10`. Cached answers cost microseconds; a full resolution costs tens of milliseconds.

**3. TCP handshake** (doc 05): SYN, SYN-ACK, ACK to `203.0.113.10:443`. Cost: **1 RTT**.

**4. TLS handshake** (doc 06): certificate verification and key exchange. Cost: **1 RTT** with TLS 1.3, 2 with TLS 1.2. (HTTP/3 merges steps 3 and 4.)

**5. HTTP request** is sent, encrypted:

```text
GET /items/42 HTTP/1.1
Host: shop.example.com
Accept: text/html
Cookie: sid=abc123
```

**6. Edge and infrastructure.** The packet may pass through a **CDN**, a **load balancer** and a **reverse proxy** (Nginx). These terminate TLS, pick a healthy backend, and may answer from cache without touching the application (docs 22, 28, 52).

**7. The server host receives it.** Inside the kernel (doc 02):

```text
 NIC ─► interrupt ─► kernel TCP/IP stack ─► socket receive buffer
                                              │
                    server process wakes ◄────┘   (epoll reports the socket is readable)
                    │
                    ▼
       read() bytes ─► parse HTTP ─► route ─► handler
```

Completed handshakes wait in the **accept queue** until the server calls `accept()` (doc 05). An event-driven server (Nginx, Node.js) uses `epoll` to watch thousands of sockets from one thread (doc 02).

**8. The application runs** the handler: authenticates the request (session/JWT, doc 18), validates input, applies business logic.

**9. The database is queried** over another TCP connection, normally taken from a **connection pool** (opening a fresh connection per request would add a handshake and authentication each time). Frequently read data may come from a cache (Redis) instead (docs 23, 24).

**10. The application builds the response**: serializes JSON or renders HTML, sets headers (`Content-Type`, `Cache-Control`, `Set-Cookie`), and writes it to the socket.

**11. Response travels back** through the proxy/CDN (which may compress and cache it) to the browser.

**12. The browser renders:** parses HTML, then issues further requests for CSS, JavaScript, images and API calls (each may reuse the open connection), and paints the page.

## A Latency Budget

Total time is the sum of the sequential parts:

$$T_{\text{total}}=T_{\text{DNS}}+T_{\text{TCP}}+T_{\text{TLS}}+T_{\text{request}}+T_{\text{server}}+T_{\text{transfer}}$$

Assume RTT = 40 ms, an uncached DNS lookup of 30 ms, TLS 1.3, and a server time of 25 ms (5 ms of application code plus a 20 ms database query). The response is small enough to ignore transfer time.

| Step | Cold (new connection) | Warm (connection reused, DNS cached) |
| --- | --- | --- |
| DNS | 30 ms | 0 |
| TCP handshake | 40 ms | 0 |
| TLS handshake | 40 ms | 0 |
| Request + response round trip | 40 ms | 40 ms |
| Server processing | 25 ms | 25 ms |
| **Total** | **175 ms** | **65 ms** |

Conclusions used throughout system design:

- **Network round trips dominate** for small responses; reusing connections removes three of them.
- Reduce round trips by **caching** (browser, CDN, doc 52), by putting servers **closer to users**, and by avoiding **chains of sequential calls** (each internal service call adds a round trip inside the data center, about 0.5 ms each).
- Larger responses also pay TCP slow start (doc 05) and transfer time $L/R$ (doc 03).

## The Whole Lifecycle as Code

A minimal HTTP server makes steps 7–10 concrete: accept, read, parse, respond.

**Python (raw sockets, no framework):**

```python
import socket

with socket.socket() as srv:
    srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    srv.bind(("0.0.0.0", 8080))
    srv.listen(128)
    while True:
        conn, _ = srv.accept()
        with conn:
            request = conn.recv(4096)                         # demo: assumes the whole request arrives at once
            request_line = request.split(b"\r\n", 1)[0]       # e.g. b"GET /items/42 HTTP/1.1"
            body = b"hello\n"
            conn.sendall(
                b"HTTP/1.1 200 OK\r\n"
                b"Content-Type: text/plain\r\n"
                b"Content-Length: " + str(len(body)).encode() + b"\r\n"
                b"Connection: close\r\n\r\n" + body
            )
```

**C:**

```c
#include <arpa/inet.h>
#include <netinet/in.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>

int main(void) {
    int srv = socket(AF_INET, SOCK_STREAM, 0);
    int yes = 1;
    setsockopt(srv, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof yes);

    struct sockaddr_in a;
    memset(&a, 0, sizeof a);
    a.sin_family = AF_INET;
    a.sin_addr.s_addr = htonl(INADDR_ANY);
    a.sin_port = htons(8080);
    bind(srv, (struct sockaddr *)&a, sizeof a);
    listen(srv, 128);

    const char *resp =
        "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n"
        "Content-Length: 6\r\nConnection: close\r\n\r\nhello\n";

    for (;;) {
        int c = accept(srv, NULL, NULL);
        char req[4096];
        read(c, req, sizeof req);                 // demo only: real servers loop until the headers end
        write(c, resp, strlen(resp));
        close(c);
    }
}
```

**Node.js:**

```js
require("http").createServer((req, res) => {
  res.writeHead(200, { "Content-Type": "text/plain" });
  res.end("hello\n");
}).listen(8080);
```

**C# (ASP.NET Core minimal API, served by the Kestrel web server):**

```csharp
var app = WebApplication.CreateBuilder(args).Build();
app.MapGet("/items/{id}", (int id) => Results.Json(new { id, name = "book" }));
app.Run("http://0.0.0.0:8080");
```

The raw versions handle **one client at a time** and trust that a whole request arrives in one read. Real servers add concurrency (threads, or an event loop), robust HTTP parsing (headers may arrive in pieces; doc 05), timeouts, keep-alive and TLS. That is what web servers and frameworks provide.

---

# 7.3 Web Servers

## Definitions

| Component | Role | Examples |
| --- | --- | --- |
| **Web server** | Speaks HTTP; serves **static files** efficiently; can proxy requests onward | Nginx, Apache httpd, Caddy, IIS |
| **Application server** | Runs your **application code** (business logic) and produces dynamic responses | Node.js/Express, Gunicorn/Uvicorn (Python), Kestrel (.NET), Tomcat (Java) |
| **Reverse proxy** | Sits in front of servers; accepts client connections and forwards requests to backends | Nginx, HAProxy, Envoy, Traefik |

The categories overlap: Node.js and Kestrel are application servers that also speak HTTP directly, and Nginx is a web server, a reverse proxy and a load balancer at once.

## Static Files vs Dynamic Requests

| | Static | Dynamic |
| --- | --- | --- |
| Content | Files as-is: HTML, CSS, JS, images, video | Generated per request from code and data |
| Cost | Tiny: read a file and send it | CPU, database, network calls |
| Best served by | Nginx or a **CDN** (doc 52) | Application server |
| Caching | Very effective (long `max-age`, versioned file names) | Selective |

A static server can use the `sendfile()` system call so the kernel copies file data from the page cache to the socket **without passing it through user space** (doc 02), which is why Nginx serves files so cheaply.

## The Reverse Proxy Pattern

```text
                                           ┌─► App server 1 ─┐
 Browser ─► CDN ─► Nginx (reverse proxy) ──┼─► App server 2 ─┼─► Database / Redis
                   · TLS termination       └─► App server 3 ─┘
                   · static files
                   · load balancing
                   · compression, caching
```

Why put a proxy in front of the application:

- **TLS termination** in one place.
- **Serve static files** directly and keep the app free for dynamic work.
- **Load balance** across several app instances and remove failed ones (doc 22).
- **Buffer slow clients:** the proxy absorbs a slow mobile client's upload/download so an app worker is not held for the whole transfer.
- **Security and shielding:** the app is not exposed directly; request size limits, rate limits and header cleanup happen at the edge.

A minimal Nginx configuration for this (details in doc 28):

```nginx
server {
    listen 80;
    location /static/ { root /var/www; expires 30d; }          # files served directly
    location /api/    { proxy_pass http://127.0.0.1:3000; }    # dynamic requests go to the app
}
```

## Server Concurrency Models

How does one server handle thousands of simultaneous connections? (Doc 02, sections 2.2 and 2.7.)

| Model | How it works | Used by | Trade-off |
| --- | --- | --- | --- |
| **Process per request/connection** | A new process (or a pooled one) handles each | Old Apache `prefork`, classic CGI | Strong isolation, heavy memory |
| **Thread per request/connection** | One OS thread each (usually from a thread pool) | Tomcat, Apache `worker`, Gunicorn threads | Simple blocking code; thousands of threads cost memory and context switches |
| **Event-driven, non-blocking** | One (or a few) threads multiplex many connections with `epoll` | Nginx, Node.js, Redis | Very memory-efficient for I/O-bound work; a CPU-heavy handler blocks the loop |
| **Hybrid** | Several event-loop processes/threads, one per core | Nginx workers, Node `cluster`, Kestrel | Uses all cores |

**How many concurrent requests are in flight?** Little's Law (doc 01) gives $L=\lambda W$. At 2,000 requests/s with an average latency of 50 ms:

$$L=2000\times0.05=100\ \text{requests in flight}$$

A thread-per-request server needs at least 100 worker threads for this load; an event loop tracks 100 pending operations on one thread. The same law sizes database connection pools: if each request holds a database connection for 20 ms,

$$L_{\text{db}}=2000\times0.02=40\ \text{connections}$$

so a pool of about 40 is enough, and a pool of 500 only adds contention on the database.

## Connecting a Server to Application Code

Languages define an interface between the web server and the code:

| Interface | Language | Note |
| --- | --- | --- |
| **CGI** | Any | Starts a new process per request: slow, obsolete |
| **FastCGI** | Any | Long-lived worker processes; used with PHP-FPM |
| **WSGI** | Python (synchronous) | A function `app(environ, start_response)`, run by Gunicorn/uWSGI |
| **ASGI** | Python (asynchronous) | Async version for FastAPI/Starlette, run by Uvicorn |
| Built-in HTTP server | Node.js, Go, .NET (Kestrel) | The runtime itself speaks HTTP |

A complete WSGI application:

```python
def app(environ, start_response):
    body = b"hello\n"
    start_response("200 OK", [("Content-Type", "text/plain"), ("Content-Length", str(len(body)))])
    return [body]

# run with several worker processes:   gunicorn -w 4 app:app
```

**A static file handler in code** (real servers must also block path traversal such as `../../etc/passwd`, and set `Content-Type` and caching headers):

```js
const http = require("http"), fs = require("fs"), path = require("path");
const ROOT = path.resolve("public");

http.createServer((req, res) => {
  const file = path.resolve(path.join(ROOT, decodeURIComponent(req.url)));
  if (!file.startsWith(ROOT + path.sep)) { res.writeHead(403).end(); return; }   // traversal guard
  fs.createReadStream(file)                                                      // stream: constant memory
    .on("error", () => res.writeHead(404).end())
    .pipe(res);
}).listen(8080);
```

## Where the Rest of the Roadmap Goes

The single server above hits limits. Each later topic exists to lift one of them:

| Limit | Remedy | Doc |
| --- | --- | --- |
| One machine is not enough | Load balancing, horizontal scaling | 21, 22 |
| Database is slow | Caching, indexing, replicas | 12, 15, 23, 24 |
| Slow work blocks requests | Message queues, workers | 25 |
| Far-away users | CDN, multi-region | 52, 43 |
| One process's failure takes everything down | Redundancy, health checks, orchestration | 36, 44 |

---

# Key Takeaways

- The standard web architecture is **client → application server → database** (3-tier). Keep application servers **stateless** so they can be replicated freely.
- A request passes through **DNS → TCP → TLS → HTTP → proxy/LB → app → database → back**. Cold connections cost several round trips; **reusing connections and caching** removes most of the latency.
- Latency is the **sum** of sequential steps; for small responses round trips dominate, so cut round trips first.
- **Web servers** serve static files and proxy; **application servers** run business logic; a **reverse proxy** terminates TLS, balances load, buffers slow clients and shields the app.
- Servers scale with **event loops + `epoll`** (Nginx, Node.js) or **thread pools**; use Little's Law $L=\lambda W$ to size concurrency and connection pools.
- Static content belongs on a web server or CDN; dynamic requests go to the application.
