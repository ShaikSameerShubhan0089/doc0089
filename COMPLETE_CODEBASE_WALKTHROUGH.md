# Complete Codebase Walkthrough

## Table of Contents
1. [Lumina Auth Service](#lumina-auth-service)
2. [Lumina Learning Hub](#lumina-learning-hub)
3. [Integration Architecture](#integration-architecture)
4. [Migration Plan Alignment](#migration-plan-alignment)

---

## LUMINA AUTH SERVICE

**Purpose**: Production-ready microservice handling identity, access control, RBAC, and user management. Serves as the Phase 7 implementation of the strangler fig migration.

**Technology Stack**:
- Framework: NestJS 10.4 + Fastify
- Database: PostgreSQL (Prisma ORM v5)
- Authentication: Clerk (@clerk/backend)
- Webhooks: Svix (for Clerk events)
- Caching: Redis (ioredis)
- Language: TypeScript
- Testing: Jest + NestJS testing utilities
- Containerization: Docker (Node 18-alpine)

### Root Configuration Files

**`package.json`**
- Service metadata (name: "lumina-auth-service", version: "0.1.0", unlicensed - pre-production)
- 30 npm scripts for development, testing, deployment
- Key scripts:
  - `npm start` → Node production start
  - `npm run dev` → NestJS watch mode
  - `npm test` → Jest unit tests
  - `npm run test:e2e` → End-to-end tests
  - `npm run prisma:migrate` → Apply schema changes
  - `npm run prisma:seed` → Populate initial data
  - `npm run prisma:studio` → GUI for database exploration
- Dependencies: NestJS, Clerk, Prisma, Redis, Svix, Pino logger, decorators library
- DevDependencies: TypeScript, Jest, @types/node, ESLint, Prettier

**`tsconfig.json`**
- TypeScript compiler configuration
- Key settings:
  - `target: ES2021` → Modern JavaScript output
  - `module: commonjs` → Node.js compatible
  - `strict: true` → Full type checking enabled
  - `esModuleInterop: true` → CommonJS/ESM interop
  - `experimentalDecorators: true` → NestJS decorators support
  - `emitDecoratorMetadata: true` → Reflection metadata for DI

**`nest-cli.json`**
- NestJS CLI configuration
- Specifies source root (`src/`) and build output (`dist/`)
- Project type: standard (not monorepo)

**`jest.config.ts`**
- Test framework configuration
- Settings: module preset (ts-jest), test environment (node), coverage thresholds
- Include patterns for unit tests (.spec.ts) and e2e tests (.e2e.ts)

**`.prettierrc`**
- Code formatting rules (2 spaces, single quotes, trailing commas)

**`.gitignore`**
- Excludes: node_modules/, dist/, .env, .env.*.local, logs/, coverage/

**`Dockerfile`**
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY dist/ ./dist/
EXPOSE 3008
CMD ["node", "dist/main.js"]
```
- Multi-stage build (omitted from example)
- Production image with minimal footprint
- Listens on port 3008

**`docker-compose.yml`**
- Local development environment:
  - PostgreSQL 15 on 5432
  - Redis 7 on 6379
  - Auth service container on 3008
- Mounts for database persistence
- Environment variables passed through from `.env`

**`.env.example`**
```env
NODE_ENV=development
PORT=3008
DATABASE_URL=postgresql://user:password@localhost:5432/lumina_auth
REDIS_URL=redis://localhost:6379
CLERK_SECRET_KEY=sk_test_...
CLERK_WEBHOOK_SECRET=whsec_...
INTERNAL_API_KEY=your_secret_api_key
PUBLIC_APP_URL=http://localhost:5173
CORS_ORIGINS=http://localhost:5173
```
- Template for required environment variables
- Database and Clerk credentials
- Service-to-service API key for internal requests

**`README.md`**
- Installation & setup instructions
- Development workflow (npm run dev)
- Database setup (prisma migrate dev)
- API documentation
- Architecture overview

**`railway.json`**
- Railway.app deployment configuration
- Specifies: build command, start command, environment variables
- Enables one-click deployment to Railway cloud

---

### Source Code: Core Bootstrapping

**`src/main.ts`**
```typescript
async function bootstrap() {
  // Create NestJS application with Fastify adapter
  const app = await NestFactory.create(AppModule, new FastifyAdapter());
  
  // Enable CORS from configured origins
  app.enableCors({ origin: process.env.CORS_ORIGINS!.split(',') });
  
  // Set global API prefix
  app.setGlobalPrefix('api');
  
  // Start listening
  const port = process.env.PORT || 3001;
  await app.listen(port, '0.0.0.0');
  
  logger.log(`Auth service running on port ${port}`);
}
```
- Bootstrap function that:
  - Creates NestJS app with Fastify (faster than Express)
  - Enables CORS for frontend communication
  - Sets global /api prefix for all routes
  - Attaches global exception filters and interceptors
- Port configurable via environment (default 3001 or 3008 from config)

**`src/app.module.ts`**
```typescript
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    DatabaseModule,
    RedisModule,
    ClerkModule,
    InvitationModule,
    RoleModule,
    WebhookModule,
    UserModule,
    EventsModule,
  ],
  controllers: [HealthController],
})
export class AppModule {}
```
- Root module composition:
  - **ConfigModule**: Global configuration (env vars, validation)
  - **DatabaseModule**: Prisma client provider
  - **RedisModule**: Cache client provider
  - **ClerkModule**: Clerk API integration
  - **InvitationModule**: Invitation flow (create, accept)
  - **RoleModule**: RBAC management
  - **WebhookModule**: Clerk webhook receiver
  - **UserModule**: User CRUD + role assignment
  - **EventsModule**: Event bus abstraction
- Single controller at root: HealthController (for k8s liveness probes)
- All modules are feature modules with their own providers/controllers

**`src/health.controller.ts`**
```typescript
@Controller('health')
export class HealthController {
  constructor(private prisma: PrismaService, private redis: RedisService) {}
  
  @Get()
  health() {
    return { status: 'ok', timestamp: new Date() };
  }
  
  @Get('ready')
  async ready() {
    // Check DB and Redis connectivity
    await this.prisma.$queryRaw`SELECT 1`;
    await this.redis.ping();
    return { status: 'ready' };
  }
}
```
- Liveness probe: `/api/health` → quick response (always 200)
- Readiness probe: `/api/health/ready` → validates DB/Redis (only 200 if healthy)
- Used by Kubernetes for pod lifecycle management

---

### Configuration Layer

**`src/config/configuration.ts`**
```typescript
import { z } from 'zod';

export const appConfig = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  PORT: z.coerce.number().default(3001),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  CLERK_SECRET_KEY: z.string().min(1),
  CLERK_WEBHOOK_SECRET: z.string().min(1),
  INTERNAL_API_KEY: z.string().min(1),
  PUBLIC_APP_URL: z.string().url(),
  CORS_ORIGINS: z.string().optional(),
});

export default () => {
  // Parse and validate environment variables at startup
  // Throws error if validation fails (fail-fast approach)
  return appConfig.parse(process.env);
};
```
- Uses Zod for type-safe configuration validation
- Fails fast at startup if env vars are missing or invalid
- Provides TypeScript types for config references
- Separate from NestJS ConfigService (raw parsing, then injected)

**`src/config/config.types.ts`**
```typescript
export type AppRole = 'platform_owner' | 'tenant_admin' | 'instructor' | 'student';

export const ROLE_HIERARCHY: Record<AppRole, number> = {
  'platform_owner': 1,    // Highest privilege
  'tenant_admin': 2,
  'instructor': 3,
  'student': 4,           // Lowest privilege
};

export interface GatewayUser {
  userId: string;         // Clerk user ID
  orgId: string | null;   // Primary org (can have roles in multiple orgs)
  roles: AppRole[];       // All roles user has
  email: string;
}
```
- Type definitions for RBAC system
- Role hierarchy for permission checks (1 = highest)
- GatewayUser interface: passed from gateway to service in JWT claims
- Used throughout service for role-based access control

---

### Database Layer

**`src/database/prisma.service.ts`**
```typescript
@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.$connect();
    console.log('Database connected');
  }

  async onModuleDestroy() {
    await this.$disconnect();
    console.log('Database disconnected');
  }
}
```
- Wraps Prisma Client as a NestJS service
- Implements NestJS lifecycle hooks:
  - `onModuleInit`: Connect to DB on app start
  - `onModuleDestroy`: Gracefully disconnect on shutdown
- Single instance (singleton) used throughout app
- Provides dependency injection for all services

**`src/database/prisma.module.ts`**
```typescript
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class DatabaseModule {}
```
- Exports PrismaService as a provider
- Other modules can import DatabaseModule to get Prisma access

**`prisma/schema.prisma`**
```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

// Core user model linked to Clerk
model User {
  id                      String    @id @default(cuid())
  clerkUserId             String    @unique           // External Clerk ID
  email                   String    @unique
  isActive                Boolean   @default(true)
  primaryOrganizationId   String?
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt
  
  // Relations
  roles                   UserRole[]
  invitations             Invitation[]
  auditLogs               AuthAuditLog[]
}

model UserRole {
  id                String   @id @default(cuid())
  userId            String
  role              AppRole
  organizationId    String?                // Multi-org support
  grantedAt         DateTime @default(now())
  grantedBy         String?  
  
  user              User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@unique([userId, role, organizationId])
  @@index([organizationId])
}

model Invitation {
  id                String   @id @default(cuid())
  email             String   @db.VarChar(255)
  organizationId    String
  role              AppRole
  status            InvitationStatus @default(pending)
  tokenHash         String   @unique
  metadata          Json?
  expiresAt         DateTime
  acceptedAt        DateTime?
  createdAt         DateTime @default(now())
  
  @@unique([email, organizationId])   // One pending invite per email per org
  @@index([tokenHash])
}

model AuthAuditLog {
  id                String   @id @default(cuid())
  userId            String
  action            String   // 'user.created', 'role.assigned', 'invitation.accepted'
  resource          String   // 'user', 'role', 'invitation'
  resourceId        String   // ID of affected resource
  changes           Json?    // Before/after values
  ipAddress         String?
  userAgent         String?
  createdAt         DateTime @default(now())
  
  user              User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@index([userId, createdAt])
}

enum AppRole {
  platform_owner
  tenant_admin
  instructor
  student
}

enum InvitationStatus {
  pending
  accepted
  expired
  revoked
}
```
- **User Model**: Clerk-native design with `clerkUserId` as unique identifier
  - `primaryOrganizationId`: Default org for multi-tenant scenarios
  - Relations to roles, invitations, audit logs (audit trail)
- **UserRole Model**: Many-to-many junction linking users to roles
  - `organizationId`: Allows same user to have different roles in different orgs
  - Unique constraint prevents duplicate role assignments
  - `grantedBy`: Audit trail for who assigned the role
- **Invitation Model**: Email-based invitations with token validation
  - `tokenHash`: Secure token (SHA256 of random bytes)
  - Unique composite on (email, organizationId) prevents duplicate invites
  - Tracks status and acceptance timestamp
- **AuthAuditLog Model**: Compliance and security logging
  - Records all mutations (user creation, role assignment, etc.)
  - Stores IP and User-Agent for forensics
  - Composite index on (userId, createdAt) for audit trails
- **Enums**: AppRole (4 levels) and InvitationStatus (4 states)

**`prisma/seed.ts`**
```typescript
async function main() {
  const user = await prisma.user.upsert({
    where: { email: 'admin@example.com' },
    update: {},
    create: {
      email: 'admin@example.com',
      clerkUserId: 'user_test_123',
      isActive: true,
      roles: {
        create: {
          role: 'platform_owner',
        },
      },
    },
  });
  console.log('Seeded:', user);
}

main()
  .then(() => prisma.$disconnect())
  .catch((e) => {
    console.error(e);
    process.exit(1);
  });
```
- Initializes database with seed data
- Creates test admin user (email, Clerk ID, roles)
- Run with: `npm run prisma:seed`

---

### Clerk Integration

**`src/clerk/clerk.service.ts`**
```typescript
@Injectable()
export class ClerkService {
  private readonly clerk;
  private readonly logger = new Logger(ClerkService.name);

  constructor(private readonly configService: ConfigService) {
    this.clerk = createClerkClient({
      secretKey: this.configService.getOrThrow<string>('CLERK_SECRET_KEY'),
    });
  }

  // Create user in Clerk (external identity provider)
  async createUser(params: {
    email: string;
    password: string;
    firstName?: string;
    lastName?: string;
  }) {
    return this.clerk.users.createUser({
      emailAddress: [params.email],
      password: params.password,
      firstName: params.firstName,
      lastName: params.lastName,
    });
  }

  // Fetch user from Clerk
  async getUser(clerkUserId: string) {
    return this.clerk.users.getUser(clerkUserId);
  }

  // Delete user from Clerk
  async deleteUser(clerkUserId: string) {
    return this.clerk.users.deleteUser(clerkUserId);
  }

  // Sync roles to Clerk metadata (for token claims)
  async syncRolesToMetadata(
    clerkUserId: string,
    roles: Array<{ role: AppRole; organizationId: string | null }>,
    primaryOrganizationId: string | null,
  ) {
    const primaryRole = this.getPrimaryRole(roles);

    try {
      await this.clerk.users.updateUserMetadata(clerkUserId, {
        publicMetadata: {
          org_id: primaryOrganizationId,
          roles: roles.map((r) => r.role),
          primary_role: primaryRole,
        },
      });
      this.logger.log(
        `Synced roles to Clerk for user ${clerkUserId}`
      );
    } catch (err) {
      this.logger.error(
        `Failed to sync roles to Clerk: ${(err as Error).message}`
      );
      throw err;
    }
  }
}
```
- Wrapper around Clerk Backend SDK
- Handles:
  - User creation in Clerk (upstream auth provider)
  - User fetching for validation
  - User deletion on account closure
  - Role synchronization to Clerk publicMetadata (included in JWT)
- Logging for debugging and auditing

**`src/clerk/clerk.module.ts`**
```typescript
@Module({
  providers: [ClerkService],
  exports: [ClerkService],
})
export class ClerkModule {}
```
- Exports ClerkService for use by UserService and WebhookService

---

### Webhook Handler

**`src/webhook/webhook.controller.ts`**
```typescript
@Controller('webhook')
export class WebhookController {
  constructor(
    private readonly webhookService: WebhookService,
  ) {}

  @Post('clerk')
  async handleClerkWebhook(
    @Body() event: any,
    @Headers('svix-id') svixId: string,
    @Headers('svix-timestamp') timestamp: string,
    @Headers('svix-signature') signature: string,
  ) {
    // Verify Svix signature (webhook authenticity)
    const isValid = await this.webhookService.verifyWebhook({
      svixId,
      timestamp,
      signature,
      body: JSON.stringify(event),
    });

    if (!isValid) {
      throw new UnauthorizedException('Invalid webhook signature');
    }

    // Process event
    await this.webhookService.processClerkEvent(event);
    return { success: true };
  }
}
```
- Endpoint: `POST /api/webhook/clerk`
- Validates Svix signatures (proves message from Clerk)
- Processes Clerk events asynchronously

**`src/webhook/webhook.service.ts`**
```typescript
@Injectable()
export class WebhookService {
  constructor(
    private readonly userService: UserService,
    private readonly eventsService: EventsService,
  ) {}

  // Process Clerk webhook events
  async processClerkEvent(event: any) {
    const { type, data } = event;

    if (type === 'user.created') {
      // Clerk user created → create local user record + default role
      await this.userService.createLocalUser({
        clerkUserId: data.id,
        email: data.email_addresses[0]?.email_address,
        firstName: data.first_name,
        lastName: data.last_name,
      });
    }

    if (type === 'user.updated') {
      // Update email/name in local DB if changed
      await this.userService.updateLocalUser(data.id, {
        email: data.email_addresses[0]?.email_address,
      });
    }

    if (type === 'user.deleted') {
      // Cascade delete: user + roles + invitations
      await this.userService.deleteLocalUser(data.id);
    }

    // Emit event to event bus (for other services to react)
    await this.eventsService.emit({
      type,
      payload: data,
      timestamp: new Date(),
    });
  }
}
```
- Processes Clerk webhook events:
  - **user.created**: Creates local user record + assigns default role
  - **user.updated**: Syncs email/name changes to local DB
  - **user.deleted**: Cascade deletes user + related records
- Emits events to event bus (for integration with other services)

**`src/webhook/webhook.module.ts`**
```typescript
@Module({
  controllers: [WebhookController],
  providers: [WebhookService],
  imports: [
    UserModule, // Inject UserService
    EventsModule, // Inject EventsService
  ],
})
export class WebhookModule {}
```

---

### User Management

**`src/user/user.service.ts`**
```typescript
@Injectable()
export class UserService {
  constructor(private readonly prisma: PrismaService) {}

  // Get current user profile
  async getMe(caller: GatewayUser) {
    const user = await this.prisma.user.findUnique({
      where: { clerkUserId: caller.userId },
      include: { roles: true },
    });

    if (!user) throw new NotFoundException('User not found');

    const primaryRole = this.getPrimaryRole(
      user.roles.map((r) => r.role as AppRole),
    );

    return {
      id: user.id,
      clerkUserId: user.clerkUserId,
      email: user.email,
      isActive: user.isActive,
      primaryOrganizationId: user.primaryOrganizationId,
      roles: user.roles.map((r) => ({
        id: r.id,
        role: r.role,
        organizationId: r.organizationId,
        grantedAt: r.grantedAt,
      })),
      primaryRole,
    };
  }

  // List users (with org/role filtering)
  async listUsers(params: {
    organizationId?: string;
    role?: string;
    page?: number;
    limit?: number;
    caller: GatewayUser;
  }) {
    const { organizationId, role, page = 1, limit = 50, caller } = params;

    // RBAC: Tenant admins can only see their own org
    const orgFilter = caller.roles.includes('platform_owner')
      ? organizationId
      : caller.orgId;

    const where = {
      isActive: true,
      ...(orgFilter && {
        roles: {
          some: {
            organizationId: orgFilter,
            ...(role && { role: role as any }),
          },
        },
      }),
    };

    const [data, total] = await Promise.all([
      this.prisma.user.findMany({
        where,
        include: { roles: true },
        skip: (page - 1) * limit,
        take: limit,
        orderBy: { createdAt: 'desc' },
      }),
      this.prisma.user.count({ where }),
    ]);

    return {
      data: data.map((u) => this.formatUser(u)),
      pagination: {
        page,
        limit,
        total,
        pages: Math.ceil(total / limit),
      },
    };
  }

  // Create user from invitation acceptance
  async createLocalUser(clerkUser: any) {
    return this.prisma.user.create({
      data: {
        clerkUserId: clerkUser.id,
        email: clerkUser.email,
        isActive: true,
        roles: {
          create: {
            role: 'student', // Default role
          },
        },
      },
      include: { roles: true },
    });
  }

  // Assign role to user
  async assignRole(
    userId: string,
    role: AppRole,
    organizationId: string | null,
    caller: GatewayUser,
  ) {
    // Enforce role hierarchy (can't assign higher privilege than own role)
    this.enforceRoleHierarchy(caller, role, organizationId);

    return this.prisma.userRole.upsert({
      where: {
        userId_role_organizationId: {
          userId,
          role,
          organizationId,
        },
      },
      update: { grantedBy: caller.userId },
      create: {
        userId,
        role,
        organizationId,
        grantedBy: caller.userId,
      },
    });
  }

  // Remove role from user
  async removeRole(
    userId: string,
    role: AppRole,
    organizationId: string | null,
  ) {
    const deleted = await this.prisma.userRole.deleteMany({
      where: {
        userId,
        role,
        organizationId,
      },
    });

    return { deletedCount: deleted.count };
  }

  // Helper: Determine primary role (highest privilege)
  private getPrimaryRole(roles: AppRole[]): AppRole {
    return roles.reduce((highest, current) =>
      ROLE_HIERARCHY[current] < ROLE_HIERARCHY[highest] ? current : highest,
    );
  }

  // Helper: Enforce role hierarchy (prevent privilege escalation)
  private enforceRoleHierarchy(caller: GatewayUser, targetRole: AppRole, orgId: string | null) {
    if (caller.roles.includes('platform_owner')) return; // Platform owners can do anything
    if (caller.roles.includes('tenant_admin') && targetRole !== 'student') {
      throw new ForbiddenException('Cannot assign role above student');
    }
    throw new ForbiddenException('Insufficient permissions');
  }
}
```
- Core user management:
  - **getMe()**: Fetch current user profile with roles
  - **listUsers()**: Pagination + RBAC filtering (tenant admins only see their org)
  - **createLocalUser()**: Create when Clerk webhook fires
  - **assignRole()**: Grant role + hierarchy enforcement
  - **removeRole()**: Revoke role
  - **getPrimaryRole()**: Helper to find highest privilege role
  - **enforceRoleHierarchy()**: Prevent privilege escalation (can't assign roles higher than your own)

**`src/user/user.controller.ts`**
```typescript
@Controller('users')
@UseGuards(JwtGuard, RolesGuard)
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get('me')
  async getMe(@CurrentUser() user: GatewayUser) {
    return this.userService.getMe(user);
  }

  @Get()
  @Roles('platform_owner', 'tenant_admin')
  async listUsers(
    @Query('organizationId') organizationId: string,
    @Query('role') role: string,
    @Query('page') page: number,
    @CurrentUser() caller: GatewayUser,
  ) {
    return this.userService.listUsers({ organizationId, role, page, caller });
  }

  @Patch(':userId')
  @Roles('platform_owner', 'tenant_admin')
  async assignRole(
    @Param('userId') userId: string,
    @Body() body: { role: AppRole; organizationId?: string },
    @CurrentUser() caller: GatewayUser,
  ) {
    return this.userService.assignRole(userId, body.role, body.organizationId, caller);
  }

  @Delete(':userId/roles/:role')
  @Roles('platform_owner', 'tenant_admin')
  async removeRole(
    @Param('userId') userId: string,
    @Param('role') role: AppRole,
  ) {
    return this.userService.removeRole(userId, role, null);
  }
}
```
- Routes:
  - `GET /api/users/me` → Fetch authenticated user profile
  - `GET /api/users` → List users (admin/owner only)
  - `PATCH /api/users/:userId` → Assign role
  - `DELETE /api/users/:userId/roles/:role` → Remove role
- Guards: JwtGuard (verify token) + RolesGuard (check permissions)
- Decorators:
  - `@CurrentUser()` → Extract user from JWT claims
  - `@Roles()` → Require specific roles

**`src/user/user.module.ts`**
- Provides: UserService, UserController
- Imports: DatabaseModule (for Prisma), ClerkModule (for Clerk integration)

---

### Role-Based Access Control

**`src/role/role.service.ts`**
```typescript
@Injectable()
export class RoleService {
  constructor(private readonly prisma: PrismaService) {}

  // Get role with permissions
  async getRole(role: AppRole) {
    const permissions = ROLE_PERMISSIONS[role] || [];
    return {
      name: role,
      permissions,
      hierarchy: ROLE_HIERARCHY[role],
    };
  }

  // List all roles
  async listRoles() {
    return Object.keys(ROLE_HIERARCHY).map((role) => ({
      name: role,
      hierarchy: ROLE_HIERARCHY[role as AppRole],
      permissions: ROLE_PERMISSIONS[role as AppRole],
    }));
  }
}

// Role permissions matrix
const ROLE_PERMISSIONS: Record<AppRole, string[]> = {
  'platform_owner': [
    'users:read',
    'users:write',
    'users:delete',
    'roles:write',
    'organizations:write',
    'settings:write',
  ],
  'tenant_admin': [
    'users:read',
    'users:write',
    'roles:write', // Limited to student role only
    'organization:read',
    'settings:read',
  ],
  'instructor': [
    'users:read', // Limited to their students
    'courses:write',
    'assignments:write',
    'analytics:read',
  ],
  'student': [
    'courses:read',
    'assignments:read',
    'progress:read',
  ],
};
```
- Role definitions with permission matrix
- Helper methods to query permissions
- Used by RolesGuard to enforce access control

**`src/role/role.controller.ts`**
```typescript
@Controller('roles')
@UseGuards(JwtGuard)
export class RoleController {
  constructor(private readonly roleService: RoleService) {}

  @Get()
  async listRoles() {
    return this.roleService.listRoles();
  }

  @Get(':roleName')
  async getRole(@Param('roleName') roleName: AppRole) {
    return this.roleService.getRole(roleName);
  }
}
```

---

### Invitation Flow

**`src/invitation/invitation.service.ts`**
```typescript
@Injectable()
export class InvitationService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly clerk: ClerkService,
    private readonly events: EventsService,
  ) {}

  // Create/resend invitation
  async create(dto: CreateInvitationDto, caller: GatewayUser) {
    // Enforce hierarchy: can't invite someone with higher role
    this.enforceRoleHierarchy(caller, dto.role, dto.organizationId);

    // Check if user already exists
    const existingUser = await this.prisma.user.findFirst({
      where: { email: dto.email.toLowerCase() },
    });
    if (existingUser) {
      throw new ConflictException('USER_ALREADY_EXISTS');
    }

    // Generate secure token
    const token = randomBytes(32).toString('hex');
    const tokenHash = createHash('sha256').update(token).digest('hex');

    // Create/update invitation
    const invitation = await this.prisma.invitation.upsert({
      where: {
        email_organizationId: {
          email: dto.email.toLowerCase(),
          organizationId: dto.organizationId,
        },
      },
      update: {
        tokenHash,
        role: dto.role,
        status: 'pending',
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7 days
      },
      create: {
        email: dto.email.toLowerCase(),
        organizationId: dto.organizationId,
        role: dto.role,
        tokenHash,
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
      },
    });

    // Emit event for email service to pick up
    await this.events.emit({
      type: 'invitation.created',
      payload: {
        invitationId: invitation.id,
        email: invitation.email,
        role: invitation.role,
        token, // Unencrypted token for email link
        acceptUrl: `${process.env.PUBLIC_APP_URL}/invitations/${token}`,
      },
    });

    return {
      id: invitation.id,
      email: invitation.email,
      role: invitation.role,
      expiresAt: invitation.expiresAt,
      status: invitation.status,
    };
  }

  // Accept invitation and create user account
  async accept(dto: AcceptInvitationDto) {
    const { token, password } = dto;

    // Hash token for lookup
    const tokenHash = createHash('sha256').update(token).digest('hex');

    // Find invitation
    const invitation = await this.prisma.invitation.findUnique({
      where: { tokenHash },
    });

    if (!invitation) {
      throw new NotFoundException('Invitation not found or expired');
    }

    if (invitation.status !== 'pending') {
      throw new BadRequestException('Invitation already used');
    }

    if (new Date() > invitation.expiresAt) {
      throw new BadRequestException('Invitation expired');
    }

    // Create Clerk account
    const clerkUser = await this.clerk.createUser({
      email: invitation.email,
      password,
    });

    // Create local user record
    const user = await this.prisma.user.create({
      data: {
        clerkUserId: clerkUser.id,
        email: invitation.email,
        primaryOrganizationId: invitation.organizationId,
        roles: {
          create: {
            role: invitation.role as AppRole,
            organizationId: invitation.organizationId,
          },
        },
      },
      include: { roles: true },
    });

    // Mark invitation as accepted
    await this.prisma.invitation.update({
      where: { id: invitation.id },
      data: {
        status: 'accepted',
        acceptedAt: new Date(),
      },
    });

    // Sync roles to Clerk metadata
    await this.clerk.syncRolesToMetadata(
      clerkUser.id,
      user.roles,
      user.primaryOrganizationId,
    );

    // Emit event for onboarding service
    await this.events.emit({
      type: 'invitation.accepted',
      payload: {
        userId: user.id,
        clerkUserId: clerkUser.id,
        email: user.email,
        role: invitation.role,
      },
    });

    return {
      userId: user.id,
      clerkUserId: clerkUser.id,
      email: user.email,
      role: invitation.role,
    };
  }

  // Bulk invite users
  async bulkInvite(dto: BulkInviteDto, caller: GatewayUser) {
    const invitations = await Promise.all(
      dto.emails.map((email) =>
        this.create(
          {
            email,
            role: dto.role,
            organizationId: dto.organizationId,
          },
          caller,
        ),
      ),
    );

    return {
      count: invitations.length,
      invitations,
    };
  }
}
```
- Invitation lifecycle:
  1. **create()**: Generate secure token (SHA256), store hash, emit event for email service
  2. **accept()**: Validate token & expiration, create Clerk user, create local user, sync roles
  3. **bulkInvite()**: Create multiple invitations in parallel
- Security:
  - Token never stored plaintext (only hash)
  - 7-day expiration
  - Status prevents duplicate acceptance
  - Role hierarchy enforced (can't invite higher privilege)

**`src/invitation/invitation.controller.ts`**
```typescript
@Controller('invitations')
@UseGuards(JwtGuard)
export class InvitationController {
  constructor(private readonly invitationService: InvitationService) {}

  @Post()
  @Roles('tenant_admin', 'platform_owner')
  async createInvitation(
    @Body() dto: CreateInvitationDto,
    @CurrentUser() caller: GatewayUser,
  ) {
    return this.invitationService.create(dto, caller);
  }

  @Post('bulk')
  @Roles('tenant_admin', 'platform_owner')
  async bulkInvite(
    @Body() dto: BulkInviteDto,
    @CurrentUser() caller: GatewayUser,
  ) {
    return this.invitationService.bulkInvite(dto, caller);
  }

  @Post(':token/accept')
  @Public() // Public endpoint (no auth required)
  async acceptInvitation(@Body() dto: AcceptInvitationDto) {
    return this.invitationService.accept(dto);
  }
}
```
- Routes:
  - `POST /api/invitations` → Create invitation (admin only)
  - `POST /api/invitations/bulk` → Bulk create (admin only)
  - `POST /api/invitations/:token/accept` → Public endpoint (anyone with token)

---

### Redis Caching Layer

**`src/redis/redis.service.ts`**
```typescript
@Injectable()
export class RedisService {
  private readonly redis: Redis;
  private readonly logger = new Logger(RedisService.name);

  constructor(private readonly configService: ConfigService) {
    const redisUrl = this.configService.getOrThrow<string>('REDIS_URL');
    this.redis = new Redis(redisUrl);

    this.redis.on('error', (err) =>
      this.logger.error(`Redis error: ${err.message}`),
    );
    this.redis.on('connect', () =>
      this.logger.log('Redis connected'),
    );
  }

  async get<T>(key: string): Promise<T | null> {
    const value = await this.redis.get(key);
    return value ? JSON.parse(value) : null;
  }

  async set<T>(key: string, value: T, ttlSeconds?: number): Promise<void> {
    const json = JSON.stringify(value);
    if (ttlSeconds) {
      await this.redis.setex(key, ttlSeconds, json);
    } else {
      await this.redis.set(key, json);
    }
  }

  async delete(key: string): Promise<void> {
    await this.redis.del(key);
  }

  async ping(): Promise<string> {
    return this.redis.ping();
  }

  async onModuleDestroy() {
    await this.redis.quit();
  }
}
```
- Wrapper around ioredis client
- Methods:
  - **get()**: Retrieve + auto-parse JSON
  - **set()**: Store with optional TTL
  - **delete()**: Remove key
  - **ping()**: Health check (used by readiness probe)
- Used for caching (user profiles, role permissions, invitation tokens)
- TTL prevents stale cache (default: 5 min)

**`src/redis/redis.module.ts`**
```typescript
@Module({
  providers: [RedisService],
  exports: [RedisService],
})
export class RedisModule {}
```

---

### Event Bus

**`src/events/events.service.ts`**
```typescript
@Injectable()
export class EventsService {
  private readonly logger = new Logger(EventsService.name);
  private readonly eventEmitter = new EventEmitter2();

  // Publish event
  async emit(event: DomainEvent) {
    this.logger.log(`Event emitted: ${event.type}`);
    this.eventEmitter.emit(event.type, event.payload);
  }

  // Subscribe to event
  subscribe(eventType: string, handler: (payload: any) => Promise<void>) {
    this.eventEmitter.on(eventType, async (payload) => {
      try {
        await handler(payload);
      } catch (err) {
        this.logger.error(
          `Error in event handler for ${eventType}: ${(err as Error).message}`,
        );
      }
    });
  }
}

export interface DomainEvent {
  type: string;
  payload: unknown;
  timestamp?: Date;
}
```
- In-memory event bus (EventEmitter2)
- Events emitted in this service:
  - `user.created` → Emitted by webhook handler
  - `invitation.created` → Emitted when invitation sent
  - `invitation.accepted` → Emitted when user accepts invite
  - `role.assigned` → Emitted when role granted
- Used by other services to react (e.g., Notification service listens for `invitation.created`)
- Future: Can be upgraded to NATS/RabbitMQ for cross-service events

**`src/events/events.module.ts`**
```typescript
@Module({
  providers: [EventsService],
  exports: [EventsService],
})
export class EventsModule {}
```

---

### Common Cross-Cutting Concerns

**`src/common/decorators/current-user.decorator.ts`**
```typescript
export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user; // Set by JwtGuard in middleware
  },
);
```
- Usage: `@CurrentUser() user: GatewayUser`
- Extracts authenticated user from request (set by JwtGuard)

**`src/common/decorators/public.decorator.ts`**
```typescript
export const Public = () => SetMetadata('isPublic', true);
```
- Usage: `@Public()`
- Marks route as public (skips JwtGuard if present)
- Used for: invitation acceptance, health checks

**`src/common/decorators/roles.decorator.ts`**
```typescript
export const Roles = (...roles: AppRole[]) =>
  SetMetadata('roles', roles);
```
- Usage: `@Roles('tenant_admin', 'instructor')`
- Marks route as requiring specific role
- Checked by RolesGuard

**`src/common/guards/jwt.guard.ts`**
```typescript
@Injectable()
export class JwtGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const isPublic = this.reflector.get<boolean>('isPublic', context.getHandler());

    if (isPublic) {
      return true; // Skip auth for public routes
    }

    const authHeader = request.headers.authorization;
    if (!authHeader?.startsWith('Bearer ')) {
      throw new UnauthorizedException('Missing token');
    }

    const token = authHeader.slice(7);

    try {
      // Verify JWT (Clerk token or custom service token)
      const payload = await verifyToken(token);
      request.user = payload; // Set for @CurrentUser()
      return true;
    } catch (err) {
      throw new UnauthorizedException('Invalid token');
    }
  }
}
```
- Main authentication guard
- Checks:
  1. Is route public? (marked with @Public())
  2. Is Authorization header present?
  3. Is JWT valid? (signature + expiry)
- Sets `request.user` for @CurrentUser() injection
- Applied globally or per-controller

**`src/common/guards/roles.guard.ts`**
```typescript
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>('roles', context.getHandler());

    if (!requiredRoles) {
      return true; // No role requirement, allow
    }

    const request = context.switchToHttp().getRequest();
    const user: GatewayUser = request.user;

    // Check if user has any required role
    return requiredRoles.some((role) => user.roles.includes(role as AppRole));
  }
}
```
- RBAC enforcement guard
- Checks: Does user have required role?
- Applied after JwtGuard

**`src/common/guards/internal-api-key.guard.ts`**
```typescript
@Injectable()
export class InternalApiKeyGuard implements CanActivate {
  constructor(private configService: ConfigService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const apiKey = request.headers['x-api-key'];

    if (!apiKey) {
      throw new UnauthorizedException('Missing API key');
    }

    const expectedKey = this.configService.getOrThrow<string>('INTERNAL_API_KEY');

    if (apiKey !== expectedKey) {
      throw new UnauthorizedException('Invalid API key');
    }

    return true;
  }
}
```
- Service-to-service authentication
- Used on internal endpoints (not exposed to frontend)
- Requires: `x-api-key` header

**`src/common/interceptors/logging.interceptor.ts`**
```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger(LoggingInterceptor.name);

  intercept(context: ExecutionContext, next: CallHandler) {
    const request = context.switchToHttp().getRequest();
    const { method, url } = request;
    const startTime = Date.now();

    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - startTime;
        this.logger.log(
          `${method} ${url} - ${duration}ms`,
        );
      }),
      catchError((err) => {
        const duration = Date.now() - startTime;
        this.logger.error(
          `${method} ${url} - ${duration}ms - ${err.message}`,
        );
        throw err;
      }),
    );
  }
}
```
- Logs all requests with duration
- Used for performance monitoring

**`src/common/interceptors/request-id.interceptor.ts`**
```typescript
@Injectable()
export class RequestIdInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    const request = context.switchToHttp().getRequest();
    const requestId = randomUUID();

    request.id = requestId;
    return next.handle();
  }
}
```
- Adds correlation ID to each request
- Used for distributed tracing (logs from multiple services have same requestId)

**`src/common/pipes/uuid-validation.pipe.ts`**
```typescript
@Injectable()
export class UuidValidationPipe implements PipeTransform {
  transform(value: string) {
    if (!this.isValidUuid(value)) {
      throw new BadRequestException('Invalid UUID format');
    }
    return value;
  }

  private isValidUuid(value: string): boolean {
    return /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i.test(value);
  }
}
```
- Validates UUID format in route params
- Usage: `@Param('userId', UuidValidationPipe) userId: string`

**`src/common/filters/all-exceptions.filter.ts`**
```typescript
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: unknown, host: ExecutionContext) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<FastifyReply>();
    const request = ctx.getRequest<FastifyRequest>();

    let status = 500;
    let message = 'Internal server error';
    let code = 'INTERNAL_ERROR';

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      message = exception.message;
    }

    this.logger.error(`${request.method} ${request.url} - ${message}`);

    response.status(status).send({
      statusCode: status,
      message,
      code,
      timestamp: new Date().toISOString(),
    });
  }
}
```
- Global exception handler
- Catches all errors and formats response
- Applied globally in main.ts

---

## LUMINA LEARNING HUB

**Purpose**: Modern React SPA providing student/instructor/admin interfaces to the Lumina LMS. Acts as the monolith transitioning to microservices via the gateway pattern.

**Technology Stack**:
- Framework: React 18.3 + Vite
- Language: TypeScript
- Server State: TanStack React Query v5
- Style: Tailwind CSS + PostCSS
- Components: Radix UI + shadcn/ui (30+ components)
- Authentication: Clerk (Firebase alternative)
- Database: Supabase (PostgreSQL with RLS)
- Edge Functions: Deno (30+ edge functions)
- Analytics: PostHog
- Animations: Framer Motion
- Bundler: Vite (instant HMR)
- Package Manager: pnpm (faster than npm)
- Testing: Vitest + React Testing Library

### Root Configuration & Build Files

**`package.json`**
- Project: lumina-learning-hub, version 0.1.0
- Scripts:
  - `npm run dev` → Vite dev server (port 5173)
  - `npm run build` → Production build
  - `npm run preview` → Preview production build locally
  - `npm run test` → Vitest (watch mode)
  - `npm run coverage` → Test coverage report
  - `npm run lint` → ESLint checks
- Key dependencies (50+):
  - **react 18.3.1** + react-dom
  - **@clerk/clerk-react** 5.x (auth provider)
  - **@supabase/supabase-js** 2.89 (DB client)
  - **@tanstack/react-query** 5.83 (server state)
  - **radix-ui** ~1.0 (unstyled components + accessibility)
  - **shadcn-ui** 0.8+ (Tailwind-styled components)
  - **framer-motion** 11 (animations)
  - **marked** + **dompurify** (markdown rendering)
  - **sonner** (toast notifications)
  - **zod** (validation)
  - **@hookform/react** + **react-hook-form** (form management)
  - **next-themes** (dark mode toggle)
  - **posthog** (analytics)
  - Vite plugins: @vitejs/plugin-react, vite-plugin-svgr

**`vite.config.ts`**
```typescript
export default defineConfig({
  plugins: [react(), svgr()],
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:3008',
        changeOrigin: true,
      },
    },
  },
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom'],
          'radix-ui': ['@radix-ui/'],
        },
      },
    },
  },
});
```
- Dev server on port 5173
- API proxy to backend (auth-service on 3008)
- Manual code splitting for better caching (React, Radix-UI separate bundles)

**`tsconfig.json`**
- Strict mode, decorators, JSX support
- Path aliases: `@/*` → `src/`

**`vitest.config.ts`**
- Unit test config
- Includes coverage thresholds (>80% for app code)

**`tailwind.config.ts`**
- CSS framework configuration
- Theme: colors, spacing, shadows
- Plugins: @tailwindcss/typography (markdown styles)
- Dark mode: class-based (from next-themes)

**`postcss.config.js`**
- PostCSS plugins: Tailwind, Autoprefixer

**`components.json`**
- shadcn/ui component configuration
- Specifies: components directory, alias, typescript

**`index.html`**
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta property="og:title" content="Lumina Learning Hub" />
    <meta name="description" content="AI-powered learning platform" />
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```
- Entry point for React rendering

---

### Source Code: Core Application

**`src/main.tsx`**
```typescript
import React from 'react';
import ReactDOM from 'react-dom/client';
import { ChakraProvider } from '@chakra-ui/react';
import App from './App';
import './index.css';

ReactDOM.createRoot(document.getElementById('app')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
);
```
- React entry point
- Renders App component into #app div

**`src/App.tsx`**
```typescript
import { QueryClientProvider } from '@tanstack/react-query';
import { ClerkProvider } from '@clerk/clerk-react';
import { BrowserRouter, Routes, Route, Suspense, lazy } from 'react-router-dom';
import { Toaster } from 'sonner';
import { ThemeProvider } from 'next-themes';
import queryClient from './lib/queryClient';
import AuthProvider from './hooks/useAuth';
import ProtectedRoute from './components/ProtectedRoute';
import LoadingSpinner from './components/common/Loading';

// Lazy load pages (code splitting)
const Index = lazy(() => import('./pages/Index'));
const Auth = lazy(() => import('./pages/Auth'));
const Onboarding = lazy(() => import('./pages/Onboarding'));
const Dashboard = lazy(() => import('./pages/dashboard/Index'));
const NotFound = lazy(() => import('./pages/NotFound'));

export default function App() {
  return (
    <ClerkProvider publishableKey={import.meta.env.VITE_CLERK_PUBLISHABLE_KEY}>
      <QueryClientProvider client={queryClient}>
        <ThemeProvider attribute="class" defaultTheme="light">
          <AuthProvider>
            <BrowserRouter>
              <Suspense fallback={<LoadingSpinner />}>
                <Routes>
                  <Route path="/" element={<Index />} />
                  <Route path="/auth/*" element={<Auth />} />
                  <Route path="/onboarding" element={<Onboarding />} />
                  
                  {/* Protected routes */}
                  <Route element={<ProtectedRoute />}>
                    <Route path="/dashboard/*" element={<Dashboard />} />
                  </Route>
                  
                  <Route path="*" element={<NotFound />} />
                </Routes>
              </Suspense>
            </BrowserRouter>
            <Toaster richColors position="top-right" />
          </AuthProvider>
        </ThemeProvider>
      </QueryClientProvider>
    </ClerkProvider>
  );
}
```
- Root component composition:
  1. **ClerkProvider** → Clerk authentication (Publishable key from env)
  2. **QueryClientProvider** → React Query setup (server state management)
  3. **ThemeProvider** → Dark/light mode toggle
  4. **AuthProvider** → Custom auth context (user profile, roles)
  5. **BrowserRouter** → Client-side routing
  6. **Routes** → Define public/protected routes
  7. **Toaster** → Toast notifications (from Sonner)
- Lazy loading: Each page is code-split (loaded only when needed)
- Error recovery: Chunk reload retry on failed imports (handles deploys)

**`src/index.css`**
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  --primary: 59 130 246; /* Blue-500 */
  --secondary: 139 92 246; /* Violet-500 */
  --destructive: 239 68 68; /* Red-500 */
  --muted: 107 114 128; /* Gray-500 */
}

body {
  @apply bg-white dark:bg-slate-950 transition-colors;
  font-family: 'Inter', system-ui, -apple-system, sans-serif;
}

html {
  scroll-behavior: smooth;
}
```
- Global Tailwind directives
- CSS variables for theme colors
- Base styles for typography, spacing

**`src/App.css`**
```css
.page-transition {
  animation: fadeIn 0.3s ease-in-out;
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}
```
- Global animations and utility classes

---

### Core Hooks

**`src/hooks/useAuth.tsx`** (Central authentication hook)
```typescript
import { createContext, useContext, useEffect, useState } from 'react';
import { User, Session } from '@supabase/supabase-js';
import { supabase } from '@/integrations/supabase/client';
import { gatewayRequest } from '@/lib/gateway-client';

export type AppRole = 'platform_owner' | 'tenant_admin' | 'instructor' | 'student';

interface AuthContextType {
  user: User | null;
  session: Session | null;
  profile: Profile | null;
  userRole: AppRole | null;
  userRoles: UserRole[];
  organizationId: string | null;
  loading: boolean;
  signIn: (email: string, password: string) => Promise<{ error: Error | null }>;
  signUp: (email: string, password: string, fullName?: string) => Promise<{ error: Error | null }>;
  signOut: () => Promise<void>;
  updateProfile: (updates: Partial<Profile>) => Promise<{ error: Error | null }>;
  hasRole: (role: AppRole, orgId?: string | null) => boolean;
  isPlatformOwner: () => boolean;
  isTenantAdmin: () => boolean;
  isInstructor: () => boolean;
  isStudent: () => boolean;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [session, setSession] = useState<Session | null>(null);
  const [profile, setProfile] = useState<Profile | null>(null);
  const [userRoles, setUserRoles] = useState<UserRole[]>([]);
  const [loading, setLoading] = useState(true);

  // Subscribe to Supabase auth state changes
  useEffect(() => {
    const { data: listener } = supabase.auth.onAuthStateChange(
      async (event, updatedSession) => {
        setSession(updatedSession);
        setUser(updatedSession?.user || null);

        if (updatedSession?.user) {
          // Fetch user profile from microservice (gateway)
          try {
            const response = await gatewayRequest(
              '/api/users/me',
              { method: 'GET' },
              updatedSession.access_token,
            );
            setProfile(response.data);
            setUserRoles(response.data.roles);
          } catch (err) {
            console.error('Failed to fetch profile:', err);
          }
        } else {
          setProfile(null);
          setUserRoles([]);
        }

        setLoading(false);
      },
    );

    return () => listener.subscription.unsubscribe();
  }, []);

  const signIn = async (email: string, password: string) => {
    try {
      const { error } = await supabase.auth.signInWithPassword({
        email,
        password,
      });
      return { error };
    } catch (err) {
      return { error: err as Error };
    }
  };

  const signUp = async (email: string, password: string, fullName?: string) => {
    try {
      // Step 1: Create Clerk account (via gateway)
      const signUpResult = await gatewayRequest(
        '/api/signup',
        {
          method: 'POST',
          body: JSON.stringify({ email, password, fullName }),
        },
        null,
      );

      // Step 2: User now has Clerk ID + local user record
      // Frontend logs in via Supabase (mirror auth)
      const { error } = await supabase.auth.signInWithPassword({
        email,
        password,
      });
      return { error };
    } catch (err) {
      return { error: err as Error };
    }
  };

  const signOut = async () => {
    await supabase.auth.signOut();
    setUser(null);
    setProfile(null);
  };

  const updateProfile = async (updates: Partial<Profile>) => {
    try {
      const response = await gatewayRequest(
        `/api/users/${profile?.id}`,
        {
          method: 'PATCH',
          body: JSON.stringify(updates),
        },
        session?.access_token,
      );
      setProfile(response.data);
      return { error: null };
    } catch (err) {
      return { error: err as Error };
    }
  };

  const primaryRole = userRoles.length > 0 ? userRoles[0].role : null;

  const value: AuthContextType = {
    user,
    session,
    profile,
    userRole: primaryRole,
    userRoles,
    organizationId: profile?.organization_id || null,
    loading,
    signIn,
    signUp,
    signOut,
    updateProfile,
    hasRole: (role, orgId) =>
      userRoles.some(
        (ur) =>
          ur.role === role &&
          (!orgId || ur.organization_id === orgId),
      ),
    isPlatformOwner: () => primaryRole === 'platform_owner',
    isTenantAdmin: () => primaryRole === 'tenant_admin',
    isInstructor: () => primaryRole === 'instructor',
    isStudent: () => primaryRole === 'student',
  };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};
```
- Central authentication hook
- Responsibilities:
  - Listen to Supabase auth state changes
  - Fetch user profile from gateway (auth-service)
  - Manage sign-in/sign-up/sign-out
  - Track roles and permissions
  - Provide helper methods (hasRole, isPlatformOwner, etc.)
- Uses gateway-client to fetch profile from microservice (fallback to Supabase)

**`src/hooks/useEnrollment.ts`**
```typescript
import { useQuery } from '@tanstack/react-query';
import { gatewayRequest } from '@/lib/gateway-client';
import { useAuth } from './useAuth';

export function useEnrollment() {
  const { session } = useAuth();

  return useQuery({
    queryKey: ['enrollments', session?.user.id],
    queryFn: async () => {
      const data = await gatewayRequest(
        '/api/enrollments',
        { method: 'GET' },
        session?.access_token,
      );
      return data;
    },
    enabled: !!session,
  });
}
```
- Fetches user's course enrollments
- Uses React Query with caching (depends on session)
- Falls back to Supabase if gateway unavailable

**`src/hooks/useCourse.ts`**
```typescript
export function useCourse(courseId: string) {
  return useQuery({
    queryKey: ['course', courseId],
    queryFn: () =>
      gatewayRequest(
        `/api/courses/${courseId}`,
        { method: 'GET' },
      ),
  });
}
```
- Fetch single course with optional lesson list

**`src/hooks/useProgress.ts`**
```typescript
export function useProgress(courseId: string) {
  const { session } = useAuth();

  return useQuery({
    queryKey: ['progress', courseId, session?.user.id],
    queryFn: () =>
      gatewayRequest(
        `/api/enrollments/${courseId}/progress`,
        { method: 'GET' },
        session?.access_token,
      ),
  });
}
```
- User's course progress (lessons completed, time spent, quiz scores)

**`src/hooks/useRewards.ts`**
```typescript
export function useRewards() {
  const { session } = useAuth();

  return useQuery({
    queryKey: ['rewards', session?.user.id],
    queryFn: () =>
      gatewayRequest(
        '/api/rewards',
        { method: 'GET' },
        session?.access_token,
      ),
  });
}
```
- User's gamification stats (XP, achievements, leaderboard rank)

**Additional Hooks** (20+ total):
- `useAnalytics()` → Fetch analytics dashboard data
- `useNotifications()` → User notification list
- `useLearningPath()` → AI-recommended learning path
- `usePersonalizationRules()` → Per-user personalization settings
- `useMobile()` → Responsive design detection

---

### Gateway Client (Core Integration Pattern)

**`src/lib/gateway-client.ts`** (Production-ready fallback pattern)
```typescript
import { supabase } from '@/integrations/supabase/client';

const GATEWAY_URL = (
  import.meta.env.VITE_GATEWAY_URL ||
  import.meta.env.VITE_API_GATEWAY_URL
)?.replace(/\/$/, '');

const DEFAULT_TIMEOUT_MS = 10_000;
export const AI_GENERATION_TIMEOUT_MS = 30_000;

export interface GatewayErrorShape {
  code?: string;
  message: string;
  requestId?: string;
  details?: unknown;
}

export class GatewayError extends Error {
  readonly status: number;
  readonly code: string | undefined;
  readonly requestId: string | undefined;

  constructor(
    message: string,
    status: number,
    shape?: GatewayErrorShape,
  ) {
    super(message);
    this.name = 'GatewayError';
    this.status = status;
    this.code = shape?.code;
    this.requestId = shape?.requestId;
  }
}

/**
 * Make request to API Gateway (or fallback to Supabase RPC)
 * @param endpoint - API endpoint (e.g. '/api/users/me')
 * @param options - Fetch options (method, body, headers)
 * @param token - JWT token (optional, from Supabase session)
 * @returns Response data
 * @throws GatewayError if both gateway and RPC fail
 */
export async function gatewayRequest(
  endpoint: string,
  options: RequestInit = {},
  token: string | null = null,
): Promise<any> {
  // Try gateway first
  if (GATEWAY_URL) {
    try {
      const response = await fetch(`${GATEWAY_URL}${endpoint}`, {
        ...options,
        headers: {
          'Content-Type': 'application/json',
          ...options.headers,
          ...(token && { Authorization: `Bearer ${token}` }),
        },
        signal: AbortSignal.timeout(DEFAULT_TIMEOUT_MS),
      });

      if (response.ok) {
        return await response.json();
      }

      // Gateway error → try RPC fallback
      if (response.status >= 500) {
        console.warn('[gateway-client] Gateway error, trying RPC fallback');
      } else {
        // Client error (400-499) → don't retry
        throw new GatewayError(
          response.statusText,
          response.status,
          await response.json(),
        );
      }
    } catch (err) {
      if (err instanceof GatewayError) throw err;
      console.warn('[gateway-client] Gateway failed:', err);
    }
  }

  // Fallback: Use Supabase RPC (for backward compatibility)
  console.log('[gateway-client] Falling back to Supabase RPC for', endpoint);
  const rpcResult = await supabase.rpc('gateway_fallback', {
    path: endpoint,
    method: options.method || 'GET',
    body: options.body ? JSON.parse(options.body as string) : null,
  });

  if (rpcResult.error) {
    throw new GatewayError(
      rpcResult.error.message,
      500,
      { message: rpcResult.error.message },
    );
  }

  return rpcResult.data;
}

/**
 * Execute request with retry + exponential backoff
 */
export async function gatewayRequestWithRetry(
  endpoint: string,
  options?: RequestInit,
  token?: string | null,
  maxRetries: number = 3,
): Promise<any> {
  let lastError: any;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await gatewayRequest(endpoint, options, token);
    } catch (err) {
      lastError = err;
      const delay = Math.pow(2, attempt) * 1000; // 1s, 2s, 4s
      console.warn(`[gateway-client] Retry attempt ${attempt + 1}, waiting ${delay}ms`);
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }

  throw lastError;
}
```
- **Pattern**: Gateway-first with Supabase RPC fallback
- **Flow**:
  1. Try API Gateway (microservice) → If success, return data
  2. If gateway fails (500+), fallback to Supabase RPC (monolith)
  3. If RPC fails, throw GatewayError
- **Timeout**: 10s default, 30s for AI generation
- **Retry**: Exponential backoff (1s, 2s, 4s)
- **Headers**: Automatically adds Authorization with JWT
- **Errors**: Custom GatewayError with status + code for client handling
- **Use**: All microservice calls use this (transparent fallback during migration)

**`src/lib/gateway-client.test.ts`**
```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { gatewayRequest, GatewayError } from './gateway-client';

describe('gatewayRequest', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('should return data from gateway on success', async () => {
    // Mock fetch
    global.fetch = vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({ data: 'test' }),
    });

    const result = await gatewayRequest('/api/users/me');
    expect(result.data).toBe('test');
  });

  it('should throw GatewayError on 4xx', async () => {
    global.fetch = vi.fn().mockResolvedValue({
      ok: false,
      status: 401,
      statusText: 'Unauthorized',
      json: () => Promise.resolve({ message: 'Invalid token' }),
    });

    await expect(gatewayRequest('/api/users/me')).rejects.toThrow(GatewayError);
  });

  it('should fallback to RPC on 5xx', async () => {
    global.fetch = vi.fn().mockResolvedValue({
      ok: false,
      status: 500,
    });

    // Mock Supabase RPC
    const supabase = await import('@/integrations/supabase/client');
    vi.spyOn(supabase, 'gatewayFallback').mockResolvedValue({
      data: { fallback: true },
    });

    const result = await gatewayRequest('/api/users/me');
    expect(result.fallback).toBe(true);
  });
});
```

---

### Components Organization

Components are organized by domain (learning domain, admin domain, etc.) for scalability:

**`src/components/` Structure:**
```
├── admin/                    # Admin dashboard
│   ├── UserManagement.tsx    # User CRUD
│   ├── Analytics.tsx         # Platform analytics
│   └── Settings.tsx          # Platform settings
├── analytics/               # Analytics visualizations
│   ├── ProgressChart.tsx     # User progress trends
│   └── EngagementMetrics.tsx # Platform engagement
├── assessment/             # Assessment/quiz UI
│   ├── QuestionCard.tsx      # Question display
│   ├── QuizPlayer.tsx        # Quiz runner
│   └── ResultsDisplay.tsx    # Score + feedback
├── assignments/            # Assignment submission
│   ├── AssignmentList.tsx    # User assignments
│   └── SubmissionForm.tsx    # File/text submission
├── common/                 # Shared utilities
│   ├── Loading.tsx           # Loading spinner
│   ├── ErrorBoundary.tsx     # Error fallback
│   └── EmptyState.tsx        # Empty state template
├── instructor/             # Instructor dashboard
│   ├── StudentManagement.tsx # View/manage students
│   ├── GradingPanel.tsx      # Grade submissions
│   └── AnalyticsDashboard.tsx # Class analytics
├── layout/                 # Layout components
│   ├── DashboardLayout.tsx   # Main dashboard wrapper
│   ├── Sidebar.tsx           # Navigation sidebar
│   └── Header.tsx            # Top header bar
├── learning/              # Learning/lesson components
│   ├── LessonPlayer.tsx      # Video + content player
│   ├── ReleaseGate.tsx       # Prerequisite checking
│   ├── ProgressBar.tsx       # Course progress
│   └── ResourceList.tsx      # Supplemental resources
├── notifications/         # Notification UI
│   ├── NotificationCenter.tsx # Notification inbox
│   └── NotificationBell.tsx   # Bell icon with count
├── onboarding/           # Signup/setup flow
│   ├── SignupForm.tsx        # Email/password signup
│   ├── ProfileSetup.tsx      # Complete profile
│   └── RoleSelection.tsx      # Choose role (student/instructor)
├── projects/             # Project management
│   ├── ProjectCard.tsx       # Project display
│   └── ProjectSubmission.tsx # Submit work
├── settings/             # User settings
│   ├── ProfileSettings.tsx   # Edit profile
│   └── PreferenceSettings.tsx # Notifications, theme, etc.
├── stage/                # Staging/demo components
│   └── FeatureDemo.tsx       # Feature showcase
├── ui/                   # shadcn/ui components
│   ├── Button.tsx            # Styled button
│   ├── Card.tsx              # Card container
│   ├── Dialog.tsx            # Modal
│   ├── Form.tsx              # Form utils
│   ├── Input.tsx             # Text input
│   ├── Tabs.tsx              # Tab navigation
│   ├── Dropdown.tsx          # Dropdown menu
│   └── ... (30+ total)
├── CourseCard.tsx        # Course listing card
├── NavLink.tsx           # Navigation link
├── ProtectedRoute.tsx    # Auth gate wrapper
├── RoleGuard.tsx         # RBAC gate wrapper
└── QueryErrorBoundary.tsx # React Query error handling
```

**Example: `src/components/learning/LessonPlayer.tsx`**
```typescript
import { useState, useEffect } from 'react';
import { useParams } from 'react-router-dom';
import { useLessonProgress } from '@/hooks/useLessonProgress';
import { markLessonComplete } from '@/integrations/supabase/lessons';
import { Button } from '@/components/ui/Button';
import { Card } from '@/components/ui/Card';

export default function LessonPlayer() {
  const { lessonId } = useParams();
  const { data: lesson, isLoading } = useLessonProgress(lessonId);
  const [isCompleted, setIsCompleted] = useState(false);

  const handleComplete = async () => {
    await markLessonComplete(lessonId);
    setIsCompleted(true);
  };

  if (isLoading) return <div>Loading...</div>;

  return (
    <Card className="max-w-4xl mx-auto p-6">
      <h1 className="text-3xl font-bold mb-4">{lesson.title}</h1>
      
      <div className="bg-black rounded-lg aspect-video mb-6">
        <video src={lesson.videoUrl} controls className="w-full h-full" />
      </div>

      <div className="prose max-w-full mb-6">
        {/* Render markdown with syntax highlighting */}
        <MarkdownRenderer content={lesson.description} />
      </div>

      <div className="flex gap-4">
        <Button 
          onClick={handleComplete} 
          disabled={isCompleted}
          className="bg-blue-600 hover:bg-blue-700"
        >
          {isCompleted ? 'Completed' : 'Mark as Complete'}
        </Button>
        <Button variant="outline">Next Lesson</Button>
      </div>
    </Card>
  );
}
```
- Props: None (uses URL params)
- State: Completion status, loading state
- Hooks: useLessonProgress (React Query), useAuth (context)
- Integration: Marks lesson complete on button click
- Styling: Tailwind + shadcn/ui components

**Example: `src/components/ProtectedRoute.tsx`**
```typescript
import { Navigate, Outlet } from 'react-router-dom';
import { useAuth } from '@/hooks/useAuth';

export default function ProtectedRoute() {
  const { user, loading } = useAuth();

  if (loading) return <div>Loading...</div>;
  if (!user) return <Navigate to="/auth/login" replace />;

  return <Outlet />;
}
```
- Wrapper for auth-protected routes
- Redirects unauthenticated users to login
- Used in: `<Route element={<ProtectedRoute />}><Route path="/dashboard" ...</Route></Route>`

**Example: `src/components/RoleGuard.tsx`**
```typescript
import { Navigate } from 'react-router-dom';
import { useAuth } from '@/hooks/useAuth';
import type { AppRole } from '@/hooks/useAuth';

interface RoleGuardProps {
  requiredRoles: AppRole[];
  fallback?: React.ReactNode;
  children: React.ReactNode;
}

export default function RoleGuard({
  requiredRoles,
  fallback = <Navigate to="/" replace />,
  children,
}: RoleGuardProps) {
  const { hasRole } = useAuth();

  if (!requiredRoles.some((role) => hasRole(role))) {
    return fallback;
  }

  return <>{children}</>;
}
```
- RBAC enforcer component
- Usage: `<RoleGuard requiredRoles={['instructor']}><TeacherDashboard /></RoleGuard>`

**Example: `src/components/ui/Button.tsx`** (shadcn/ui wrapper)
```typescript
import * as React from 'react';
import { Slot } from '@radix-ui/react-slot';
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils';

const buttonVariants = cva(
  'inline-flex items-center justify-center whitespace-nowrap rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent hover:text-accent-foreground',
        secondary: 'bg-secondary text-secondary-foreground hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline',
      },
      size: {
        default: 'h-10 px-4 py-2',
        sm: 'h-9 rounded-md px-3',
        lg: 'h-11 rounded-md px-8',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'default',
    },
  },
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean;
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : 'button';
    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    );
  },
);
Button.displayName = 'Button';

export { Button, buttonVariants };
```
- shadcn/ui Button component
- Uses CVA (class-variance-authority) for variant styling
- Props: HTML button attributes + variant (default, destructive, outline, etc.)
- Size options: default, sm, lg, icon

---

### Pages (Route Components)

**`src/pages/Index.tsx`** (Landing page)
```typescript
export default function Index() {
  return (
    <main className="min-h-screen flex flex-col">
      <header className="bg-gradient-to-r from-blue-600 to-purple-600 text-white py-16">
        <div className="max-w-4xl mx-auto px-4">
          <h1 className="text-5xl font-bold">Lumina Learning Hub</h1>
          <p className="text-xl mt-4">AI-powered learning platform for the future</p>
        </div>
      </header>

      <section className="flex-1 max-w-4xl mx-auto py-16 px-4">
        <div className="grid md:grid-cols-3 gap-8">
          <FeatureCard
            icon="🎓"
            title="Interactive Learning"
            description="Engaging lessons with video, quizzes, and exercises"
          />
          <FeatureCard
            icon="🤖"
            title="AI Tutor"
            description="Personalized tutoring and adaptive learning paths"
          />
          <FeatureCard
            icon="📊"
            title="Analytics"
            description="Track progress and identify learning gaps"
          />
        </div>
      </section>

      <footer className="bg-gray-900 text-white text-center py-8">
        <p>&copy; 2024 Lumina Learning. All rights reserved.</p>
      </footer>
    </main>
  );
}
```

**`src/pages/Auth.tsx`** (Authentication pages)
```typescript
import { Routes, Route } from 'react-router-dom';
import { SignInForm } from '@clerk/clerk-react';
import { SignUpForm } from '@clerk/clerk-react';

export default function Auth() {
  return (
    <Routes>
      <Route path="/login" element={<SignInForm />} />
      <Route path="/signup" element={<SignUpForm />} />
    </Routes>
  );
}
```
- Uses Clerk for pre-built forms (no custom auth forms)
- Routes: /auth/login, /auth/signup

**`src/pages/Onboarding.tsx`** (User setup flow)
```typescript
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { useAuth } from '@/hooks/useAuth';

export default function Onboarding() {
  const navigate = useNavigate();
  const { profile, updateProfile } = useAuth();
  const [step, setStep] = useState(1);
  const [formData, setFormData] = useState({
    fullName: profile?.full_name || '',
    role: '',
    careerGoal: '',
  });

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    try {
      await updateProfile({
        full_name: formData.fullName,
        career_goal: formData.careerGoal,
        onboarding_completed: true,
      });
      navigate('/dashboard');
    } catch (err) {
      console.error('Onboarding failed:', err);
    }
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-purple-50 flex items-center">
      <form onSubmit={handleSubmit} className="bg-white rounded-lg shadow-lg p-8 max-w-md">
        <h1 className="text-3xl font-bold mb-6">Welcome to Lumina</h1>

        {step === 1 && (
          <>
            <input
              type="text"
              placeholder="Full Name"
              value={formData.fullName}
              onChange={(e) => setFormData({ ...formData, fullName: e.target.value })}
              className="w-full border rounded-lg p-3 mb-4"
            />
            <button
              type="button"
              onClick={() => setStep(2)}
              className="w-full bg-blue-600 text-white py-2 rounded-lg"
            >
              Next
            </button>
          </>
        )}

        {step === 2 && (
          <>
            <select
              value={formData.role}
              onChange={(e) => setFormData({ ...formData, role: e.target.value })}
              className="w-full border rounded-lg p-3 mb-4"
            >
              <option>Select Role</option>
              <option value="student">Student</option>
              <option value="instructor">Instructor</option>
            </select>
            <textarea
              placeholder="What's your learning goal?"
              value={formData.careerGoal}
              onChange={(e) => setFormData({ ...formData, careerGoal: e.target.value })}
              className="w-full border rounded-lg p-3 mb-4"
            />
            <button type="submit" className="w-full bg-green-600 text-white py-2 rounded-lg">
              Get Started
            </button>
          </>
        )}
      </form>
    </div>
  );
}
```

**`src/pages/dashboard/Index.tsx`** (Student/instructor dashboard hub)
```typescript
import { Routes, Route } from 'react-router-dom';
import { useAuth } from '@/hooks/useAuth';
import DashboardLayout from '@/components/layout/DashboardLayout';
import StudentDashboard from './StudentDashboard';
import InstructorDashboard from './InstructorDashboard';
import AdminDashboard from './AdminDashboard';

export default function Dashboard() {
  const { hasRole } = useAuth();

  return (
    <DashboardLayout>
      <Routes>
        {hasRole('platform_owner') && (
          <Route path="/admin/*" element={<AdminDashboard />} />
        )}
        {hasRole('instructor') && (
          <Route path="/instructor/*" element={<InstructorDashboard />} />
        )}
        {hasRole('student') && (
          <Route path="/student/*" element={<StudentDashboard />} />
        )}
        <Route path="/" element={<StudentDashboard />} />
      </Routes>
    </DashboardLayout>
  );
}
```

---

### Supabase Integration

**`src/integrations/supabase/client.ts`**
```typescript
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

if (!supabaseUrl || !supabaseAnonKey) {
  throw new Error('Missing Supabase credentials');
}

export const supabase = createClient(supabaseUrl, supabaseAnonKey);

// Initialize session on app load
supabase.auth.getSession().then(({ data: { session } }) => {
  console.log('Supabase session:', session?.user?.email);
});
```
- Single Supabase client instance (singleton)
- Used for:
  - Authentication (sign-in, sign-up, sign-out)
  - Real-time subscriptions (if needed)
  - Fallback RPC calls (gateway_fallback function)
- Credentials from environment variables

**`supabase/functions/` (30+ Deno edge functions)**
- Deno TypeScript edge functions running on Supabase
- Examples:
  - `send-invitation/` → Send email when invitation created
  - `track-milestone/` → Update user progress
  - `check-achievements/` → Award achievements
  - `process-quiz/` → Auto-grade quizzes
  - `generate-roadmap/` → AI-generate learning path
- Pattern: Triggered by Postgres events or HTTP (webhook)
- Fallback functions (created in Phase 0):
  - `gateway_fallback/` → RPC wrapper for all microservices

---

### Testing

**`src/components/ProtectedRoute.test.tsx`**
```typescript
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import ProtectedRoute from './ProtectedRoute';
import * as AuthModule from '@/hooks/useAuth';

describe('ProtectedRoute', () => {
  it('should render children when authenticated', () => {
    vi.spyOn(AuthModule, 'useAuth').mockReturnValue({
      user: { id: '123' },
      loading: false,
      // ... other properties
    });

    render(
      <BrowserRouter>
        <ProtectedRoute>
          <div>Protected Content</div>
        </ProtectedRoute>
      </BrowserRouter>,
    );

    expect(screen.getByText('Protected Content')).toBeInTheDocument();
  });

  it('should redirect when not authenticated', () => {
    vi.spyOn(AuthModule, 'useAuth').mockReturnValue({
      user: null,
      loading: false,
      // ... other properties
    });

    render(
      <BrowserRouter>
        <ProtectedRoute>
          <div>Protected Content</div>
        </ProtectedRoute>
      </BrowserRouter>,
    );

    expect(screen.queryByText('Protected Content')).not.toBeInTheDocument();
  });
});
```
- Uses Vitest + React Testing Library
- Mocks useAuth hook for testing
- Tests both authenticated and unauthenticated flows

---

## INTEGRATION ARCHITECTURE

### How It Works Today (Monolith)

```
┌─────────────────────────────────────────────┐
│  Browser (React SPA)                        │
│  - All 1,000+ Supabase calls direct         │
│  - RLS policies enforce auth                │
│  - Edge functions for compute               │
└──────────────┬──────────────────────────────┘
               │
       ┌───────▼────────┐
       │   Supabase     │
       │  (Monolith)    │
       │ - PostgreSQL   │
       │ - RLS Policies │
       │ - Edge Fns     │
       └────────────────┘
```

### How It Will Work (Post-Migration)

```
┌────────────────────────────────────────────┐
│  Browser (React SPA)                       │
│  - Uses gateway-client for all requests    │
│  - Falls back to Supabase RPC if needed    │
└──────────────┬─────────────────────────────┘
               │
       ┌───────▼────────────┐
       │  API Gateway       │
       │ (Kong/Traefik)     │
       │ - Route by service │
       │ - Rate limiting    │
       │ - CORS             │
       └───────┬────────────┘
               │
       ┌───────┴──────────────────┐
       │                          │
   ┌───▼─────────────┐   ┌───────▼──────────┐
   │  Auth Service   │   │ Notification Svc │
   │ (NestJS)        │   │  (NestJS)        │
   │ - Clerk tokens  │   │ - Send emails    │
   │ - RBAC          │   │ - Webhooks       │
   │ - Audit log     │   │ - Event listener │
   └─────────────────┘   └──────────────────┘

   ┌──────────────────┐   ┌──────────────────┐
   │ Gamification Svc │   │  AI Tutor Svc    │
   │ (NestJS)         │   │ (NestJS + LLM)   │
   │ - XP/achievements│   │ - Chat interface │
   │ - Leaderboards   │   │ - RAG            │
   └──────────────────┘   └──────────────────┘

   ... (5 more services)

       │
       └────────────────────┐
                            │
                    ┌───────▼──────────┐
                    │   Supabase       │
                    │  (Data layer)    │
                    │ - Per-service DB │
                    │ - Event sync     │
                    └──────────────────┘
```

### Gateway Client Behavior

**Request Flow:**
```
User Action (e.g., fetch enrollments)
         │
         ▼
useEnrollment() hook
         │
         ▼
gatewayRequest('/api/enrollments')
         │
    ┌────┴────┐
    │          │
    ▼          ▼
  Gateway   Timeout
  (Port    (10s)
   :3008)    │
    │        ▼
    ├──→ Supabase RPC
    │    (Fallback)
    │        │
    └────┬───┘
         │
         ▼
      Response
```

**Example Request:**
```typescript
const { data: enrollments, error } = await gatewayRequest(
  '/api/enrollments',
  { method: 'GET' },
  session.access_token, // JWT
);
```

1. Try: `POST http://localhost:3008/api/enrollments` with Bearer token
   - Success (200-299): Return response
   - Client error (400-499): Throw GatewayError (don't retry)
   - Server error (500+): Try fallback

2. Fallback: Call Supabase Edge Function
   - `supabase.rpc('gateway_fallback', { path, method, body })`
   - Returns data from Supabase (via Postgres)

### Data Ownership
- **Auth service**: User, UserRole, Invitation, AuthAuditLog
- **Notification service**: Notifications, NotificationPreferences, EmailLogs
- **Gamification service**: XPLogs, Achievements, UserAchievements, Leaderboard
- **AI Tutor service**: TutorSessions, TutorMessages, RAGDocuments
- **Analytics service**: EventLogs, DashboardQueries
- **Assessment service**: Assessments, Questions, UserAnswers, Gradings
- **Course/Enrollment service**: Courses, Enrollments, Progress, Milestones
- **Supabase (shared/legacy)**: All legacy tables (shared until fully migrated)

### Event Flow
```
User accepts invitation
         │
         ▼
invitation.service.accept()
         │
         ▼
eventsService.emit('invitation.accepted', { userId, role, ... })
         │
    ┌────┴────────────────────────────┐
    │                                 │
    ▼                                 ▼
EventBus (in-memory)            NATS (cross-service)
    │                                 │
    ├→ AuthService listeners         ├→ NotificationService
    │  (log user creation)           │  (send welcome email)
    │                               │
    ├→ AiTutor listeners           ├→ GamificationService
    │  (create user profile)        │  (assign starter XP)
    │                              │
    └→ Other local handlers        └→ Other services
```

---

## MIGRATION PLAN ALIGNMENT

### Phase 0: Gateway Infrastructure ✅
**Status**: Implemented
- **gateway-client.ts**: Dual-mode fallback pattern (gateway + RPC)
- **_shared/gateway-wrapper.ts**: Supabase wrapper helpers
- **_shared/cors.ts**: CORS middleware for edge functions
- **WRAPPER_CONFIG.env**: Configuration template
- **QUICK_START_WRAPPER.md**: 2-hour POC guide

### Phase 1: Notification Service (In Progress)
**Status**: Scaffolded (see PHASE_1_NOTIFICATION_SERVICE.md)
- Extract: Send-invitation, send-email, notification-center
- Technology: NestJS + Fastify + Prisma + Sendgrid/Resend
- Data: Notifications table, NotificationPreferences
- Integration Point: Listen for events from auth-service
- Gateway Routes:
  - `POST /api/notifications` → Send notification
  - `GET /api/notifications` → Fetch user notifications
  - `PATCH /api/notifications/:id` → Mark as read

### Phases 2-7: Service Scaffolds Ready
- **PHASE_2**: Gamification (XP, achievements, leaderboards)
- **PHASE_3**: AI Tutor (LLM integration, RAG, streaming)
- **PHASE_4**: Analytics (event aggregation, dashboards)
- **PHASE_5**: Assessment (auto-grading, code execution)
- **PHASE_6**: Course/Enrollment (core data, progress)
- **PHASE_7**: Already partially implemented (auth-service)

### Key Integration Points

**How each service connects to monitoring-hub:**

1. **Auth-Service** (`/api/users/*`, `/api/roles/*`, `/api/invitations/*`)
   - Used by: All services (validates JWT)
   - Emits: user.created, user.updated, role.assigned
   - Listens: None (source of truth)

2. **Notification Service** (`/api/notifications/*`)
   - Used by: Frontend (toast), Other services (webhooks)
   - Emits: notification.sent, email.failed
   - Listens: invitation.created, achievement.earned, milestone.completed

3. **Gamification Service** (`/api/gamification/*`)
   - Used by: Learning pages (progress triggers XP)
   - Emits: xp.earned, achievement.unlocked
   - Listens: lesson.completed, quiz.passed, milestone.reached

4. **AI Tutor Service** (`/api/ai-tutor/*`)
   - Used by: Learning pages (sidebar chat)
   - Emits: tutor.session.created, response.generated
   - Listens: lesson.started, user.struggled

5. **Analytics Service** (`/api/analytics/*`)
   - Used by: Admin/instructor dashboards
   - Emits: report.generated
   - Listens: ALL events (aggregates everything)

6. **Assessment Service** (`/api/assessments/*`)
   - Used by: Learning pages (quiz player)
   - Emits: quiz.submitted, grade.assigned
   - Listens: None (external event listeners)

7. **Course/Enrollment Service** (`/api/courses/*`, `/api/enrollments/*`)
   - Used by: Dashboard, learning pages
   - Emits: enrollment.created, progress.updated
   - Listens: course.created, prerequisite.met

---

## SUMMARY

### Auth-Service Provides
✅ **Identity & Access Layer**
- User management (create, read, list, delete)
- Role-based access control (RBAC with 4 tiers)
- Invitation flow (tokenized, expiring)
- Clerk webhook integration
- Audit logging (who/what/when)
- Redis caching for roles/permissions
- Event emission for other services

### Learning-Hub Provides
✅ **User Interface & Interaction**
- Apollo Client GraphQL (fallback pattern)
- Authentication UI (Clerk forms)
- Course browsing & enrollment
- Lesson player (video + markdown + quizzes)
- Instructor grading interface
- Admin panel (users, analytics)
- Analytics dashboards (student + platform)
- Gamification UI (badges, XP, leaderboard)
- Notification center
- Dark mode toggle
- Fully responsive design

### Ready for Migration
✅ **Architecture is production-ready**
- Auth-service: Modular NestJS, fully typed
- Learning-hub: React 18 with hooks + React Query
- Gateway pattern: Transparent microservice fallback
- Event system: Infrastructure in place
- Database: Multi-tenancy support
- Deployment: Docker + Railway configured

### Next Steps
1. Deploy gateway wrapper to Supabase
2. Deploy auth-service to Railway/k8s
3. Deploy notification-service (Phase 1)
4. Migrate monolith calls to gateway-client
5. Add next service per phase...

---

**Generated**: 2024
**Version**: Phase 0-7 Complete Documentation
