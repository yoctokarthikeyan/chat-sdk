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
  metadata JSONB DEFAULT '{}',
  settings JSONB DEFAULT '{}',
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  last_message_at TIMESTAMP
);

CREATE INDEX idx_channels_app_id ON channels(app_id);
CREATE INDEX idx_channels_type ON channels(type);
CREATE INDEX idx_channels_last_message ON channels(last_message_at DESC);
```

#### **channel_members**
```sql
CREATE TABLE channel_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  channel_id UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role VARCHAR(50) DEFAULT 'member',
  joined_at TIMESTAMP DEFAULT NOW(),
  last_read_at TIMESTAMP,
  unread_count INTEGER DEFAULT 0,
  is_muted BOOLEAN DEFAULT false,
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
  type VARCHAR(50) DEFAULT 'text',
  metadata JSONB DEFAULT '{}',
  attachments JSONB DEFAULT '[]',
  mentions JSONB DEFAULT '[]',
  is_deleted BOOLEAN DEFAULT false,
  is_edited BOOLEAN DEFAULT false,
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
