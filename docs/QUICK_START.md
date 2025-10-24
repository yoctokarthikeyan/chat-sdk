# Chat SDK - Quick Start Guide

## 🚀 Getting Started in 5 Minutes

This guide will help you integrate the Chat SDK into your application quickly.

---

## 1. Installation

### Web (React)
```bash
npm install @yourchat/sdk @yourchat/react
```

### Mobile (React Native)
```bash
npm install @yourchat/sdk @yourchat/react-native
```

### iOS (Swift)
```ruby
# CocoaPods
pod 'YourChatSDK'
```

### Android (Kotlin)
```gradle
implementation 'com.yourchat:sdk:1.0.0'
```

---

## 2. Initialize the SDK

### React

```typescript
import { ChatProvider } from '@yourchat/react';

function App() {
  return (
    <ChatProvider
      apiKey="your-api-key"
      userId="user-123"
      token="jwt-token"
    >
      <YourApp />
    </ChatProvider>
  );
}
```

### Vanilla JavaScript

```javascript
import { ChatClient } from '@yourchat/sdk';

const client = new ChatClient({
  apiKey: 'your-api-key',
  userId: 'user-123',
  token: 'jwt-token'
});

await client.connect();
```

### iOS (Swift)

```swift
import YourChatSDK

let client = ChatClient(
    apiKey: "your-api-key",
    userId: "user-123",
    token: "jwt-token"
)

client.connect()
```

### Android (Kotlin)

```kotlin
import com.yourchat.sdk.ChatClient

val client = ChatClient(
    apiKey = "your-api-key",
    userId = "user-123",
    token = "jwt-token"
)

client.connect()
```

---

## 3. Create a Channel

```typescript
const channel = await client.channels.create({
  type: 'group',
  name: 'Team Chat',
  members: ['user-1', 'user-2', 'user-3']
});
```

---

## 4. Send a Message

```typescript
await channel.sendMessage({
  text: 'Hello team!'
});
```

---

## 5. Listen to Messages

```typescript
channel.on('message.new', (message) => {
  console.log('New message:', message.text);
});
```

---

## Complete React Example

```typescript
import { useChat, useChannel, useMessages } from '@yourchat/react';

function ChatComponent() {
  const { client } = useChat();
  const { channel } = useChannel('channel-id');
  const { messages, sendMessage } = useMessages(channel);

  const handleSend = (text: string) => {
    sendMessage({ text });
  };

  return (
    <div>
      <div className="messages">
        {messages.map(msg => (
          <div key={msg.id}>
            <strong>{msg.user.name}:</strong> {msg.text}
          </div>
        ))}
      </div>
      <input
        type="text"
        onKeyPress={(e) => {
          if (e.key === 'Enter') {
            handleSend(e.target.value);
            e.target.value = '';
          }
        }}
      />
    </div>
  );
}
```

---

## Using Pre-built Components

```typescript
import {
  Chat,
  ChannelList,
  Channel,
  MessageList,
  MessageInput
} from '@yourchat/react';

function App() {
  const [activeChannel, setActiveChannel] = useState(null);

  return (
    <Chat client={client}>
      <div className="chat-container">
        <ChannelList
          onChannelSelect={setActiveChannel}
        />
        {activeChannel && (
          <Channel channel={activeChannel}>
            <MessageList />
            <MessageInput />
          </Channel>
        )}
      </div>
    </Chat>
  );
}
```

---

## Authentication

### Get User Token (Backend)

```typescript
// Your backend API
import jwt from 'jsonwebtoken';

app.post('/api/chat/token', (req, res) => {
  const token = jwt.sign(
    {
      sub: req.user.id,
      app_id: 'your-app-id'
    },
    process.env.CHAT_SDK_SECRET,
    { expiresIn: '7d' }
  );

  res.json({ token });
});
```

### Use Token in Client

```typescript
// Fetch token from your backend
const response = await fetch('/api/chat/token');
const { token } = await response.json();

// Initialize SDK with token
const client = new ChatClient({
  apiKey: 'your-api-key',
  userId: currentUser.id,
  token: token
});
```

---

## Common Use Cases

### 1. Direct Message (1-on-1)

```typescript
const dmChannel = await client.channels.create({
  type: 'direct',
  members: [currentUserId, otherUserId]
});
```

### 2. Group Chat

```typescript
const groupChannel = await client.channels.create({
  type: 'group',
  name: 'Project Team',
  members: ['user-1', 'user-2', 'user-3']
});
```

### 3. Public Channel

```typescript
const publicChannel = await client.channels.create({
  type: 'public',
  name: 'General',
  description: 'Open to everyone'
});
```

### 4. Send Image

```typescript
// Upload image
const attachment = await client.uploadFile({
  file: imageFile,
  type: 'image'
});

// Send message with image
await channel.sendMessage({
  text: 'Check this out!',
  attachments: [attachment]
});
```

### 5. Typing Indicator

```typescript
// Start typing
channel.startTyping();

// Stop typing
channel.stopTyping();

// Listen to typing
channel.on('typing.start', (user) => {
  console.log(`${user.name} is typing...`);
});
```

### 6. Read Receipts

```typescript
// Mark as read
await channel.markRead();

// Listen to read events
channel.on('message.read', (event) => {
  console.log(`${event.user.name} read the message`);
});
```

### 7. Message Reactions

```typescript
// Add reaction
await message.addReaction('👍');

// Remove reaction
await message.removeReaction('👍');

// Listen to reactions
channel.on('reaction.new', (reaction) => {
  console.log(`${reaction.user.name} reacted with ${reaction.emoji}`);
});
```

---

## Configuration Options

```typescript
const client = new ChatClient({
  apiKey: 'your-api-key',
  userId: 'user-123',
  token: 'jwt-token',
  options: {
    // Auto-connect on initialization
    autoConnect: true,
    
    // Enable offline support
    enableOfflineSupport: true,
    
    // Log level
    logLevel: 'info', // 'debug' | 'info' | 'warn' | 'error'
    
    // Reconnection settings
    reconnectAttempts: 5,
    reconnectDelay: 1000,
    
    // Timeout settings
    connectionTimeout: 10000,
    requestTimeout: 5000,
    
    // Base URL (for self-hosted)
    baseURL: 'https://api.yourchat.io',
    
    // WebSocket URL
    wsURL: 'wss://ws.yourchat.io'
  }
});
```

---

## Error Handling

```typescript
try {
  await channel.sendMessage({ text: 'Hello' });
} catch (error) {
  if (error.code === 'NETWORK_ERROR') {
    console.error('Network error, message queued for retry');
  } else if (error.code === 'PERMISSION_DENIED') {
    console.error('You do not have permission to send messages');
  } else {
    console.error('Unknown error:', error);
  }
}
```

---

## Next Steps

1. **Read the full documentation**: [API Reference](./API_REFERENCE.md)
2. **Explore examples**: Check `/examples` folder
3. **Join the community**: Discord, GitHub Discussions
4. **Get support**: support@yourchat.io

---

## Useful Links

- 📚 [Full Documentation](./ARCHITECTURE.md)
- 🎨 [UI Components](./SDK_DESIGN.md)
- 🔧 [API Reference](./API_REFERENCE.md)
- 💰 [Pricing](./BUSINESS_MODEL.md)
- 🗺️ [Roadmap](./ROADMAP.md)
- 💬 [Community](https://discord.gg/yourchat)
- 🐛 [Report Issues](https://github.com/yourchat/sdk/issues)

---

## Need Help?

- **Documentation**: https://docs.yourchat.io
- **Email**: support@yourchat.io
- **Discord**: https://discord.gg/yourchat
- **GitHub**: https://github.com/yourchat/sdk
- **Twitter**: @yourchat

Happy coding! 🚀
