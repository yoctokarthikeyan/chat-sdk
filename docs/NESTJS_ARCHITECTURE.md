# NestJS Architecture Guide - Request Flow Pattern

**Purpose**: Document the layered architecture pattern used in NestJS applications  
**Pattern**: Controller → Service → DAO → Mapper  
**Reference Implementation**: Dropper App Module

---

## 🏗️ Architecture Overview

Our NestJS applications follow a **layered architecture** pattern that separates concerns and promotes maintainability:

```
┌─────────────────────────────────────────────────────────────┐
│                        HTTP REQUEST                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  CONTROLLER LAYER                                           │
│  - Route handling                                           │
│  - Request validation (DTOs)                                │
│  - Authentication & Authorization                           │
│  - Swagger documentation                                    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  SERVICE LAYER (Business Logic)                             │
│  - Business rules & validation                              │
│  - Orchestration of multiple DAOs                           │
│  - Transaction management                                   │
│  - Error handling                                           │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  DAO LAYER (Data Access Object)                             │
│  - Database queries (Prisma)                                │
│  - CRUD operations                                          │
│  - Query building                                           │
│  - No business logic                                        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  MAPPER LAYER                                               │
│  - Prisma ↔ Domain Entity                                   │
│  - Domain Entity ↔ Response DTO                             │
│  - Data transformation                                      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                       HTTP RESPONSE                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 📁 Module Structure

```
src/modules/app/
├── app.module.ts              # Module definition
├── controllers/
│   └── app.controller.ts      # HTTP endpoints
├── services/
│   └── app.service.ts         # Business logic
├── dao/
│   └── app.dao.ts             # Database access
├── mappers/
│   └── app.mapper.ts          # Data transformation
├── entities/
│   └── app.entity.ts          # Domain entity
└── dto/
    ├── request/
    │   ├── create-app.dto.ts  # Input validation
    │   ├── update-app.dto.ts
    │   └── app-query.dto.ts
    └── response/
        ├── app-response.dto.ts       # Output format
        ├── app-with-keys-response.dto.ts
        └── app-list-response.dto.ts
```

---

## 🔄 Complete Request Flow Example

Let's trace a **"Create App"** request through all layers:

### 1️⃣ **Controller Layer** - HTTP Entry Point

**File**: `controllers/app.controller.ts`

```typescript
import { Body, Controller, Post, HttpStatus } from '@nestjs/common';
import { ApiTags, ApiBearerAuth, ApiOperation, ApiResponse } from '@nestjs/swagger';
import { CurrentUser } from '../../../common/decorators/current-user.decorator';
import { RequirePermission } from '../../../common/decorators/permissions.decorator';
import { CreateAppDto } from '../dto/request/create-app.dto';
import { AppResponseDto } from '../dto/response/app-response.dto';
import { AppService } from '../services/app.service';

@ApiTags('Apps')
@ApiBearerAuth()
@Controller('apps')
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Post()
  @RequirePermission('create', 'App')
  @ApiOperation({ summary: 'Create a new app' })
  @ApiResponse({
    status: HttpStatus.CREATED,
    description: 'App created successfully',
    type: AppResponseDto,
  })
  async create(
    @CurrentUser() user: UserContext,
    @Body() createAppDto: CreateAppDto,
  ): Promise<AppResponseDto> {
    // ✅ Controller responsibilities:
    // 1. Extract user context from JWT
    // 2. Validate request body (automatic via CreateAppDto)
    // 3. Check permissions (via @RequirePermission decorator)
    // 4. Delegate to service layer
    // 5. Return response (auto-serialized to AppResponseDto)
    
    return this.appService.create(user.customerId, createAppDto);
  }
}
```

**Controller Responsibilities:**
- ✅ Route definition (`@Post()`)
- ✅ Request validation (via DTOs)
- ✅ Authentication & authorization (via decorators)
- ✅ Swagger documentation
- ✅ Delegate to service layer
- ❌ **NO business logic**
- ❌ **NO database access**

---

### 2️⃣ **Service Layer** - Business Logic

**File**: `services/app.service.ts`

```typescript
import { Injectable, ConflictException, Logger } from '@nestjs/common';
import { AppDao } from '../dao/app.dao';
import { AppMapper } from '../mappers/app.mapper';
import { CreateAppDto } from '../dto/request/create-app.dto';
import { AppResponseDto } from '../dto/response/app-response.dto';
import { AppEntity } from '../entities/app.entity';

@Injectable()
export class AppService {
  private readonly logger = new Logger(AppService.name);

  constructor(
    private readonly appDao: AppDao,
    private readonly appMapper: AppMapper,
    private readonly apiKeyService: ApiKeyService,
  ) {}

  async create(
    customerId: string,
    createAppDto: CreateAppDto,
  ): Promise<AppResponseDto> {
    // ✅ Business Logic Step 1: Check if app name already exists
    const exists = await this.appDao.existsByNameAndCustomer(
      createAppDto.name,
      customerId,
    );

    if (exists) {
      throw new ConflictException(
        `An app with the name "${createAppDto.name}" already exists`,
      );
    }

    // ✅ Business Logic Step 2: Determine environment (defaults to production)
    const environment = createAppDto.environment || AppEnvironment.PRODUCTION;

    // ✅ Business Logic Step 3: Create domain entity
    const appEntity: Partial<AppEntity> = {
      ...createAppDto,
      environment,
    };

    // ✅ Step 4: Map to Prisma input and create via DAO
    const createInput = this.appMapper.toCreateInput(appEntity, customerId);
    const createdApp = await this.appDao.create(createInput);

    // ✅ Business Logic Step 5: Generate API keys (orchestrate multiple services)
    let apiKeys: { publishableKey: string; secretKey: string } | undefined;

    try {
      apiKeys = await this.apiKeyService.createKeysForApp(
        createdApp.id,
        environment,
      );
      
      this.logger.log(
        `Generated API keys for app ${createdApp.id} (${environment})`,
      );
    } catch (error) {
      this.logger.error('Failed to generate API keys for new app', error);
      // Don't fail app creation if key generation fails
    }

    // ✅ Step 6: Map Prisma entity → Domain entity → Response DTO
    const domainEntity = this.appMapper.toDomain(createdApp);

    // ✅ Step 7: Return appropriate response
    if (apiKeys) {
      return this.appMapper.toResponseDtoWithKeys(domainEntity, apiKeys);
    }

    return this.appMapper.toResponseDto(domainEntity);
  }
}
```

**Service Responsibilities:**
- ✅ Business rules & validation
- ✅ Orchestration of multiple DAOs/services
- ✅ Transaction management
- ✅ Error handling & logging
- ✅ Domain entity creation
- ✅ Use mappers for data transformation
- ❌ **NO direct database queries**
- ❌ **NO HTTP concerns**

---

### 3️⃣ **DAO Layer** - Database Access

**File**: `dao/app.dao.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../../../prisma/prisma.service';
import { App, Prisma } from '@prisma/client';

export interface PaginationOptions {
  skip: number;
  take: number;
}

@Injectable()
export class AppDao {
  constructor(private readonly prisma: PrismaService) {}

  // ✅ Create operation
  async create(data: Prisma.AppCreateInput): Promise<App> {
    return this.prisma.app.create({ data });
  }

  // ✅ Read operation
  async findById(id: string): Promise<App | null> {
    return this.prisma.app.findUnique({
      where: { id },
      include: {
        customer: true,
      },
    });
  }

  // ✅ Read many with pagination
  async findMany(
    where: Prisma.AppWhereInput,
    pagination: PaginationOptions,
  ): Promise<App[]> {
    return this.prisma.app.findMany({
      where,
      skip: pagination.skip,
      take: pagination.take,
      orderBy: { createdAt: 'desc' },
    });
  }

  // ✅ Count operation
  async count(where: Prisma.AppWhereInput): Promise<number> {
    return this.prisma.app.count({ where });
  }

  // ✅ Update operation
  async update(id: string, data: Prisma.AppUpdateInput): Promise<App> {
    return this.prisma.app.update({
      where: { id },
      data,
    });
  }

  // ✅ Delete operation
  async delete(id: string): Promise<App> {
    return this.prisma.app.delete({
      where: { id },
    });
  }

  // ✅ Custom query for business logic
  async existsByNameAndCustomer(
    name: string,
    customerId: string,
    excludeId?: string,
  ): Promise<boolean> {
    const count = await this.prisma.app.count({
      where: {
        name,
        customerId,
        ...(excludeId && { id: { not: excludeId } }),
      },
    });

    return count > 0;
  }
}
```

**DAO Responsibilities:**
- ✅ Database queries (Prisma)
- ✅ CRUD operations
- ✅ Query building
- ✅ Return Prisma types
- ❌ **NO business logic**
- ❌ **NO data transformation**
- ❌ **NO error handling** (let Prisma errors bubble up)

---

### 4️⃣ **Mapper Layer** - Data Transformation

**File**: `mappers/app.mapper.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { App, Prisma } from '@prisma/client';
import { plainToClass } from 'class-transformer';
import { AppEntity } from '../entities/app.entity';
import { AppResponseDto } from '../dto/response/app-response.dto';
import { AppWithKeysResponseDto } from '../dto/response/app-with-keys-response.dto';

@Injectable()
export class AppMapper {
  /**
   * Map Prisma entity → Domain entity
   * Purpose: Convert database model to business domain model
   */
  toDomain(prismaApp: App): AppEntity {
    const entity = new AppEntity();
    entity.id = prismaApp.id;
    entity.name = prismaApp.name;
    entity.description = prismaApp.description ?? undefined;
    entity.environment = prismaApp.environment;
    entity.settings = prismaApp.settings as Record<string, any>;
    entity.customerId = prismaApp.customerId;
    entity.createdAt = prismaApp.createdAt;
    entity.updatedAt = prismaApp.updatedAt;
    return entity;
  }

  /**
   * Map Domain entity → Response DTO
   * Purpose: Convert business model to API response
   */
  toResponseDto(entity: AppEntity): AppResponseDto {
    return plainToClass(AppResponseDto, entity, {
      excludeExtraneousValues: true, // Only @Expose() fields
    });
  }

  /**
   * Map Domain entity → Response DTO with API keys
   * Purpose: Include sensitive data only when appropriate
   */
  toResponseDtoWithKeys(
    entity: AppEntity,
    keys: { publishableKey: string; secretKey: string },
  ): AppWithKeysResponseDto {
    const response = this.toResponseDto(entity);
    return plainToClass(
      AppWithKeysResponseDto,
      { ...response, ...keys },
      { excludeExtraneousValues: true },
    );
  }

  /**
   * Map array of Prisma entities → array of Response DTOs
   * Purpose: Bulk transformation for list endpoints
   */
  toResponseDtoList(prismaApps: App[]): AppResponseDto[] {
    return prismaApps.map((app) => {
      const entity = this.toDomain(app);
      return this.toResponseDto(entity);
    });
  }

  /**
   * Map Domain entity → Prisma create input
   * Purpose: Convert business model to database create format
   */
  toCreateInput(
    entity: Partial<AppEntity>,
    customerId: string,
  ): Prisma.AppCreateInput {
    return {
      name: entity.name!,
      description: entity.description,
      environment: entity.environment,
      settings: entity.settings,
      customer: {
        connect: { id: customerId },
      },
    };
  }

  /**
   * Map Domain entity → Prisma update input
   * Purpose: Convert business model to database update format
   */
  toUpdateInput(entity: Partial<AppEntity>): Prisma.AppUpdateInput {
    const data: Prisma.AppUpdateInput = {};

    if (entity.name !== undefined) data.name = entity.name;
    if (entity.description !== undefined) data.description = entity.description;
    if (entity.settings !== undefined) data.settings = entity.settings;

    return data;
  }
}
```

**Mapper Responsibilities:**
- ✅ Prisma entity ↔ Domain entity conversion
- ✅ Domain entity ↔ Response DTO conversion
- ✅ Domain entity → Prisma input conversion
- ✅ Handle null/undefined transformations
- ✅ Use `class-transformer` for DTO serialization
- ❌ **NO business logic**
- ❌ **NO database access**

---

#### **Advanced: Composition Pattern for Nested Entities**

When working with Prisma's `include` for related entities, use the **Composition Pattern** instead of recursion.

**Problem**: How to map entities with nested relations (e.g., Post with Author)?

```typescript
// Prisma query with include
const post = await prisma.post.findUnique({
  where: { id },
  include: {
    author: true,        // User entity
    comments: true,      // Comment[] entities
  },
});
```

**Solution**: Compose mappers by injecting related mappers as dependencies.

##### **Example: PostMapper with UserMapper**

**Step 1: Base Mapper (No Dependencies)**

```typescript
// src/modules/user/mappers/user.mapper.ts
@Injectable()
export class UserMapper {
  toDomain(prismaUser: User): UserEntity {
    const entity = new UserEntity();
    entity.id = prismaUser.id;
    entity.email = prismaUser.email;
    entity.displayName = prismaUser.displayName;
    entity.avatarUrl = prismaUser.avatarUrl;
    return entity;
  }

  toResponseDto(entity: UserEntity): UserResponseDto {
    return plainToClass(UserResponseDto, entity, {
      excludeExtraneousValues: true,
    });
  }
}
```

**Step 2: Mapper with Relations (Composition)**

```typescript
// src/modules/post/mappers/post.mapper.ts
import { Injectable } from '@nestjs/common';
import { Post, User } from '@prisma/client';
import { UserMapper } from '../../user/mappers/user.mapper';

// Type for Post with included author
type PostWithAuthor = Post & {
  author: User;
};

@Injectable()
export class PostMapper {
  constructor(
    private readonly userMapper: UserMapper, // ✅ Inject related mapper
  ) {}

  /**
   * Map Post without relations
   */
  toDomain(prismaPost: Post): PostEntity {
    const entity = new PostEntity();
    entity.id = prismaPost.id;
    entity.title = prismaPost.title;
    entity.content = prismaPost.content;
    entity.authorId = prismaPost.authorId;
    return entity;
  }

  toResponseDto(entity: PostEntity): PostResponseDto {
    return plainToClass(PostResponseDto, entity, {
      excludeExtraneousValues: true,
    });
  }

  /**
   * Map Post with Author (nested relation)
   * ✅ This handles Prisma's include
   */
  toResponseDtoWithAuthor(prismaPost: PostWithAuthor): PostWithAuthorResponseDto {
    // Map the post itself
    const postEntity = this.toDomain(prismaPost);
    const postDto = this.toResponseDto(postEntity);

    // Use UserMapper for the author relation
    const authorEntity = this.userMapper.toDomain(prismaPost.author);
    const authorDto = this.userMapper.toResponseDto(authorEntity);

    // Combine into nested DTO
    return plainToClass(
      PostWithAuthorResponseDto,
      {
        ...postDto,
        author: authorDto, // ✅ Nested relation
      },
      { excludeExtraneousValues: true },
    );
  }

  /**
   * Map array of Posts with Authors
   */
  toResponseDtoListWithAuthor(
    prismaPosts: PostWithAuthor[],
  ): PostWithAuthorResponseDto[] {
    return prismaPosts.map((post) => this.toResponseDtoWithAuthor(post));
  }
}
```

**Step 3: DTO with Nested Relation**

```typescript
// src/modules/post/dto/response/post-with-author-response.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { Exclude, Expose, Type } from 'class-transformer';
import { UserResponseDto } from '../../../user/dto/response/user-response.dto';

@Exclude()
export class PostWithAuthorResponseDto {
  @Expose()
  @ApiProperty()
  id: string;

  @Expose()
  @ApiProperty()
  title: string;

  @Expose()
  @ApiProperty()
  content: string;

  @Expose()
  @ApiProperty()
  authorId: string;

  @Expose()
  @ApiProperty({ type: () => UserResponseDto }) // ✅ Nested DTO
  @Type(() => UserResponseDto) // ✅ Required for class-transformer
  author: UserResponseDto;

  @Expose()
  @ApiProperty()
  createdAt: Date;
}
```

**Step 4: DAO with Include**

```typescript
// src/modules/post/dao/post.dao.ts
type PostWithAuthor = Post & {
  author: User;
};

@Injectable()
export class PostDao {
  constructor(private readonly prisma: PrismaService) {}

  async findByIdWithAuthor(id: string): Promise<PostWithAuthor | null> {
    return this.prisma.post.findUnique({
      where: { id },
      include: {
        author: true, // ✅ Prisma include
      },
    });
  }
}
```

**Step 5: Service Usage**

```typescript
// src/modules/post/services/post.service.ts
@Injectable()
export class PostService {
  constructor(
    private readonly postDao: PostDao,
    private readonly postMapper: PostMapper,
  ) {}

  async findByIdWithAuthor(id: string): Promise<PostWithAuthorResponseDto> {
    const post = await this.postDao.findByIdWithAuthor(id);
    
    if (!post) {
      throw new NotFoundException('Post not found');
    }

    // Mapper handles the nested mapping
    return this.postMapper.toResponseDtoWithAuthor(post);
  }
}
```

**Step 6: Module Registration**

```typescript
// user.module.ts
@Module({
  providers: [UserMapper],
  exports: [UserMapper], // ✅ Export so PostModule can use it
})
export class UserModule {}

// post.module.ts
@Module({
  imports: [UserModule], // ✅ Import UserModule to access UserMapper
  providers: [PostService, PostDao, PostMapper],
  exports: [PostMapper],
})
export class PostModule {}
```

##### **Pattern: Multiple Relations**

```typescript
type PostWithRelations = Post & {
  author: User;
  comments: Comment[];
  tags: Tag[];
};

@Injectable()
export class PostMapper {
  constructor(
    private readonly userMapper: UserMapper,
    private readonly commentMapper: CommentMapper,
    private readonly tagMapper: TagMapper,
  ) {}

  toResponseDtoFull(prismaPost: PostWithRelations): PostFullResponseDto {
    const postDto = this.toResponseDto(this.toDomain(prismaPost));

    return plainToClass(
      PostFullResponseDto,
      {
        ...postDto,
        author: this.mapAuthor(prismaPost.author),
        comments: this.mapComments(prismaPost.comments),
        tags: this.mapTags(prismaPost.tags),
      },
      { excludeExtraneousValues: true },
    );
  }

  // Helper methods for clean code
  private mapAuthor(author: User): UserResponseDto {
    return this.userMapper.toResponseDto(this.userMapper.toDomain(author));
  }

  private mapComments(comments: Comment[]): CommentResponseDto[] {
    return comments.map(c => 
      this.commentMapper.toResponseDto(this.commentMapper.toDomain(c))
    );
  }

  private mapTags(tags: Tag[]): TagResponseDto[] {
    return tags.map(t => 
      this.tagMapper.toResponseDto(this.tagMapper.toDomain(t))
    );
  }
}
```

##### **Pattern: Optional Relations**

```typescript
// Single DTO with optional nested fields
@Exclude()
export class PostResponseDto {
  @Expose()
  id: string;

  @Expose()
  title: string;

  @Expose()
  @Type(() => UserResponseDto)
  author?: UserResponseDto; // ✅ Optional relation

  @Expose()
  @Type(() => CommentResponseDto)
  comments?: CommentResponseDto[]; // ✅ Optional array relation
}

// Mapper handles optionals
toResponseDto(
  post: Post & { author?: User; comments?: Comment[] }
): PostResponseDto {
  const dto = plainToClass(PostResponseDto, this.toDomain(post), {
    excludeExtraneousValues: true,
  });

  // Map optional relations if present
  if (post.author) {
    dto.author = this.userMapper.toResponseDto(
      this.userMapper.toDomain(post.author)
    );
  }

  if (post.comments) {
    dto.comments = post.comments.map(c =>
      this.commentMapper.toResponseDto(this.commentMapper.toDomain(c))
    );
  }

  return dto;
}
```

##### **Composition Pattern Benefits**

✅ **Type Safe** - Full TypeScript support  
✅ **Reusable** - Mappers can be used independently or composed  
✅ **Testable** - Each mapper can be tested in isolation  
✅ **Maintainable** - Easy to update individual mappers  
✅ **No Recursion Issues** - Explicit composition prevents infinite loops  
✅ **Clear Dependencies** - Easy to see what each mapper needs  

##### **Anti-Pattern: Avoid Recursion**

```typescript
// ❌ BAD: Recursive mapping can cause infinite loops
toResponseDto(entity: any): any {
  const dto = { ...entity };
  
  for (const key in entity) {
    if (typeof entity[key] === 'object') {
      dto[key] = this.toResponseDto(entity[key]); // ❌ Infinite loop risk!
    }
  }
  
  return dto;
}
```

**Problem**: Can cause infinite loops with circular references (User → Post → User)

##### **Best Practices for Nested Mappers**

1. **One mapper per entity** - Keep mappers focused
2. **Inject related mappers** - Use dependency injection
3. **Create specific methods** - One method per include combination
   - `toResponseDto()` - No relations
   - `toResponseDtoWithAuthor()` - With author
   - `toResponseDtoFull()` - With all relations
4. **Use separate DTOs** - Different includes = different DTOs (usually)
5. **Add `@Type()` decorator** - Required for nested DTOs in class-transformer
6. **Export mappers** - So other modules can use them
7. **Type your includes** - Use TypeScript types for Prisma includes

---

### 5️⃣ **Entity Layer** - Domain Model

**File**: `entities/app.entity.ts`

```typescript
import { AppEnvironment } from '@prisma/client';

export class AppEntity {
  id: string;
  name: string;
  description?: string;
  environment: AppEnvironment;
  settings?: Record<string, any>;
  customerId: string;
  createdAt: Date;
  updatedAt: Date;

  // ✅ Domain methods (business logic that belongs to the entity)
  isTestEnvironment(): boolean {
    return this.environment === AppEnvironment.TEST;
  }

  isProductionEnvironment(): boolean {
    return this.environment === AppEnvironment.PRODUCTION;
  }

  shouldBillUsage(): boolean {
    // Only bill production environments
    return this.isProductionEnvironment();
  }
}
```

**Entity Responsibilities:**
- ✅ Domain model definition
- ✅ Domain methods (entity-specific logic)
- ✅ Type safety
- ❌ **NO database concerns**
- ❌ **NO HTTP concerns**

---

### 6️⃣ **DTO Layer** - Data Transfer Objects

#### **Request DTO** (Input Validation)

**File**: `dto/request/create-app.dto.ts`

```typescript
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { AppEnvironment } from '@prisma/client';
import {
  IsEnum,
  IsObject,
  IsOptional,
  IsString,
  MaxLength,
  MinLength,
} from 'class-validator';

export class CreateAppDto {
  @ApiProperty({ description: 'App name', example: 'My Media App' })
  @IsString()
  @MinLength(1)
  @MaxLength(100)
  name: string;

  @ApiPropertyOptional({
    description: 'App description',
    example: 'Handles media uploads for our platform',
  })
  @IsString()
  @IsOptional()
  @MaxLength(500)
  description?: string;

  @ApiProperty({
    description: 'App environment type',
    enum: AppEnvironment,
    example: AppEnvironment.PRODUCTION,
    default: AppEnvironment.PRODUCTION,
  })
  @IsEnum(AppEnvironment)
  @IsOptional()
  environment?: AppEnvironment;

  @ApiPropertyOptional({ description: 'App settings (JSON object)' })
  @IsObject()
  @IsOptional()
  settings?: Record<string, any>;
}
```

#### **Response DTO** (Output Format)

**File**: `dto/response/app-response.dto.ts`

```typescript
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { AppEnvironment } from '@prisma/client';
import { Exclude, Expose } from 'class-transformer';

@Exclude() // Exclude all by default
export class AppResponseDto {
  @Expose() // Only expose explicitly marked fields
  @ApiProperty({ description: 'App ID' })
  id: string;

  @Expose()
  @ApiProperty({ description: 'App name' })
  name: string;

  @Expose()
  @ApiPropertyOptional({ description: 'App description' })
  description?: string;

  @Expose()
  @ApiProperty({ description: 'Environment type: production or test' })
  environment: AppEnvironment;

  @Expose()
  @ApiPropertyOptional({ description: 'App settings' })
  settings?: Record<string, any>;

  @Expose()
  @ApiProperty({ description: 'Customer ID who owns this app' })
  customerId: string;

  @Expose()
  @ApiProperty({ description: 'Creation timestamp' })
  createdAt: Date;

  @Expose()
  @ApiProperty({ description: 'Last update timestamp' })
  updatedAt: Date;
}
```

**DTO Responsibilities:**
- ✅ Input validation (`class-validator`)
- ✅ Output serialization (`class-transformer`)
- ✅ Swagger documentation
- ✅ Type safety
- ❌ **NO business logic**

---

## 🔄 Complete Flow Diagram

```
POST /apps
{
  "name": "My App",
  "environment": "PRODUCTION"
}
        ↓
┌─────────────────────────────────────────────────────────────┐
│ 1. CONTROLLER (app.controller.ts)                          │
│    - Validate CreateAppDto                                  │
│    - Check @RequirePermission('create', 'App')              │
│    - Extract user from JWT via @CurrentUser()               │
│    - Call: appService.create(customerId, createAppDto)      │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. SERVICE (app.service.ts)                                │
│    - Check if name exists: appDao.existsByNameAndCustomer() │
│    - Throw ConflictException if exists                      │
│    - Set default environment                                │
│    - Create AppEntity                                       │
│    - Map to Prisma input: appMapper.toCreateInput()         │
│    - Create in DB: appDao.create()                          │
│    - Generate API keys: apiKeyService.createKeysForApp()    │
│    - Map to response: appMapper.toResponseDto()             │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. DAO (app.dao.ts)                                        │
│    - Execute: prisma.app.create({ data })                   │
│    - Return: App (Prisma type)                              │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. MAPPER (app.mapper.ts)                                  │
│    - Prisma App → AppEntity: toDomain()                     │
│    - AppEntity → AppResponseDto: toResponseDto()            │
│    - Use class-transformer for serialization                │
└─────────────────────────────────────────────────────────────┘
        ↓
HTTP 201 Created
{
  "id": "uuid",
  "name": "My App",
  "environment": "PRODUCTION",
  "customerId": "uuid",
  "createdAt": "2025-10-30T...",
  "updatedAt": "2025-10-30T..."
}
```

---

## 📋 Layer Responsibilities Summary

| Layer | Responsibilities | What NOT to do |
|-------|-----------------|----------------|
| **Controller** | • Route handling<br>• Request validation<br>• Auth/permissions<br>• Swagger docs<br>• Delegate to service | • Business logic<br>• Database access<br>• Data transformation |
| **Service** | • Business rules<br>• Orchestration<br>• Transactions<br>• Error handling<br>• Logging | • HTTP concerns<br>• Direct DB queries<br>• Data mapping |
| **DAO** | • Database queries<br>• CRUD operations<br>• Query building<br>• Return Prisma types | • Business logic<br>• Data transformation<br>• Error handling |
| **Mapper** | • Prisma ↔ Entity<br>• Entity ↔ DTO<br>• Data transformation<br>• Serialization<br>• **Compose other mappers** | • Business logic<br>• Database access<br>• Validation<br>• Recursive mapping |
| **Entity** | • Domain model<br>• Domain methods<br>• Type safety | • Database concerns<br>• HTTP concerns<br>• Validation |
| **DTO** | • Input validation<br>• Output format<br>• Swagger docs<br>• Type safety | • Business logic<br>• Database access |

---

## 🎯 Best Practices

### ✅ DO:

1. **Keep layers separate** - Each layer has a single responsibility
2. **Use dependency injection** - Inject services, DAOs, mappers via constructor
3. **Use DTOs for validation** - Validate input at controller level
4. **Use mappers for transformation** - Never expose Prisma types directly
5. **Compose mappers for nested entities** - Inject related mappers, don't use recursion
6. **Handle errors in service layer** - Throw appropriate NestJS exceptions
7. **Use domain entities** - Represent business logic in entities
8. **Log important operations** - Use NestJS Logger in service layer
9. **Use transactions** - For multi-step operations in service layer
10. **Document with Swagger** - Use decorators for API documentation
11. **Use type safety** - Leverage TypeScript everywhere
12. **Create specific mapper methods** - One method per include combination

### ❌ DON'T:

1. **Don't put business logic in controllers** - Keep controllers thin
2. **Don't access database from controllers** - Always go through service
3. **Don't expose Prisma types** - Always map to DTOs
4. **Don't put business logic in DAOs** - Keep DAOs pure data access
5. **Don't handle HTTP in services** - Services should be HTTP-agnostic
6. **Don't skip validation** - Always validate input with DTOs
7. **Don't return sensitive data** - Use @Exclude() in response DTOs
8. **Don't mix concerns** - Each layer has one job
9. **Don't bypass layers** - Follow the flow: Controller → Service → DAO
10. **Don't forget error handling** - Handle errors appropriately at each layer
11. **Don't use recursive mapping** - Use composition pattern for nested entities

---

## 🔐 Security Patterns

### 1. **Permission Checking**

```typescript
@RequirePermission('create', 'App')
async create(@CurrentUser() user: UserContext, @Body() dto: CreateAppDto) {
  // Permission already checked by decorator
  return this.appService.create(user.customerId, dto);
}
```

### 2. **Data Isolation**

```typescript
// Service ensures data belongs to customer
async findById(id: string, customerId: string): Promise<AppResponseDto> {
  const app = await this.appDao.findById(id);
  
  if (!app || app.customerId !== customerId) {
    throw new NotFoundException(`App with ID "${id}" not found`);
  }
  
  return this.appMapper.toResponseDto(app);
}
```

### 3. **Sensitive Data Protection**

```typescript
// Use @Exclude() and @Expose() to control what's returned
@Exclude()
export class AppResponseDto {
  @Expose()
  id: string;
  
  @Expose()
  name: string;
  
  // passwordHash is NOT exposed (no @Expose())
}
```

---

## 📊 Example: Complex Query with CASL Permissions

```typescript
// Service Layer
async findAll(
  user: UserContext,
  ability: AppAbility,
  query: AppQueryDto,
): Promise<AppListResponseDto> {
  const { page = 1, limit = 10, search } = query;

  // Build base filters
  const baseWhere: Prisma.AppWhereInput = {
    customerId: user.customerId,
    ...(search && {
      OR: [
        { name: { contains: search, mode: 'insensitive' } },
        { description: { contains: search, mode: 'insensitive' } },
      ],
    }),
  };

  // Get CASL permission conditions
  // Super Admin: No restrictions
  // Owner/Admin: customerId filter
  // Developer/Viewer: appId IN (user.appIds)
  const caslWhere = accessibleBy(ability, 'read').App;

  // Merge filters
  const where: Prisma.AppWhereInput = caslWhere
    ? { AND: [baseWhere, caslWhere] }
    : baseWhere;

  const pagination = { skip: (page - 1) * limit, take: limit };

  // Execute queries via DAO
  const [apps, total] = await Promise.all([
    this.appDao.findMany(where, pagination),
    this.appDao.count(where),
  ]);

  // Map to response DTOs
  const data = this.appMapper.toResponseDtoList(apps);

  return {
    data,
    total,
    page,
    limit,
    totalPages: Math.ceil(total / limit),
  };
}
```

---

## 🧪 Testing Strategy

### **Controller Tests** (E2E)
```typescript
describe('AppController (e2e)', () => {
  it('POST /apps - should create app', async () => {
    return request(app.getHttpServer())
      .post('/apps')
      .set('Authorization', `Bearer ${token}`)
      .send({ name: 'Test App' })
      .expect(201)
      .expect((res) => {
        expect(res.body.name).toBe('Test App');
      });
  });
});
```

### **Service Tests** (Unit)
```typescript
describe('AppService', () => {
  it('should throw ConflictException if name exists', async () => {
    jest.spyOn(appDao, 'existsByNameAndCustomer').mockResolvedValue(true);
    
    await expect(
      service.create('customerId', { name: 'Existing' }),
    ).rejects.toThrow(ConflictException);
  });
});
```

### **DAO Tests** (Integration)
```typescript
describe('AppDao', () => {
  it('should create app', async () => {
    const app = await dao.create({
      name: 'Test',
      customer: { connect: { id: 'customerId' } },
    });
    
    expect(app.name).toBe('Test');
  });
});
```

---

## 📚 Summary

This architecture provides:

✅ **Separation of Concerns** - Each layer has a clear responsibility  
✅ **Testability** - Easy to unit test each layer independently  
✅ **Maintainability** - Changes are isolated to specific layers  
✅ **Scalability** - Easy to add new features without breaking existing code  
✅ **Type Safety** - TypeScript throughout the stack  
✅ **Security** - Permission checks, data isolation, sensitive data protection  
✅ **Documentation** - Swagger auto-generated from decorators  

**Remember**: Follow the flow, respect the boundaries, and keep each layer focused on its single responsibility!
