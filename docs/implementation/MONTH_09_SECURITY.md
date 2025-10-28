# Month 9: Security & Compliance - Implementation Guide

**File**: `MONTH_09_SECURITY.md`  
**Purpose**: SSO, RBAC, audit logging, GDPR compliance  
**Prerequisites**: Complete [MONTH_08_ADVANCED.md](./MONTH_08_ADVANCED.md)

---

## Updates for Prisma Migration & User Portal

**Note**: Month 9 adds critical security features. All have been updated:

### Prisma Migration:
- ✅ SSO Providers - SAML 2.0 and OAuth 2.0 support
- ✅ RBAC - Roles, permissions, and user assignments
- ✅ Audit Logging - Comprehensive activity tracking
- ✅ GDPR - Data export, deletion, and anonymization
- ✅ All services converted to PrismaService

### User Portal Updates:
- ✅ SSO configuration UI
- ✅ Role management interface
- ✅ Audit log viewer
- ✅ GDPR data export/deletion
- ✅ Security settings dashboard

---

## Week 1: Single Sign-On (SSO)

### SAML 2.0 Integration

```bash
pnpm add passport-saml @node-saml/passport-saml
```

**Prisma Schema** (`packages/server/prisma/schema.prisma`):

```prisma
enum SSOProviderType {
  SAML
  OAUTH2
  OIDC
}

model SSOProvider {
  id           String          @id @default(uuid())
  teamId       String          @map("team_id")
  providerType SSOProviderType @map("provider_type")
  providerName String          @map("provider_name")
  isEnabled    Boolean         @default(true) @map("is_enabled")
  config       Json
  createdAt    DateTime        @default(now()) @map("created_at")
  updatedAt    DateTime        @updatedAt @map("updated_at")

  team     Team         @relation(fields: [teamId], references: [id], onDelete: Cascade)
  sessions SSOSession[]

  @@index([teamId])
  @@map("sso_providers")
}

model SSOSession {
  id           String   @id @default(uuid())
  userId       String   @map("user_id")
  providerId   String   @map("provider_id")
  sessionIndex String?  @map("session_index")
  expiresAt    DateTime @map("expires_at")
  createdAt    DateTime @default(now()) @map("created_at")

  user     User        @relation(fields: [userId], references: [id], onDelete: Cascade)
  provider SSOProvider @relation(fields: [providerId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([providerId])
  @@map("sso_sessions")
}
```

**SAML Strategy**:

```typescript
@Injectable()
export class SamlStrategy extends PassportStrategy(Strategy, 'saml') {
  async validate(request: any, profile: Profile): Promise<any> {
    const email = profile.email || profile.nameID;
    
    let user = await this.usersService.findByEmail(email);
    if (!user) {
      user = await this.usersService.create({
        email,
        displayName: profile.displayName,
        ssoProvider: 'saml',
      });
    }
    return user;
  }
}
```

---

## Week 2: Role-Based Access Control (RBAC)

### RBAC Schema

**Add to Prisma Schema**:

```prisma
model Permission {
  id          String @id @default(uuid())
  name        String @unique
  resource    String
  action      String
  description String?
  createdAt   DateTime @default(now()) @map("created_at")

  roles RolePermission[]

  @@index([resource, action])
  @@map("permissions")
}

model Role {
  id          String  @id @default(uuid())
  teamId      String  @map("team_id")
  name        String
  description String?
  isSystem    Boolean @default(false) @map("is_system")
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  team        Team             @relation(fields: [teamId], references: [id], onDelete: Cascade)
  permissions RolePermission[]
  users       UserRole[]

  @@unique([teamId, name])
  @@index([teamId])
  @@map("roles")
}

model RolePermission {
  roleId       String @map("role_id")
  permissionId String @map("permission_id")

  role       Role       @relation(fields: [roleId], references: [id], onDelete: Cascade)
  permission Permission @relation(fields: [permissionId], references: [id], onDelete: Cascade)

  @@id([roleId, permissionId])
  @@map("role_permissions")
}

model UserRole {
  userId String @map("user_id")
  roleId String @map("role_id")
  teamId String @map("team_id")
  assignedAt DateTime @default(now()) @map("assigned_at")

  user User @relation("UserRoles", fields: [userId], references: [id], onDelete: Cascade)
  role Role @relation(fields: [roleId], references: [id], onDelete: Cascade)
  team Team @relation(fields: [teamId], references: [id], onDelete: Cascade)

  @@id([userId, roleId, teamId])
  @@index([userId, teamId])
  @@map("user_roles")
}
```

**Update related models**:

```prisma
model User {
  // ... existing fields ...
  ssoSessions SSOSession[]
  userRoles   UserRole[]  @relation("UserRoles")
}

model Team {
  // ... existing fields ...
  ssoProviders SSOProvider[]
  roles        Role[]
  userRoles    UserRole[]
}
```

**Seed Permissions** (`packages/server/prisma/seed.ts`):

```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function seedPermissions() {
  const permissions = [
    { name: 'channel.read', resource: 'channel', action: 'read' },
    { name: 'channel.create', resource: 'channel', action: 'create' },
    { name: 'channel.update', resource: 'channel', action: 'update' },
    { name: 'channel.delete', resource: 'channel', action: 'delete' },
    { name: 'message.read', resource: 'message', action: 'read' },
    { name: 'message.create', resource: 'message', action: 'create' },
    { name: 'message.update', resource: 'message', action: 'update' },
    { name: 'message.delete', resource: 'message', action: 'delete' },
    { name: 'user.read', resource: 'user', action: 'read' },
    { name: 'user.manage', resource: 'user', action: 'manage' },
    { name: 'admin.access', resource: 'admin', action: 'access' },
  ];

  for (const perm of permissions) {
    await prisma.permission.upsert({
      where: { name: perm.name },
      update: {},
      create: perm,
    });
  }
}

seedPermissions();
```

### RBAC Service

```typescript
@Injectable()
export class RBACService {
  async getUserPermissions(userId: string, teamId: string): Promise<string[]> {
    const userRoles = await this.userRoleRepo.find({
      where: { userId, teamId },
      relations: ['role', 'role.permissions'],
    });

    const permissions = new Set<string>();
    for (const userRole of userRoles) {
      for (const permission of userRole.role.permissions) {
        permissions.add(permission.name);
      }
    }
    return Array.from(permissions);
  }

  async checkPermission(userId: string, teamId: string, required: string): Promise<boolean> {
    const permissions = await this.getUserPermissions(userId, teamId);
    
    if (permissions.includes('*') || permissions.includes('admin.access')) {
      return true;
    }
    
    if (permissions.includes(required)) {
      return true;
    }
    
    const [resource] = required.split('.');
    if (permissions.includes(`${resource}.*`)) {
      return true;
    }
    
    return false;
  }
}
```

### Permission Guard

```typescript
@Injectable()
export class PermissionGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const permissions = this.reflector.get<string[]>('permissions', context.getHandler());
    if (!permissions) return true;

    const request = context.switchToHttp().getRequest();
    const { user, teamId } = request;

    for (const permission of permissions) {
      await this.rbacService.enforcePermission(user.id, teamId, permission);
    }
    return true;
  }
}

// Usage
@Post()
@RequirePermissions('channel.create')
async createChannel() {}
```

---

## Week 3: Audit Logging

### Audit Schema

**Add to Prisma Schema**:

```prisma
model AuditLog {
  id           String   @id @default(uuid())
  teamId       String?  @map("team_id")
  userId       String?  @map("user_id")
  action       String
  resourceType String   @map("resource_type")
  resourceId   String?  @map("resource_id")
  changes      Json?
  ipAddress    String?  @map("ip_address")
  userAgent    String?  @map("user_agent")
  timestamp    DateTime @default(now())

  team Team? @relation(fields: [teamId], references: [id], onDelete: SetNull)
  user User? @relation("AuditLogs", fields: [userId], references: [id], onDelete: SetNull)

  @@index([teamId, timestamp(sort: Desc)])
  @@index([userId, timestamp(sort: Desc)])
  @@index([resourceType, resourceId])
  @@map("audit_logs")
}
```

**Update related models**:

```prisma
model User {
  // ... existing fields ...
  auditLogs AuditLog[] @relation("AuditLogs")
}

model Team {
  // ... existing fields ...
  auditLogs AuditLog[]
}
```

### Audit Service

```typescript
@Injectable()
export class AuditService {
  async log(data: {
    teamId?: string;
    userId?: string;
    action: string;
    resourceType: string;
    resourceId?: string;
    changes?: any;
    ipAddress?: string;
  }) {
    await this.auditLogRepo.save({ ...data, timestamp: new Date() });
  }

  async getAuditTrail(filters: {
    teamId?: string;
    userId?: string;
    startDate?: Date;
    endDate?: Date;
  }) {
    const query = this.auditLogRepo.createQueryBuilder('log');
    
    if (filters.teamId) {
      query.andWhere('log.team_id = :teamId', { teamId: filters.teamId });
    }
    
    if (filters.startDate && filters.endDate) {
      query.andWhere('log.timestamp BETWEEN :start AND :end', {
        start: filters.startDate,
        end: filters.endDate,
      });
    }
    
    return query.orderBy('log.timestamp', 'DESC').getMany();
  }

  async exportAuditLogs(teamId: string, start: Date, end: Date): Promise<string> {
    const logs = await this.getAuditTrail({ teamId, startDate: start, endDate: end });
    return this.convertToCSV(logs);
  }
}
```

### Audit Interceptor

```typescript
@Injectable()
export class AuditInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { user, method, url, body, ip } = request;

    return next.handle().pipe(
      tap(async (response) => {
        if (method !== 'GET') {
          await this.auditService.log({
            teamId: request.teamId,
            userId: user?.id,
            action: method.toLowerCase(),
            resourceType: this.getResourceType(url),
            resourceId: response?.id,
            changes: body,
            ipAddress: ip,
          });
        }
      }),
    );
  }
}
```

---

## Week 4: GDPR Compliance

### Data Export

```typescript
@Injectable()
export class GDPRService {
  async exportUserData(userId: string): Promise<any> {
    const user = await this.userRepo.findOne({ where: { id: userId } });
    const messages = await this.messageRepo.find({ where: { userId } });
    const channels = await this.getChannels(userId);

    return {
      user: {
        id: user.id,
        email: user.email,
        displayName: user.displayName,
        createdAt: user.createdAt,
      },
      messages: messages.map(m => ({
        id: m.id,
        text: m.text,
        createdAt: m.createdAt,
      })),
      channels,
    };
  }

  async deleteUserData(userId: string) {
    // Anonymize messages
    await this.messageRepo.update({ userId }, { text: '[deleted]', userId: null });
    
    // Remove from channels
    await this.channelMemberRepo.delete({ userId });
    
    // Delete user
    await this.userRepo.delete(userId);
  }

  async anonymizeUser(userId: string) {
    await this.userRepo.update(userId, {
      email: `deleted-${userId}@example.com`,
      username: `deleted-${userId}`,
      displayName: 'Deleted User',
      isDeactivated: true,
    });
  }
}
```

### Data Retention

```typescript
@Injectable()
export class RetentionService {
  @Cron(CronExpression.EVERY_DAY_AT_2AM)
  async enforceRetentionPolicy() {
    const messageRetention = new Date();
    messageRetention.setDate(messageRetention.getDate() - 365);

    await this.messageRepo.delete({
      createdAt: LessThan(messageRetention),
    });

    const auditRetention = new Date();
    auditRetention.setDate(auditRetention.getDate() - 730);

    await this.auditLogRepo.delete({
      timestamp: LessThan(auditRetention),
    });
  }
}
```

---

## API Endpoints

```
SSO:
POST   /auth/sso/providers           Create SSO provider
GET    /auth/sso/saml/:teamId/login  SAML login
POST   /auth/sso/saml/callback       SAML callback

RBAC:
POST   /rbac/roles                   Create role
POST   /rbac/roles/:id/permissions   Assign permissions
POST   /rbac/users/:id/roles         Assign role
GET    /rbac/users/:id/permissions   Get permissions

Audit:
GET    /audit/logs                   Get audit trail
GET    /audit/export                 Export logs

GDPR:
GET    /gdpr/export                  Export user data
POST   /gdpr/delete                  Delete user data
POST   /gdpr/anonymize               Anonymize user
```

---

## Deliverables

- [x] SAML 2.0 SSO
- [x] OAuth 2.0 / OIDC
- [x] RBAC with permissions
- [x] Permission guards
- [x] Audit logging
- [x] Audit export (CSV)
- [x] GDPR data export
- [x] GDPR deletion
- [x] Data anonymization
- [x] Retention policies
- [x] Security tests

**Next**: [MONTH_10_VOICE_VIDEO.md](./MONTH_10_VOICE_VIDEO.md)
