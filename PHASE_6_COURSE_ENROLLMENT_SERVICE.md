# Phase 6: Course & Enrollment Service Implementation Guide

**Duration:** 6 weeks (Weeks 25–30)  
**Risk Level:** High (core data, many dependencies)  
**Dependencies:** Phase 0 (API Gateway), Phase 1-5 (all prior services)

---

## 1. Service Structure

```
course-enrollment-service/
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   ├── config/
│   │   └── configuration.ts
│   ├── database/
│   │   └── schema.prisma
│   ├── courses/
│   │   ├── courses.module.ts
│   │   ├── courses.service.ts
│   │   ├── courses.controller.ts
│   │   └── dto/
│   ├── lessons/
│   │   ├── lessons.service.ts
│   │   ├── lessons.controller.ts
│   │   └── dto/
│   ├── enrollments/
│   │   ├── enrollments.module.ts
│   │   ├── enrollments.service.ts
│   │   ├── enrollments.controller.ts
│   │   └── dto/
│   ├── curriculum/
│   │   ├── curriculum.service.ts
│   │   └── curriculum.controller.ts
│   ├── progress/
│   │   ├── progress.service.ts
│   │   ├── progress.controller.ts
│   │   └── tracking.service.ts
│   ├── events/
│   │   ├── events.service.ts
│   │   └── handlers/
│   │       ├── lesson-completed.handler.ts
│   │       ├── assessment-graded.handler.ts
│   │       └── enrollment-created.handler.ts
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

model Course {
  id              String    @id @default(cuid())
  organizationId  String
  
  name            String
  description     String    @db.Text
  category        String    // 'programming', 'design', 'business'
  
  difficulty      String    @default("beginner")  // beginner, intermediate, advanced
  estimatedHours  Int       @default(40)
  
  thumbnail       String?   // URL
  
  lessons         Lesson[]
  assessments     Assessment[]
  enrollments     Enrollment[]
  
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  publishedAt     DateTime?
  
  @@index([organizationId])
  @@index([category])
  @@index([difficulty])
}

model Lesson {
  id              String    @id @default(cuid())
  courseId        String
  course          Course @relation(fields: [courseId], references: [id], onDelete: Cascade)
  
  title           String
  description     String    @db.Text
  
  order           Int       // Lesson sequence
  
  content         String    @db.Text
  videoUrl        String?
  
  duration        Int       @default(0)  // Minutes
  
  prerequisites   String[]  // IDs of prerequisite lessons
  
  completions     LessonCompletion[]
  
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  @@unique([courseId, order])
  @@index([courseId])
}

model Assessment {
  id              String    @id @default(cuid())
  courseId        String
  course          Course @relation(fields: [courseId], references: [id], onDelete: Cascade)
  
  name            String
  description     String    @db.Text
  
  type            String    // 'quiz', 'assignment', 'project'
  totalPoints     Int       @default(100)
  passingScore    Int       @default(60)
  
  // Link to remote assessment service
  assessmentServiceId String  // ID in assessment-service DB
  
  order           Int
  isRequired      Boolean   @default(false)
  
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  @@unique([courseId, order])
  @@index([courseId])
}

model Enrollment {
  id              String    @id @default(cuid())
  userId          String
  organizationId  String
  courseId        String
  course          Course @relation(fields: [courseId], references: [id], onDelete: Cascade)
  
  enrollmentDate  DateTime  @default(now())
  startDate       DateTime?
  completionDate  DateTime?
  
  status          String    @default("active")  // active, completed, dropped, paused
  
  progress        Int       @default(0)  // Percentage
  grade           Float?
  
  // References to other service data
  lessonProgress  Json      // { lesson_id: { completed: true, score: 85 } }
  assessmentProgress Json  // { assessment_id: { attempts: 2, passed: true, score: 92 } }
  
  cohortId        String?
  
  completions     LessonCompletion[]
  attempts        AssessmentAttempt[]
  
  updatedat       DateTime  @updatedAt
  
  @@unique([userId, courseId])
  @@index([userId])
  @@index([organizationId])
  @@index([status])
  @@index([progress])
}

model LessonCompletion {
  id              String    @id @default(cuid())
  enrollmentId    String
  enrollment      Enrollment @relation(fields: [enrollmentId], references: [id], onDelete: Cascade)
  
  lessonId        String
  lesson          Lesson @relation(fields: [lessonId], references: [id], onDelete: Cascade)
  
  completedAt     DateTime  @default(now())
  score           Float?    // For interactive lessons
  
  @@unique([enrollmentId, lessonId])
  @@index([enrollmentId])
  @@index([lessonId])
}

model AssessmentAttempt {
  id              String    @id @default(cuid())
  enrollmentId    String
  enrollment      Enrollment @relation(fields: [enrollmentId], references: [id], onDelete: Cascade)
  
  assessmentId    String
  attemptNumber   Int
  
  score           Float?
  isPassed        Boolean?
  
  submittedAt     DateTime?
  gradedAt        DateTime?
  
  @@unique([enrollmentId, assessmentId, attemptNumber])
  @@index([enrollmentId])
  @@index([assessmentId])
}

model Cohort {
  id              String    @id @default(cuid())
  organizationId  String
  
  name            String
  courseId        String
  
  startDate       DateTime
  endDate         DateTime
  
  createdAt       DateTime  @default(now())
  
  @@index([organizationId])
}
```

---

## 3. Course Service

File: `src/courses/courses.service.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '../database/prisma.service';
import { EventsService } from '../events/events.service';

@Injectable()
export class CoursesService {
  private readonly logger = new Logger(CoursesService.name);

  constructor(
    private prisma: PrismaService,
    private events: EventsService,
  ) {}

  /**
   * Create a course with lessons and assessments
   */
  async createCourse(data: {
    organizationId: string;
    name: string;
    description: string;
    difficulty: string;
    estimatedHours: number;
    lessons: any[];
    assessments: any[];
  }) {
    const course = await this.prisma.course.create({
      data: {
        organizationId: data.organizationId,
        name: data.name,
        description: data.description,
        difficulty: data.difficulty,
        estimatedHours: data.estimatedHours,
        lessons: {
          createMany: {
            data: data.lessons.map((l, idx) => ({
              title: l.title,
              description: l.description,
              order: idx,
              content: l.content,
              videoUrl: l.videoUrl,
              duration: l.duration,
            })),
          },
        },
        assessments: {
          createMany: {
            data: data.assessments.map((a, idx) => ({
              name: a.name,
              type: a.type,
              order: idx,
              totalPoints: a.totalPoints,
              passingScore: a.passingScore,
              assessmentServiceId: a.id,
            })),
          },
        },
      },
      include: { lessons: true, assessments: true },
    });

    this.logger.log(`Course ${course.id} created`);
    return course;
  }

  /**
   * Publish course (make available to students)
   */
  async publishCourse(courseId: string) {
    return this.prisma.course.update({
      where: { id: courseId },
      data: { publishedAt: new Date() },
    });
  }

  /**
   * Get course with all content
   */
  async getCourse(courseId: string) {
    return this.prisma.course.findUnique({
      where: { id: courseId },
      include: {
        lessons: { orderBy: { order: 'asc' } },
        assessments: { orderBy: { order: 'asc' } },
      },
    });
  }

  /**
   * Get all courses for organization
   */
  async getOrgCourses(organizationId: string) {
    return this.prisma.course.findMany({
      where: { organizationId, publishedAt: { not: null } },
      select: {
        id: true,
        name: true,
        description: true,
        difficulty: true,
        estimatedHours: true,
        thumbnail: true,
      },
    });
  }
}
```

---

## 4. Enrollment Service

File: `src/enrollments/enrollments.service.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '../database/prisma.service';
import { EventsService } from '../events/events.service';
import { ProgressService } from '../progress/progress.service';

@Injectable()
export class EnrollmentsService {
  private readonly logger = new Logger(EnrollmentsService.name);

  constructor(
    private prisma: PrismaService,
    private events: EventsService,
    private progress: ProgressService,
  ) {}

  /**
   * Enroll user in course
   */
  async enrollUser(
    userId: string,
    organizationId: string,
    courseId: string,
  ) {
    // Check if already enrolled
    const existing = await this.prisma.enrollment.findUnique({
      where: { userId_courseId: { userId, courseId } },
    });

    if (existing) {
      if (existing.status === 'dropped') {
        // Re-enroll
        return this.prisma.enrollment.update({
          where: { id: existing.id },
          data: { status: 'active', startDate: new Date() },
        });
      }
      return existing;
    }

    const enrollment = await this.prisma.enrollment.create({
      data: {
        userId,
        organizationId,
        courseId,
        startDate: new Date(),
        lessonProgress: {},
        assessmentProgress: {},
      },
    });

    // Emit event
    await this.events.publish('enrollment.created', {
      userId,
      courseId,
      enrollmentId: enrollment.id,
    });

    this.logger.log(`User ${userId} enrolled in course ${courseId}`);
    return enrollment;
  }

  /**
   * Get user's enrollment in course
   */
  async getEnrollment(userId: string, courseId: string) {
    return this.prisma.enrollment.findUnique({
      where: { userId_courseId: { userId, courseId } },
      include: {
        course: { include: { lessons: true, assessments: true } },
      },
    });
  }

  /**
   * Get user's courses
   */
  async getUserCourses(userId: string) {
    return this.prisma.enrollment.findMany({
      where: { userId, status: 'active' },
      include: {
        course: {
          select: {
            id: true,
            name: true,
            estimatedHours: true,
            thumbnail: true,
          },
        },
      },
    });
  }

  /**
   * Mark lesson as completed
   */
  async completeLesson(
    userId: string,
    courseId: string,
    lessonId: string,
    score?: number,
  ) {
    const enrollment = await this.prisma.enrollment.findUnique({
      where: { userId_courseId: { userId, courseId } },
    });

    if (!enrollment) throw new Error('User not enrolled');

    // Record completion
    await this.prisma.lessonCompletion.upsert({
      where: {
        enrollmentId_lessonId: { enrollmentId: enrollment.id, lessonId },
      },
      create: {
        enrollmentId: enrollment.id,
        lessonId,
        score,
      },
      update: { score },
    });

    // Update progress
    await this.progress.updateEnrollmentProgress(enrollment.id);

    // Emit event
    await this.events.publish('lesson.completed', {
      userId,
      courseId,
      lessonId,
      enrollmentId: enrollment.id,
    });

    this.logger.log(`Lesson ${lessonId} completed by user ${userId}`);

    return { success: true };
  }

  /**
   * Complete course
   */
  async completeCourse(enrollmentId: string) {
    const enrollment = await this.prisma.enrollment.findUnique({
      where: { id: enrollmentId },
    });

    if (!enrollment) throw new Error('Enrollment not found');

    const updated = await this.prisma.enrollment.update({
      where: { id: enrollmentId },
      data: {
        status: 'completed',
        completionDate: new Date(),
        progress: 100,
      },
    });

    // Emit event
    await this.events.publish('enrollment.completed', {
      userId: enrollment.userId,
      courseId: enrollment.courseId,
      enrollmentId,
    });

    return updated;
  }
}
```

---

## 5. Progress Tracking

File: `src/progress/progress.service.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../database/prisma.service';

@Injectable()
export class ProgressService {
  constructor(private prisma: PrismaService) {}

  /**
   * Update enrollment progress (called after lesson/assessment completion)
   */
  async updateEnrollmentProgress(enrollmentId: string) {
    const enrollment = await this.prisma.enrollment.findUnique({
      where: { id: enrollmentId },
      include: {
        course: { include: { lessons: true, assessments: true } },
        completions: true,
        attempts: true,
      },
    });

    if (!enrollment) return;

    const totalLessons = enrollment.course.lessons.length;
    const completedLessons = enrollment.completions.length;

    const requiredAssessments = enrollment.course.assessments.filter(
      (a) => a.isRequired,
    ).length;
    const completedAssessments = enrollment.attempts.filter(
      (a) => a.isPassed,
    ).length;

    // Calculate progress
    const progress = Math.floor(
      ((completedLessons + completedAssessments) /
        (totalLessons + requiredAssessments)) *
        100,
    );

    // Calculate estimated time remaining
    const estimatedTotalHours = enrollment.course.estimatedHours;
    const completedHours = (progress / 100) * estimatedTotalHours;
    const remainingHours = estimatedTotalHours - completedHours;

    return this.prisma.enrollment.update({
      where: { id: enrollmentId },
      data: {
        progress: Math.min(progress, 100),
      },
    });
  }

  /**
   * Check if user can access lesson (prerequisites met)
   */
  async canAccessLesson(
    enrollmentId: string,
    lessonId: string,
  ): Promise<boolean> {
    const lesson = await this.prisma.lesson.findUnique({
      where: { id: lessonId },
    });

    if (!lesson || lesson.prerequisites.length === 0) return true;

    // Check if all prerequisites completed
    const enrollment = await this.prisma.enrollment.findUnique({
      where: { id: enrollmentId },
    });

    for (const prereqId of lesson.prerequisites) {
      const completion = await this.prisma.lessonCompletion.findUnique({
        where: {
          enrollmentId_lessonId: { enrollmentId, lessonId: prereqId },
        },
      });

      if (!completion) return false;
    }

    return true;
  }
}
```

---

## 6. Event Handlers

File: `src/events/handlers/assessment-graded.handler.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { EventsService } from '../events.service';
import { EnrollmentsService } from '../../enrollments/enrollments.service';

@Injectable()
export class AssessmentGradedHandler {
  private readonly logger = new Logger(AssessmentGradedHandler.name);

  constructor(
    private events: EventsService,
    private enrollments: EnrollmentsService,
  ) {
    this.setupSubscription();
  }

  private setupSubscription() {
    this.events.subscribe('assessment.graded', async (data) => {
      const { userId, assessmentId, score, isPassed } = data;

      // Get enrollment for this user's course
      // Record assessment attempt in course-service
      
      this.logger.log(`Recording assessment attempt for user ${userId}`);

      // Check if course completion logic needed
      // (all required assessments passed + lessons completed)
    });
  }
}
```

---

## 7. API Endpoints

File: `src/enrollments/enrollments.controller.ts`

```typescript
import {
  Controller,
  Post,
  Get,
  Body,
  Param,
  UseGuards,
} from '@nestjs/common';
import { EnrollmentsService } from './enrollments.service';
import { JwtGuard } from '../common/guards/jwt.guard';
import { CurrentUser } from '../common/decorators/current-user.decorator';

@Controller('api/enrollments')
@UseGuards(JwtGuard)
export class EnrollmentsController {
  constructor(private enrollments: EnrollmentsService) {}

  /**
   * POST /api/enrollments
   * Enroll in course
   */
  @Post()
  async enrollUser(
    @Body() dto: { courseId: string },
    @CurrentUser() userId: string,
    @CurrentUser('orgId') organizationId: string,
  ) {
    return this.enrollments.enrollUser(userId, organizationId, dto.courseId);
  }

  /**
   * GET /api/enrollments/my-courses
   * Get user's courses
   */
  @Get('my-courses')
  async getUserCourses(@CurrentUser() userId: string) {
    return this.enrollments.getUserCourses(userId);
  }

  /**
   * GET /api/enrollments/:courseId
   * Get enrollment status
   */
  @Get(':courseId')
  async getEnrollment(
    @Param('courseId') courseId: string,
    @CurrentUser() userId: string,
  ) {
    return this.enrollments.getEnrollment(userId, courseId);
  }

  /**
   * POST /api/enrollments/:courseId/lessons/:lessonId/complete
   * Mark lesson complete
   */
  @Post(':courseId/lessons/:lessonId/complete')
  async completeLesson(
    @Param('courseId') courseId: string,
    @Param('lessonId') lessonId: string,
    @Body() dto: { score?: number },
    @CurrentUser() userId: string,
  ) {
    return this.enrollments.completeLesson(userId, courseId, lessonId, dto.score);
  }
}
```

---

## 8. Wrapper Functions

File: `lumina-learning-hub/supabase/functions/enroll-course/index.ts`

```typescript
import { handleWrapperRequest } from "../_shared/gateway-wrapper.ts";
import { corsHeaders } from "../_shared/cors.ts";

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }

  return handleWrapperRequest(req, {
    microserviceUrl:
      Deno.env.get("COURSE_ENROLLMENT_SERVICE_URL") || "http://localhost:3006",
    microservicePath: "/api/enrollments",
    fallbackRpcName: "enroll_course_legacy",
    timeoutMs: 5000,
    enabled: Deno.env.get("WRAPPER_ENABLE_COURSES") !== "false",
  });
});
```

---

## 9. Docker Setup

File: `docker-compose.yml`

```yaml
version: '3.8'

services:
  course-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: course
      POSTGRES_PASSWORD: password
      POSTGRES_DB: course
    ports:
      - "5438:5432"
    volumes:
      - course-db-data:/var/lib/postgresql/data

  course-enrollment-service:
    build: .
    ports:
      - "3006:3006"
    environment:
      DATABASE_URL: postgresql://course:password@course-db:5432/course
      NATS_URL: nats://nats:4222
      # Service APIs
      ASSESSMENT_SERVICE_URL: http://localhost:3005
      NOTIFICATION_SERVICE_URL: http://localhost:3001
    depends_on:
      - course-db

volumes:
  course-db-data:
```

---

## 10. Deployment Checklist

- [ ] PostgreSQL configured
- [ ] Courses + lessons + assessments created
- [ ] Enrollment flow tested
- [ ] Progress tracking calculated correctly
- [ ] Pre-requisite logic working
- [ ] Event handlers subscribed
- [ ] Wrapper functions deployed
- [ ] Fallback RPCs created
- [ ] Canary traffic routed (10% → 50% → 100%)
- [ ] Data consistency checks (lessons/assessments match between services)

## Success Criteria

✅ Enrollment completes in <1 second  
✅ Course loads with all linked content <500ms  
✅ Progress updates within 2 seconds of lesson completion  
✅ 100% accurate data consistency across services  
✅ Zero data loss during migration from Supabase RPC

**Phase 6 is the backbone. All other services depend on course/enrollment data. Highest-risk phase.**
