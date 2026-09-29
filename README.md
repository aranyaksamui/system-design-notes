# System Design & Backend Architecture Notes

Each roadmap section has its own document. Read them **in order**: every document only depends on the ones before it.

| # | File | Topic |
| --- | --- | --- |
| 00 | `00-prerequisites.md` | Programming fundamentals, data structures & algorithms |
| 01 | `01-computer-systems-fundamentals.md` | How a computer works, CPU & memory |
| 02 | `02-operating-systems.md` | Processes, threads, scheduling, concurrency, memory, I/O |
| 03 | `03-networking-fundamentals.md` | Networks, OSI, TCP/IP |
| 04 | `04-ip-networking.md` | IP addressing, routing, NAT, DNS |
| 05 | `05-tcp-udp-network-communication.md` | TCP, UDP |
| 06 | `06-http-web-fundamentals.md` | HTTP, headers, status codes, versions, HTTPS |
| 07 | `07-web-architecture-fundamentals.md` | Client-server, request lifecycle, web servers |
| 08 | `08-oop-fundamentals.md` | Classes, relationships, advanced OOP |
| 09 | `09-software-design-principles.md` | DRY, KISS, SOLID-style principles |
| 10 | `10-design-patterns.md` | Creational, structural, behavioral, backend patterns |
| 11 | `11-dbms-fundamentals.md` | Relational model, keys, SQL |
| 12 | `12-database-internals.md` | Storage, indexing, query execution |
| 13 | `13-database-design.md` | Normalization, transactions, ACID |
| 14 | `14-database-concurrency.md` | Isolation levels, locks, MVCC, deadlocks |
| 15 | `15-database-scaling.md` | Replication, partitioning, sharding, CAP |
| 16 | `16-nodejs-fundamentals.md` | V8, libuv, event loop, async |
| 17 | `17-building-backend-apis-nodejs.md` | Express/Fastify, REST, API design |
| 18 | `18-authentication-authorization.md` | Sessions, JWT, RBAC, web security |
| 19 | `19-backend-architecture.md` | Layered, MVC, clean, hexagonal, DI |
| 20 | `20-lld-low-level-design.md` | LLD process, UML, LLD problems |
| 21 | `21-hld-high-level-design.md` | Requirements, scalability, performance, availability |
| 22 | `22-load-balancing.md` | Algorithms and concepts |
| 23 | `23-caching.md` | Strategies and problems |
| 24 | `24-redis.md` | Data structures, use cases, scaling |
| 25 | `25-message-queues.md` | Sync vs async, queue concepts |
| 26 | `26-event-driven-architecture.md` | Events, pub/sub, event sourcing, CQRS |
| 27 | `27-apache-kafka.md` | Topics, partitions, delivery semantics |
| 28 | `28-nginx.md` | Reverse proxy, load balancer, SSL termination |
| 29 | `29-docker.md` | Images, containers, networking, volumes, Compose |
| 30 | `30-reverse-proxy-production-architecture.md` | Combining components |
| 31 | `31-microservices.md` | Monolith vs microservices |
| 32 | `32-service-discovery.md` | Registry, client/server-side discovery |
| 33 | `33-distributed-systems-fundamentals.md` | Failures, CAP, consistency, coordination |
| 34 | `34-distributed-data-systems.md` | Replication models, quorum, consistent hashing |
| 35 | `35-distributed-transactions.md` | 2PC, Saga |
| 36 | `36-reliability-engineering.md` | Fault tolerance, retries, failure handling |
| 37 | `37-api-reliability.md` | Timeouts, idempotency, backpressure |
| 38 | `38-rate-limiting.md` | Algorithms and implementation |
| 39 | `39-api-gateway.md` | Responsibilities and architecture |
| 40 | `40-observability.md` | Logs, metrics, traces |
| 41 | `41-monitoring-alerting.md` | RED, USE, alerting |
| 42 | `42-security-architecture.md` | Application, network, identity security |
| 43 | `43-cloud-fundamentals.md` | Compute, storage, networking on AWS |
| 44 | `44-container-orchestration.md` | Kubernetes |
| 45 | `45-cicd.md` | CI, CD, tools |
| 46 | `46-infrastructure-deployment.md` | IaC, deployment strategies, environments |
| 47 | `47-advanced-distributed-systems.md` | Consensus, LSM trees, advanced databases |
| 48 | `48-nosql.md` | Key-value, document, wide-column, graph |
| 49 | `49-advanced-database-concepts.md` | WAL, MVCC, amplification, Bloom filters |
| 50 | `50-search-systems.md` | Inverted index, ranking, Elasticsearch |
| 51 | `51-file-object-storage.md` | Block vs object storage, multipart, presigned URLs |
| 52 | `52-cdn.md` | Edge, caching, invalidation |
| 53 | `53-real-time-systems.md` | WebSockets, SSE, long polling |
| 54 | `54-large-scale-system-design-patterns.md` | Classic system design problems |
| 55 | `55-system-design-estimation.md` | Back-of-the-envelope calculations |
| 56 | `56-system-design-trade-offs.md` | Trade-off reasoning |
| 57 | `57-complete-dependency-chain.md` | Full learning order |
| 58 | `58-four-major-learning-levels.md` | The four levels |

**Conventions used in every document**

- Math is written in KaTeX: inline as `$...$`, block as `$$...$$`.
- Code is always in fenced code blocks with a language tag.
- Each document ends with a short **Key Takeaways** list.
