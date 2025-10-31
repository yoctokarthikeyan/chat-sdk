# Why Mappers are @Injectable() (Not Static Classes)

**Question**: Why mark Mapper as `@Injectable()`? Wouldn't a static class work?  
**Short Answer**: No! `@Injectable()` enables dependency injection, which is crucial for the Composition Pattern.

---

## 🎯 The Core Reason: Dependency Injection

### **Problem with Static Classes**

```typescript
// ❌ BAD: Static class cannot inject dependencies
export class PostMapper {
  static toResponseDtoWithAuthor(post: PostWithAuthor): PostWithAuthorDto {
    // ❌ How do we access UserMapper here?
    // We can't inject it into a static class!
    
    // Option 1: Import and use statically (tight coupling)
    const authorDto = UserMapper.toResponseDto(post.author); // ❌ Tight coupling
    
    // Option 2: Create new instance every time (inefficient)
    const userMapper = new UserMapper(); // ❌ No DI, inefficient
    const authorDto = userMapper.toResponseDto(post.author);
    
    return { ...postDto, author: authorDto };
  }
}
```

### **Solution with @Injectable()**

```typescript
// ✅ GOOD: Injectable allows dependency injection
@Injectable()
export class PostMapper {
  constructor(
    private readonly userMapper: UserMapper, // ✅ Injected by NestJS
  ) {}

  toResponseDtoWithAuthor(post: PostWithAuthor): PostWithAuthorDto {
    // ✅ Use injected mapper (loose coupling, testable)
    const authorDto = this.userMapper.toResponseDto(
      this.userMapper.toDomain(post.author)
    );
    
    return plainToClass(PostWithAuthorDto, {
      ...postDto,
      author: authorDto,
    });
  }
}
```

---

## 📋 Detailed Comparison

| Aspect | @Injectable() ✅ | Static Class ❌ |
|--------|-----------------|----------------|
| **Dependency Injection** | ✅ Can inject other mappers | ❌ Cannot inject dependencies |
| **Composition Pattern** | ✅ Easy to compose mappers | ❌ Tight coupling or manual instantiation |
| **Testability** | ✅ Easy to mock dependencies | ❌ Hard to test, can't mock |
| **Singleton Management** | ✅ NestJS manages lifecycle | ⚠️ Manual singleton pattern needed |
| **Circular Dependencies** | ✅ NestJS handles with forwardRef | ❌ Cannot handle |
| **Configuration** | ✅ Can inject ConfigService | ❌ Cannot access config |
| **Logging** | ✅ Can inject Logger | ❌ Cannot inject logger |
| **Memory Efficiency** | ✅ Single instance per app | ✅ No instances needed |
| **NestJS Integration** | ✅ First-class citizen | ❌ Outside DI container |

---

## 🔍 Real-World Scenarios

### **Scenario 1: Nested Entity Mapping (Composition)**

#### ❌ **With Static Class (Problematic)**

```typescript
// UserMapper - Static
export class UserMapper {
  static toDomain(prisma: User): UserEntity {
    // ... mapping logic
  }
  
  static toResponseDto(entity: UserEntity): UserResponseDto {
    // ... mapping logic
  }
}

// PostMapper - Static (PROBLEMS!)
export class PostMapper {
  static toResponseDtoWithAuthor(post: PostWithAuthor): PostWithAuthorDto {
    // ❌ Problem 1: Tight coupling - directly calling UserMapper
    const authorDto = UserMapper.toResponseDto(
      UserMapper.toDomain(post.author)
    );
    
    // ❌ Problem 2: Can't mock UserMapper in tests
    // ❌ Problem 3: If UserMapper needs dependencies, we're stuck
    
    return plainToClass(PostWithAuthorDto, {
      ...postDto,
      author: authorDto,
    });
  }
}
```

**Problems:**
1. **Tight Coupling**: `PostMapper` directly depends on `UserMapper` class
2. **Not Testable**: Can't mock `UserMapper` in unit tests
3. **No Flexibility**: If `UserMapper` needs dependencies later, major refactor needed
4. **Violates SOLID**: Violates Dependency Inversion Principle

#### ✅ **With @Injectable() (Correct)**

```typescript
// UserMapper - Injectable
@Injectable()
export class UserMapper {
  toDomain(prisma: User): UserEntity {
    // ... mapping logic
  }
  
  toResponseDto(entity: UserEntity): UserResponseDto {
    // ... mapping logic
  }
}

// PostMapper - Injectable
@Injectable()
export class PostMapper {
  constructor(
    private readonly userMapper: UserMapper, // ✅ Dependency injected
  ) {}

  toResponseDtoWithAuthor(post: PostWithAuthor): PostWithAuthorDto {
    // ✅ Loose coupling - using injected dependency
    const authorDto = this.userMapper.toResponseDto(
      this.userMapper.toDomain(post.author)
    );
    
    // ✅ Easy to mock in tests
    // ✅ If UserMapper needs dependencies, no changes here
    
    return plainToClass(PostWithAuthorDto, {
      ...postDto,
      author: authorDto,
    });
  }
}
```

**Benefits:**
1. **Loose Coupling**: Depends on abstraction, not concrete class
2. **Testable**: Easy to mock `UserMapper`
3. **Flexible**: Can change `UserMapper` implementation
4. **Follows SOLID**: Dependency Inversion Principle

---

### **Scenario 2: Mapper Needs Configuration**

#### ❌ **With Static Class (Impossible)**

```typescript
export class AppMapper {
  // ❌ How do we access ConfigService in static methods?
  static toResponseDto(entity: AppEntity): AppResponseDto {
    // Need to know if we should include sensitive data based on environment
    // ❌ Can't inject ConfigService into static class!
    
    const isProduction = process.env.NODE_ENV === 'production'; // ❌ Bad practice
    
    return plainToClass(AppResponseDto, entity);
  }
}
```

#### ✅ **With @Injectable() (Easy)**

```typescript
@Injectable()
export class AppMapper {
  constructor(
    private readonly configService: ConfigService, // ✅ Injected
  ) {}

  toResponseDto(entity: AppEntity): AppResponseDto {
    // ✅ Can access config through injected service
    const isProduction = this.configService.get('NODE_ENV') === 'production';
    
    // Conditionally include fields based on environment
    const dto = plainToClass(AppResponseDto, entity);
    
    if (!isProduction) {
      // Include debug info in non-production
      dto.debugInfo = { /* ... */ };
    }
    
    return dto;
  }
}
```

---

### **Scenario 3: Testing**

#### ❌ **With Static Class (Hard to Test)**

```typescript
// post.mapper.spec.ts
describe('PostMapper', () => {
  it('should map post with author', () => {
    const post = { /* ... */ };
    
    // ❌ Can't mock UserMapper - it's called directly
    // ❌ If UserMapper has complex logic or dependencies, test becomes complex
    const result = PostMapper.toResponseDtoWithAuthor(post);
    
    expect(result.author).toBeDefined();
  });
});
```

**Problems:**
- Can't isolate `PostMapper` from `UserMapper`
- Test depends on `UserMapper` implementation
- If `UserMapper` fails, `PostMapper` test fails
- Can't test edge cases easily

#### ✅ **With @Injectable() (Easy to Test)**

```typescript
// post.mapper.spec.ts
describe('PostMapper', () => {
  let postMapper: PostMapper;
  let userMapper: UserMapper;

  beforeEach(() => {
    // ✅ Create mock of UserMapper
    userMapper = {
      toDomain: jest.fn().mockReturnValue({ id: '1', email: 'test@test.com' }),
      toResponseDto: jest.fn().mockReturnValue({ id: '1', email: 'test@test.com' }),
    } as any;

    // ✅ Inject mock into PostMapper
    postMapper = new PostMapper(userMapper);
  });

  it('should map post with author', () => {
    const post = { /* ... */ };
    
    // ✅ PostMapper is isolated from UserMapper implementation
    const result = postMapper.toResponseDtoWithAuthor(post);
    
    expect(result.author).toBeDefined();
    expect(userMapper.toDomain).toHaveBeenCalledWith(post.author);
  });
});
```

**Benefits:**
- Complete isolation of `PostMapper`
- Can mock `UserMapper` behavior
- Test only `PostMapper` logic
- Easy to test edge cases

---

### **Scenario 4: Circular Dependencies**

Sometimes you might have circular dependencies between entities:

```typescript
// User has Posts
// Post has Author (User)
// Post has Comments
// Comment has Author (User) and Post
```

#### ❌ **With Static Class (Cannot Handle)**

```typescript
// ❌ Circular dependency - cannot resolve
export class UserMapper {
  static toResponseDtoWithPosts(user: UserWithPosts) {
    // Need PostMapper, but PostMapper needs UserMapper
    const posts = PostMapper.toResponseDtoList(user.posts); // ❌ Circular!
  }
}

export class PostMapper {
  static toResponseDtoWithAuthor(post: PostWithAuthor) {
    const author = UserMapper.toResponseDto(post.author); // ❌ Circular!
  }
}
```

#### ✅ **With @Injectable() (NestJS Handles It)**

```typescript
@Injectable()
export class UserMapper {
  constructor(
    @Inject(forwardRef(() => PostMapper)) // ✅ NestJS resolves circular dependency
    private readonly postMapper: PostMapper,
  ) {}

  toResponseDtoWithPosts(user: UserWithPosts) {
    const posts = this.postMapper.toResponseDtoList(user.posts); // ✅ Works!
  }
}

@Injectable()
export class PostMapper {
  constructor(
    @Inject(forwardRef(() => UserMapper))
    private readonly userMapper: UserMapper,
  ) {}

  toResponseDtoWithAuthor(post: PostWithAuthor) {
    const author = this.userMapper.toResponseDto(post.author); // ✅ Works!
  }
}
```

---

### **Scenario 5: Logging and Monitoring**

#### ❌ **With Static Class**

```typescript
export class AppMapper {
  static toResponseDto(entity: AppEntity): AppResponseDto {
    // ❌ How do we log mapping operations?
    console.log('Mapping app entity'); // ❌ Bad practice
    
    return plainToClass(AppResponseDto, entity);
  }
}
```

#### ✅ **With @Injectable()**

```typescript
@Injectable()
export class AppMapper {
  private readonly logger = new Logger(AppMapper.name);

  constructor(
    private readonly configService: ConfigService,
  ) {}

  toResponseDto(entity: AppEntity): AppResponseDto {
    // ✅ Proper logging
    this.logger.debug(`Mapping app entity: ${entity.id}`);
    
    const dto = plainToClass(AppResponseDto, entity);
    
    // ✅ Can add monitoring, metrics, etc.
    if (this.configService.get('ENABLE_MAPPER_METRICS')) {
      // Track mapping performance
    }
    
    return dto;
  }
}
```

---

## 🎓 When Would Static Make Sense?

Static methods are appropriate for **pure utility functions** with **no dependencies**:

```typescript
// ✅ OK for pure utility functions
export class StringUtils {
  static capitalize(str: string): string {
    return str.charAt(0).toUpperCase() + str.slice(1);
  }
  
  static slugify(str: string): string {
    return str.toLowerCase().replace(/\s+/g, '-');
  }
}

// ✅ OK for pure transformations
export class DateUtils {
  static formatDate(date: Date): string {
    return date.toISOString();
  }
}
```

**Characteristics of good static methods:**
- ✅ Pure functions (no side effects)
- ✅ No dependencies
- ✅ No state
- ✅ Simple transformations
- ✅ Don't need to be mocked in tests

**Mappers DON'T fit this pattern because:**
- ❌ They compose other mappers (dependencies)
- ❌ They may need configuration
- ❌ They may need logging
- ❌ They need to be testable with mocks

---

## 🏗️ Architecture Benefits

### **With @Injectable() - Proper Layered Architecture**

```typescript
// Module defines dependencies clearly
@Module({
  imports: [UserModule], // ✅ Clear dependency
  providers: [
    PostService,
    PostDao,
    PostMapper, // ✅ Managed by DI container
  ],
  exports: [PostMapper], // ✅ Can be used by other modules
})
export class PostModule {}
```

**Benefits:**
1. **Clear Dependencies**: Module imports show what's needed
2. **Lifecycle Management**: NestJS manages singleton instances
3. **Testability**: Easy to replace with mocks in tests
4. **Modularity**: Can swap implementations easily
5. **Consistency**: All services use same DI pattern

### **With Static - No Architecture**

```typescript
// ❌ No clear dependencies
@Module({
  providers: [PostService, PostDao],
  // PostMapper is not in DI container
  // Dependencies are hidden in static calls
})
export class PostModule {}
```

**Problems:**
1. Hidden dependencies
2. No lifecycle management
3. Hard to test
4. Tight coupling
5. Inconsistent with NestJS patterns

---

## 📊 Performance Considerations

**Myth**: "Static classes are faster because no instantiation"

**Reality**: With `@Injectable()`, NestJS creates **one singleton instance** per application lifecycle.

```typescript
// With @Injectable()
// NestJS creates ONE instance when app starts
const postMapper = new PostMapper(userMapper); // Created once
// Reused for all requests ✅

// With Static
// No instance needed, but:
// - Can't inject dependencies
// - Can't maintain state if needed
// - Can't be mocked
```

**Performance Impact**: Negligible! The singleton instance is created once and reused.

**Memory Impact**: Minimal! One instance vs zero instances is insignificant.

---

## ✅ Best Practices Summary

### **Use @Injectable() for Mappers When:**

1. ✅ Mapper needs to compose other mappers (Composition Pattern)
2. ✅ Mapper needs configuration (ConfigService)
3. ✅ Mapper needs logging (Logger)
4. ✅ You want to write unit tests with mocks
5. ✅ You want to follow NestJS conventions
6. ✅ You want loose coupling
7. ✅ You might need to swap implementations

### **Use Static Methods When:**

1. ✅ Pure utility functions (no dependencies)
2. ✅ Simple transformations
3. ✅ No state needed
4. ✅ No need to mock in tests
5. ✅ No configuration needed

---

## 🎯 Conclusion

**Why Mappers are @Injectable():**

1. **Dependency Injection** - Can inject other mappers for composition
2. **Testability** - Easy to mock dependencies in unit tests
3. **Flexibility** - Can add dependencies later without breaking changes
4. **NestJS Integration** - First-class citizen in DI container
5. **SOLID Principles** - Follows Dependency Inversion Principle
6. **Maintainability** - Loose coupling, easy to refactor
7. **Consistency** - Same pattern as Services, DAOs, etc.

**The composition pattern (injecting related mappers) is the killer feature that makes @Injectable() essential for mappers!**

---

## 💡 Quick Reference

```typescript
// ❌ DON'T: Static mapper (tight coupling, not testable)
export class PostMapper {
  static toResponseDtoWithAuthor(post: PostWithAuthor) {
    const authorDto = UserMapper.toResponseDto(post.author); // Tight coupling
    return { ...postDto, author: authorDto };
  }
}

// ✅ DO: Injectable mapper (loose coupling, testable)
@Injectable()
export class PostMapper {
  constructor(private readonly userMapper: UserMapper) {} // DI
  
  toResponseDtoWithAuthor(post: PostWithAuthor) {
    const authorDto = this.userMapper.toResponseDto(post.author); // Injected
    return { ...postDto, author: authorDto };
  }
}
```

**Remember**: If your mapper composes other mappers, it MUST be `@Injectable()`!
