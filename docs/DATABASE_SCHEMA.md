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
  attachments JSONB DEFAULT '[]',
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
  events TEXT[], -- Array of event types to subscribe to
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
  attachments JSONB DEFAULT '[]',
  mentions JSONB DEFAULT '[]',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(channel_id, user_id)
);

CREATE INDEX idx_drafts_user ON message_drafts(user_id);
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
  webhook_url TEXT,
  webhook_secret VARCHAR(255),
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
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
