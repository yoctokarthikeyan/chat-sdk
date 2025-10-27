# Chat SDK - Database Schema

## PostgreSQL Schema

### Core Tables

#### **users**
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_id UUID NOT NULL REFERENCES applications(id),
  external_id VARCHAR(255) NOT NULL,
  username VARCHAR(255),
  email VARCHAR(255),
  password_hash VARCHAR(255),
  display_name VARCHAR(255),
  avatar_url TEXT,
  bio TEXT,
  user_type VARCHAR(50) DEFAULT 'regular', -- 'regular', 'anonymous', 'guest'
  is_deactivated BOOLEAN DEFAULT false,
  deactivated_at TIMESTAMP,
  team_id UUID REFERENCES teams(id), -- Multi-tenancy support
  metadata JSONB DEFAULT '{}',
  is_online BOOLEAN DEFAULT false,
  last_seen_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(app_id, external_id),
  UNIQUE(app_id, email)
);

CREATE INDEX idx_users_app_id ON users(app_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_online ON users(is_online) WHERE is_online = true;
CREATE INDEX idx_users_team ON users(team_id);
CREATE INDEX idx_users_type ON users(user_type);
CREATE INDEX idx_users_active ON users(is_deactivated) WHERE is_deactivated = false;
```

#### **channels**
```sql
CREATE TABLE channels (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_id UUID NOT NULL REFERENCES applications(id),
  type VARCHAR(50) NOT NULL,
  name VARCHAR(255),
  description TEXT,
  avatar_url TEXT,
  team_id UUID REFERENCES teams(id), -- Multi-tenancy support
  is_distinct BOOLEAN DEFAULT false, -- Auto-dedupe for DMs
  is_frozen BOOLEAN DEFAULT false, -- Read-only mode
  slow_mode INTEGER DEFAULT 0, -- Seconds between messages
  cooldown INTEGER DEFAULT 0, -- Channel-wide rate limit
  metadata JSONB DEFAULT '{}',
  settings JSONB DEFAULT '{}',
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  last_message_at TIMESTAMP,
  truncated_at TIMESTAMP -- Last time channel was truncated
);

CREATE INDEX idx_channels_app_id ON channels(app_id);
CREATE INDEX idx_channels_type ON channels(type);
CREATE INDEX idx_channels_last_message ON channels(last_message_at DESC);
CREATE INDEX idx_channels_team ON channels(team_id);
CREATE INDEX idx_channels_distinct ON channels(is_distinct) WHERE is_distinct = true;
CREATE INDEX idx_channels_frozen ON channels(is_frozen) WHERE is_frozen = true;
```

#### **channel_members**
```sql
CREATE TABLE channel_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  channel_id UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role VARCHAR(50) DEFAULT 'member', -- 'owner', 'admin', 'moderator', 'member'
  joined_at TIMESTAMP DEFAULT NOW(),
  last_read_at TIMESTAMP,
  unread_count INTEGER DEFAULT 0,
  is_muted BOOLEAN DEFAULT false,
  is_banned BOOLEAN DEFAULT false,
  is_shadow_banned BOOLEAN DEFAULT false,
  banned_at TIMESTAMP,
  ban_expires_at TIMESTAMP,
  is_hidden BOOLEAN DEFAULT false, -- User-specific channel visibility
  invite_accepted_at TIMESTAMP,
  invite_rejected_at TIMESTAMP,
  UNIQUE(channel_id, user_id)
);

CREATE INDEX idx_channel_members_channel ON channel_members(channel_id);
CREATE INDEX idx_channel_members_user ON channel_members(user_id);
```

#### **messages**
```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  channel_id UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id),
  parent_message_id UUID REFERENCES messages(id),
  text TEXT,
  type VARCHAR(50) DEFAULT 'text', -- 'text', 'image', 'video', 'file', 'system', 'silent', 'pending', 'draft'
  status VARCHAR(50) DEFAULT 'sent', -- 'sent', 'pending', 'approved', 'rejected'
  is_silent BOOLEAN DEFAULT false, -- No push notification
  is_pinned BOOLEAN DEFAULT false,
  pinned_at TIMESTAMP,
  pinned_by UUID REFERENCES users(id),
  scheduled_for TIMESTAMP, -- Scheduled messages
  metadata JSONB DEFAULT '{}',
  mentions JSONB DEFAULT '[]',
  quoted_message_id UUID REFERENCES messages(id), -- Quoted reply
  is_deleted BOOLEAN DEFAULT false,
  deleted_at TIMESTAMP,
  is_hard_deleted BOOLEAN DEFAULT false, -- Permanent deletion
  is_edited BOOLEAN DEFAULT false,
  edited_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_messages_channel_created ON messages(channel_id, created_at DESC);
CREATE INDEX idx_messages_user ON messages(user_id);
CREATE INDEX idx_messages_parent ON messages(parent_message_id);
CREATE INDEX idx_messages_text_search ON messages USING gin(to_tsvector('english', text));
CREATE INDEX idx_messages_quoted ON messages(quoted_message_id);
CREATE INDEX idx_messages_status ON messages(status) WHERE status != 'sent';
CREATE INDEX idx_messages_pinned ON messages(is_pinned) WHERE is_pinned = true;
```

#### **message_attachments**
```sql
CREATE TABLE message_attachments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  message_id UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
  attachment_type VARCHAR(50) NOT NULL, -- 'image', 'video', 'audio', 'file', 'location'
  url TEXT NOT NULL,
  thumbnail_url TEXT,
  file_name VARCHAR(255),
  file_size BIGINT, -- Size in bytes
  mime_type VARCHAR(100),
  width INTEGER, -- For images/videos
  height INTEGER, -- For images/videos
  duration INTEGER, -- For audio/video in seconds
  metadata JSONB DEFAULT '{}',
  upload_status VARCHAR(50) DEFAULT 'completed', -- 'uploading', 'completed', 'failed'
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_attachments_message ON message_attachments(message_id);
CREATE INDEX idx_attachments_type ON message_attachments(attachment_type);
```

#### **message_reactions**
```sql
CREATE TABLE message_reactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  message_id UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  emoji VARCHAR(50) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(message_id, user_id, emoji)
);

CREATE INDEX idx_reactions_message ON message_reactions(message_id);
```

#### **message_reads**
```sql
CREATE TABLE message_reads (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  message_id UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  read_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(message_id, user_id)
);

CREATE INDEX idx_reads_message ON message_reads(message_id);
CREATE INDEX idx_reads_user ON message_reads(user_id);
```

#### **teams**
```sql
CREATE TABLE teams (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_id UUID NOT NULL REFERENCES applications(id),
  name VARCHAR(255) NOT NULL,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_teams_app_id ON teams(app_id);
```

#### **user_devices**
```sql
CREATE TABLE user_devices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  device_id VARCHAR(255) NOT NULL,
  push_provider VARCHAR(50), -- 'fcm', 'apns'
  push_token TEXT,
  platform VARCHAR(50), -- 'android', 'ios', 'web'
  last_active_at TIMESTAMP,
  registered_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(user_id, device_id)
);

CREATE INDEX idx_devices_user ON user_devices(user_id);
CREATE INDEX idx_devices_push_token ON user_devices(push_token);
```

#### **webhooks**
```sql
CREATE TABLE webhooks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_id UUID NOT NULL REFERENCES applications(id),
  name VARCHAR(255) NOT NULL,
  url TEXT NOT NULL,
  webhook_type VARCHAR(50) NOT NULL, -- 'before_message_send', 'push', 'custom_action'
  secret VARCHAR(255), -- For signature verification
  is_active BOOLEAN DEFAULT true,
  retry_count INTEGER DEFAULT 3,
  timeout_ms INTEGER DEFAULT 5000,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_webhooks_app ON webhooks(app_id);
CREATE INDEX idx_webhooks_type ON webhooks(webhook_type);
```

#### **webhook_event_types**
```sql
CREATE TABLE webhook_event_types (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_name VARCHAR(100) UNIQUE NOT NULL, -- 'message.new', 'message.updated', 'channel.created', etc.
  category VARCHAR(50), -- 'message', 'channel', 'user', 'member', 'reaction', etc.
  description TEXT,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_webhook_event_types_category ON webhook_event_types(category);
CREATE INDEX idx_webhook_event_types_active ON webhook_event_types(is_active) WHERE is_active = true;

-- Pre-populate with standard event types
INSERT INTO webhook_event_types (event_name, category, description) VALUES
  ('message.new', 'message', 'Triggered when a new message is sent'),
  ('message.updated', 'message', 'Triggered when a message is edited'),
  ('message.deleted', 'message', 'Triggered when a message is deleted'),
  ('channel.created', 'channel', 'Triggered when a channel is created'),
  ('channel.updated', 'channel', 'Triggered when a channel is updated'),
  ('channel.deleted', 'channel', 'Triggered when a channel is deleted'),
  ('user.presence.changed', 'user', 'Triggered when user presence changes'),
  ('user.banned', 'user', 'Triggered when a user is banned'),
  ('user.updated', 'user', 'Triggered when user profile is updated'),
  ('member.added', 'member', 'Triggered when a member is added to a channel'),
  ('member.removed', 'member', 'Triggered when a member is removed from a channel'),
  ('reaction.new', 'reaction', 'Triggered when a reaction is added'),
  ('reaction.deleted', 'reaction', 'Triggered when a reaction is removed'),
  ('typing.start', 'typing', 'Triggered when a user starts typing'),
  ('typing.stop', 'typing', 'Triggered when a user stops typing');
```

#### **webhook_subscriptions**
```sql
CREATE TABLE webhook_subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  webhook_id UUID NOT NULL REFERENCES webhooks(id) ON DELETE CASCADE,
  event_type_id UUID NOT NULL REFERENCES webhook_event_types(id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(webhook_id, event_type_id)
);

CREATE INDEX idx_webhook_subs_webhook ON webhook_subscriptions(webhook_id);
CREATE INDEX idx_webhook_subs_event_type ON webhook_subscriptions(event_type_id);

-- Composite index for efficient lookups
CREATE INDEX idx_webhook_subs_lookup ON webhook_subscriptions(event_type_id, webhook_id);
```

#### **webhook_logs**
```sql
CREATE TABLE webhook_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  webhook_id UUID NOT NULL REFERENCES webhooks(id) ON DELETE CASCADE,
  event_type VARCHAR(100) NOT NULL,
  payload JSONB,
  response_status INTEGER,
  response_body TEXT,
  attempt_count INTEGER DEFAULT 1,
  delivered_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_webhook_logs_webhook ON webhook_logs(webhook_id, created_at DESC);
```

#### **campaigns**
```sql
CREATE TABLE campaigns (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_id UUID NOT NULL REFERENCES applications(id),
  name VARCHAR(255) NOT NULL,
  message JSONB NOT NULL, -- Message content
  target_channels UUID[],
  target_users UUID[],
  scheduled_for TIMESTAMP,
  status VARCHAR(50) DEFAULT 'draft', -- 'draft', 'scheduled', 'sending', 'completed', 'failed'
  sent_count INTEGER DEFAULT 0,
  delivered_count INTEGER DEFAULT 0,
  read_count INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_campaigns_app ON campaigns(app_id);
CREATE INDEX idx_campaigns_status ON campaigns(status);
CREATE INDEX idx_campaigns_scheduled ON campaigns(scheduled_for);
```

#### **block_lists**
```sql
CREATE TABLE block_lists (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_id UUID NOT NULL REFERENCES applications(id),
  name VARCHAR(255) NOT NULL,
  list_type VARCHAR(50) NOT NULL, -- 'word', 'phrase', 'domain', 'ip'
  entries TEXT[] NOT NULL,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_block_lists_app ON block_lists(app_id);
```

#### **moderation_queue**
```sql
CREATE TABLE moderation_queue (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_id UUID NOT NULL REFERENCES applications(id),
  message_id UUID REFERENCES messages(id),
  user_id UUID REFERENCES users(id),
  reason VARCHAR(255),
  status VARCHAR(50) DEFAULT 'pending', -- 'pending', 'approved', 'rejected'
  reviewed_by UUID REFERENCES users(id),
  reviewed_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_moderation_queue_app ON moderation_queue(app_id, status);
CREATE INDEX idx_moderation_queue_message ON moderation_queue(message_id);
```

#### **message_drafts**
```sql
CREATE TABLE message_drafts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  channel_id UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  text TEXT,
  mentions JSONB DEFAULT '[]',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(channel_id, user_id)
);

CREATE INDEX idx_drafts_user ON message_drafts(user_id);

-- Note: Draft attachments can be stored temporarily in message_attachments
-- with a reference to a draft message ID, or handled client-side until message is sent
```

#### **applications**
```sql
CREATE TABLE applications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  api_key VARCHAR(255) UNIQUE NOT NULL,
  api_secret VARCHAR(255) NOT NULL,
  plan_id UUID REFERENCES subscription_plans(id),
  settings JSONB DEFAULT '{}',
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Note: Webhook configuration is now managed via the 'webhooks' table
-- instead of storing webhook_url and webhook_secret directly in applications
```

#### **subscription_plans**
```sql
CREATE TABLE subscription_plans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  tier VARCHAR(50) NOT NULL,
  monthly_price DECIMAL(10, 2),
  yearly_price DECIMAL(10, 2),
  features JSONB DEFAULT '{}',
  limits JSONB DEFAULT '{}',
  created_at TIMESTAMP DEFAULT NOW()
);
```

#### **usage_metrics**
```sql
CREATE TABLE usage_metrics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_id UUID NOT NULL REFERENCES applications(id),
  metric_type VARCHAR(100) NOT NULL,
  value BIGINT NOT NULL,
  timestamp TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_usage_app_time ON usage_metrics(app_id, timestamp DESC);
```

## Redis Data Structures

### User Presence
```
Key: user:presence:{userId}
Type: String
Value: "online" | "offline" | "away"
TTL: 300 seconds

SET user:presence:user-123 "online" EX 300
```

### Typing Indicators
```
Key: channel:typing:{channelId}:{userId}
Type: String
Value: "1"
TTL: 5 seconds

SETEX channel:typing:channel-123:user-456 5 "1"
```

### Online Users in Channel
```
Key: channel:online:{channelId}
Type: Set
Members: userId[]

SADD channel:online:channel-123 user-456
SREM channel:online:channel-123 user-456
```

### Channel Watchers (Currently Viewing)
```
Key: channel:watchers:{channelId}
Type: Set
Members: userId[]
TTL: None (manually managed)

SADD channel:watchers:channel-123 user-456
SREM channel:watchers:channel-123 user-456
SMEMBERS channel:watchers:channel-123
```

### Draft Messages Cache
```
Key: draft:{channelId}:{userId}
Type: Hash
Fields: text, attachments, mentions, updated_at
TTL: 7 days

HSET draft:channel-123:user-456 text "Work in progress..."
HGET draft:channel-123:user-456 text
```

### Token Revocation List
```
Key: revoked:token:{tokenId}
Type: String
Value: "1"
TTL: Token expiration time

SET revoked:token:abc123 "1" EX 3600
EXISTS revoked:token:abc123
```

### Rate Limiting (Slow Mode)
```
Key: ratelimit:channel:{channelId}:{userId}
Type: String
Value: timestamp
TTL: slow_mode seconds

SET ratelimit:channel:channel-123:user-456 "1" EX 30
```

### Webhook Retry Queue
```
Key: webhook:retry:{webhookId}
Type: List
Members: JSON payloads

LPUSH webhook:retry:webhook-123 '{"event":"message.new",...}'
RPOP webhook:retry:webhook-123
```

### Session Storage
```
Key: session:{sessionId}
Type: Hash
Fields: userId, deviceId, lastActivity
TTL: 86400 seconds

HSET session:sess-123 userId user-456 deviceId dev-789
EXPIRE session:sess-123 86400
```

### Rate Limiting
```
Key: ratelimit:{userId}:{endpoint}
Type: String
Value: request count
TTL: 60 seconds

INCR ratelimit:user-123:send-message
EXPIRE ratelimit:user-123:send-message 60
```

### Message Queue
```
Key: queue:messages
Type: List

LPUSH queue:messages '{"messageId":"msg-123",...}'
RPOP queue:messages
```

---

## Database Schema Design Principles

### Relational Model Improvements

This schema follows best practices for relational database design by avoiding JSON arrays for relational data and using proper foreign key relationships instead.

#### **1. Webhook Events - Three-Table Relational Approach**

**Previous Design (Anti-pattern):**
```sql
-- ❌ Storing events as JSON array
CREATE TABLE webhooks (
  events TEXT[] -- ['message.new', 'channel.created', ...]
);
```

**Current Design (Best Practice):**
```sql
-- ✅ Proper three-table relational model

-- 1. Webhooks table (webhook configuration)
CREATE TABLE webhooks (
  id UUID PRIMARY KEY,
  app_id UUID REFERENCES applications(id),
  name VARCHAR(255) NOT NULL,
  url TEXT NOT NULL,
  -- ... other webhook config
);

-- 2. Webhook Event Types table (master list of all event types)
CREATE TABLE webhook_event_types (
  id UUID PRIMARY KEY,
  event_name VARCHAR(100) UNIQUE NOT NULL,
  category VARCHAR(50),
  description TEXT,
  is_active BOOLEAN DEFAULT true
);

-- 3. Webhook Subscriptions table (many-to-many relationship)
CREATE TABLE webhook_subscriptions (
  id UUID PRIMARY KEY,
  webhook_id UUID REFERENCES webhooks(id) ON DELETE CASCADE,
  event_type_id UUID REFERENCES webhook_event_types(id) ON DELETE CASCADE,
  UNIQUE(webhook_id, event_type_id)
);
```

**Benefits:**
- ✅ **Centralized Event Management**: Single source of truth for all event types
- ✅ **Query Efficiency**: Can efficiently query webhooks by event type using indexes
- ✅ **Data Integrity**: Foreign key constraints ensure referential integrity
- ✅ **Event Discovery**: Easy to list all available event types
- ✅ **Event Metadata**: Can add description, category, and other metadata per event
- ✅ **Scalability**: Better performance for large numbers of webhooks and events
- ✅ **Maintainability**: Easier to audit and manage event subscriptions
- ✅ **Analytics**: Track which events are most popular across all webhooks

**Example Queries:**
```sql
-- Find all webhooks subscribed to 'message.new'
SELECT w.* 
FROM webhooks w
JOIN webhook_subscriptions ws ON w.id = ws.webhook_id
JOIN webhook_event_types wet ON ws.event_type_id = wet.id
WHERE wet.event_name = 'message.new' AND w.is_active = true;

-- Get all events for a specific webhook
SELECT wet.event_name, wet.category, wet.description
FROM webhook_event_types wet
JOIN webhook_subscriptions ws ON wet.id = ws.event_type_id
WHERE ws.webhook_id = 'webhook-123'
ORDER BY wet.category, wet.event_name;

-- Count webhooks per event type
SELECT 
  wet.event_name,
  wet.category,
  COUNT(ws.webhook_id) as webhook_count
FROM webhook_event_types wet
LEFT JOIN webhook_subscriptions ws ON wet.id = ws.event_type_id
GROUP BY wet.id, wet.event_name, wet.category
ORDER BY webhook_count DESC;

-- Find all available event types by category
SELECT category, array_agg(event_name ORDER BY event_name) as events
FROM webhook_event_types
WHERE is_active = true
GROUP BY category;

-- Get webhooks NOT subscribed to a specific event
SELECT w.*
FROM webhooks w
WHERE w.id NOT IN (
  SELECT ws.webhook_id
  FROM webhook_subscriptions ws
  JOIN webhook_event_types wet ON ws.event_type_id = wet.id
  WHERE wet.event_name = 'message.new'
);

-- Most popular event types
SELECT 
  wet.event_name,
  wet.category,
  COUNT(ws.webhook_id) as subscription_count,
  ROUND(100.0 * COUNT(ws.webhook_id) / (SELECT COUNT(*) FROM webhooks), 2) as adoption_rate
FROM webhook_event_types wet
LEFT JOIN webhook_subscriptions ws ON wet.id = ws.event_type_id
GROUP BY wet.id, wet.event_name, wet.category
ORDER BY subscription_count DESC
LIMIT 10;
```

---

#### **2. Message Attachments - Relational Approach**

**Previous Design (Anti-pattern):**
```sql
-- ❌ Storing attachments as JSON
CREATE TABLE messages (
  attachments JSONB DEFAULT '[]' -- [{"type":"image","url":"..."}]
);
```

**Current Design (Best Practice):**
```sql
-- ✅ Proper relational model
CREATE TABLE messages (
  id UUID PRIMARY KEY,
  channel_id UUID REFERENCES channels(id),
  -- ... other fields (no attachments field)
);

CREATE TABLE message_attachments (
  id UUID PRIMARY KEY,
  message_id UUID REFERENCES messages(id) ON DELETE CASCADE,
  attachment_type VARCHAR(50) NOT NULL,
  url TEXT NOT NULL,
  thumbnail_url TEXT,
  file_name VARCHAR(255),
  file_size BIGINT,
  mime_type VARCHAR(100),
  width INTEGER,
  height INTEGER,
  duration INTEGER,
  metadata JSONB DEFAULT '{}',
  upload_status VARCHAR(50) DEFAULT 'completed',
  created_at TIMESTAMP DEFAULT NOW()
);
```

**Benefits:**
- ✅ **Type Safety**: Each attachment field has proper data type validation
- ✅ **Query Efficiency**: Can query attachments by type, size, status independently
- ✅ **Storage Optimization**: No JSON parsing overhead for large attachments
- ✅ **Indexing**: Can create indexes on attachment properties
- ✅ **Upload Tracking**: Can track upload status per attachment
- ✅ **File Management**: Easier to implement file cleanup and storage management

**Example Queries:**
```sql
-- Find all image attachments in a channel
SELECT ma.* FROM message_attachments ma
JOIN messages m ON ma.message_id = m.id
WHERE m.channel_id = 'channel-123' 
  AND ma.attachment_type = 'image'
ORDER BY ma.created_at DESC;

-- Calculate total storage used by user
SELECT u.id, u.username, SUM(ma.file_size) as total_bytes
FROM users u
JOIN messages m ON u.id = m.user_id
JOIN message_attachments ma ON m.id = ma.message_id
GROUP BY u.id, u.username;

-- Find large video files
SELECT m.id, m.text, ma.file_name, ma.file_size
FROM messages m
JOIN message_attachments ma ON m.id = ma.message_id
WHERE ma.attachment_type = 'video' 
  AND ma.file_size > 104857600 -- 100MB
ORDER BY ma.file_size DESC;

-- Track upload failures
SELECT COUNT(*) as failed_uploads
FROM message_attachments
WHERE upload_status = 'failed'
  AND created_at > NOW() - INTERVAL '24 hours';
```

---

#### **3. Application Webhooks - Separation of Concerns**

**Previous Design (Anti-pattern):**
```sql
-- ❌ Mixing webhook config with application settings
CREATE TABLE applications (
  webhook_url TEXT,
  webhook_secret VARCHAR(255)
  -- Only supports one webhook per application
);
```

**Current Design (Best Practice):**
```sql
-- ✅ Dedicated webhooks table
CREATE TABLE applications (
  id UUID PRIMARY KEY,
  -- ... application fields only
  -- No webhook fields
);

CREATE TABLE webhooks (
  id UUID PRIMARY KEY,
  app_id UUID REFERENCES applications(id),
  name VARCHAR(255) NOT NULL,
  url TEXT NOT NULL,
  webhook_type VARCHAR(50) NOT NULL,
  secret VARCHAR(255),
  -- ... webhook-specific configuration
);
```

**Benefits:**
- ✅ **Multiple Webhooks**: Applications can have multiple webhooks
- ✅ **Webhook Types**: Support different webhook types (before_send, push, custom)
- ✅ **Independent Management**: Webhooks can be managed separately from applications
- ✅ **Audit Trail**: Track webhook changes via webhook_logs table
- ✅ **Scalability**: Easy to add webhook-specific features

**Example Queries:**
```sql
-- Get all webhooks for an application
SELECT * FROM webhooks
WHERE app_id = 'app-123' AND is_active = true;

-- Find webhooks by type
SELECT w.*, a.name as app_name
FROM webhooks w
JOIN applications a ON w.app_id = a.id
WHERE w.webhook_type = 'before_message_send';

-- Webhook delivery success rate
SELECT 
  w.id,
  w.name,
  COUNT(wl.id) as total_deliveries,
  SUM(CASE WHEN wl.response_status BETWEEN 200 AND 299 THEN 1 ELSE 0 END) as successful,
  ROUND(100.0 * SUM(CASE WHEN wl.response_status BETWEEN 200 AND 299 THEN 1 ELSE 0 END) / COUNT(wl.id), 2) as success_rate
FROM webhooks w
LEFT JOIN webhook_logs wl ON w.id = wl.webhook_id
WHERE wl.created_at > NOW() - INTERVAL '7 days'
GROUP BY w.id, w.name;
```

---

### Key Relationships Summary

```
applications (1) ──→ (N) webhooks
webhooks (N) ──→ (M) webhook_event_types (via webhook_subscriptions)
webhooks (1) ──→ (N) webhook_logs
webhook_event_types (1) ──→ (N) webhook_subscriptions

messages (1) ──→ (N) message_attachments
messages (1) ──→ (N) message_reactions
messages (1) ──→ (N) message_reads

channels (1) ──→ (N) messages
channels (1) ──→ (N) channel_members

users (1) ──→ (N) messages
users (1) ──→ (N) user_devices
users (N) ──→ (1) teams

applications (1) ──→ (N) teams
applications (1) ──→ (N) campaigns
applications (1) ──→ (N) block_lists
```

**Note**: The webhook system uses a many-to-many relationship:
- One webhook can subscribe to multiple event types
- One event type can be used by multiple webhooks
- `webhook_subscriptions` is the junction table connecting them

---

### Migration Notes

When migrating from JSON-based storage to relational tables:

1. **Webhook Events Migration:**
```sql
-- Step 1: Create webhook_event_types table and populate with standard events
CREATE TABLE webhook_event_types (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_name VARCHAR(100) UNIQUE NOT NULL,
  category VARCHAR(50),
  description TEXT,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW()
);

INSERT INTO webhook_event_types (event_name, category, description) VALUES
  ('message.new', 'message', 'Triggered when a new message is sent'),
  ('message.updated', 'message', 'Triggered when a message is edited'),
  ('message.deleted', 'message', 'Triggered when a message is deleted'),
  ('channel.created', 'channel', 'Triggered when a channel is created'),
  ('channel.updated', 'channel', 'Triggered when a channel is updated'),
  ('channel.deleted', 'channel', 'Triggered when a channel is deleted'),
  ('user.presence.changed', 'user', 'Triggered when user presence changes'),
  ('user.banned', 'user', 'Triggered when a user is banned'),
  ('user.updated', 'user', 'Triggered when user profile is updated'),
  ('member.added', 'member', 'Triggered when a member is added to a channel'),
  ('member.removed', 'member', 'Triggered when a member is removed from a channel'),
  ('reaction.new', 'reaction', 'Triggered when a reaction is added'),
  ('reaction.deleted', 'reaction', 'Triggered when a reaction is removed'),
  ('typing.start', 'typing', 'Triggered when a user starts typing'),
  ('typing.stop', 'typing', 'Triggered when a user stops typing');

-- Step 2: Create webhook_subscriptions table
CREATE TABLE webhook_subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  webhook_id UUID NOT NULL REFERENCES webhooks(id) ON DELETE CASCADE,
  event_type_id UUID NOT NULL REFERENCES webhook_event_types(id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(webhook_id, event_type_id)
);

-- Step 3: Migrate existing events from JSON array to subscriptions
INSERT INTO webhook_subscriptions (webhook_id, event_type_id)
SELECT 
  w.id,
  wet.id
FROM webhooks w,
unnest(w.events) as event_name
JOIN webhook_event_types wet ON wet.event_name = event_name
WHERE w.events IS NOT NULL;

-- Step 4: Remove old events column
ALTER TABLE webhooks DROP COLUMN IF EXISTS events;

-- Step 5: Create indexes
CREATE INDEX idx_webhook_subs_webhook ON webhook_subscriptions(webhook_id);
CREATE INDEX idx_webhook_subs_event_type ON webhook_subscriptions(event_type_id);
CREATE INDEX idx_webhook_subs_lookup ON webhook_subscriptions(event_type_id, webhook_id);
```

2. **Message Attachments Migration:**
```sql
-- Extract attachments from JSON to relational table
INSERT INTO message_attachments (
  message_id, attachment_type, url, file_name, file_size, mime_type
)
SELECT 
  m.id,
  a->>'type',
  a->>'url',
  a->>'fileName',
  (a->>'size')::BIGINT,
  a->>'mimeType'
FROM messages m,
jsonb_array_elements(m.attachments) as a
WHERE m.attachments IS NOT NULL AND jsonb_array_length(m.attachments) > 0;

-- Remove old attachments column
ALTER TABLE messages DROP COLUMN attachments;
```

3. **Application Webhooks Migration:**
```sql
-- Migrate existing webhook config to webhooks table
INSERT INTO webhooks (app_id, name, url, secret, webhook_type, is_active)
SELECT 
  id,
  name || ' Default Webhook',
  webhook_url,
  webhook_secret,
  'push',
  true
FROM applications
WHERE webhook_url IS NOT NULL;

-- Remove old webhook columns
ALTER TABLE applications 
  DROP COLUMN webhook_url,
  DROP COLUMN webhook_secret;
```

---

### Performance Considerations

1. **Indexes**: All foreign keys have corresponding indexes for optimal join performance
2. **Cascade Deletes**: ON DELETE CASCADE ensures referential integrity without orphaned records
3. **Partial Indexes**: Used for boolean flags to optimize queries on filtered data
4. **GIN Indexes**: Used for full-text search and JSONB queries where appropriate
5. **Composite Indexes**: Created for common query patterns (e.g., channel_id + created_at)

---

### Data Integrity Rules

1. **Foreign Keys**: All relationships enforced via foreign key constraints
2. **Unique Constraints**: Prevent duplicate webhook events, channel members, etc.
3. **Check Constraints**: Can be added for business rules (e.g., file_size > 0)
4. **NOT NULL**: Required fields marked as NOT NULL
5. **Default Values**: Sensible defaults for optional fields
