# Phase 4: Analytics Service Implementation Guide

**Duration:** 4 weeks (Weeks 16–19)  
**Risk Level:** Low  
**Dependencies:** Phase 0 (API Gateway), Phase 1 (Notification + events), Phase 2 (Gamification), Phase 3 (AI Tutor)

---

## 1. Service Structure

```
analytics-service/
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   ├── config/
│   │   ├── config.module.ts
│   │   └── configuration.ts
│   ├── database/
│   │   ├── prisma.service.ts
│   │   └── schema.prisma
│   ├── events/
│   │   ├── events.module.ts
│   │   ├── events.service.ts
│   │   └── handlers/
│   │       ├── lesson-completed.handler.ts
│   │       ├── assessment-graded.handler.ts
│   │       ├── xp-awarded.handler.ts
│   │       └── challenge-completed.handler.ts
│   ├── analytics/
│   │   ├── analytics.module.ts
│   │   ├── analytics.service.ts
│   │   ├── analytics.controller.ts
│   │   └── dto/
│   ├── dashboards/
│   │   ├── dashboards.service.ts
│   │   ├── dashboards.controller.ts
│   │   └── dto/
│   ├── reports/
│   │   ├── reports.service.ts
│   │   ├── reports.controller.ts
│   │   └── dto/
│   ├── timeseries/
│   │   ├── timeseries.module.ts
│   │   ├── timeseries.service.ts (ClickHouse integration)
│   │   └── queries.ts
│   ├── export/
│   │   ├── export.service.ts
│   │   └── export.controller.ts
│   └── common/
│       ├── decorators/
│       └── guards/
├── prisma/
└── docker-compose.yml
```

---

## 2. Database Schema

File: `prisma/schema.prisma`

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model AnalyticsEvent {
  id              String    @id @default(cuid())
  userId          String
  organizationId  String
  
  eventType       String    // 'lesson_completed', 'assessment_graded', 'xp_awarded'
  eventData       Json      // Raw event data
  
  timestamp       DateTime  @default(now())
  
  @@index([userId])
  @@index([organizationId])
  @@index([eventType])
  @@index([timestamp])
}

model UserLearningMetrics {
  id              String    @id @default(cuid())
  userId          String    @unique
  organizationId  String
  
  lessonsCompleted    Int       @default(0)
  assessmentsPassed   Int       @default(0)
  assessmentsFailed   Int       @default(0)
  averageScore        Float     @default(0)
  
  totalXp             Int       @default(0)
  level               Int       @default(1)
  
  streakDays          Int       @default(0)
  totalLearningHours  Float     @default(0)
  
  lastActivityAt      DateTime?
  
  createdAt           DateTime  @default(now())
  updatedAt           DateTime  @updatedAt
  
  @@index([organizationId])
  @@index([lessonsCompleted])
  @@index([averageScore])
}

model CourseLearningPath {
  id              String    @id @default(cuid())
  userId          String
  organizationId  String
  courseId        String
  
  lessonProgress  Json      // { lesson_id: { completed: true, score: 85 } }
  assessmentProgress Json  // { assessment_id: { attempts: 2, finalScore: 92 } }
  
  startedAt       DateTime  @default(now())
  completedAt     DateTime?
  
  estimatedHoursRemaining Float @default(0)
  completionPercentage    Int   @default(0)
  
  @@unique([userId, organizationId, courseId])
  @@index([userId])
  @@index([completionPercentage])
}

model CohortAnalytics {
  id              String    @id @default(cuid())
  organizationId  String
  cohortId        String
  
  totalUsers      Int       @default(0)
  activeUsers     Int       @default(0)
  averageProgress Float     @default(0)
  averageScore    Float     @default(0)
  
  dropoutRate     Float     @default(0)
  completionRate  Float     @default(0)
  
  updatedAt       DateTime  @default(now())
  
  @@unique([organizationId, cohortId])
  @@index([organizationId])
}

model LessonAnalytics {
  id              String    @id @default(cuid())
  organizationId  String
  lessonId        String
  
  totalCompletions    Int       @default(0)
  averageCompletionTime Float   @default(0)
  averageScore        Float     @default(0)
  
  completionRate      Float     @default(0)
  abandonmentRate     Float     @default(0)
  avgTutorInteractions Int      @default(0)
  
  updatedAt           DateTime  @default(now())
  
  @@unique([organizationId, lessonId])
  @@index([organizationId])
  @@index([completionRate])
  @@index([abandonmentRate])
}

model AssessmentAnalytics {
  id              String    @id @default(cuid())
  organizationId  String
  assessmentId    String
  
  totalAttempts       Int       @default(0)
  passRate            Float     @default(0)
  averageScore        Float     @default(0)
  averageAttempts     Float     @default(0)
  
  difficultQuestions  String[]  // Question IDs that are hard
  
  updatedAt           DateTime  @default(now())
  
  @@unique([organizationId, assessmentId])
  @@index([organizationId])
  @@index([passRate])
}

model TimeSeriesDataPoint {
  id              String    @id @default(cuid())
  organizationId  String
  
  metric          String    // 'active_users', 'lessons_completed', 'avg_score'
  value           Float
  
  timestamp       DateTime
  
  @@index([organizationId])
  @@index([metric])
  @@index([timestamp])
}
```

---

## 3. Event Aggregation Service

File: `src/analytics/analytics.service.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '../database/prisma.service';
import { EventsService } from '../events/events.service';

@Injectable()
export class AnalyticsService {
  private readonly logger = new Logger(AnalyticsService.name);

  constructor(
    private prisma: PrismaService,
    private events: EventsService,
  ) {}

  /**
   * Track raw event for later analysis
   */
  async trackEvent(
    userId: string,
    organizationId: string,
    eventType: string,
    eventData: any,
  ) {
    return this.prisma.analyticsEvent.create({
      data: {
        userId,
        organizationId,
        eventType,
        eventData,
      },
    });
  }

  /**
   * Update user learning metrics after activity
   */
  async updateUserMetrics(
    userId: string,
    organizationId: string,
    updates: {
      lessonsCompleted?: number;
      assessmentsPassed?: number;
      assessmentsFailed?: number;
      xpAwarded?: number;
      hoursLearned?: number;
    },
  ) {
    const user = await this.prisma.userLearningMetrics.upsert({
      where: { userId },
      create: {
        userId,
        organizationId,
        lessonsCompleted: updates.lessonsCompleted || 0,
        assessmentsPassed: updates.assessmentsPassed || 0,
        assessmentsFailed: updates.assessmentsFailed || 0,
        totalXp: updates.xpAwarded || 0,
        totalLearningHours: updates.hoursLearned || 0,
      },
      update: {
        lessonsCompleted: { increment: updates.lessonsCompleted || 0 },
        assessmentsPassed: { increment: updates.assessmentsPassed || 0 },
        assessmentsFailed: { increment: updates.assessmentsFailed || 0 },
        totalXp: { increment: updates.xpAwarded || 0 },
        totalLearningHours: { increment: updates.hoursLearned || 0 },
        lastActivityAt: new Date(),
      },
    });

    // Recalculate derived metrics
    user.averageScore =
      user.assessmentsPassed + user.assessmentsFailed > 0
        ? (user.assessmentsPassed * 100) /
          (user.assessmentsPassed + user.assessmentsFailed)
        : 0;

    user.level = Math.floor(user.totalXp / 1000) + 1;

    await this.prisma.userLearningMetrics.update({
      where: { userId },
      data: {
        averageScore: user.averageScore,
        level: user.level,
      },
    });

    return user;
  }

  /**
   * Get user's learning dashboard
   */
  async getUserDashboard(userId: string) {
    const metrics = await this.prisma.userLearningMetrics.findUnique({
      where: { userId },
    });

    const recentEvents = await this.prisma.analyticsEvent.findMany({
      where: { userId },
      orderBy: { timestamp: 'desc' },
      take: 10,
    });

    const activeCourses = await this.prisma.courseLearningPath.findMany({
      where: { userId, completedAt: null },
    });

    return {
      metrics,
      recentActivity: recentEvents,
      activeCourses,
    };
  }

  /**
   * Get organization-wide analytics
   */
  async getOrgAnalytics(organizationId: string) {
    const totalUsers = await this.prisma.userLearningMetrics.count({
      where: { organizationId },
    });

    const activeUsers = await this.prisma.userLearningMetrics.count({
      where: {
        organizationId,
        lastActivityAt: {
          gte: new Date(Date.now() - 24 * 60 * 60 * 1000), // Last 24 hours
        },
      },
    });

    const avgScore = await this.prisma.userLearningMetrics.aggregate({
      where: { organizationId },
      _avg: { averageScore: true },
    });

    const avgLessonsCompleted =
      await this.prisma.userLearningMetrics.aggregate({
        where: { organizationId },
        _avg: { lessonsCompleted: true },
      });

    return {
      totalUsers,
      activeUsers: {
        count: activeUsers,
        percentage: ((activeUsers / totalUsers) * 100).toFixed(2),
      },
      avgScore: avgScore._avg.averageScore,
      avgLessonsCompleted: avgLessonsCompleted._avg.lessonsCompleted,
      engagementRate: ((activeUsers / totalUsers) * 100).toFixed(2),
    };
  }

  /**
   * Get cohort analytics (for admin dashboards)
   */
  async getCohortAnalytics(organizationId: string, cohortId: string) {
    let cohort = await this.prisma.cohortAnalytics.findUnique({
      where: {
        organizationId_cohortId: { organizationId, cohortId },
      },
    });

    if (!cohort) {
      cohort = await this.prisma.cohortAnalytics.create({
        data: { organizationId, cohortId, totalUsers: 0 },
      });
    }

    return cohort;
  }
}
```

---

## 4. Dashboard Endpoints

File: `src/dashboards/dashboards.controller.ts`

```typescript
import { Controller, Get, UseGuards } from '@nestjs/common';
import { AnalyticsService } from '../analytics/analytics.service';
import { JwtGuard } from '../common/guards/jwt.guard';
import { CurrentUser } from '../common/decorators/current-user.decorator';

@Controller('api/analytics')
@UseGuards(JwtGuard)
export class DashboardsController {
  constructor(private analytics: AnalyticsService) {}

  /**
   * GET /api/analytics/dashboard/user
   * Get user's personal learning dashboard
   */
  @Get('dashboard/user')
  async getUserDashboard(@CurrentUser() userId: string) {
    return this.analytics.getUserDashboard(userId);
  }

  /**
   * GET /api/analytics/dashboard/org
   * Get organization analytics (instructor/admin only)
   */
  @Get('dashboard/org')
  async getOrgDashboard(@CurrentUser('orgId') organizationId: string) {
    return this.analytics.getOrgAnalytics(organizationId);
  }

  /**
   * GET /api/analytics/dashboard/cohort/:cohortId
   * Get cohort analytics
   */
  @Get('dashboard/cohort/:cohortId')
  async getCohortDashboard(@Param('cohortId') cohortId: string) {
    return this.analytics.getCohortAnalytics('org_placeholder', cohortId);
  }
}
```

---

## 5. Event Handlers

File: `src/events/handlers/lesson-completed.handler.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { EventsService } from '../events.service';
import { AnalyticsService } from '../../analytics/analytics.service';

@Injectable()
export class LessonCompletedHandler {
  private readonly logger = new Logger(LessonCompletedHandler.name);

  constructor(
    private events: EventsService,
    private analytics: AnalyticsService,
  ) {
    this.setupSubscription();
  }

  private setupSubscription() {
    this.events.subscribe('lesson.completed', async (data) => {
      const { userId, organizationId, lessonId, completionTime } = data;

      this.logger.log(`Recording lesson completion for user ${userId}`);

      // Track event
      await this.analytics.trackEvent(
        userId,
        organizationId,
        'lesson_completed',
        { lessonId, completionTime },
      );

      // Update user metrics
      await this.analytics.updateUserMetrics(userId, organizationId, {
        lessonsCompleted: 1,
        hoursLearned: completionTime / 3600,
      });

      // Update lesson analytics
      // (aggregated in batch job)

      // Publish to next service (if needed)
      await this.events.publish('analytics.lesson_recorded', { userId, lessonId });
    });
  }
}
```

---

## 6. Time Series: ClickHouse Integration

File: `src/timeseries/timeseries.service.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Client } from '@clickhouse/client';

interface TimeSeriesQuery {
  metric: string;
  organizationId: string;
  startTime: Date;
  endTime: Date;
  granularity: 'hour' | 'day' | 'week' | 'month';
}

@Injectable()
export class TimeSeriesService {
  private readonly logger = new Logger(TimeSeriesService.name);
  private client: Client;

  constructor() {
    this.client = new Client({
      host: process.env.CLICKHOUSE_HOST || 'localhost',
      port: parseInt(process.env.CLICKHOUSE_PORT || '8123'),
      database: 'analytics',
    });
  }

  /**
   * Insert time series data point
   */
  async insertDataPoint(
    organizationId: string,
    metric: string,
    value: number,
    timestamp: Date,
  ) {
    try {
      await this.client.insert({
        table: 'timeseries_metrics',
        values: [
          {
            organization_id: organizationId,
            metric,
            value,
            timestamp,
          },
        ],
      });
    } catch (error) {
      this.logger.error(`Failed to insert time series data:`, error);
    }
  }

  /**
   * Query time series data
   */
  async queryMetric(query: TimeSeriesQuery) {
    try {
      const sql = `
        SELECT
          toStartOfInterval(timestamp, INTERVAL 1 ${query.granularity.toUpperCase()}) as time,
          avg(value) as avg_value,
          max(value) as max_value,
          min(value) as min_value,
          count() as datapoints
        FROM timeseries_metrics
        WHERE organization_id = '${query.organizationId}'
          AND metric = '${query.metric}'
          AND timestamp >= '${query.startTime.toISOString()}'
          AND timestamp <= '${query.endTime.toISOString()}'
        GROUP BY time
        ORDER BY time ASC
      `;

      const result = await this.client.query({
        query: sql,
        format: 'JSONEachRow',
      });

      return result.json();
    } catch (error) {
      this.logger.error(`Failed to query time series:`, error);
      throw error;
    }
  }

  /**
   * Get trending metrics
   */
  async getTrends(organizationId: string, metric: string, days: number = 7) {
    const endTime = new Date();
    const startTime = new Date(Date.now() - days * 24 * 60 * 60 * 1000);

    return this.queryMetric({
      organizationId,
      metric,
      startTime,
      endTime,
      granularity: 'day',
    });
  }
}
```

---

## 7. Reports & Exports

File: `src/reports/reports.service.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '../database/prisma.service';
import { createObjectCsvWriter } from 'csv-writer';

@Injectable()
export class ReportsService {
  private readonly logger = new Logger(ReportsService.name);

  constructor(private prisma: PrismaService) {}

  /**
   * Generate CSV export of user metrics
   */
  async exportUserMetrics(organizationId: string): Promise<string> {
    const users = await this.prisma.userLearningMetrics.findMany({
      where: { organizationId },
    });

    const csvWriter = createObjectCsvWriter({
      path: `/tmp/user-metrics-${Date.now()}.csv`,
      header: [
        { id: 'userId', title: 'User ID' },
        { id: 'lessonsCompleted', title: 'Lessons Completed' },
        { id: 'assessmentsPassed', title: 'Assessments Passed' },
        { id: 'averageScore', title: 'Average Score' },
        { id: 'totalXp', title: 'Total XP' },
        { id: 'level', title: 'Level' },
      ],
    });

    await csvWriter.writeRecords(users);
    return csvWriter.path;
  }

  /**
   * Generate custom report
   */
  async generateReport(query: {
    organizationId: string;
    startDate: Date;
    endDate: Date;
    metrics: string[];
  }) {
    const events = await this.prisma.analyticsEvent.findMany({
      where: {
        organizationId: query.organizationId,
        timestamp: {
          gte: query.startDate,
          lte: query.endDate,
        },
      },
      orderBy: { timestamp: 'desc' },
    });

    // Aggregate events
    const aggregated = {};
    for (const event of events) {
      if (!aggregated[event.eventType]) {
        aggregated[event.eventType] = 0;
      }
      aggregated[event.eventType]++;
    }

    return {
      period: { start: query.startDate, end: query.endDate },
      totalEvents: events.length,
      eventBreakdown: aggregated,
      generatedAt: new Date(),
    };
  }
}
```

---

## 8. Wrapper Functions

File: `lumina-learning-hub/supabase/functions/get-user-analytics/index.ts`

```typescript
import { handleWrapperRequest } from "../_shared/gateway-wrapper.ts";
import { corsHeaders } from "../_shared/cors.ts";

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }

  return handleWrapperRequest(req, {
    microserviceUrl:
      Deno.env.get("ANALYTICS_SERVICE_URL") || "http://localhost:3004",
    microservicePath: "/api/analytics/dashboard/user",
    fallbackRpcName: "get_user_analytics_legacy",
    timeoutMs: 3000,
    enabled: Deno.env.get("WRAPPER_ENABLE_ANALYTICS") !== "false",
  });
});
```

---

## 9. Docker Setup

File: `docker-compose.yml`

```yaml
version: '3.8'

services:
  analytics-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: analytics
      POSTGRES_PASSWORD: password
      POSTGRES_DB: analytics
    ports:
      - "5436:5432"
    volumes:
      - analytics-db-data:/var/lib/postgresql/data

  clickhouse:
    image: clickhouse/clickhouse-server:latest
    ports:
      - "8123:8123"
    environment:
      CLICKHOUSE_DB: analytics
    volumes:
      - clickhouse-data:/var/lib/clickhouse

  analytics-service:
    build: .
    ports:
      - "3004:3004"
    environment:
      DATABASE_URL: postgresql://analytics:password@analytics-db:5432/analytics
      CLICKHOUSE_HOST: clickhouse
      NATS_URL: nats://nats:4222
    depends_on:
      - analytics-db
      - clickhouse

volumes:
  analytics-db-data:
  clickhouse-data:
```

---

## 10. Deployment Checklist

- [ ] PostgreSQL + ClickHouse configured
- [ ] Database migrations complete
- [ ] Event handlers subscribed
- [ ] Dashboard endpoints working
- [ ] Time series data collection started
- [ ] Wrapper functions deployed
- [ ] Monitoring queries configured
- [ ] Canary traffic routed
- [ ] Historical data ingested
- [ ] Reports tested

## Success Criteria

✅ Metrics updated within 2 seconds of events  
✅ Dashboards load in <500ms  
✅ Time series queries return in <1 second  
✅ Historical data (30 days) fully ingested

**Phase 4 aggregates all learner activity. Essential baseline before assessment/course services.**
