# Phase 7: Identity & Access Service Implementation Guide

**Duration:** 4 weeks (Weeks 31–34)  
**Risk Level:** Critical (authentication is mandatory)  
**Dependencies:** Phase 0 (API Gateway), all prior phases (depend on users)

---

## Overview

**NOTE:** The `lumina-auth-service` (NestJS + Prisma + Clerk) already exists in your workspace as a proof-of-concept implementation of Phase 7. This guide expands it to production scale and integrates with all other services.

---

## 1. Complete Service Architecture

The existing auth-service structure:

```
lumina-auth-service/
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   ├── clerk/
│   │   ├── clerk-webhook.guard.ts       # Webhook validation
│   │   ├── clerk.module.ts
│   │   ├── clerk.service.ts             # Clerk API integration
│   │   └── clerk.controller.ts          # Webhook receiver
│   ├── auth/
│   │   ├── auth.module.ts
│   │   ├── auth.service.ts
│   │   ├── auth.controller.ts
│   │   ├── strategies/
│   │   │   ├── jwt.strategy.ts
│   │   │   └── clerk.strategy.ts
│   │   └── decorators/
│   │       └── current-user.decorator.ts
│   ├── user/
│   │   ├── user.module.ts
│   │   ├── user.service.ts
│   │   ├── user.controller.ts
│   │   └── dto/
│   │       ├── create-user.dto.ts
│   │       └── update-user.dto.ts
│   ├── role/
│   │   ├── role.module.ts
│   │   ├── role.service.ts
│   │   ├── role.controller.ts
│   │   └── dto/
│   ├── invitation/
│   │   ├── invitation.module.ts
│   │   ├── invitation.service.ts
│   │   ├── invitation.controller.ts
│   │   ├── decorators/
│   │   └── dto/
│   ├── organization/
│   │   ├── organization.module.ts
│   │   ├── organization.service.ts
│   │   ├── organization.controller.ts
│   │   └── dto/
│   ├── rbac/
│   │   ├── rbac.module.ts
│   │   ├── rbac.service.ts             # Role-based access control
│   │   ├── roles.enum.ts
│   │   └── permissions.enum.ts
│   ├── audit/
│   │   ├── audit.module.ts
│   │   ├── audit.service.ts            # Audit logging
│   │   └── audit.interceptor.ts
│   ├── events/
│   │   ├── events.module.ts
│   │   ├── events.service.ts
│   │   └── handlers/
│   │       ├── user-created.handler.ts
│   │       ├── user-deleted.handler.ts
│   │       └── role-updated.handler.ts
│   ├── cache/
│   │   └── jwt-cache.service.ts        # Cache invalidation on role change
│   ├── common/
│   │   ├── decorators/
│   │   │   └── current-user.decorator.ts
│   │   ├── filters/
│   │   │   └── all-exceptions.filter.ts
│   │   ├── guards/
│   │   │   ├── jwt.guard.ts
│   │   │   ├── rbac.guard.ts
│   │   │   └── clerk-auth.guard.ts
│   │   ├── interceptors/
│   │   │   └── audit.interceptor.ts
│   │   └── middleware/
│   │       └── clerk-auth.middleware.ts
│   └── health.controller.ts
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── docker-compose.yml
├── Dockerfile
├── package.json
└── tsconfig.json
```

---

## 2. Expanded Database Schema

File: `prisma/schema.prisma` (additions to existing)

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model Organization {
  id              String    @id @default(cuid())
  clerkOrgId      String    @unique  // Link to Clerk org
  
  name            String
  displayName     String?
  email           String?
  logo            String?   // URL
  
  subscription    String    @default("free")  // free, starter, pro, enterprise
  maxUsers        Int       @default(10)
  
  users           User[]
  roles           Role[]
  invitations     Invitation[]
  
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  @@index([clerkOrgId])
}

model User {
  id              String    @id @default(cuid())
  clerkUserId     String    @unique  // Link to Clerk user
  organizationId  String
  organization    Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)
  
  email           String
  firstName       String?
  lastName        String?
  profileImage    String?   // URL
  
  roles           UserRole[]
  
  // Metadata from Clerk
  lastSignInAt    DateTime?
  
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  @@unique([organizationId, email])
  @@index([organizationId])
  @@index([clerkUserId])
}

model Role {
  id              String    @id @default(cuid())
  organizationId  String
  organization    Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)
  
  name            String    // 'admin', 'instructor', 'student'
  displayName     String
  description     String?
  
  permissions     String[]  // ['user:read', 'user:write', 'course:admin']
  
  users           UserRole[]
  
  @@unique([organizationId, name])
  @@index([organizationId])
}

model UserRole {
  id              String    @id @default(cuid())
  userId          String
  user            User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  roleId          String
  role            Role @relation(fields: [roleId], references: [id], onDelete: Cascade)
  
  assignedAt      DateTime  @default(now())
  
  @@unique([userId, roleId])
  @@index([userId])
}

model Invitation {
  id              String    @id @default(cuid())
  organizationId  String
  organization    Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)
  
  email           String
  invitedAt       DateTime  @default(now())
  expiresAt       DateTime  // 1 week from created
  
  acceptedAt      DateTime?
  
  roleId          String    // Role to assign upon acceptance
  
  token           String    @unique  // For email link
  
  @@index([organizationId])
  @@index([email])
  @@index([expiresAt])
}

model AuditLog {
  id              String    @id @default(cuid())
  organizationId  String
  userId          String?
  
  action          String    // 'user_created', 'role_assigned', etc.
  resourceType    String    // 'user', 'role', 'organization'
  resourceId      String
  
  changes         Json      // { before: {}, after: {} }
  ipAddress       String?
  userAgent       String?
  
  createdAt       DateTime  @default(now())
  
  @@index([organizationId])
  @@index([userId])
  @@index([action])
  @@index([createdAt])
}

model Session {
  id              String    @id @default(cuid())
  userId          String
  clerkSessionId  String    @unique
  
  ipAddress       String?
  userAgent       String?
  
  createdAt       DateTime  @default(now())
  expiresAt       DateTime
  
  @@index([userId])
  @@index([expiresAt])
}
```

---

## 3. Enhanced User Service

File: `src/user/user.service.ts` (expansion)

```typescript
import { Injectable, Logger, Conflict } from '@nestjs/common';
import { PrismaService } from '../database/prisma.service';
import { ClerkService } from '../clerk/clerk.service';
import { EventsService } from '../events/events.service';
import { AuditService } from '../audit/audit.service';
import { RedisService } from '../cache/redis.service';

@Injectable()
export class UserService {
  private readonly logger = new Logger(UserService.name);

  constructor(
    private prisma: PrismaService,
    private clerk: ClerkService,
    private events: EventsService,
    private audit: AuditService,
    private redis: RedisService,
  ) {}

  /**
   * Create user when Clerk webhook fires
   */
  async createUserFromClerk(clerkUserId: string, organizationId: string) {
    // Fetch from Clerk
    const clerkUser = await this.clerk.getUser(clerkUserId);

    if (!clerkUser) throw new Error(`User ${clerkUserId} not found in Clerk`);

    // Check if already exists
    const existing = await this.prisma.user.findUnique({
      where: { clerkUserId },
    });

    if (existing) {
      this.logger.log(`User ${clerkUserId} already exists`);
      return existing;
    }

    // Create in database
    const user = await this.prisma.user.create({
      data: {
        clerkUserId,
        organizationId,
        email: clerkUser.email_addresses?.[0]?.email_address || '',
        firstName: clerkUser.first_name,
        lastName: clerkUser.last_name,
        profileImage: clerkUser.profile_image_url,
      },
    });

    // Assign default role (student)
    const studentRole = await this.prisma.role.findFirst({
      where: { organizationId, name: 'student' },
    });

    if (studentRole) {
      await this.prisma.userRole.create({
        data: {
          userId: user.id,
          roleId: studentRole.id,
        },
      });
    }

    // Audit log
    await this.audit.log(organizationId, null, 'user_created', 'user', user.id, {
      clerkUserId,
    });

    // Emit event
    await this.events.publish('user.created', {
      userId: user.id,
      organizationId,
      email: user.email,
    });

    this.logger.log(`User ${user.id} created from Clerk ${clerkUserId}`);
    return user;
  }

  /**
   * Get user with roles and permissions
   */
  async getUserWithPermissions(userId: string) {
    const user = await this.prisma.user.findUnique({
      where: { id: userId },
      include: {
        roles: {
          include: {
            role: {
              select: { name: true, displayName: true, permissions: true },
            },
          },
        },
        organization: true,
      },
    });

    if (!user) throw new Error(`User ${userId} not found`);

    // Flatten permissions
    const allPermissions = user.roles.flatMap((ur) => ur.role.permissions);
    const uniquePermissions = Array.from(new Set(allPermissions));

    return {
      ...user,
      permissions: uniquePermissions,
      roles: user.roles.map((ur) => ur.role),
    };
  }

  /**
   * Update user metadata from Clerk
   */
  async syncUserFromClerk(clerkUserId: string) {
    const clerkUser = await this.clerk.getUser(clerkUserId);
    if (!clerkUser) return;

    const user = await this.prisma.user.findUnique({
      where: { clerkUserId },
    });

    if (!user) return;

    const before = { ...user };

    const updated = await this.prisma.user.update({
      where: { id: user.id },
      data: {
        email: clerkUser.email_addresses?.[0]?.email_address || user.email,
        firstName: clerkUser.first_name || user.firstName,
        lastName: clerkUser.last_name || user.lastName,
        profileImage: clerkUser.profile_image_url || user.profileImage,
        lastSignInAt: new Date(),
      },
    });

    // Audit log
    await this.audit.log(user.organizationId, user.id, 'user_synced', 'user', user.id, {
      before,
      after: updated,
    });

    return updated;
  }

  /**
   * Delete user from all services
   */
  async deleteUser(userId: string) {
    const user = await this.prisma.user.findUnique({
      where: { id: userId },
    });

    if (!user) throw new Error(`User not found`);

    // Delete from DB (cascade deletes roles)
    await this.prisma.user.delete({
      where: { id: userId },
    });

    // Audit log
    await this.audit.log(user.organizationId, userId, 'user_deleted', 'user', userId, {
      email: user.email,
    });

    // Emit event (other services clean up)
    await this.events.publish('user.deleted', {
      userId,
      organizationId: user.organizationId,
    });

    // Invalidate JWT cache
    await this.redis.del(`jwt:${userId}:*`);

    this.logger.log(`User ${userId} deleted`);
  }

  /**
   * Get users in organization
   */
  async getOrgUsers(organizationId: string) {
    return this.prisma.user.findMany({
      where: { organizationId },
      include: {
        roles: { include: { role: { select: { name: true } } } },
      },
    });
  }
}
```

---

## 4. RBAC Service

File: `src/rbac/rbac.service.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../database/prisma.service';

@Injectable()
export class RbacService {
  constructor(private prisma: PrismaService) {}

  /**
   * Check if user has permission
   */
  async hasPermission(userId: string, permission: string): Promise<boolean> {
    const user = await this.prisma.user.findUnique({
      where: { id: userId },
      include: {
        roles: {
          include: { role: { select: { permissions: true } } },
        },
      },
    });

    if (!user) return false;

    const allPermissions = user.roles.flatMap((ur) => ur.role.permissions);
    return allPermissions.includes(permission);
  }

  /**
   * Check if user has role
   */
  async hasRole(userId: string, roleName: string): Promise<boolean> {
    const userRole = await this.prisma.userRole.findFirst({
      where: {
        user: { id: userId },
        role: { name: roleName },
      },
    });

    return !!userRole;
  }

  /**
   * Assign role to user
   */
  async assignRole(userId: string, roleId: string) {
    return this.prisma.userRole.create({
      data: { userId, roleId },
    });
  }

  /**
   * Remove role from user
   */
  async removeRole(userId: string, roleId: string) {
    return this.prisma.userRole.deleteMany({
      where: { userId, roleId },
    });
  }

  /**
   * Create role with permissions
   */
  async createRole(organizationId: string, data: {
    name: string;
    displayName: string;
    permissions: string[];
  }) {
    return this.prisma.role.create({
      data: {
        organizationId,
        ...data,
      },
    });
  }
}
```

---

## 5. Invitation System

File: `src/invitation/invitation.service.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '../database/prisma.service';
import { EventsService } from '../events/events.service';
import { randomBytes } from 'crypto';

@Injectable()
export class InvitationService {
  private readonly logger = new Logger(InvitationService.name);

  constructor(
    private prisma: PrismaService,
    private events: EventsService,
  ) {}

  /**
   * Create invitation to join organization
   */
  async sendInvitation(organizationId: string, email: string, roleId: string) {
    // Check if already invited/joined
    const existingUser = await this.prisma.user.findFirst({
      where: { organizationId, email },
    });

    if (existingUser) {
      throw new Error('User already in organization');
    }

    const token = randomBytes(32).toString('hex');
    const expiresAt = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000); // 1 week

    const invitation = await this.prisma.invitation.create({
      data: {
        organizationId,
        email,
        roleId,
        token,
        expiresAt,
      },
    });

    // Emit event (notification service sends email)
    await this.events.publish('invitation.sent', {
      organizationId,
      email,
      invitationId: invitation.id,
      token,
    });

    this.logger.log(`Invitation sent to ${email} for org ${organizationId}`);
    return invitation;
  }

  /**
   * Accept invitation
   */
  async acceptInvitation(token: string, clerkUserId: string) {
    const invitation = await this.prisma.invitation.findUnique({
      where: { token },
    });

    if (!invitation) throw new Error('Invalid invitation');
    if (invitation.expiresAt < new Date()) throw new Error('Invitation expired');
    if (invitation.acceptedAt) throw new Error('Invitation already accepted');

    // Update user's organization
    const user = await this.prisma.user.findUnique({
      where: { clerkUserId },
    });

    if (!user) throw new Error('User not found');

    // Assign role
    await this.prisma.userRole.create({
      data: {
        userId: user.id,
        roleId: invitation.roleId,
      },
    });

    // Mark invitation accepted
    await this.prisma.invitation.update({
      where: { id: invitation.id },
      data: { acceptedAt: new Date() },
    });

    // Emit event
    await this.events.publish('invitation.accepted', {
      userId: user.id,
      organizationId: invitation.organizationId,
    });

    return { success: true };
  }
}
```

---

## 6. API Endpoints

File: `src/auth/auth.controller.ts` (additions)

```typescript
import {
  Controller,
  Post,
  Get,
  Body,
  UseGuards,
  Logger,
} from '@nestjs/common';
import { UserService } from '../user/user.service';
import { JwtGuard } from '../common/guards/jwt.guard';
import { CurrentUser } from '../common/decorators/current-user.decorator';
import { AuthService } from './auth.service';

@Controller('api/auth')
export class AuthController {
  private readonly logger = new Logger(AuthController.name);

  constructor(
    private auth: AuthService,
    private user: UserService,
  ) {}

  /**
   * POST /api/auth/callback
   * Handle Clerk webhook (user created, updated, deleted)
   */
  @Post('callback')
  async handleClerkWebhook(@Body() payload: any) {
    const { type, data } = payload;

    switch (type) {
      case 'user.created':
        await this.user.createUserFromClerk(
          data.id,
          data.org_id || 'default-org',
        );
        break;
      case 'user.updated':
        await this.user.syncUserFromClerk(data.id);
        break;
      case 'user.deleted':
        await this.user.deleteUser(data.id);
        break;
    }

    return { success: true };
  }

  /**
   * GET /api/auth/me
   * Get current user with permissions
   */
  @Get('me')
  @UseGuards(JwtGuard)
  async getCurrentUser(@CurrentUser() userId: string) {
    return this.user.getUserWithPermissions(userId);
  }

  /**
   * POST /api/auth/refresh
   * Refresh JWT token
   */
  @Post('refresh')
  @UseGuards(JwtGuard)
  async refreshToken(@CurrentUser() userId: string) {
    return this.auth.generateToken(userId);
  }
}
```

File: `src/invitation/invitation.controller.ts`

```typescript
import {
  Controller,
  Post,
  Get,
  Body,
  Param,
  UseGuards,
} from '@nestjs/common';
import { InvitationService } from './invitation.service';
import { JwtGuard } from '../common/guards/jwt.guard';
import { RbacGuard } from '../common/guards/rbac.guard';
import { CurrentUser } from '../common/decorators/current-user.decorator';

@Controller('api/invitations')
@UseGuards(JwtGuard)
export class InvitationController {
  constructor(private invitations: InvitationService) {}

  /**
   * POST /api/invitations
   * Send invitation (org admin only)
   */
  @Post()
  @UseGuards(RbacGuard('organization:invite'))
  async sendInvitation(
    @Body() dto: { email: string; roleId: string },
    @CurrentUser('orgId') organizationId: string,
  ) {
    return this.invitations.sendInvitation(organizationId, dto.email, dto.roleId);
  }

  /**
   * POST /api/invitations/:token/accept
   * Accept invitation (public endpoint)
   */
  @Post(':token/accept')
  async acceptInvitation(
    @Param('token') token: string,
    @Body() dto: { clerkUserId: string },
  ) {
    return this.invitations.acceptInvitation(token, dto.clerkUserId);
  }
}
```

---

## 7. Clerk Webhook Integration

File: `src/clerk/clerk.controller.ts`

```typescript
import {
  Controller,
  Post,
  Body,
  Headers,
  Logger,
} from '@nestjs/common';
import { ClerkService } from './clerk.service';
import { ClerkWebhookGuard } from './clerk-webhook.guard';
import { UseGuards } from '@nestjs/common';

@Controller('api/webhooks')
export class ClerkController {
  private readonly logger = new Logger(ClerkController.name);

  constructor(private clerk: ClerkService) {}

  /**
   * POST /api/webhooks/clerk
   * Receive Clerk events (user.created, user.updated, user.deleted, org.created, etc.)
   */
  @Post('clerk')
  @UseGuards(ClerkWebhookGuard)
  async handleClerkWebhook(@Body() payload: any) {
    const { type, data } = payload;

    this.logger.log(`Received Clerk webhook: ${type}`);

    // Route to appropriate handler
    // Handlers: UserService, OrganizationService, etc.

    return { received: true };
  }
}
```

File: `src/clerk/clerk-webhook.guard.ts`

```typescript
import { Injectable, CanActivate, ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { Request } from 'express';
import { verify } from 'svix';

@Injectable()
export class ClerkWebhookGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest<Request>();

    const payload = JSON.stringify(request.body);
    const headers = request.headers;

    try {
      const msg = verify(payload, headers as any, process.env.CLERK_WEBHOOK_SECRET);
      (request as any).webhookData = msg;
      return true;
    } catch (error) {
      throw new UnauthorizedException('Invalid webhook signature');
    }
  }
}
```

---

## 8. Audit Service

File: `src/audit/audit.service.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../database/prisma.service';
import { Request } from 'express';

@Injectable()
export class AuditService {
  constructor(private prisma: PrismaService) {}

  async log(
    organizationId: string,
    userId: string | null,
    action: string,
    resourceType: string,
    resourceId: string,
    changes?: any,
    request?: Request,
  ) {
    return this.prisma.auditLog.create({
      data: {
        organizationId,
        userId,
        action,
        resourceType,
        resourceId,
        changes: changes || {},
        ipAddress: request?.ip,
        userAgent: request?.get('user-agent'),
      },
    });
  }

  async getAuditLog(organizationId: string, limit = 100) {
    return this.prisma.auditLog.findMany({
      where: { organizationId },
      orderBy: { createdAt: 'desc' },
      take: limit,
    });
  }
}
```

---

## 9. Environment Configuration

File: `.env`

```bash
NODE_ENV=production
PORT=3008

# Database
DATABASE_URL=postgresql://auth:password@localhost:5432/auth

# Clerk
CLERK_API_KEY=sk_live_...
CLERK_FRONTEND_API=pk_live_...
CLERK_WEBHOOK_SECRET=whsec_...

# JWT
JWT_SECRET=your-jwt-secret-key
JWT_EXPIRATION=24h

# Redis (for JWT invalidation)
REDIS_URL=redis://localhost:6379

# Event Bus
NATS_URL=nats://localhost:4222

# Service URLs
NOTIFICATION_SERVICE_URL=http://localhost:3001

# Logging
LOG_LEVEL=info
```

---

## 10. Docker Setup

File: `docker-compose.yml`

```yaml
version: '3.8'

services:
  auth-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: auth
      POSTGRES_PASSWORD: password
      POSTGRES_DB: auth
    ports:
      - "5432:5432"
    volumes:
      - auth-db-data:/var/lib/postgresql/data

  auth-service:
    build: .
    ports:
      - "3008:3008"
    environment:
      DATABASE_URL: postgresql://auth:password@auth-db:5432/auth
      CLERK_API_KEY: ${CLERK_API_KEY}
      CLERK_WEBHOOK_SECRET: ${CLERK_WEBHOOK_SECRET}
      JWT_SECRET: ${JWT_SECRET}
      NATS_URL: nats://nats:4222
      REDIS_URL: redis://redis:6379
    depends_on:
      - auth-db

volumes:
  auth-db-data:
```

---

## 11. Deployment Checklist

- [ ] PostgreSQL configured
- [ ] Clerk organization created
- [ ] Webhook endpoint configured in Clerk
- [ ] JWT secret configured
- [ ] Default roles created (admin, instructor, student)
- [ ] Audit logging tested
- [ ] User creation flow tested (Clerk → DB)
- [ ] Invitation system tested
- [ ] RBAC permissions enforced
- [ ] Canary traffic routed
- [ ] All services updated to use identity service

---

## 12. Integration Points

Each service needs JWT validation:

```typescript
// Example: In any other service controller
@UseGuards(JwtGuard)
@Get('some-protected-endpoint')
async someMethod(@CurrentUser() userId: string) {
  // JWT is validated via guard
  // User ID extracted from token
}
```

---

## 13. Success Criteria

✅ User creation <500ms (Clerk → DB → Event)  
✅ JWT token generated in <100ms  
✅ Permission check <10ms (cached)  
✅ Audit log recorded synchronously  
✅ 100% data consistency with Clerk  
✅ Zero authentication failures in critical path  

---

## 14. Phase 7 Finalizes Migration

Phase 7 completes the migration:

| Phase | Service | Status |
|-------|---------|--------|
| 0 | API Gateway + BFF | ✅ |
| 1 | Notification | ✅ |
| 2 | Gamification | ✅ |
| 3 | AI Tutor | ✅ |
| 4 | Analytics | ✅ |
| 5 | Assessment | ✅ |
| 6 | Course + Enrollment | ✅ |
| 7 | Identity & Access | ✅ |

**All monolith functionality extracted. Migration complete in 34 weeks (8 months).**

## Key Differences: Monolith → Microservices

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| Auth | Clerk in Supabase RPC | NestJS auth-service + JWT |
| User Sync | Manual | Clerk webhooks + DB |
| Permissions | RLS in Supabase | RBAC in identity service |
| Audit | None | Full audit logs |
| Invitations | Edge function | Identity service API |
| Multi-tenancy | Implicit (org_id) | Explicit (Organization model) |
| Scalability | Supabase limits | Unlimited (per DB) |

---

## 15. Cutover Strategy

Week 31: Deploy identity service, run in shadow mode (logs events, doesn't affect auth)  
Week 32: Enable for 10% of users (new signups only)  
Week 33: Enable for 50% (existing users opt-in)  
Week 34: 100% cutover (all users, disables old RPC)  

**Rollback:** Revert users' `clerkUserId` back to null, stop using identity service JWT

---

## 16. Running Locally

```bash
docker-compose up -d

# Install deps
npm install

# Setup env
cp .env.example .env
# Add CLERK_API_KEY, CLERK_WEBHOOK_SECRET

# Run migrations
npx prisma migrate dev

# Start
npm run start:dev

# Test user creation
curl -X POST http://localhost:3008/api/auth/callback \
  -H "Content-Type: application/json" \
  -d '{"type":"user.created","data":{"id":"user_123","first_name":"John"}}'
```

**Phase 7 is the final piece. With all 7 phases deployed, the monolith is fully strangled and can be decommissioned.**
