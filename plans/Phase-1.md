# Phase 1: Enhanced Discovery & Architecture

## Overview
**Duration**: 3-4 weeks  
**Team Size**: 3-4 developers + 1 architect  
**Objective**: Implement core infrastructure, set up real-time communication, and establish production-ready development environment

## Objectives & Goals

### Primary Objectives
1. **Core Infrastructure**: Implement robust NestJS backend with PostgreSQL and Redis
2. **Real-time Communication**: Set up WebSocket infrastructure for live updates
3. **Background Processing**: Implement job queues for async operations
4. **File Management**: Integrate secure file storage with AWS S3
5. **CI/CD Pipeline**: Establish automated testing and deployment pipeline

### Success Criteria
- Backend API serving all core endpoints
- Real-time updates working across all clients
- File upload/download system operational
- Background jobs processing notifications
- CI/CD pipeline deploying to staging automatically

## Technical Specifications

### Backend Infrastructure Architecture

#### NestJS Application Structure
```
src/
├── app.module.ts
├── main.ts
├── common/
│   ├── decorators/
│   ├── filters/
│   ├── guards/
│   ├── interceptors/
│   ├── pipes/
│   └── utils/
├── config/
│   ├── database.config.ts
│   ├── redis.config.ts
│   ├── jwt.config.ts
│   └── s3.config.ts
├── modules/
│   ├── auth/
│   ├── users/
│   ├── care-packages/
│   ├── tasks/
│   ├── competencies/
│   └── files/
├── shared/
│   ├── entities/
│   ├── dto/
│   ├── interfaces/
│   └── types/
└── infrastructure/
    ├── database/
    ├── cache/
    ├── queue/
    └── storage/
```

#### Database Configuration with Prisma
```typescript
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id                    String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  email                 String    @unique @db.VarChar(255)
  passwordHash          String    @map("password_hash") @db.VarChar(255)
  role                  UserRole  @default(CARER)
  isActive              Boolean   @default(true) @map("is_active")
  twoFactorEnabled      Boolean   @default(false) @map("two_factor_enabled")
  twoFactorSecret       String?   @map("two_factor_secret") @db.VarChar(255)
  emailVerified         Boolean   @default(false) @map("email_verified")
  lastLogin             DateTime? @map("last_login")
  failedLoginAttempts   Int       @default(0) @map("failed_login_attempts")
  lockedUntil           DateTime? @map("locked_until")
  createdAt             DateTime  @default(now()) @map("created_at")
  updatedAt             DateTime  @updatedAt @map("updated_at")

  profile               UserProfile?
  refreshTokens         RefreshToken[]
  createdCarePackages   CarePackage[] @relation("CreatedBy")
  createdTasks          Task[] @relation("CreatedBy")
  auditLogs             AuditLog[]
  carerAssignments      CarerAssignment[]
  taskProgress          TaskProgress[]
  competencies          CarerCompetency[]
  assessedCompetencies  CarerCompetency[] @relation("AssessedBy")

  @@map("users")
}

enum UserRole {
  ADMIN
  CARER
  ASSESSOR
  VIEWER

  @@map("user_role")
}

model UserProfile {
  id        String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  userId    String   @unique @map("user_id") @db.Uuid
  firstName String   @map("first_name") @db.VarChar(100)
  lastName  String   @map("last_name") @db.VarChar(100)
  phone     String?  @db.VarChar(20)
  postcode  String?  @db.VarChar(10)
  avatarUrl String?  @map("avatar_url") @db.VarChar(500)
  bio       String?  @db.Text
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("user_profiles")
}

model CarePackage {
  id           String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  name         String   @db.VarChar(255)
  postcode     String   @db.VarChar(10)
  description  String?  @db.Text
  clientName   String?  @map("client_name") @db.VarChar(255) // encrypted
  addressLine1 String?  @map("address_line1") @db.VarChar(255) // encrypted
  addressLine2 String?  @map("address_line2") @db.VarChar(255) // encrypted
  isActive     Boolean  @default(true) @map("is_active")
  createdBy    String   @map("created_by") @db.Uuid
  createdAt    DateTime @default(now()) @map("created_at")
  updatedAt    DateTime @updatedAt @map("updated_at")
  deletedAt    DateTime? @map("deleted_at")

  creator          User @relation("CreatedBy", fields: [createdBy], references: [id])
  carerAssignments CarerAssignment[]
  taskProgress     TaskProgress[]

  @@map("care_packages")
}

model Task {
  id                       String          @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  name                     String          @db.VarChar(255)
  description              String?         @db.Text
  category                 String?         @db.VarChar(100)
  targetCount              Int             @default(1) @map("target_count")
  isCompetencyRequired     Boolean         @default(false) @map("is_competency_required")
  minimumCompetencyLevel   CompetencyLevel @default(COMPETENT) @map("minimum_competency_level")
  isActive                 Boolean         @default(true) @map("is_active")
  createdBy                String          @map("created_by") @db.Uuid
  createdAt                DateTime        @default(now()) @map("created_at")
  updatedAt                DateTime        @updatedAt @map("updated_at")
  deletedAt                DateTime?       @map("deleted_at")

  creator      User @relation("CreatedBy", fields: [createdBy], references: [id])
  progress     TaskProgress[]
  competencies CarerCompetency[]

  @@map("tasks")
}

enum CompetencyLevel {
  NOT_ASSESSED
  COMPETENT
  PROFICIENT
  EXPERT

  @@map("competency_level")
}

model CarerAssignment {
  id            String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  carerId       String   @map("carer_id") @db.Uuid
  carePackageId String   @map("care_package_id") @db.Uuid
  assignedAt    DateTime @default(now()) @map("assigned_at")
  assignedBy    String   @map("assigned_by") @db.Uuid
  isActive      Boolean  @default(true) @map("is_active")

  carer       User        @relation(fields: [carerId], references: [id])
  carePackage CarePackage @relation(fields: [carePackageId], references: [id])

  @@unique([carerId, carePackageId])
  @@map("carer_assignments")
}

model TaskProgress {
  id            String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  carerId       String    @map("carer_id") @db.Uuid
  taskId        String    @map("task_id") @db.Uuid
  carePackageId String    @map("care_package_id") @db.Uuid
  currentCount  Int       @default(0) @map("current_count")
  targetCount   Int       @map("target_count")
  isCompleted   Boolean   @default(false) @map("is_completed")
  completedAt   DateTime? @map("completed_at")
  lastUpdated   DateTime  @default(now()) @map("last_updated")

  carer       User        @relation(fields: [carerId], references: [id])
  task        Task        @relation(fields: [taskId], references: [id])
  carePackage CarePackage @relation(fields: [carePackageId], references: [id])

  @@unique([carerId, taskId, carePackageId])
  @@map("task_progress")
}

model CarerCompetency {
  id          String          @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  carerId     String          @map("carer_id") @db.Uuid
  taskId      String          @map("task_id") @db.Uuid
  level       CompetencyLevel @default(NOT_ASSESSED)
  assessedBy  String?         @map("assessed_by") @db.Uuid
  assessedAt  DateTime?       @map("assessed_at")
  notes       String?         @db.Text
  isConfirmed Boolean         @default(false) @map("is_confirmed")
  confirmedAt DateTime?       @map("confirmed_at")
  createdAt   DateTime        @default(now()) @map("created_at")
  updatedAt   DateTime        @updatedAt @map("updated_at")

  carer    User  @relation(fields: [carerId], references: [id])
  task     Task  @relation(fields: [taskId], references: [id])
  assessor User? @relation("AssessedBy", fields: [assessedBy], references: [id])

  @@unique([carerId, taskId])
  @@map("carer_competencies")
}

model RefreshToken {
  id        String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  userId    String    @map("user_id") @db.Uuid
  tokenHash String    @map("token_hash") @db.VarChar(255)
  expiresAt DateTime  @map("expires_at")
  isRevoked Boolean   @default(false) @map("is_revoked")
  createdAt DateTime  @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id])

  @@unique([tokenHash])
  @@map("refresh_tokens")
}

model AuditLog {
  id         String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  userId     String?   @map("user_id") @db.Uuid
  entityType String    @map("entity_type") @db.VarChar(100)
  entityId   String    @map("entity_id") @db.Uuid
  action     String    @db.VarChar(50)
  oldValues  Json?     @map("old_values")
  newValues  Json?     @map("new_values")
  ipAddress  String?   @map("ip_address") @db.Inet
  userAgent  String?   @map("user_agent") @db.Text
  createdAt  DateTime  @default(now()) @map("created_at")

  user User? @relation(fields: [userId], references: [id])

  @@map("audit_logs")
}
```

#### Real-time WebSocket Implementation with Enhanced Authentication
```typescript
// src/modules/websocket/websocket.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
  OnGatewayInit,
  OnGatewayConnection,
  OnGatewayDisconnect,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { Logger, UseGuards } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { RedisService } from '../redis/redis.service';
import { User } from '../users/entities/user.entity';

@WebSocketGateway({
  cors: {
    origin: process.env.FRONTEND_URL || 'http://localhost:3001',
    credentials: true,
  },
  namespace: '/api/ws',
  transports: ['websocket', 'polling'],
})
export class WebSocketGateway
  implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect
{
  @WebSocketServer() server: Server;
  private logger: Logger = new Logger('WebSocketGateway');
  private connectedUsers = new Map<string, string>(); // userId -> socketId

  constructor(
    private jwtService: JwtService,
    private redisService: RedisService,
  ) {}

  afterInit(server: Server) {
    this.logger.log('WebSocket Gateway initialized');
  }

  async handleConnection(client: Socket, ...args: any[]) {
    try {
      // Enhanced authentication
      const token = client.handshake.auth.token || client.handshake.headers.authorization?.split(' ')[1];
      if (!token) {
        client.disconnect();
        return;
      }

      const payload = this.jwtService.verify(token);
      const sessionValid = await this.redisService.get(`session:${payload.sessionId}`);
      
      if (!sessionValid) {
        client.disconnect();
        return;
      }

      this.logger.log(`Client connected: ${client.id} (User: ${payload.sub})`);
      
      // Store connection
      this.connectedUsers.set(payload.sub, client.id);
      client.join(`user_${payload.sub}`);
      client.join(`role_${payload.role}`);
      
      // Update last activity
      await this.redisService.setex(`user_activity:${payload.sub}`, 3600, Date.now().toString());
      
    } catch (error) {
      this.logger.error('Authentication failed for WebSocket connection', error);
      client.disconnect();
    }
  }

  handleDisconnect(client: Socket) {
    this.logger.log(`Client disconnected: ${client.id}`);
    
    // Remove user from connected users
    for (const [userId, socketId] of this.connectedUsers.entries()) {
      if (socketId === client.id) {
        this.connectedUsers.delete(userId);
        break;
      }
    }
  }

  @SubscribeMessage('join_care_package')
  handleJoinCarePackage(
    @MessageBody() data: { carePackageId: string },
    @ConnectedSocket() client: Socket,
    @CurrentUser() user: User,
  ) {
    client.join(`care_package_${data.carePackageId}`);
    this.logger.log(`User ${user.id} joined care package ${data.carePackageId}`);
  }

  // Real-time event emitters
  emitTaskProgressUpdate(carerId: string, taskId: string, progress: any) {
    this.server.to(`user_${carerId}`).emit('task_progress_updated', {
      taskId,
      progress,
    });
  }

  emitCompetencyUpdate(carerId: string, competency: any) {
    this.server.to(`user_${carerId}`).emit('competency_updated', competency);
  }

  emitShiftNotification(carerId: string, shift: any) {
    this.server.to(`user_${carerId}`).emit('shift_notification', shift);
  }

  emitCarePackageUpdate(carePackageId: string, update: any) {
    this.server.to(`care_package_${carePackageId}`).emit('care_package_updated', update);
  }
}
```

#### Background Job Processing with Bull
```typescript
// src/modules/queue/queue.module.ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { NotificationProcessor } from './processors/notification.processor';
import { EmailProcessor } from './processors/email.processor';
import { ReportProcessor } from './processors/report.processor';

@Module({
  imports: [
    BullModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: async (configService: ConfigService) => ({
        redis: {
          host: configService.get('REDIS_HOST'),
          port: configService.get('REDIS_PORT'),
          password: configService.get('REDIS_PASSWORD'),
        },
        defaultJobOptions: {
          removeOnComplete: 100,
          removeOnFail: 50,
        },
      }),
      inject: [ConfigService],
    }),
    BullModule.registerQueue(
      { name: 'notifications' },
      { name: 'emails' },
      { name: 'reports' },
    ),
  ],
  providers: [NotificationProcessor, EmailProcessor, ReportProcessor],
  exports: [BullModule],
})
export class QueueModule {}

// src/modules/queue/processors/notification.processor.ts
import { Process, Processor } from '@nestjs/bull';
import { Job } from 'bull';
import { Logger } from '@nestjs/common';
import { WebSocketGateway } from '../../websocket/websocket.gateway';

@Processor('notifications')
export class NotificationProcessor {
  private readonly logger = new Logger(NotificationProcessor.name);

  constructor(private readonly websocketGateway: WebSocketGateway) {}

  @Process('send_push_notification')
  async handlePushNotification(job: Job) {
    const { userId, title, message, data } = job.data;
    
    try {
      // Send push notification via WebSocket
      this.websocketGateway.server.to(`user_${userId}`).emit('push_notification', {
        title,
        message,
        data,
      });

      // Here you would also send to mobile push notification service
      // await this.pushNotificationService.send(userId, title, message, data);
      
      this.logger.log(`Push notification sent to user ${userId}`);
    } catch (error) {
      this.logger.error(`Failed to send push notification to user ${userId}:`, error);
      throw error;
    }
  }

  @Process('send_email_notification')
  async handleEmailNotification(job: Job) {
    const { userId, template, data } = job.data;
    
    try {
      // Process email notification
      this.logger.log(`Email notification sent to user ${userId}`);
    } catch (error) {
      this.logger.error(`Failed to send email notification to user ${userId}:`, error);
      throw error;
    }
  }
}
```

#### File Storage with AWS S3
```typescript
// src/modules/files/files.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as AWS from 'aws-sdk';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class FilesService {
  private s3: AWS.S3;

  constructor(private configService: ConfigService) {
    this.s3 = new AWS.S3({
      accessKeyId: this.configService.get('AWS_ACCESS_KEY_ID'),
      secretAccessKey: this.configService.get('AWS_SECRET_ACCESS_KEY'),
      region: this.configService.get('AWS_REGION'),
    });
  }

  async uploadFile(file: Express.Multer.File, folder: string = 'uploads'): Promise<string> {
    const key = `${folder}/${uuidv4()}-${file.originalname}`;
    
    const params = {
      Bucket: this.configService.get('AWS_S3_BUCKET'),
      Key: key,
      Body: file.buffer,
      ContentType: file.mimetype,
      ACL: 'private',
    };

    const result = await this.s3.upload(params).promise();
    return result.Location;
  }

  async getSignedUrl(key: string, expiresIn: number = 3600): Promise<string> {
    const params = {
      Bucket: this.configService.get('AWS_S3_BUCKET'),
      Key: key,
      Expires: expiresIn,
    };

    return this.s3.getSignedUrl('getObject', params);
  }

  async deleteFile(key: string): Promise<void> {
    const params = {
      Bucket: this.configService.get('AWS_S3_BUCKET'),
      Key: key,
    };

    await this.s3.deleteObject(params).promise();
  }
}

// src/modules/files/files.controller.ts
import {
  Controller,
  Post,
  Get,
  Delete,
  Param,
  UseInterceptors,
  UploadedFile,
  UseGuards,
  Query,
} from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';
import { FilesService } from './files.service';
import { ApiTags, ApiOperation, ApiConsumes, ApiBody } from '@nestjs/swagger';

@ApiTags('Files')
@Controller('files')
@UseGuards(JwtAuthGuard)
export class FilesController {
  constructor(private readonly filesService: FilesService) {}

  @Post('upload')
  @ApiOperation({ summary: 'Upload a file' })
  @ApiConsumes('multipart/form-data')
  @ApiBody({
    schema: {
      type: 'object',
      properties: {
        file: {
          type: 'string',
          format: 'binary',
        },
        folder: {
          type: 'string',
          description: 'Optional folder name',
        },
      },
    },
  })
  @UseInterceptors(FileInterceptor('file'))
  async uploadFile(
    @UploadedFile() file: Express.Multer.File,
    @Query('folder') folder?: string,
  ) {
    const url = await this.filesService.uploadFile(file, folder);
    return { url };
  }

  @Get('signed-url/:key')
  @ApiOperation({ summary: 'Get signed URL for file access' })
  async getSignedUrl(
    @Param('key') key: string,
    @Query('expiresIn') expiresIn?: number,
  ) {
    const url = await this.filesService.getSignedUrl(key, expiresIn);
    return { url };
  }

  @Delete(':key')
  @ApiOperation({ summary: 'Delete a file' })
  async deleteFile(@Param('key') key: string) {
    await this.filesService.deleteFile(key);
    return { message: 'File deleted successfully' };
  }
}
```

### CI/CD Pipeline Configuration

#### GitHub Actions Workflow
```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: '18'
  POSTGRES_VERSION: '15'
  REDIS_VERSION: '7'

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run database migrations
        run: npm run db:migrate
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb

      - name: Run unit tests
        run: npm run test:unit

      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379

      - name: Run E2E tests
        run: npm run test:e2e
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379

      - name: Generate coverage report
        run: npm run test:coverage

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info

  security:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Run security audit
        run: npm audit --audit-level moderate

      - name: Run dependency vulnerability scan
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
      - run: |
          npm ci
          npx audit-ci --moderate

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: javascript, typescript

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2

  build:
    needs: [test, security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build application
        run: npm run build

      - name: Build Docker image
        run: |
          docker build -t care-management-api:${{ github.sha }} .
          docker tag care-management-api:${{ github.sha }} care-management-api:latest

      - name: Login to container registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Push Docker image
        run: |
          docker tag care-management-api:latest ghcr.io/${{ github.repository_owner }}/care-management-api:latest
          docker tag care-management-api:latest ghcr.io/${{ github.repository_owner }}/care-management-api:${{ github.sha }}
          docker push ghcr.io/${{ github.repository_owner }}/care-management-api:latest
          docker push ghcr.io/${{ github.repository_owner }}/care-management-api:${{ github.sha }}

  deploy-staging:
    needs: [build]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging
    
    steps:
      - name: Deploy to staging
        run: |
          echo "Deploying to staging environment..."
          # Add deployment script here

  deploy-production:
    needs: [build]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - name: Deploy to production
        run: |
          echo "Deploying to production environment..."
          # Add deployment script here
```

#### Docker Configuration
```dockerfile
# Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY prisma ./prisma/

# Install dependencies
RUN npm ci --only=production && npm cache clean --force

# Generate Prisma client
RUN npx prisma generate

# Copy source code
COPY . .

# Build application
RUN npm run build

# Production stage
FROM node:18-alpine AS runner

WORKDIR /app

# Install dumb-init for proper signal handling
RUN apk add --no-cache dumb-init

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nestjs -u 1001

# Copy built application
COPY --from=builder --chown=nestjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nestjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nestjs:nodejs /app/prisma ./prisma
COPY --from=builder --chown=nestjs:nodejs /app/package.json ./package.json

# Switch to non-root user
USER nestjs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD node dist/health-check.js

# Start application
CMD ["dumb-init", "node", "dist/main.js"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:password@db:5432/caredb
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=your-secret-key
      - AWS_ACCESS_KEY_ID=your-access-key
      - AWS_SECRET_ACCESS_KEY=your-secret-key
      - AWS_REGION=us-east-1
      - AWS_S3_BUCKET=care-management-files
    depends_on:
      - db
      - redis
    volumes:
      - ./src:/app/src
      - ./prisma:/app/prisma
    command: npm run start:dev

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=caredb
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

  pgadmin:
    image: dpage/pgadmin4
    environment:
      - PGADMIN_DEFAULT_EMAIL=admin@admin.com
      - PGADMIN_DEFAULT_PASSWORD=admin
    ports:
      - "5050:80"
    depends_on:
      - db

volumes:
  postgres_data:
  redis_data:
```

## Detailed Task Breakdown

### Week 1: Core Infrastructure Setup

#### Day 1-2: Project Structure & Dependencies
- [ ] **Monorepo Setup**: Create monorepo structure with Lerna/Nx
- [ ] **Backend Project**: Initialize NestJS project with TypeScript
- [ ] **Database Setup**: Configure PostgreSQL with Prisma ORM
- [ ] **Redis Configuration**: Set up Redis for caching and sessions
- [ ] **Environment Configuration**: Set up environment variables and validation

#### Day 3-4: Database Implementation
- [ ] **Prisma Schema**: Implement complete Prisma schema
- [ ] **Database Migrations**: Create initial migration files
- [ ] **Seed Data**: Create comprehensive seed data for development
- [ ] **Database Indexes**: Implement performance indexes
- [ ] **Database Testing**: Set up test database configuration

#### Day 5: Authentication Foundation
- [ ] **JWT Configuration**: Set up JWT authentication with refresh tokens
- [ ] **User Module**: Create user module with basic CRUD operations
- [ ] **Auth Guards**: Implement authentication and authorization guards
- [ ] **Password Hashing**: Implement secure password hashing with bcrypt
- [ ] **Session Management**: Set up session management with Redis

### Week 2: Core Business Logic

#### Day 1-2: User Management
- [ ] **User Registration**: Implement user registration with email verification
- [ ] **User Profile Management**: Create user profile CRUD operations
- [ ] **Role Management**: Implement role-based access control
- [ ] **User Invitation System**: Create invitation-based user registration
- [ ] **User Authentication**: Complete login/logout with 2FA support

#### Day 3-4: Care Package Management
- [ ] **Care Package Module**: Create care package CRUD operations
- [ ] **Carer Assignment**: Implement carer assignment to care packages
- [ ] **Data Validation**: Add comprehensive input validation
- [ ] **Soft Delete**: Implement soft delete for care packages
- [ ] **Search & Filtering**: Add search and filtering capabilities

#### Day 5: Task Management
- [ ] **Task Module**: Create task CRUD operations
- [ ] **Task Assignment**: Implement task assignment to care packages
- [ ] **Task Progress**: Set up task progress tracking
- [ ] **Competency Integration**: Link tasks with competency requirements
- [ ] **Task Analytics**: Basic task analytics and reporting

### Week 3: Real-time Features & File Management

#### Day 1-2: Enhanced WebSocket Implementation
- [ ] **WebSocket Gateway**: Set up WebSocket gateway with Socket.io
- [ ] **Enhanced Authentication**: JWT + session validation for WebSocket connections
- [ ] **Real-time Updates**: Implement real-time updates for task progress
- [ ] **User Presence**: Track user online/offline status with Redis
- [ ] **Room Management**: Implement room-based updates with proper cleanup
- [ ] **Connection Management**: Handle reconnection logic and connection limits
- [ ] **Rate Limiting**: Implement WebSocket rate limiting per user

#### Day 3-4: Background Processing
- [ ] **Bull Queue Setup**: Configure Bull queue with Redis
- [ ] **Notification Jobs**: Create notification processing jobs
- [ ] **Email Jobs**: Set up email processing jobs
- [ ] **Report Generation**: Background report generation jobs
- [ ] **Job Monitoring**: Implement job monitoring and error handling

#### Day 5: Enhanced File Management
- [ ] **AWS S3 Integration**: Set up AWS S3 for file storage
- [ ] **File Upload**: Implement secure file upload endpoints with virus scanning
- [ ] **File Validation**: Add comprehensive file type, size, and content validation
- [ ] **Signed URLs**: Generate signed URLs for secure file access
- [ ] **File Cleanup**: Implement file cleanup and garbage collection
- [ ] **CDN Integration**: Set up CloudFront for file delivery
- [ ] **File Versioning**: Implement file versioning and backup strategy

### Week 4: Testing & CI/CD

#### Day 1-2: Testing Framework
- [ ] **Unit Tests**: Set up Jest for unit testing
- [ ] **Integration Tests**: Create integration tests for API endpoints
- [ ] **Database Testing**: Set up database testing with test containers
- [ ] **Mock Services**: Create mock services for external dependencies
- [ ] **Test Coverage**: Set up code coverage reporting

#### Day 3-4: CI/CD Pipeline
- [ ] **GitHub Actions**: Set up GitHub Actions workflows
- [ ] **Docker Configuration**: Create Dockerfile and docker-compose files
- [ ] **Automated Testing**: Configure automated testing in CI
- [ ] **Security Scanning**: Add security scanning to CI pipeline
- [ ] **Deployment Scripts**: Create deployment scripts for staging/production

#### Day 5: Documentation & Monitoring
- [ ] **API Documentation**: Generate Swagger/OpenAPI documentation
- [ ] **Development Documentation**: Create development setup guide
- [ ] **Monitoring Setup**: Configure logging and monitoring
- [ ] **Error Tracking**: Set up error tracking with Sentry
- [ ] **Performance Monitoring**: Add performance monitoring

## Security Considerations

### Authentication & Authorization Security
- **JWT Security**: Short-lived access tokens (15 minutes), secure refresh tokens
- **Password Security**: bcrypt with salt rounds ≥12, password complexity requirements
- **Session Security**: Secure session management with Redis, session timeout
- **Two-Factor Authentication**: TOTP implementation with backup codes
- **Account Security**: Account lockout after failed attempts, suspicious activity detection

### API Security
- **Input Validation**: Comprehensive input validation with class-validator
- **Rate Limiting**: Implement rate limiting to prevent abuse
- **CORS Configuration**: Restrictive CORS policy for production
- **Security Headers**: Use Helmet.js for security headers
- **SQL Injection Prevention**: Use parameterized queries with Prisma

### Data Security
- **Encryption at Rest**: Encrypt sensitive data in database
- **Encryption in Transit**: Use HTTPS/TLS for all communications
- **Data Anonymization**: Anonymize sensitive data in logs
- **Secure File Storage**: Store files securely in AWS S3 with proper permissions
- **Audit Logging**: Log all security-relevant events

### Infrastructure Security
- **Container Security**: Use non-root user in Docker containers
- **Network Security**: Implement proper network segmentation
- **Secret Management**: Use environment variables for secrets
- **Vulnerability Scanning**: Regular vulnerability scanning of dependencies
- **Security Monitoring**: Monitor for security incidents and anomalies

## Testing Requirements

### Unit Testing Strategy
- **Test Coverage**: Target 80% code coverage minimum
- **Test Framework**: Jest with TypeScript support
- **Test Structure**: Arrange-Act-Assert pattern
- **Mock Strategy**: Mock external dependencies and services
- **Test Data**: Use factories for generating test data

### Integration Testing
- **API Testing**: Test all API endpoints with Supertest
- **Database Testing**: Test database interactions with test containers
- **Authentication Testing**: Test authentication and authorization flows
- **Real-time Testing**: Test WebSocket connections and events
- **File Upload Testing**: Test file upload and storage functionality

### End-to-End Testing
- **Critical Path Testing**: Test main user journeys
- **Cross-browser Testing**: Test across different browsers
- **Mobile Testing**: Test mobile responsiveness
- **Performance Testing**: Test API response times and database queries
- **Security Testing**: Test security controls and vulnerabilities

## Deployment Steps

### Development Environment
1. **Local Setup**: Use Docker Compose for local development
2. **Database**: PostgreSQL with sample data
3. **Cache**: Redis for session and caching
4. **File Storage**: Local file system or MinIO
5. **Monitoring**: Basic logging and error tracking

### Staging Environment
1. **Cloud Deployment**: Deploy to AWS/Render staging
2. **Database**: Managed PostgreSQL instance
3. **Cache**: Managed Redis instance
4. **File Storage**: AWS S3 bucket
5. **Monitoring**: Full monitoring and alerting

### Production Environment
1. **High Availability**: Multi-AZ deployment
2. **Load Balancing**: Application load balancer
3. **Auto Scaling**: Auto-scaling groups
4. **Backup**: Automated backup and disaster recovery
5. **Monitoring**: Comprehensive observability stack

## Milestone Criteria

### M1.1: Infrastructure Foundation Complete
- [ ] NestJS application running with basic modules
- [ ] PostgreSQL database connected with Prisma
- [ ] Redis cache operational
- [ ] Docker development environment working
- [ ] Basic authentication system functional

### M1.2: Core Business Logic Implemented
- [ ] User management system complete
- [ ] Care package CRUD operations working
- [ ] Task management system functional
- [ ] Basic competency tracking implemented
- [ ] API endpoints documented with Swagger

### M1.3: Real-time Features Operational
- [ ] WebSocket gateway functional
- [ ] Real-time updates working
- [ ] Background job processing operational
- [ ] File upload/download system working
- [ ] Notification system basic implementation

### M1.4: Production Ready
- [ ] CI/CD pipeline operational
- [ ] Comprehensive test suite passing
- [ ] Security scanning clean
- [ ] Monitoring and logging configured
- [ ] Deployment to staging successful

## Dependencies & Risks

### Technical Dependencies
- **Database Performance**: Requires proper indexing and query optimization
- **Redis Availability**: Critical for session management and caching
- **AWS Services**: Dependency on AWS S3 and other services
- **External APIs**: Integration with email and notification services
- **Container Runtime**: Docker environment for development and deployment

### Business Dependencies
- **User Requirements**: Clear requirements for user management and permissions
- **Security Requirements**: Security review and approval
- **Compliance Requirements**: Regulatory compliance approval
- **Infrastructure Budget**: Budget for cloud services and infrastructure
- **Team Availability**: Full development team availability

### Risk Mitigation Strategies

#### Technical Risks
- **Database Performance Issues**: Implement proper indexing, query optimization, connection pooling
- **Redis Failures**: Implement Redis clustering and failover
- **File Storage Issues**: Implement backup storage and error handling
- **Real-time Communication Problems**: Implement reconnection logic and fallback mechanisms
- **Security Vulnerabilities**: Regular security audits and penetration testing

#### Business Risks
- **Scope Creep**: Clear requirements definition and change management
- **Timeline Delays**: Buffer time and parallel development streams
- **Resource Constraints**: Resource planning and backup team members
- **Quality Issues**: Comprehensive testing and quality assurance
- **Compliance Failures**: Regular compliance reviews and audits

## Success Metrics

### Technical Success Metrics
- **API Performance**: < 200ms response time for 95% of requests
- **Database Performance**: < 50ms query time for 90% of queries
- **System Uptime**: 99.9% availability
- **Test Coverage**: > 80% code coverage
- **Security**: Zero critical vulnerabilities

### Business Success Metrics
- **Development Velocity**: All milestones completed on time
- **Code Quality**: Sonarqube quality gates passing
- **Team Productivity**: High team velocity and satisfaction
- **Stakeholder Satisfaction**: Positive feedback from stakeholders
- **Risk Management**: All risks identified and mitigated

## Next Phase Preparation

### Phase 2 Prerequisites
- [ ] Complete backend API with all core endpoints
- [ ] Authentication and authorization system fully operational
- [ ] Database with all required tables and relationships
- [ ] Real-time communication system working
- [ ] File management system operational

### Handoff Documentation
- [ ] API documentation complete and up-to-date
- [ ] Database schema documentation
- [ ] Development environment setup guide
- [ ] Testing procedures and guidelines
- [ ] Deployment procedures and runbooks

### Knowledge Transfer
- [ ] Team training on architecture and codebase
- [ ] Code review standards and procedures
- [ ] Security best practices and guidelines
- [ ] Testing standards and procedures
- [ ] Monitoring and troubleshooting procedures