# Month 1: Foundation & Infrastructure - Implementation Guide

**File**: `MONTH_01_FOUNDATION.md`  
**Purpose**: Step-by-step implementation guide for Month 1 (Foundation & Infrastructure)  
**Link to Roadmap**: [../ROADMAP.md](../ROADMAP.md)

---

## Assumptions

- Team: 1 Tech Lead, 2 Backend Engineers, 1 Frontend Engineer, 1 DevOps Engineer
- Cloud: AWS (can substitute GCP with minor changes)
- Node.js 20 LTS, TypeScript 5.3+, NestJS 10+, Next.js 14+, PostgreSQL 15+, Redis 7+
- **ORM**: Prisma (not TypeORM)
- **User Portal**: Next.js 14 with App Router
- GitHub for version control and CI/CD
- Docker Desktop installed locally
- Basic familiarity with TypeScript, NestJS, Next.js, Prisma, PostgreSQL

---

## Week 1-2: Project Setup

### Milestone: Development Environment Ready

#### Prerequisites

Install the following tools with exact versions:

```bash
# Node.js 20 LTS
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# Verify
node --version  # v20.x.x
npm --version   # 10.x.x

# pnpm (package manager)
npm install -g pnpm@8.15.0
pnpm --version  # 8.15.0

# Docker Desktop
# Download from https://www.docker.com/products/docker-desktop
docker --version  # 24.x.x
docker-compose --version  # 2.x.x

# NestJS CLI
pnpm install -g @nestjs/cli@10.3.0
nest --version  # 10.3.0

# PostgreSQL client (for local testing)
sudo apt-get install -y postgresql-client-15

# Redis CLI
sudo apt-get install -y redis-tools
```

---

### Task 1: Monorepo Structure

**Files to Create**:

```
chat-sdk/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
├── packages/
│   ├── server/          # NestJS backend API (Prisma)
│   ├── user-portal/     # Next.js customer portal
│   ├── sdk-core/        # TypeScript SDK
│   └── sdk-react/       # React SDK
├── docker/
│   ├── Dockerfile.server
│   ├── Dockerfile.portal
│   └── docker-compose.yml
├── .gitignore
├── .nvmrc
├── pnpm-workspace.yaml
├── package.json
└── README.md
```

**Create root `package.json`**:

```json
{
  "name": "chat-sdk-monorepo",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "pnpm --parallel --filter server --filter user-portal dev",
    "dev:server": "pnpm --filter server dev",
    "dev:portal": "pnpm --filter user-portal dev",
    "build": "pnpm -r build",
    "test": "pnpm -r test",
    "lint": "pnpm -r lint",
    "format": "prettier --write \"**/*.{ts,tsx,json,md}\""
  },
  "devDependencies": {
    "@typescript-eslint/eslint-plugin": "^6.19.0",
    "@typescript-eslint/parser": "^6.19.0",
    "eslint": "^8.56.0",
    "eslint-config-prettier": "^9.1.0",
    "prettier": "^3.2.4",
    "typescript": "^5.3.3"
  },
  "engines": {
    "node": ">=20.0.0",
    "pnpm": ">=8.0.0"
  }
}
```

**Create `pnpm-workspace.yaml`**:

```yaml
packages:
  - 'packages/*'
```

**Create `.nvmrc`**:

```
20.11.0
```

**Commands**:

```bash
mkdir -p chat-sdk/{packages/{server,user-portal,sdk-core,sdk-react},docker,.github/workflows}
cd chat-sdk
pnpm init
# Copy package.json content above
pnpm install
```

---

### Task 2: TypeScript & Linting Configuration

**Create `tsconfig.json`** (root):

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "declaration": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "moduleResolution": "node"
  }
}
```

**Create `.eslintrc.js`**:

```javascript
module.exports = {
  parser: '@typescript-eslint/parser',
  parserOptions: {
    project: 'tsconfig.json',
    sourceType: 'module',
  },
  plugins: ['@typescript-eslint/eslint-plugin'],
  extends: [
    'plugin:@typescript-eslint/recommended',
    'prettier',
  ],
  root: true,
  env: {
    node: true,
    jest: true,
  },
  rules: {
    '@typescript-eslint/interface-name-prefix': 'off',
    '@typescript-eslint/explicit-function-return-type': 'off',
    '@typescript-eslint/explicit-module-boundary-types': 'off',
    '@typescript-eslint/no-explicit-any': 'warn',
  },
};
```

**Create `.prettierrc`**:

```json
{
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "semi": true
}
```

---

### Task 3: Docker Development Environment

**Create `docker/docker-compose.yml`**:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: chat-db
    environment:
      POSTGRES_USER: chatuser
      POSTGRES_PASSWORD: chatpass
      POSTGRES_DB: chatdb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U chatuser"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: chat-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  adminer:
    image: adminer:latest
    container_name: chat-adminer
    ports:
      - "8080:8080"
    depends_on:
      - postgres

volumes:
  postgres_data:
  redis_data:
```

**Commands**:

```bash
cd docker
docker-compose up -d
docker-compose ps  # Verify all services are running
```

**Smoke Test**:

```bash
# Test PostgreSQL
psql -h localhost -U chatuser -d chatdb -c "SELECT version();"

# Test Redis
redis-cli ping  # Should return PONG
```

---

### Task 4: NestJS Backend Initialization

**Create `packages/server/package.json`**:

```json
{
  "name": "@chat-sdk/server",
  "version": "0.1.0",
  "scripts": {
    "dev": "nest start --watch",
    "build": "nest build",
    "start": "node dist/main",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "lint": "eslint \"{src,test}/**/*.ts\"",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev",
    "prisma:deploy": "prisma migrate deploy",
    "prisma:studio": "prisma studio",
    "prisma:seed": "ts-node prisma/seed.ts"
  },
  "dependencies": {
    "@nestjs/common": "^10.3.0",
    "@nestjs/core": "^10.3.0",
    "@nestjs/platform-express": "^10.3.0",
    "@nestjs/config": "^3.1.1",
    "@nestjs/jwt": "^10.2.0",
    "@nestjs/passport": "^10.0.3",
    "@nestjs/swagger": "^7.2.0",
    "@nestjs/websockets": "^10.3.0",
    "@nestjs/platform-socket.io": "^10.3.0",
    "@prisma/client": "^5.8.0",
    "pg": "^8.11.3",
    "redis": "^4.6.12",
    "bcrypt": "^5.1.1",
    "passport": "^0.7.0",
    "passport-jwt": "^4.0.1",
    "class-validator": "^0.14.1",
    "class-transformer": "^0.5.1",
    "socket.io": "^4.6.1",
    "uuid": "^9.0.1"
  },
  "devDependencies": {
    "@nestjs/cli": "^10.3.0",
    "@nestjs/schematics": "^10.1.0",
    "@nestjs/testing": "^10.3.0",
    "@types/bcrypt": "^5.0.2",
    "@types/express": "^4.17.21",
    "@types/jest": "^29.5.11",
    "@types/node": "^20.11.5",
    "@types/passport-jwt": "^4.0.0",
    "@types/uuid": "^9.0.7",
    "jest": "^29.7.0",
    "prisma": "^5.8.0",
    "ts-jest": "^29.1.1",
    "ts-node": "^10.9.2"
  }
}
```

**Initialize NestJS**:

```bash
cd packages/server
pnpm install
nest new . --skip-git --package-manager pnpm
```

**Initialize Prisma**:

```bash
cd packages/server
npx prisma init
```

This creates:
- `prisma/schema.prisma` - Database schema
- `.env` - Environment variables

**Create `packages/server/prisma/schema.prisma`**:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// Models will be added in Week 3-4
```

**Create Prisma Service** `packages/server/src/prisma/prisma.service.ts`:

```typescript
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

**Create Prisma Module** `packages/server/src/prisma/prisma.module.ts`:

```typescript
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

**Create `.env` file**:

```bash
NODE_ENV=development
PORT=3000

# Prisma Database URL
DATABASE_URL="postgresql://chatuser:chatpass@localhost:5432/chatdb?schema=public"

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# JWT
JWT_SECRET=your-super-secret-jwt-key-change-in-production
JWT_EXPIRES_IN=7d
```

---

### Task 5: CI/CD Pipeline (GitHub Actions)

**Create `.github/workflows/ci.yml`**:

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: chatuser
          POSTGRES_PASSWORD: chatpass
          POSTGRES_DB: chatdb_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 3s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Install pnpm
        uses: pnpm/action-setup@v2
        with:
          version: 8
      
      - name: Get pnpm store directory
        id: pnpm-cache
        run: echo "STORE_PATH=$(pnpm store path)" >> $GITHUB_OUTPUT
      
      - name: Setup pnpm cache
        uses: actions/cache@v3
        with:
          path: ${{ steps.pnpm-cache.outputs.STORE_PATH }}
          key: ${{ runner.os }}-pnpm-store-${{ hashFiles('**/pnpm-lock.yaml') }}
          restore-keys: |
            ${{ runner.os }}-pnpm-store-
      
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
      
      - name: Lint
        run: pnpm lint
      
      - name: Type check
        run: pnpm --filter server build
      
      - name: Run tests
        run: pnpm test
        env:
          DB_HOST: localhost
          DB_PORT: 5432
          DB_USER: chatuser
          DB_PASSWORD: chatpass
          DB_NAME: chatdb_test
          REDIS_HOST: localhost
          REDIS_PORT: 6379
          JWT_SECRET: test-secret
```

---

## Week 3-4: Core Backend - Authentication

### Milestone: Authentication System Working

### Task 1: Prisma Schema Design

**Update `packages/server/prisma/schema.prisma`**:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum UserType {
  REGULAR
  ANONYMOUS
  GUEST
}

model User {
  id            String    @id @default(uuid())
  email         String?   @unique
  username      String?   @unique
  passwordHash  String?   @map("password_hash")
  displayName   String?   @map("display_name")
  avatarUrl     String?   @map("avatar_url")
  userType      UserType  @default(REGULAR) @map("user_type")
  isDeactivated Boolean   @default(false) @map("is_deactivated")
  deactivatedAt DateTime? @map("deactivated_at")
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")

  refreshTokens RefreshToken[]
  devices       UserDevice[]

  @@index([email])
  @@index([username])
  @@index([userType])
  @@map("users")
}

model RefreshToken {
  id        String   @id @default(uuid())
  userId    String   @map("user_id")
  token     String   @unique
  expiresAt DateTime @map("expires_at")
  createdAt DateTime @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([token])
  @@map("refresh_tokens")
}

model UserDevice {
  id           String    @id @default(uuid())
  userId       String    @map("user_id")
  deviceId     String    @map("device_id")
  pushProvider String?   @map("push_provider")
  pushToken    String?   @map("push_token")
  isActive     Boolean   @default(true) @map("is_active")
  lastActiveAt DateTime  @default(now()) @map("last_active_at")
  createdAt    DateTime  @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([userId, deviceId])
  @@index([userId])
  @@map("user_devices")
}
```

**Generate Prisma Client and Create Migration**:

```bash
cd packages/server

# Generate Prisma Client
npx prisma generate

# Create and apply migration
npx prisma migrate dev --name init

# Open Prisma Studio to view data (optional)
npx prisma studio
```

---

### Task 2: User Types and DTOs

**Create `packages/server/src/users/types/user.types.ts`**:

```typescript
import { User, UserType } from '@prisma/client';
import { Exclude } from 'class-transformer';

// Export Prisma types
export { User, UserType };

// User response DTO (excludes sensitive fields)
export class UserResponseDto {
  id: string;
  email: string | null;
  username: string | null;
  displayName: string | null;
  avatarUrl: string | null;
  userType: UserType;
  isDeactivated: boolean;
  createdAt: Date;
  updatedAt: Date;

  @Exclude()
  passwordHash?: string;

  constructor(partial: Partial<User>) {
    Object.assign(this, partial);
  }
}

// Create user DTO
export class CreateUserDto {
  email?: string;
  username?: string;
  password?: string;
  displayName?: string;
  userType?: UserType;
}

// Update user DTO
export class UpdateUserDto {
  email?: string;
  username?: string;
  displayName?: string;
  avatarUrl?: string;
}
```

---

### Task 3: Authentication Module

**Create `packages/server/src/auth/auth.module.ts`**:

```typescript
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { JwtStrategy } from './strategies/jwt.strategy';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [
    UsersModule,
    PassportModule.register({ defaultStrategy: 'jwt' }),
    JwtModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        secret: config.get('JWT_SECRET'),
        signOptions: {
          expiresIn: config.get('JWT_EXPIRES_IN', '7d'),
        },
      }),
    }),
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy],
  exports: [AuthService, JwtStrategy, PassportModule],
})
export class AuthModule {}
```

**Create `packages/server/src/auth/auth.service.ts`**:

```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import * as bcrypt from 'bcrypt';
import { PrismaService } from '../prisma/prisma.service';
import { User, UserType } from '@prisma/client';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';

@Injectable()
export class AuthService {
  constructor(
    private prisma: PrismaService,
    private jwtService: JwtService,
  ) {}

  async register(dto: RegisterDto) {
    const passwordHash = await bcrypt.hash(dto.password, 10);
    
    const user = await this.prisma.user.create({
      data: {
        email: dto.email,
        username: dto.username,
        passwordHash,
        displayName: dto.displayName,
        userType: UserType.REGULAR,
      },
    });

    return this.generateTokens(user);
  }

  async login(dto: LoginDto) {
    // Find user by email or username
    const user = await this.prisma.user.findFirst({
      where: {
        OR: [
          { email: dto.identifier },
          { username: dto.identifier },
        ],
      },
    });

    if (!user || !user.passwordHash) {
      throw new UnauthorizedException('Invalid credentials');
    }

    const isPasswordValid = await bcrypt.compare(dto.password, user.passwordHash);
    if (!isPasswordValid) {
      throw new UnauthorizedException('Invalid credentials');
    }

    if (user.isDeactivated) {
      throw new UnauthorizedException('Account is deactivated');
    }

    return this.generateTokens(user);
  }

  async refreshToken(refreshToken: string) {
    const token = await this.prisma.refreshToken.findUnique({
      where: { token: refreshToken },
      include: { user: true },
    });

    if (!token || token.expiresAt < new Date()) {
      throw new UnauthorizedException('Invalid refresh token');
    }

    return this.generateTokens(token.user);
  }

  async revokeToken(userId: string, tokenId: string) {
    await this.prisma.refreshToken.delete({
      where: {
        id: tokenId,
        userId,
      },
    });
  }

  private async generateTokens(user: User) {
    const payload = { sub: user.id, email: user.email, type: user.userType };
    
    const accessToken = this.jwtService.sign(payload);
    
    const refreshToken = await this.prisma.refreshToken.create({
      data: {
        userId: user.id,
        token: this.jwtService.sign(payload, { expiresIn: '30d' }),
        expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
      },
    });

    return {
      accessToken,
      refreshToken: refreshToken.token,
      user: {
        id: user.id,
        email: user.email,
        username: user.username,
        displayName: user.displayName,
      },
    };
  }
}
```

---

### Smoke Tests

**Create `packages/server/src/auth/auth.service.spec.ts`**:

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { AuthService } from './auth.service';
import { PrismaService } from '../prisma/prisma.service';
import { JwtService } from '@nestjs/jwt';

describe('AuthService', () => {
  let service: AuthService;
  let prisma: PrismaService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        AuthService,
        {
          provide: PrismaService,
          useValue: {
            user: {
              create: jest.fn(),
              findFirst: jest.fn(),
              findUnique: jest.fn(),
            },
            refreshToken: {
              create: jest.fn(),
              findUnique: jest.fn(),
              delete: jest.fn(),
            },
          },
        },
        {
          provide: JwtService,
          useValue: {
            sign: jest.fn(() => 'test-token'),
          },
        },
      ],
    }).compile();

    service = module.get<AuthService>(AuthService);
    prisma = module.get<PrismaService>(PrismaService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });
});
```

**Run Tests**:

```bash
pnpm test
```

---

## Task 4: User Portal Setup (Next.js)

### Initialize Next.js User Portal

**Create `packages/user-portal/package.json`**:

```json
{
  "name": "@chat-sdk/user-portal",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev -p 3001",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "next": "^14.1.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "next-auth": "^4.24.5",
    "axios": "^1.6.5",
    "@tanstack/react-query": "^5.17.19",
    "tailwindcss": "^3.4.1",
    "autoprefixer": "^10.4.17",
    "postcss": "^8.4.33"
  },
  "devDependencies": {
    "@types/node": "^20.11.5",
    "@types/react": "^18.2.48",
    "@types/react-dom": "^18.2.18",
    "typescript": "^5.3.3",
    "eslint": "^8.56.0",
    "eslint-config-next": "^14.1.0"
  }
}
```

**Initialize Next.js**:

```bash
cd packages/user-portal
pnpm install
npx create-next-app@latest . --typescript --tailwind --app --no-src-dir
```

**Create `packages/user-portal/next.config.js`**:

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  env: {
    NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3000',
  },
};

module.exports = nextConfig;
```

**Create `packages/user-portal/.env.local`**:

```bash
NEXT_PUBLIC_API_URL=http://localhost:3000
NEXTAUTH_URL=http://localhost:3001
NEXTAUTH_SECRET=your-nextauth-secret-change-in-production
```

### Create Basic Layout

**Create `packages/user-portal/app/layout.tsx`**:

```typescript
import type { Metadata } from 'next';
import { Inter } from 'next/font/google';
import './globals.css';

const inter = Inter({ subsets: ['latin'] });

export const metadata: Metadata = {
  title: 'Chat SDK Portal',
  description: 'Manage your Chat SDK integration',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={inter.className}>{children}</body>
    </html>
  );
}
```

**Create `packages/user-portal/app/page.tsx`**:

```typescript
export default function Home() {
  return (
    <main className="flex min-h-screen flex-col items-center justify-center p-24">
      <h1 className="text-4xl font-bold mb-4">Chat SDK Portal</h1>
      <p className="text-xl text-gray-600">
        Welcome to your Chat SDK management portal
      </p>
    </main>
  );
}
```

### Create API Client

**Create `packages/user-portal/lib/api-client.ts`**:

```typescript
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor to add auth token
apiClient.interceptors.request.use((config) => {
  if (typeof window !== 'undefined') {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
  }
  return config;
});

// Response interceptor for error handling
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Redirect to login
      if (typeof window !== 'undefined') {
        window.location.href = '/login';
      }
    }
    return Promise.reject(error);
  }
);

export const api = {
  auth: {
    login: (data: { identifier: string; password: string }) =>
      apiClient.post('/auth/login', data),
    register: (data: { email: string; username: string; password: string }) =>
      apiClient.post('/auth/register', data),
    me: () => apiClient.get('/auth/me'),
  },
  users: {
    list: () => apiClient.get('/users'),
    get: (id: string) => apiClient.get(`/users/${id}`),
  },
  // More endpoints will be added in later months
};

export default apiClient;
```

### Create Authentication Pages

**Create `packages/user-portal/app/(auth)/login/page.tsx`**:

```typescript
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { api } from '@/lib/api-client';

export default function LoginPage() {
  const router = useRouter();
  const [identifier, setIdentifier] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');
    setLoading(true);

    try {
      const response = await api.auth.login({ identifier, password });
      localStorage.setItem('token', response.data.accessToken);
      router.push('/dashboard');
    } catch (err: any) {
      setError(err.response?.data?.message || 'Login failed');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="flex min-h-screen items-center justify-center bg-gray-50">
      <div className="w-full max-w-md space-y-8 p-8 bg-white rounded-lg shadow">
        <h2 className="text-3xl font-bold text-center">Sign in to Chat SDK</h2>
        
        {error && (
          <div className="bg-red-50 text-red-600 p-3 rounded">{error}</div>
        )}

        <form onSubmit={handleSubmit} className="space-y-6">
          <div>
            <label className="block text-sm font-medium text-gray-700">
              Email or Username
            </label>
            <input
              type="text"
              value={identifier}
              onChange={(e) => setIdentifier(e.target.value)}
              className="mt-1 block w-full rounded-md border border-gray-300 px-3 py-2"
              required
            />
          </div>

          <div>
            <label className="block text-sm font-medium text-gray-700">
              Password
            </label>
            <input
              type="password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              className="mt-1 block w-full rounded-md border border-gray-300 px-3 py-2"
              required
            />
          </div>

          <button
            type="submit"
            disabled={loading}
            className="w-full bg-blue-600 text-white py-2 px-4 rounded-md hover:bg-blue-700 disabled:opacity-50"
          >
            {loading ? 'Signing in...' : 'Sign in'}
          </button>
        </form>
      </div>
    </div>
  );
}
```

**Create `packages/user-portal/app/(dashboard)/dashboard/page.tsx`**:

```typescript
export default function DashboardPage() {
  return (
    <div className="p-8">
      <h1 className="text-3xl font-bold mb-4">Dashboard</h1>
      <p className="text-gray-600">
        Welcome to your Chat SDK dashboard. More features coming in Month 6!
      </p>
    </div>
  );
}
```

**Test User Portal**:

```bash
cd packages/user-portal
pnpm dev
# Visit http://localhost:3001
```

---

## Update AppModule with Prisma

**Update `packages/server/src/app.module.ts`**:

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { PrismaModule } from './prisma/prisma.module';
import { AuthModule } from './auth/auth.module';
import { UsersModule } from './users/users.module';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
    }),
    PrismaModule,
    AuthModule,
    UsersModule,
  ],
})
export class AppModule {}
```

---

## Deliverables Checklist

- [x] Monorepo structure with pnpm workspaces
- [x] TypeScript, ESLint, Prettier configured
- [x] Docker Compose with PostgreSQL and Redis
- [x] NestJS backend initialized with Prisma ORM
- [x] Prisma schema defined and migrations created
- [x] PrismaService and PrismaModule created
- [x] User authentication (JWT) implemented with Prisma
- [x] Token refresh mechanism
- [x] Next.js User Portal initialized
- [x] User Portal authentication pages (login)
- [x] API client for User Portal
- [x] CI/CD pipeline (GitHub Actions)
- [x] Basic tests passing

---

## Next Steps

Proceed to [MONTH_02_MESSAGING.md](./MONTH_02_MESSAGING.md) for core messaging features implementation.
