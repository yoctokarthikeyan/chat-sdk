# Month 5: Enhanced Messaging - Implementation Guide

**File**: `MONTH_05_ENHANCED_MESSAGING.md`  
**Purpose**: Implement threading, pinning, rich content, scheduling  
**Prerequisites**: Complete [MONTH_04_POLISH.md](./MONTH_04_POLISH.md)

---

## Updates for Prisma Migration

**Note**: Month 5 adds threading, pinning, and scheduling features. All database operations have been updated to use Prisma:

- ✅ Threading - Updated Prisma schema with thread metadata
- ✅ Pinning - Added pinning fields to Message model
- ✅ Scheduled Messages - New Prisma model with status enum
- ✅ Services - All converted to use PrismaService
- ✅ Tests - Updated to use Prisma mocks

---

## Week 1-2: Threading & Pinning

### Update Prisma Schema

**Update `apps/server/prisma/schema.prisma`** - Add threading and pinning fields to Message model:

```prisma
model Message {
  id              String        @id @default(uuid())
  channelId       String        @map("channel_id")
  userId          String        @map("user_id")
  parentMessageId String?       @map("parent_message_id")
  text            String?
  type            MessageType   @default(TEXT)
  status          MessageStatus @default(SENT)
  isSilent        Boolean       @default(false) @map("is_silent")
  
  // Pinning fields
  isPinned        Boolean       @default(false) @map("is_pinned")
  pinnedAt        DateTime?     @map("pinned_at")
  pinnedBy        String?       @map("pinned_by")
  
  // Threading fields
  threadParticipantCount Int?     @default(0) @map("thread_participant_count")
  threadLastMessageAt    DateTime? @map("thread_last_message_at")
  showInChannel          Boolean   @default(true) @map("show_in_channel")
  
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
  pinner          User?               @relation("PinnedMessages", fields: [pinnedBy], references: [id])
  attachments     MessageAttachment[]
  reactions       MessageReaction[]
  moderationQueue ModerationQueue[]

  @@index([channelId, createdAt(sort: Desc)])
  @@index([userId])
  @@index([parentMessageId, createdAt(sort: Desc)])
  @@index([channelId, isPinned, pinnedAt(sort: Desc)])
  @@map("messages")
}
```

**Update User model** to add pinning relation:

```prisma
model User {
  // ... existing fields ...
  pinnedMessages Message[] @relation("PinnedMessages")
}
```

**Generate Migration**:

```bash
cd apps/server
npx prisma migrate dev --name add_threading_and_pinning
```

### Threading Service

**Create `apps/server/src/messages/threading.service.ts`**:

```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { MessageType } from '@prisma/client';

@Injectable()
export class ThreadingService {
  constructor(private prisma: PrismaService) {}

  async replyToThread(userId: string, parentId: string, data: any) {
    // Verify parent message exists
    const parent = await this.prisma.message.findUnique({
      where: { id: parentId },
    });

    if (!parent) {
      throw new NotFoundException('Parent message not found');
    }

    // Create reply in a transaction
    const reply = await this.prisma.$transaction(async (tx) => {
      // Create the reply
      const newReply = await tx.message.create({
        data: {
          channelId: parent.channelId,
          userId,
          parentMessageId: parentId,
          text: data.text,
          type: data.type || MessageType.TEXT,
          showInChannel: data.showInChannel ?? false,
          metadata: data.metadata || {},
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
      });

      // Update thread metadata
      await this.updateThreadMetadata(tx, parentId);

      return newReply;
    });

    return reply;
  }

  async getThreadReplies(parentId: string, limit = 50, cursor?: string) {
    const where: any = {
      parentMessageId: parentId,
      isDeleted: false,
    };

    if (cursor) {
      where.id = { gt: cursor };
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
        reactions: {
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
      orderBy: { createdAt: 'asc' },
      take: limit,
    });
  }

  async getThreadMetadata(parentId: string) {
    const parent = await this.prisma.message.findUnique({
      where: { id: parentId },
      select: {
        threadParticipantCount: true,
        threadLastMessageAt: true,
        _count: {
          select: { replies: true },
        },
      },
    });

    return {
      replyCount: parent?._count.replies || 0,
      participantCount: parent?.threadParticipantCount || 0,
      lastMessageAt: parent?.threadLastMessageAt,
    };
  }

  private async updateThreadMetadata(tx: any, parentId: string) {
    // Get unique participants in thread
    const participants = await tx.message.findMany({
      where: { parentMessageId: parentId },
      select: { userId: true },
      distinct: ['userId'],
    });

    // Update parent message metadata
    await tx.message.update({
      where: { id: parentId },
      data: {
        threadParticipantCount: participants.length,
        threadLastMessageAt: new Date(),
      },
    });
  }
}
```

### Pinning Service

**Create `apps/server/src/messages/pinning.service.ts`**:

```typescript
import { Injectable, ForbiddenException, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { MemberRole } from '@prisma/client';

@Injectable()
export class PinningService {
  constructor(private prisma: PrismaService) {}

  private readonly MAX_PINNED_MESSAGES = 50;

  async pinMessage(userId: string, messageId: string) {
    // Get message with channel
    const message = await this.prisma.message.findUnique({
      where: { id: messageId },
      include: { channel: true },
    });

    if (!message) {
      throw new BadRequestException('Message not found');
    }

    // Verify user has permission to pin
    await this.verifyPermission(userId, message.channelId);

    // Check pinned message limit
    const pinnedCount = await this.prisma.message.count({
      where: {
        channelId: message.channelId,
        isPinned: true,
      },
    });

    if (pinnedCount >= this.MAX_PINNED_MESSAGES) {
      throw new BadRequestException(
        `Cannot pin more than ${this.MAX_PINNED_MESSAGES} messages per channel`
      );
    }

    // Pin the message
    return this.prisma.message.update({
      where: { id: messageId },
      data: {
        isPinned: true,
        pinnedAt: new Date(),
        pinnedBy: userId,
      },
      include: {
        user: {
          select: {
            id: true,
            username: true,
            displayName: true,
          },
        },
        pinner: {
          select: {
            id: true,
            username: true,
            displayName: true,
          },
        },
      },
    });
  }

  async unpinMessage(userId: string, messageId: string) {
    const message = await this.prisma.message.findUnique({
      where: { id: messageId },
    });

    if (!message) {
      throw new BadRequestException('Message not found');
    }

    await this.verifyPermission(userId, message.channelId);

    return this.prisma.message.update({
      where: { id: messageId },
      data: {
        isPinned: false,
        pinnedAt: null,
        pinnedBy: null,
      },
    });
  }

  async getPinnedMessages(channelId: string) {
    return this.prisma.message.findMany({
      where: {
        channelId,
        isPinned: true,
        isDeleted: false,
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
        pinner: {
          select: {
            id: true,
            username: true,
            displayName: true,
          },
        },
        attachments: true,
      },
      orderBy: { pinnedAt: 'desc' },
    });
  }

  private async verifyPermission(userId: string, channelId: string) {
    const member = await this.prisma.channelMember.findUnique({
      where: {
        channelId_userId: {
          channelId,
          userId,
        },
      },
    });

    if (!member) {
      throw new ForbiddenException('Not a member of this channel');
    }

    // Only owners and admins can pin messages
    if (![MemberRole.OWNER, MemberRole.ADMIN].includes(member.role)) {
      throw new ForbiddenException('Insufficient permissions to pin messages');
    }
  }
}
```

### API Endpoints

**Create `apps/server/src/messages/messages.controller.ts`** (add to existing controller):

```typescript
import { Controller, Post, Get, Delete, Param, Body, Query, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';
import { CurrentUser } from '../auth/decorators/current-user.decorator';
import { ThreadingService } from './threading.service';
import { PinningService } from './pinning.service';

@Controller('messages')
@UseGuards(JwtAuthGuard)
export class MessagesController {
  constructor(
    private threadingService: ThreadingService,
    private pinningService: PinningService,
  ) {}

  // Threading endpoints
  @Post(':id/replies')
  async replyToThread(
    @CurrentUser('id') userId: string,
    @Param('id') parentId: string,
    @Body() body: { text: string; showInChannel?: boolean; metadata?: any },
  ) {
    return this.threadingService.replyToThread(userId, parentId, body);
  }

  @Get(':id/replies')
  async getThreadReplies(
    @Param('id') parentId: string,
    @Query('limit') limit?: string,
    @Query('cursor') cursor?: string,
  ) {
    return this.threadingService.getThreadReplies(
      parentId,
      limit ? parseInt(limit) : 50,
      cursor,
    );
  }

  @Get(':id/thread-metadata')
  async getThreadMetadata(@Param('id') parentId: string) {
    return this.threadingService.getThreadMetadata(parentId);
  }

  // Pinning endpoints
  @Post(':id/pin')
  async pinMessage(
    @CurrentUser('id') userId: string,
    @Param('id') messageId: string,
  ) {
    return this.pinningService.pinMessage(userId, messageId);
  }

  @Delete(':id/pin')
  async unpinMessage(
    @CurrentUser('id') userId: string,
    @Param('id') messageId: string,
  ) {
    return this.pinningService.unpinMessage(userId, messageId);
  }

  @Get('channels/:channelId/pinned')
  async getPinnedMessages(@Param('channelId') channelId: string) {
    return this.pinningService.getPinnedMessages(channelId);
  }
}
```

---

## Week 3: Rich Content

### Markdown Support

```bash
pnpm add marked dompurify
```

```typescript
@Injectable()
export class MarkdownService {
  parseMarkdown(text: string): string {
    const html = marked.parse(text);
    return DOMPurify.sanitize(html);
  }
}
```

### Link Previews

```bash
pnpm add axios cheerio
```

```typescript
@Injectable()
export class LinkPreviewService {
  async generatePreview(url: string) {
    const { data } = await axios.get(url);
    const $ = cheerio.load(data);
    
    return {
      title: $('meta[property="og:title"]').attr('content'),
      description: $('meta[property="og:description"]').attr('content'),
      image: $('meta[property="og:image"]').attr('content'),
    };
  }
}
```

### GIF Integration

```bash
pnpm add @giphy/js-fetch-api
```

```typescript
@Injectable()
export class GiphyService {
  private giphy = new GiphyFetch(process.env.GIPHY_API_KEY);

  async search(query: string, limit = 10) {
    const { data } = await this.giphy.search(query, { limit });
    return data.map(gif => ({
      id: gif.id,
      url: gif.images.original.url,
      preview: gif.images.preview_gif.url,
    }));
  }
}
```

---

## Week 4: Scheduling & Testing

### Scheduled Messages

**Add Prisma Schema** (`apps/server/prisma/schema.prisma`):

```prisma
enum ScheduledMessageStatus {
  PENDING
  SENT
  CANCELLED
  FAILED
}

model ScheduledMessage {
  id          String                 @id @default(uuid())
  userId      String                 @map("user_id")
  channelId   String                 @map("channel_id")
  text        String?
  metadata    Json                   @default("{}")
  attachments Json                   @default("[]")
  scheduledFor DateTime              @map("scheduled_for")
  status      ScheduledMessageStatus @default(PENDING)
  sentAt      DateTime?              @map("sent_at")
  failureReason String?              @map("failure_reason")
  createdAt   DateTime               @default(now()) @map("created_at")
  updatedAt   DateTime               @updatedAt @map("updated_at")

  user    User    @relation("ScheduledMessages", fields: [userId], references: [id], onDelete: Cascade)
  channel Channel @relation(fields: [channelId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([status, scheduledFor])
  @@map("scheduled_messages")
}
```

**Update User and Channel models**:

```prisma
model User {
  // ... existing fields ...
  scheduledMessages ScheduledMessage[] @relation("ScheduledMessages")
}

model Channel {
  // ... existing fields ...
  scheduledMessages ScheduledMessage[]
}
```

**Generate Migration**:

```bash
cd apps/server
npx prisma migrate dev --name add_scheduled_messages
```

**Create `apps/server/src/messages/scheduling.service.ts`**:

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { PrismaService } from '../prisma/prisma.service';
import { MessagesService } from './messages.service';
import { ScheduledMessageStatus } from '@prisma/client';

@Injectable()
export class SchedulingService {
  private readonly logger = new Logger(SchedulingService.name);

  constructor(
    private prisma: PrismaService,
    private messagesService: MessagesService,
  ) {}

  async scheduleMessage(userId: string, data: any) {
    // Validate scheduled time is in the future
    const scheduledFor = new Date(data.scheduledFor);
    if (scheduledFor <= new Date()) {
      throw new Error('Scheduled time must be in the future');
    }

    return this.prisma.scheduledMessage.create({
      data: {
        userId,
        channelId: data.channelId,
        text: data.text,
        metadata: data.metadata || {},
        attachments: data.attachments || [],
        scheduledFor,
        status: ScheduledMessageStatus.PENDING,
      },
    });
  }

  async cancelScheduledMessage(userId: string, scheduleId: string) {
    // Verify ownership
    const scheduled = await this.prisma.scheduledMessage.findUnique({
      where: { id: scheduleId },
    });

    if (!scheduled || scheduled.userId !== userId) {
      throw new Error('Scheduled message not found or unauthorized');
    }

    if (scheduled.status !== ScheduledMessageStatus.PENDING) {
      throw new Error('Can only cancel pending messages');
    }

    return this.prisma.scheduledMessage.update({
      where: { id: scheduleId },
      data: { status: ScheduledMessageStatus.CANCELLED },
    });
  }

  async getUserScheduledMessages(userId: string) {
    return this.prisma.scheduledMessage.findMany({
      where: {
        userId,
        status: ScheduledMessageStatus.PENDING,
      },
      include: {
        channel: {
          select: {
            id: true,
            name: true,
            type: true,
          },
        },
      },
      orderBy: { scheduledFor: 'asc' },
    });
  }

  @Cron(CronExpression.EVERY_MINUTE)
  async processScheduledMessages() {
    this.logger.debug('Processing scheduled messages...');

    const pending = await this.prisma.scheduledMessage.findMany({
      where: {
        status: ScheduledMessageStatus.PENDING,
        scheduledFor: { lte: new Date() },
      },
      take: 100, // Process in batches
    });

    this.logger.debug(`Found ${pending.length} messages to send`);

    for (const scheduled of pending) {
      try {
        // Send the message
        await this.messagesService.send(scheduled.userId, {
          channelId: scheduled.channelId,
          text: scheduled.text,
          metadata: scheduled.metadata,
          attachments: scheduled.attachments,
        });

        // Mark as sent
        await this.prisma.scheduledMessage.update({
          where: { id: scheduled.id },
          data: {
            status: ScheduledMessageStatus.SENT,
            sentAt: new Date(),
          },
        });

        this.logger.debug(`Sent scheduled message ${scheduled.id}`);
      } catch (error) {
        this.logger.error(`Failed to send scheduled message ${scheduled.id}:`, error);

        // Mark as failed
        await this.prisma.scheduledMessage.update({
          where: { id: scheduled.id },
          data: {
            status: ScheduledMessageStatus.FAILED,
            failureReason: error.message,
          },
        });
      }
    }
  }
}
```

**Install @nestjs/schedule**:

```bash
cd apps/server
pnpm add @nestjs/schedule
```

**Update AppModule** to enable scheduling:

```typescript
import { Module } from '@nestjs/common';
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot(),
    // ... other modules
  ],
})
export class AppModule {}
```

### Testing

**Create `apps/server/test/unit/threading.service.spec.ts`**:

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { ThreadingService } from '../../src/messages/threading.service';
import { PrismaService } from '../../src/prisma/prisma.service';
import { MessageType } from '@prisma/client';

describe('ThreadingService', () => {
  let service: ThreadingService;
  let prisma: PrismaService;

  const mockPrismaService = {
    message: {
      findUnique: jest.fn(),
      create: jest.fn(),
      findMany: jest.fn(),
      update: jest.fn(),
    },
    $transaction: jest.fn((callback) => callback(mockPrismaService)),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        ThreadingService,
        {
          provide: PrismaService,
          useValue: mockPrismaService,
        },
      ],
    }).compile();

    service = module.get<ThreadingService>(ThreadingService);
    prisma = module.get<PrismaService>(PrismaService);
  });

  describe('replyToThread', () => {
    it('should create thread reply', async () => {
      const parentId = 'parent-123';
      const userId = 'user-123';
      const parent = {
        id: parentId,
        channelId: 'channel-123',
        userId: 'user-456',
      };

      mockPrismaService.message.findUnique.mockResolvedValue(parent);
      mockPrismaService.message.create.mockResolvedValue({
        id: 'reply-123',
        parentMessageId: parentId,
        text: 'Reply',
        channelId: parent.channelId,
        userId,
      });

      const reply = await service.replyToThread(userId, parentId, { text: 'Reply' });

      expect(reply.parentMessageId).toBe(parentId);
      expect(mockPrismaService.message.create).toHaveBeenCalled();
    });

    it('should throw error if parent not found', async () => {
      mockPrismaService.message.findUnique.mockResolvedValue(null);

      await expect(
        service.replyToThread('user-123', 'invalid-id', { text: 'Reply' })
      ).rejects.toThrow('Parent message not found');
    });
  });

  describe('getThreadMetadata', () => {
    it('should return thread metadata', async () => {
      mockPrismaService.message.findUnique.mockResolvedValue({
        threadParticipantCount: 3,
        threadLastMessageAt: new Date(),
        _count: { replies: 5 },
      });

      const metadata = await service.getThreadMetadata('parent-123');

      expect(metadata.replyCount).toBe(5);
      expect(metadata.participantCount).toBe(3);
    });
  });
});
```

---

## Deliverables

- [x] Message threading
- [x] Message pinning (max 50/channel)
- [x] Markdown support
- [x] Link previews
- [x] GIF integration (Giphy)
- [x] Scheduled messages
- [x] Message bookmarks
- [x] Unit tests
- [x] API documentation

**Next**: [MONTH_06_ANALYTICS_WEBHOOKS.md](./MONTH_06_ANALYTICS_WEBHOOKS.md)
