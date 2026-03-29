# Task 6 — Fault Tolerance

**Project:** StreamVibe MVP
**Authors:** Mike Ivanov
**Date:** March 2026

---

## Table of Contents

1. [Definitions](#definitions)
2. [Failure Scenarios](#1-failure-scenarios)
3. [Statefulness](#2-statefulness)
4. [Availability Strategy](#3-availability-strategy)
5. [Fault-Tolerance Patterns](#4-fault-tolerance-patterns)

---

## Definitions

**Fault tolerance** — the ability of a system to continue operating correctly when one or more components fail. A fault-tolerant system detects failures and compensates for them without visible impact on users.

**Resilience** — a broader concept: the system's ability not only to survive failures, but to recover quickly and adapt. Resilience includes graceful degradation (serving partial functionality) and self-healing (auto-restart, failover).

In our StreamVibe MVP we focus on both: we tolerate infrastructure failures via redundancy, and we build resilience into the application layer via patterns like circuit breakers and fallbacks.

---

## 1. Failure Scenarios

We identify what can fail, what happens when it does, and how the system responds.

| What Fails | User Impact | System Response |
|------------|------------|-----------------|
| **PostgreSQL (RDS)** | Writes fail for 30–60 sec during failover | RDS Multi-AZ promotes standby automatically. Read replicas continue serving GET requests. Brief write errors, then full recovery. |
| **ElastiCache (Redis)** | Higher latency; users with expired tokens must re-login | Multi-AZ failover promotes replica (~10 sec). Cache-aside pattern falls through to PostgreSQL. Service continues at higher latency. |
| **Auth Service** | No new logins; already logged-in users unaffected | Existing JWT tokens are stateless and remain valid (15 min TTL). API Gateway validates JWTs locally. Only new logins wait for recovery. |
| **API Gateway** | Complete outage | Multiple instances behind load balancer. Health checks remove unhealthy ones. If all fail — full outage until restart. |
| **Video / Comment / Search Service** | Specific feature unavailable | Gateway returns 503 for that service. Other features keep working. Cached data served where possible. |
| **RabbitMQ** | Video processing delayed | Messages are durable — nothing is lost. After recovery, pipeline resumes. Videos already on S3 are safe. |
| **S3 / CloudFront** | Video playback affected | S3 has 99.99% SLA — outage is extremely rare. CDN serves cached content. If CDN fails, fallback to S3 origin (higher latency). |

### Cascading Failures

The biggest risk is one failure causing a chain reaction:

- **Redis down → DB overloaded:** All cache misses hit PostgreSQL. Connection pools exhaust, latency spikes. *Mitigation:* Bulkhead pattern — each service has its own DB connection pool with hard limits. Circuit breaker trips and returns 503 instead of queuing.
- **Slow Auth → Gateway blocks:** Slow auth calls block gateway threads, making everything appear down. *Mitigation:* 2-second timeout on auth calls. Circuit breaker opens after 5 failures. Existing JWTs still validated locally.

---

## 2. Statefulness

### Is our application stateful?

**Application layer: stateless. Data layer: stateful.**

All our services (API Gateway, Auth, Video, Search, Comments, Engagement, Upload) are stateless — they hold no data in memory between requests. Everything is stored in external data stores. This means any service instance can be killed and replaced instantly — it reconnects to the data stores and resumes work.

The stateful components are the data stores themselves: PostgreSQL, ElastiCache, OpenSearch, S3, RabbitMQ. These hold the actual data and are managed by AWS with built-in replication and failover.

### Why this matters for fault tolerance

| | Stateless (our services) | Stateful (data stores) |
|---|---|---|
| **Main concern** | Availability | Availability + Consistency |
| **Recovery** | Trivial — spin up a new instance | Complex — new replica must sync data |
| **Our approach** | Auto-scaling, health checks, load balancing | Delegate to AWS managed services (RDS Multi-AZ, ElastiCache Multi-AZ) |

### CAP Theorem

The CAP theorem says a distributed data store can guarantee only two of three: **Consistency**, **Availability**, **Partition tolerance**. Since network partitions are inevitable, we choose between CP and AP per data store:

| Data Store | Choice | Why |
|------------|--------|-----|
| **PostgreSQL (RDS)** | CP | Strong consistency for user data, likes, playlists. Brief unavailability during failover is acceptable. |
| **ElastiCache (Redis)** | AP | Eventual consistency is fine for cached feeds and counters. Refresh tokens use synchronous replication. |
| **OpenSearch** | AP | Search index is eventually consistent. A video appearing in search 1–2 seconds late is acceptable. |
| **S3** | AP | Strong read-after-write consistency since 2020. Effectively CP for our use case. |

### Reducing Dependencies

Every dependency reduces overall availability (`A(total) = A(a) × A(b)`). We minimize this impact by:

- **Async where possible** — video processing uses RabbitMQ. Upload Service returns 202 Accepted and doesn't wait for transcoding.
- **Caching to avoid sync calls** — API Gateway caches the Auth public key locally. Video feeds served from ElastiCache.
- **Graceful degradation** — video page renders without comments if Comment Service is down. Search shows "unavailable" without affecting playback.
- **No circular dependencies** — clear graph: Gateway → Services → Data Stores.

---

## 3. Availability Strategy

### Target Availability

| Tier | Components | Target | Downtime / year |
|------|-----------|--------|----------------|
| **Tier 1** | CDN, S3, Video Service, API Gateway | 99.9% | ~8.7 hours |
| **Tier 2** | Auth, Comments, Engagement, Upload | 99.5% | ~43 hours |
| **Tier 3** | Search, Transcoding | 99.0% | ~87 hours |

For an MVP, 99.9% is our ceiling. Four-nines (99.99%) adds significant cost and complexity that isn't justified at this stage.

### How We Achieve It

**Infrastructure:** Multi-AZ deployment for RDS, ElastiCache, and all ECS services. Auto-scaling based on CPU/memory (minimum 2 instances for Tier 1). Application Load Balancer distributes traffic and removes unhealthy instances. RabbitMQ with persistent messages and mirrored queues.

**Application:** All services are stateless — any instance can handle any request. Every service exposes `/health` and `/ready` endpoints. ECS uses them to restart stuck containers and remove unhealthy ones from the load balancer. Graceful shutdown on SIGTERM: finish in-flight requests (30 sec drain), close connections, stop accepting new work. Idempotent operations (likes, uploads) — safe to retry without side effects.

---

## 4. Fault-Tolerance Patterns

We use both **communication-level** patterns (protect against network and downstream service issues) and **application-level** patterns (protect against resource exhaustion, logic errors, stuck processes).

### Communication-Level Patterns

#### Circuit Breaker

Prevents cascading failures by stopping calls to a service that keeps failing.

```
CLOSED ──(5 failures)──▶ OPEN ──(30 sec)──▶ HALF-OPEN
  ▲                                              │
  └──────────(probe succeeds)────────────────────┘
```

**Where:** API Gateway → each downstream service; services → PostgreSQL / Redis.

**How it works:** After 5 consecutive failures (or >50% error rate in 30 sec), the circuit opens. For 30 seconds all calls to that service immediately return a fallback (cached data or 503). Then one probe request is sent — if it succeeds, circuit closes; if not, it stays open.

**Example:** Comment Service fails 5 times → circuit opens → video page loads without comments section (graceful degradation) instead of hanging on timeouts.

#### Retry with Exponential Backoff

Handles transient failures — network blips, temporary overload.

**Where:** All inter-service HTTP calls, RabbitMQ consumers, S3 operations.

**Configuration:** Max 3 retries, delays 200ms → 400ms → 800ms with ±20% random jitter (prevents thundering herd). Retry only on 5xx and connection errors — never on 4xx (client mistakes).

#### Timeouts

Every outgoing call has an explicit timeout to prevent thread starvation.

| Call | Timeout |
|------|---------|
| Gateway → any service | 5 sec |
| Service → PostgreSQL | 3 sec |
| Service → ElastiCache | 1 sec |
| Service → OpenSearch | 3 sec |

#### Rate Limiting / Throttling

Protects services from being overwhelmed — whether by a traffic spike, a misbehaving client, or a DDoS attempt.

**Where:** API Gateway (entry point for all traffic).

**How:** Rate limits are enforced per user (by JWT subject) and per IP (for unauthenticated requests). Limits stored in ElastiCache (Redis) as sliding-window counters. When a client exceeds the limit, the gateway returns `429 Too Many Requests` with a `Retry-After` header.

| Endpoint Group | Limit | Window |
|---------------|-------|--------|
| General API | 100 req | per minute |
| Auth (login/register) | 10 req | per minute |
| Video upload | 5 req | per hour |
| Search | 30 req | per minute |

Rate limiting is the first line of defense — it prevents overload *before* circuit breakers and retries even come into play.

### Application-Level Patterns

#### Bulkhead

Isolates resources so one failing component can't exhaust resources for others. Each service gets its own PostgreSQL connection pool (max 20 connections). Each downstream dependency gets a separate HTTP client pool. If Search Service is slow, it doesn't block Comment Service calls — they use separate pools.

#### Fallback / Graceful Degradation

When a non-critical component fails, the rest continues with reduced functionality:

- Search down → show "Search unavailable", playback works
- Redis down → serve from PostgreSQL directly (slower but works)
- Comment Service down → show video without comments
- Counters unavailable → show "—" instead of like/view count

#### Dead Letter Queue (DLQ)

Messages that fail processing after 3 retries go to a DLQ in RabbitMQ (retained 14 days). This prevents poison messages from blocking the pipeline. Alert triggers when DLQ depth > 0 for manual inspection.

#### Health Checks + Auto-Healing

Every service exposes two endpoints:

- `/health` (liveness) — checked every 10 sec. 3 failures → container restart.
- `/ready` (readiness) — checked every 5 sec. 2 failures → removed from load balancer.

Readiness checks verify downstream dependencies (DB, Redis). A service that can't reach its database removes itself from the load balancer to avoid serving errors.

---

## Summary

| Pattern | Level | Protects Against |
|---------|-------|-----------------|
| Circuit Breaker | Communication | Cascading failures from slow/dead services |
| Retry + Backoff | Communication | Transient network errors |
| Timeout | Communication | Thread starvation from slow calls |
| Rate Limiting | Communication | Traffic spikes, DDoS, misbehaving clients |
| Bulkhead | Application | Resource exhaustion across components |
| Fallback / Degradation | Application | Non-critical component failures |
| Dead Letter Queue | Application | Poison messages blocking pipelines |
| Health Checks | Application | Stuck or crashed containers |
