# Visual Architecture Summary

## High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CHECK24 USERS                            │
│                   (Web, iOS App, Android App)                    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                    CDN / EDGE CACHE                              │
│         • Global distribution (CloudFront, Cloudflare)           │
│         • Static & anonymous content                             │
│         • < 50ms response time                                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│              HOME ORCHESTRATION SERVICE                          │
│                                                                   │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ Widget Selector │  │ Personalization │  │ Layout Manager  │ │
│  │  • Eligibility  │  │   • User Rules  │  │ • Responsive    │ │
│  │  • Ranking      │  │   • Context     │  │ • Platform-spec │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │           API GATEWAY / BFF                             │    │
│  │  /api/web/widgets   /api/ios/widgets   /api/android/... │    │
│  └─────────────────────────────────────────────────────────┘    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                   WIDGET CACHE (Multi-Layer)                     │
│                                                                   │
│  ┌────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │  User Cache    │  │  Generic Cache  │  │ Fallback Cache  │  │
│  │ • Redis/Mem    │  │  • Anonymous    │  │ • Stale data    │  │
│  │ • Per user ID  │  │  • Default data │  │ • Graceful fail │  │
│  │ • TTL: 5-30min │  │  • TTL: 1-24hr  │  │ • No expiry     │  │
│  └────────────────┘  └─────────────────┘  └─────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                   EVENT PROCESSING SYSTEM                        │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                 Event Stream (Kafka/SQS/PubSub)         │    │
│  │  • user.completed_quote                                 │    │
│  │  • user.booked_trip                                     │    │
│  │  • user.saved_deal                                      │    │
│  │  • product.widget_updated                               │    │
│  └───────────────────────┬─────────────────────────────────┘    │
│                          ↓                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │           Event Processors (Async Workers)              │    │
│  │  • Parse & validate events                              │    │
│  │  • Trigger cache invalidation                           │    │
│  │  • Update user context                                  │    │
│  │  • Cross-device sync                                    │    │
│  └─────────────────────────────────────────────────────────┘    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│              WIDGET REGISTRY & CONFIGURATION                     │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Widget Metadata Database (PostgreSQL/DynamoDB)         │    │
│  │  • Product registrations                                │    │
│  │  • API endpoints                                        │    │
│  │  • Display rules & targeting                            │    │
│  │  • Lifecycle state (active/paused/archived)             │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │         User Context Store (Redis/DynamoDB)             │    │
│  │  • Recent user actions                                  │    │
│  │  • Active journeys                                      │    │
│  │  • Preferences                                          │    │
│  │  • Cross-device state                                   │    │
│  └─────────────────────────────────────────────────────────┘    │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│              PRODUCT WIDGET SERVICES (60+)                       │
│                                                                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────┐│
│  │ Flight Widget    │  │ Car Insurance    │  │ Internet Tariff││
│  │   Service        │  │   Widget Service │  │  Widget Service││
│  │                  │  │                  │  │                ││
│  │ GET /widget      │  │ GET /widget      │  │ GET /widget    ││
│  │ • User context   │  │ • Personalized   │  │ • Targeted     ││
│  │ • JSON response  │  │ • Business logic │  │ • Own database ││
│  └──────────────────┘  └──────────────────┘  └────────────────┘│
│                                                                   │
│  ... + 57 more product widget services ...                       │
│                                                                   │
│  Each product:                                                    │
│  • Owns its service & infrastructure                             │
│  • Implements standard API contract                              │
│  • Controls content & personalization                            │
│  • Emits events on user actions                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Data Flow: User Visits Home (Returning User)

```
User Opens App/Website
         │
         ↓
    [Edge Cache]
         │
         ├─ Cache HIT (anonymous content)
         │     └─> Serve immediately (50ms)
         │
         └─ User-specific content needed
                │
                ↓
       [Home Orchestration]
                │
                ↓
    Check [User Cache] for widgets
                │
                ├─ Cache HIT
                │     └─> Serve cached widgets (100ms)
                │
                └─ Cache MISS/Expired
                       │
                       ↓
            Fetch from Product Services
                (with timeout & circuit breaker)
                       │
                       ├─ SUCCESS
                       │     ├─> Update cache
                       │     └─> Serve to user (300ms)
                       │
                       └─ FAILURE/TIMEOUT
                             └─> Use [Fallback Cache]
                                  └─> Serve stale (150ms)
```

---

## Data Flow: User Action Triggers Update

```
User Completes Car Insurance Quote
         │
         ↓
[Car Insurance Product]
         │
         ├─> Save to product database
         │
         └─> Emit event
               │
               ↓
        [Event Stream]
               │
               ↓
      [Event Processor]
               │
               ├─> Update [User Context]
               │     • Mark quote as completed
               │     • Update widget eligibility
               │
               ├─> Invalidate [User Cache]
               │     • Remove old car insurance widget
               │     • Trigger refresh
               │
               └─> Cross-device sync
                     • Update all active sessions
                     • WebSocket push (if connected)
                     • Otherwise, on next poll

Next Time User Visits Any Device:
         │
         ↓
    Cache refreshed
         │
         └─> New widgets shown
               (car insurance hidden,
                home insurance suggested)
```

---

## Multi-Platform Rendering

```
                [Product Widget Service]
                          │
                          ↓
                Standard JSON Response
                {
                  "widgetId": "car-insurance-123",
                  "type": "promotional",
                  "content": {
                    "title": "Complete your quote",
                    "description": "Save up to 30%",
                    "imageUrl": "...",
                    "ctaText": "Continue",
                    "ctaUrl": "/car-insurance/quote/123"
                  },
                  "metadata": {...}
                }
                          │
                          ↓
              ┌───────────┴───────────┐
              │                       │
         [Web Platform]         [Native Platform]
              │                       │
              ↓                       ↓
    React/Vue Component      Swift/Kotlin Component
              │                       │
              ↓                       ↓
        HTML + CSS              Native UI Views
    • Responsive layout      • Platform-native styling
    • Web animations        • iOS/Android guidelines
    • Browser-optimized     • App-optimized
```

---

## Failure Handling Hierarchy

```
Level 1: Product Service Responds Normally
  └─> Serve fresh data ✓

Level 2: Product Service Slow (> 500ms)
  └─> Return cached data
      └─> Async refresh in background

Level 3: Product Service Timeout/Error
  └─> Circuit breaker opens
      └─> Serve from fallback cache (stale data)
          └─> Show "Last updated: X mins ago"

Level 4: No Cache Available
  └─> Hide widget
      └─> Log error
          └─> Show other widgets normally

Level 5: Multiple Products Down
  └─> Graceful degradation
      └─> Show essential widgets only
          └─> Display system status message

Level 6: Core System Issues
  └─> Serve static fallback page
      └─> Basic navigation + error message
          └─> Home never completely down
```

---

## Widget Lifecycle States

```
┌─────────────┐
│  Created    │ Product registers new widget
└──────┬──────┘
       │
       ↓
┌─────────────┐
│   Testing   │ Internal testing, not visible to users
└──────┬──────┘
       │
       ↓
┌─────────────┐
│   Active    │ Live, visible to targeted users
└──────┬──────┘
       │
       ├─────────────> ┌─────────────┐
       │               │   Paused    │ Temporarily disabled
       │               └──────┬──────┘
       │                      │
       │<─────────────────────┘
       │
       ↓
┌─────────────┐
│ Deprecated  │ Being phased out, showing to fewer users
└──────┬──────┘
       │
       ↓
┌─────────────┐
│  Archived   │ No longer in use, kept for analytics
└─────────────┘
```

---

## Caching Strategy Summary

| Cache Type | Location | TTL | Purpose | Invalidation |
|-----------|----------|-----|---------|-------------|
| **Edge Cache** | CDN | 5-60 min | Anonymous content, assets | Time-based |
| **User Cache** | Redis/Memory | 5-30 min | Personalized widgets | Event-based |
| **Generic Cache** | Redis/Memory | 1-24 hours | Default/fallback widgets | Time-based |
| **Fallback Cache** | Database | No expiry | Stale data for failures | Manual |

---

## Technology Stack Recommendations

### Core Services
- **Language**: Node.js / Python / Go (pick one, justify)
- **API Framework**: Express / FastAPI / Gin
- **Container**: Docker
- **Orchestration**: Kubernetes / ECS / Cloud Run

### Data Storage
- **Cache**: Redis (user cache, session data)
- **Database**: PostgreSQL (widget registry) / DynamoDB (serverless option)
- **Event Stream**: Kafka / AWS SQS / Google Pub/Sub

### Frontend
- **Web**: React / Vue / Svelte + TypeScript
- **iOS**: SwiftUI / UIKit
- **Android**: Jetpack Compose / Kotlin

### Infrastructure
- **Cloud**: AWS / GCP / Azure (must justify choice)
- **CDN**: CloudFront / Cloudflare / Fastly
- **Monitoring**: Prometheus + Grafana / DataDog / New Relic
- **Logging**: ELK Stack / CloudWatch / Stackdriver

### CI/CD
- **Git**: GitHub Actions / GitLab CI / CircleCI
- **IaC**: Terraform / CloudFormation / Pulumi

---

## Performance Targets

| Metric | Target | Strategy |
|--------|--------|----------|
| **Home Load Time** | < 500ms | Edge caching + CDN |
| **Widget Render** | < 100ms per widget | Pre-computed cache |
| **Cache Hit Rate** | > 95% | Smart TTL + event invalidation |
| **API Response** | < 200ms (p95) | Optimized queries + caching |
| **Event Processing** | < 2s | Async workers + queue |
| **Cross-Device Sync** | < 3s | Real-time events + polling |
| **Uptime** | 99.9%+ | Redundancy + graceful degradation |

---

## Security Considerations

### Authentication & Authorization
- Product-to-Home: API keys + mTLS
- User-to-Home: OAuth 2.0 / JWT tokens
- Service-to-service: Service mesh / mutual TLS

### Data Protection
- GDPR compliance for user data
- Data minimization (only store necessary context)
- Encryption at rest and in transit
- Audit logging for data access

### Rate Limiting
- Per-product quotas
- User-based rate limits
- DDoS protection (WAF)
- Circuit breakers

---

## Monitoring & Alerting

### Key Metrics
- Request latency (p50, p95, p99)
- Error rate per service
- Cache hit/miss rates
- Widget load success rate
- User experience metrics (CLS, FCP, TTI)

### Alerts
- Service downtime > 1 minute
- Error rate > 1%
- Latency > 1s (p95)
- Cache hit rate < 90%
- Event processing lag > 5 minutes

### Dashboards
- Real-time system health
- Widget performance by product
- User experience metrics
- Business metrics (conversions, CTR)

---

This visual summary provides a quick reference for understanding the complete architecture. Refer to the detailed documents for implementation specifics.
