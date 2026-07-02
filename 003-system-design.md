# 🏗️ System Design & Architecture Guide

A comprehensive reference for backend engineers — covering authentication, system reliability, distributed tracing, and software architecture patterns.

---

## 📋 Table of Contents

- [OAuth & SSO](#-oauth--single-sign-on)
- [JWT & OAuth Relationship](#-jwt--oauth-relationship)
- [When JWT Tokens Get Stolen](#-when-jwt-tokens-get-stolen)
- [Preventing Downtime in Kubernetes](#-preventing-downtime-in-kubernetes)
- [Before Migrating to Microservices](#-before-migrating-to-microservices)
- [Before Using Kafka](#-before-using-kafka)
- [ClickHouse & OLAP Analytics](#-clickhouse--olap-analytics)
- [Software Architecture Patterns](#-software-architecture-patterns)
- [Timeout](#-timeout)
- [Circuit Breaker](#-circuit-breaker)
- [Backpressure](#-backpressure)
- [OpenTelemetry & Distributed Tracing](#-opentelemetry--distributed-tracing)
- [Quick Reference](#-quick-reference)

---

## 🔐 OAuth & Single Sign-On

> **Key Insight:** OAuth 2.0 is a standard protocol for centralizing authentication. When a company has multiple applications, OAuth lets users log in once and access all of them — this is called SSO (Single Sign-On).

**The 4 Core Roles in OAuth**

```
┌──────────────────────────────────────────────────────────────┐
│                      OAUTH ROLES                             │
│                                                              │
│  Resource Owner  ──  The user who owns the data             │
│  Resource Server ──  Where the user's protected data lives  │
│  Client          ──  The app acting on behalf of the user   │
│  Auth Server     ──  Verifies identity & issues tokens      │
│                                                              │
│  Note: Auth Server and Resource Server can be the same app  │
│        or completely separate applications.                  │
└──────────────────────────────────────────────────────────────┘
```

**Basic OAuth Flow**

```
User                Client App            Auth Server        Resource Server
 │                      │                     │                    │
 │── enter credentials ─►│                     │                    │
 │                      │──── send to auth ───►│                    │
 │                      │                     │── verify ──────────│
 │                      │◄── access token ────│                    │
 │                      │                     │                    │
 │                      │──────────── request + token ────────────►│
 │                      │◄─────────────────── protected data ──────│
```

**SSO Flow with Authorization Code Grant**

Used when a company has multiple separate websites (e.g., Travel, Store, E-Learning).

```
User           Client App          Auth Server (Centralized)
 │                 │                        │
 │─ click Login ──►│                        │
 │                 │──── redirect ─────────►│
 │─────────────────────── enter username/password ──────────────►│
 │                 │◄─── authorization code ─────────────────────│
 │                 │──── exchange code for token ───────────────►│
 │                 │◄─── access token ──────────────────────────│
 │                 │                        │
 │                 │  (user goes to another app)
 │                 │                        │
 │─ click Login ──►│                        │
 │                 │──── redirect ─────────►│
 │                 │                   [session found]
 │                 │◄─── new auth code ─────│  ← no password needed!
 │                 │◄─── access token ──────│
```

**SSO Solutions**
- Build your own Authorization Server
- Use open-source: **Keycloak**, **CAS**

---

## 🔗 JWT & OAuth Relationship

> **Key Insight:** JWT and OAuth are completely independent standards. There is no rule that says you must use JWT with OAuth — but there's a practical reason why most people combine them.

**The Problem Without JWT**

Every resource server must verify tokens by asking the auth server.

```
Client ──► Resource Server A ──► Auth Server (verify token)
Client ──► Resource Server B ──► Auth Server (verify token)
Client ──► Resource Server C ──► Auth Server (verify token)

Result:
- 1,000 requests → 1,000 calls to Auth Server
- Auth Server becomes a bottleneck
- If Auth Server dies → ALL resource servers fail
```

**The Solution With JWT**

Resource servers validate tokens locally using a shared Secret Key — no auth server call needed.

```
Client ──► Resource Server A  [validates JWT locally] ✅
Client ──► Resource Server B  [validates JWT locally] ✅
Client ──► Resource Server C  [validates JWT locally] ✅

Result:
- Zero calls to Auth Server per request
- Auth Server failure does not affect resource servers
```

**JWT Structure**

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VybmFtZSI6ImVrbyJ9.abc123signature
│─────── Header ────│  │───────── Payload ──────│  │─ Signature ─│

Header  → algorithm info (e.g., HS256)       — base64 encoded JSON
Payload → user data (username, user_id, exp) — base64 encoded JSON
Signature → HMAC(base64(header) + "." + base64(payload), SECRET_KEY)
```

**Why JWT is Secure**

```
Attacker decodes payload:  {"username": "eko"}
Attacker changes payload:  {"username": "admin"}
Attacker re-encodes it...

BUT: Cannot regenerate a valid signature without SECRET_KEY

Resource server checks:
  recomputed_sig = HMAC(header + payload, SECRET_KEY)
  if recomputed_sig ≠ token.signature → REJECTED ❌
```

**What to Put in JWT Payload**

| ✅ Safe to Include | ❌ Never Include |
|---|---|
| user_id / username | password |
| email (if static) | sensitive PII |
| expiry time (exp) | payment info |
| role / permissions | secrets |

> Payload is base64-encoded — anyone can decode it. Only the signature guarantees integrity.

---

## 🛡️ When JWT Tokens Get Stolen

> **Key Insight:** If an access token is stolen, it can be used by anyone. OAuth minimizes the damage window through short-lived access tokens and revocable refresh tokens.

**Two Tokens in OAuth**

```
┌──────────────────────────────────────────────────────────────┐
│                   TOKEN LIFECYCLE                            │
│                                                              │
│  Access Token                  Refresh Token                 │
│  ─────────────                 ─────────────                 │
│  Short-lived (e.g., 1 hour)    Long-lived (e.g., 1 month)   │
│  Used for every API request    Used ONLY to get new tokens   │
│  Often JWT (stateless)         Stored in database (stateful) │
│  Validated by resource server  Validated by auth server      │
└──────────────────────────────────────────────────────────────┘
```

**Token Renewal Flow**

```
Client                Resource Server          Auth Server        DB
  │                         │                      │              │
  │──── request + AT ──────►│                      │              │
  │                         │  [AT valid → OK]     │              │
  │◄─── data ───────────────│                      │              │
  │                         │                      │              │
  │  (1 hour passes, AT expires)                   │              │
  │                         │                      │              │
  │──── request + AT ──────►│                      │              │
  │◄─── 401 Unauthorized ───│  [AT expired]        │              │
  │                         │                      │              │
  │────────────────── send Refresh Token ─────────►│              │
  │                         │                      │──── check ──►│
  │                         │                      │◄─── found ───│
  │◄────────────────── new Access Token ───────────│              │
  │                         │                      │              │
  │──── request + new AT ──►│  [valid again] ──────│              │
```

**If Access Token is Stolen**

```
Attacker steals AT
     │
     └──► Can impersonate user for up to 1 hour
          After expiry: AT is useless, attacker can't renew
          (they don't have the Refresh Token)
```

**If Refresh Token is Stolen**

```
Attacker steals RT
     │
     └──► Can keep generating new ATs indefinitely
          BUT: RT is stored in DB — admin can delete it

Remediation:
  1. User notices unauthorized session
  2. Go to "Active Sessions" in account settings
  3. Click "Log out this device"
  4. Server deletes RT from DB
  5. Attacker can no longer generate new ATs ✅
```

**Active Sessions Table (in DB)**

```
┌────────────┬──────────────┬───────────────────────┐
│ session_id │ refresh_token│ user_agent            │
├────────────┼──────────────┼───────────────────────┤
│ abc001     │ RT-xyz...    │ Chrome / MacBook      │
│ abc002     │ RT-def...    │ Safari / iPhone       │
│ abc003     │ RT-ghi...    │ Chrome / Windows      │  ← suspicious?
└────────────┴──────────────┴───────────────────────┘
              Delete this row → device is logged out
```

**Best Practices**

| Concern | Recommendation |
|---|---|
| Access token lifespan | Keep short: 15–60 minutes |
| Refresh token | Always store in database |
| Token revocation | Delete RT row from DB |
| Public Wi-Fi | Tokens can be intercepted — use HTTPS always |
| Forgot to log out | Old AT still active until expiry |

---

## ☸️ Preventing Downtime in Kubernetes

> **Key Insight:** Using Kubernetes does not guarantee zero downtime. Kubernetes is only as reliable as your configuration and capacity planning.

**The 4 Critical Areas**

### 1. Set Correct Min/Max Pod Limits

```
❌ Minimum Pods = 1
   │
   └──► Pod crashes → app is DOWN during restart
        (even 30 seconds of restart = visible downtime)

✅ Minimum Pods = 2 (or more)
   │
   └──► Pod A crashes → Pod B continues serving traffic
        Kubernetes restarts Pod A in background
        Zero user-visible downtime
```

> Maximum pods must match your actual hardware capacity — setting max=100 pods on a 3-node cluster that can only support 20 is useless.

### 2. Know Your Application's Capacity (Performance Testing)

```
Step 1: Run performance test on 1 Pod
        → Find its max RPS (e.g., 150 RPS per Pod)

Step 2: Estimate required pods for peak traffic
        Target: 5,000 RPS
        Per Pod: 150 RPS
        Required: 5,000 / 150 = ~34 Pods

Step 3: Provision hardware accordingly
        Set maxReplicas = 34 (or higher with buffer)
```

> Language (Java, Go, PHP) is NOT what determines RPS. Application logic and optimization determine it.

### 3. Consider Application Startup Time

```
Scenario: App takes 60 seconds to start

Traffic spike hits → Kubernetes triggers autoscale
New Pod starts...
But: 60 seconds before it's ready!

During those 60 seconds:
  - Existing pods absorb all traffic
  - May crash under load
  - New pods can't help in time
```

**Fix:** Trigger autoscaling earlier

```
❌ Scale when CPU = 80% → too late, app already struggling
✅ Scale when CPU = 60% → gives pods time to warm up before peak
```

### 4. Implement a Rate Limiter

```
Designed capacity: 5,000 RPS
Actual traffic:    6,000 RPS

Without rate limiter:
  All 6,000 hit the app → queue overflows → total crash ❌

With rate limiter:
  First 5,000 RPS: served normally ✅
  Remaining 1,000: rejected immediately with 429 ✅
  → 5,000 users happy, 1,000 get a clear error (retry later)
```

> Sacrificing the overflow is better than crashing the entire system.

---

## 🔀 Before Migrating to Microservices

> **Key Insight:** Microservices should be the last resort, not the first instinct. Most teams migrate too early, introducing complexity they don't yet need.

**The Migration Ladder**

```
        Microservices  ◄── Only when team/org complexity demands it
              │
            CQRS        ◄── When search/query patterns become unpredictable
              │
     Optimize Monolith  ◄── Fix the root cause (slow query, bad logic)
              │
           Monolith      ◄── Always start here
```

### Stage 1: Monolith First

```
Advantages:
  ✅ Single codebase — easy to read and debug
  ✅ Simple deployment (one unit)
  ✅ Fast to build and iterate

When to stay here:
  → App runs fine, traffic is manageable, team is small
  → Internal tools, MVPs, early-stage products
```

### Stage 2: Optimize the Monolith

Before rebuilding, diagnose the actual bottleneck:

```
App is slow
    │
    ├── Database slow?
    │       ├── Run EXPLAIN on queries
    │       ├── Add indexes on frequently searched columns
    │       └── Add caching layer (Redis)
    │
    └── Application logic slow?
            ├── Fix inefficient code / long logic chains
            ├── Make sequential tasks asynchronous
            └── Cache expensive computation results
```

### Stage 3: CQRS (Command Query Responsibility Segregation)

**Problem:** Dynamic search filters on millions of rows — adding indexes for every combination slows down writes.

```
Without CQRS:
  One database handles ALL operations
  → Heavy read filters slow down write transactions

With CQRS:
┌─────────────────────┐     ┌──────────────────────────┐
│   Command App       │     │   Query App               │
│                     │     │                           │
│  INSERT / UPDATE /  │     │  Complex search & filter  │
│  DELETE / simple    │     │  Elasticsearch or similar │
│  SELECT by ID       │     │  read-optimized DB        │
│  → MySQL/Postgres   │     │                           │
└─────────────────────┘     └──────────────────────────┘
          │                              ▲
          └───── CDC (Change Data ───────┘
                 Capture) syncs data
                 automatically
```

### Stage 4: Microservices — When to Actually Migrate

The trigger is almost always **organizational**, not technical:

```
Technical performance is already solved by CQRS.
Migrate to microservices when:

├── Team grew from 5 → 50+ developers
│       → One codebase causes too many merge conflicts
│
├── Multiple teams have different release schedules
│       → Team A can't deploy because Team B isn't ready
│
└── Need clear ownership boundaries
        → Team Payment owns payment service
          Team Product owns product service
          No fear of breaking each other's code
```

---

## 📨 Before Using Kafka

> **Key Insight:** Kafka is powerful but complex. Understanding its characteristics prevents wrong architecture decisions and costly production issues.

**8 Key Characteristics**

### 1. Scalability — Designed for Clusters

```
Minimum setup: 3 nodes
Scaling: Add nodes in odd numbers (3 → 5 → 7)
         New node joins cluster, data distributes automatically
         No restart required
```

### 2. Throughput — Extremely High

```
3-machine Kafka cluster → up to 2,000,000 writes/second

Best for:
  ✅ Analytics events (clicks, views, impressions)
  ✅ Log aggregation from millions of users
  ✅ High-frequency financial data

Not ideal for:
  ❌ ~1,000 RPS apps (regular message queue is simpler)
```

### 3. Message Size Limit

```
Default max: 1 MB per message

For large files:
  ❌ Don't put the file inside Kafka
  ✅ Upload file to Cloud Storage → put the URL in Kafka

Warning: Increasing max message size causes consumer timeout issues
```

### 4. Partitions

```
Topic
  ├── Partition 0 ──► Consumer A
  ├── Partition 1 ──► Consumer B
  └── Partition 2 ──► Consumer C

Rules:
  - 1 partition = 1 consumer (within same group)
  - 5 consumers + 3 partitions = 2 consumers idle
  - Too many partitions = coordination overhead + memory cost
```

### 5. Order Guarantee

```
Traditional queue (e.g., RabbitMQ):
  No ordering guarantee

Kafka:
  Order guaranteed WITHIN the same partition

Example — Balance history must be sequential:
  ✅ Correct:  Top-up +100 → Purchase -50 → Balance 50
  ❌ Scrambled: Purchase -50 → Top-up +100 → (wrong calculation)

Trade-off: If one message in a partition is stuck,
           all messages behind it are blocked (head-of-line blocking)
```

### 6. Persistence & Retention

```
Traditional queue: message deleted after consumed
Kafka: messages stored on disk, consumer moves an Offset pointer

┌────────────────────────────────────────┐
│  Partition log (on disk)               │
│  [msg1][msg2][msg3][msg4][msg5]...     │
│                        ▲              │
│                     Consumer Offset   │
└────────────────────────────────────────┘

Benefits:
  → Consumer can rewind offset to reprocess old messages
  → Multiple consumer groups read independently

Retention options:
  → By time:  delete messages older than 30 days
  → By size:  delete oldest when partition exceeds 50 GB
```

### 7. Rebalance Effect

```
Event: New consumer added (autoscale) or consumer dies

Kafka rebalance process:
  → Redistributes partitions to active consumers
  → OLD Kafka versions: ALL consumers pause during rebalance
    (could be 10–30+ seconds of downtime)
  → NEW Kafka versions: improved cooperative rebalance

Recommendation:
  Keep consumer count stable — avoid extreme up/down autoscaling
```

### 8. Consumer Groups

```
                   Topic: Orders
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
  Consumer Group A           Consumer Group B
  (Payment Service)          (Notification Service)
          │                         │
  Processes each order       Sends email/push per order
  independently              independently

Each group has its own offset — no competition for messages
```

---

## 📊 ClickHouse & OLAP Analytics

> **Key Insight:** ClickHouse is a column-oriented OLAP database built for analyzing huge volumes of data fast — it complements an OLTP database like PostgreSQL/MySQL, it does not replace it.

**OLTP vs OLAP**

```
┌──────────────────────────────────────────────────────────────┐
│                      OLTP vs OLAP                            │
│                                                              │
│  OLTP (Online Transaction Processing)                        │
│    Row-oriented    →  PostgreSQL, MySQL, Oracle              │
│    Optimized for many small, fast transactions                │
│    INSERT / UPDATE / DELETE on individual rows                │
│                                                              │
│  OLAP (Online Analytical Processing)                          │
│    Column-oriented →  ClickHouse                              │
│    Optimized for reading huge volumes for analysis            │
│    Aggregation, business insight, large scale SELECT          │
└──────────────────────────────────────────────────────────────┘
```

**Syncing Data: Batch ETL vs CDC**

```
Batch ETL (traditional):
  Extract → Transform → Load   (runs on a schedule, e.g. hourly)
  OLAP data is stale until the next run ❌

CDC (Change Data Capture) — recommended:
  OLTP write ──► WAL (Write-Ahead Log) ──► CDC tool captures the change
                                                    │
                                                    ▼
                                        Streamed into OLAP in seconds ✅
```

**Demo Architecture**

```
PostgreSQL (OLTP)
      │  WAL
      ▼
  Debezium (CDC tool)
      │
      ▼
  Apache Kafka (message broker)
      │
      ▼
  ClickHouse (OLAP)
      │
      ▼
  Spring Boot (data generator + dashboard)

Tables captured by Debezium:
  customers, products, orders, order_items
```

**ClickHouse's Native Kafka Integration**

```
No bridge app needed (e.g. Logstash) — ClickHouse reads Kafka natively
via a declarative Kafka Engine table.

      Kafka topic
          │
          ▼
  [Kafka Engine table]   ← staging / ingestion table
          │
          │  Materialized View (acts like a trigger)
          ▼
  [Main table]            ← MergeTree / ReplacingMergeTree
```

**Table Engines**

| Engine | Behavior | Best For |
|---|---|---|
| **MergeTree** | Append-only — data keeps growing | Time-series / chronological event data |
| **ReplacingMergeTree** | Overwrites the row with the same ID, keeping the latest by timestamp | Mutable entities (customers, products) — avoids expensive OLAP `UPDATE`s |

**5 Analytics Features (Demo Dashboard)**

Demo app generates 5 orders every 2 seconds to simulate live traffic.

### 1. Streaming

```
One query, multiple time windows — countIf() / sumIf():
  → order count & revenue for last 1m / 5m / 15m / 1h, simultaneously

lagInFrame() compares the current window vs the previous one
  → reveals velocity / acceleration of incoming data
```

### 2. Funnel

```
windowFunnel() tracks an ordered sequence of statuses per order:

  place ──► paid ──► ship ──► delivered

Shows conversion rate at each step and where customers drop off
```

### 3. Distribution (Cardinality Estimation)

```
Problem: count(DISTINCT user_id) over billions of rows is very expensive

Solution: uniqCombined() — HyperLogLog algorithm
  → Approximates the unique count
  → ~1% margin of error
  → Instant even at billion-row scale
```

### 4. Pre-Aggregated

```
AggregatingMergeTree + Materialized View
  → Rolls raw events up into per-minute aggregates as they arrive
  → Dashboard queries the small pre-aggregated table, not raw events
  → Chart loads instantly regardless of underlying data volume
```

### 5. Time Travel

```
Data is append-only (never overwritten in place)
  → Move a time slider to any past moment
  → Reconstruct the exact state of the data at that point in time
  → Useful for auditing / historical comparison
```

**Conclusion**

```
ClickHouse complements an OLTP database — it does NOT replace
PostgreSQL or MySQL.

  OLTP → source of truth, real-time transactions
  OLAP → analytics, reporting, business insight

Note: OLAP data has a small replication lag — not guaranteed to be
query-ready at the exact same millisecond it was inserted into OLTP.
```

> Demo source code: `oltp-olap-demo` (GitHub)
> Stack: PostgreSQL → Debezium → Kafka → ClickHouse → Spring Boot

---

## 🏛️ Software Architecture Patterns

> **Key Insight:** No pattern is universally right or wrong — only appropriate or inappropriate for a given business context.

**Classification**

```
┌──────────────────────────────────────────────────────────────┐
│               ARCHITECTURE CLASSIFICATION                    │
│                                                              │
│  MONOLITHIC                    DISTRIBUTED                   │
│  ─────────                     ───────────                   │
│  Single deployment unit        Multiple independent services │
│  Simple, fast to build         Scalable, fault-tolerant      │
│  Great for small teams         Needed for large teams        │
│  Low operational overhead      High operational complexity   │
└──────────────────────────────────────────────────────────────┘
```

**Partitioning Approaches**

```
Technical Partitioning          Domain Partitioning
─────────────────────           ──────────────────
Group by function:              Group by business domain:
  Presentation Layer              /customer  (UI + logic + data)
  Business Layer                  /product   (UI + logic + data)
  Persistence Layer               /payment   (UI + logic + data)
  Database Layer

Basis: DDD (Domain-Driven Design) by Eric Evans
```

---

### A. Layered Architecture (n-Tier)

```
Presentation Layer   ← handles HTTP requests / responses
       ↓
Business Layer       ← core logic, rules, validation
       ↓
Persistence Layer    ← data access (repositories, queries)
       ↓
Database Layer       ← actual DB storage

Principle: "Layers of Isolation"
  → Changes in lower layers don't break upper layers
  → DB column renamed? Only Persistence Layer changes.
```

---

### B. MVC (Model-View-Controller)

```
         HTTP Request
              ↓
         Controller  ──── reads/writes ────► Model (data)
              │
              └──── renders ────────────────► View (HTML template)
                                                    ↓
                                            HTTP Response
```

---

### C. Hexagonal Architecture (Ports & Adapters)

**Problem it solves:** Application logic tied to a specific technology (e.g., MySQL) — hard to swap.

```
┌──────────────────────────────────────────────────────┐
│                 Core Application                     │
│                                                      │
│   Business Logic  ←──── Port (Interface/Contract)   │
│                              │                       │
└──────────────────────────────┼───────────────────────┘
                               │
                    ┌──────────┴──────────┐
                    ▼                     ▼
              Adapter (MySQL)       Adapter (PostgreSQL)
              Adapter (Midtrans)    Adapter (Xendit)

Swap payment gateway: replace Adapter only, core logic untouched
```

---

### D. Microkernel (Plugin Architecture)

```
┌──────────────────────┐
│    Core System       │   ← minimal, stable engine
│  (management only)   │
└──────────┬───────────┘
           │
    ┌──────┼──────┐
    ▼      ▼      ▼
Plugin1  Plugin2  Plugin3  ← features added/removed independently

Examples: VS Code extensions, Chrome extensions, Android Studio plugins
```

---

### E. Client-Server

```
Client (UI)  ←──── TCP/IP (HTTP) ────►  Server (Logic + DB)
  Web
  iOS         All share one server
  Android
```

---

### F. Master-Slave & Master-Master

```
Master-Slave:
  All writes → Master
  Master replicates → Slave(s)
  Master fails → Slave promoted to Master

Master-Master:
  Any node accepts writes
  Nodes replicate to each other
  Higher availability, risk of write conflicts

Used by: MySQL, PostgreSQL, Elasticsearch, Cassandra
```

---

### G. Peer-to-Peer (P2P)

```
Every node is both Client AND Server

Node A ◄──► Node B ◄──► Node C
  ▲                        │
  └────────────────────────┘

No central point of failure
Used by: BitTorrent, Bitcoin/blockchain
```

---

### H. Microservices

```
API Gateway (entry point)
     │
     ├──► User Service    (owns user DB)
     ├──► Order Service   (owns order DB)
     ├──► Payment Service (owns payment DB)
     └──► Product Service (owns product DB)

Rules:
  - Each service has its own database (Bounded Context)
  - Services communicate via API — NEVER share DB directly
  - Each service deployable independently
```

---

### I. Event-Driven Architecture

**Problem it solves:** Synchronous microservices create cascading failures — if Service A is down, Service B can't continue.

```
Synchronous (fragile):
  Order Service ──► Payment Service  [if payment is down → orders fail]

Asynchronous (resilient):
  Order Service ──► [Message Broker] ──► Payment Service
                                     └──► Notification Service

Trade-off: Data is eventually consistent (small delay), not instant
```

---

### J. Pipeline Architecture (Data Pipeline)

```
Data Stream Input
     │
     ▼
 [Filter 1: parse]
     │
     ▼
 [Filter 2: transform]
     │
     ▼
 [Filter 3: aggregate]
     │
     ▼
 Output / Storage

Used by: AWS Lambda, Google Cloud Functions, log processing systems
```

---

### K. Space-Based Architecture

**Problem it solves:** Database becomes a bottleneck during extreme traffic spikes (flash sales, ticket wars).

```
Traditional:
  Request → App → Database  [DB is the bottleneck]

Space-Based:
  Request → App → In-Memory Data Grid (RAM cluster)
                       │
               (async, background)
                       ↓
                   Database  [written later, not blocking]

Trade-off: Complexity increases significantly; data durability risk
```

---

**Combining Patterns** (Common in Production)

```
Example architecture:
  Microservices  ← inter-service structure
    └── Event-Driven  ← communication between services
          └── Hexagonal  ← internal code structure per service
```

---

## ⏱️ Timeout

> **Key Insight:** Forgetting to set a timeout is one of the most common causes of cascading application failure in distributed systems.

**What Happens Without a Timeout**

```
App ──── query ────► Database  (network issue — no response)
 │
 └──► Thread waits... and waits... and waits...

New request arrives → new thread waits
New request arrives → another thread waits

Thread pool exhausted:
  All threads blocked waiting for DB responses
  No threads left to serve new users
  App appears online but is completely frozen ❌
```

**How to Set the Right Timeout**

```
Internal systems (own DB / Redis):
  → You control the infrastructure
  → Set based on worst-case acceptable latency (e.g., 30s)

Third-party APIs (payment gateways, banks, logistics):
  → Read their documentation for their timeout values
  → Set YOUR timeout slightly higher than theirs
```

**The Double-Charge Trap**

```
Payment gateway processes transactions up to 30 seconds.
You set timeout to 5 seconds.

Second 5:  Your app gives up → tells user "failed"
Second 10: Payment gateway succeeds → money deducted

User retries payment → Double charge! 💸

Fix: Set your timeout to 35 seconds (above the 30s gateway limit)
```

**Golden Rule**

```
Your app timeout > Third-party server timeout

Example:
  Bank processes in max 30s
  Your timeout: 35s

  → Your app always waits long enough for the bank to respond
  → Data stays consistent
```

---

## ⚡ Circuit Breaker

> **Key Insight:** When a downstream service degrades, a Circuit Breaker prevents it from dragging your entire system down with it.

**The Problem**

```
Your App (100 RPS capacity)
     │
     └──► Third-Party API (20 RPS capacity)

Excess 80 RPS pile up in memory queue
Response time grows exponentially
Eventually YOUR app crashes — not theirs ❌
```

**The Circuit Breaker Pattern (3 States)**

```
┌─────────────────────────────────────────────────────────────┐
│                   CIRCUIT BREAKER STATES                    │
│                                                             │
│   CLOSED ──────────────────────────────────────────────    │
│   (normal)                                                  │
│   All requests flow through to third-party                  │
│   Monitor: response time + error rate                       │
│       │                                                     │
│       │ threshold breached (e.g., >5s response for 10s)    │
│       ▼                                                     │
│   OPEN ────────────────────────────────────────────────    │
│   (tripped)                                                 │
│   ALL requests rejected immediately (no network call)       │
│   Fallback triggered (backup API / error message)           │
│   Timer starts (e.g., 60 seconds rest period)               │
│       │                                                     │
│       │ rest period ends                                    │
│       ▼                                                     │
│   HALF-OPEN ───────────────────────────────────────────    │
│   (testing)                                                 │
│   Let 10% of traffic through as probe                       │
│       │                                                     │
│       ├── success → back to CLOSED (fully open) ✅          │
│       └── failure → back to OPEN (rest again) ↩            │
└─────────────────────────────────────────────────────────────┘
```

**Fallback Strategies When Circuit is Open**

```
Option 1: Route to backup provider
  Midtrans down → automatically try Xendit

Option 2: User-friendly message
  "Payment service is temporarily busy. Please try again later."

Option 3: Return sentinel value
  Return -1 → frontend hides the payment button temporarily
```

**Implementation**

```
Don't build this from scratch — use a battle-tested library:

  Java:    Resilience4j
  Node.js: opossum
  Go:      gobreaker
  Python:  circuitbreaker
```

---

## 🌊 Backpressure

> **Key Insight:** Backpressure occurs when incoming requests arrive faster than your system can process them. Without a strategy, the queue grows unboundedly until the system crashes.

**What is Backpressure?**

```
Client sends: 100 RPS
App processes: 50 RPS
Unprocessed:  +50 per second

Second 1: 50 waiting
Second 2: 100 waiting
Second 3: 150 waiting
Second N: Out of memory → CRASH ❌
```

**5 Solutions**

### 1. Multi-Threading

```
1 thread processes 5 RPS
Target: 100 RPS → need 20–30 threads

Config example (e.g., Tomcat/Spring):
  server.tomcat.threads.max=30

Best for: blocking I/O (DB queries, external API calls)
Caution:  CPU-intensive tasks → too many threads = context switching overhead
```

### 2. Horizontal Scaling

```
1 server: 50 RPS
                    ┌──► Server A (50 RPS)
Client ──► Load Balancer
                    └──► Server B (50 RPS)
Total: 100 RPS

Add more servers = linear capacity increase
No code changes required
Limitation: budget (cost of servers)
```

### 3. In-App Queue

```
Spike: 110 RPS for 2 seconds (normal is 100 RPS)

Without queue: 10 requests dropped ❌
With queue:    10 extra requests buffered in memory
               Processed when traffic subsides ✅

Languages with built-in support:
  Java: ThreadPoolExecutor queue
  Node.js: event loop queue
```

### 4. Rate Limiting (Reject Overflow)

```
Capacity: 100 RPS

Traffic spike: 1,000 RPS

Without rate limiter:
  1,000 requests → queue fills RAM → OOM crash ❌

With rate limiter:
  First 100 RPS: processed normally ✅
  Remaining 900 RPS: immediately rejected with 429 ✅
  → System remains stable for valid users

Response to rejected requests:
  HTTP 429: "Too Many Requests"
  Message: "System is busy, please try again shortly."
```

### 5. Message Broker (Async Buffer)

```
Synchronous (fragile under load):
  Client ──HTTP──► Backend ──► DB

Asynchronous with broker:
  Client ──► [Message Broker] ──► Backend (consumes at 50 RPS)
              (absorbs the spike)
              can hold millions
              of pending messages

Example: Kafka, RabbitMQ, Google Pub/Sub

Trade-off: response is no longer instant — use only for
           non-real-time operations (notifications, analytics, etc.)
```

**Choosing the Right Strategy**

```
Traffic spike is small & brief     → In-App Queue
Need more sustained capacity       → Multi-Threading or Horizontal Scaling
Traffic is wildly unpredictable    → Rate Limiter (protect from abuse)
Processing can be async            → Message Broker
Budget is the only constraint      → Horizontal Scaling (just add servers)
```

---

## 🔭 OpenTelemetry & Distributed Tracing

> **Key Insight:** Logs alone are not enough for microservices debugging. OpenTelemetry provides a language-agnostic standard for tracing a single request across every service it touches — turning hours of log hunting into seconds of visual inspection.

**The Problem Without Distributed Tracing**

```
User reports: "Create order returned 500 Internal Server Error"

           Order Service
               │
               ├──► Payment Service
               │         │
               │         └──► Product Service
               │
               └──► Notification Service

Question: Which service caused the 500?

Developer has to:
  1. Search logs in Order Service       — manually grep
  2. Search logs in Payment Service     — manually grep
  3. Search logs in Product Service     — manually grep
  4. Search logs in Notification Service— manually grep
  → Time-consuming, error-prone, no visual overview
```

**The Traditional (Manual) Workaround: Request ID**

```
✅ Simple but manual:

  1. Entry service (Order) generates X-Request-ID: "req-001"
  2. Every downstream call must forward this header
  3. Every log line must include the Request ID

Problems:
  - One service forgets to forward the header → trace breaks
  - No visualization — still just logs
  - 100 services = 100 log files to check manually
```

**The Solution: OpenTelemetry**

```
┌──────────────────────────────────────────────────────────────┐
│                    OPENTELEMETRY OVERVIEW                    │
│                                                              │
│  Standard SDK (per language)  →  Trace data  →  Collector   │
│                                                              │
│  Java: otel-javaagent.jar  (zero code change, agent mode)   │
│  Go:   otel-go SDK         (instrument HTTP client)         │
│  Node.js: @opentelemetry/* (setup + start.js)               │
│  Bun/TS: @opentelemetry/* (same as Node.js)                 │
│                                                              │
│  Collector options:                                          │
│    Free/OSS:  Jaeger UI, Zipkin                             │
│    Commercial: Datadog, New Relic, Dynatrace                 │
└──────────────────────────────────────────────────────────────┘
```

**Core Concepts**

| Concept | Description |
|---|---|
| **Trace ID** | Unique ID assigned to one end-to-end request; propagated automatically across all services |
| **Span** | One unit of work (an HTTP call, a DB query); each span belongs to a Trace |
| **Parent Span** | The caller's span; child spans appear nested underneath it |
| **Exporter** | The SDK component that sends trace data to the collector |
| **Collector** | Central backend (Jaeger, Datadog, etc.) that stores and visualizes traces |

**How Trace ID Propagates**

```
User request arrives at Order Service
        │
        ▼
  [SDK generates Trace ID: "abc-123"]
        │
        ├──► HTTP call to Payment Service
        │       Header: traceparent: abc-123
        │       Payment SDK: "I have a Trace ID, I'll use it"
        │           │
        │           └──► HTTP call to Product Service
        │                   Header: traceparent: abc-123
        │                   (continues propagating automatically)
        │
        └──► HTTP call to Notification Service
                Header: traceparent: abc-123

All spans share Trace ID "abc-123" → Jaeger can reconstruct the full tree
```

**Setup Per Language**

```
Java — agent mode (zero application code change):
  Dockerfile:
    ADD otel-javaagent.jar /app/
    ENV JAVA_TOOL_OPTIONS="-javaagent:/app/otel-javaagent.jar"
    ENV OTEL_SERVICE_NAME=order-service
    ENV OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4318

Go — SDK mode (must instrument HTTP client manually):
  // setup otel exporter
  tp := initTracer()
  defer tp.Shutdown(ctx)

  // wrap http client so trace context is forwarded
  client := &http.Client{
    Transport: otelhttp.NewTransport(http.DefaultTransport),
  }

Node.js / Bun — SDK mode:
  // start.js loaded before app starts
  const sdk = new NodeSDK({ traceExporter, instrumentations })
  sdk.start()
  // all HTTP calls and DB queries traced automatically
```

**What You See in Jaeger UI**

```
Trace: abc-123  (total: 312ms)
│
├── Order Service: POST /order              [0ms – 312ms]
│     │
│     ├── Product Service: GET /product/1  [5ms – 48ms]
│     │     └── DB: SELECT * FROM products [8ms – 20ms]
│     │           db.statement: SELECT * FROM products WHERE id=1
│     │
│     ├── Payment Service: POST /payment   [50ms – 180ms]
│     │     └── DB: INSERT INTO payments   [60ms – 100ms]
│     │
│     ├── Notification: POST /notify       [185ms – 240ms]
│     │
│     └── DB: UPDATE orders SET status=   [245ms – 290ms]

Color coding:
  ■ Green  = success
  ■ Red    = error (500, timeout, connection refused)
```

**Debugging an Error with Jaeger**

```
Scenario: Order Service returns 500

Step 1: Find trace by Trace ID in Jaeger
        (or filter by service = "order-service", tag = error=true)

Step 2: Expand the trace tree

Step 3: Find the red span
        → Product Service: GET /product/999  [404 Not Found]

Step 4: Root cause identified in seconds:
        Order Service called a product that doesn't exist,
        didn't handle the 404, and propagated it as a 500

Without Jaeger: 30–60 minutes of log digging
With Jaeger:    < 60 seconds to find the red span
```

**Best Practice: Return Trace ID to the Client**

```
HTTP Response Header:
  X-Trace-Id: abc-123-def-456

Why:
  → When a user reports an error, they can include the Trace ID
  → Support / developer pastes the ID into Jaeger → instant root cause

User: "I got an error, Trace ID: abc-123-def-456"
Dev:  [paste into Jaeger search]
Dev:  "Found it — Product Service returned 404 for product ID 999"
      Root cause identified in < 1 minute
```

**OpenTelemetry with Message Brokers**

```
Works beyond HTTP — supports Kafka, RabbitMQ, Redis:

Payment Service ──► Kafka topic: payments
                        │   (trace context in message headers)
                        ▼
              Notification Service consumes message
                        │
                        ▼
              Trace continues — same Trace ID visible in Jaeger

Jaeger shows:
  Payment Service: publish to Kafka  [span]
    └── Notification Service: consume from Kafka  [child span]
```

**Implementation Checklist**

```
Per service:
  ✅ Add OpenTelemetry SDK (or agent for Java)
  ✅ Set OTEL_SERVICE_NAME (unique per service)
  ✅ Set OTEL_EXPORTER_OTLP_ENDPOINT (point to collector)
  ✅ Instrument HTTP client (Go, Node.js, Bun)
  ✅ Return X-Trace-Id header in HTTP responses

Infrastructure:
  ✅ Deploy Jaeger (or chosen collector)
  ✅ Expose Jaeger UI (port 16686)
  ✅ Open OTLP port (4317 gRPC / 4318 HTTP)
```

---

## 📌 Quick Reference

| Topic | Core Problem | Key Solution |
|---|---|---|
| **OAuth / SSO** | Login fragmented across apps | Centralized Auth Server + Authorization Code Grant |
| **JWT** | Auth Server becomes bottleneck | Resource servers validate JWT locally with Secret Key |
| **Stolen Token** | Access token compromised | Short-lived AT + DB-backed RT (revocable) |
| **Kubernetes** | App still crashes despite K8s | Min pods ≥ 2, performance test, early autoscale trigger |
| **Microservices** | Migrating too early | Monolith → Optimize → CQRS → Microservices (last resort) |
| **Kafka** | Wrong tool selection | Understand partitions, ordering guarantees, and 1MB limit first |
| **ClickHouse (OLAP)** | OLTP DB not built for analytics at scale | Column-oriented store fed via CDC (Debezium → Kafka) in near real-time |
| **Architecture** | Wrong pattern for context | No right/wrong — only appropriate/inappropriate |
| **Timeout** | App hangs forever | Set timeout above third-party server's own timeout |
| **Circuit Breaker** | Downstream failure cascades | Open circuit → drop requests → fallback → half-open test |
| **Backpressure** | Requests pile up faster than processed | Queue / Rate limit / Broker depending on traffic pattern |
| **OpenTelemetry** | Impossible to trace errors across services | SDK per service + Trace ID propagation + Jaeger UI |
