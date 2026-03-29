# StreamVibe MVP — Software Architecture Academy

Architecture documentation for a video sharing platform (StreamVibe), developed as part of the Software Architecture Academy course.

**Author:** Mike Ivanov
**Date:** February – March 2026
**Cloud Provider:** AWS (eu-central-1, Frankfurt)

---

## Project Overview

A Minimum Viable Product of a video sharing platform: upload, watch, search, and interact with video content. Designed for 50K MAU with a microservice architecture on AWS.

### Tech Stack

| Layer | Technology |
|-------|-----------|
| API | REST HTTP/2, JSON |
| Auth | JWT (RS256) + OAuth 2.0 |
| Backend | Microservices on ECS Fargate |
| Database | Amazon RDS PostgreSQL 16 |
| Cache | Amazon ElastiCache (Redis 7) |
| Search | OpenSearch 2.x |
| Storage | Amazon S3 + CloudFront CDN |
| Messaging | RabbitMQ (Amazon MQ) |
| CI/CD | GitHub Actions → ECR → ECS |
| IaC | Terraform |

---

## Tasks

| # | Task | Description |
|---|------|-------------|
| 1 | [Topic & Requirements](task1/task1-topic-and-requirements.md) | Topic choice, features, user stories, UI mockups |
| 2 | [NFRs, ASRs & Diagrams](task2/task2-nfr-asr-diagrams.md) | Quality attributes, architecture-significant requirements, C4 diagrams, sequence diagrams, ERD |
| 3 | [Microservice Architecture](task3/task3-microservices.md) | Service decomposition, inter-service communication, data patterns, deployment |
| 4 | [Event-Driven Architecture](task4/task4-event-driven.md) | EDA use cases, patterns (saga, pub/sub, DLQ), RabbitMQ topology |
| 5 | [Architecture Decisions](task5/task5-architecture-decisions.md) | ADR-001 Database, ADR-002 API, Identity Management, RBAC |
| 6 | [Fault Tolerance](task6/task6-fault-tolerance.md) | Failure scenarios, statefulness, availability strategy, fault-tolerance patterns |
| 7 | [Capacity Planning & Cost](task7/task7-capacity-cost.md) | Traffic/storage/bandwidth estimation, AWS hosting cost breakdown, optimization strategies |

---

## Architecture Overview

```
                         ┌──────────────┐
                         │  CloudFront  │
                         │     CDN      │
                         └──────┬───────┘
                                │
┌─────────┐              ┌──────┴───────┐              ┌──────────┐
│  React  │──── HTTPS ──▶│  API Gateway │──── REST ───▶│ Services │
│   SPA   │              │  JWT + Rate  │              │ Auth     │
└─────────┘              │   Limiting   │              │ Video    │
                         └──────────────┘              │ Upload   │
                                                       │ Comment  │
                                                       │ Search   │
        ┌───────────┐    ┌──────────────┐              │ Engage   │
        │ Transcoder│◀───│  RabbitMQ    │◀─────────────│ Stream   │
        │  Workers  │    │  (async)     │              └──────────┘
        └─────┬─────┘    └──────────────┘                   │
              │                                             │
              ▼                                             ▼
        ┌───────────┐    ┌──────────────┐    ┌──────────────────────┐
        │    S3     │    │    Redis     │    │    PostgreSQL (RDS)  │
        │  (video)  │    │   (cache)   │    │  + Read Replica      │
        └───────────┘    └──────────────┘    └──────────────────────┘
```

## License

This project is for educational purposes as part of the Software Architecture Academy.
