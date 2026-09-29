# 06 - HTTP & Web Fundamentals

**HTTP (HyperText Transfer Protocol)** is the application-layer protocol of the web and of almost every backend API. It defines how a client asks for something and how a server answers. This document covers HTTP messages, methods, headers, status codes, versions, and HTTPS (HTTP over TLS).

Depends on: docs 03–05 (sockets, IP, DNS, TCP).

---

# 6.1 HTTP

## Key Properties

- **Request/response:** the client sends a request; the server sends one response.
- **Stateless:** each request is independent; the server keeps no memory of earlier requests unless the application adds it (cookies, tokens, sessions; doc 18).
- **Text-based (HTTP/1.x):** messages are readable text. HTTP/2 and HTTP/3 are binary but keep the same semantics.
- **Runs on a reliable transport:** TCP for HTTP/1.1 and HTTP/2, QUIC for HTTP/3.

## Anatomy of a URL

```text
 https://user@www.example.com:8443/shop/items?id=42&sort=asc#reviews
 └─┬──┘ └─┬─┘ └──────┬──────┘└─┬─┘└────┬────┘└──────┬──────┘└──┬───┘
 scheme userinfo    host     port    path         query     fragment
```

- **scheme** picks the protocol (`http` → default port 80, `https` → 443).
- **host** is resolved with DNS (doc 04). **path** and **query** identify the resource.
- The **fragment** (`#reviews`) is used only by the browser and is **never sent to the server**.

## HTTP Request

```text
POST /api/orders HTTP/1.1\r\n            ← request line: METHOD  PATH  VERSION
Host: api.example.com\r\n                ← headers, one per line
Content-Type: application/json\r\n
Content-Length: 27\r\n
Authorization: Bearer eyJhbGciOi...\r\n
\r\n                                      ← blank line ends the headers
{"item":"book","quantity":2}              ← body (27 bytes)
```

Every line ends with **CRLF** (`\r\n`). The blank line separates headers from the body.

## HTTP Response

```text
HTTP/1.1 201 Created\r\n                 ← status line: VERSION  CODE  REASON
Content-Type: application/json\r\n
Content-Length: 35\r\n
Location: /api/orders/981\r\n
\r\n
{"id":981,"item":"book","quantity":2}
```

## How the Message Body Is Delimited

Because TCP is a byte stream (doc 05), the receiver must know where the body ends. HTTP/1.1 uses one of:

1. **`Content-Length: N`**: read exactly $N$ bytes.
2. **`Transfer-Encoding: chunked`**: the body is sent in chunks, each prefixed by its size in hexadecimal, ending with a zero-length chunk. Used when the total size is not known in advance (streaming).

```text
HTTP/1.1 200 OK
Transfer-Encoding: chunked

5\r\n
Hello\r\n
7\r\n
, World\r\n
0\r\n
\r\n
```

1. Or the server **closes the connection** (HTTP/1.0 style; no longer efficient).

Mismatched framing between a proxy and a server is the root of *HTTP request smuggling* attacks (doc 42).

## HTTP Methods

| Method | Purpose | Body in request | Safe | Idempotent | Cacheable |
| --- | --- | --- | --- | --- | --- |
| **GET** | Read a resource | No | Yes | Yes | Yes |
| **HEAD** | Like GET but returns headers only | No | Yes | Yes | Yes |
| **POST** | Create a resource / trigger processing | Yes | No | **No** | Rarely |
| **PUT** | Replace a resource entirely (or create at a known URL) | Yes | No | Yes | No |
| **PATCH** | Partially modify a resource | Yes | No | Not guaranteed | No |
| **DELETE** | Remove a resource | Usually no | No | Yes | No |
| **OPTIONS** | Ask which methods/headers are allowed (CORS preflight) | No | Yes | Yes | No |

Definitions:

- **Safe:** does not change server state (read-only).
- **Idempotent:** performing the request $n$ times has the same effect as performing it once. Formally, for a state transformation $f$: $f(f(s))=f(s)$.

Examples: `DELETE /users/5` twice leaves the same final state (user 5 gone; the second call may return 404). `POST /orders` twice creates **two** orders. This is why retries of `POST` are dangerous and why APIs add **idempotency keys** (docs 17, 37).

## HTTP from Code

**Raw HTTP over a socket (Python)** shows there is nothing magical:

```python
import socket

with socket.create_connection(("example.com", 80)) as s:
    s.sendall(b"GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n")
    response = b""
    while chunk := s.recv(4096):
        response += chunk

head, _, body = response.partition(b"\r\n\r\n")
print(head.decode().splitlines()[0])        # e.g. HTTP/1.1 200 OK
```

**A tiny HTTP server: Node.js:**

```js
const http = require("http");

const server = http.createServer((req, res) => {
  if (req.method === "GET" && req.url === "/hello") {
    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ message: "hello" }));
  } else {
    res.writeHead(404).end();
  }
});
server.listen(3000);

// client (Node 18+ has fetch built in)
// const r = await fetch("http://localhost:3000/hello"); console.log(r.status, await r.json());
```

**Python (client with `requests`, server with the standard library):**

```python
import requests
r = requests.get("https://api.example.com/items", params={"id": 42}, timeout=5)
print(r.status_code, r.headers["Content-Type"], r.json())

from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        body = b'{"message":"hello"}'
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

# HTTPServer(("0.0.0.0", 3000), Handler).serve_forever()
```

**C# (.NET): client and minimal API server:**

```csharp
// client: create ONE HttpClient and reuse it (creating one per request exhausts sockets)
using var http = new HttpClient();
HttpResponseMessage resp = await http.GetAsync("https://api.example.com/items?id=42");
Console.WriteLine((int)resp.StatusCode);
Console.WriteLine(await resp.Content.ReadAsStringAsync());

// server (separate project): ASP.NET Core minimal API
// var app = WebApplication.CreateBuilder(args).Build();
// app.MapGet("/hello", () => Results.Json(new { message = "hello" }));
// app.Run();
```

**C and C++ (libcurl, the standard HTTP client library):**

```c
#include <curl/curl.h>

int main(void) {
    CURL *h = curl_easy_init();
    curl_easy_setopt(h, CURLOPT_URL, "https://example.com/");
    curl_easy_setopt(h, CURLOPT_TIMEOUT, 5L);
    CURLcode rc = curl_easy_perform(h);          // prints the body to stdout by default
    long status = 0;
    curl_easy_getinfo(h, CURLINFO_RESPONSE_CODE, &status);
    curl_easy_cleanup(h);
    return rc == CURLE_OK && status == 200 ? 0 : 1;
}
// build: gcc app.c -lcurl      (C++ uses exactly the same API)
```

**Command line:**

```bash
curl -i https://example.com                       # show response headers and body
curl -v https://example.com                       # also show the request and the TLS handshake
curl -X POST https://api.example.com/orders \
     -H "Content-Type: application/json" -d '{"item":"book","quantity":2}'
```

---

# 6.2 HTTP Headers

Headers are `Name: value` pairs that carry metadata about the message. Names are case-insensitive.

## Content-Type

Says what the body is, using a **MIME type**, optionally with a charset.

```text
Content-Type: application/json
Content-Type: text/html; charset=utf-8
Content-Type: multipart/form-data; boundary=----abc123      (file uploads)
Content-Type: application/x-www-form-urlencoded              (HTML form posts)
```

## Accept (and friends)

The client states what it can handle; the server picks (**content negotiation**).

```text
Accept: application/json
Accept-Encoding: gzip, br          ← compression the client understands
Accept-Language: en-IN, hi;q=0.8   ← q = preference weight from 0 to 1
```

The server answers with `Content-Encoding: gzip` when it compresses. Compression cuts text payloads (JSON, HTML) by a large factor and saves bandwidth.

## Authorization

Carries credentials.

```text
Authorization: Bearer <token>                  ← JWT / OAuth access token (doc 18)
Authorization: Basic dXNlcjpwYXNz              ← base64("user:pass")
```

**Base64 is an encoding, not encryption.** Anyone can decode Basic credentials, so they must only be sent over HTTPS.

## Cookie / Set-Cookie

The server stores a small value in the browser with `Set-Cookie`; the browser sends it back on later requests with `Cookie`. This is how stateless HTTP gets sessions.

```text
Set-Cookie: sid=abc123; Max-Age=3600; Path=/; Secure; HttpOnly; SameSite=Lax
Cookie: sid=abc123
```

| Attribute | Effect |
| --- | --- |
| `Secure` | Sent only over HTTPS |
| `HttpOnly` | Not readable by JavaScript (limits XSS damage) |
| `SameSite` | Restricts cross-site sending (CSRF defense): `Strict`, `Lax`, `None` |
| `Max-Age` / `Expires` | Lifetime |
| `Domain` / `Path` | Which requests include the cookie |

## Cache-Control

Controls caching by browsers, CDNs and proxies (doc 23, 52).

| Directive | Meaning |
| --- | --- |
| `max-age=N` | Fresh for $N$ seconds; reuse without asking the server |
| `s-maxage=N` | Like `max-age` but for shared caches (CDNs) |
| `no-store` | Never store the response (sensitive data) |
| `no-cache` | May store, but **must revalidate** with the server before each use |
| `public` / `private` | Any cache may store / only the user's own browser may store |
| `immutable` | Will never change during its freshness lifetime |
| `stale-while-revalidate=N` | May serve stale content for $N$ seconds while refreshing in the background |

**Conditional requests** avoid re-sending unchanged data:

```text
Response:   ETag: "v42"                          (a fingerprint of this version)
Later request:  If-None-Match: "v42"
Response:   304 Not Modified                     (no body: reuse your copy)
```

`Last-Modified` / `If-Modified-Since` work the same way with timestamps.

## User-Agent

Identifies the client software: `User-Agent: curl/8.5.0`. Used for analytics and (carefully) for compatibility handling; it is trivially spoofable, so never use it for security.

## Host

The domain (and port) the request is for: `Host: api.example.com`. It is **mandatory in HTTP/1.1** because one IP address and port can serve many websites (**virtual hosting**); the server picks the site from `Host`. Reverse proxies route by it (doc 28).

## Content-Length

The body size in bytes. It must match exactly, otherwise the receiver misreads the stream (see framing above).

## Other Headers You Will Meet

| Header | Purpose |
| --- | --- |
| `Location` | Target URL for redirects and newly created resources |
| `Origin`, `Access-Control-Allow-Origin` | CORS (doc 18) |
| `X-Forwarded-For`, `X-Forwarded-Proto`, `Forwarded` | Original client IP/protocol, added by proxies |
| `Connection: keep-alive` | Reuse the TCP connection |
| `Retry-After` | How long to wait before retrying (with 429/503) |
| `Strict-Transport-Security` | HSTS: browser must use HTTPS for this site |
| `Content-Disposition` | Suggests a download filename |

---

# 6.3 HTTP Status Codes

The three-digit code tells the client what happened. The first digit is the class.

| Class | Meaning |
| --- | --- |
| **1xx** | Informational (e.g. `101 Switching Protocols` for WebSocket) |
| **2xx** | Success |
| **3xx** | Redirection: the client must take further action |
| **4xx** | **Client error**: the request is wrong; retrying the same request will not help |
| **5xx** | **Server error**: the server failed; a retry may succeed |

## The Important Codes

| Code | Name | When to use / what it means |
| --- | --- | --- |
| **200** | OK | Successful request with a body |
| **201** | Created | A resource was created; include `Location` |
| **204** | No Content | Success with no body (typical for DELETE / PUT) |
| **301** | Moved Permanently | Resource has a new permanent URL; caches and search engines update |
| **302** | Found | Temporary redirect |
| **304** | Not Modified | Reply to a conditional request: use your cached copy |
| **400** | Bad Request | Malformed or invalid request (bad JSON, failed validation) |
| **401** | Unauthorized | **Not authenticated** (missing/invalid credentials). Despite the name, it is about identity |
| **403** | Forbidden | Authenticated but **not allowed** |
| **404** | Not Found | No such resource |
| **409** | Conflict | Conflicts with current state (duplicate, version mismatch) |
| **429** | Too Many Requests | Rate limit exceeded; see `Retry-After` |
| **500** | Internal Server Error | Unexpected failure in the server (a bug) |
| **502** | Bad Gateway | A proxy/gateway got an **invalid response** from the upstream server |
| **503** | Service Unavailable | Server temporarily cannot serve (overloaded, maintenance) |
| **504** | Gateway Timeout | A proxy/gateway got **no response in time** from the upstream |

Others worth knowing: `202 Accepted` (queued for async processing), `307`/`308` (redirects that **preserve the method**, unlike 301/302 which browsers often turn into GET), `405 Method Not Allowed`, `415 Unsupported Media Type`, `422 Unprocessable Content` (well-formed but semantically invalid).

## 401 vs 403, 502 vs 504

```text
401: "Who are you?"           → send credentials (log in)
403: "I know you; not allowed" → logging in again will not help

502: upstream answered with garbage / connection refused / crashed
504: upstream did not answer before the proxy's timeout
```

When debugging behind Nginx or a load balancer (doc 28, 30), 502 usually means the app is down or crashing; 504 means it is alive but too slow.

## Retry Guidance

| Response | Retry? |
| --- | --- |
| 4xx (except 408, 429) | No, fix the request |
| 429 | Yes, after `Retry-After`, with backoff |
| 500, 502, 503, 504 | Yes, with exponential backoff and jitter, **only if the operation is idempotent** (doc 36, 37) |

---

# 6.4 HTTP Versions

## HTTP/1.0

One TCP connection **per request**: connect, request, response, close. Each request pays the handshake and slow-start costs (doc 05).

## HTTP/1.1 (1997)

- **Persistent connections** (keep-alive) by default: reuse one connection for many requests.
- `Host` header (virtual hosting), chunked transfer encoding, caching improvements, range requests.
- **Pipelining** (send several requests without waiting) exists but is effectively unused because of head-of-line blocking.
- **Problem:** on one connection, responses must come back **in order**, so one slow response blocks the ones behind it (**HTTP-level head-of-line blocking**). Browsers work around it by opening about 6 parallel connections per host.

## HTTP/2 (2015)

- **Binary framing** instead of text.
- **Multiplexing:** many concurrent **streams** share **one** TCP connection; responses can interleave, so there is no HTTP-level blocking and no need for 6 connections.
- **Header compression (HPACK)**: repeated headers such as cookies are sent as tiny references.
- Stream prioritization; server push (little used and now largely removed).
- **Remaining problem:** it is still one TCP connection, so a single lost packet stalls **all** streams (**TCP-level head-of-line blocking**).

## HTTP/3 (2022) and QUIC

**QUIC** is a transport protocol built on **UDP** (doc 05) that provides:

- **Independent streams:** loss on one stream does not block others.
- **Built-in TLS 1.3:** the transport and encryption handshakes are combined, so a new connection needs **1 RTT** (and **0-RTT** when resuming a previous session).
- **Connection IDs:** a connection survives a change of IP address (Wi-Fi to mobile data).
- Congestion control and reliability implemented in user space, so they evolve without OS updates.

HTTP/3 is HTTP semantics over QUIC (header compression: QPACK).

## Startup Latency by Version

Time until the first response byte on a **new** connection, ignoring DNS and server time (one round trip = RTT):

| Stack | Round trips |
| --- | --- |
| HTTP/1.1 or 2 + TCP + TLS 1.2 | $1\ (\text{TCP})+2\ (\text{TLS})+1\ (\text{request})=4$ |
| HTTP/1.1 or 2 + TCP + TLS 1.3 | $1+1+1=3$ |
| HTTP/3 (QUIC, first visit) | $1+1=2$ |
| HTTP/3 with 0-RTT resumption | $1$ |

At RTT = 100 ms the difference between 4 and 2 round trips is 200 ms per fresh connection.

## Comparison

| | 1.1 | 2 | 3 |
| --- | --- | --- | --- |
| Format | Text | Binary | Binary |
| Transport | TCP | TCP | QUIC over UDP |
| Concurrency | ~6 connections/host | Multiplexed streams | Multiplexed streams |
| HOL blocking | HTTP level | TCP level | None between streams |
| Encryption | Optional | Effectively required | Always (TLS 1.3) |

**Connection reuse in code** (always reuse connections; creating a new one per request repeats the handshake and slow start):

```js
const https = require("https");
const agent = new https.Agent({ keepAlive: true, maxSockets: 50 });   // Node.js
https.get("https://api.example.com/x", { agent }, (res) => res.resume());
```

```python
import requests
session = requests.Session()             # Python: a Session pools and reuses connections
session.get("https://api.example.com/x")
```

```bash
curl --http2 -I https://example.com      # ask for HTTP/2
curl --http3 -I https://example.com      # needs a curl build with HTTP/3 support
```

---

# 6.5 HTTPS

**HTTPS** is HTTP carried inside **TLS (Transport Layer Security)**. Previously called SSL; only TLS 1.2 and 1.3 are considered secure today.

## What TLS Provides

| Property | Meaning |
| --- | --- |
| **Confidentiality** | Eavesdroppers cannot read the traffic (encryption) |
| **Integrity** | Tampering is detected (authenticated encryption / MAC) |
| **Authentication** | The client can verify it is talking to the real server (certificates) |

Without TLS, anyone on the path (Wi-Fi, ISP, proxy) can read or modify HTTP traffic.

## Building Blocks

**Symmetric encryption** (AES, ChaCha20): one shared key encrypts and decrypts. Very fast, used for the bulk data. Problem: how do both sides get the key safely?

**Asymmetric (public-key) cryptography:** a **key pair**: the **public key** is shared openly; the **private key** stays secret.

- Encrypt with the public key → only the private key decrypts.
- **Sign** with the private key → anyone can verify with the public key.

It is slow, so TLS uses it only for setup (key exchange and authentication), then switches to symmetric encryption.

**Key exchange (Diffie-Hellman).** Two parties create a shared secret over an open channel. With public numbers $p$ (prime) and $g$:

$$A=g^{a}\bmod p,\quad B=g^{b}\bmod p,\quad s=B^{a}\bmod p=A^{b}\bmod p=g^{ab}\bmod p$$

Toy example: $p=23$, $g=5$, private $a=6$, $b=15$. Then $A=5^{6}\bmod23=8$, $B=5^{15}\bmod23=19$, and both compute $s=19^{6}\bmod23=8^{15}\bmod23=2$. An eavesdropper sees $p,g,A,B$ but cannot feasibly recover $s$ (with large numbers or elliptic curves, ECDHE). Because fresh keys are used for every session, past traffic stays safe even if the server's long-term key leaks later: **forward secrecy**.

## Certificates and Certificate Authorities

A key exchange alone does not prove *who* you are talking to. A **certificate** binds a domain name to a public key. It contains:

- the **subject** and the domain names it covers (**Subject Alternative Names**),
- the server's **public key**,
- the **validity period**,
- the **issuer** (the CA) and the CA's **digital signature** over all of this.

A **Certificate Authority (CA)** is a trusted organization that verifies control of a domain and signs certificates. Browsers and operating systems ship with a **trust store** of root CA certificates.

**Chain of trust:**

```text
 Root CA certificate         ← already in your OS/browser trust store (self-signed)
        │ signs
 Intermediate CA certificate ← sent by the server
        │ signs
 Leaf certificate            ← for example.com (sent by the server)
```

The client accepts a server certificate only if all of these hold:

1. The signature chain leads to a trusted root.
2. The current date is within each certificate's validity period.
3. The **hostname matches** a name in the certificate.
4. The certificate is not revoked (OCSP / CRL checks, or short lifetimes).

**Let's Encrypt** (using the ACME protocol) issues free, automatically renewed certificates, typically valid for a few months. **Expired certificates are a top cause of outages**; automate renewal and monitor expiry (doc 41).

## The TLS 1.3 Handshake (1 RTT)

```text
 Client                                              Server
   │ ClientHello                                        │
   │  · supported cipher suites                         │
   │  · key share (ECDHE public value)                  │
   │  · SNI (server name), ALPN (h2, http/1.1)          │
   │ ─────────────────────────────────────────────────► │
   │                                       ServerHello  │
   │                          · chosen cipher, key share│
   │  ◄──── everything below is already encrypted ───── │
   │                        {Certificate, signature,    │
   │                         Finished}                  │
   │ verify certificate chain + hostname + signature    │
   │ {Finished}                                         │
   │ ─────────────────────────────────────────────────► │
   │ ═════════ encrypted application data (HTTP) ═════ │
```

1. **ClientHello:** offered cipher suites, the client's ECDHE public value, the server name (**SNI**) and the desired application protocol (**ALPN**, e.g. `h2`).
2. **ServerHello:** picks the cipher and sends its ECDHE public value. Both sides now compute the same shared secret and derive symmetric session keys.
3. **Server proves identity:** sends its certificate chain and a **signature** made with its private key over the handshake transcript. Only the real owner of the private key can produce it.
4. **Client verifies** the chain, hostname and signature, then sends `Finished`. Application data flows encrypted with the session keys.

**TLS 1.2** needed 2 round trips for the same result. **Session resumption** (tickets) and **0-RTT** (TLS 1.3, replayable, so only safe for idempotent requests) shorten repeat connections. **SNI** is sent in clear text in normal TLS, so observers see the hostname (Encrypted Client Hello hides it).

## HTTPS in Practice

- **TLS termination:** the reverse proxy or load balancer decrypts HTTPS and forwards plain HTTP (or re-encrypts) to the app servers, keeping certificates and CPU-heavy crypto in one place (docs 28, 30).
- **HSTS** (`Strict-Transport-Security: max-age=31536000; includeSubDomains`): tells browsers to refuse plain HTTP for the site.
- **mTLS (mutual TLS):** the server also verifies a client certificate; common for service-to-service traffic (doc 42).
- **Cost:** the handshake (round trips plus crypto) is paid once per connection, which is another reason to reuse connections.

## TLS from Code

**Python:**

```python
import socket
import ssl

ctx = ssl.create_default_context()        # verifies chain and hostname using the system trust store
with socket.create_connection(("example.com", 443)) as raw:
    with ctx.wrap_socket(raw, server_hostname="example.com") as tls:   # server_hostname = SNI + check
        print(tls.version())              # e.g. TLSv1.3
        print(tls.cipher())
        cert = tls.getpeercert()
        print(cert["subject"], cert["notAfter"])
```

**Node.js:**

```js
const tls = require("tls");

const s = tls.connect(443, "example.com", { servername: "example.com" }, () => {
  console.log(s.getProtocol(), "verified:", s.authorized);
  const c = s.getPeerCertificate();
  console.log(c.subject, "expires", c.valid_to);
  s.end();
});
```

**C# (.NET):**

```csharp
using System.Net.Security;
using System.Net.Sockets;

using var tcp = new TcpClient("example.com", 443);
using var tls = new SslStream(tcp.GetStream());
await tls.AuthenticateAsClientAsync("example.com");     // handshake + certificate validation
Console.WriteLine(tls.SslProtocol);
Console.WriteLine(tls.RemoteCertificate?.Subject);
```

**C / C++ (OpenSSL):**

```c
#include <openssl/ssl.h>

// sockfd is an already connected TCP socket to example.com:443
SSL_CTX *ctx = SSL_CTX_new(TLS_client_method());
SSL_CTX_set_default_verify_paths(ctx);                 // load the system's trusted CAs
SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER, NULL);        // reject invalid certificates

SSL *ssl = SSL_new(ctx);
SSL_set_fd(ssl, sockfd);
SSL_set_tlsext_host_name(ssl, "example.com");          // SNI
SSL_set1_host(ssl, "example.com");                     // hostname verification
if (SSL_connect(ssl) != 1) { /* handshake or verification failed */ }

SSL_write(ssl, "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n", 37);
char buf[4096];
int n = SSL_read(ssl, buf, sizeof buf);

SSL_shutdown(ssl);
SSL_free(ssl);
SSL_CTX_free(ctx);
// link with: -lssl -lcrypto   (C++ uses the same API)
```

Never disable certificate verification in production code (for example `rejectUnauthorized: false` in Node or `verify=False` in Python `requests`): that removes the authentication guarantee and enables man-in-the-middle attacks.

**Command line:**

```bash
openssl s_client -connect example.com:443 -servername example.com   # handshake details, cert chain
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates -subject
```

---

# Key Takeaways

- HTTP is a **stateless request/response** protocol: request line + headers + blank line + body. Message length comes from `Content-Length` or chunked encoding, because TCP has no message boundaries.
- Know method semantics: **safe** (GET, HEAD, OPTIONS), **idempotent** (GET, PUT, DELETE), and that **POST is neither**; this determines what can be retried safely.
- Headers carry metadata: `Content-Type`, `Authorization`, `Cookie`, `Cache-Control` (+ `ETag` for revalidation), `Host` (virtual hosting), `Content-Length`.
- Status classes: 2xx success, 3xx redirect, **4xx client error, 5xx server error**. 401 = unauthenticated, 403 = forbidden, 502 = bad upstream response, 504 = upstream timeout.
- HTTP/1.1 reuses connections; HTTP/2 multiplexes streams on one TCP connection; HTTP/3 runs over QUIC/UDP to remove TCP head-of-line blocking and cut handshake round trips.
- **HTTPS = HTTP + TLS**: asymmetric crypto and certificates for authentication and key exchange, then fast symmetric encryption. The chain of trust runs leaf → intermediate → root CA; the client also checks hostname and expiry.
- Reuse connections and automate certificate renewal; never turn off certificate verification.
