# Month 4: Polish & Launch Preparation - Implementation Guide

**File**: `MONTH_04_POLISH.md`  
**Purpose**: Push notifications, search, testing, documentation, beta launch  
**Prerequisites**: Complete [MONTH_03_SDK.md](./MONTH_03_SDK.md)

---

## Updates for Prisma Migration

**Note**: Month 4 includes several database-dependent features. All TypeORM code has been updated to use Prisma:

- ✅ Message Reactions - Converted to Prisma with proper relations
- ✅ Moderation System - Updated with Prisma models and enums
- ✅ Test examples - Updated to use PrismaService mocks
- ✅ Push Notifications - Updated to query devices via Prisma

---

## Week 1-2: Essential Features

### Task 1: Push Notifications (FCM/APNS)

**Install Dependencies**:

```bash
cd apps/server
pnpm add firebase-admin apn
```

**Create `apps/server/src/notifications/notifications.service.ts`**:

```typescript
import { Injectable } from '@nestjs/common';
import * as admin from 'firebase-admin';
import apn from 'apn';

@Injectable()
export class NotificationsService {
  private fcm: admin.messaging.Messaging;
  private apnProvider: apn.Provider;

  constructor() {
    // Initialize Firebase
    admin.initializeApp({
      credential: admin.credential.cert({
        projectId: process.env.FIREBASE_PROJECT_ID,
        clientEmail: process.env.FIREBASE_CLIENT_EMAIL,
        privateKey: process.env.FIREBASE_PRIVATE_KEY?.replace(/\\n/g, '\n'),
      }),
    });
    this.fcm = admin.messaging();

    // Initialize APNS
    this.apnProvider = new apn.Provider({
      token: {
        key: process.env.APNS_KEY_PATH,
        keyId: process.env.APNS_KEY_ID,
        teamId: process.env.APNS_TEAM_ID,
      },
      production: process.env.NODE_ENV === 'production',
    });
  }

  async sendPushNotification(userId: string, notification: {
    title: string;
    body: string;
    data?: Record<string, string>;
  }) {
    // Get user devices
    const devices = await this.getUserDevices(userId);

    for (const device of devices) {
      if (device.pushProvider === 'fcm') {
        await this.sendFCM(device.pushToken, notification);
      } else if (device.pushProvider === 'apns') {
        await this.sendAPNS(device.pushToken, notification);
      }
    }
  }

  private async sendFCM(token: string, notification: any) {
    try {
      await this.fcm.send({
        token,
        notification: {
          title: notification.title,
          body: notification.body,
        },
        data: notification.data,
      });
    } catch (error) {
      console.error('FCM error:', error);
    }
  }

  private async sendAPNS(token: string, notification: any) {
    const note = new apn.Notification();
    note.alert = {
      title: notification.title,
      body: notification.body,
    };
    note.payload = notification.data || {};
    note.topic = process.env.APNS_BUNDLE_ID;

    try {
      await this.apnProvider.send(note, token);
    } catch (error) {
      console.error('APNS error:', error);
    }
  }

  private async getUserDevices(userId: string) {
    // Query user devices from Prisma
    const { PrismaService } = require('../prisma/prisma.service');
    const prisma = new PrismaService();
    
    return prisma.userDevice.findMany({
      where: {
        userId,
        isActive: true,
        pushToken: { not: null },
      },
    });
  }
}
```

---

### Task 2: Message Search (Elasticsearch)

**Install Dependencies**:

```bash
pnpm add @elastic/elasticsearch
```

**Create `apps/server/src/search/search.service.ts`**:

```typescript
import { Injectable } from '@nestjs/common';
import { Client } from '@elastic/elasticsearch';

@Injectable()
export class SearchService {
  private client: Client;

  constructor() {
    this.client = new Client({
      node: process.env.ELASTICSEARCH_URL || 'http://localhost:9200',
    });
  }

  async indexMessage(message: any) {
    await this.client.index({
      index: 'messages',
      id: message.id,
      document: {
        text: message.text,
        channelId: message.channelId,
        userId: message.userId,
        createdAt: message.createdAt,
      },
    });
  }

  async searchMessages(query: string, channelId?: string) {
    const body: any = {
      query: {
        bool: {
          must: [
            {
              multi_match: {
                query,
                fields: ['text'],
                fuzziness: 'AUTO',
              },
            },
          ],
        },
      },
    };

    if (channelId) {
      body.query.bool.filter = [{ term: { channelId } }];
    }

    const result = await this.client.search({
      index: 'messages',
      body,
    });

    return result.hits.hits.map((hit: any) => hit._source);
  }
}
```

---

### Task 3: Message Reactions

**Update Prisma Schema** (`apps/server/prisma/schema.prisma`):

```prisma
model MessageReaction {
  id        String   @id @default(uuid())
  messageId String   @map("message_id")
  userId    String   @map("user_id")
  emoji     String
  createdAt DateTime @default(now()) @map("created_at")

  message Message @relation(fields: [messageId], references: [id], onDelete: Cascade)
  user    User    @relation("UserReactions", fields: [userId], references: [id], onDelete: Cascade)

  @@unique([messageId, userId, emoji])
  @@index([messageId])
  @@map("message_reactions")
}
```

**Update Message model** to add reactions relation:

```prisma
model Message {
  // ... existing fields ...
  reactions MessageReaction[]
}
```

**Update User model** to add reactions relation:

```prisma
model User {
  // ... existing fields ...
  reactions MessageReaction[] @relation("UserReactions")
}
```

**Generate Migration**:

```bash
cd apps/server
npx prisma migrate dev --name add_message_reactions
```

**Create `apps/server/src/reactions/reactions.service.ts`**:

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

@Injectable()
export class ReactionsService {
  constructor(private prisma: PrismaService) {}

  async addReaction(messageId: string, userId: string, emoji: string) {
    // Use upsert to handle duplicate reactions
    return this.prisma.messageReaction.upsert({
      where: {
        messageId_userId_emoji: {
          messageId,
          userId,
          emoji,
        },
      },
      create: {
        messageId,
        userId,
        emoji,
      },
      update: {}, // No update needed if exists
    });
  }

  async removeReaction(messageId: string, userId: string, emoji: string) {
    await this.prisma.messageReaction.delete({
      where: {
        messageId_userId_emoji: {
          messageId,
          userId,
          emoji,
        },
      },
    });
  }

  async getReactions(messageId: string) {
    return this.prisma.messageReaction.findMany({
      where: { messageId },
      include: {
        user: {
          select: {
            id: true,
            username: true,
            displayName: true,
            avatarUrl: true,
          },
        },
      },
    });
  }

  async getReactionCounts(messageId: string) {
    const reactions = await this.prisma.messageReaction.groupBy({
      by: ['emoji'],
      where: { messageId },
      _count: { emoji: true },
    });

    return reactions.map(r => ({
      emoji: r.emoji,
      count: r._count.emoji,
    }));
  }
}
```

---

### Task 4: Moderation System

**First, add Prisma models** (`apps/server/prisma/schema.prisma`):

```prisma
enum ModerationStatus {
  PENDING
  APPROVED
  REJECTED
}

model ModerationQueue {
  id         String           @id @default(uuid())
  messageId  String           @map("message_id")
  userId     String           @map("user_id")
  reason     String
  status     ModerationStatus @default(PENDING)
  reviewedBy String?          @map("reviewed_by")
  reviewedAt DateTime?        @map("reviewed_at")
  createdAt  DateTime         @default(now()) @map("created_at")

  message  Message @relation(fields: [messageId], references: [id], onDelete: Cascade)
  user     User    @relation("ModerationQueueUser", fields: [userId], references: [id])
  reviewer User?   @relation("ModerationQueueReviewer", fields: [reviewedBy], references: [id])

  @@index([messageId])
  @@index([status])
  @@map("moderation_queue")
}

model BlockList {
  id        String   @id @default(uuid())
  pattern   String
  reason    String?
  createdBy String   @map("created_by")
  createdAt DateTime @default(now()) @map("created_at")

  creator User @relation("BlockListCreator", fields: [createdBy], references: [id])

  @@index([pattern])
  @@map("block_list")
}
```

**Update Message and User models**:

```prisma
model Message {
  // ... existing fields ...
  moderationQueue ModerationQueue[]
}

model User {
  // ... existing fields ...
  moderationQueueAsUser     ModerationQueue[] @relation("ModerationQueueUser")
  moderationQueueAsReviewer ModerationQueue[] @relation("ModerationQueueReviewer")
  blockListEntries          BlockList[]       @relation("BlockListCreator")
}
```

**Generate Migration**:

```bash
cd apps/server
npx prisma migrate dev --name add_moderation_system
```

**Create `apps/server/src/moderation/moderation.service.ts`**:

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { ModerationStatus } from '@prisma/client';

@Injectable()
export class ModerationService {
  private profanityWords = ['bad', 'word']; // Load from config

  constructor(private prisma: PrismaService) {}

  async checkMessage(message: any): Promise<boolean> {
    // Check profanity
    if (this.containsProfanity(message.text)) {
      await this.queueForReview(message, 'profanity');
      return false;
    }

    // Check block list
    const blocked = await this.isBlocked(message.text);
    if (blocked) {
      await this.queueForReview(message, 'blocked_content');
      return false;
    }

    return true;
  }

  private containsProfanity(text: string): boolean {
    const lowerText = text.toLowerCase();
    return this.profanityWords.some((word) => lowerText.includes(word));
  }

  private async isBlocked(text: string): boolean {
    const blockList = await this.prisma.blockList.findMany();
    return blockList.some((item) => text.includes(item.pattern));
  }

  private async queueForReview(message: any, reason: string) {
    await this.prisma.moderationQueue.create({
      data: {
        messageId: message.id,
        userId: message.userId,
        reason,
        status: ModerationStatus.PENDING,
      },
    });
  }

  async approveMessage(queueId: string, reviewerId: string) {
    await this.prisma.moderationQueue.update({
      where: { id: queueId },
      data: {
        status: ModerationStatus.APPROVED,
        reviewedBy: reviewerId,
        reviewedAt: new Date(),
      },
    });
  }

  async rejectMessage(queueId: string, reviewerId: string) {
    await this.prisma.moderationQueue.update({
      where: { id: queueId },
      data: {
        status: ModerationStatus.REJECTED,
        reviewedBy: reviewerId,
        reviewedAt: new Date(),
      },
    });
  }

  async getPendingQueue(limit: number = 50) {
    return this.prisma.moderationQueue.findMany({
      where: { status: ModerationStatus.PENDING },
      include: {
        message: true,
        user: {
          select: {
            id: true,
            username: true,
            displayName: true,
          },
        },
      },
      orderBy: { createdAt: 'asc' },
      take: limit,
    });
  }

  async addToBlockList(pattern: string, reason: string, createdBy: string) {
    return this.prisma.blockList.create({
      data: {
        pattern,
        reason,
        createdBy,
      },
    });
  }
}
```

---

## Week 3-4: Testing & Documentation

### Task 1: Unit Tests

**Create `apps/server/test/unit/channels.service.spec.ts`**:

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { ChannelsService } from '../../src/channels/channels.service';
import { PrismaService } from '../../src/prisma/prisma.service';
import { ChannelType, MemberRole } from '@prisma/client';

describe('ChannelsService', () => {
  let service: ChannelsService;
  let prisma: PrismaService;

  const mockPrismaService = {
    channel: {
      create: jest.fn(),
      findMany: jest.fn(),
      findUnique: jest.fn(),
      update: jest.fn(),
    },
    channelMember: {
      create: jest.fn(),
      findUnique: jest.fn(),
    },
    $transaction: jest.fn((callback) => callback(mockPrismaService)),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        ChannelsService,
        {
          provide: PrismaService,
          useValue: mockPrismaService,
        },
      ],
    }).compile();

    service = module.get<ChannelsService>(ChannelsService);
    prisma = module.get<PrismaService>(PrismaService);
  });

  it('should create a channel', async () => {
    const channelData = {
      type: ChannelType.GROUP,
      name: 'Test',
      isDistinct: false,
      metadata: {},
      createdBy: 'user-1',
    };

    mockPrismaService.channel.create.mockResolvedValue({
      id: '123',
      ...channelData,
      createdAt: new Date(),
      updatedAt: new Date(),
    });

    const result = await service.create('user-1', {
      type: ChannelType.GROUP,
      name: 'Test',
    });

    expect(result.id).toBe('123');
    expect(mockPrismaService.channel.create).toHaveBeenCalled();
  });
});
```

---

### Task 2: Integration Tests

**Create `apps/server/test/integration/messages.e2e-spec.ts`**:

```typescript
import { Test } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../../src/app.module';

describe('Messages (e2e)', () => {
  let app: INestApplication;
  let authToken: string;
  let channelId: string;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleRef.createNestApplication();
    await app.init();

    // Setup: Login and create channel
    const loginRes = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ identifier: 'test@example.com', password: 'password' });
    authToken = loginRes.body.accessToken;

    const channelRes = await request(app.getHttpServer())
      .post('/channels')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ type: 'group', name: 'Test' });
    channelId = channelRes.body.id;
  });

  it('should send a message', () => {
    return request(app.getHttpServer())
      .post('/messages')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ channelId, text: 'Hello world' })
      .expect(201)
      .expect((res) => {
        expect(res.body.text).toBe('Hello world');
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
```

---

### Task 3: Load Testing (k6)

**Create `test/load/message-sending.js`**:

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 20 },
    { duration: '1m', target: 50 },
    { duration: '30s', target: 0 },
  ],
};

const BASE_URL = 'http://localhost:3000';
const TOKEN = __ENV.AUTH_TOKEN;

export default function () {
  const payload = JSON.stringify({
    channelId: 'test-channel-id',
    text: 'Load test message',
  });

  const params = {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${TOKEN}`,
    },
  };

  const res = http.post(`${BASE_URL}/messages`, payload, params);

  check(res, {
    'status is 201': (r) => r.status === 201,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  sleep(1);
}
```

**Run Load Test**:

```bash
k6 run test/load/message-sending.js
```

---

### Task 4: API Documentation

**Update `apps/server/src/main.ts`**:

```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Swagger setup
  const config = new DocumentBuilder()
    .setTitle('Chat SDK API')
    .setDescription('Real-time chat API documentation')
    .setVersion('1.0')
    .addBearerAuth()
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api/docs', app, document);

  await app.listen(3000);
}
bootstrap();
```

---

### Task 5: SDK Documentation

**Create `packages/sdk-core/README.md`**:

```markdown
# @chat-sdk/core

TypeScript SDK for Chat SDK.

## Installation

```bash
npm install @chat-sdk/core
```

## Quick Start

```typescript
import { ChatClient } from '@chat-sdk/core';

const client = new ChatClient({
  apiUrl: 'https://api.example.com',
  token: 'your-jwt-token',
  autoConnect: true,
});

// Listen for events
client.on('connected', () => {
  console.log('Connected!');
});

client.on('message.new', (message) => {
  console.log('New message:', message);
});

// Send a message
await client.messages.send('channel-id', {
  text: 'Hello world!',
});
```

## API Reference

See [API.md](./API.md) for full documentation.
```

---

## Week 4: Beta Launch

### Task 1: Production Deployment

**Create `docker/Dockerfile.production`**:

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json pnpm-lock.yaml ./
RUN npm install -g pnpm@8.15.0
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build

FROM node:20-alpine

WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

EXPOSE 3000
CMD ["node", "dist/main"]
```

**Build and Deploy**:

```bash
docker build -f docker/Dockerfile.production -t chat-sdk:v0.1.0 .
docker tag chat-sdk:v0.1.0 registry.example.com/chat-sdk:v0.1.0
docker push registry.example.com/chat-sdk:v0.1.0
```

---

### Task 2: Monitoring Setup

**Create `docker/prometheus.yml`**:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'chat-api'
    static_configs:
      - targets: ['localhost:3000']
```

**Create `docker/docker-compose.monitoring.yml`**:

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  prometheus_data:
  grafana_data:
```

---

### Task 3: Error Tracking (Sentry)

**Install Sentry**:

```bash
pnpm add @sentry/node
```

**Configure in `apps/server/src/main.ts`**:

```typescript
import * as Sentry from '@sentry/node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 1.0,
});
```

---

## Deliverables Checklist

- [x] Push notifications (FCM, APNS)
- [x] Message search (Elasticsearch)
- [x] Message reactions
- [x] Moderation system
- [x] Unit tests (80%+ coverage)
- [x] Integration tests
- [x] Load tests (k6)
- [x] API documentation (Swagger)
- [x] SDK documentation
- [x] Production deployment
- [x] Monitoring (Prometheus, Grafana)
- [x] Error tracking (Sentry)
- [x] Beta launch ready

**Next**: [MONTH_05_ENHANCED_MESSAGING.md](./MONTH_05_ENHANCED_MESSAGING.md)
