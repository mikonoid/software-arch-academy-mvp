# Task 1 — Topic, Features, and Functional Requirements

**Project:** StreamVibe MVP
**Authors:** Mike Ivanov
**Date:** March 2026

---

## 1. Topic: StreamVibe MVP (Video Sharing Platform)

We build a **Minimum Viable Product** of a video sharing platform — StreamVibe. The goal is to allow users to upload, watch, search, and interact with video content.

**Why this topic:**
- Video platforms are architecturally rich — they combine CRUD, streaming, async processing, search, and CDN delivery.
- Clear separation between read-heavy (watching) and write-heavy (uploading) workloads.
- Good fit for microservice architecture with event-driven processing pipeline.

---

## 2. Core Features

| # | Feature | Description |
|---|---------|-------------|
| F1 | **Video Upload** | Creators upload videos via the browser. Videos are stored and transcoded into streaming format (HLS). |
| F2 | **Video Playback** | Users watch videos with adaptive bitrate streaming. Playback is served via CDN. |
| F3 | **Video Search** | Full-text search by title, description, and tags with relevance ranking. |
| F4 | **Home Feed** | Personalized feed showing recent and popular videos. |
| F5 | **Comments** | Users can add, edit, and delete comments on videos. |
| F6 | **Likes / Dislikes** | Users can like or dislike videos. Counts are displayed on the video page. |
| F7 | **Subscriptions** | Users subscribe to channels and see new videos in their subscription feed. |
| F8 | **User Registration & Auth** | Email/password registration + social login (Google, GitHub). JWT-based authentication. |
| F9 | **Channel Page** | Each creator has a channel page listing their videos. |
| F10 | **Playlists** | Users can create and manage ordered collections of videos. |

### Out of Scope (MVP)

- Live streaming
- Monetization / ads
- Admin panel / moderation
- Recommendations engine (ML-based)
- Video editing tools
- Notifications (push/email)
- Multi-language support

---

## 3. Key Scenarios

### Scenario 1: Video Upload and Processing

1. Creator authenticates and selects a video file.
2. Browser uploads the file directly to S3 via a pre-signed URL.
3. Upload Service creates a video record in the database with status `PROCESSING`.
4. A message is published to the message broker (RabbitMQ).
5. Transcoder worker picks up the message, downloads the raw file from S3, transcodes it to HLS (multiple resolutions), and uploads segments back to S3.
6. Transcoder sends a completion event; Video Service updates status to `READY` and triggers search index update.

### Scenario 2: Video Playback

1. User opens a video page.
2. API Gateway routes the request to Video Service, which returns video metadata (title, description, channel info, engagement counts).
3. The video player requests HLS manifest and segments from CDN (CloudFront).
4. CDN serves cached segments from edge locations; on cache miss, fetches from S3 origin.

### Scenario 3: Search

1. User types a query in the search bar.
2. API Gateway routes to Search Service, which queries OpenSearch.
3. Results are returned ranked by relevance (title match > description match > tags).
4. Each result includes video thumbnail URL (served via CDN), title, channel name, view count.

---

## 4. Functional Requirements (User Stories)

### Authentication

| ID | User Story | Acceptance Criteria |
|----|-----------|-------------------|
| US-01 | As a visitor, I want to register with email and password so that I can interact with videos. | Account created, JWT issued, email uniqueness enforced. |
| US-02 | As a visitor, I want to sign in with Google so that I don't need a separate password. | OAuth 2.0 flow completes, JWT issued, account linked to Google profile. |
| US-03 | As a user, I want to log out so that my session ends on this device. | Access token cleared, refresh token revoked in Redis. |

### Video Upload

| ID | User Story | Acceptance Criteria |
|----|-----------|-------------------|
| US-04 | As a creator, I want to upload a video file so that it becomes available on the platform. | File uploaded to S3, transcoded to HLS, video status becomes `READY`. |
| US-05 | As a creator, I want to set a title, description, and tags for my video. | Metadata saved in PostgreSQL, searchable in OpenSearch within 30 seconds. |
| US-06 | As a creator, I want to see the processing status of my upload. | Status displayed: `UPLOADING → PROCESSING → READY` or `FAILED`. |

### Video Playback

| ID | User Story | Acceptance Criteria |
|----|-----------|-------------------|
| US-07 | As a user, I want to watch a video with adaptive quality so that playback is smooth on any connection. | HLS player switches between 360p / 480p / 720p based on bandwidth. |
| US-08 | As a visitor (no account), I want to watch videos without logging in. | Video page loads and plays without authentication. |

### Engagement

| ID | User Story | Acceptance Criteria |
|----|-----------|-------------------|
| US-09 | As a user, I want to like or dislike a video so that I can express my opinion. | Like/dislike is idempotent (one per user per video), count updates on the page. |
| US-10 | As a user, I want to comment on a video so that I can discuss content. | Comment appears in the comment section, ordered by newest first. |
| US-11 | As a user, I want to subscribe to a channel so that I see their new videos. | Subscription saved, channel videos appear in the subscription feed. |

### Search & Discovery

| ID | User Story | Acceptance Criteria |
|----|-----------|-------------------|
| US-12 | As a user, I want to search for videos by keyword so that I can find content I'm interested in. | Results returned within 500ms, ranked by relevance. |
| US-13 | As a user, I want to see a home feed with recent and popular videos. | Feed loads within 1 second, shows a mix of recent uploads and trending videos. |

### Channel & Playlists

| ID | User Story | Acceptance Criteria |
|----|-----------|-------------------|
| US-14 | As a visitor, I want to view a channel page so that I can see all videos by a creator. | Channel page shows creator info and video list, paginated. |
| US-15 | As a user, I want to create a playlist and add videos to it. | Playlist created, videos can be added/removed/reordered. |

---

## 5. UI Mockups

### 5.1 Home Page

```
┌─────────────────────────────────────────────────────────┐
│  [Logo]  StreamVibe         [Search Bar]    [Sign In]   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Trending Videos                                        │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │
│  │ ▶ thumb │  │ ▶ thumb │  │ ▶ thumb │  │ ▶ thumb │   │
│  │ Title   │  │ Title   │  │ Title   │  │ Title   │   │
│  │ Channel │  │ Channel │  │ Channel │  │ Channel │   │
│  │ 12K views│ │ 8K views│  │ 5K views│  │ 3K views│   │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘   │
│                                                         │
│  Recent Uploads                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │
│  │ ▶ thumb │  │ ▶ thumb │  │ ▶ thumb │  │ ▶ thumb │   │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 5.2 Video Page

```
┌─────────────────────────────────────────────────────────┐
│  [Logo]  StreamVibe         [Search Bar]    [Profile]   │
├──────────────────────────────────┬──────────────────────┤
│                                  │  Up Next             │
│  ┌──────────────────────────┐    │  ┌────────┐          │
│  │                          │    │  │ ▶ thumb│ Title    │
│  │      VIDEO PLAYER        │    │  └────────┘          │
│  │      (HLS adaptive)      │    │  ┌────────┐          │
│  │                          │    │  │ ▶ thumb│ Title    │
│  └──────────────────────────┘    │  └────────┘          │
│                                  │  ┌────────┐          │
│  Video Title                     │  │ ▶ thumb│ Title    │
│  Channel Name  [Subscribe]       │  └────────┘          │
│  1,234 views · 2 hours ago       │                      │
│  👍 156  👎 12                   │                      │
│                                  │                      │
│  Description text...             │                      │
│  ──────────────────────          │                      │
│  Comments (24)                   │                      │
│  ┌──────────────────────────┐    │                      │
│  │ [User avatar] Username    │    │                      │
│  │ Comment text here...      │    │                      │
│  └──────────────────────────┘    │                      │
│  ┌──────────────────────────┐    │                      │
│  │ [User avatar] Username    │    │                      │
│  │ Another comment...        │    │                      │
│  └──────────────────────────┘    │                      │
└──────────────────────────────────┴──────────────────────┘
```

### 5.3 Upload Page

```
┌─────────────────────────────────────────────────────────┐
│  [Logo]  StreamVibe         [Search Bar]    [Profile]   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Upload Video                                           │
│  ┌─────────────────────────────────────────────────┐    │
│  │                                                 │    │
│  │        Drag & drop video file here              │    │
│  │              or [Select File]                    │    │
│  │                                                 │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  Title:       [________________________]                │
│  Description: [________________________]                │
│               [________________________]                │
│  Tags:        [________________________]                │
│                                                         │
│  [Upload & Publish]                                     │
│                                                         │
│  Processing status: ████████░░ 80% Transcoding...       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 5.4 Search Results

```
┌─────────────────────────────────────────────────────────┐
│  [Logo]  StreamVibe  [Search: "cooking"]    [Profile]   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Results for "cooking" (42 videos)                      │
│                                                         │
│  ┌──────────┐  Video Title One                          │
│  │ ▶ thumb  │  Channel Name · 12K views · 3 days ago   │
│  │          │  Short description of the video...        │
│  └──────────┘                                           │
│                                                         │
│  ┌──────────┐  Video Title Two                          │
│  │ ▶ thumb  │  Channel Name · 8K views · 1 week ago    │
│  │          │  Short description of the video...        │
│  └──────────┘                                           │
│                                                         │
│  ┌──────────┐  Video Title Three                        │
│  │ ▶ thumb  │  Channel Name · 2K views · 2 weeks ago   │
│  │          │  Short description of the video...        │
│  └──────────┘                                           │
│                                                         │
│  [Load More]                                            │
└─────────────────────────────────────────────────────────┘
```
