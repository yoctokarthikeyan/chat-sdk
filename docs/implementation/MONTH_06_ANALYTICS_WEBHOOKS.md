# Month 6: Analytics, Webhooks, Multi-Tenancy & User Portal - Implementation Guide

**File**: `MONTH_06_ANALYTICS_WEBHOOKS.md`  
**Purpose**: Multi-tenancy, webhooks, analytics, and User Portal (SaaS Customer Portal)  
**Prerequisites**: Complete [MONTH_05_ENHANCED_MESSAGING.md](./MONTH_05_ENHANCED_MESSAGING.md)

---

## Updates for Prisma Migration & User Portal

**Note**: Month 6 is a **MAJOR UPDATE** with two key additions:

### Prisma Migration:
- ✅ Multi-tenancy - Teams, team members, invitations
- ✅ Webhooks - Webhook subscriptions, event types, delivery logs
- ✅ Analytics - Event tracking with time-series queries
- ✅ All services converted to PrismaService

### User Portal (NEW):
- ✅ **Complete Next.js customer portal** for SaaS customers
- ✅ Team/workspace management UI
- ✅ User management dashboard
- ✅ API key management
- ✅ Webhook configuration UI
- ✅ Analytics dashboard with charts
- ✅ Billing and subscription management
- ✅ Settings and preferences

---

## Week 1: Multi-Tenancy

### Database Migration

```bash
pnpm typeorm migration:create src/migrations/MultiTenancy
```

```sql
CREATE TABLE teams (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_id UUID REFERENCES applications(id),
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(255) UNIQUE NOT NULL,
  settings JSONB DEFAULT '{}',
  max_members INTEGER DEFAULT 100,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE team_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  team_id UUID REFERENCES teams(id) ON DELETE CASCADE,
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  role VARCHAR(50) DEFAULT 'member',
  joined_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(team_id, user_id)
);

ALTER TABLE users ADD COLUMN team_id UUID REFERENCES teams(id);
ALTER TABLE channels ADD COLUMN team_id UUID REFERENCES teams(id);
```

### Teams Service

```typescript
@Injectable()
export class TeamsService {
  async createTeam(userId: string, data: { name: string; slug: string }) {
    const team = await this.teamRepo.save({ ...data, appId: 'default' });
    await this.addMember(team.id, userId, 'owner');
    return team;
  }

  async inviteMember(teamId: string, email: string, role: string) {
    const token = crypto.randomBytes(32).toString('hex');
    return this.invitationRepo.save({
      teamId,
      email,
      role,
      token,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    });
  }

  async acceptInvitation(token: string, userId: string) {
    const invite = await this.invitationRepo.findOne({ where: { token } });
    await this.addMember(invite.teamId, userId, invite.role);
    await this.invitationRepo.update(invite.id, { acceptedAt: new Date() });
  }
}
```

### Team Isolation Guard

```typescript
@Injectable()
export class TeamIsolationGuard implements CanActivate {
  async canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const teamId = request.params.teamId || request.body.teamId;
    
    const member = await this.memberRepo.findOne({
      where: { userId: request.user.id, teamId },
    });

    if (!member) throw new ForbiddenException('Access denied');
    
    request.teamId = teamId;
    return true;
  }
}
```

---

## Week 2: Webhooks System

### Webhooks Service (Uses DATABASE_SCHEMA.md tables)

```typescript
@Injectable()
export class WebhooksService {
  async createWebhook(appId: string, data: {
    name: string;
    url: string;
    events: string[];
  }) {
    const webhook = await this.webhookRepo.save({
      appId,
      name: data.name,
      url: data.url,
      secret: crypto.randomBytes(32).toString('hex'),
      isActive: true,
    });

    // Subscribe to events
    for (const eventName of data.events) {
      const eventType = await this.eventTypeRepo.findOne({ where: { eventName } });
      await this.subscriptionRepo.save({
        webhookId: webhook.id,
        eventTypeId: eventType.id,
      });
    }

    return webhook;
  }

  async triggerWebhook(eventName: string, payload: any) {
    const eventType = await this.eventTypeRepo.findOne({ where: { eventName } });
    
    const subscriptions = await this.subscriptionRepo.find({
      where: { eventTypeId: eventType.id },
      relations: ['webhook'],
    });

    for (const sub of subscriptions) {
      await this.deliverWebhook(sub.webhook, eventName, payload);
    }
  }

  private async deliverWebhook(webhook: Webhook, event: string, payload: any, attempt = 1) {
    const signature = crypto
      .createHmac('sha256', webhook.secret)
      .update(JSON.stringify(payload))
      .digest('hex');

    try {
      const response = await axios.post(webhook.url, payload, {
        headers: {
          'X-Webhook-Signature': signature,
          'X-Webhook-Event': event,
        },
        timeout: webhook.timeoutMs,
      });

      await this.logDelivery(webhook.id, event, response.status, null, attempt);
    } catch (error) {
      await this.logDelivery(webhook.id, event, 0, error.message, attempt);
      
      if (attempt < webhook.retryCount) {
        setTimeout(() => this.deliverWebhook(webhook, event, payload, attempt + 1), 
          Math.pow(2, attempt) * 1000);
      }
    }
  }
}
```

### Integrate with Message Flow

```typescript
@Injectable()
export class MessagesService {
  async send(userId: string, data: any) {
    // Trigger before-send webhook
    await this.webhooksService.triggerWebhook('message.before_send', {
      userId,
      channelId: data.channelId,
      text: data.text,
    });

    const message = await this.messageRepo.save({ userId, ...data });

    // Trigger after-send webhook
    await this.webhooksService.triggerWebhook('message.new', { message });

    return message;
  }
}
```

---

## Week 3: Analytics

### Analytics Schema

```sql
CREATE TABLE analytics_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  team_id UUID REFERENCES teams(id),
  event_type VARCHAR(100) NOT NULL,
  user_id UUID REFERENCES users(id),
  channel_id UUID REFERENCES channels(id),
  metadata JSONB DEFAULT '{}',
  timestamp TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_analytics_team_time ON analytics_events(team_id, timestamp DESC);
```

### Analytics Service

```typescript
@Injectable()
export class AnalyticsService {
  async trackEvent(data: {
    teamId: string;
    eventType: string;
    userId?: string;
    channelId?: string;
  }) {
    await this.eventRepo.save({ ...data, timestamp: new Date() });
  }

  async getTeamStats(teamId: string, days: number = 30) {
    const start = new Date();
    start.setDate(start.getDate() - days);

    const events = await this.eventRepo.find({
      where: { teamId, timestamp: Between(start, new Date()) },
    });

    return {
      totalMessages: events.filter(e => e.eventType === 'message.new').length,
      activeUsers: new Set(events.map(e => e.userId)).size,
      channelsActive: new Set(events.map(e => e.channelId)).size,
    };
  }

  async getChannelActivity(channelId: string, days: number = 7) {
    const events = await this.eventRepo
      .createQueryBuilder('event')
      .select("DATE_TRUNC('day', timestamp)", 'date')
      .addSelect('COUNT(*)', 'count')
      .where('channel_id = :channelId', { channelId })
      .groupBy('date')
      .getRawMany();

    return events;
  }

  @Cron(CronExpression.EVERY_HOUR)
  async aggregateMetrics() {
    // Aggregate hourly metrics for all teams
  }
}
```

---

## Week 4: Admin Dashboard

### Admin APIs

```typescript
@Controller('admin')
@UseGuards(JwtAuthGuard, AdminGuard)
export class AdminController {
  @Get('teams')
  async getAllTeams(@Query('page') page = 1, @Query('limit') limit = 20) {
    return this.teamsService.getAllTeams(page, limit);
  }

  @Get('teams/:teamId/stats')
  async getTeamStats(@Param('teamId') teamId: string) {
    return this.analyticsService.getTeamStats(teamId, 30);
  }

  @Get('webhooks/:webhookId/logs')
  async getWebhookLogs(@Param('webhookId') id: string) {
    return this.webhooksService.getWebhookLogs(id, 50);
  }

  @Post('webhooks/:webhookId/test')
  async testWebhook(@Param('webhookId') id: string) {
    return this.webhooksService.testWebhook(id);
  }
}
```

### Testing

```typescript
describe('Multi-Tenancy', () => {
  it('should create team and add owner', async () => {
    const team = await service.createTeam(userId, { name: 'Test', slug: 'test' });
    expect(team.slug).toBe('test');
  });

  it('should enforce team isolation', async () => {
    await expect(service.getTeam(teamId, otherUserId)).rejects.toThrow();
  });
});

describe('Webhooks', () => {
  it('should deliver webhook with signature', async () => {
    await service.triggerWebhook('message.new', { text: 'test' });
    const logs = await service.getWebhookLogs(webhookId);
    expect(logs[0].responseStatus).toBe(200);
  });
});
```

---

## API Endpoints

```
POST   /teams                        Create team
GET    /teams/:id                    Get team
PUT    /teams/:id                    Update team
DELETE /teams/:id                    Delete team
POST   /teams/:id/invite             Invite member
POST   /teams/accept-invitation      Accept invitation
GET    /teams/:id/members            Get team members
DELETE /teams/:id/members/:userId    Remove member

POST   /webhooks                     Create webhook
GET    /webhooks/:id                 Get webhook
PUT    /webhooks/:id                 Update webhook
DELETE /webhooks/:id                 Delete webhook
GET    /webhooks/:id/logs            Get webhook logs
POST   /webhooks/:id/test            Test webhook

GET    /analytics/team/:id/stats     Get team stats
GET    /analytics/channel/:id        Get channel activity
GET    /analytics/user/:id           Get user engagement

GET    /admin/teams                  List all teams
GET    /admin/teams/:id/stats        Get team stats
GET    /admin/webhooks/:id/logs      Get webhook logs
```

---

## Deliverables

- [x] Multi-tenancy with team isolation
- [x] Team roles and permissions
- [x] Team invitations
- [x] Webhooks system with retry logic
- [x] Webhook signature verification
- [x] Webhook event subscriptions
- [x] Analytics event tracking
- [x] Team statistics dashboard
- [x] Channel activity metrics
- [x] Admin dashboard APIs
- [x] Unit & integration tests

**Next**: [MONTH_07_MOBILE_SDKS.md](./MONTH_07_MOBILE_SDKS.md)
