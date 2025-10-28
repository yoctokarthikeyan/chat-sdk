# Month 2: Core Messaging Features - Implementation Guide

**File**: `MONTH_02_MESSAGING.md`  
**Purpose**: Implement channels, messages, file uploads, and real-time WebSocket layer  
**Prerequisites**: Complete [MONTH_01_FOUNDATION.md](./MONTH_01_FOUNDATION.md)

---

## Week 1-2: Messaging Backend

### Prisma Schema Update

**Update `apps/server/prisma/schema.prisma`** (add to existing schema):

```prisma
enum ChannelType {
  DIRECT
  GROUP
  PUBLIC
}

enum MemberRole {
  OWNER
  ADMIN
  MEMBER
}

enum MessageType {
  TEXT
  SYSTEM
  DELETED
}

enum MessageStatus {
  SENT
  PENDING
  FAILED
}

model Channel {
  id          String      @id @default(uuid())
  type        ChannelType
  name        String?
  description String?
  isDistinct  Boolean     @default(false) @map("is_distinct")
  isFrozen    Boolean     @default(false) @map("is_frozen")
  slowMode    Int         @default(0) @map("slow_mode")
  metadata    Json        @default("{}")
  createdBy   String?     @map("created_by")
  createdAt   DateTime    @default(now()) @map("created_at")
  updatedAt   DateTime    @updatedAt @map("updated_at")
  truncatedAt DateTime?   @map("truncated_at")

  creator User?           @relation("CreatedChannels", fields: [createdBy], references: [id])
  members ChannelMember[]
  messages Message[]

  @@index([type])
  @@map("channels")
}

model ChannelMember {
  id             String    @id @default(uuid())
  channelId      String    @map("channel_id")
  userId         String    @map("user_id")
  role           MemberRole @default(MEMBER)
  joinedAt       DateTime  @default(now()) @map("joined_at")
  lastReadAt     DateTime? @map("last_read_at")
  isBanned       Boolean   @default(false) @map("is_banned")
  isShadowBanned Boolean   @default(false) @map("is_shadow_banned")
  isHidden       Boolean   @default(false) @map("is_hidden")

  channel Channel @relation(fields: [channelId], references: [id], onDelete: Cascade)
  user    User    @relation("ChannelMemberships", fields: [userId], references: [id], onDelete: Cascade)

  @@unique([channelId, userId])
  @@index([channelId])
  @@index([userId])
  @@map("channel_members")
}

model Message {
  id              String        @id @default(uuid())
  channelId       String        @map("channel_id")
  userId          String        @map("user_id")
  parentMessageId String?       @map("parent_message_id")
  text            String?
  type            MessageType   @default(TEXT)
  status          MessageStatus @default(SENT)
  isSilent        Boolean       @default(false) @map("is_silent")
  isPinned        Boolean       @default(false) @map("is_pinned")
  metadata        Json          @default("{}")
  mentions        Json          @default("[]")
  quotedMessageId String?       @map("quoted_message_id")
  isDeleted       Boolean       @default(false) @map("is_deleted")
  createdAt       DateTime      @default(now()) @map("created_at")
  updatedAt       DateTime      @updatedAt @map("updated_at")

  channel         Channel             @relation(fields: [channelId], references: [id], onDelete: Cascade)
  user            User                @relation("SentMessages", fields: [userId], references: [id])
  parentMessage   Message?            @relation("Replies", fields: [parentMessageId], references: [id])
  replies         Message[]           @relation("Replies")
  quotedMessage   Message?            @relation("QuotedMessages", fields: [quotedMessageId], references: [id])
  quotedBy        Message[]           @relation("QuotedMessages")
  attachments     MessageAttachment[]

  @@index([channelId, createdAt(sort: Desc)])
  @@index([userId])
  @@map("messages")
}

model MessageAttachment {
  id             String   @id @default(uuid())
  messageId      String   @map("message_id")
  attachmentType String   @map("attachment_type")
  url            String
  fileName       String?  @map("file_name")
  fileSize       BigInt?  @map("file_size")
  mimeType       String?  @map("mime_type")
  createdAt      DateTime @default(now()) @map("created_at")

  message Message @relation(fields: [messageId], references: [id], onDelete: Cascade)

  @@index([messageId])
  @@map("message_attachments")
}
```

**Also update User model** to add relations:

```prisma
model User {
  // ... existing fields ...
  
  // Add these relations
  createdChannels  Channel[]       @relation("CreatedChannels")
  channelMembers   ChannelMember[] @relation("ChannelMemberships")
  sentMessages     Message[]       @relation("SentMessages")
}
```

**Generate Migration**:

```bash
cd apps/server
npx prisma migrate dev --name messaging_schema
```

---

### Channel Types & DTOs

**Create `apps/server/src/channels/types/channel.types.ts`**:

```typescript
import { Channel, ChannelType, ChannelMember, MemberRole } from '@prisma/client';

// Export Prisma types
export { Channel, ChannelType, ChannelMember, MemberRole };

// Create channel DTO
export class CreateChannelDto {
  type: ChannelType;
  name?: string;
  description?: string;
  isDistinct?: boolean;
  members?: string[];
  metadata?: Record<string, any>;
}

// Update channel DTO
export class UpdateChannelDto {
  name?: string;
  description?: string;
  metadata?: Record<string, any>;
}
```

**Create `apps/server/src/channels/channels.service.ts`**:

```typescript
import { Injectable, NotFoundException, ForbiddenException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { ChannelType, MemberRole } from '@prisma/client';
import { CreateChannelDto } from './types/channel.types';

@Injectable()
export class ChannelsService {
  constructor(private prisma: PrismaService) {}

  async create(userId: string, dto: CreateChannelDto) {
    // Check for distinct channel
    if (dto.isDistinct && dto.members) {
      const existing = await this.findDistinctChannel([userId, ...dto.members]);
      if (existing) return existing;
    }

    // Create channel with members in a transaction
    const channel = await this.prisma.$transaction(async (tx) => {
      const newChannel = await tx.channel.create({
        data: {
          type: dto.type,
          name: dto.name,
          description: dto.description,
          isDistinct: dto.isDistinct || false,
          metadata: dto.metadata || {},
          createdBy: userId,
        },
      });

      // Add creator as owner
      await tx.channelMember.create({
        data: {
          channelId: newChannel.id,
          userId,
          role: MemberRole.OWNER,
        },
      });

      // Add other members
      if (dto.members) {
        await tx.channelMember.createMany({
          data: dto.members.map(memberId => ({
            channelId: newChannel.id,
            userId: memberId,
            role: MemberRole.MEMBER,
          })),
        });
      }

      return newChannel;
    });

    return channel;
  }

  async findDistinctChannel(memberIds: string[]) {
    // Find distinct channels with exact member match
    const channels = await this.prisma.channel.findMany({
      where: {
        isDistinct: true,
      },
      include: {
        members: {
          select: { userId: true },
        },
      },
    });

    // Filter to find exact match
    return channels.find(channel => {
      const channelMemberIds = channel.members.map(m => m.userId).sort();
      const searchMemberIds = memberIds.sort();
      return (
        channelMemberIds.length === searchMemberIds.length &&
        channelMemberIds.every((id, index) => id === searchMemberIds[index])
      );
    });
  }

  async addMember(channelId: string, userId: string, role: MemberRole = MemberRole.MEMBER) {
    return this.prisma.channelMember.create({
      data: {
        channelId,
        userId,
        role,
      },
    });
  }

  async freeze(channelId: string, userId: string) {
    await this.checkPermission(channelId, userId, [MemberRole.OWNER, MemberRole.ADMIN]);
    return this.prisma.channel.update({
      where: { id: channelId },
      data: { isFrozen: true },
    });
  }

  async truncate(channelId: string, userId: string) {
    await this.checkPermission(channelId, userId, [MemberRole.OWNER, MemberRole.ADMIN]);
    return this.prisma.channel.update({
      where: { id: channelId },
      data: { truncatedAt: new Date() },
    });
  }

  async getUserChannels(userId: string) {
    return this.prisma.channel.findMany({
      where: {
        members: {
          some: {
            userId,
            isHidden: false,
          },
        },
      },
      include: {
        members: {
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
        },
      },
      orderBy: {
        updatedAt: 'desc',
      },
    });
  }

  private async checkPermission(channelId: string, userId: string, allowedRoles: MemberRole[]) {
    const member = await this.prisma.channelMember.findUnique({
      where: {
        channelId_userId: {
          channelId,
          userId,
        },
      },
    });

    if (!member || !allowedRoles.includes(member.role)) {
      throw new ForbiddenException('Insufficient permissions');
    }
  }
}
```

---

### Message Service

**Create `apps/server/src/messages/types/message.types.ts`**:

```typescript
import { Message, MessageType, MessageStatus, MessageAttachment } from '@prisma/client';

// Export Prisma types
export { Message, MessageType, MessageStatus, MessageAttachment };

// Send message DTO
export class SendMessageDto {
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

// Update message DTO
export class UpdateMessageDto {
  text?: string;
  metadata?: Record<string, any>;
}
```

**Create `apps/server/src/messages/messages.service.ts`**:

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { MessageType } from '@prisma/client';
import { SendMessageDto, UpdateMessageDto } from './types/message.types';

@Injectable()
export class MessagesService {
  constructor(private prisma: PrismaService) {}

  async send(userId: string, dto: SendMessageDto) {
    // Create message with attachments in a transaction
    const message = await this.prisma.message.create({
      data: {
        channelId: dto.channelId,
        userId,
        text: dto.text,
        type: dto.type || MessageType.TEXT,
        isSilent: dto.silent || false,
        metadata: dto.metadata || {},
        mentions: dto.mentions || [],
        quotedMessageId: dto.quotedMessageId,
        attachments: dto.attachments
          ? {
              createMany: {
                data: dto.attachments,
              },
            }
          : undefined,
      },
      include: {
        attachments: true,
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

    return message;
  }

  async getHistory(channelId: string, limit: number = 50, before?: string) {
    const where: any = {
      channelId,
      isDeleted: false,
    };

    if (before) {
      const beforeMessage = await this.prisma.message.findUnique({
        where: { id: before },
        select: { createdAt: true },
      });
      
      if (beforeMessage) {
        where.createdAt = { lt: beforeMessage.createdAt };
      }
    }

    return this.prisma.message.findMany({
      where,
      include: {
        user: {
          select: {
            id: true,
            username: true,
            displayName: true,
            avatarUrl: true,
          },
        },
        attachments: true,
        quotedMessage: {
          include: {
            user: {
              select: {
                id: true,
                username: true,
                displayName: true,
              },
            },
          },
        },
      },
      orderBy: {
        createdAt: 'desc',
      },
      take: limit,
    });
  }

  async update(messageId: string, userId: string, dto: UpdateMessageDto) {
    // Verify ownership
    const message = await this.prisma.message.findUnique({
      where: { id: messageId },
    });

    if (!message || message.userId !== userId) {
      throw new Error('Message not found or unauthorized');
    }

    return this.prisma.message.update({
      where: { id: messageId },
      data: {
        text: dto.text,
        metadata: dto.metadata,
        updatedAt: new Date(),
      },
      include: {
        attachments: true,
      },
    });
  }

  async delete(messageId: string, userId: string, hard: boolean = false) {
    // Verify ownership
    const message = await this.prisma.message.findUnique({
      where: { id: messageId },
    });

    if (!message || message.userId !== userId) {
      throw new Error('Message not found or unauthorized');
    }

    if (hard) {
      // Hard delete
      return this.prisma.message.delete({
        where: { id: messageId },
      });
    } else {
      // Soft delete
      return this.prisma.message.update({
        where: { id: messageId },
        data: {
          isDeleted: true,
          text: null,
        },
      });
    }
  }

  async search(channelId: string, query: string, limit: number = 20) {
    return this.prisma.message.findMany({
      where: {
        channelId,
        isDeleted: false,
        text: {
          contains: query,
          mode: 'insensitive',
        },
      },
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
      orderBy: {
        createdAt: 'desc',
      },
      take: limit,
    });
  }
}
```

---

## Week 3-4: Real-time WebSocket Layer

### WebSocket Gateway

**Create `apps/server/src/websocket/websocket.gateway.ts`**:

```typescript
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  OnGatewayConnection,
  OnGatewayDisconnect,
  ConnectedSocket,
  MessageBody,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { UseGuards } from '@nestjs/common';
import { WsJwtGuard } from '../auth/guards/ws-jwt.guard';
import { MessagesService } from '../messages/messages.service';

@WebSocketGateway({
  cors: { origin: '*' },
  namespace: '/chat',
})
export class WebsocketGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  private userSockets = new Map<string, Set<string>>();

  constructor(private messagesService: MessagesService) {}

  async handleConnection(client: Socket) {
    const userId = client.handshake.auth.userId;
    if (!userId) {
      client.disconnect();
      return;
    }

    if (!this.userSockets.has(userId)) {
      this.userSockets.set(userId, new Set());
    }
    this.userSockets.get(userId).add(client.id);

    // Broadcast presence
    this.server.emit('user.presence', { userId, status: 'online' });
  }

  async handleDisconnect(client: Socket) {
    const userId = client.handshake.auth.userId;
    if (userId && this.userSockets.has(userId)) {
      this.userSockets.get(userId).delete(client.id);
      if (this.userSockets.get(userId).size === 0) {
        this.userSockets.delete(userId);
        this.server.emit('user.presence', { userId, status: 'offline' });
      }
    }
  }

  @UseGuards(WsJwtGuard)
  @SubscribeMessage('message.send')
  async handleMessage(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: any,
  ) {
    const userId = client.handshake.auth.userId;
    const message = await this.messagesService.send(userId, data);

    // Broadcast to channel
    this.server.to(`channel:${data.channelId}`).emit('message.new', message);

    return { success: true, messageId: message.id };
  }

  @SubscribeMessage('channel.join')
  async handleJoinChannel(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { channelId: string },
  ) {
    client.join(`channel:${data.channelId}`);
    return { success: true };
  }

  @SubscribeMessage('typing.start')
  async handleTypingStart(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { channelId: string },
  ) {
    const userId = client.handshake.auth.userId;
    client.to(`channel:${data.channelId}`).emit('typing.start', { userId });
  }

  @SubscribeMessage('typing.stop')
  async handleTypingStop(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { channelId: string },
  ) {
    const userId = client.handshake.auth.userId;
    client.to(`channel:${data.channelId}`).emit('typing.stop', { userId });
  }
}
```

---

### File Upload (S3)

**Install AWS SDK**:

```bash
pnpm add @aws-sdk/client-s3 @aws-sdk/s3-request-presigner multer
pnpm add -D @types/multer
```

**Create `apps/server/src/upload/upload.service.ts`**:

```typescript
import { Injectable } from '@nestjs/common';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class UploadService {
  private s3Client: S3Client;
  private bucket = process.env.S3_BUCKET;

  constructor() {
    this.s3Client = new S3Client({
      region: process.env.AWS_REGION,
      credentials: {
        accessKeyId: process.env.AWS_ACCESS_KEY_ID,
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
      },
    });
  }

  async uploadFile(file: Express.Multer.File, userId: string) {
    const key = `uploads/${userId}/${uuidv4()}-${file.originalname}`;

    const command = new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      Body: file.buffer,
      ContentType: file.mimetype,
    });

    await this.s3Client.send(command);

    return {
      url: `https://${this.bucket}.s3.amazonaws.com/${key}`,
      fileName: file.originalname,
      fileSize: file.size,
      mimeType: file.mimetype,
    };
  }

  async getSignedUrl(key: string, expiresIn: number = 3600) {
    const command = new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
    });

    return getSignedUrl(this.s3Client, command, { expiresIn });
  }
}
```

---

## API Endpoints

### Channels

```
POST   /api/v1/channels              Create channel
GET    /api/v1/channels              List channels
GET    /api/v1/channels/:id          Get channel
PATCH  /api/v1/channels/:id          Update channel
DELETE /api/v1/channels/:id          Delete channel
POST   /api/v1/channels/:id/members  Add member
DELETE /api/v1/channels/:id/members/:userId  Remove member
POST   /api/v1/channels/:id/freeze   Freeze channel
POST   /api/v1/channels/:id/truncate Truncate channel
```

### Messages

```
POST   /api/v1/messages              Send message
GET    /api/v1/channels/:id/messages Get message history
PATCH  /api/v1/messages/:id          Update message
DELETE /api/v1/messages/:id          Delete message
POST   /api/v1/messages/:id/pin      Pin message
```

### Upload

```
POST   /api/v1/upload                Upload file
GET    /api/v1/upload/signed-url     Get signed URL
```

---

## Testing

**Create `apps/server/test/channels.e2e-spec.ts`**:

```typescript
import { Test } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('Channels (e2e)', () => {
  let app: INestApplication;
  let authToken: string;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleRef.createNestApplication();
    await app.init();

    // Login to get token
    const response = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ identifier: 'test@example.com', password: 'password' });
    
    authToken = response.body.accessToken;
  });

  it('/channels (POST)', () => {
    return request(app.getHttpServer())
      .post('/channels')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ type: 'group', name: 'Test Channel' })
      .expect(201);
  });

  afterAll(async () => {
    await app.close();
  });
});
```

---

## Deliverables

- [x] Channel CRUD APIs
- [x] Message sending/receiving
- [x] WebSocket real-time messaging
- [x] File upload to S3
- [x] Typing indicators
- [x] User presence
- [x] E2E tests

**Next**: [MONTH_03_SDK.md](./MONTH_03_SDK.md)
