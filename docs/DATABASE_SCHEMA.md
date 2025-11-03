# Chat SDK - Database Schema

**Last Updated**: November 3, 2025  
**Version**: 3.0 (Aligned with Dropper hierarchical structure)

---

## Overview

This schema follows the **Customer → Member → App** hierarchy pattern:

1. **Customer** - Organization/company account (your customers)
2. **Member** - Portal users who manage the platform (belong to a Customer)
3. **App** - Chat application instances (belong to a Customer)
4. **AppUser** - SDK end-users who participate in chat (belong to an App)

**Key Features:**
- Role-Based Access Control (RBAC) with customer-level and app-level roles
- API key management with publishable and secret keys
- Webhook endpoints for event notifications
- Multi-tenancy with proper data isolation

---

## PostgreSQL Schema

### Core Hierarchy

#### **customers**
```sql
CREATE TABLE customers (
  id VARCHAR(30) PRIMARY KEY, -- cuid
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  -- Customer information
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  phone VARCHAR(50),
  website VARCHAR(255),
  
  -- Billing information
  billing_email VARCHAR(255),
  billing_address TEXT,
  tax_id VARCHAR(100),
  
  -- External integration (e.g., ChargeBee, Stripe)
  external_billing_id VARCHAR(255) UNIQUE
);

CREATE INDEX idx_customers_email ON customers(email);
CREATE INDEX idx_customers_external_billing_id ON customers(external_billing_id);
```

#### **members**
```sql
CREATE TABLE members (
  id VARCHAR(30) PRIMARY KEY, -- cuid
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  -- Member information
  email VARCHAR(255) NOT NULL,
  name VARCHAR(255) NOT NULL,
  password_hash VARCHAR(255),
  avatar_url TEXT,
  last_login_at TIMESTAMP,
  is_active BOOLEAN DEFAULT true,
  
  -- Email verification
  email_verified BOOLEAN DEFAULT false,
  email_verification_token VARCHAR(255),
  email_verification_expires TIMESTAMP,
  
  -- Password reset
  password_reset_token VARCHAR(255),
  password_reset_expires TIMESTAMP,
  
  -- Authentication
  auth_provider VARCHAR(50), -- 'email', 'google', 'github', etc.
  auth_provider_id VARCHAR(255),
  
  -- Super Admin flag (platform-wide access)
  is_super_admin BOOLEAN DEFAULT false,
  
  -- Relationships
  customer_id VARCHAR(30) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  
  UNIQUE(email, customer_id)
);

CREATE INDEX idx_members_email ON members(email);
CREATE INDEX idx_members_customer_id ON members(customer_id);
CREATE INDEX idx_members_active ON members(is_active) WHERE is_active = true;
```

#### **refresh_tokens**
```sql
CREATE TABLE refresh_tokens (
  id VARCHAR(30) PRIMARY KEY, -- cuid
  created_at TIMESTAMP DEFAULT NOW(),
  
  -- Token information
  token VARCHAR(255) UNIQUE NOT NULL,
  expires_at TIMESTAMP NOT NULL,
  is_revoked BOOLEAN DEFAULT false,
  revoked_at TIMESTAMP,
  
  -- Device/session information
  user_agent TEXT,
  ip_address VARCHAR(45),
  
  -- Relationships
  member_id VARCHAR(30) NOT NULL REFERENCES members(id) ON DELETE CASCADE
);

CREATE INDEX idx_refresh_tokens_member_id ON refresh_tokens(member_id);
CREATE INDEX idx_refresh_tokens_token_expires ON refresh_tokens(token, expires_at);
```

#### **applications**
```sql
CREATE TABLE applications (
  id VARCHAR(30) PRIMARY KEY, -- cuid
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  -- App information
  name VARCHAR(255) NOT NULL,
  description TEXT,
  
  -- Environment configuration
  environment VARCHAR(20) DEFAULT 'PRODUCTION', -- 'PRODUCTION', 'TEST'
  
  -- Settings
  settings JSONB,
  
  -- Relationships
  customer_id VARCHAR(30) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  
  UNIQUE(customer_id, name)
);

CREATE INDEX idx_applications_customer_id ON applications(customer_id);
```

---

### API Key Management

#### **api_keys**
```sql
CREATE TABLE api_keys (
  id VARCHAR(30) PRIMARY KEY, -- cuid
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  -- Key information
  publishable_key VARCHAR(255) UNIQUE NOT NULL, -- pk_chat_test_xxx or pk_chat_live_xxx
  secret_key VARCHAR(255) UNIQUE NOT NULL, -- sk_chat_test_xxx or sk_chat_live_xxx (hashed)
  
  -- Status
  is_active BOOLEAN DEFAULT true,
  last_used_at TIMESTAMP,
  
  -- Key rotation
  expires_at TIMESTAMP,
  
  -- Relationships
  app_id VARCHAR(30) NOT NULL REFERENCES applications(id) ON DELETE CASCADE
);

CREATE INDEX idx_api_keys_app_id ON api_keys(app_id);
CREATE INDEX idx_api_keys_publishable_key ON api_keys(publishable_key);
CREATE INDEX idx_api_keys_secret_key ON api_keys(secret_key);

COMMENT ON COLUMN api_keys.publishable_key IS 'Client-side key (safe to expose)';
COMMENT ON COLUMN api_keys.secret_key IS 'Server-side key (must be kept secret, stored hashed)';
```

---

### RBAC (Role-Based Access Control)

#### **roles**
```sql
CREATE TABLE roles (
  id VARCHAR(30) PRIMARY KEY, -- cuid
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  -- Role information
  name VARCHAR(100) UNIQUE NOT NULL,
  description TEXT,
  is_system_role BOOLEAN DEFAULT false,
  
  -- RBAC configuration
  scope VARCHAR(20) DEFAULT 'APP', -- 'PLATFORM', 'CUSTOMER', 'APP'
  hierarchy_level INTEGER DEFAULT 0 -- Higher = more powerful (100=Super Admin, 80=Owner, 60=Admin, 40=Developer, 20=Viewer)
);

CREATE INDEX idx_roles_scope ON roles(scope);

COMMENT ON COLUMN roles.scope IS 'PLATFORM=Super Admin, CUSTOMER=Owner/Admin, APP=Developer/Viewer';
COMMENT ON COLUMN roles.hierarchy_level IS '100=Super Admin, 80=Owner, 60=Admin, 40=Developer, 20=Viewer';
```

#### **permissions**
```sql
CREATE TABLE permissions (
  id VARCHAR(30) PRIMARY KEY, -- cuid
  created_at TIMESTAMP DEFAULT NOW(),
  
  -- Permission information
  name VARCHAR(100) UNIQUE NOT NULL,
  description TEXT
);

CREATE INDEX idx_permissions_name ON permissions(name);
```

#### **role_permissions**
```sql
CREATE TABLE role_permissions (
  id VARCHAR(30) PRIMARY KEY, -- cuid
  created_at TIMESTAMP DEFAULT NOW(),
  
  -- CASL conditions for Prisma WHERE clauses
  conditions JSONB, -- Store CASL conditions for @casl/prisma
  
  -- Relationships
  role_id VARCHAR(30) NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
  permission_id VARCHAR(30) NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
  
  UNIQUE(role_id, permission_id)
);

CREATE INDEX idx_role_permissions_role_id ON role_permissions(role_id);
CREATE INDEX idx_role_permissions_permission_id ON role_permissions(permission_id);

COMMENT ON COLUMN role_permissions.conditions IS 'Example: {"appId": {"in": "{{user.appIds}}"}}';
```

#### **member_customer_roles**
```sql
CREATE TABLE member_customer_roles (
  id VARCHAR(30) PRIMARY KEY, -- cuid
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  -- Relationships
  member_id VARCHAR(30) NOT NULL REFERENCES members(id) ON DELETE CASCADE,
  customer_id VARCHAR(30) NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  role_id VARCHAR(30) NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
  
  UNIQUE(member_id, customer_id)
);

CREATE INDEX idx_member_customer_roles_member_id ON member_customer_roles(member_id);
CREATE INDEX idx_member_customer_roles_customer_id ON member_customer_roles(customer_id);

COMMENT ON TABLE member_customer_roles IS 'Customer-level role assignments (Owner, Admin)';
```

#### **member_app_roles**
```sql
CREATE TABLE member_app_roles (
  id VARCHAR(30) PRIMARY KEY, -- cuid
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  -- Relationships
  member_id VARCHAR(30) NOT NULL REFERENCES members(id) ON DELETE CASCADE,
  app_id VARCHAR(30) NOT NULL REFERENCES applications(id) ON DELETE CASCADE,
  role_id VARCHAR(30) NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
  
  UNIQUE(member_id, app_id, role_id)
);

CREATE INDEX idx_member_app_roles_member_id ON member_app_roles(member_id);
CREATE INDEX idx_member_app_roles_app_id ON member_app_roles(app_id);

COMMENT ON TABLE member_app_roles IS 'App-level role assignments (Developer, Viewer)';
```

---

### SDK Layer - End-Users

#### **app_users**
```sql
CREATE TABLE app_users (
  id VARCHAR(30) PRIMARY KEY, -- cuid
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  -- App user information
  external_id VARCHAR(255) NOT NULL, -- Customer's user ID
  username VARCHAR(255),
  display_name VARCHAR(255),
  avatar_url TEXT,
  bio TEXT,
  user_type VARCHAR(50) DEFAULT 'regular', -- 'regular', 'anonymous', 'guest', 'bot'
  is_deactivated BOOLEAN DEFAULT false,
  deactivated_at TIMESTAMP,
  metadata JSONB DEFAULT '{}',
  is_online BOOLEAN DEFAULT false,
  last_seen_at TIMESTAMP,
  
  -- Relationships
  app_id VARCHAR(30) NOT NULL REFERENCES applications(id) ON DELETE CASCADE,
  
  UNIQUE(app_id, external_id)
);

CREATE INDEX idx_app_users_app_id ON app_users(app_id);
CREATE INDEX idx_app_users_external_id ON app_users(app_id, external_id);
CREATE INDEX idx_app_users_online ON app_users(is_online) WHERE is_online = true;
CREATE INDEX idx_app_users_type ON app_users(user_type);
CREATE INDEX idx_app_users_active ON app_users(is_deactivated) WHERE is_deactivated = false;
```

#### **app_user_devices**
```sql
CREATE TABLE app_user_devices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_user_id UUID NOT NULL REFERENCES app_users(id) ON DELETE CASCADE,
  device_id VARCHAR(255) NOT NULL,
  push_provider VARCHAR(50), -- 'fcm', 'apns', 'web'
  push_token TEXT,
  platform VARCHAR(50), -- 'android', 'ios', 'web'
  app_version VARCHAR(50),
  os_version VARCHAR(50),
  last_active_at TIMESTAMP,
  registered_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(app_user_id, device_id)
);

CREATE INDEX idx_app_user_devices_user ON app_user_devices(app_user_id);
CREATE INDEX idx_app_user_devices_push_token ON app_user_devices(push_token);
CREATE INDEX idx_app_user_devices_platform ON app_user_devices(platform);
```

---

### Core Tables

#### **channels**
```sql
CREATE TABLE channels (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_id UUID NOT NULL REFERENCES applications(id) ON DELETE CASCADE,
  type VARCHAR(50) NOT NULL,
  name VARCHAR(255),
  description TEXT,
  avatar_url TEXT,
  is_distinct BOOLEAN DEFAULT false, -- Auto-dedupe for DMs
  is_frozen BOOLEAN DEFAULT false, -- Read-only mode
  slow_mode INTEGER DEFAULT 0, -- Seconds between messages
  cooldown INTEGER DEFAULT 0, -- Channel-wide rate limit
  metadata JSONB DEFAULT '{}',
  settings JSONB DEFAULT '{}',
  created_by UUID REFERENCES app_users(id), -- ✅ Changed from users
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  last_message_at TIMESTAMP,
  truncated_at TIMESTAMP -- Last time channel was truncated
);

CREATE INDEX idx_channels_app_id ON channels(app_id);
CREATE INDEX idx_channels_type ON channels(type);
CREATE INDEX idx_channels_last_message ON channels(last_message_at DESC);
CREATE INDEX idx_channels_distinct ON channels(is_distinct) WHERE is_distinct = true;
CREATE INDEX idx_channels_frozen ON channels(is_frozen) WHERE is_frozen = true;
```

#### **channel_members**
```sql
CREATE TABLE channel_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  channel_id UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
  app_user_id UUID NOT NULL REFERENCES app_users(id) ON DELETE CASCADE, -- ✅ Changed from user_id
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
  UNIQUE(channel_id, app_user_id) -- ✅ Changed
);

CREATE INDEX idx_channel_members_channel ON channel_members(channel_id);
CREATE INDEX idx_channel_members_user ON channel_members(app_user_id); -- ✅ Changed
```

#### **messages**
```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  channel_id UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
  app_user_id UUID NOT NULL REFERENCES app_users(id), -- ✅ Changed from user_id
  parent_message_id UUID REFERENCES messages(id),
  text TEXT,
  type VARCHAR(50) DEFAULT 'text', -- 'text', 'image', 'video', 'file', 'system', 'silent', 'pending', 'draft'
  status VARCHAR(50) DEFAULT 'sent', -- 'sent', 'pending', 'approved', 'rejected'
  is_silent BOOLEAN DEFAULT false, -- No push notification
  is_pinned BOOLEAN DEFAULT false,
  pinned_at TIMESTAMP,
  pinned_by UUID REFERENCES app_users(id), -- ✅ Changed from users
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
CREATE INDEX idx_messages_user ON messages(app_user_id); -- ✅ Changed
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
  app_user_id UUID NOT NULL REFERENCES app_users(id) ON DELETE CASCADE, -- ✅ Changed
  emoji VARCHAR(50) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(message_id, app_user_id, emoji) -- ✅ Changed
);

CREATE INDEX idx_reactions_message ON message_reactions(message_id);
CREATE INDEX idx_reactions_user ON message_reactions(app_user_id); -- ✅ Changed
```

#### **message_reads**
```sql
CREATE TABLE message_reads (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  message_id UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
  app_user_id UUID NOT NULL REFERENCES app_users(id) ON DELETE CASCADE, -- ✅ Changed
  read_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(message_id, app_user_id) -- ✅ Changed
);

CREATE INDEX idx_reads_message ON message_reads(message_id);
CREATE INDEX idx_reads_user ON message_reads(app_user_id); -- ✅ Changed
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

#### **message_drafts**
```sql
CREATE TABLE message_drafts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  channel_id UUID NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
  app_user_id UUID NOT NULL REFERENCES app_users(id) ON DELETE CASCADE,
  text TEXT,
  mentions JSONB DEFAULT '[]',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(channel_id, app_user_id)
);

CREATE INDEX idx_drafts_user ON message_drafts(app_user_id);
```

---

### Webhooks & Events

#### **webhook_endpoints**
```sql
CREATE TABLE webhook_endpoints (
  id VARCHAR(30) PRIMARY KEY, -- cuid
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  -- Webhook information
  url TEXT NOT NULL,
  description TEXT,
  secret VARCHAR(255) NOT NULL, -- For signature verification
  is_active BOOLEAN DEFAULT true,
  
  -- Event types to trigger this webhook (array of event names)
  events TEXT[], -- ['message.new', 'channel.created', 'user.joined', etc.]
  
  -- Relationships
  app_id VARCHAR(30) NOT NULL REFERENCES applications(id) ON DELETE CASCADE
);

CREATE INDEX idx_webhook_endpoints_app_id ON webhook_endpoints(app_id);
CREATE INDEX idx_webhook_endpoints_active ON webhook_endpoints(is_active) WHERE is_active = true;

COMMENT ON COLUMN webhook_endpoints.secret IS 'Used for HMAC signature verification';
COMMENT ON COLUMN webhook_endpoints.events IS 'Array of event types: message.new, channel.created, etc.';
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
  app_user_id UUID REFERENCES app_users(id), -- ✅ Changed
  reason VARCHAR(255),
  status VARCHAR(50) DEFAULT 'pending', -- 'pending', 'approved', 'rejected'
  reviewed_by UUID REFERENCES customer_users(id), -- ✅ Changed to customer_users
  reviewed_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_moderation_queue_app ON moderation_queue(app_id, status);
CREATE INDEX idx_moderation_queue_message ON moderation_queue(message_id);
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

---

### End-to-End Encryption Tables

#### **user_encryption_keys**
```sql
CREATE TABLE user_encryption_keys (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_user_id UUID NOT NULL REFERENCES app_users(id) ON DELETE CASCADE,
  device_id VARCHAR(255) NOT NULL,
  
  -- Identity Key (permanent)
  identity_public_key TEXT NOT NULL,
  
  -- Signed Pre-Key (rotates every 30 days)
  signed_prekey_id INTEGER NOT NULL,
  signed_prekey_public TEXT NOT NULL,
  signed_prekey_signature TEXT NOT NULL,
  
  -- Metadata
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(app_user_id, device_id)
);

CREATE INDEX idx_user_encryption_keys_user ON user_encryption_keys(app_user_id);
CREATE INDEX idx_user_encryption_keys_device ON user_encryption_keys(device_id);
```

#### **one_time_prekeys**
```sql
CREATE TABLE one_time_prekeys (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_user_id UUID NOT NULL REFERENCES app_users(id) ON DELETE CASCADE,
  device_id VARCHAR(255) NOT NULL,
  prekey_id INTEGER NOT NULL,
  public_key TEXT NOT NULL,
  is_used BOOLEAN DEFAULT false,
  used_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(app_user_id, device_id, prekey_id)
);

CREATE INDEX idx_one_time_prekeys_user_device ON one_time_prekeys(app_user_id, device_id);
CREATE INDEX idx_one_time_prekeys_unused ON one_time_prekeys(app_user_id, device_id, is_used) 
  WHERE is_used = false;
```

#### **encrypted_messages**
```sql
CREATE TABLE encrypted_messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  message_id UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
  recipient_device_id VARCHAR(255) NOT NULL,
  
  -- Encrypted content
  ciphertext TEXT NOT NULL,
  
  -- Key exchange info (for first message)
  ephemeral_public_key TEXT,
  used_prekey_id INTEGER,
  
  created_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(message_id, recipient_device_id)
);

CREATE INDEX idx_encrypted_messages_message ON encrypted_messages(message_id);
CREATE INDEX idx_encrypted_messages_device ON encrypted_messages(recipient_device_id);
```

#### **encryption_sessions**
```sql
CREATE TABLE encryption_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_user_id UUID NOT NULL REFERENCES app_users(id) ON DELETE CASCADE,
  device_id VARCHAR(255) NOT NULL,
  peer_user_id UUID NOT NULL REFERENCES app_users(id) ON DELETE CASCADE,
  peer_device_id VARCHAR(255) NOT NULL,
  
  -- Session state (encrypted, only client can decrypt)
  session_state TEXT NOT NULL,
  
  -- Metadata
  last_message_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  UNIQUE(app_user_id, device_id, peer_user_id, peer_device_id)
);

CREATE INDEX idx_encryption_sessions_user ON encryption_sessions(app_user_id, device_id);
CREATE INDEX idx_encryption_sessions_peer ON encryption_sessions(peer_user_id, peer_device_id);
```

**Note**: See [E2E_ENCRYPTION.md](./E2E_ENCRYPTION.md) for complete E2EE implementation details.

---

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
PORTAL LAYER:
customer_users (1) ──→ (N) refresh_tokens
customer_users (N) ──→ (M) teams (via customer_user_teams)
customer_user_teams (junction table)

APPLICATION LAYER:
teams (1) ──→ (1) applications
applications (1) ──→ (N) webhooks
applications (1) ──→ (N) campaigns
applications (1) ──→ (N) block_lists
applications (1) ──→ (N) app_users

SDK LAYER:
app_users (1) ──→ (N) messages
app_users (1) ──→ (N) app_user_devices
app_users (1) ──→ (N) channel_members
app_users (1) ──→ (N) message_reactions
app_users (1) ──→ (N) message_reads
app_users (1) ──→ (N) message_drafts

CHANNELS & MESSAGES:
channels (1) ──→ (N) messages
channels (1) ──→ (N) channel_members
messages (1) ──→ (N) message_attachments
messages (1) ──→ (N) message_reactions
messages (1) ──→ (N) message_reads

WEBHOOKS:
webhooks (N) ──→ (M) webhook_event_types (via webhook_subscriptions)
webhooks (1) ──→ (N) webhook_logs
```

**Key Notes**:
- ✅ **customer_users** manage the portal (your customers)
- ✅ **app_users** participate in chat (your customers' users)
- ✅ **customer_user_teams** is a many-to-many junction table
- ✅ All chat operations reference **app_users**, not customer_users
- ✅ Moderation is reviewed by **customer_users**

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
