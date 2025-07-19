# Phase 2: Robust Backend Core

## Overview
**Duration**: 5-6 weeks  
**Team Size**: 3-4 backend developers + 1 security specialist  
**Objective**: Build production-ready backend with comprehensive authentication, business logic, and security features

## Objectives & Goals

### Primary Objectives
1. **Advanced Authentication**: Implement 2FA, session management, and security controls
2. **Business Logic Implementation**: Complete care package, task, and competency management
3. **Security Hardening**: Implement comprehensive security measures and audit logging
4. **Performance Optimization**: Optimize database queries and implement caching strategies
5. **API Completeness**: Implement all required API endpoints with proper validation

### Success Criteria
- All core business logic APIs operational
- 2FA authentication system working
- Comprehensive audit logging implemented
- Security scans passing with zero critical vulnerabilities
- API response times < 200ms for 95% of requests
- Test coverage > 80% for all core modules

## Technical Specifications

### Enhanced Authentication System

#### Two-Factor Authentication Implementation
```typescript
// src/modules/auth/services/two-factor.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as speakeasy from 'speakeasy';
import * as QRCode from 'qrcode';
import { UsersService } from '../users/users.service';

@Injectable()
export class TwoFactorService {
  constructor(
    private configService: ConfigService,
    private usersService: UsersService,
  ) {}

  async generateTwoFactorSecret(userId: string) {
    const user = await this.usersService.findById(userId);
    
    const secret = speakeasy.generateSecret({
      name: `Care Management (${user.email})`,
      issuer: 'Care Management System',
      length: 32,
    });

    await this.usersService.update(userId, {
      twoFactorSecret: secret.base32,
    });

    const qrCodeDataUrl = await QRCode.toDataURL(secret.otpauth_url);

    return {
      secret: secret.base32,
      qrCode: qrCodeDataUrl,
      backupCodes: await this.generateBackupCodes(userId),
    };
  }

  async verifyTwoFactorToken(userId: string, token: string): Promise<boolean> {
    const user = await this.usersService.findById(userId);
    
    if (!user.twoFactorSecret) {
      return false;
    }

    const isValid = speakeasy.totp.verify({
      secret: user.twoFactorSecret,
      encoding: 'base32',
      token,
      window: 1,
    });

    // Check backup codes if TOTP fails
    if (!isValid) {
      return await this.verifyBackupCode(userId, token);
    }

    return isValid;
  }

  async enableTwoFactor(userId: string, token: string): Promise<boolean> {
    const isValid = await this.verifyTwoFactorToken(userId, token);
    
    if (isValid) {
      await this.usersService.update(userId, {
        twoFactorEnabled: true,
      });
    }

    return isValid;
  }

  async disableTwoFactor(userId: string, token: string): Promise<boolean> {
    const isValid = await this.verifyTwoFactorToken(userId, token);
    
    if (isValid) {
      await this.usersService.update(userId, {
        twoFactorEnabled: false,
        twoFactorSecret: null,
      });
    }

    return isValid;
  }

  private async generateBackupCodes(userId: string): Promise<string[]> {
    const codes = [];
    for (let i = 0; i < 10; i++) {
      codes.push(this.generateRandomCode());
    }

    // Store hashed backup codes
    const hashedCodes = await Promise.all(
      codes.map(code => bcrypt.hash(code, 12))
    );

    await this.usersService.update(userId, {
      backupCodes: hashedCodes,
    });

    return codes;
  }

  private generateRandomCode(): string {
    return Math.random().toString(36).substring(2, 10).toUpperCase();
  }

  private async verifyBackupCode(userId: string, code: string): Promise<boolean> {
    const user = await this.usersService.findById(userId);
    
    if (!user.backupCodes) {
      return false;
    }

    for (const hashedCode of user.backupCodes) {
      const isValid = await bcrypt.compare(code, hashedCode);
      if (isValid) {
        // Remove used backup code
        const updatedCodes = user.backupCodes.filter(c => c !== hashedCode);
        await this.usersService.update(userId, {
          backupCodes: updatedCodes,
        });
        return true;
      }
    }

    return false;
  }
}
```

#### Advanced Session Management
```typescript
// src/modules/auth/services/session.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { RedisService } from '../redis/redis.service';
import { JwtService } from '@nestjs/jwt';
import { User } from '../users/entities/user.entity';

@Injectable()
export class SessionService {
  constructor(
    private redisService: RedisService,
    private jwtService: JwtService,
    private configService: ConfigService,
  ) {}

  async createSession(user: User, ipAddress: string, userAgent: string) {
    const sessionId = this.generateSessionId();
    const accessToken = this.generateAccessToken(user, sessionId);
    const refreshToken = this.generateRefreshToken(user, sessionId);

    const sessionData = {
      userId: user.id,
      sessionId,
      ipAddress,
      userAgent,
      createdAt: new Date(),
      lastActivity: new Date(),
    };

    // Store session in Redis
    await this.redisService.setex(
      `session:${sessionId}`,
      this.configService.get('SESSION_TIMEOUT'),
      JSON.stringify(sessionData)
    );

    // Store refresh token
    await this.redisService.setex(
      `refresh:${refreshToken}`,
      this.configService.get('REFRESH_TOKEN_TIMEOUT'),
      JSON.stringify({ userId: user.id, sessionId })
    );

    return {
      accessToken,
      refreshToken,
      expiresIn: this.configService.get('JWT_EXPIRES_IN'),
    };
  }

  async validateSession(sessionId: string): Promise<any> {
    const sessionData = await this.redisService.get(`session:${sessionId}`);
    
    if (!sessionData) {
      return null;
    }

    const session = JSON.parse(sessionData);
    
    // Update last activity
    session.lastActivity = new Date();
    await this.redisService.setex(
      `session:${sessionId}`,
      this.configService.get('SESSION_TIMEOUT'),
      JSON.stringify(session)
    );

    return session;
  }

  async refreshToken(refreshToken: string): Promise<any> {
    const tokenData = await this.redisService.get(`refresh:${refreshToken}`);
    
    if (!tokenData) {
      throw new Error('Invalid refresh token');
    }

    const { userId, sessionId } = JSON.parse(tokenData);
    const session = await this.validateSession(sessionId);
    
    if (!session) {
      throw new Error('Session expired');
    }

    // Generate new access token
    const user = await this.usersService.findById(userId);
    const newAccessToken = this.generateAccessToken(user, sessionId);

    return {
      accessToken: newAccessToken,
      expiresIn: this.configService.get('JWT_EXPIRES_IN'),
    };
  }

  async revokeSession(sessionId: string): Promise<void> {
    await this.redisService.del(`session:${sessionId}`);
  }

  async revokeAllUserSessions(userId: string): Promise<void> {
    const pattern = `session:*`;
    const keys = await this.redisService.keys(pattern);
    
    for (const key of keys) {
      const sessionData = await this.redisService.get(key);
      if (sessionData) {
        const session = JSON.parse(sessionData);
        if (session.userId === userId) {
          await this.redisService.del(key);
        }
      }
    }
  }

  private generateSessionId(): string {
    return require('crypto').randomBytes(32).toString('hex');
  }

  private generateAccessToken(user: User, sessionId: string): string {
    return this.jwtService.sign({
      sub: user.id,
      email: user.email,
      role: user.role,
      sessionId,
    });
  }

  private generateRefreshToken(user: User, sessionId: string): string {
    return this.jwtService.sign(
      {
        sub: user.id,
        sessionId,
        type: 'refresh',
      },
      {
        expiresIn: this.configService.get('REFRESH_TOKEN_EXPIRES_IN'),
      }
    );
  }
}
```

### Care Package Management System

#### Care Package Service Implementation
```typescript
// src/modules/care-packages/services/care-packages.service.ts
import { Injectable, NotFoundException, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { AuditService } from '../audit/audit.service';
import { CacheService } from '../cache/cache.service';
import { CreateCarePackageDto, UpdateCarePackageDto, CarePackageQueryDto } from '../dto';
import { User } from '../users/entities/user.entity';

@Injectable()
export class CarePackagesService {
  constructor(
    private prisma: PrismaService,
    private auditService: AuditService,
    private cacheService: CacheService,
  ) {}

  async create(createDto: CreateCarePackageDto, user: User) {
    const carePackage = await this.prisma.carePackage.create({
      data: {
        ...createDto,
        createdBy: user.id,
      },
      include: {
        creator: {
          select: { id: true, email: true, profile: true },
        },
        carerAssignments: {
          where: { isActive: true },
          include: {
            carer: {
              select: { id: true, email: true, profile: true },
            },
          },
        },
      },
    });

    // Assign initial carers if provided
    if (createDto.assignedCarerIds && createDto.assignedCarerIds.length > 0) {
      await this.assignCarers(carePackage.id, createDto.assignedCarerIds, user);
    }

    await this.auditService.log({
      userId: user.id,
      entityType: 'CarePackage',
      entityId: carePackage.id,
      action: 'CREATE',
      newValues: carePackage,
    });

    // Clear cache
    await this.cacheService.del('care-packages:*');

    return carePackage;
  }

  async findAll(query: CarePackageQueryDto, user: User) {
    const cacheKey = `care-packages:${JSON.stringify(query)}`;
    const cached = await this.cacheService.get(cacheKey);
    
    if (cached) {
      return cached;
    }

    const { page = 1, limit = 20, search, postcode, isActive } = query;
    const skip = (page - 1) * limit;

    const where: any = {
      deletedAt: null,
    };

    if (search) {
      where.OR = [
        { name: { contains: search, mode: 'insensitive' } },
        { description: { contains: search, mode: 'insensitive' } },
      ];
    }

    if (postcode) {
      where.postcode = { contains: postcode, mode: 'insensitive' };
    }

    if (isActive !== undefined) {
      where.isActive = isActive;
    }

    const [carePackages, total] = await Promise.all([
      this.prisma.carePackage.findMany({
        where,
        include: {
          creator: {
            select: { id: true, email: true, profile: true },
          },
          carerAssignments: {
            where: { isActive: true },
            include: {
              carer: {
                select: { id: true, email: true, profile: true },
              },
            },
          },
          _count: {
            select: {
              taskProgress: true,
            },
          },
        },
        skip,
        take: limit,
        orderBy: { createdAt: 'desc' },
      }),
      this.prisma.carePackage.count({ where }),
    ]);

    const result = {
      carePackages,
      pagination: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
        hasNext: page < Math.ceil(total / limit),
        hasPrev: page > 1,
      },
    };

    await this.cacheService.setex(cacheKey, 300, result); // Cache for 5 minutes

    return result;
  }

  async findOne(id: string, user: User) {
    const carePackage = await this.prisma.carePackage.findFirst({
      where: {
        id,
        deletedAt: null,
      },
      include: {
        creator: {
          select: { id: true, email: true, profile: true },
        },
        carerAssignments: {
          where: { isActive: true },
          include: {
            carer: {
              select: { id: true, email: true, profile: true },
            },
          },
        },
        taskProgress: {
          include: {
            task: true,
            carer: {
              select: { id: true, email: true, profile: true },
            },
          },
        },
      },
    });

    if (!carePackage) {
      throw new NotFoundException('Care package not found');
    }

    return carePackage;
  }

  async update(id: string, updateDto: UpdateCarePackageDto, user: User) {
    const existingPackage = await this.findOne(id, user);
    
    const updatedPackage = await this.prisma.carePackage.update({
      where: { id },
      data: updateDto,
      include: {
        creator: {
          select: { id: true, email: true, profile: true },
        },
        carerAssignments: {
          where: { isActive: true },
          include: {
            carer: {
              select: { id: true, email: true, profile: true },
            },
          },
        },
      },
    });

    await this.auditService.log({
      userId: user.id,
      entityType: 'CarePackage',
      entityId: id,
      action: 'UPDATE',
      oldValues: existingPackage,
      newValues: updatedPackage,
    });

    // Clear cache
    await this.cacheService.del('care-packages:*');

    return updatedPackage;
  }

  async assignCarers(carePackageId: string, carerIds: string[], user: User) {
    const carePackage = await this.findOne(carePackageId, user);
    
    // Validate carer IDs
    const carers = await this.prisma.user.findMany({
      where: {
        id: { in: carerIds },
        role: 'CARER',
        isActive: true,
      },
    });

    if (carers.length !== carerIds.length) {
      throw new BadRequestException('One or more carer IDs are invalid');
    }

    // Create assignments
    const assignments = await Promise.all(
      carerIds.map(carerId =>
        this.prisma.carerAssignment.upsert({
          where: {
            carerId_carePackageId: {
              carerId,
              carePackageId,
            },
          },
          create: {
            carerId,
            carePackageId,
            assignedBy: user.id,
            isActive: true,
          },
          update: {
            isActive: true,
            assignedBy: user.id,
          },
        })
      )
    );

    await this.auditService.log({
      userId: user.id,
      entityType: 'CarePackage',
      entityId: carePackageId,
      action: 'ASSIGN_CARERS',
      newValues: { carerIds },
    });

    return assignments;
  }

  async unassignCarer(carePackageId: string, carerId: string, user: User) {
    const assignment = await this.prisma.carerAssignment.findFirst({
      where: {
        carerId,
        carePackageId,
        isActive: true,
      },
    });

    if (!assignment) {
      throw new NotFoundException('Assignment not found');
    }

    await this.prisma.carerAssignment.update({
      where: { id: assignment.id },
      data: { isActive: false },
    });

    await this.auditService.log({
      userId: user.id,
      entityType: 'CarePackage',
      entityId: carePackageId,
      action: 'UNASSIGN_CARER',
      oldValues: { carerId },
    });

    return { message: 'Carer unassigned successfully' };
  }

  async softDelete(id: string, user: User) {
    const carePackage = await this.findOne(id, user);
    
    await this.prisma.carePackage.update({
      where: { id },
      data: {
        deletedAt: new Date(),
        isActive: false,
      },
    });

    await this.auditService.log({
      userId: user.id,
      entityType: 'CarePackage',
      entityId: id,
      action: 'SOFT_DELETE',
      oldValues: carePackage,
    });

    // Clear cache
    await this.cacheService.del('care-packages:*');

    return { message: 'Care package deleted successfully' };
  }

  async restore(id: string, user: User) {
    const carePackage = await this.prisma.carePackage.findFirst({
      where: { id },
    });

    if (!carePackage) {
      throw new NotFoundException('Care package not found');
    }

    if (!carePackage.deletedAt) {
      throw new BadRequestException('Care package is not deleted');
    }

    const restored = await this.prisma.carePackage.update({
      where: { id },
      data: {
        deletedAt: null,
        isActive: true,
      },
    });

    await this.auditService.log({
      userId: user.id,
      entityType: 'CarePackage',
      entityId: id,
      action: 'RESTORE',
      newValues: restored,
    });

    return restored;
  }
}
```

### Task Management System

#### Task Service with Progress Tracking
```typescript
// src/modules/tasks/services/tasks.service.ts
import { Injectable, NotFoundException, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { AuditService } from '../audit/audit.service';
import { WebSocketGateway } from '../websocket/websocket.gateway';
import { CreateTaskDto, UpdateTaskDto, TaskQueryDto, UpdateTaskProgressDto } from '../dto';
import { User } from '../users/entities/user.entity';

@Injectable()
export class TasksService {
  constructor(
    private prisma: PrismaService,
    private auditService: AuditService,
    private websocketGateway: WebSocketGateway,
  ) {}

  async create(createDto: CreateTaskDto, user: User) {
    const task = await this.prisma.task.create({
      data: {
        ...createDto,
        createdBy: user.id,
      },
      include: {
        creator: {
          select: { id: true, email: true, profile: true },
        },
        _count: {
          select: {
            progress: true,
            competencies: true,
          },
        },
      },
    });

    await this.auditService.log({
      userId: user.id,
      entityType: 'Task',
      entityId: task.id,
      action: 'CREATE',
      newValues: task,
    });

    return task;
  }

  async findAll(query: TaskQueryDto, user: User) {
    const { page = 1, limit = 20, search, category, isActive } = query;
    const skip = (page - 1) * limit;

    const where: any = {
      deletedAt: null,
    };

    if (search) {
      where.OR = [
        { name: { contains: search, mode: 'insensitive' } },
        { description: { contains: search, mode: 'insensitive' } },
      ];
    }

    if (category) {
      where.category = { contains: category, mode: 'insensitive' };
    }

    if (isActive !== undefined) {
      where.isActive = isActive;
    }

    const [tasks, total] = await Promise.all([
      this.prisma.task.findMany({
        where,
        include: {
          creator: {
            select: { id: true, email: true, profile: true },
          },
          _count: {
            select: {
              progress: true,
              competencies: true,
            },
          },
        },
        skip,
        take: limit,
        orderBy: { createdAt: 'desc' },
      }),
      this.prisma.task.count({ where }),
    ]);

    // Calculate average progress for each task
    const tasksWithProgress = await Promise.all(
      tasks.map(async (task) => {
        const progressData = await this.prisma.taskProgress.aggregate({
          where: { taskId: task.id },
          _avg: {
            currentCount: true,
          },
        });

        const averageProgress = progressData._avg.currentCount || 0;
        const progressPercentage = (averageProgress / task.targetCount) * 100;

        return {
          ...task,
          averageProgress: Math.min(progressPercentage, 100),
        };
      })
    );

    return {
      tasks: tasksWithProgress,
      pagination: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
        hasNext: page < Math.ceil(total / limit),
        hasPrev: page > 1,
      },
    };
  }

  async findOne(id: string, user: User) {
    const task = await this.prisma.task.findFirst({
      where: {
        id,
        deletedAt: null,
      },
      include: {
        creator: {
          select: { id: true, email: true, profile: true },
        },
        progress: {
          include: {
            carer: {
              select: { id: true, email: true, profile: true },
            },
            carePackage: {
              select: { id: true, name: true, postcode: true },
            },
          },
        },
        competencies: {
          include: {
            carer: {
              select: { id: true, email: true, profile: true },
            },
          },
        },
      },
    });

    if (!task) {
      throw new NotFoundException('Task not found');
    }

    return task;
  }

  async update(id: string, updateDto: UpdateTaskDto, user: User) {
    const existingTask = await this.findOne(id, user);
    
    const updatedTask = await this.prisma.task.update({
      where: { id },
      data: updateDto,
      include: {
        creator: {
          select: { id: true, email: true, profile: true },
        },
        _count: {
          select: {
            progress: true,
            competencies: true,
          },
        },
      },
    });

    await this.auditService.log({
      userId: user.id,
      entityType: 'Task',
      entityId: id,
      action: 'UPDATE',
      oldValues: existingTask,
      newValues: updatedTask,
    });

    return updatedTask;
  }

  async updateProgress(
    taskId: string,
    carerId: string,
    carePackageId: string,
    updateDto: UpdateTaskProgressDto,
    user: User
  ) {
    const task = await this.findOne(taskId, user);
    
    // Validate carer assignment to care package
    const assignment = await this.prisma.carerAssignment.findFirst({
      where: {
        carerId,
        carePackageId,
        isActive: true,
      },
    });

    if (!assignment) {
      throw new BadRequestException('Carer is not assigned to this care package');
    }

    const existingProgress = await this.prisma.taskProgress.findFirst({
      where: {
        taskId,
        carerId,
        carePackageId,
      },
    });

    const newCount = Math.min(updateDto.currentCount, task.targetCount);
    const isCompleted = newCount >= task.targetCount;

    let progress;
    if (existingProgress) {
      progress = await this.prisma.taskProgress.update({
        where: { id: existingProgress.id },
        data: {
          currentCount: newCount,
          isCompleted,
          completedAt: isCompleted ? new Date() : null,
          lastUpdated: new Date(),
        },
        include: {
          task: true,
          carer: {
            select: { id: true, email: true, profile: true },
          },
          carePackage: {
            select: { id: true, name: true, postcode: true },
          },
        },
      });
    } else {
      progress = await this.prisma.taskProgress.create({
        data: {
          taskId,
          carerId,
          carePackageId,
          currentCount: newCount,
          targetCount: task.targetCount,
          isCompleted,
          completedAt: isCompleted ? new Date() : null,
        },
        include: {
          task: true,
          carer: {
            select: { id: true, email: true, profile: true },
          },
          carePackage: {
            select: { id: true, name: true, postcode: true },
          },
        },
      });
    }

    await this.auditService.log({
      userId: user.id,
      entityType: 'TaskProgress',
      entityId: progress.id,
      action: existingProgress ? 'UPDATE' : 'CREATE',
      oldValues: existingProgress,
      newValues: progress,
    });

    // Send real-time update
    this.websocketGateway.emitTaskProgressUpdate(carerId, taskId, progress);

    return progress;
  }

  async getCarerProgress(carerId: string, user: User) {
    const progress = await this.prisma.taskProgress.findMany({
      where: { carerId },
      include: {
        task: {
          select: {
            id: true,
            name: true,
            description: true,
            category: true,
            targetCount: true,
            isCompetencyRequired: true,
            minimumCompetencyLevel: true,
          },
        },
        carePackage: {
          select: {
            id: true,
            name: true,
            postcode: true,
          },
        },
      },
      orderBy: { lastUpdated: 'desc' },
    });

    // Get carer competencies
    const competencies = await this.prisma.carerCompetency.findMany({
      where: { carerId },
      include: {
        task: {
          select: {
            id: true,
            name: true,
            category: true,
          },
        },
      },
    });

    const competencyMap = new Map();
    competencies.forEach(comp => {
      competencyMap.set(comp.taskId, comp);
    });

    const progressWithCompetencies = progress.map(p => {
      const competency = competencyMap.get(p.taskId);
      const progressPercentage = (p.currentCount / p.targetCount) * 100;
      
      return {
        ...p,
        progressPercentage: Math.min(progressPercentage, 100),
        competency: competency || null,
      };
    });

    return progressWithCompetencies;
  }

  async softDelete(id: string, user: User) {
    const task = await this.findOne(id, user);
    
    await this.prisma.task.update({
      where: { id },
      data: {
        deletedAt: new Date(),
        isActive: false,
      },
    });

    await this.auditService.log({
      userId: user.id,
      entityType: 'Task',
      entityId: id,
      action: 'SOFT_DELETE',
      oldValues: task,
    });

    return { message: 'Task deleted successfully' };
  }
}
```

### Competency Management System

#### Competency Service Implementation
```typescript
// src/modules/competencies/services/competencies.service.ts
import { Injectable, NotFoundException, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { AuditService } from '../audit/audit.service';
import { WebSocketGateway } from '../websocket/websocket.gateway';
import { NotificationService } from '../notifications/notification.service';
import { UpdateCompetencyDto, CompetencyQueryDto } from '../dto';
import { User } from '../users/entities/user.entity';
import { CompetencyLevel } from '@prisma/client';

@Injectable()
export class CompetenciesService {
  constructor(
    private prisma: PrismaService,
    private auditService: AuditService,
    private websocketGateway: WebSocketGateway,
    private notificationService: NotificationService,
  ) {}

  async getCarerCompetencies(carerId: string, user: User) {
    const competencies = await this.prisma.carerCompetency.findMany({
      where: { carerId },
      include: {
        task: {
          select: {
            id: true,
            name: true,
            description: true,
            category: true,
            targetCount: true,
            isCompetencyRequired: true,
            minimumCompetencyLevel: true,
          },
        },
        carer: {
          select: {
            id: true,
            email: true,
            profile: true,
          },
        },
        assessor: {
          select: {
            id: true,
            email: true,
            profile: true,
          },
        },
      },
      orderBy: { updatedAt: 'desc' },
    });

    // Get task progress for each competency
    const competenciesWithProgress = await Promise.all(
      competencies.map(async (competency) => {
        const progress = await this.prisma.taskProgress.findMany({
          where: {
            carerId,
            taskId: competency.taskId,
          },
          include: {
            carePackage: {
              select: {
                id: true,
                name: true,
                postcode: true,
              },
            },
          },
        });

        const totalProgress = progress.reduce((sum, p) => sum + p.currentCount, 0);
        const totalTarget = progress.reduce((sum, p) => sum + p.targetCount, 0);
        const progressPercentage = totalTarget > 0 ? (totalProgress / totalTarget) * 100 : 0;

        return {
          ...competency,
          progress: Math.min(progressPercentage, 100),
          taskProgress: progress,
        };
      })
    );

    return competenciesWithProgress;
  }

  async updateCompetency(
    carerId: string,
    taskId: string,
    updateDto: UpdateCompetencyDto,
    user: User
  ) {
    // Validate that the user has permission to assess
    if (user.role !== 'ADMIN' && user.role !== 'ASSESSOR') {
      throw new BadRequestException('Only admins and assessors can update competencies');
    }

    const existingCompetency = await this.prisma.carerCompetency.findFirst({
      where: {
        carerId,
        taskId,
      },
    });

    let competency;
    if (existingCompetency) {
      competency = await this.prisma.carerCompetency.update({
        where: { id: existingCompetency.id },
        data: {
          level: updateDto.level,
          assessedBy: user.id,
          assessedAt: new Date(),
          notes: updateDto.notes,
          isConfirmed: false, // Reset confirmation when competency is updated
        },
        include: {
          task: true,
          carer: {
            select: { id: true, email: true, profile: true },
          },
          assessor: {
            select: { id: true, email: true, profile: true },
          },
        },
      });
    } else {
      competency = await this.prisma.carerCompetency.create({
        data: {
          carerId,
          taskId,
          level: updateDto.level,
          assessedBy: user.id,
          assessedAt: new Date(),
          notes: updateDto.notes,
        },
        include: {
          task: true,
          carer: {
            select: { id: true, email: true, profile: true },
          },
          assessor: {
            select: { id: true, email: true, profile: true },
          },
        },
      });
    }

    // If competency is below required level, reset task progress
    if (competency.level === CompetencyLevel.NOT_ASSESSED ||
        (competency.task.minimumCompetencyLevel === CompetencyLevel.COMPETENT && 
         competency.level === CompetencyLevel.NOT_ASSESSED)) {
      await this.resetTaskProgress(carerId, taskId, user);
    }

    await this.auditService.log({
      userId: user.id,
      entityType: 'CarerCompetency',
      entityId: competency.id,
      action: existingCompetency ? 'UPDATE' : 'CREATE',
      oldValues: existingCompetency,
      newValues: competency,
    });

    // Send real-time update
    this.websocketGateway.emitCompetencyUpdate(carerId, competency);

    // Send notification to carer
    await this.notificationService.sendNotification({
      userId: carerId,
      type: 'competency_updated',
      title: 'Competency Updated',
      message: `Your competency for ${competency.task.name} has been updated to ${competency.level}`,
      data: { competencyId: competency.id, taskId },
    });

    return competency;
  }

  async confirmCompetency(carerId: string, taskId: string, user: User) {
    // Only the carer can confirm their own competency
    if (user.id !== carerId) {
      throw new BadRequestException('You can only confirm your own competencies');
    }

    const competency = await this.prisma.carerCompetency.findFirst({
      where: {
        carerId,
        taskId,
      },
    });

    if (!competency) {
      throw new NotFoundException('Competency not found');
    }

    const updatedCompetency = await this.prisma.carerCompetency.update({
      where: { id: competency.id },
      data: {
        isConfirmed: true,
        confirmedAt: new Date(),
      },
      include: {
        task: true,
        carer: {
          select: { id: true, email: true, profile: true },
        },
        assessor: {
          select: { id: true, email: true, profile: true },
        },
      },
    });

    await this.auditService.log({
      userId: user.id,
      entityType: 'CarerCompetency',
      entityId: competency.id,
      action: 'CONFIRM',
      newValues: { isConfirmed: true, confirmedAt: new Date() },
    });

    return updatedCompetency;
  }

  async resetTaskProgress(carerId: string, taskId: string, user: User) {
    const progressRecords = await this.prisma.taskProgress.findMany({
      where: {
        carerId,
        taskId,
      },
    });

    const resetPromises = progressRecords.map(progress =>
      this.prisma.taskProgress.update({
        where: { id: progress.id },
        data: {
          currentCount: 0,
          isCompleted: false,
          completedAt: null,
          lastUpdated: new Date(),
        },
      })
    );

    await Promise.all(resetPromises);

    await this.auditService.log({
      userId: user.id,
      entityType: 'TaskProgress',
      entityId: taskId,
      action: 'RESET',
      newValues: { carerId, taskId, reason: 'competency_updated' },
    });

    // Send notification to carer
    await this.notificationService.sendNotification({
      userId: carerId,
      type: 'task_progress_reset',
      title: 'Task Progress Reset',
      message: `Your progress for tasks has been reset due to competency changes`,
      data: { taskId },
    });
  }

  async getCompetencyStatistics(query: CompetencyQueryDto, user: User) {
    const { carePackageId, taskId } = query;

    const where: any = {};
    if (carePackageId) {
      // Get carers assigned to the care package
      const assignments = await this.prisma.carerAssignment.findMany({
        where: {
          carePackageId,
          isActive: true,
        },
        select: { carerId: true },
      });
      where.carerId = { in: assignments.map(a => a.carerId) };
    }

    if (taskId) {
      where.taskId = taskId;
    }

    const competencies = await this.prisma.carerCompetency.findMany({
      where,
      include: {
        task: {
          select: {
            id: true,
            name: true,
            category: true,
          },
        },
        carer: {
          select: {
            id: true,
            email: true,
            profile: true,
          },
        },
      },
    });

    // Calculate statistics
    const stats = {
      total: competencies.length,
      byLevel: {
        [CompetencyLevel.NOT_ASSESSED]: 0,
        [CompetencyLevel.COMPETENT]: 0,
        [CompetencyLevel.PROFICIENT]: 0,
        [CompetencyLevel.EXPERT]: 0,
      },
      byTask: new Map(),
      byCategory: new Map(),
    };

    competencies.forEach(competency => {
      // Count by level
      stats.byLevel[competency.level]++;

      // Count by task
      const taskKey = competency.task.name;
      if (!stats.byTask.has(taskKey)) {
        stats.byTask.set(taskKey, {
          total: 0,
          byLevel: { ...stats.byLevel },
        });
      }
      const taskStats = stats.byTask.get(taskKey);
      taskStats.total++;
      taskStats.byLevel[competency.level]++;

      // Count by category
      if (competency.task.category) {
        const categoryKey = competency.task.category;
        if (!stats.byCategory.has(categoryKey)) {
          stats.byCategory.set(categoryKey, {
            total: 0,
            byLevel: { ...stats.byLevel },
          });
        }
        const categoryStats = stats.byCategory.get(categoryKey);
        categoryStats.total++;
        categoryStats.byLevel[competency.level]++;
      }
    });

    return {
      ...stats,
      byTask: Object.fromEntries(stats.byTask),
      byCategory: Object.fromEntries(stats.byCategory),
    };
  }
}
```

### Comprehensive Audit Logging

#### Audit Service Implementation
```typescript
// src/modules/audit/services/audit.service.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { CreateAuditLogDto, AuditLogQueryDto } from '../dto';
import { User } from '../users/entities/user.entity';

@Injectable()
export class AuditService {
  constructor(private prisma: PrismaService) {}

  async log(auditData: CreateAuditLogDto) {
    return this.prisma.auditLog.create({
      data: auditData,
    });
  }

  async findAll(query: AuditLogQueryDto, user: User) {
    const {
      page = 1,
      limit = 50,
      entityType,
      entityId,
      action,
      userId,
      dateFrom,
      dateTo,
    } = query;
    
    const skip = (page - 1) * limit;
    const where: any = {};

    if (entityType) {
      where.entityType = entityType;
    }

    if (entityId) {
      where.entityId = entityId;
    }

    if (action) {
      where.action = action;
    }

    if (userId) {
      where.userId = userId;
    }

    if (dateFrom || dateTo) {
      where.createdAt = {};
      if (dateFrom) {
        where.createdAt.gte = new Date(dateFrom);
      }
      if (dateTo) {
        where.createdAt.lte = new Date(dateTo);
      }
    }

    const [logs, total] = await Promise.all([
      this.prisma.auditLog.findMany({
        where,
        include: {
          user: {
            select: {
              id: true,
              email: true,
              role: true,
              profile: {
                select: {
                  firstName: true,
                  lastName: true,
                },
              },
            },
          },
        },
        skip,
        take: limit,
        orderBy: { createdAt: 'desc' },
      }),
      this.prisma.auditLog.count({ where }),
    ]);

    return {
      logs,
      pagination: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
        hasNext: page < Math.ceil(total / limit),
        hasPrev: page > 1,
      },
    };
  }

  async exportLogs(query: AuditLogQueryDto, user: User) {
    const { entityType, entityId, action, userId, dateFrom, dateTo } = query;
    
    const where: any = {};

    if (entityType) where.entityType = entityType;
    if (entityId) where.entityId = entityId;
    if (action) where.action = action;
    if (userId) where.userId = userId;
    if (dateFrom || dateTo) {
      where.createdAt = {};
      if (dateFrom) where.createdAt.gte = new Date(dateFrom);
      if (dateTo) where.createdAt.lte = new Date(dateTo);
    }

    const logs = await this.prisma.auditLog.findMany({
      where,
      include: {
        user: {
          select: {
            id: true,
            email: true,
            role: true,
            profile: {
              select: {
                firstName: true,
                lastName: true,
              },
            },
          },
        },
      },
      orderBy: { createdAt: 'desc' },
    });

    return logs;
  }

  async getAuditStatistics(dateFrom?: string, dateTo?: string) {
    const where: any = {};
    if (dateFrom || dateTo) {
      where.createdAt = {};
      if (dateFrom) where.createdAt.gte = new Date(dateFrom);
      if (dateTo) where.createdAt.lte = new Date(dateTo);
    }

    const [totalLogs, actionStats, entityStats, userStats] = await Promise.all([
      this.prisma.auditLog.count({ where }),
      this.prisma.auditLog.groupBy({
        by: ['action'],
        where,
        _count: true,
      }),
      this.prisma.auditLog.groupBy({
        by: ['entityType'],
        where,
        _count: true,
      }),
      this.prisma.auditLog.groupBy({
        by: ['userId'],
        where,
        _count: true,
        take: 10,
        orderBy: { _count: { userId: 'desc' } },
      }),
    ]);

    return {
      totalLogs,
      actionStats,
      entityStats,
      userStats,
    };
  }
}
```

## Detailed Task Breakdown

### Week 1: Enhanced Authentication & Security

#### Day 1-2: Two-Factor Authentication
- [ ] **2FA Setup**: Implement TOTP-based 2FA with speakeasy
- [ ] **QR Code Generation**: Generate QR codes for authenticator apps
- [ ] **Backup Codes**: Implement backup code system
- [ ] **2FA Verification**: Create verification endpoints
- [ ] **2FA Management**: Enable/disable 2FA functionality

#### Day 3-4: Advanced Session Management
- [ ] **Session Storage**: Implement Redis-based session storage
- [ ] **Session Validation**: Create session validation middleware
- [ ] **Session Lifecycle**: Handle session creation, refresh, and revocation
- [ ] **Multi-Session Support**: Allow multiple active sessions per user
- [ ] **Session Security**: Implement session hijacking protection

#### Day 5: Enhanced Security Hardening
- [ ] **Rate Limiting**: Implement API rate limiting with sliding window
- [ ] **Account Lockout**: Create account lockout mechanism with progressive delays
- [ ] **Password Policy**: Enforce strong password requirements with complexity scoring
- [ ] **Security Headers**: Add comprehensive security headers with CSP
- [ ] **Input Sanitization**: Implement input sanitization and validation with DOMPurify
- [ ] **XSS Prevention**: Implement comprehensive XSS prevention measures
- [ ] **GDPR Compliance**: Add data deletion and export procedures
- [ ] **Security Monitoring**: Implement security event logging and monitoring

### Week 2: Care Package Management

#### Day 1-2: Enhanced CRUD Operations
- [ ] **Care Package Service**: Implement comprehensive care package service
- [ ] **CRUD Endpoints**: Create all CRUD endpoints with validation
- [ ] **Search & Filtering**: Add advanced search and filtering capabilities
- [ ] **Pagination**: Implement efficient pagination with cursor-based pagination
- [ ] **Caching**: Add Redis caching with proper cache invalidation strategy
- [ ] **Connection Pooling**: Implement database connection pooling
- [ ] **Query Optimization**: Optimize database queries with proper indexing

#### Day 3-4: Carer Assignment System
- [ ] **Assignment Logic**: Implement carer assignment to care packages
- [ ] **Assignment Validation**: Validate carer assignments
- [ ] **Assignment History**: Track assignment history
- [ ] **Bulk Operations**: Support bulk assignment operations
- [ ] **Assignment Notifications**: Send notifications on assignment changes

#### Day 5: Advanced Features
- [ ] **Soft Delete**: Implement soft delete functionality
- [ ] **Data Validation**: Add comprehensive data validation
- [ ] **Audit Integration**: Integrate audit logging for all operations
- [ ] **Error Handling**: Implement comprehensive error handling
- [ ] **Performance Optimization**: Optimize database queries

### Week 3: Task Management & Progress Tracking

#### Day 1-2: Task Management System
- [ ] **Task Service**: Implement comprehensive task service
- [ ] **Task CRUD**: Create all task CRUD operations
- [ ] **Task Categories**: Implement task categorization
- [ ] **Task Validation**: Add task validation rules
- [ ] **Task Analytics**: Basic task analytics and reporting

#### Day 3-4: Progress Tracking
- [ ] **Progress Service**: Implement task progress tracking
- [ ] **Progress Updates**: Create progress update endpoints
- [ ] **Progress Validation**: Validate progress updates
- [ ] **Real-time Updates**: Implement real-time progress updates
- [ ] **Progress Analytics**: Calculate progress statistics

#### Day 5: Integration & Optimization
- [ ] **Competency Integration**: Link tasks with competency requirements
- [ ] **Progress Reporting**: Generate progress reports
- [ ] **Performance Optimization**: Optimize progress queries
- [ ] **Error Recovery**: Implement error recovery mechanisms
- [ ] **Testing**: Comprehensive testing of task management

### Week 4: Competency Management

#### Day 1-2: Competency System
- [ ] **Competency Service**: Implement comprehensive competency service
- [ ] **Competency CRUD**: Create competency management endpoints
- [ ] **Competency Validation**: Validate competency assessments
- [ ] **Competency Confirmation**: Allow carers to confirm competencies
- [ ] **Competency History**: Track competency assessment history

#### Day 3-4: Assessment Integration
- [ ] **Assessment Logic**: Implement assessment-based competency updates
- [ ] **Progress Reset**: Reset progress when competency changes
- [ ] **Notification System**: Send notifications for competency changes
- [ ] **Competency Analytics**: Generate competency statistics
- [ ] **Competency Reporting**: Create competency reports

#### Day 5: Advanced Features
- [ ] **Competency Recommendations**: Suggest competency improvements
- [ ] **Competency Workflows**: Implement competency approval workflows
- [ ] **Bulk Operations**: Support bulk competency operations
- [ ] **Integration Testing**: Test competency integration with other systems
- [ ] **Performance Optimization**: Optimize competency queries

### Week 5: Audit System & Performance

#### Day 1-2: Enhanced Audit System
- [ ] **Audit Service**: Implement comprehensive audit logging
- [ ] **Audit Endpoints**: Create audit log query endpoints
- [ ] **Audit Export**: Implement audit log export functionality
- [ ] **Audit Statistics**: Generate audit statistics
- [ ] **Audit Retention**: Implement audit log retention policies with automated cleanup
- [ ] **GDPR Compliance**: Add personal data audit trails and deletion tracking
- [ ] **Performance Optimization**: Optimize audit log queries with proper indexing

#### Day 3-4: Performance Optimization
- [ ] **Database Optimization**: Optimize database queries and indexes
- [ ] **Caching Strategy**: Implement comprehensive caching strategy
- [ ] **Query Optimization**: Optimize slow queries
- [ ] **Connection Pooling**: Implement database connection pooling
- [ ] **Performance Monitoring**: Add performance monitoring

#### Day 5: Integration & Testing
- [ ] **Integration Testing**: Comprehensive integration testing
- [ ] **Performance Testing**: Load testing and performance validation
- [ ] **Security Testing**: Security vulnerability testing
- [ ] **API Documentation**: Complete API documentation
- [ ] **Code Review**: Comprehensive code review

### Week 6: Final Integration & Deployment

#### Day 1-2: Final Integration
- [ ] **End-to-End Testing**: Complete end-to-end testing
- [ ] **Bug Fixes**: Fix any identified bugs
- [ ] **Performance Tuning**: Final performance tuning
- [ ] **Security Hardening**: Final security hardening
- [ ] **Documentation**: Complete documentation

#### Day 3-4: Deployment Preparation
- [ ] **Deployment Scripts**: Create deployment scripts
- [ ] **Environment Configuration**: Configure production environment
- [ ] **Monitoring Setup**: Set up monitoring and alerting
- [ ] **Backup Strategy**: Implement backup strategy
- [ ] **Rollback Plan**: Create rollback procedures

#### Day 5: Production Deployment
- [ ] **Staging Deployment**: Deploy to staging environment
- [ ] **Staging Testing**: Test in staging environment
- [ ] **Production Deployment**: Deploy to production
- [ ] **Post-Deployment Testing**: Validate production deployment
- [ ] **Go-Live**: Complete go-live procedures

## Security Considerations

### Enhanced Security Measures
- **Two-Factor Authentication**: TOTP-based 2FA with backup codes
- **Session Security**: Secure session management with Redis
- **Rate Limiting**: Comprehensive rate limiting to prevent abuse
- **Input Validation**: Strict input validation and sanitization
- **SQL Injection Prevention**: Parameterized queries with Prisma
- **XSS Prevention**: Output encoding and CSP headers
- **CSRF Protection**: CSRF token validation
- **Security Headers**: Comprehensive security headers with Helmet

### Audit & Compliance
- **Comprehensive Logging**: Log all security-relevant events
- **Audit Trail**: Complete audit trail for all operations
- **Data Protection**: Encrypt sensitive data at rest and in transit
- **Access Control**: Granular role-based access control
- **Compliance**: GDPR and data protection compliance

## Testing Requirements

### Unit Testing
- **Coverage Target**: 85% code coverage for all modules
- **Test Framework**: Jest with comprehensive test suites
- **Mock Strategy**: Mock external dependencies and services
- **Test Data**: Factory-based test data generation
- **Security Testing**: Test security controls and validations

### Integration Testing
- **API Testing**: Test all API endpoints with real database
- **Authentication Testing**: Test authentication and authorization flows
- **Real-time Testing**: Test WebSocket connections and events
- **Database Testing**: Test database operations and constraints
- **Performance Testing**: Test API performance and response times

### Security Testing
- **Vulnerability Scanning**: Automated vulnerability scanning
- **Penetration Testing**: Manual security testing
- **Authentication Testing**: Test authentication mechanisms
- **Authorization Testing**: Test access control enforcement
- **Input Validation Testing**: Test input validation and sanitization

## Deployment Steps

### Staging Environment
1. **Database Migration**: Run database migrations
2. **Environment Setup**: Configure staging environment variables
3. **Application Deployment**: Deploy application to staging
4. **Integration Testing**: Run integration tests in staging
5. **Performance Testing**: Run performance tests

### Production Environment
1. **Pre-deployment Checks**: Verify all prerequisites
2. **Database Backup**: Create database backup
3. **Application Deployment**: Deploy to production
4. **Health Checks**: Verify application health
5. **Monitoring**: Activate monitoring and alerting

## Milestone Criteria

### M2.1: Enhanced Authentication Complete
- [ ] Two-factor authentication system operational
- [ ] Advanced session management working
- [ ] Security hardening implemented
- [ ] Account lockout mechanism functional
- [ ] Security tests passing

### M2.2: Care Package Management Complete
- [ ] All CRUD operations working
- [ ] Carer assignment system operational
- [ ] Search and filtering working
- [ ] Audit logging implemented
- [ ] Performance targets met

### M2.3: Task Management & Progress Complete
- [ ] Task management system operational
- [ ] Progress tracking working
- [ ] Real-time updates functional
- [ ] Integration with competencies working
- [ ] Analytics and reporting operational

### M2.4: Competency Management Complete
- [ ] Competency assessment system working
- [ ] Competency confirmation process operational
- [ ] Integration with task progress working
- [ ] Notification system functional
- [ ] Competency analytics working

### M2.5: Production Ready Backend
- [ ] All APIs documented and tested
- [ ] Security scans passing
- [ ] Performance targets met
- [ ] Audit system operational
- [ ] Ready for production deployment

## Dependencies & Risks

### Technical Dependencies
- **Database Performance**: Requires optimized database queries
- **Redis Availability**: Critical for session management and caching
- **Security Review**: Security team review and approval
- **Performance Testing**: Load testing infrastructure
- **Monitoring Tools**: Monitoring and alerting tools

### Business Dependencies
- **Security Requirements**: Clear security requirements and approval
- **Compliance Requirements**: Regulatory compliance approval
- **Performance Requirements**: Clear performance targets
- **User Acceptance**: User acceptance testing
- **Documentation**: Complete documentation requirements

### Risk Mitigation
- **Performance Issues**: Comprehensive performance testing and optimization
- **Security Vulnerabilities**: Regular security testing and reviews
- **Data Loss**: Comprehensive backup and recovery procedures
- **Integration Issues**: Thorough integration testing
- **Scalability Issues**: Load testing and capacity planning

## Success Metrics

### Technical Metrics
- **API Performance**: < 200ms response time for 95% of requests
- **Database Performance**: < 50ms query time for 90% of queries
- **Test Coverage**: > 85% code coverage
- **Security**: Zero critical vulnerabilities
- **Uptime**: 99.9% availability

### Business Metrics
- **Feature Completeness**: 100% of planned features implemented
- **Quality**: All quality gates passing
- **User Satisfaction**: Positive feedback from stakeholders
- **Performance**: All performance targets met
- **Security**: All security requirements met

## Next Phase Preparation

### Phase 3 Prerequisites
- [ ] Complete backend API with all endpoints
- [ ] Authentication and authorization fully operational
- [ ] Database with all data and relationships
- [ ] Real-time communication working
- [ ] Security measures implemented and tested

### Handoff Documentation
- [ ] Complete API documentation
- [ ] Database schema and relationships
- [ ] Security implementation guide
- [ ] Performance optimization guide
- [ ] Troubleshooting procedures

### Knowledge Transfer
- [ ] Backend architecture training
- [ ] Security best practices training
- [ ] API usage training
- [ ] Database optimization training
- [ ] Monitoring and troubleshooting training