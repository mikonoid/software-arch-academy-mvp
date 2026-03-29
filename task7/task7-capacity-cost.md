# Task 7 — Capacity Planning & Cost Estimation

**Project:** StreamVibe MVP
**Authors:** Mike Ivanov
**Date:** March 2026
**Provider:** AWS (eu-central-1, Frankfurt)

---

## Table of Contents

1. [Assumptions](#1-assumptions)
2. [Traffic Estimation](#2-traffic-estimation)
3. [Storage Estimation](#3-storage-estimation)
4. [Bandwidth](#4-bandwidth)
5. [Hosting Cost](#5-hosting-cost-aws-monthly)
6. [Total Monthly Cost](#6-total-monthly-cost)
7. [How to Reduce Costs](#7-how-to-reduce-costs)

---

## 1. Assumptions

Before calculating anything we need to define our starting numbers. This is a *back-of-the-envelope estimation* — we round, simplify, and write down every assumption so that anyone can follow the math and challenge the numbers if needed.

We plan for a StreamVibe MVP with 50,000 monthly active users. About 30% of them use the platform daily. A small fraction (1%) are content creators who upload videos, while the majority are viewers.

| Parameter | Value |
|-----------|-------|
| Monthly Active Users (MAU) | 50,000 |
| Daily Active Users (DAU) | 15,000 (30% of MAU) |
| Content creators | 500 (~1% of MAU) |
| Videos uploaded per day | 250 (0.5 per creator) |
| Average video duration | 5 min |
| Average raw video file size | 200 MB |
| Transcoded output (HLS, multiple resolutions) | ~300 MB per video |
| Videos watched per user per day | 8 |
| Average watch time per video | 3 min (60% of full video) |
| Comments per user per day | 2 |
| Likes per user per day | 5 |
| Data retention | Indefinite — videos stay forever |
| Peak-to-average traffic ratio | 2× |

---

## 2. Traffic Estimation

We measure traffic in **QPS (queries per second)** — how many requests hit our servers every second. We calculate average QPS by dividing daily actions by 86,400 (seconds in a day), then multiply by 2 for peak hours.

Our platform is **read-heavy**: users mostly watch videos, browse feeds, and read comments. Writes (uploads, comments, likes) are a tiny fraction of the traffic.

### Read Traffic

| What users do | Math | QPS |
|--------------|------|-----|
| Open a video page | 15K users × 8 videos / 86,400 sec | ~1.4 |
| Stream video (HLS segment requests) | 3 min watch × 10 segments/min × 1.4 | ~45 |
| Browse homepage / feed | 15K × 3 visits / 86,400 | ~0.5 |
| Search for a video | 15K × 2 searches / 86,400 | ~0.35 |
| Read comments on a video | loaded with the video page | ~1.4 |
| **Total average reads** | | **~49 QPS** |
| **Peak reads (2×)** | | **~100 QPS** |

### Write Traffic

| What users do | Math | QPS |
|--------------|------|-----|
| Upload a video | 250 per day / 86,400 | ~0.003 |
| Post a comment | 15K × 2 / 86,400 | ~0.35 |
| Like a video | 15K × 5 / 86,400 | ~0.87 |
| Register an account | ~50 per day | ~0.001 |
| **Total average writes** | | **~1.2 QPS** |
| **Peak writes (2×)** | | **~2.5 QPS** |

**What this means:** At peak we handle about 100 read requests and 2.5 write requests per second. This is a very light load — a single PostgreSQL database with a Redis cache in front handles this easily. We don't need sharding, partitioning, or any complex scaling at this stage.

---

## 3. Storage Estimation

Storage is dominated by video files. Everything else — user data, comments, likes, search indexes — is tiny in comparison. This confirms our decision to use S3 for videos and PostgreSQL only for metadata.

### Video Storage (S3)

Every uploaded video is stored twice: the original raw file (200 MB) and the transcoded output in multiple resolutions for streaming (300 MB). That's 500 MB per video.

| Metric | Value |
|--------|-------|
| New videos per day | 250 |
| Storage per video (raw + transcoded) | 500 MB |
| **Daily new storage** | **125 GB** |
| **Monthly new storage** | **~3.75 TB** |
| **Year 1 total** | **~45 TB** |

### Database Storage (PostgreSQL)

The database stores structured data: user profiles, video titles and descriptions, comments, likes, playlists, and subscriptions. All of this is lightweight text and numbers — no large files.

| Data Type | Year 1 Size |
|-----------|------------|
| User accounts (50K users) | ~50 MB |
| Video metadata (90K videos) | ~180 MB |
| Comments (15K users × 2/day × 365 days) | ~5.5 GB |
| Likes (15K users × 5/day × 365 days) | ~2.7 GB |
| Subscriptions, playlists, etc. | ~1 GB |
| **Total with indexes** | **~12 GB** |

### Cache and Search

Redis cache holds frequently accessed data in memory for fast reads: popular video metadata, user sessions, and rate-limiting counters. The search index stores a lightweight copy of video titles and descriptions for full-text search. Both are small.

| Component | Size |
|-----------|------|
| ElastiCache Redis (working set) | ~43 MB |
| OpenSearch index (90K video documents) | ~0.5 GB |

### How Storage Grows Over Time

Video storage (S3) grows by 3.75 TB every month and never shrinks — videos stay forever. The database grows much slower.

| Month | New S3 Data | Total S3 | Total PostgreSQL |
|-------|------------|----------|-----------------|
| Month 1 | 3.75 TB | 3.75 TB | 1 GB |
| Month 3 | 3.75 TB | 11.25 TB | 3 GB |
| Month 6 | 3.75 TB | 22.5 TB | 6 GB |
| Month 12 | 3.75 TB | 45 TB | 12 GB |

**Key takeaway:** Video files are 99.9% of all storage. The database stays very small. S3 is our main cost driver for storage — and S3 is designed exactly for this: unlimited, cheap, durable object storage.

---

## 4. Bandwidth

Bandwidth is how much data flows in and out of the system per second.

**Outbound (video streaming):** When users watch videos, the CDN serves ~45 video segments per second, each about 500 KB. That's roughly 22 MB/s (~180 Mbps). This traffic is handled by CloudFront CDN at edge locations close to users — our backend servers don't deal with it at all.

**Inbound (video uploads):** Creators upload 250 videos per day, each 200 MB. That's about 0.6 MB/s (~5 Mbps). Uploads go directly to S3 via pre-signed URLs — they don't pass through our backend either.

This means our ECS services only handle lightweight API requests (metadata, comments, likes, auth) — the heavy traffic (video bytes) never touches them.

---

## 5. Hosting Cost (AWS, monthly)

We break the cost down by category. All prices are on-demand (no commitments) for the Frankfurt (eu-central-1) AWS region.

### Compute — ECS Fargate

Our microservices run as containers on AWS Fargate. Critical services (Gateway, Auth, Video) run 2 instances each for high availability. Less critical ones run a single instance. These are small containers — the traffic is light.

| Service | Instances | Size | Cost / month |
|---------|-----------|------|-------------|
| API Gateway | 2 | 0.5 vCPU, 1 GB | $30 |
| Auth Service | 2 | 0.25 vCPU, 0.5 GB | $15 |
| Video Service | 2 | 0.5 vCPU, 1 GB | $30 |
| Comment Service | 1 | 0.25 vCPU, 0.5 GB | $8 |
| Engagement Service | 1 | 0.25 vCPU, 0.5 GB | $8 |
| Upload Service | 1 | 0.25 vCPU, 0.5 GB | $8 |
| Search Service | 1 | 0.25 vCPU, 0.5 GB | $8 |
| Transcoder Worker | 1 | 1 vCPU, 2 GB | $30 |
| **Subtotal Compute** | | | **~$137** |

### Database — Amazon RDS PostgreSQL

The database is the backbone of the system — it stores all user data, video metadata, comments, likes, and playlists. We run three instances:

- **Primary** — handles all writes and most reads
- **Multi-AZ standby** — an automatic backup copy in another data center. If the primary fails, AWS switches to the standby in 30–60 seconds with zero data loss. This is required for our availability target (99.9%).
- **Read replica** — offloads read-heavy queries (e.g., video metadata for the feed) from the primary, keeping it responsive for writes.

| Component | Spec | Cost / month |
|-----------|------|-------------|
| Primary instance | db.t4g.medium (2 vCPU, 4 GB) | $70 |
| Multi-AZ standby | same (automatic failover) | $70 |
| Read replica | db.t4g.medium | $70 |
| Storage (50 GB gp3) | | $6 |
| Automated backups (7 days) | included | $0 |
| **Subtotal RDS** | | **~$216** |

At our scale (12 GB data, 2.5 peak writes/sec), `db.t4g.medium` is more than enough. When we outgrow it, upgrading to a larger instance or Aurora PostgreSQL requires no code changes.

### Cache — Amazon ElastiCache (Redis)

Redis keeps frequently accessed data in memory so we don't hit the database on every request. At 43 MB working set, the smallest instance is more than sufficient. We run a replica for automatic failover.

| Component | Spec | Cost / month |
|-----------|------|-------------|
| Primary node | cache.t4g.micro (0.5 GB) | $13 |
| Replica (Multi-AZ) | cache.t4g.micro | $13 |
| **Subtotal ElastiCache** | | **~$26** |

### Search — Amazon OpenSearch

OpenSearch powers video search (by title, description, tags). With a 0.5 GB index, a single small node is plenty.

| Component | Spec | Cost / month |
|-----------|------|-------------|
| Single node | t3.small.search (2 vCPU, 2 GB) | $36 |
| Storage | 10 GB gp3 | $1 |
| **Subtotal OpenSearch** | | **~$37** |

### Video Storage — Amazon S3

S3 stores all video files (raw uploads and transcoded HLS segments). This is the only cost that grows significantly over time — every month adds 3.75 TB of new video data. Everything else stays roughly flat.

| Component | Volume | Cost / month |
|-----------|--------|-------------|
| S3 Standard storage | 3.75 TB (month 1) → 45 TB (month 12) | $86 → $1,035 |
| Upload requests (PUT) | ~7.5K/mo | <$1 |
| Streaming requests (GET) | ~115M/mo | $46 |
| **Subtotal S3 (month 1 → month 12)** | | **~$132 → ~$1,081** |

### CDN — Amazon CloudFront

CloudFront is a content delivery network — it caches video segments at edge locations around the world so users get fast playback without every request going back to S3. For a video platform, CDN is always the biggest cost because it's charged by the amount of data delivered to viewers.

With 15K daily users watching 8 videos each, we deliver about 60 TB of video data per month through the CDN.

| Component | Cost / month |
|-----------|-------------|
| Data transfer out (~60 TB, Europe pricing tiers) | $4,650 |
| HTTPS requests (~115M) | $138 |
| **Subtotal CloudFront** | **~$4,788** |

> **Note:** CDN is 88% of the total bill — this is **normal for every video platform**. Netflix, TikTok and similar platforms all spend most of their infrastructure budget on content delivery. We have several ways to optimize this: limit video quality to 720p for MVP, set longer cache TTL (7 days), and use CloudFront Savings Plan for up to 30% discount.

### Messaging & Networking

RabbitMQ handles our asynchronous video processing pipeline (upload → transcode → index). Networking includes the load balancer, NAT gateway for private subnets, DNS, and monitoring.

| Component | Spec | Cost / month |
|-----------|------|-------------|
| RabbitMQ (Amazon MQ) | mq.t3.micro + 20 GB | $32 |
| Application Load Balancer | 1 ALB | $22 |
| NAT Gateway | 1 | $45 |
| Route 53 (DNS) | 1 hosted zone | $1 |
| CloudWatch (monitoring) | basic | $10 |
| ECR (container images) | ~5 GB | $1 |
| **Subtotal Messaging + Networking** | | **~$111** |

---

## 6. Total Monthly Cost

| Category | Month 1 | Month 12 |
|----------|---------|----------|
| Compute (ECS Fargate) | $137 | $137 |
| Database (RDS PostgreSQL) | $216 | $216 |
| Cache (ElastiCache Redis) | $26 | $26 |
| Search (OpenSearch) | $37 | $37 |
| Video Storage (S3) | $132 | $1,081 |
| CDN (CloudFront) | $4,788 | $4,788 |
| Messaging + Networking | $111 | $111 |
| **TOTAL** | **~$5,447** | **~$6,396** |

### Where does the money go?

The cost structure of a video platform is fundamentally different from a typical web application:

1. **CDN — 88% of the total bill.** This is the cost of delivering video bytes to viewers. Every video platform has this as its dominant cost. It's not a sign of waste — it's the nature of the business.
2. **S3 storage — grows from 2% to 17%.** This is the only cost that increases month over month as more videos are uploaded. After 12 months, storage becomes the second biggest expense.
3. **Database — 4%.** Fixed cost. PostgreSQL handles our scale easily with room to grow 10×.
4. **Everything else — 6%.** Compute, cache, search, messaging, networking — all minor at MVP scale.

### Backend cost (without CDN)

If we look at the cost we actually control (everything except CDN), the picture is very affordable:

| | Month 1 | Month 12 |
|---|---------|----------|
| **Backend only (without CloudFront)** | **~$659** | **~$1,608** |

$659/month for a fully operational video platform backend serving 50K users — that's a realistic and manageable budget for an MVP.

---

## 7. How to Reduce Costs

There are several strategies to reduce costs as the platform grows, without changing the architecture:

**Video storage optimization (data tiering):** Not all videos are watched equally. Popular new videos need fast access (S3 Standard), but older or rarely watched videos can be moved to cheaper storage automatically. S3 Intelligent-Tiering does this with no code changes — it monitors access patterns and moves objects between tiers, saving up to 40% on storage.

**Raw file cleanup:** After transcoding, we don't need the raw upload anymore — viewers watch the transcoded HLS version. We set a lifecycle rule to move raw files to S3 Infrequent Access after 30 days (45% cheaper per GB) or delete them entirely.

**CDN caching:** Setting a long TTL (7 days) on transcoded video segments means popular videos are served from CloudFront edge locations without fetching from S3 again. This reduces both S3 GET costs and CDN origin fetches.

**Reserved pricing:** For components we know we'll use for at least a year (database, cache, compute), AWS offers 30–40% discounts with 1-year commitments:

| Component | On-demand | Reserved (1 year) |
|-----------|----------|-------------------|
| RDS PostgreSQL (3 instances) | $210/mo | ~$140/mo |
| ElastiCache Redis | $26/mo | ~$18/mo |
| Fargate compute | $137/mo | ~$96/mo |
| **Total savings** | **$373/mo** | **~$254/mo (32% off)** |
