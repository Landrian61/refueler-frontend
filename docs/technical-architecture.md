# Technical Architecture Diagrams

This document provides detailed technical diagrams and architectural specifications to supplement the main Technical Feasibility Report.

## System Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                             │
├─────────────────────────────────────────────────────────────────┤
│  Web Browser    │  Mobile App   │  Desktop App  │  Smart TV     │
│  (React)        │  (React Native)│  (Electron)   │  (Android TV) │
└─────────────────┬───────────────┬───────────────┬───────────────┘
                  │               │               │
                  ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                     CDN / Edge Layer                            │
├─────────────────────────────────────────────────────────────────┤
│  CloudFlare CDN │  AWS CloudFront │  Regional Caches            │
│  - Static Assets│  - Audio Files  │  - Geo-distributed          │
│  - API Caching  │  - Streaming    │  - Edge Computing           │
└─────────────────┬───────────────┬───────────────────────────────┘
                  │               │
                  ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    API Gateway Layer                           │
├─────────────────────────────────────────────────────────────────┤
│  Load Balancer  │  Rate Limiting │  Authentication │  Routing   │
│  (HAProxy)      │  (Redis)       │  (JWT/OAuth)    │  (Kong)    │
└─────────────────┬───────────────┬───────────────┬───────────────┘
                  │               │               │
                  ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Microservices Layer                          │
├─────────────────┬───────────────┬───────────────┬───────────────┤
│  Auth Service   │ Content Mgmt  │ Streaming     │ User Mgmt     │
│  (Node.js)      │ (Node.js)     │ (Go)          │ (Node.js)     │
├─────────────────┼───────────────┼───────────────┼───────────────┤
│  Recommendation │ Analytics     │ Notification  │ Payment       │
│  (Python/ML)    │ (Node.js)     │ (Node.js)     │ (Node.js)     │
└─────────────────┬───────────────┬───────────────┬───────────────┘
                  │               │               │
                  ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Data Layer                                 │
├─────────────────┬───────────────┬───────────────┬───────────────┤
│  PostgreSQL     │  Redis Cache  │  Elasticsearch│  Object Store │
│  (User Data)    │  (Sessions)   │  (Search)     │  (Audio Files)│
│  (Metadata)     │  (Hot Data)   │  (Analytics)  │  (S3/GCS)     │
└─────────────────┴───────────────┴───────────────┴───────────────┘
```

## Audio Processing Pipeline

### Upload and Processing Flow

```
┌─────────────┐
│ File Upload │
│  (Client)   │
└──────┬──────┘
       ▼
┌─────────────┐
│   Virus     │
│  Scanning   │
│ (ClamAV)    │
└──────┬──────┘
       ▼
┌─────────────┐
│   Format    │
│ Validation  │
│ (FFprobe)   │
└──────┬──────┘
       ▼
┌─────────────┐
│  Metadata   │
│ Extraction  │
│ (FFmpeg)    │
└──────┬──────┘
       ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Transcoding │    │ Transcoding │    │ Transcoding │
│   MP3 320   │    │   AAC 256   │    │   OPUS 128  │
│  (FFmpeg)   │    │  (FFmpeg)   │    │  (FFmpeg)   │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
       ▼                  ▼                  ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Upload    │    │   Upload    │    │   Upload    │
│   to CDN    │    │   to CDN    │    │   to CDN    │
│   (S3/GCS)  │    │   (S3/GCS)  │    │   (S3/GCS)  │
└─────────────┘    └─────────────┘    └─────────────┘
```

### Streaming Protocol Flow

```
┌─────────────┐
│   Client    │
│   Request   │
└──────┬──────┘
       ▼
┌─────────────┐    No   ┌─────────────┐
│   CDN       │────────►│   Origin    │
│   Cache?    │         │   Server    │
└──────┬──────┘    Yes  └──────┬──────┘
       ▼                        ▼
┌─────────────┐         ┌─────────────┐
│   Serve     │         │   Generate  │
│   from CDN  │         │   HLS/DASH  │
└─────────────┘         └──────┬──────┘
                               ▼
                        ┌─────────────┐
                        │   Cache in  │
                        │     CDN     │
                        └──────┬──────┘
                               ▼
                        ┌─────────────┐
                        │   Serve     │
                        │   to Client │
                        └─────────────┘
```

## Database Schema Design

### Core Tables Structure

```sql
-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    avatar_url VARCHAR(500),
    subscription_tier VARCHAR(20) DEFAULT 'free',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP,
    is_active BOOLEAN DEFAULT true
);

-- Artists table
CREATE TABLE artists (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    bio TEXT,
    avatar_url VARCHAR(500),
    verified BOOLEAN DEFAULT false,
    monthly_listeners INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Albums table
CREATE TABLE albums (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    artist_id UUID REFERENCES artists(id),
    release_date DATE,
    cover_art_url VARCHAR(500),
    album_type VARCHAR(20) DEFAULT 'album', -- album, single, EP
    total_tracks INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tracks table
CREATE TABLE tracks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    artist_id UUID REFERENCES artists(id),
    album_id UUID REFERENCES albums(id),
    duration INTEGER NOT NULL, -- in seconds
    track_number INTEGER,
    file_url VARCHAR(500) NOT NULL,
    file_size BIGINT,
    bitrate INTEGER,
    format VARCHAR(10), -- mp3, aac, flac
    play_count BIGINT DEFAULT 0,
    is_explicit BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Playlists table
CREATE TABLE playlists (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    cover_art_url VARCHAR(500),
    is_public BOOLEAN DEFAULT true,
    is_collaborative BOOLEAN DEFAULT false,
    track_count INTEGER DEFAULT 0,
    total_duration INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Playlist tracks junction table
CREATE TABLE playlist_tracks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    playlist_id UUID REFERENCES playlists(id),
    track_id UUID REFERENCES tracks(id),
    position INTEGER NOT NULL,
    added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    added_by UUID REFERENCES users(id),
    UNIQUE(playlist_id, track_id)
);

-- User listening history
CREATE TABLE listening_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    track_id UUID REFERENCES tracks(id),
    played_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    duration_played INTEGER, -- seconds actually played
    completion_percentage DECIMAL(5,2),
    platform VARCHAR(50), -- web, mobile, desktop
    ip_address INET,
    user_agent TEXT
);
```

## API Specification

### Authentication Endpoints

```yaml
# Authentication API
/api/v1/auth:
  POST /register:
    summary: User registration
    request:
      email: string
      username: string
      password: string
      first_name: string
      last_name: string
    response:
      user: User
      token: JWT

  POST /login:
    summary: User login
    request:
      email: string
      password: string
    response:
      user: User
      token: JWT
      refresh_token: JWT

  POST /refresh:
    summary: Refresh JWT token
    request:
      refresh_token: JWT
    response:
      token: JWT

  POST /logout:
    summary: User logout
    headers:
      Authorization: Bearer {token}
    response:
      message: string
```

### Streaming Endpoints

```yaml
# Streaming API
/api/v1/stream:
  GET /{track_id}:
    summary: Get streaming URL for track
    parameters:
      track_id: UUID
      quality: enum [low, medium, high]
      format: enum [mp3, aac, opus]
    headers:
      Authorization: Bearer {token}
    response:
      stream_url: string
      expires_at: timestamp
      duration: integer

  POST /{track_id}/play:
    summary: Record play event
    parameters:
      track_id: UUID
    headers:
      Authorization: Bearer {token}
    request:
      timestamp: timestamp
      position: integer
    response:
      success: boolean

  GET /{track_id}/manifest.m3u8:
    summary: HLS manifest for adaptive streaming
    parameters:
      track_id: UUID
    response:
      content-type: application/vnd.apple.mpegurl
      body: HLS manifest
```

### Search Endpoints

```yaml
# Search API
/api/v1/search:
  GET /:
    summary: Search across all content
    parameters:
      q: string (required)
      type: enum [all, tracks, artists, albums, playlists]
      limit: integer (default: 20, max: 100)
      offset: integer (default: 0)
    response:
      tracks: Track[]
      artists: Artist[]
      albums: Album[]
      playlists: Playlist[]
      total: integer

  GET /suggestions:
    summary: Search suggestions
    parameters:
      q: string (required)
      limit: integer (default: 10)
    response:
      suggestions: string[]

  GET /trending:
    summary: Trending searches
    parameters:
      period: enum [hour, day, week, month]
      limit: integer (default: 10)
    response:
      trends: Trend[]
```

## Performance Benchmarks

### Target Performance Metrics

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| Time to First Byte (TTFB) | < 200ms | API response time |
| Audio Start Time | < 2s | Click to audio playback |
| Buffer Health | 30s+ | Continuous buffer ahead |
| Search Response Time | < 300ms | Query to results display |
| Page Load Time | < 3s | Full page render |
| CDN Cache Hit Ratio | > 95% | CloudFlare analytics |
| Database Query Time | < 50ms | Average query duration |
| Concurrent Users | 10K+ | Per server instance |

### Load Testing Scenarios

```javascript
// Artillery.js load test configuration
module.exports = {
  config: {
    target: 'https://api.refueler.com',
    phases: [
      { duration: 60, arrivalRate: 10 }, // Warm up
      { duration: 300, arrivalRate: 50 }, // Ramp up
      { duration: 600, arrivalRate: 100 }, // Sustained load
      { duration: 300, arrivalRate: 200 }, // Peak load
      { duration: 60, arrivalRate: 10 }  // Cool down
    ]
  },
  scenarios: [
    {
      name: 'Stream audio',
      weight: 60,
      flow: [
        { post: { url: '/api/v1/auth/login', json: { /* credentials */ } } },
        { get: { url: '/api/v1/stream/{{ trackId }}' } },
        { think: 180 } // Simulate 3 minutes of listening
      ]
    },
    {
      name: 'Search content',
      weight: 30,
      flow: [
        { get: { url: '/api/v1/search?q={{ searchTerm }}' } },
        { think: 5 }
      ]
    },
    {
      name: 'Browse playlists',
      weight: 10,
      flow: [
        { get: { url: '/api/v1/playlists?limit=20' } },
        { get: { url: '/api/v1/playlists/{{ playlistId }}' } },
        { think: 10 }
      ]
    }
  ]
};
```

## Security Implementation

### Authentication Flow

```
┌─────────────┐    1. Login Request    ┌─────────────┐
│   Client    │──────────────────────►│   Server    │
└─────────────┘                       └─────────────┘
       ▲                                      │
       │                                      │ 2. Validate
       │                                      ▼
       │                               ┌─────────────┐
       │                               │  Database   │
       │                               └─────────────┘
       │                                      │
       │ 5. JWT + Refresh Token               │ 3. User Found
       │◄─────────────────────────────────────┘
       │
       │ 6. Subsequent Requests with JWT
       │──────────────────────────►┌─────────────┐
       │                           │   Server    │
       │                           └─────────────┘
       │                                  │
       │                                  │ 7. Validate JWT
       │                                  ▼
       │                           ┌─────────────┐
       │ 8. Protected Resource     │   Redis     │
       │◄──────────────────────────│   Cache     │
                                   └─────────────┘
```

### Content Protection Strategy

```yaml
DRM Implementation:
  Widevine:
    - Chrome, Firefox, Edge support
    - Hardware-backed protection
    - License server integration
  
  FairPlay:
    - Safari and iOS support
    - Apple ecosystem integration
    - HLS streaming protection
  
  PlayReady:
    - Microsoft ecosystem
    - Enterprise scenarios
    - Windows platform support

Token-based Protection:
  Signed URLs:
    - Time-limited access
    - IP-based restrictions
    - Referer validation
  
  Streaming Tokens:
    - Per-track authorization
    - Usage tracking
    - Concurrent stream limits
```

## Monitoring and Alerting

### Key Performance Indicators (KPIs)

```yaml
Technical KPIs:
  Availability:
    - Uptime percentage: > 99.9%
    - Mean Time to Recovery (MTTR): < 15 minutes
    - Error rate: < 0.1%
  
  Performance:
    - API response time: p95 < 500ms
    - Audio start time: p95 < 3s
    - Search latency: p95 < 400ms
  
  Infrastructure:
    - CPU utilization: < 70% average
    - Memory usage: < 80% average
    - Disk I/O: < 80% capacity

Business KPIs:
  User Engagement:
    - Daily Active Users (DAU)
    - Session duration: > 20 minutes average
    - Tracks per session: > 8 average
  
  Content Consumption:
    - Hours streamed per day
    - Completion rate: > 80%
    - Skip rate: < 15%
  
  Growth Metrics:
    - New user registration rate
    - User retention: Day 1, Day 7, Day 30
    - Conversion rate: Free to Premium
```

### Alert Configuration

```yaml
Critical Alerts (PagerDuty):
  - Service downtime > 2 minutes
  - Error rate > 5% for 5 minutes
  - Database connection failures
  - CDN failures affecting > 10% users

Warning Alerts (Slack):
  - Response time > 1s for 10 minutes
  - Memory usage > 85%
  - Disk space < 20%
  - Failed audio uploads > 10/hour

Info Alerts (Email):
  - Daily performance report
  - Weekly user analytics
  - Monthly cost analysis
  - Security scan results
```

This comprehensive technical documentation provides the detailed architectural foundation for implementing a robust, scalable audio streaming platform.