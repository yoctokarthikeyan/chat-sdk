# Chat SDK - API Reference

## REST API Endpoints

Base URL: `https://api.yourchat.io/v1`

All requests require authentication via Bearer token in the Authorization header:
```
Authorization: Bearer {jwt-token}
```

---

## 1. Authentication

### Register User
```http
POST /api/v1/auth/register
Content-Type: application/json

Request:
{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "secure_password",
  "displayName": "John Doe"
}

Response 201:
{
  "userId": "user-123",
  "token": "jwt-token",
  "refreshToken": "refresh-token",
  "expiresIn": 900
}
```

### Login
```http
POST /api/v1/auth/login
Content-Type: application/json

Request:
{
  "email": "john@example.com",
  "password": "secure_password"
}

Response 200:
{
  "userId": "user-123",
  "token": "jwt-token",
  "refreshToken": "refresh-token",
  "expiresIn": 900
}
```

### Refresh Token
```http
POST /api/v1/auth/refresh
Content-Type: application/json

Request:
{
  "refreshToken": "refresh-token"
}

Response 200:
{
  "token": "new-jwt-token",
  "expiresIn": 900
}
```

### Logout
```http
POST /api/v1/auth/logout
Authorization: Bearer {token}

Response 204: No Content
```

### Get Current User
```http
GET /api/v1/auth/me
Authorization: Bearer {token}

Response 200:
{
  "id": "user-123",
  "username": "john_doe",
  "email": "john@example.com",
  "displayName": "John Doe",
  "avatarUrl": "https://cdn.example.com/avatar.jpg",
  "isOnline": true,
  "lastSeenAt": "2024-01-15T10:30:00Z",
  "createdAt": "2024-01-01T00:00:00Z",
  "metadata": {}
}
```

---

## 2. Users

### Get User by ID
```http
GET /api/v1/users/{userId}
Authorization: Bearer {token}

Response 200:
{
  "id": "user-456",
  "username": "jane_doe",
  "displayName": "Jane Doe",
  "avatarUrl": "https://cdn.example.com/avatar2.jpg",
  "bio": "Software Engineer",
  "isOnline": false,
  "lastSeenAt": "2024-01-15T09:00:00Z",
  "metadata": {}
}
```

### Update User Profile
```http
PUT /api/v1/users/{userId}
Authorization: Bearer {token}
Content-Type: application/json

Request:
{
  "displayName": "John Smith",
  "avatarUrl": "https://cdn.example.com/new-avatar.jpg",
  "bio": "Software Engineer",
  "metadata": {
    "department": "Engineering",
    "location": "San Francisco"
  }
}

Response 200:
{
  "id": "user-123",
  "displayName": "John Smith",
  "avatarUrl": "https://cdn.example.com/new-avatar.jpg",
  "bio": "Software Engineer",
  "metadata": {
    "department": "Engineering",
    "location": "San Francisco"
  }
}
```

### Search Users
```http
GET /api/v1/users/search?q=john&limit=10&offset=0
Authorization: Bearer {token}

Response 200:
{
  "users": [
    {
      "id": "user-123",
      "username": "john_doe",
      "displayName": "John Doe",
      "avatarUrl": "https://cdn.example.com/avatar.jpg",
      "isOnline": true
    }
  ],
  "total": 1,
  "limit": 10,
  "offset": 0
}
```

### Block User
```http
POST /api/v1/users/{userId}/block
Authorization: Bearer {token}

Response 204: No Content
```

### Unblock User
```http
DELETE /api/v1/users/{userId}/block
Authorization: Bearer {token}

Response 204: No Content
```

### Get Blocked Users
```http
GET /api/v1/users/blocked
Authorization: Bearer {token}

Response 200:
{
  "users": [
    {
      "id": "user-789",
      "username": "blocked_user",
      "displayName": "Blocked User",
      "blockedAt": "2024-01-10T12:00:00Z"
    }
  ]
}
```

---

## 3. Channels

### Create Channel
```http
POST /api/v1/channels
Authorization: Bearer {token}
Content-Type: application/json

Request:
{
  "type": "group",
  "name": "Team Chat",
  "description": "Engineering team discussion",
  "members": ["user-1", "user-2", "user-3"],
  "avatarUrl": "https://cdn.example.com/channel-avatar.jpg",
  "metadata": {
    "department": "Engineering"
  }
}

Response 201:
{
  "id": "channel-123",
  "type": "group",
  "name": "Team Chat",
  "description": "Engineering team discussion",
  "avatarUrl": "https://cdn.example.com/channel-avatar.jpg",
  "members": [
    {
      "userId": "user-1",
      "role": "owner",
      "joinedAt": "2024-01-15T10:00:00Z"
    },
    {
      "userId": "user-2",
      "role": "member",
      "joinedAt": "2024-01-15T10:00:00Z"
    }
  ],
  "createdBy": "user-1",
  "createdAt": "2024-01-15T10:00:00Z",
  "metadata": {
    "department": "Engineering"
  }
}
```

### Get Channel
```http
GET /api/v1/channels/{channelId}
Authorization: Bearer {token}

Response 200:
{
  "id": "channel-123",
  "type": "group",
  "name": "Team Chat",
  "description": "Engineering team discussion",
  "avatarUrl": "https://cdn.example.com/channel-avatar.jpg",
  "members": [
    {
      "userId": "user-1",
      "role": "owner",
      "joinedAt": "2024-01-15T10:00:00Z",
      "unreadCount": 0
    }
  ],
  "unreadCount": 5,
  "lastMessage": {
    "id": "msg-456",
    "text": "Hello team!",
    "userId": "user-2",
    "createdAt": "2024-01-15T11:00:00Z"
  },
  "createdAt": "2024-01-15T10:00:00Z",
  "updatedAt": "2024-01-15T11:00:00Z"
}
```

### Update Channel
```http
PUT /api/v1/channels/{channelId}
Authorization: Bearer {token}
Content-Type: application/json

Request:
{
  "name": "Updated Team Chat",
  "description": "New description",
  "avatarUrl": "https://cdn.example.com/new-avatar.jpg",
  "metadata": {
    "department": "Engineering",
    "project": "Chat SDK"
  }
}

Response 200:
{
  "id": "channel-123",
  "name": "Updated Team Chat",
  "description": "New description",
  "avatarUrl": "https://cdn.example.com/new-avatar.jpg",
  "metadata": {
    "department": "Engineering",
    "project": "Chat SDK"
  }
}
```

### Delete Channel
```http
DELETE /api/v1/channels/{channelId}
Authorization: Bearer {token}

Response 204: No Content
```

### Query Channels
```http
POST /api/v1/channels/query
Authorization: Bearer {token}
Content-Type: application/json

Request:
{
  "filter": {
    "type": "group",
    "members": { "$in": ["user-123"] }
  },
  "sort": { "lastMessageAt": -1 },
  "limit": 20,
  "offset": 0
}

Response 200:
{
  "channels": [
    {
      "id": "channel-123",
      "type": "group",
      "name": "Team Chat",
      "unreadCount": 5,
      "lastMessage": {
        "id": "msg-456",
        "text": "Hello!",
        "createdAt": "2024-01-15T11:00:00Z"
      }
    }
  ],
  "total": 15,
  "hasMore": false
}
```

### Add Members to Channel
```http
POST /api/v1/channels/{channelId}/members
Authorization: Bearer {token}
Content-Type: application/json

Request:
{
  "members": ["user-4", "user-5"],
  "role": "member"
}

Response 200:
{
  "channel": {
    "id": "channel-123",
    "name": "Team Chat"
  },
  "members": [
    {
      "userId": "user-4",
      "role": "member",
      "joinedAt": "2024-01-15T12:00:00Z"
    },
    {
      "userId": "user-5",
      "role": "member",
      "joinedAt": "2024-01-15T12:00:00Z"
    }
  ]
}
```

### Remove Member from Channel
```http
DELETE /api/v1/channels/{channelId}/members/{userId}
Authorization: Bearer {token}

Response 204: No Content
```

### Leave Channel
```http
POST /api/v1/channels/{channelId}/leave
Authorization: Bearer {token}

Response 204: No Content
```

### Mark Channel as Read
```http
POST /api/v1/channels/{channelId}/read
Authorization: Bearer {token}

Response 200:
{
  "channelId": "channel-123",
  "unreadCount": 0,
  "lastReadAt": "2024-01-15T12:00:00Z"
}
```

---

## 4. Messages

### Send Message
```http
POST /api/v1/channels/{channelId}/messages
Authorization: Bearer {token}
Content-Type: application/json

Request:
{
  "text": "Hello team!",
  "mentions": ["user-2"],
  "attachments": [
    {
      "type": "image",
      "url": "https://cdn.example.com/image.jpg",
      "thumbnailUrl": "https://cdn.example.com/thumb.jpg",
      "size": 1024000,
      "mimeType": "image/jpeg",
      "width": 1920,
      "height": 1080
    }
  ],
  "metadata": {
    "priority": "high"
  }
}

Response 201:
{
  "id": "msg-123",
  "channelId": "channel-123",
  "userId": "user-1",
  "text": "Hello team!",
  "mentions": ["user-2"],
  "attachments": [
    {
      "type": "image",
      "url": "https://cdn.example.com/image.jpg",
      "thumbnailUrl": "https://cdn.example.com/thumb.jpg",
      "size": 1024000,
      "mimeType": "image/jpeg",
      "width": 1920,
      "height": 1080
    }
  ],
  "metadata": {
    "priority": "high"
  },
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T10:30:00Z"
}
```

### Get Messages
```http
GET /api/v1/channels/{channelId}/messages?limit=50&before=msg-123
Authorization: Bearer {token}

Query Parameters:
- limit: Number of messages to return (default: 50, max: 100)
- before: Message ID to fetch messages before
- after: Message ID to fetch messages after

Response 200:
{
  "messages": [
    {
      "id": "msg-123",
      "channelId": "channel-123",
      "userId": "user-1",
      "text": "Hello team!",
      "createdAt": "2024-01-15T10:30:00Z"
    }
  ],
  "hasMore": true
}
```

### Get Single Message
```http
GET /api/v1/messages/{messageId}
Authorization: Bearer {token}

Response 200:
{
  "id": "msg-123",
  "channelId": "channel-123",
  "userId": "user-1",
  "text": "Hello team!",
  "mentions": ["user-2"],
  "attachments": [],
  "reactions": [
    {
      "emoji": "👍",
      "count": 3,
      "users": ["user-2", "user-3", "user-4"]
    }
  ],
  "readBy": [
    {
      "userId": "user-2",
      "readAt": "2024-01-15T10:35:00Z"
    }
  ],
  "isEdited": false,
  "isDeleted": false,
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T10:30:00Z"
}
```

### Update Message
```http
PUT /api/v1/messages/{messageId}
Authorization: Bearer {token}
Content-Type: application/json

Request:
{
  "text": "Updated message text"
}

Response 200:
{
  "id": "msg-123",
  "text": "Updated message text",
  "isEdited": true,
  "updatedAt": "2024-01-15T10:35:00Z"
}
```

### Delete Message
```http
DELETE /api/v1/messages/{messageId}
Authorization: Bearer {token}

Response 204: No Content
```

### Reply to Message (Threading)
```http
POST /api/v1/messages/{messageId}/replies
Authorization: Bearer {token}
Content-Type: application/json

Request:
{
  "text": "This is a reply"
}

Response 201:
{
  "id": "msg-456",
  "parentMessageId": "msg-123",
  "channelId": "channel-123",
  "userId": "user-2",
  "text": "This is a reply",
  "createdAt": "2024-01-15T10:40:00Z"
}
```

### Get Message Replies
```http
GET /api/v1/messages/{messageId}/replies?limit=50
Authorization: Bearer {token}

Response 200:
{
  "replies": [
    {
      "id": "msg-456",
      "parentMessageId": "msg-123",
      "userId": "user-2",
      "text": "This is a reply",
      "createdAt": "2024-01-15T10:40:00Z"
    }
  ],
  "total": 1
}
```

### Add Reaction
```http
POST /api/v1/messages/{messageId}/reactions
Authorization: Bearer {token}
Content-Type: application/json

Request:
{
  "emoji": "👍"
}

Response 201:
{
  "messageId": "msg-123",
  "userId": "user-1",
  "emoji": "👍",
  "createdAt": "2024-01-15T10:40:00Z"
}
```

### Remove Reaction
```http
DELETE /api/v1/messages/{messageId}/reactions/{emoji}
Authorization: Bearer {token}

Response 204: No Content
```

### Search Messages
```http
GET /api/v1/messages/search?q=hello&channelId=channel-123&limit=20
Authorization: Bearer {token}

Query Parameters:
- q: Search query (required)
- channelId: Filter by channel (optional)
- userId: Filter by user (optional)
- limit: Number of results (default: 20, max: 100)
- offset: Pagination offset

Response 200:
{
  "messages": [
    {
      "id": "msg-123",
      "channelId": "channel-123",
      "userId": "user-1",
      "text": "Hello team!",
      "createdAt": "2024-01-15T10:30:00Z",
      "highlights": ["<em>Hello</em> team!"]
    }
  ],
  "total": 5,
  "hasMore": false
}
```

---

## 5. Media Upload

### Upload File
```http
POST /api/v1/media/upload
Authorization: Bearer {token}
Content-Type: multipart/form-data

Request:
- file: File to upload
- type: File type (image, video, audio, file)

Response 201:
{
  "id": "media-123",
  "type": "image",
  "url": "https://cdn.example.com/uploads/image.jpg",
  "thumbnailUrl": "https://cdn.example.com/uploads/thumb.jpg",
  "size": 1024000,
  "mimeType": "image/jpeg",
  "width": 1920,
  "height": 1080,
  "createdAt": "2024-01-15T10:30:00Z"
}
```

### Get Media
```http
GET /api/v1/media/{mediaId}
Authorization: Bearer {token}

Response 200:
{
  "id": "media-123",
  "type": "image",
  "url": "https://cdn.example.com/uploads/image.jpg",
  "thumbnailUrl": "https://cdn.example.com/uploads/thumb.jpg",
  "size": 1024000,
  "mimeType": "image/jpeg",
  "width": 1920,
  "height": 1080,
  "createdAt": "2024-01-15T10:30:00Z"
}
```

### Delete Media
```http
DELETE /api/v1/media/{mediaId}
Authorization: Bearer {token}

Response 204: No Content
```

---

## 6. WebSocket Events

### Connection

**Connect to WebSocket**
```javascript
const socket = io('wss://ws.yourchat.io', {
  auth: {
    token: 'jwt-token'
  }
});
```

### Client → Server Events

#### Authenticate
```javascript
socket.emit('authenticate', {
  token: 'jwt-token'
});
```

#### Join Channel
```javascript
socket.emit('channel.join', {
  channelId: 'channel-123'
});
```

#### Leave Channel
```javascript
socket.emit('channel.leave', {
  channelId: 'channel-123'
});
```

#### Send Message
```javascript
socket.emit('message.send', {
  channelId: 'channel-123',
  text: 'Hello!',
  mentions: ['user-2']
});
```

#### Start Typing
```javascript
socket.emit('typing.start', {
  channelId: 'channel-123'
});
```

#### Stop Typing
```javascript
socket.emit('typing.stop', {
  channelId: 'channel-123'
});
```

#### Update Presence
```javascript
socket.emit('presence.update', {
  status: 'online' | 'away' | 'offline'
});
```

### Server → Client Events

#### Connected
```javascript
socket.on('connected', (data) => {
  console.log('Connected:', data);
  // { userId: 'user-123', sessionId: 'session-456' }
});
```

#### Authenticated
```javascript
socket.on('authenticated', (data) => {
  console.log('Authenticated:', data);
  // { userId: 'user-123', success: true }
});
```

#### New Message
```javascript
socket.on('message.new', (message) => {
  console.log('New message:', message);
  // { id: 'msg-123', channelId: 'channel-123', text: 'Hello!', ... }
});
```

#### Message Updated
```javascript
socket.on('message.updated', (message) => {
  console.log('Message updated:', message);
  // { id: 'msg-123', text: 'Updated text', isEdited: true, ... }
});
```

#### Message Deleted
```javascript
socket.on('message.deleted', (data) => {
  console.log('Message deleted:', data);
  // { messageId: 'msg-123', channelId: 'channel-123' }
});
```

#### Typing Started
```javascript
socket.on('typing.started', (data) => {
  console.log('User typing:', data);
  // { channelId: 'channel-123', userId: 'user-2', user: { ... } }
});
```

#### Typing Stopped
```javascript
socket.on('typing.stopped', (data) => {
  console.log('User stopped typing:', data);
  // { channelId: 'channel-123', userId: 'user-2' }
});
```

#### User Presence Changed
```javascript
socket.on('user.presence.changed', (data) => {
  console.log('Presence changed:', data);
  // { userId: 'user-2', status: 'online', lastSeenAt: '...' }
});
```

#### Channel Updated
```javascript
socket.on('channel.updated', (channel) => {
  console.log('Channel updated:', channel);
  // { id: 'channel-123', name: 'New Name', ... }
});
```

#### Channel Deleted
```javascript
socket.on('channel.deleted', (data) => {
  console.log('Channel deleted:', data);
  // { channelId: 'channel-123' }
});
```

#### Member Added
```javascript
socket.on('member.added', (data) => {
  console.log('Member added:', data);
  // { channelId: 'channel-123', userId: 'user-4', user: { ... } }
});
```

#### Member Removed
```javascript
socket.on('member.removed', (data) => {
  console.log('Member removed:', data);
  // { channelId: 'channel-123', userId: 'user-3' }
});
```

#### Notification
```javascript
socket.on('notification.new', (notification) => {
  console.log('New notification:', notification);
  // { type: 'mention', message: { ... }, channel: { ... } }
});
```

---

## 7. Error Responses

### Error Format
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {}
  }
}
```

### Common Error Codes

| Status | Code | Description |
|--------|------|-------------|
| 400 | INVALID_REQUEST | Invalid request parameters |
| 401 | UNAUTHORIZED | Missing or invalid authentication |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Resource not found |
| 409 | CONFLICT | Resource conflict (e.g., duplicate) |
| 429 | RATE_LIMIT_EXCEEDED | Too many requests |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

### Specific Error Codes

| Code | Description |
|------|-------------|
| USER_NOT_FOUND | User does not exist |
| CHANNEL_NOT_FOUND | Channel does not exist |
| MESSAGE_NOT_FOUND | Message does not exist |
| PERMISSION_DENIED | User lacks required permission |
| CHANNEL_FULL | Channel has reached member limit |
| FILE_TOO_LARGE | Uploaded file exceeds size limit |
| INVALID_FILE_TYPE | File type not supported |
| TOKEN_EXPIRED | JWT token has expired |
| TOKEN_INVALID | JWT token is invalid |

---

## 8. Rate Limiting

### Rate Limit Headers
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1642252800
```

### Rate Limits by Tier

| Tier | Requests/Minute | Burst |
|------|-----------------|-------|
| Free | 60 | 10 |
| Basic | 300 | 50 |
| Advanced | 1000 | 100 |
| Enterprise | Custom | Custom |

### Rate Limit Response
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Please try again later.",
    "details": {
      "limit": 100,
      "remaining": 0,
      "resetAt": "2024-01-15T11:00:00Z"
    }
  }
}
```

---

## 9. Pagination

### Cursor-based Pagination
```http
GET /api/v1/channels/{channelId}/messages?limit=50&before=msg-123

Response:
{
  "messages": [...],
  "hasMore": true,
  "nextCursor": "msg-100"
}
```

### Offset-based Pagination
```http
GET /api/v1/users/search?q=john&limit=20&offset=0

Response:
{
  "users": [...],
  "total": 100,
  "limit": 20,
  "offset": 0,
  "hasMore": true
}
```

---

## 10. Webhooks

### Configure Webhook
```http
POST /api/v1/webhooks
Authorization: Bearer {token}
Content-Type: application/json

Request:
{
  "url": "https://your-app.com/webhooks/chat",
  "events": ["message.sent", "channel.created"],
  "secret": "your-webhook-secret"
}

Response 201:
{
  "id": "webhook-123",
  "url": "https://your-app.com/webhooks/chat",
  "events": ["message.sent", "channel.created"],
  "createdAt": "2024-01-15T10:00:00Z"
}
```

### Webhook Events

| Event | Description |
|-------|-------------|
| message.sent | New message sent |
| message.updated | Message updated |
| message.deleted | Message deleted |
| channel.created | Channel created |
| channel.updated | Channel updated |
| channel.deleted | Channel deleted |
| user.created | User registered |
| user.banned | User banned |
| moderation.flagged | Content flagged by moderation |

### Webhook Payload
```json
{
  "event": "message.sent",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "message": {
      "id": "msg-123",
      "channelId": "channel-123",
      "userId": "user-1",
      "text": "Hello!",
      "createdAt": "2024-01-15T10:30:00Z"
    }
  }
}
```

### Webhook Signature
```
X-Webhook-Signature: sha256=hash_of_payload
```

Verify signature:
```javascript
const crypto = require('crypto');
const signature = crypto
  .createHmac('sha256', webhookSecret)
  .update(JSON.stringify(payload))
  .digest('hex');
```

---

This API reference provides complete documentation for all REST and WebSocket endpoints in the Chat SDK.
