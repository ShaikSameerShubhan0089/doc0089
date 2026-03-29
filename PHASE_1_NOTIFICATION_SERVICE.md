# Phase 1: Notification Service Implementation Guide

**Duration:** 4 weeks  
**Risk Level:** Low  
**Dependencies:** Phase 0 (API Gateway + wrappers)

---

## 1. Service Structure

Create a new NestJS service with the following structure:

```
notification-service/
├── src/
│   ├── main.ts                    # Service entry
│   ├── app.module.ts              # Main module
│   ├── config/
│   │   ├── config.module.ts
│   │   └── configuration.ts       # Env vars + validation
│   ├── database/
│   │   ├── prisma.service.ts
│   │   ├── migrations/
│   │   └── schema.prisma          # DB schema
│   ├── notifications/
│   │   ├── notifications.module.ts
│   │   ├── notifications.service.ts
│   │   ├── notifications.controller.ts
│   │   ├── dto/
│   │   │   ├── send-invitation.dto.ts
│   │   │   ├── create-notification.dto.ts
│   │   │   └── mark-as-read.dto.ts
│   │   └── notifications.service.spec.ts
│   ├── preferences/
│   │   ├── preferences.module.ts
│   │   ├── preferences.service.ts
│   │   ├── preferences.controller.ts
│   │   └── dto/
│   │       └── update-preferences.dto.ts
│   ├── email/
│   │   ├── email.service.ts       # Resend integration
│   │   ├── templates/
│   │   │   ├── invitation.ts
│   │   │   ├── assignment.ts
│   │   │   └── achievement.ts
│   ├── events/
│   │   ├── events.module.ts
│   │   ├── events.service.ts      # NATS/RabbitMQ integration
│   │   └── handlers/
│   │       ├── achievement-unlocked.handler.ts
│   │       ├── assessment-graded.handler.ts
│   │       └── user-created.handler.ts
│   └── common/
│       ├── decorators/
│       │   └── current-user.decorator.ts
│       ├── filters/
│       │   └── all-exceptions.filter.ts
│       └── guards/
│           └── jwt.guard.ts
├── prisma/
│   └── schema.prisma              # Database schema
├── docker-compose.yml
├── Dockerfile
├── package.json
└── tsconfig.json
```

---

## 2. Database Schema

File: `prisma/schema.prisma`

```prisma
// Notification Service Database Schema

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model Invitation {
  id            String    @id @default(cuid())
  email         String
  organizationId String
  role          String    // 'instructor', 'student', 'tenant_admin'
  
  invitedBy     String    // user_id from identity service
  token         String    @unique
  
  acceptedAt    DateTime?
  expiresAt     DateTime
  
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  @@index([email])
  @@index([organizationId])
  @@index([expiresAt])
}

model Notification {
  id            String    @id @default(cuid())
  userId        String    // from identity service
  organizationId String
  
  title         String
  message       String
  type          String    // 'achievement', 'assignment', 'enrollment', etc.
  relatedId     String?   // achievement_id, assignment_id, etc.
  
  isRead        Boolean   @default(false)
  readAt        DateTime?
  
  createdAt     DateTime  @default(now())
  
  @@index([userId])
  @@index([organizationId])
  @@index([isRead])
  @@index([createdAt])
}

model NotificationPreference {
  id            String    @id @default(cuid())
  userId        String    @unique
  organizationId String
  
  // Channel preferences
  emailEnabled  Boolean   @default(true)
  inAppEnabled  Boolean   @default(true)
  
  // Category preferences
  achievementEmails   Boolean @default(true)
  assignmentEmails    Boolean @default(true)
  enrollmentEmails    Boolean @default(true)
  summaryFrequency    String  @default("daily") // 'daily', 'weekly', 'never'
  
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  
  @@index([organizationId])
}

model ScheduledNudge {
  id            String    @id @default(cuid())
  userId        String
  organizationId String
  
  title         String
  message       String
  scheduledFor  DateTime
  
  sentAt        DateTime?
  
  createdAt     DateTime  @default(now())
  
  @@index([userId])
  @@index([organizationId])
  @@index([scheduledFor])
  @@index([sentAt])
}

model SupportTicket {
  id            String    @id @default(cuid())
  organizationId String
  userId        String
  
  subject       String
  description   String
  status        String    @default("open") // 'open', 'in-progress', 'resolved', 'closed'
  priority      String    @default("medium") // 'low', 'medium', 'high', 'urgent'
  
  resolvedAt    DateTime?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  
  @@index([organizationId])
  @@index([status])
  @@index([priority])
}

model EmailLog {
  id            String    @id @default(cuid())
  recipientEmail String
  subject       String
  type          String    // 'invitation', 'assignment', 'achievement'
  relatedId     String?   // invitation_id, etc.
  
  status        String    @default("sent") // 'sent', 'bounced', 'failed', 'delivered'
  errorMessage  String?
  
  sentAt        DateTime  @default(now())
  
  @@index([recipientEmail])
  @@index([status])
  @@index([sentAt])
}
```

---

## 3. API Endpoints

### Notifications

```typescript
// POST /api/notifications/send-invitation
// Body: { email, organizationId, inviterName, inviteeRole }
// Response: { invitationId, email, status, expiresAt }

// GET /api/notifications
// Query: userId, organizationId, limit, offset
// Response: { notifications: Notification[], total: number }

// GET /api/notifications/:id
// Response: Notification

// PATCH /api/notifications/:id/read
// Response: { success: boolean }

// DELETE /api/notifications/:id
// Response: { success: boolean }

// POST /api/notifications/send-ticket
// Body: { subject, description, organizationId }
// Response: { ticketId, status, createdAt }
```

### Preferences

```typescript
// GET /api/preferences/:userId
// Response: NotificationPreference

// PATCH /api/preferences/:userId
// Body: { emailEnabled, inAppEnabled, ... }
// Response: NotificationPreference
```

### Health

```typescript
// GET /health
// Response: { status: "ok" }

// GET /ready
// Response: { status: "ready" } (or 503 if not ready)
```

---

## 4. Event Integration

### Subscribe to Events

```typescript
// File: src/events/handlers/achievement-unlocked.handler.ts

import { Injectable } from '@nestjs/common';
import { EventsService } from '../events.service';
import { NotificationsService } from '../../notifications/notifications.service';

@Injectable()
export class AchievementUnlockedHandler {
  constructor(
    private events: EventsService,
    private notifications: NotificationsService,
  ) {
    this.setupSubscription();
  }

  private setupSubscription() {
    this.events.subscribe('achievement.unlocked', async (data) => {
      const { userId, achievementName, organizationId } = data;

      // Create in-app notification
      await this.notifications.create({
        userId,
        organizationId,
        title: `🏆 New Achievement Unlocked!`,
        message: `Congratulations! You earned "${achievementName}"`,
        type: 'achievement',
        relatedId: data.achievementId,
      });

      // Send email if preference enabled
      const prefs = await this.notifications.getPreferences(userId);
      if (prefs.achievementEmails) {
        await this.notifications.sendEmail('achievement', {
          userId,
          organizationId,
          achievementName,
        });
      }
    });
  }
}
```

### Publish Events

```typescript
// After successful invitation sent
await this.events.publish('invitation.created', {
  invitationId: invitation.id,
  email: invitation.email,
  organizationId: invitation.organizationId,
  role: invitation.role,
  timestamp: new Date().toISOString(),
});
```

---

## 5. Email Service Integration (Resend)

File: `src/email/email.service.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Resend } from 'resend';
import { ConfigService } from '@nestjs/config';
import { PrismaService } from '../database/prisma.service';

@Injectable()
export class EmailService {
  private readonly resend: Resend;
  private readonly logger = new Logger(EmailService.name);
  private readonly fromEmail: string;

  constructor(
    private config: ConfigService,
    private prisma: PrismaService,
  ) {
    this.resend = new Resend(config.get('RESEND_API_KEY'));
    this.fromEmail = 'noreply@pathwisse.com';
  }

  async sendInvitation(email: string, data: {
    inviterName: string;
    organizationId: string;
    invitationLink: string;
  }) {
    try {
      const response = await this.resend.emails.send({
        from: this.fromEmail,
        to: email,
        subject: `You're invited to ${data.organizationId}`,
        html: this.invitationTemplate(data),
      });

      // Log email send
      await this.prisma.emailLog.create({
        data: {
          recipientEmail: email,
          subject: `You're invited to ${data.organizationId}`,
          type: 'invitation',
          status: 'sent',
        },
      });

      this.logger.log(`Invitation email sent to ${email}`);
      return response;
    } catch (error) {
      this.logger.error(`Failed to send invitation email to ${email}:`, error);
      throw error;
    }
  }

  private invitationTemplate(data: any): string {
    return `
      <h2>You're Invited!</h2>
      <p>Hello!</p>
      <p>${data.inviterName} has invited you to join ${data.organizationId}.</p>
      <a href="${data.invitationLink}">Accept Invitation</a>
      <p>This invitation expires in 7 days.</p>
    `;
  }
}
```

---

## 6. Testing

File: `src/notifications/notifications.service.spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { NotificationsService } from './notifications.service';
import { PrismaService } from '../database/prisma.service';
import { EventsService } from '../events/events.service';

describe('NotificationsService', () => {
  let service: NotificationsService;
  let prisma: PrismaService;
  let events: EventsService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        NotificationsService,
        {
          provide: PrismaService,
          useValue: {
            notification: {
              create: jest.fn(),
              findMany: jest.fn(),
            },
          },
        },
        {
          provide: EventsService,
          useValue: {
            publish: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<NotificationsService>(NotificationsService);
    prisma = module.get<PrismaService>(PrismaService);
    events = module.get<EventsService>(EventsService);
  });

  it('should create a notification', async () => {
    const notif = {
      userId: 'user123',
      organizationId: 'org123',
      title: 'Test',
      message: 'Test message',
      type: 'test',
    };

    await service.create(notif);

    expect(prisma.notification.create).toHaveBeenCalledWith({
      data: notif,
    });
  });

  it('should emit event after creating notification', async () => {
    // Test that events are published correctly
  });
});
```

---

## 7. Deployment Checklist

- [ ] Service runs on port 3001
- [ ] Database migrations complete
- [ ] Health endpoints working (`/health`, `/ready`)
- [ ] JWT middleware validates tokens from gateway
- [ ] Resend API key configured
- [ ] Event bus (NATS/RabbitMQ) connected
- [ ] Wrapper function deployed and working
- [ ] Fallback RPC created in Supabase
- [ ] Canary traffic routed (10% → 50% → 100%)
- [ ] Monitoring dashboard created
- [ ] Incident runbook documented

---

## 8. Running Locally

```bash
# Start PostgreSQL
docker run -d --name notification-db \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=notification \
  -p 5433:5432 \
  postgres:15

# Setup env
cp .env.example .env
# Edit .env with local DB connection

# Install dependencies
npm install

# Run migrations
npx prisma migrate dev

# Start service
npm run start:dev

# Test wrapper
curl -X POST http://localhost:3001/api/notifications/send-invitation \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","organizationId":"org123"}'
```

---

## 9. Observability Setup

Integrate OpenTelemetry:

```typescript
// src/main.ts
import { NodeTracerProvider } from '@opentelemetry/node';
import { registerInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { JaegerExporter } from '@opentelemetry/exporter-jaeger';

const jaegerExporter = new JaegerExporter({
  endpoint: 'http://localhost:14268/api/traces',
});

const tracerProvider = new NodeTracerProvider();
tracerProvider.addSpanProcessor(
  new BatchSpanProcessor(jaegerExporter),
);

tracerProvider.register();
registerInstrumentations();
```

This enables distributed tracing across all services.

---

## 10. Next Steps

1. **Week 3**: Scaffold service, create DB schema
2. **Week 4**: Implement endpoints, email service
3. **Week 5**: Event integration, testing
4. **Week 6**: Deploy, canary, monitor

Refer to `MIGRATION_CHECKLIST.md` (Phase 1 section) for full tracking.
