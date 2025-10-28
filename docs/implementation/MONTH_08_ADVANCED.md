# Month 8: Advanced Features - Implementation Guide

**File**: `MONTH_08_ADVANCED.md`  
**Purpose**: Campaigns, AI features, advanced search, message translation  
**Prerequisites**: Complete [MONTH_07_MOBILE_SDKS.md](./MONTH_07_MOBILE_SDKS.md)

---

## Updates for Prisma Migration

**Note**: Month 8 adds advanced features with database operations. All have been updated to use Prisma:

- ✅ Campaign System - Prisma models with status enums
- ✅ Campaign Recipients - Batch processing with Prisma
- ✅ AI Features - No database changes (external APIs)
- ✅ Advanced Search - Elasticsearch integration (no ORM)
- ✅ All services converted to PrismaService

---

## Assumptions

- User base growing (~20K users, ~500 teams)
- Need for marketing campaigns and broadcasts
- AI/ML features for better UX
- Advanced search beyond Elasticsearch basics
- Team: 2 Backend Engineers, 1 ML Engineer, 1 Frontend Engineer

---

## Week 1: Campaign System

### Task 1: Prisma Schema

**Update `packages/server/prisma/schema.prisma`**:

```prisma
enum CampaignStatus {
  DRAFT
  SCHEDULED
  RUNNING
  COMPLETED
  CANCELLED
}

enum CampaignTargetType {
  ALL
  CHANNELS
  USERS
  SEGMENTS
}

enum CampaignRecipientStatus {
  PENDING
  SENT
  FAILED
}

model Campaign {
  id              String              @id @default(uuid())
  teamId          String              @map("team_id")
  name            String
  description     String?
  messageTemplate Json                @map("message_template")
  targetType      CampaignTargetType  @map("target_type")
  targetChannels  String[]            @map("target_channels")
  targetUsers     String[]            @map("target_users")
  segmentFilters  Json?               @map("segment_filters")
  status          CampaignStatus      @default(DRAFT)
  scheduledFor    DateTime?           @map("scheduled_for")
  sentAt          DateTime?           @map("sent_at")
  sentCount       Int                 @default(0) @map("sent_count")
  failedCount     Int                 @default(0) @map("failed_count")
  createdBy       String              @map("created_by")
  createdAt       DateTime            @default(now()) @map("created_at")
  updatedAt       DateTime            @updatedAt @map("updated_at")

  team       Team                @relation(fields: [teamId], references: [id], onDelete: Cascade)
  creator    User                @relation("CreatedCampaigns", fields: [createdBy], references: [id])
  recipients CampaignRecipient[]

  @@index([teamId])
  @@index([status])
  @@index([scheduledFor])
  @@map("campaigns")
}

model CampaignRecipient {
  id           String                  @id @default(uuid())
  campaignId   String                  @map("campaign_id")
  userId       String                  @map("user_id")
  channelId    String?                 @map("channel_id")
  status       CampaignRecipientStatus @default(PENDING)
  sentAt       DateTime?               @map("sent_at")
  errorMessage String?                 @map("error_message")
  createdAt    DateTime                @default(now()) @map("created_at")

  campaign Campaign @relation(fields: [campaignId], references: [id], onDelete: Cascade)
  user     User     @relation("CampaignRecipients", fields: [userId], references: [id])
  channel  Channel? @relation(fields: [channelId], references: [id])

  @@index([campaignId])
  @@index([status])
  @@map("campaign_recipients")
}
```

**Update related models**:

```prisma
model User {
  // ... existing fields ...
  createdCampaigns   Campaign[]          @relation("CreatedCampaigns")
  campaignRecipients CampaignRecipient[] @relation("CampaignRecipients")
}

model Team {
  // ... existing fields ...
  campaigns Campaign[]
}

model Channel {
  // ... existing fields ...
  campaignRecipients CampaignRecipient[]
}
```

**Generate Migration**:

```bash
cd packages/server
npx prisma migrate dev --name add_campaigns
```

---

### Task 2: Campaign Service

**Create `packages/server/src/campaigns/campaigns.service.ts`**:

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { PrismaService } from '../prisma/prisma.service';
import { MessagesService } from '../messages/messages.service';
import { CampaignStatus, CampaignTargetType, CampaignRecipientStatus } from '@prisma/client';

@Injectable()
export class CampaignsService {
  private readonly logger = new Logger(CampaignsService.name);

  constructor(
    private prisma: PrismaService,
    private messagesService: MessagesService,
  ) {}

  async createCampaign(userId: string, data: {
    teamId: string;
    name: string;
    description?: string;
    messageTemplate: any;
    targetType: string;
    targetChannels?: string[];
    targetUsers?: string[];
    segmentFilters?: any;
    scheduledFor?: Date;
  }) {
    const campaign = this.campaignRepository.create({
      ...data,
      createdBy: userId,
      status: data.scheduledFor ? 'scheduled' : 'draft',
    });

    await this.campaignRepository.save(campaign);

    // Generate recipients
    await this.generateRecipients(campaign);

    return campaign;
  }

  async executeCampaign(campaignId: string) {
    const campaign = await this.campaignRepository.findOne({
      where: { id: campaignId },
    });

    if (!campaign) {
      throw new Error('Campaign not found');
    }

    if (campaign.status === 'completed') {
      throw new Error('Campaign already completed');
    }

    await this.campaignRepository.update(campaignId, {
      status: 'running',
      sentAt: new Date(),
    });

    // Get pending recipients
    const recipients = await this.recipientRepository.find({
      where: { campaignId, status: 'pending' },
      take: 1000, // Process in batches
    });

    let sentCount = 0;
    let failedCount = 0;

    for (const recipient of recipients) {
      try {
        await this.sendCampaignMessage(campaign, recipient);
        
        await this.recipientRepository.update(recipient.id, {
          status: 'sent',
          sentAt: new Date(),
        });
        
        sentCount++;
      } catch (error) {
        await this.recipientRepository.update(recipient.id, {
          status: 'failed',
          errorMessage: error.message,
        });
        
        failedCount++;
      }

      // Rate limiting: 100 messages per second
      await this.sleep(10);
    }

    // Update campaign stats
    await this.campaignRepository.update(campaignId, {
      sentCount: campaign.sentCount + sentCount,
      failedCount: campaign.failedCount + failedCount,
      status: recipients.length < 1000 ? 'completed' : 'running',
    });

    this.logger.log(`Campaign ${campaignId}: sent ${sentCount}, failed ${failedCount}`);
  }

  private async sendCampaignMessage(campaign: Campaign, recipient: CampaignRecipient) {
    const message = this.interpolateTemplate(campaign.messageTemplate, recipient);

    await this.messagesService.send(campaign.createdBy, {
      channelId: recipient.channelId,
      text: message.text,
      type: 'campaign',
      metadata: {
        campaignId: campaign.id,
        campaignName: campaign.name,
      },
    });
  }

  private interpolateTemplate(template: any, recipient: CampaignRecipient): any {
    // Replace variables like {{user.name}} with actual values
    let text = template.text;
    
    // Simple interpolation (can be enhanced)
    text = text.replace(/\{\{user\.id\}\}/g, recipient.userId);
    
    return { ...template, text };
  }

  private async generateRecipients(campaign: Campaign) {
    let userIds: string[] = [];

    switch (campaign.targetType) {
      case 'all':
        userIds = await this.getAllUsers(campaign.teamId);
        break;
      case 'users':
        userIds = campaign.targetUsers || [];
        break;
      case 'channels':
        userIds = await this.getUsersFromChannels(campaign.targetChannels || []);
        break;
      case 'segments':
        userIds = await this.getUsersBySegment(campaign.segmentFilters);
        break;
    }

    // Create recipients
    for (const userId of userIds) {
      const channelId = await this.getOrCreateDirectChannel(campaign.createdBy, userId);
      
      const recipient = this.recipientRepository.create({
        campaignId: campaign.id,
        userId,
        channelId,
        status: 'pending',
      });
      
      await this.recipientRepository.save(recipient);
    }
  }

  private async getAllUsers(teamId: string): Promise<string[]> {
    // Get all users in team
    return [];
  }

  private async getUsersFromChannels(channelIds: string[]): Promise<string[]> {
    // Get unique users from channels
    return [];
  }

  private async getUsersBySegment(filters: any): Promise<string[]> {
    // Apply segment filters (e.g., active in last 7 days, sent > 10 messages)
    return [];
  }

  private async getOrCreateDirectChannel(senderId: string, recipientId: string): Promise<string> {
    // Get or create 1-1 channel
    return 'channel-id';
  }

  @Cron(CronExpression.EVERY_MINUTE)
  async processScheduledCampaigns() {
    const now = new Date();
    
    const scheduled = await this.campaignRepository.find({
      where: {
        status: 'scheduled',
        scheduledFor: In([now]), // LessThanOrEqual
      },
      take: 10,
    });

    for (const campaign of scheduled) {
      try {
        await this.executeCampaign(campaign.id);
      } catch (error) {
        this.logger.error(`Failed to execute campaign ${campaign.id}:`, error);
      }
    }
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

---

## Week 2: AI Features

### Task 1: Message Translation

**Install Dependencies**:

```bash
pnpm add @google-cloud/translate
```

**Create `packages/server/src/ai/translation.service.ts`**:

```typescript
import { Injectable } from '@nestjs/common';
import { Translate } from '@google-cloud/translate/build/src/v2';

@Injectable()
export class TranslationService {
  private translate: Translate;

  constructor() {
    this.translate = new Translate({
      key: process.env.GOOGLE_TRANSLATE_API_KEY,
    });
  }

  async translateMessage(text: string, targetLanguage: string): Promise<string> {
    try {
      const [translation] = await this.translate.translate(text, targetLanguage);
      return translation;
    } catch (error) {
      throw new Error(`Translation failed: ${error.message}`);
    }
  }

  async detectLanguage(text: string): Promise<string> {
    try {
      const [detection] = await this.translate.detect(text);
      return detection.language;
    } catch (error) {
      throw new Error(`Language detection failed: ${error.message}`);
    }
  }

  async getSupportedLanguages(): Promise<any[]> {
    const [languages] = await this.translate.getLanguages();
    return languages;
  }
}
```

---

### Task 2: Smart Replies (OpenAI)

**Install Dependencies**:

```bash
pnpm add openai
```

**Create `packages/server/src/ai/smart-replies.service.ts`**:

```typescript
import { Injectable } from '@nestjs/common';
import OpenAI from 'openai';

@Injectable()
export class SmartRepliesService {
  private openai: OpenAI;

  constructor() {
    this.openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY,
    });
  }

  async generateReplies(messageHistory: string[], count: number = 3): Promise<string[]> {
    const prompt = this.buildPrompt(messageHistory);

    try {
      const completion = await this.openai.chat.completions.create({
        model: 'gpt-3.5-turbo',
        messages: [
          {
            role: 'system',
            content: `Generate ${count} short, contextual reply suggestions. Each reply should be on a new line and be concise (max 50 characters).`,
          },
          {
            role: 'user',
            content: prompt,
          },
        ],
        max_tokens: 150,
        temperature: 0.7,
      });

      const replies = completion.choices[0].message.content?.split('\n').filter(r => r.trim()) || [];
      return replies.slice(0, count);
    } catch (error) {
      throw new Error(`Smart replies generation failed: ${error.message}`);
    }
  }

  private buildPrompt(messageHistory: string[]): string {
    return `Recent conversation:\n${messageHistory.slice(-5).join('\n')}\n\nSuggest appropriate replies:`;
  }
}
```

---

### Task 3: Content Moderation (AI)

**Create `packages/server/src/ai/content-moderation.service.ts`**:

```typescript
import { Injectable } from '@nestjs/common';
import OpenAI from 'openai';
import vision from '@google-cloud/vision';

@Injectable()
export class ContentModerationService {
  private openai: OpenAI;
  private visionClient: vision.ImageAnnotatorClient;

  constructor() {
    this.openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY,
    });
    
    this.visionClient = new vision.ImageAnnotatorClient();
  }

  async moderateText(text: string): Promise<{
    flagged: boolean;
    categories: string[];
    score: number;
  }> {
    try {
      const moderation = await this.openai.moderations.create({
        input: text,
      });

      const result = moderation.results[0];
      
      return {
        flagged: result.flagged,
        categories: Object.keys(result.categories).filter(k => result.categories[k]),
        score: Math.max(...Object.values(result.category_scores)),
      };
    } catch (error) {
      throw new Error(`Text moderation failed: ${error.message}`);
    }
  }

  async moderateImage(imageUrl: string): Promise<{
    safe: boolean;
    adult: string;
    violence: string;
    racy: string;
  }> {
    try {
      const [result] = await this.visionClient.safeSearchDetection(imageUrl);
      const detections = result.safeSearchAnnotation;

      return {
        safe: this.isSafe(detections),
        adult: detections.adult,
        violence: detections.violence,
        racy: detections.racy,
      };
    } catch (error) {
      throw new Error(`Image moderation failed: ${error.message}`);
    }
  }

  private isSafe(detections: any): boolean {
    const unsafe = ['VERY_LIKELY', 'LIKELY'];
    return !unsafe.includes(detections.adult) &&
           !unsafe.includes(detections.violence) &&
           !unsafe.includes(detections.racy);
  }
}
```

---

### Task 4: Message Sentiment Analysis

**Create `packages/server/src/ai/sentiment.service.ts`**:

```typescript
import { Injectable } from '@nestjs/common';
import OpenAI from 'openai';

@Injectable()
export class SentimentService {
  private openai: OpenAI;

  constructor() {
    this.openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY,
    });
  }

  async analyzeSentiment(text: string): Promise<{
    sentiment: 'positive' | 'negative' | 'neutral';
    score: number;
    confidence: number;
  }> {
    try {
      const completion = await this.openai.chat.completions.create({
        model: 'gpt-3.5-turbo',
        messages: [
          {
            role: 'system',
            content: 'Analyze the sentiment of the following text. Respond with JSON: {"sentiment": "positive|negative|neutral", "score": 0-1, "confidence": 0-1}',
          },
          {
            role: 'user',
            content: text,
          },
        ],
        max_tokens: 50,
        temperature: 0.3,
      });

      const result = JSON.parse(completion.choices[0].message.content || '{}');
      return result;
    } catch (error) {
      return { sentiment: 'neutral', score: 0.5, confidence: 0 };
    }
  }
}
```

---

## Week 3: Advanced Search

### Task 1: Enhanced Elasticsearch Integration

**Create `packages/server/src/search/advanced-search.service.ts`**:

```typescript
import { Injectable } from '@nestjs/common';
import { Client } from '@elastic/elasticsearch';

@Injectable()
export class AdvancedSearchService {
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
        teamId: message.teamId,
        createdAt: message.createdAt,
        mentions: message.mentions || [],
        attachments: message.attachments || [],
      },
    });
  }

  async search(query: {
    text?: string;
    channelId?: string;
    userId?: string;
    teamId?: string;
    dateFrom?: Date;
    dateTo?: Date;
    hasAttachments?: boolean;
    mentions?: string[];
  }) {
    const must: any[] = [];
    const filter: any[] = [];

    // Text search with fuzzy matching
    if (query.text) {
      must.push({
        multi_match: {
          query: query.text,
          fields: ['text^2', 'attachments.fileName'],
          fuzziness: 'AUTO',
          operator: 'and',
        },
      });
    }

    // Filters
    if (query.channelId) {
      filter.push({ term: { channelId: query.channelId } });
    }

    if (query.userId) {
      filter.push({ term: { userId: query.userId } });
    }

    if (query.teamId) {
      filter.push({ term: { teamId: query.teamId } });
    }

    if (query.dateFrom || query.dateTo) {
      const range: any = {};
      if (query.dateFrom) range.gte = query.dateFrom;
      if (query.dateTo) range.lte = query.dateTo;
      filter.push({ range: { createdAt: range } });
    }

    if (query.hasAttachments) {
      filter.push({ exists: { field: 'attachments' } });
    }

    if (query.mentions && query.mentions.length > 0) {
      filter.push({ terms: { mentions: query.mentions } });
    }

    const result = await this.client.search({
      index: 'messages',
      body: {
        query: {
          bool: {
            must,
            filter,
          },
        },
        highlight: {
          fields: {
            text: {},
          },
        },
        sort: [{ createdAt: 'desc' }],
        size: 50,
      },
    });

    return result.hits.hits.map((hit: any) => ({
      ...hit._source,
      highlight: hit.highlight,
      score: hit._score,
    }));
  }

  async searchSuggestions(prefix: string, teamId: string): Promise<string[]> {
    const result = await this.client.search({
      index: 'messages',
      body: {
        query: {
          bool: {
            must: [
              {
                match_phrase_prefix: {
                  text: prefix,
                },
              },
            ],
            filter: [
              { term: { teamId } },
            ],
          },
        },
        size: 5,
        _source: ['text'],
      },
    });

    return result.hits.hits.map((hit: any) => hit._source.text);
  }

  async aggregateByChannel(teamId: string, dateFrom: Date, dateTo: Date) {
    const result = await this.client.search({
      index: 'messages',
      body: {
        query: {
          bool: {
            filter: [
              { term: { teamId } },
              { range: { createdAt: { gte: dateFrom, lte: dateTo } } },
            ],
          },
        },
        aggs: {
          by_channel: {
            terms: {
              field: 'channelId',
              size: 20,
            },
          },
        },
        size: 0,
      },
    });

    return result.aggregations.by_channel.buckets;
  }
}
```

---

## Week 4: Integration & Testing

### Task 1: AI Controller

**Create `packages/server/src/ai/ai.controller.ts`**:

```typescript
import { Controller, Post, Get, Body, Query, UseGuards } from '@nestjs/common';
import { ApiTags, ApiBearerAuth } from '@nestjs/swagger';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';
import { TranslationService } from './translation.service';
import { SmartRepliesService } from './smart-replies.service';
import { ContentModerationService } from './content-moderation.service';
import { SentimentService } from './sentiment.service';

@ApiTags('AI')
@ApiBearerAuth()
@UseGuards(JwtAuthGuard)
@Controller('ai')
export class AIController {
  constructor(
    private translationService: TranslationService,
    private smartRepliesService: SmartRepliesService,
    private moderationService: ContentModerationService,
    private sentimentService: SentimentService,
  ) {}

  @Post('translate')
  async translate(@Body() body: { text: string; targetLanguage: string }) {
    return this.translationService.translateMessage(body.text, body.targetLanguage);
  }

  @Post('smart-replies')
  async smartReplies(@Body() body: { messageHistory: string[] }) {
    return this.smartRepliesService.generateReplies(body.messageHistory);
  }

  @Post('moderate/text')
  async moderateText(@Body() body: { text: string }) {
    return this.moderationService.moderateText(body.text);
  }

  @Post('moderate/image')
  async moderateImage(@Body() body: { imageUrl: string }) {
    return this.moderationService.moderateImage(body.imageUrl);
  }

  @Post('sentiment')
  async analyzeSentiment(@Body() body: { text: string }) {
    return this.sentimentService.analyzeSentiment(body.text);
  }

  @Get('languages')
  async getSupportedLanguages() {
    return this.translationService.getSupportedLanguages();
  }
}
```

---

### Task 2: Campaign Controller

**Create `packages/server/src/campaigns/campaigns.controller.ts`**:

```typescript
import { Controller, Post, Get, Body, Param, UseGuards } from '@nestjs/common';
import { ApiTags, ApiBearerAuth } from '@nestjs/swagger';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';
import { CurrentUser } from '../auth/decorators/current-user.decorator';
import { CampaignsService } from './campaigns.service';

@ApiTags('Campaigns')
@ApiBearerAuth()
@UseGuards(JwtAuthGuard)
@Controller('campaigns')
export class CampaignsController {
  constructor(private campaignsService: CampaignsService) {}

  @Post()
  async createCampaign(@CurrentUser() user: any, @Body() body: any) {
    return this.campaignsService.createCampaign(user.id, body);
  }

  @Post(':id/execute')
  async executeCampaign(@Param('id') id: string) {
    return this.campaignsService.executeCampaign(id);
  }

  @Get(':id')
  async getCampaign(@Param('id') id: string) {
    return this.campaignsService.getCampaign(id);
  }

  @Get(':id/stats')
  async getCampaignStats(@Param('id') id: string) {
    return this.campaignsService.getCampaignStats(id);
  }
}
```

---

### Task 3: Testing

```typescript
describe('CampaignsService', () => {
  it('should create and execute campaign', async () => {
    const campaign = await service.createCampaign(userId, {
      name: 'Test Campaign',
      messageTemplate: { text: 'Hello {{user.name}}' },
      targetType: 'all',
    });

    await service.executeCampaign(campaign.id);

    const stats = await service.getCampaignStats(campaign.id);
    expect(stats.sentCount).toBeGreaterThan(0);
  });
});

describe('AI Services', () => {
  it('should translate message', async () => {
    const translated = await translationService.translateMessage('Hello', 'es');
    expect(translated).toBe('Hola');
  });

  it('should generate smart replies', async () => {
    const replies = await smartRepliesService.generateReplies(['How are you?']);
    expect(replies.length).toBe(3);
  });

  it('should moderate content', async () => {
    const result = await moderationService.moderateText('inappropriate content');
    expect(result.flagged).toBe(true);
  });
});
```

---

## API Endpoints

```
Campaigns:
POST   /campaigns                    Create campaign
GET    /campaigns/:id                Get campaign
POST   /campaigns/:id/execute        Execute campaign
GET    /campaigns/:id/stats          Get campaign stats

AI:
POST   /ai/translate                 Translate message
POST   /ai/smart-replies             Generate smart replies
POST   /ai/moderate/text             Moderate text
POST   /ai/moderate/image            Moderate image
POST   /ai/sentiment                 Analyze sentiment
GET    /ai/languages                 Get supported languages

Search:
POST   /search/messages              Advanced search
GET    /search/suggestions           Search suggestions
GET    /search/aggregations          Search aggregations
```

---

## Deliverables

- [x] Campaign system with scheduling
- [x] Campaign recipient targeting
- [x] Message translation (Google Translate)
- [x] Smart reply suggestions (OpenAI)
- [x] Content moderation (text & image)
- [x] Sentiment analysis
- [x] Advanced Elasticsearch search
- [x] Search suggestions & autocomplete
- [x] Search aggregations
- [x] Rate limiting for campaigns
- [x] Unit & integration tests
- [x] API documentation

**Next**: [MONTH_09_SECURITY.md](./MONTH_09_SECURITY.md)
