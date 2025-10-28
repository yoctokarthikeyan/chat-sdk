# Month 3: Core SDK Development - Implementation Guide

**File**: `MONTH_03_SDK.md`  
**Purpose**: Build TypeScript Core SDK and React SDK  
**Prerequisites**: Complete [MONTH_02_MESSAGING.md](./MONTH_02_MESSAGING.md)

---

## Updates for Prisma Migration

**Note**: The SDK is ORM-agnostic and doesn't directly interact with the database. However, type definitions have been updated to match the Prisma schema from Month 2:

- ✅ Updated type enums to match Prisma (e.g., `'REGULAR'` instead of `'regular'`)
- ✅ Added missing fields like `isDeactivated`, `isShadowBanned`, `parentMessageId`
- ✅ Updated `Attachment` interface to match Prisma schema
- ✅ Added comprehensive WebSocket event types
- ✅ Added API request/response types

All SDK functionality remains the same - only type definitions are aligned with the backend.

---

## Week 1-2: Core SDK (TypeScript)

### Task 1: SDK Package Setup

**Create `packages/sdk-core/package.json`**:

```json
{
  "name": "@chat-sdk/core",
  "version": "0.1.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "dev": "tsc --watch",
    "build": "tsc",
    "test": "jest",
    "prepublishOnly": "pnpm build"
  },
  "dependencies": {
    "socket.io-client": "^4.6.1",
    "axios": "^1.6.5",
    "eventemitter3": "^5.0.1"
  },
  "devDependencies": {
    "@types/node": "^20.11.5",
    "typescript": "^5.3.3",
    "jest": "^29.7.0",
    "ts-jest": "^29.1.1"
  }
}
```

**Create `packages/sdk-core/tsconfig.json`**:

```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.spec.ts"]
}
```

---

### Task 2: SDK Architecture

**Directory Structure**:

```
packages/sdk-core/src/
├── index.ts
├── client/
│   └── ChatClient.ts
├── auth/
│   └── AuthManager.ts
├── channels/
│   └── ChannelManager.ts
├── messages/
│   └── MessageManager.ts
├── websocket/
│   └── WebSocketManager.ts
├── storage/
│   └── LocalStorage.ts
├── types/
│   ├── index.ts
│   ├── user.types.ts
│   ├── channel.types.ts
│   └── message.types.ts
└── utils/
    ├── logger.ts
    └── retry.ts
```

---

### Task 3: Type Definitions

**Create `packages/sdk-core/src/types/index.ts`**:

```typescript
// User types - matching backend Prisma schema
export type UserType = 'REGULAR' | 'ANONYMOUS' | 'GUEST';

export interface User {
  id: string;
  email?: string;
  username?: string;
  displayName?: string;
  avatarUrl?: string;
  userType: UserType;
  isDeactivated?: boolean;
  isOnline?: boolean;
  createdAt: Date;
  updatedAt: Date;
}

// Channel types - matching backend Prisma schema
export type ChannelType = 'DIRECT' | 'GROUP' | 'PUBLIC';
export type MemberRole = 'OWNER' | 'ADMIN' | 'MEMBER';

export interface Channel {
  id: string;
  type: ChannelType;
  name?: string;
  description?: string;
  isDistinct?: boolean;
  isFrozen?: boolean;
  slowMode?: number;
  metadata?: Record<string, any>;
  members?: ChannelMember[];
  lastMessage?: Message;
  unreadCount?: number;
  createdAt: Date;
  updatedAt: Date;
}

export interface ChannelMember {
  id: string;
  userId: string;
  channelId: string;
  role: MemberRole;
  joinedAt: Date;
  lastReadAt?: Date;
  isBanned?: boolean;
  isShadowBanned?: boolean;
  isHidden?: boolean;
  user?: User;
}

// Message types - matching backend Prisma schema
export type MessageType = 'TEXT' | 'SYSTEM' | 'DELETED';
export type MessageStatus = 'SENT' | 'PENDING' | 'FAILED';

export interface Message {
  id: string;
  channelId: string;
  userId: string;
  text?: string;
  type: MessageType;
  status: MessageStatus;
  isSilent?: boolean;
  isPinned?: boolean;
  attachments?: Attachment[];
  mentions?: string[];
  quotedMessageId?: string;
  quotedMessage?: Message;
  parentMessageId?: string;
  metadata?: Record<string, any>;
  user?: User;
  createdAt: Date;
  updatedAt: Date;
  isDeleted?: boolean;
}

export interface Attachment {
  id: string;
  messageId: string;
  attachmentType: string;
  url: string;
  fileName?: string;
  fileSize?: number;
  mimeType?: string;
  createdAt: Date;
}

// SDK Configuration
export interface ChatClientConfig {
  apiUrl: string;
  wsUrl?: string;
  apiKey?: string;
  userId?: string;
  token?: string;
  autoConnect?: boolean;
  enableOfflineSupport?: boolean;
  logLevel?: 'debug' | 'info' | 'warn' | 'error';
}

// API Request/Response types
export interface CreateChannelRequest {
  type: ChannelType;
  name?: string;
  description?: string;
  isDistinct?: boolean;
  members?: string[];
  metadata?: Record<string, any>;
}

export interface SendMessageRequest {
  channelId: string;
  text?: string;
  type?: MessageType;
  silent?: boolean;
  metadata?: Record<string, any>;
  mentions?: string[];
  quotedMessageId?: string;
  attachments?: Array<{
    attachmentType: string;
    url: string;
    fileName?: string;
    fileSize?: number;
    mimeType?: string;
  }>;
}

export interface PaginationParams {
  limit?: number;
  before?: string;
  after?: string;
}

// WebSocket Events
export interface WebSocketEvents {
  'message.new': (message: Message) => void;
  'message.updated': (message: Message) => void;
  'message.deleted': (messageId: string) => void;
  'channel.updated': (channel: Channel) => void;
  'channel.deleted': (channelId: string) => void;
  'member.joined': (member: ChannelMember) => void;
  'member.left': (data: { channelId: string; userId: string }) => void;
  'typing.start': (data: { channelId: string; userId: string }) => void;
  'typing.stop': (data: { channelId: string; userId: string }) => void;
  'user.online': (userId: string) => void;
  'user.offline': (userId: string) => void;
  'connected': () => void;
  'disconnected': () => void;
  'error': (error: Error) => void;
}
```

---

### Task 4: WebSocket Manager

**Create `packages/sdk-core/src/websocket/WebSocketManager.ts`**:

```typescript
import { io, Socket } from 'socket.io-client';
import EventEmitter from 'eventemitter3';

export class WebSocketManager extends EventEmitter {
  private socket: Socket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private reconnectDelay = 1000;

  constructor(
    private wsUrl: string,
    private token: string,
  ) {
    super();
  }

  connect() {
    this.socket = io(this.wsUrl, {
      auth: { token: this.token },
      transports: ['websocket'],
      reconnection: true,
      reconnectionAttempts: this.maxReconnectAttempts,
      reconnectionDelay: this.reconnectDelay,
    });

    this.setupEventListeners();
  }

  private setupEventListeners() {
    if (!this.socket) return;

    this.socket.on('connect', () => {
      this.reconnectAttempts = 0;
      this.emit('connected');
    });

    this.socket.on('disconnect', (reason) => {
      this.emit('disconnected', reason);
    });

    this.socket.on('error', (error) => {
      this.emit('error', error);
    });

    // Message events
    this.socket.on('message.new', (data) => {
      this.emit('message.new', data);
    });

    this.socket.on('message.updated', (data) => {
      this.emit('message.updated', data);
    });

    this.socket.on('message.deleted', (data) => {
      this.emit('message.deleted', data);
    });

    // Typing events
    this.socket.on('typing.start', (data) => {
      this.emit('typing.start', data);
    });

    this.socket.on('typing.stop', (data) => {
      this.emit('typing.stop', data);
    });

    // Presence events
    this.socket.on('user.presence', (data) => {
      this.emit('user.presence', data);
    });
  }

  send(event: string, data: any) {
    if (!this.socket?.connected) {
      throw new Error('WebSocket not connected');
    }
    this.socket.emit(event, data);
  }

  joinChannel(channelId: string) {
    this.send('channel.join', { channelId });
  }

  leaveChannel(channelId: string) {
    this.send('channel.leave', { channelId });
  }

  disconnect() {
    this.socket?.disconnect();
    this.socket = null;
  }
}
```

---

### Task 5: Chat Client

**Create `packages/sdk-core/src/client/ChatClient.ts`**:

```typescript
import axios, { AxiosInstance } from 'axios';
import EventEmitter from 'eventemitter3';
import { WebSocketManager } from '../websocket/WebSocketManager';
import { ChannelManager } from '../channels/ChannelManager';
import { MessageManager } from '../messages/MessageManager';
import { AuthManager } from '../auth/AuthManager';
import { ChatClientConfig, User } from '../types';

export class ChatClient extends EventEmitter {
  private http: AxiosInstance;
  private ws: WebSocketManager | null = null;
  private currentUser: User | null = null;

  public auth: AuthManager;
  public channels: ChannelManager;
  public messages: MessageManager;

  constructor(private config: ChatClientConfig) {
    super();

    this.http = axios.create({
      baseURL: config.apiUrl,
      headers: {
        'Content-Type': 'application/json',
      },
    });

    // Set auth token if provided
    if (config.token) {
      this.setAuthToken(config.token);
    }

    // Initialize managers
    this.auth = new AuthManager(this.http);
    this.channels = new ChannelManager(this.http);
    this.messages = new MessageManager(this.http);

    // Auto-connect if enabled
    if (config.autoConnect && config.token) {
      this.connect();
    }
  }

  async connect() {
    if (!this.config.token) {
      throw new Error('Token required for connection');
    }

    const wsUrl = this.config.wsUrl || this.config.apiUrl.replace('http', 'ws');
    this.ws = new WebSocketManager(wsUrl, this.config.token);

    // Forward WebSocket events
    this.ws.on('connected', () => this.emit('connected'));
    this.ws.on('disconnected', (reason) => this.emit('disconnected', reason));
    this.ws.on('error', (error) => this.emit('error', error));
    this.ws.on('message.new', (msg) => this.emit('message.new', msg));
    this.ws.on('message.updated', (msg) => this.emit('message.updated', msg));
    this.ws.on('message.deleted', (msg) => this.emit('message.deleted', msg));
    this.ws.on('typing.start', (data) => this.emit('typing.start', data));
    this.ws.on('typing.stop', (data) => this.emit('typing.stop', data));
    this.ws.on('user.presence', (data) => this.emit('user.presence', data));

    this.ws.connect();

    // Fetch current user
    await this.fetchCurrentUser();
  }

  disconnect() {
    this.ws?.disconnect();
    this.ws = null;
    this.emit('disconnected', 'manual');
  }

  private async fetchCurrentUser() {
    try {
      const response = await this.http.get('/users/me');
      this.currentUser = response.data;
      this.emit('user.updated', this.currentUser);
    } catch (error) {
      this.emit('error', error);
    }
  }

  getCurrentUser(): User | null {
    return this.currentUser;
  }

  setAuthToken(token: string) {
    this.config.token = token;
    this.http.defaults.headers.common['Authorization'] = `Bearer ${token}`;
  }

  getWebSocket(): WebSocketManager | null {
    return this.ws;
  }
}
```

---

### Task 6: Channel Manager

**Create `packages/sdk-core/src/channels/ChannelManager.ts`**:

```typescript
import { AxiosInstance } from 'axios';
import { Channel } from '../types';

export class ChannelManager {
  constructor(private http: AxiosInstance) {}

  async create(data: {
    type: string;
    name?: string;
    members?: string[];
    isDistinct?: boolean;
  }): Promise<Channel> {
    const response = await this.http.post('/channels', data);
    return response.data;
  }

  async get(channelId: string): Promise<Channel> {
    const response = await this.http.get(`/channels/${channelId}`);
    return response.data;
  }

  async list(params?: { limit?: number; offset?: number }): Promise<Channel[]> {
    const response = await this.http.get('/channels', { params });
    return response.data;
  }

  async update(channelId: string, data: Partial<Channel>): Promise<Channel> {
    const response = await this.http.patch(`/channels/${channelId}`, data);
    return response.data;
  }

  async delete(channelId: string): Promise<void> {
    await this.http.delete(`/channels/${channelId}`);
  }

  async addMembers(channelId: string, userIds: string[]): Promise<void> {
    await this.http.post(`/channels/${channelId}/members`, { userIds });
  }

  async removeMember(channelId: string, userId: string): Promise<void> {
    await this.http.delete(`/channels/${channelId}/members/${userId}`);
  }

  async freeze(channelId: string): Promise<void> {
    await this.http.post(`/channels/${channelId}/freeze`);
  }

  async unfreeze(channelId: string): Promise<void> {
    await this.http.post(`/channels/${channelId}/unfreeze`);
  }
}
```

---

### Task 7: Export SDK

**Create `packages/sdk-core/src/index.ts`**:

```typescript
export { ChatClient } from './client/ChatClient';
export * from './types';
```

---

## Week 3-4: React SDK

### Task 1: React SDK Setup

**Create `packages/sdk-react/package.json`**:

```json
{
  "name": "@chat-sdk/react",
  "version": "0.1.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "dev": "tsc --watch",
    "build": "tsc",
    "test": "jest"
  },
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  },
  "dependencies": {
    "@chat-sdk/core": "workspace:*"
  },
  "devDependencies": {
    "@types/react": "^18.2.48",
    "@types/react-dom": "^18.2.18",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "typescript": "^5.3.3"
  }
}
```

---

### Task 2: Chat Provider

**Create `packages/sdk-react/src/ChatProvider.tsx`**:

```typescript
import React, { createContext, useContext, useEffect, useState } from 'react';
import { ChatClient, ChatClientConfig } from '@chat-sdk/core';

interface ChatContextValue {
  client: ChatClient | null;
  isConnected: boolean;
  currentUser: any;
}

const ChatContext = createContext<ChatContextValue>({
  client: null,
  isConnected: false,
  currentUser: null,
});

export const useChatContext = () => useContext(ChatContext);

interface ChatProviderProps {
  config: ChatClientConfig;
  children: React.ReactNode;
}

export const ChatProvider: React.FC<ChatProviderProps> = ({ config, children }) => {
  const [client] = useState(() => new ChatClient(config));
  const [isConnected, setIsConnected] = useState(false);
  const [currentUser, setCurrentUser] = useState(null);

  useEffect(() => {
    client.on('connected', () => setIsConnected(true));
    client.on('disconnected', () => setIsConnected(false));
    client.on('user.updated', (user) => setCurrentUser(user));

    return () => {
      client.disconnect();
    };
  }, [client]);

  return (
    <ChatContext.Provider value={{ client, isConnected, currentUser }}>
      {children}
    </ChatContext.Provider>
  );
};
```

---

### Task 3: useChannel Hook

**Create `packages/sdk-react/src/hooks/useChannel.ts`**:

```typescript
import { useState, useEffect } from 'react';
import { useChatContext } from '../ChatProvider';
import { Channel } from '@chat-sdk/core';

export const useChannel = (channelId: string) => {
  const { client } = useChatContext();
  const [channel, setChannel] = useState<Channel | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    if (!client || !channelId) return;

    const fetchChannel = async () => {
      try {
        setLoading(true);
        const data = await client.channels.get(channelId);
        setChannel(data);
      } catch (err) {
        setError(err as Error);
      } finally {
        setLoading(false);
      }
    };

    fetchChannel();

    // Join channel via WebSocket
    const ws = client.getWebSocket();
    ws?.joinChannel(channelId);

    return () => {
      ws?.leaveChannel(channelId);
    };
  }, [client, channelId]);

  return { channel, loading, error };
};
```

---

### Task 4: useMessages Hook

**Create `packages/sdk-react/src/hooks/useMessages.ts`**:

```typescript
import { useState, useEffect } from 'react';
import { useChatContext } from '../ChatProvider';
import { Message } from '@chat-sdk/core';

export const useMessages = (channelId: string) => {
  const { client } = useChatContext();
  const [messages, setMessages] = useState<Message[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (!client || !channelId) return;

    const fetchMessages = async () => {
      try {
        setLoading(true);
        const data = await client.messages.list(channelId);
        setMessages(data);
      } finally {
        setLoading(false);
      }
    };

    fetchMessages();

    // Listen for new messages
    const handleNewMessage = (msg: Message) => {
      if (msg.channelId === channelId) {
        setMessages((prev) => [...prev, msg]);
      }
    };

    client.on('message.new', handleNewMessage);

    return () => {
      client.off('message.new', handleNewMessage);
    };
  }, [client, channelId]);

  const sendMessage = async (text: string) => {
    if (!client) return;
    await client.messages.send(channelId, { text });
  };

  return { messages, loading, sendMessage };
};
```

---

### Task 5: Export React SDK

**Create `packages/sdk-react/src/index.ts`**:

```typescript
export { ChatProvider, useChatContext } from './ChatProvider';
export { useChannel } from './hooks/useChannel';
export { useMessages } from './hooks/useMessages';
export { useTypingIndicator } from './hooks/useTypingIndicator';
```

---

## Testing

**Create `packages/sdk-core/src/__tests__/ChatClient.spec.ts`**:

```typescript
import { ChatClient } from '../client/ChatClient';

describe('ChatClient', () => {
  it('should initialize with config', () => {
    const client = new ChatClient({
      apiUrl: 'http://localhost:3000',
      autoConnect: false,
    });

    expect(client).toBeDefined();
    expect(client.channels).toBeDefined();
    expect(client.messages).toBeDefined();
  });

  it('should set auth token', () => {
    const client = new ChatClient({
      apiUrl: 'http://localhost:3000',
      autoConnect: false,
    });

    client.setAuthToken('test-token');
    expect(client['config'].token).toBe('test-token');
  });
});
```

---

## Deliverables

- [x] Core SDK package (@chat-sdk/core)
- [x] TypeScript types
- [x] WebSocket manager
- [x] Channel & message managers
- [x] React SDK package (@chat-sdk/react)
- [x] React hooks (useChannel, useMessages)
- [x] Chat provider
- [x] Unit tests
- [x] NPM packages ready

**Next**: [MONTH_04_POLISH.md](./MONTH_04_POLISH.md)
