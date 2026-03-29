# Task 2 — Non-Functional Requirements, ASRs, and Architecture Diagrams

**Project:** YouTube MVP
**Authors:** Mike Ivanov
**Date:** March 2026

---

## Table of Contents

1. [Quality Attribute Scenarios (NFRs)](#1-quality-attribute-scenarios-nfrs)
2. [Business Constraints](#2-business-constraints)
3. [Architecture-Significant Requirements (ASRs)](#3-architecture-significant-requirements-asrs)
4. [C4 Model — Level 1 (System Context)](#4-c4-model--level-1-system-context)
5. [C4 Model — Level 2 (Container Diagram)](#5-c4-model--level-2-container-diagram)
6. [Sequence Diagram: Video Upload & Processing](#6-sequence-diagram-video-upload--processing)
7. [Sequence Diagram: Video Playback](#7-sequence-diagram-video-playback)
8. [ERD — Core Data Model](#8-erd--core-data-model)

---

## 1. Quality Attribute Scenarios (NFRs)

### QAS-1: Performance — Video Playback Start Time

| Element | Description |
|---------|-------------|
| **Source** | User (viewer) |
| **Stimulus** | Clicks play on a video |
| **Artifact** | CDN + Streaming Service |
| **Environment** | Normal operation, 15K concurrent users |
| **Response** | First video frame begins rendering |
| **Response Measure** | ≤ 2 seconds (p95) from click to first frame |

**Rationale:** Slow playback start causes users to abandon. Industry standard for video platforms is under 2 seconds for first frame. CDN edge caching and HLS pre-loading are key enablers.

---

### QAS-2: Availability — Core Playback Path

| Element | Description |
|---------|-------------|
| **Source** | Any user |
| **Stimulus** | Requests video playback or feed at any time |
| **Artifact** | CDN, S3, Video Service, API Gateway |
| **Environment** | Normal + single AZ failure |
| **Response** | System continues serving video content |
| **Response Measure** | 99.9% uptime (~8.7 hours downtime/year) |

**Rationale:** If users can't watch videos, the platform has no value. This is the one path that must survive infrastructure failures. Achieved via Multi-AZ deployments and CDN redundancy.

---

### QAS-3: Scalability — Handling Traffic Growth

| Element | Description |
|---------|-------------|
| **Source** | Growing user base |
| **Stimulus** | MAU grows from 50K to 500K over 12 months |
| **Artifact** | All backend services, database, cache |
| **Environment** | Normal operation |
| **Response** | System handles 10× traffic without architecture change |
| **Response Measure** | p99 API latency stays ≤ 500ms; no service rewrites needed |

**Rationale:** An MVP that succeeds will grow. The architecture must scale horizontally (more container instances, read replicas, cache sharding) without redesigning the system.

---

### QAS-4: Security — Authentication and Data Protection

| Element | Description |
|---------|-------------|
| **Source** | Malicious actor |
| **Stimulus** | Attempts brute-force login, JWT tampering, or unauthorized access |
| **Artifact** | Auth Service, API Gateway |
| **Environment** | Normal operation |
| **Response** | Attack is blocked; no unauthorized access granted |
| **Response Measure** | Login attempts rate-limited (5/min/IP); JWT signed with RS256; zero unauthorized data access |

**Rationale:** User accounts and content must be protected. Compromised accounts means creators lose control of their content and users lose trust.

---

### QAS-5: Reliability — Video Processing Pipeline

| Element | Description |
|---------|-------------|
| **Source** | Creator |
| **Stimulus** | Uploads a video file |
| **Artifact** | Upload Service, RabbitMQ, Transcoder |
| **Environment** | Normal operation, including transient failures |
| **Response** | Video is transcoded and made available |
| **Response Measure** | 99.5% of uploads processed successfully within 10 minutes; zero data loss on transient failures |

**Rationale:** Failed uploads frustrate creators. The pipeline must be durable — if a transcoder crashes mid-job, the message stays in the queue and another worker picks it up.

---

### QAS-6: Modifiability — Adding New Features

| Element | Description |
|---------|-------------|
| **Source** | Development team (2–4 engineers) |
| **Stimulus** | Need to add a new feature (e.g., notifications, recommendations) |
| **Artifact** | Microservice architecture |
| **Environment** | Development and deployment |
| **Response** | New feature added as independent service without modifying existing services |
| **Response Measure** | New service deployed independently; zero downtime for existing services; ≤ 2 weeks to add a standard feature |

**Rationale:** MVP will evolve rapidly. Microservice boundaries must be clean enough that adding a notifications service doesn't require touching the video or comment services.

---

## 2. Business Constraints

| Constraint | Impact on Architecture |
|------------|----------------------|
| **Small team (2–4 engineers)** | Favor managed services (RDS, ElastiCache, S3) over self-hosted. Minimize operational overhead. One protocol (REST) everywhere to reduce cognitive load. |
| **Budget: ~$5,000–7,000/month for infrastructure** | Use cost-effective instance types (t4g family). Avoid over-provisioning. CDN costs dominate — limit video quality to 720p for MVP. |
| **Time to market: 3–4 months** | No custom solutions where managed services exist. No premature optimization. Ship working features, optimize later. |
| **Region: EU (GDPR compliance)** | Deploy in Frankfurt (eu-central-1). All user data stays in EU. S3 bucket with EU-only replication. |
| **No DevOps hire initially** | CI/CD must be simple (GitHub Actions). Infrastructure as Code (Terraform) for reproducibility. Logging and monitoring via AWS-managed CloudWatch. |

---

## 3. Architecture-Significant Requirements (ASRs)

From the functional (Task 1) and non-functional requirements above, we identify which are **architecture-significant** — meaning they directly influence the structure of the system.

| # | Requirement | Why Architecture-Significant |
|---|------------|------------------------------|
| **ASR-1** | Video playback ≤ 2s start time (QAS-1) | Requires CDN architecture with edge caching, HLS format decision, and separation of streaming path from API path. Shapes the entire content delivery layer. |
| **ASR-2** | 99.9% availability for playback (QAS-2) | Forces Multi-AZ deployment for all Tier 1 components, minimum 2 instances for critical services, automatic failover for database. Defines infrastructure topology. |
| **ASR-3** | Async video processing pipeline (QAS-5, US-04) | Requires a message broker (RabbitMQ), separate transcoder workers, durable queues. This is not a simple CRUD operation — it introduces an entire async processing layer. |
| **ASR-4** | Separation of read and write paths | System is read-heavy (49 QPS reads vs 1.2 QPS writes). Requires cache layer (Redis), read replicas, CDN — the architecture must be optimized for reads. |
| **ASR-5** | Stateless services for horizontal scaling (QAS-3) | Services must store no local state. All state goes to external stores (PostgreSQL, Redis, S3). This determines how services are designed, deployed, and scaled. |
| **ASR-6** | JWT-based auth for microservices (QAS-4) | Authentication cannot be a bottleneck — every request passes through it. Stateless JWT validation at the gateway (no round-trip to Auth Service) is an architectural pattern that affects all service-to-service communication. |
| **ASR-7** | Independent deployability (QAS-6) | Each service must have its own deployment pipeline, its own database schema, and communicate only through APIs or events. Defines service boundaries and data ownership. |

**Not architecture-significant (examples):**
- Playlist ordering (UI/implementation detail, no structural impact)
- Password minimum length (configuration, not architecture)
- Comment pagination (standard pattern, doesn't affect system structure)

---

## 4. C4 Model — Level 1 (System Context)

Shows the YouTube MVP as a black box and its external actors.

```mermaid
graph TB
    Viewer["👤 Viewer<br/>(watches videos, searches,<br/>comments, likes)"]
    Creator["👤 Creator<br/>(uploads videos,<br/>manages channel)"]
    OAuthProviders["🔐 OAuth Providers<br/>(Google, GitHub)"]

    YouTubeMVP["📺 YouTube MVP<br/><br/>Video sharing platform<br/>Upload, watch, search,<br/>comment, like"]

    CDNProvider["☁️ CDN<br/>(Amazon CloudFront)<br/>Video delivery"]

    Viewer -->|"watches, searches,<br/>likes, comments"| YouTubeMVP
    Creator -->|"uploads videos,<br/>manages channel"| YouTubeMVP
    YouTubeMVP -->|"authenticates<br/>via OAuth 2.0"| OAuthProviders
    YouTubeMVP -->|"serves video<br/>segments"| CDNProvider
    CDNProvider -->|"streams video"| Viewer
```

---

## 5. C4 Model — Level 2 (Container Diagram)

Shows the internal containers (services, databases, queues) of the YouTube MVP.

```mermaid
graph TB
    subgraph "Clients"
        WebSPA["Web SPA<br/>(React)"]
        Mobile["Mobile App<br/>(future)"]
    end

    subgraph "API Layer"
        Gateway["API Gateway<br/>JWT validation, rate limiting,<br/>routing"]
    end

    subgraph "Business Services"
        AuthSvc["Auth Service<br/>Register, Login,<br/>OAuth 2.0, JWT"]
        VideoSvc["Video Service<br/>Metadata CRUD,<br/>feed generation"]
        UploadSvc["Upload Service<br/>Pre-signed URLs,<br/>upload coordination"]
        CommentSvc["Comment Service<br/>CRUD comments"]
        EngagementSvc["Engagement Service<br/>Likes, subscriptions"]
        SearchSvc["Search Service<br/>Full-text search"]
        StreamSvc["Streaming Service<br/>HLS manifest,<br/>segment URLs"]
    end

    subgraph "Processing"
        RabbitMQ["RabbitMQ<br/>(Message Broker)"]
        Transcoder["Transcoder Workers<br/>FFmpeg, HLS output"]
    end

    subgraph "Data Stores"
        PostgreSQL[("PostgreSQL<br/>(RDS)<br/>Users, videos,<br/>comments, likes")]
        Redis[("Redis<br/>(ElastiCache)<br/>Cache, sessions,<br/>rate limits")]
        OpenSearch[("OpenSearch<br/>Video search<br/>index")]
        S3[("S3<br/>Video files,<br/>thumbnails")]
    end

    subgraph "Content Delivery"
        CloudFront["CloudFront CDN<br/>Edge caching"]
    end

    WebSPA --> Gateway
    Mobile --> Gateway

    Gateway --> AuthSvc
    Gateway --> VideoSvc
    Gateway --> UploadSvc
    Gateway --> CommentSvc
    Gateway --> EngagementSvc
    Gateway --> SearchSvc
    Gateway --> StreamSvc

    AuthSvc --> PostgreSQL
    AuthSvc --> Redis
    VideoSvc --> PostgreSQL
    VideoSvc --> Redis
    CommentSvc --> PostgreSQL
    EngagementSvc --> PostgreSQL
    EngagementSvc --> Redis
    SearchSvc --> OpenSearch

    UploadSvc --> S3
    UploadSvc --> RabbitMQ
    StreamSvc --> S3
    StreamSvc --> CloudFront

    RabbitMQ --> Transcoder
    Transcoder --> S3
    Transcoder --> RabbitMQ

    VideoSvc -->|"metadata updated"| RabbitMQ
    RabbitMQ -->|"index update"| SearchSvc

    CloudFront --> S3
```

---

## 6. Sequence Diagram: Video Upload & Processing

This is the most complex flow in the system — a multi-step async pipeline that spans several services.

```mermaid
sequenceDiagram
    participant C as Creator (Browser)
    participant GW as API Gateway
    participant US as Upload Service
    participant S3 as Amazon S3
    participant MQ as RabbitMQ
    participant TC as Transcoder Worker
    participant VS as Video Service
    participant OS as OpenSearch

    C->>GW: POST /api/v1/uploads (title, description, tags)
    GW->>GW: Validate JWT
    GW->>US: Forward request
    US->>US: Create video record (status: UPLOADING)
    US->>S3: Generate pre-signed upload URL
    US-->>C: Return { uploadUrl, videoId }

    C->>S3: PUT raw video file (direct upload)
    S3-->>C: 200 OK

    C->>GW: POST /api/v1/uploads/{videoId}/complete
    GW->>US: Forward
    US->>US: Update status: PROCESSING
    US->>MQ: Publish "video.uploaded" event
    US-->>C: 202 Accepted

    MQ->>TC: Deliver message
    TC->>S3: Download raw video
    TC->>TC: Transcode to HLS (360p, 480p, 720p)
    TC->>S3: Upload HLS segments + manifest
    TC->>MQ: Publish "video.transcoded" event

    MQ->>VS: Deliver event
    VS->>VS: Update status: READY, save HLS URLs
    VS->>MQ: Publish "video.metadata.updated"

    MQ->>OS: Deliver index event
    OS->>OS: Index video (title, description, tags)
```

---

## 7. Sequence Diagram: Video Playback

Shows how a video page load works, including CDN cache interaction.

```mermaid
sequenceDiagram
    participant U as User (Browser)
    participant GW as API Gateway
    participant VS as Video Service
    participant RD as Redis Cache
    participant PG as PostgreSQL
    participant CF as CloudFront CDN
    participant S3 as Amazon S3

    U->>GW: GET /api/v1/videos/{id}
    GW->>VS: Forward (with user context)
    VS->>RD: Check cache for video metadata

    alt Cache HIT
        RD-->>VS: Return cached metadata
    else Cache MISS
        VS->>PG: SELECT video + channel + engagement counts
        PG-->>VS: Return data
        VS->>RD: Store in cache (TTL: 5 min)
    end

    VS-->>U: Return video metadata JSON (title, HLS URL, counts)

    U->>CF: GET /video/{id}/master.m3u8 (HLS manifest)

    alt CDN Cache HIT
        CF-->>U: Return cached manifest
    else CDN Cache MISS
        CF->>S3: Fetch manifest from origin
        S3-->>CF: Return manifest
        CF-->>U: Return manifest (cached at edge)
    end

    loop For each video segment
        U->>CF: GET /video/{id}/segment_N.ts
        CF-->>U: Return video segment (from edge cache)
    end
```

---

## 8. ERD — Core Data Model

```mermaid
erDiagram
    USERS {
        uuid id PK
        string email UK
        string password_hash
        string display_name
        string avatar_url
        timestamp created_at
    }

    CHANNELS {
        uuid id PK
        uuid user_id FK
        string name
        string description
        timestamp created_at
    }

    VIDEOS {
        uuid id PK
        uuid channel_id FK
        string title
        text description
        string[] tags
        string status
        string raw_s3_key
        string hls_s3_key
        string thumbnail_url
        int duration_sec
        bigint view_count
        timestamp created_at
    }

    COMMENTS {
        uuid id PK
        uuid video_id FK
        uuid user_id FK
        text body
        boolean is_pinned
        timestamp created_at
        timestamp updated_at
    }

    LIKES {
        uuid id PK
        uuid video_id FK
        uuid user_id FK
        string type
        timestamp created_at
    }

    SUBSCRIPTIONS {
        uuid id PK
        uuid subscriber_id FK
        uuid channel_id FK
        timestamp created_at
    }

    PLAYLISTS {
        uuid id PK
        uuid user_id FK
        string title
        timestamp created_at
    }

    PLAYLIST_VIDEOS {
        uuid playlist_id FK
        uuid video_id FK
        int position
    }

    USERS ||--o{ CHANNELS : "owns"
    CHANNELS ||--o{ VIDEOS : "publishes"
    USERS ||--o{ COMMENTS : "writes"
    VIDEOS ||--o{ COMMENTS : "has"
    USERS ||--o{ LIKES : "gives"
    VIDEOS ||--o{ LIKES : "receives"
    USERS ||--o{ SUBSCRIPTIONS : "subscribes"
    CHANNELS ||--o{ SUBSCRIPTIONS : "has subscribers"
    USERS ||--o{ PLAYLISTS : "creates"
    PLAYLISTS ||--o{ PLAYLIST_VIDEOS : "contains"
    VIDEOS ||--o{ PLAYLIST_VIDEOS : "included in"
```

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| UUID primary keys | Safe for distributed generation, no sequential ID enumeration |
| Separate `CHANNELS` table | A user can have 0 or 1 channel; channel is created on first upload |
| `LIKES` with unique constraint on `(video_id, user_id)` | Enforces one like/dislike per user per video (idempotent) |
| `view_count` on `VIDEOS` table | Denormalized counter; updated via Redis buffering (batch flush every 60s) |
| `status` enum on `VIDEOS` | Tracks async processing pipeline: `UPLOADING → PROCESSING → READY → FAILED` |
| `tags` as PostgreSQL array | Simple for MVP; supports `GIN` index for tag search |
