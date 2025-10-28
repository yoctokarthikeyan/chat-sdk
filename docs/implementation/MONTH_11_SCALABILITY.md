# Month 11: Scalability & Performance - Implementation Guide

**File**: `MONTH_11_SCALABILITY.md`  
**Purpose**: Multi-region deployment, database sharding, Redis cluster, CDN, performance optimization  
**Prerequisites**: Complete [MONTH_10_VOICE_VIDEO.md](./MONTH_10_VOICE_VIDEO.md)

---

## Updates for Prisma Migration & User Portal

**Note**: Month 11 focuses on scalability. Prisma updates include:

### Prisma Migration:
- ✅ Multi-database Prisma clients for sharding
- ✅ Connection pooling with Prisma
- ✅ Read replicas configuration
- ✅ Query optimization with indexes
- ✅ Prisma Accelerate for edge caching

### User Portal Updates:
- ✅ Performance monitoring dashboard
- ✅ Database health metrics
- ✅ Cache hit rate visualization
- ✅ Multi-region status

---

## Assumptions

- Growing to 100K+ users, 10K+ teams
- Need for global presence and low latency
- Database performance bottlenecks
- High availability requirements
- Team: 2 Backend Engineers, 1 DevOps Engineer, 1 Performance Engineer

---

## Week 1: Database Sharding

### Task 1: Sharding Strategy

**Shard by Team ID** (Recommended for multi-tenancy):

```typescript
// Sharding configuration
export const SHARD_CONFIG = {
  shards: [
    { id: 'shard-1', host: 'db1.example.com', port: 5432 },
    { id: 'shard-2', host: 'db2.example.com', port: 5432 },
    { id: 'shard-3', host: 'db3.example.com', port: 5432 },
    { id: 'shard-4', host: 'db4.example.com', port: 5432 },
  ],
  shardCount: 4,
};

export function getShardForTeam(teamId: string): string {
  const hash = createHash('md5').update(teamId).digest('hex');
  const shardIndex = parseInt(hash.substring(0, 8), 16) % SHARD_CONFIG.shardCount;
  return SHARD_CONFIG.shards[shardIndex].id;
}
```

---

### Task 2: Multi-Database Connection Manager

**Create `packages/server/src/database/shard-manager.ts`** (Prisma version):

```typescript
import { PrismaClient } from '@prisma/client';
import { Injectable, OnModuleInit } from '@nestjs/common';
import { SHARD_CONFIG, getShardForTeam } from './shard-config';

@Injectable()
export class ShardManager implements OnModuleInit {
  private prismaClients = new Map<string, PrismaClient>();

  async onModuleInit() {
    for (const shard of SHARD_CONFIG.shards) {
      const prisma = new PrismaClient({
        datasources: {
          db: {
            url: `postgresql://${process.env.DB_USERNAME}:${process.env.DB_PASSWORD}@${shard.host}:${shard.port}/${process.env.DB_NAME}`,
          },
        },
        log: ['error', 'warn'],
      });

      await prisma.$connect();
      this.prismaClients.set(shard.id, prisma);
    }
  }

  async onModuleDestroy() {
    // Disconnect all Prisma clients
    await Promise.all(
      Array.from(this.prismaClients.values()).map(client => client.$disconnect())
    );
  }

  getPrismaClient(teamId: string): PrismaClient {
    const shardId = getShardForTeam(teamId);
    const client = this.prismaClients.get(shardId);
    
    if (!client) {
      throw new Error(`Shard ${shardId} not found`);
    }
    
    return client;
  }

  async executeOnShard<T>(
    teamId: string,
    callback: (prisma: PrismaClient) => Promise<T>
  ): Promise<T> {
    const prisma = this.getPrismaClient(teamId);
    return callback(prisma);
  }

  async executeOnAllShards<T>(
    callback: (prisma: PrismaClient) => Promise<T>
  ): Promise<T[]> {
    const promises = Array.from(this.prismaClients.values()).map(client =>
      callback(client)
    );
    return Promise.all(promises);
  }

  // Get all shard IDs
  getShardIds(): string[] {
    return Array.from(this.prismaClients.keys());
  }
}
```

---

### Task 3: Sharded Repository Pattern

**Create `packages/server/src/database/sharded-service.ts`** (Prisma version):

```typescript
import { Injectable } from '@nestjs/common';
import { ShardManager } from './shard-manager';

@Injectable()
export class ShardedMessagesService {
  constructor(private shardManager: ShardManager) {}

  async findByChannel(teamId: string, channelId: string, limit: number = 50) {
    return this.shardManager.executeOnShard(teamId, async (prisma) => {
      return prisma.message.findMany({
        where: { channelId },
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
        },
        orderBy: { createdAt: 'desc' },
        take: limit,
      });
    });
  }

  async create(teamId: string, data: any) {
    return this.shardManager.executeOnShard(teamId, async (prisma) => {
      return prisma.message.create({
        data,
        include: {
          user: true,
          attachments: true,
        },
      });
    });
  }

  async searchAcrossShards(query: string, limit: number = 50) {
    const results = await this.shardManager.executeOnAllShards(async (prisma) => {
      return prisma.message.findMany({
        where: {
          text: {
            contains: query,
            mode: 'insensitive',
          },
          isDeleted: false,
        },
        include: {
          user: {
            select: {
              id: true,
              username: true,
              displayName: true,
            },
          },
          channel: {
            select: {
              id: true,
              name: true,
            },
          },
        },
        take: limit,
      });
    });

    // Merge and sort results from all shards
    return results
      .flat()
      .sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime())
      .slice(0, limit);
  }

  async getTeamStats(teamId: string) {
    return this.shardManager.executeOnShard(teamId, async (prisma) => {
      const [messageCount, channelCount, userCount] = await Promise.all([
        prisma.message.count({ where: { channel: { teamId } } }),
        prisma.channel.count({ where: { teamId } }),
        prisma.teamMember.count({ where: { teamId } }),
      ]);

      return { messageCount, channelCount, userCount };
    });
  }
}
```

---

## Week 2: Redis Cluster

### Task 1: Redis Cluster Setup

**Docker Compose for Redis Cluster**:

```yaml
version: '3.8'

services:
  redis-node-1:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --port 6379
    ports:
      - "6379:6379"
    volumes:
      - redis-1-data:/data

  redis-node-2:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --port 6380
    ports:
      - "6380:6380"
    volumes:
      - redis-2-data:/data

  redis-node-3:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --port 6381
    ports:
      - "6381:6381"
    volumes:
      - redis-3-data:/data

volumes:
  redis-1-data:
  redis-2-data:
  redis-3-data:
```

**Initialize Cluster**:

```bash
docker-compose up -d
docker exec -it redis-node-1 redis-cli --cluster create \
  127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6381 \
  --cluster-replicas 0
```

---

### Task 2: Redis Cluster Client

**Install Dependencies**:

```bash
pnpm add ioredis
```

**Create `packages/server/src/cache/redis-cluster.service.ts`**:

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import Redis from 'ioredis';

@Injectable()
export class RedisClusterService implements OnModuleInit {
  private cluster: Redis.Cluster;

  onModuleInit() {
    this.cluster = new Redis.Cluster([
      { host: 'localhost', port: 6379 },
      { host: 'localhost', port: 6380 },
      { host: 'localhost', port: 6381 },
    ], {
      redisOptions: {
        password: process.env.REDIS_PASSWORD,
      },
      clusterRetryStrategy: (times) => Math.min(times * 100, 2000),
    });

    this.cluster.on('error', (err) => {
      console.error('Redis Cluster Error:', err);
    });
  }

  async get(key: string): Promise<string | null> {
    return this.cluster.get(key);
  }

  async set(key: string, value: string, ttl?: number): Promise<void> {
    if (ttl) {
      await this.cluster.setex(key, ttl, value);
    } else {
      await this.cluster.set(key, value);
    }
  }

  async del(key: string): Promise<void> {
    await this.cluster.del(key);
  }

  async hget(key: string, field: string): Promise<string | null> {
    return this.cluster.hget(key, field);
  }

  async hset(key: string, field: string, value: string): Promise<void> {
    await this.cluster.hset(key, field, value);
  }

  async zadd(key: string, score: number, member: string): Promise<void> {
    await this.cluster.zadd(key, score, member);
  }

  async zrange(key: string, start: number, stop: number): Promise<string[]> {
    return this.cluster.zrange(key, start, stop);
  }

  async publish(channel: string, message: string): Promise<void> {
    await this.cluster.publish(channel, message);
  }

  async subscribe(channel: string, callback: (message: string) => void): Promise<void> {
    const subscriber = this.cluster.duplicate();
    await subscriber.subscribe(channel);
    subscriber.on('message', (ch, msg) => {
      if (ch === channel) callback(msg);
    });
  }
}
```

---

### Task 3: Distributed Caching

**Create `packages/server/src/cache/distributed-cache.service.ts`**:

```typescript
import { Injectable } from '@nestjs/common';
import { RedisClusterService } from './redis-cluster.service';

@Injectable()
export class DistributedCacheService {
  constructor(private redis: RedisClusterService) {}

  async cacheChannelMessages(channelId: string, messages: any[], ttl: number = 300) {
    const key = `channel:${channelId}:messages`;
    await this.redis.set(key, JSON.stringify(messages), ttl);
  }

  async getCachedChannelMessages(channelId: string): Promise<any[] | null> {
    const key = `channel:${channelId}:messages`;
    const cached = await this.redis.get(key);
    return cached ? JSON.parse(cached) : null;
  }

  async invalidateChannelCache(channelId: string) {
    const key = `channel:${channelId}:messages`;
    await this.redis.del(key);
  }

  async cacheUserPresence(userId: string, status: string, ttl: number = 60) {
    const key = `presence:${userId}`;
    await this.redis.set(key, status, ttl);
  }

  async getUserPresence(userId: string): Promise<string | null> {
    const key = `presence:${userId}`;
    return this.redis.get(key);
  }

  async setRateLimit(userId: string, action: string, limit: number, window: number): Promise<boolean> {
    const key = `ratelimit:${userId}:${action}`;
    const current = await this.redis.get(key);
    
    if (current && parseInt(current) >= limit) {
      return false; // Rate limit exceeded
    }

    await this.redis.set(key, current ? (parseInt(current) + 1).toString() : '1', window);
    return true;
  }
}
```

---

## Week 3: Multi-Region Deployment

### Task 1: Load Balancer Configuration

**NGINX Configuration** (`/etc/nginx/nginx.conf`):

```nginx
upstream api_servers {
    least_conn;
    server api-us-east-1.example.com:3000 weight=1;
    server api-us-west-1.example.com:3000 weight=1;
    server api-eu-west-1.example.com:3000 weight=1;
}

upstream ws_servers {
    ip_hash; # Sticky sessions for WebSocket
    server ws-us-east-1.example.com:3000;
    server ws-us-west-1.example.com:3000;
    server ws-eu-west-1.example.com:3000;
}

server {
    listen 80;
    server_name api.example.com;

    location /api/ {
        proxy_pass http://api_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /socket.io/ {
        proxy_pass http://ws_servers;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

---

### Task 2: Geographic Routing

**Route53 Geolocation Routing (AWS)**:

```typescript
// Terraform configuration
resource "aws_route53_record" "api_us" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"

  geolocation_routing_policy {
    continent = "NA"
  }

  alias {
    name                   = aws_lb.us_east.dns_name
    zone_id                = aws_lb.us_east.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "api_eu" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"

  geolocation_routing_policy {
    continent = "EU"
  }

  alias {
    name                   = aws_lb.eu_west.dns_name
    zone_id                = aws_lb.eu_west.zone_id
    evaluate_target_health = true
  }
}
```

---

### Task 3: Cross-Region Data Replication

**PostgreSQL Logical Replication**:

```sql
-- On primary database (us-east-1)
CREATE PUBLICATION chat_pub FOR ALL TABLES;

-- On replica database (eu-west-1)
CREATE SUBSCRIPTION chat_sub
  CONNECTION 'host=primary.us-east-1.rds.amazonaws.com port=5432 dbname=chatdb user=replicator password=xxx'
  PUBLICATION chat_pub;
```

**Read Replica Routing**:

```typescript
@Injectable()
export class DatabaseRouter {
  constructor(
    @InjectDataSource('primary') private primaryDb: DataSource,
    @InjectDataSource('replica') private replicaDb: DataSource,
  ) {}

  getDataSource(operation: 'read' | 'write'): DataSource {
    return operation === 'write' ? this.primaryDb : this.replicaDb;
  }
}

// Usage
const messages = await this.dbRouter
  .getDataSource('read')
  .getRepository(Message)
  .find({ where: { channelId } });
```

---

## Week 4: CDN & Performance Optimization

### Task 1: CloudFront CDN Setup

**CloudFront Distribution (Terraform)**:

```hcl
resource "aws_cloudfront_distribution" "cdn" {
  origin {
    domain_name = aws_s3_bucket.assets.bucket_regional_domain_name
    origin_id   = "S3-assets"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.oai.cloudfront_access_identity_path
    }
  }

  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"

  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-assets"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
    compress               = true
  }

  price_class = "PriceClass_All"

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    acm_certificate_arn = aws_acm_certificate.cert.arn
    ssl_support_method  = "sni-only"
  }
}
```

---

### Task 2: Image Optimization Service

**Create `packages/server/src/media/image-optimizer.service.ts`**:

```typescript
import { Injectable } from '@nestjs/common';
import sharp from 'sharp';

@Injectable()
export class ImageOptimizerService {
  async optimizeImage(buffer: Buffer, options: {
    width?: number;
    height?: number;
    quality?: number;
    format?: 'jpeg' | 'png' | 'webp';
  }): Promise<Buffer> {
    let image = sharp(buffer);

    if (options.width || options.height) {
      image = image.resize(options.width, options.height, {
        fit: 'inside',
        withoutEnlargement: true,
      });
    }

    const format = options.format || 'jpeg';
    const quality = options.quality || 80;

    switch (format) {
      case 'jpeg':
        image = image.jpeg({ quality, progressive: true });
        break;
      case 'png':
        image = image.png({ quality, compressionLevel: 9 });
        break;
      case 'webp':
        image = image.webp({ quality });
        break;
    }

    return image.toBuffer();
  }

  async generateThumbnails(buffer: Buffer): Promise<{
    small: Buffer;
    medium: Buffer;
    large: Buffer;
  }> {
    const [small, medium, large] = await Promise.all([
      this.optimizeImage(buffer, { width: 150, height: 150, format: 'webp' }),
      this.optimizeImage(buffer, { width: 400, height: 400, format: 'webp' }),
      this.optimizeImage(buffer, { width: 1200, height: 1200, format: 'webp' }),
    ]);

    return { small, medium, large };
  }
}
```

---

### Task 3: Database Query Optimization

**Add Indexes**:

```sql
-- Message queries
CREATE INDEX CONCURRENTLY idx_messages_channel_created 
  ON messages(channel_id, created_at DESC);

CREATE INDEX CONCURRENTLY idx_messages_user_created 
  ON messages(user_id, created_at DESC);

-- Full-text search
CREATE INDEX CONCURRENTLY idx_messages_text_gin 
  ON messages USING gin(to_tsvector('english', text));

-- Partial indexes
CREATE INDEX CONCURRENTLY idx_messages_unread 
  ON messages(channel_id, created_at) 
  WHERE is_read = false;

-- Analyze tables
ANALYZE messages;
ANALYZE channels;
ANALYZE users;
```

**Query Optimization**:

```typescript
// Before: N+1 query problem
const channels = await channelRepo.find();
for (const channel of channels) {
  channel.lastMessage = await messageRepo.findOne({
    where: { channelId: channel.id },
    order: { createdAt: 'DESC' },
  });
}

// After: Single query with JOIN
const channels = await channelRepo
  .createQueryBuilder('channel')
  .leftJoinAndSelect(
    'messages',
    'lastMessage',
    'lastMessage.id = (SELECT id FROM messages WHERE channel_id = channel.id ORDER BY created_at DESC LIMIT 1)'
  )
  .getMany();
```

---

### Task 4: Connection Pooling

**Configure Connection Pool**:

```typescript
const dataSource = new DataSource({
  type: 'postgres',
  host: 'localhost',
  port: 5432,
  username: 'user',
  password: 'pass',
  database: 'chatdb',
  extra: {
    max: 20, // Maximum pool size
    min: 5,  // Minimum pool size
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 2000,
  },
});
```

---

## Performance Monitoring

**Create `packages/server/src/monitoring/performance.interceptor.ts`**:

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class PerformanceInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const start = Date.now();
    const request = context.switchToHttp().getRequest();

    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - start;
        
        if (duration > 1000) {
          console.warn(`Slow request: ${request.method} ${request.url} - ${duration}ms`);
        }

        // Send to monitoring service (Prometheus, DataDog, etc.)
      }),
    );
  }
}
```

---

## Deliverables

- [x] Database sharding by team ID
- [x] Multi-database connection manager
- [x] Sharded repository pattern
- [x] Redis cluster setup
- [x] Distributed caching
- [x] Rate limiting with Redis
- [x] Multi-region deployment
- [x] Load balancer configuration
- [x] Geographic routing
- [x] Read replicas
- [x] CDN integration
- [x] Image optimization
- [x] Database query optimization
- [x] Connection pooling
- [x] Performance monitoring

**Next**: [MONTH_12_ENTERPRISE.md](./MONTH_12_ENTERPRISE.md)
