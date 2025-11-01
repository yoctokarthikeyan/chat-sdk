# Chat SDK - Architecture Diagrams

## 1. High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT APPLICATIONS                          │
│                                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │   Web    │  │  React   │  │  Mobile  │  │ Flutter  │           │
│  │   App    │  │  Native  │  │iOS/Andr. │  │   App    │           │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
└───────┼─────────────┼─────────────┼─────────────┼──────────────────┘
        │             │             │             │
        └─────────────┴─────────────┴─────────────┘
                      │
                      │ HTTPS/WSS
                      │
        ┌─────────────▼─────────────────────────────┐
        │         API GATEWAY / LOAD BALANCER        │
        │                                            │
        │  • Rate Limiting                           │
        │  • SSL Termination                         │
        │  • Request Routing                         │
        │  • Authentication                          │
        └─────────────┬─────────────────────────────┘
                      │
        ┌─────────────┴─────────────────────────────┐
        │                                            │
┌───────▼────────┐                    ┌─────────────▼──────────┐
│  REST API      │                    │  WebSocket Server      │
│  Server        │                    │  (Real-time)           │
│                │                    │                        │
│ • User Mgmt    │◄───────┐          │ • Message Broadcasting │
│ • Channels     │        │          │ • Presence Tracking    │
│ • Messages     │        │          │ • Typing Indicators    │
│ • Media Upload │        │          │ • Connection Pool      │
└───────┬────────┘        │          └─────────────┬──────────┘
        │                 │                        │
        │                 │                        │
        └─────────────────┼────────────────────────┘
                          │
        ┌─────────────────▼─────────────────────────┐
        │         BUSINESS LOGIC LAYER               │
        │                                            │
        │  • Message Processing                      │
        │  • Channel Management                      │
        │  • Permissions & ACL                       │
        │  • Event Publishing                        │
        └─────────────┬─────────────────────────────┘
                      │
        ┌─────────────┴─────────────────────────────┐
        │                                            │
┌───────▼────────┐  ┌──────────┐  ┌────────────────▼────────┐
│   PostgreSQL   │  │  Redis   │  │   Object Storage (S3)   │
│                │  │          │  │                         │
│ • Users        │  │ • Cache  │  │ • Images                │
│ • Channels     │  │ • Pub/Sub│  │ • Videos                │
│ • Messages     │  │ • Queue  │  │ • Files                 │
│ • Metadata     │  │ • Session│  │ • Attachments           │
└────────────────┘  └──────────┘  └─────────────────────────┘
```

---

## 2. Message Flow Diagram

```
┌─────────┐                                              ┌─────────┐
│ User A  │                                              │ User B  │
└────┬────┘                                              └────┬────┘
     │                                                        │
     │ 1. Send Message                                       │
     ├──────────────────────────────────────────────┐        │
     │                                              │        │
     │                                              ▼        │
     │                                    ┌──────────────────┴────┐
     │                                    │   WebSocket Server    │
     │                                    │                       │
     │                                    │ 2. Validate & Process │
     │                                    └──────────┬────────────┘
     │                                               │
     │                                               │ 3. Save to DB
     │                                               ▼
     │                                    ┌─────────────────────┐
     │                                    │    PostgreSQL       │
     │                                    │                     │
     │                                    │ • Store message     │
     │                                    │ • Update channel    │
     │                                    └──────────┬──────────┘
     │                                               │
     │                                               │ 4. Publish Event
     │                                               ▼
     │                                    ┌─────────────────────┐
     │                                    │    Redis Pub/Sub    │
     │                                    │                     │
     │                                    │ • Broadcast event   │
     │                                    └──────────┬──────────┘
     │                                               │
     │ 5. Delivery Receipt                          │ 6. Deliver Message
     │◄─────────────────────────────────────────────┼──────────────────►
     │                                               │
     │                                               │ 7. Push Notification
     │                                               │    (if offline)
     │                                               ▼
     │                                    ┌─────────────────────┐
     │                                    │  Push Service       │
     │                                    │  (FCM/APNS)         │
     │                                    └─────────────────────┘
```

---

## 3. Authentication Flow

```
┌─────────┐                                              ┌─────────────┐
│ Client  │                                              │   Backend   │
└────┬────┘                                              └──────┬──────┘
     │                                                          │
     │ 1. Request Token (username/password)                    │
     ├─────────────────────────────────────────────────────────►
     │                                                          │
     │                                        2. Validate       │
     │                                           Credentials    │
     │                                                          │
     │                                        3. Generate JWT   │
     │                                                          │
     │ 4. Return Token                                          │
     │◄─────────────────────────────────────────────────────────┤
     │                                                          │
     │ 5. Connect to WebSocket (with token)                    │
     ├─────────────────────────────────────────────────────────►
     │                                                          │
     │                                        6. Validate Token │
     │                                                          │
     │ 7. Connection Established                                │
     │◄─────────────────────────────────────────────────────────┤
     │                                                          │
     │ 8. Subscribe to Channels                                 │
     ├─────────────────────────────────────────────────────────►
     │                                                          │
     │ 9. Receive Real-time Events                              │
     │◄─────────────────────────────────────────────────────────┤
     │                                                          │
```

---

## 4. SDK Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        APPLICATION LAYER                         │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   React UI   │  │   Vue UI     │  │  Angular UI  │         │
│  │  Components  │  │  Components  │  │  Components  │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
└─────────┼──────────────────┼──────────────────┼────────────────┘
          │                  │                  │
          └──────────────────┴──────────────────┘
                             │
┌────────────────────────────▼─────────────────────────────────────┐
│                    FRAMEWORK ADAPTERS                             │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ React Hooks  │  │ Vue Compos.  │  │ Angular Srv. │          │
│  │ • useChat    │  │ • useChat    │  │ • ChatService│          │
│  │ • useChannel │  │ • useChannel │  │ • ChannelSrv │          │
│  │ • useMessages│  │ • useMessages│  │ • MessageSrv │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
└─────────┼──────────────────┼──────────────────┼──────────────────┘
          │                  │                  │
          └──────────────────┴──────────────────┘
                             │
┌────────────────────────────▼─────────────────────────────────────┐
│                        CORE SDK                                   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    ChatClient                            │    │
│  │  • connect()  • disconnect()  • getUser()               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Auth       │  │   Channels   │  │   Messages   │          │
│  │  Module      │  │   Module     │  │   Module     │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                  │                  │                  │
│  ┌──────▼──────────────────▼──────────────────▼───────┐         │
│  │              Connection Manager                     │         │
│  │  • WebSocket  • Reconnection  • Heartbeat          │         │
│  └──────┬──────────────────────────────────────────────┘         │
│         │                                                         │
│  ┌──────▼──────────────────────────────────────────────┐         │
│  │              State Manager                           │         │
│  │  • Local Cache  • Event Emitter  • Sync Queue      │         │
│  └─────────────────────────────────────────────────────┘         │
└───────────────────────────────────────────────────────────────────┘
                             │
                             │
┌────────────────────────────▼─────────────────────────────────────┐
│                    NETWORK LAYER                                  │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   HTTP       │  │  WebSocket   │  │   Storage    │          │
│  │   Client     │  │   Client     │  │   Client     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└───────────────────────────────────────────────────────────────────┘
```

---

## 5. Database Schema Relationships

### **Schema v2.0 - Separated User Types**

This schema separates **customer_users** (portal users) from **app_users** (SDK end-users).

```
PORTAL LAYER (Customer Management)
═══════════════════════════════════

┌──────────────────┐
│ customer_users   │ ◄─── Your customers who manage the platform
│  ──────────────  │
│  • id (PK)       │
│  • email         │
│  • password_hash │
│  • display_name  │
│  • is_active     │
└────────┬─────────┘
         │
         │ 1:N
         │
┌────────▼─────────┐
│ refresh_tokens   │
│  ──────────────  │
│  • id (PK)       │
│  • customer_user_id (FK)
│  • token         │
│  • expires_at    │
└──────────────────┘

┌──────────────────┐         ┌──────────────────────┐
│ customer_users   │         │       teams          │
│  ──────────────  │         │  ──────────────────  │
│  • id (PK)       │         │  • id (PK)           │
└────────┬─────────┘         │  • app_id (FK)       │
         │                   │  • name              │
         │ N:M               └──────────┬───────────┘
         │                              │
         │  ┌───────────────────────┐   │
         └──► customer_user_teams   │◄──┘
            │  ───────────────────  │
            │  • id (PK)            │
            │  • customer_user_id(FK)
            │  • team_id (FK)       │
            │  • role               │
            └───────────────────────┘


APPLICATION LAYER
═════════════════

┌─────────────────┐
│  applications   │
│  ─────────────  │
│  • id (PK)      │
│  • name         │
│  • api_key      │
│  • api_secret   │
│  • plan_id (FK) │
│  • is_active    │
└────────┬────────┘
         │
         ├──────────────────────┬──────────────────┬──────────────────┐
         │ 1:N                  │ 1:N              │ 1:N              │ 1:N
         │                      │                  │                  │
┌────────▼────────┐   ┌─────────▼────────┐  ┌─────▼──────┐  ┌───────▼────────┐
│   app_users     │   │    channels      │  │  webhooks  │  │   campaigns    │
│  ─────────────  │   │  ──────────────  │  │  ────────  │  │  ────────────  │
│  • id (PK)      │   │  • id (PK)       │  │  • id (PK) │  │  • id (PK)     │
│  • app_id (FK)  │   │  • app_id (FK)   │  │  • app_id  │  │  • app_id (FK) │
│  • external_id  │   │  • type          │  │  • name    │  │  • name        │
│  • username     │   │  • name          │  │  • url     │  │  • message     │
│  • display_name │   │  • created_by(FK)│  │  • type    │  │  • status      │
│  • user_type    │   │  • is_frozen     │  └────────────┘  └────────────────┘
│  • is_online    │   └─────────┬────────┘
└────────┬────────┘             │
         │                      │
         │ 1:N                  │
         │                      │
┌────────▼────────┐             │
│app_user_devices │             │
│  ─────────────  │             │
│  • id (PK)      │             │
│  • app_user_id  │             │
│  • device_id    │             │
│  • push_token   │             │
│  • platform     │             │
└─────────────────┘             │


SDK LAYER (Chat Features)
══════════════════════════

┌─────────────────┐         ┌─────────────────┐
│   app_users     │         │    channels     │
│  ─────────────  │         │  ─────────────  │
│  • id (PK)      │         │  • id (PK)      │
│  • app_id (FK)  │         │  • app_id (FK)  │
│  • external_id  │         │  • type         │
│  • is_online    │         │  • name         │
└────────┬────────┘         └────────┬────────┘
         │                           │
         │ N:M                       │
         │                           │
         │    ┌──────────────────┐   │
         └────► channel_members  │◄──┘
              │  ──────────────  │
              │  • id (PK)       │
              │  • channel_id(FK)│
              │  • app_user_id(FK) ◄─── Changed from user_id
              │  • role          │
              │  • unread_count  │
              │  • is_muted      │
              │  • is_banned     │
              └────────┬─────────┘
                       │
                       │ 1:N
                       │
              ┌────────▼─────────┐
              │    messages      │
              │  ──────────────  │
              │  • id (PK)       │
              │  • channel_id(FK)│
              │  • app_user_id(FK) ◄─── Changed from user_id
              │  • text          │
              │  • type          │
              │  • is_pinned     │
              │  • is_deleted    │
              └────────┬─────────┘
                       │
                       │ 1:N
                       │
         ┌─────────────┼─────────────┬─────────────────┐
         │             │             │                 │
┌────────▼────────┐ ┌──▼──────────┐ ┌▼──────────────┐ ┌▼──────────────┐
│message_         │ │message_     │ │message_reads  │ │message_drafts │
│attachments      │ │reactions    │ │  ───────────  │ │  ───────────  │
│  ─────────────  │ │  ─────────  │ │  • id (PK)    │ │  • id (PK)    │
│  • id (PK)      │ │  • id (PK)  │ │  • message_id │ │  • channel_id │
│  • message_id   │ │  • msg_id   │ │  • app_user_id│ │  • app_user_id│
│  • type         │ │  • app_user │ │  • read_at    │ │  • text       │
│  • url          │ │  • emoji    │ └───────────────┘ └───────────────┘
│  • file_size    │ └─────────────┘
└─────────────────┘


MODERATION LAYER
════════════════

┌──────────────────┐
│ moderation_queue │
│  ──────────────  │
│  • id (PK)       │
│  • app_id (FK)   │
│  • message_id(FK)│
│  • app_user_id(FK) ◄─── SDK user being moderated
│  • reviewed_by(FK) ◄─── customer_user who reviews
│  • status        │
└──────────────────┘
```

### **Key Relationships:**

**Portal Layer:**
- `customer_users` (1) → (N) `refresh_tokens`
- `customer_users` (N) → (M) `teams` via `customer_user_teams`

**Application Layer:**
- `teams` (1) → (1) `applications`
- `applications` (1) → (N) `app_users`
- `applications` (1) → (N) `channels`
- `applications` (1) → (N) `webhooks`
- `applications` (1) → (N) `campaigns`

**SDK Layer (Chat):**
- `app_users` (1) → (N) `app_user_devices`
- `app_users` (1) → (N) `messages`
- `app_users` (1) → (N) `channel_members`
- `app_users` (1) → (N) `message_reactions`
- `app_users` (1) → (N) `message_reads`
- `app_users` (1) → (N) `message_drafts`
- `channels` (1) → (N) `messages`
- `channels` (1) → (N) `channel_members`
- `messages` (1) → (N) `message_attachments`
- `messages` (1) → (N) `message_reactions`
- `messages` (1) → (N) `message_reads`

**Important Notes:**
- ✅ **customer_users** = Portal users (your customers)
- ✅ **app_users** = SDK end-users (your customers' users)
- ✅ All chat features reference **app_users**, not customer_users
- ✅ Moderation reviewed by **customer_users**
- ✅ **external_id** in app_users maps to customer's user system

---

## 6. Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         AWS CLOUD                                │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │                    Route 53 (DNS)                       │    │
│  └───────────────────────────┬────────────────────────────┘    │
│                              │                                  │
│  ┌───────────────────────────▼────────────────────────────┐    │
│  │              CloudFront (CDN)                           │    │
│  └───────────────────────────┬────────────────────────────┘    │
│                              │                                  │
│  ┌───────────────────────────▼────────────────────────────┐    │
│  │          Application Load Balancer (ALB)                │    │
│  └───────────────────────────┬────────────────────────────┘    │
│                              │                                  │
│  ┌───────────────────────────▼────────────────────────────┐    │
│  │                  EKS Cluster                            │    │
│  │                                                         │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │    │
│  │  │  API Server  │  │  WebSocket   │  │   Worker    │ │    │
│  │  │   Pods (3+)  │  │  Server      │  │   Pods      │ │    │
│  │  │              │  │  Pods (3+)   │  │   (Jobs)    │ │    │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬──────┘ │    │
│  └─────────┼──────────────────┼──────────────────┼────────┘    │
│            │                  │                  │              │
│  ┌─────────▼──────────────────▼──────────────────▼────────┐    │
│  │                    Service Mesh                         │    │
│  └─────────┬──────────────────┬──────────────────┬────────┘    │
│            │                  │                  │              │
│  ┌─────────▼────────┐  ┌──────▼──────┐  ┌───────▼────────┐    │
│  │  RDS PostgreSQL  │  │ ElastiCache │  │      S3        │    │
│  │   (Multi-AZ)     │  │   (Redis)   │  │  (Storage)     │    │
│  │                  │  │             │  │                │    │
│  │  • Primary       │  │  • Cluster  │  │  • Media files │    │
│  │  • Standby       │  │  • Replicas │  │  • Backups     │    │
│  └──────────────────┘  └─────────────┘  └────────────────┘    │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Supporting Services                          │  │
│  │                                                           │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │  │
│  │  │CloudWatch│  │  SNS/SQS │  │Elasticsearch│ │ Secrets │ │  │
│  │  │(Monitor) │  │(Messaging)│  │  (Search) │  │ Manager │ │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └─────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 7. Scaling Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                    HORIZONTAL SCALING                            │
│                                                                  │
│  Load Increases ──────────────────────────────────────►         │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Server 1 │  │ Server 2 │  │ Server 3 │  │ Server N │       │
│  │          │  │          │  │          │  │          │       │
│  │ 1K users │  │ 1K users │  │ 1K users │  │ 1K users │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │             │             │             │              │
│       └─────────────┴─────────────┴─────────────┘              │
│                     │                                           │
│            ┌────────▼────────┐                                  │
│            │  Redis Pub/Sub  │                                  │
│            │  (Message Bus)  │                                  │
│            └─────────────────┘                                  │
│                                                                  │
│  Benefits:                                                       │
│  • Each server handles subset of connections                    │
│  • Redis coordinates cross-server communication                 │
│  • Add/remove servers based on load                             │
│  • No single point of failure                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 8. Data Flow - Real-time Events

```
User A Types                                        User B Receives
    │                                                      │
    │ 1. typing.start                                     │
    ├──────────────────────────────────┐                  │
    │                                  │                  │
    │                                  ▼                  │
    │                      ┌───────────────────┐          │
    │                      │  WebSocket Server │          │
    │                      │                   │          │
    │                      │ 2. Validate user  │          │
    │                      │    in channel     │          │
    │                      └─────────┬─────────┘          │
    │                                │                    │
    │                                │ 3. Publish event   │
    │                                ▼                    │
    │                      ┌───────────────────┐          │
    │                      │   Redis Pub/Sub   │          │
    │                      │                   │          │
    │                      │ Channel: typing   │          │
    │                      └─────────┬─────────┘          │
    │                                │                    │
    │                                │ 4. Broadcast       │
    │                                ▼                    │
    │                      ┌───────────────────┐          │
    │                      │  All WS Servers   │          │
    │                      │                   │          │
    │                      │ 5. Filter by      │          │
    │                      │    channel members│          │
    │                      └─────────┬─────────┘          │
    │                                │                    │
    │                                │ 6. Send to User B  │
    │                                └────────────────────┤
    │                                                     │
    │                                                     ▼
    │                                          User B sees typing
```

This comprehensive set of diagrams should help visualize the entire Chat SDK architecture!
