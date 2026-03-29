# Phase 5: Assessment Service Implementation Guide

**Duration:** 5 weeks (Weeks 20–24)  
**Risk Level:** Medium (complex grading logic)  
**Dependencies:** Phase 0 (API Gateway), Phase 1 (Notification), Phase 4 (Analytics)

---

## 1. Service Structure

```
assessment-service/
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   ├── config/
│   │   └── configuration.ts
│   ├── database/
│   │   └── schema.prisma
│   ├── assessments/
│   │   ├── assessments.module.ts
│   │   ├── assessments.service.ts
│   │   ├── assessments.controller.ts
│   │   └── dto/
│   ├── questions/
│   │   ├── questions.service.ts
│   │   ├── questions.controller.ts
│   │   └── types/
│   │       ├── multiple-choice.type.ts
│   │       ├── short-answer.type.ts
│   │       ├── essay.type.ts
│   │       └── coding.type.ts
│   ├── attempts/
│   │   ├── attempts.service.ts
│   │   ├── attempts.controller.ts
│   │   └── dto/
│   ├── grading/
│   │   ├── grading.service.ts
│   │   ├── grading.controller.ts
│   │   ├── engines/
│   │   │   ├── auto-grader.service.ts
│   │   │   ├── rubric-grader.service.ts
│   │   │   └── code-executor.service.ts
│   │   └── dto/
│   ├── feedback/
│   │   ├── feedback.service.ts
│   │   └── dto/
│   ├── analytics/
│   │   ├── assessment-analytics.service.ts
│   │   └── question-analytics.service.ts
│   ├── events/
│   │   ├── events.service.ts
│   │   └── handlers/
│   │       ├── assessment-started.handler.ts
│   │       ├── assessment-submitted.handler.ts
│   │       └── assessment-graded.handler.ts
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

model Assessment {
  id              String    @id @default(cuid())
  organizationId  String
  courseId        String
  
  name            String
  description     String    @db.Text
  
  type            String    // 'quiz', 'midterm', 'final', 'assignment'
  
  totalPoints     Int       @default(100)
  passingScore    Int       @default(60)
  
  questions       Question[]
  attempts        AssessmentAttempt[]
  
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  @@index([organizationId])
  @@index([courseId])
}

model Question {
  id              String    @id @default(cuid())
  assessmentId    String
  assessment      Assessment @relation(fields: [assessmentId], references: [id], onDelete: Cascade)
  
  order           Int       // Question sequence
  type            String    // 'multiple_choice', 'short_answer', 'essay', 'coding'
  
  stem            String    @db.Text  // Question text
  
  // Multiple choice structure
  choices         String[]?  // For MC questions
  correctAnswer   String?    // For MC/SA questions
  
  // Code question structure
  codeTemplate    String?    @db.Text
  codeTests       String?    @db.Text  // JSON test cases
  
  // Rubric structure (for essays)
  rubric          String?    @db.Text  // JSON rubric with criteria
  
  points          Int       @default(1)
  
  responses       QuestionResponse[]
  
  createdAt       DateTime  @default(now())
  
  @@index([assessmentId])
}

model AssessmentAttempt {
  id              String    @id @default(cuid())
  userId          String
  organizationId  String
  assessmentId    String
  assessment      Assessment @relation(fields: [assessmentId], references: [id], onDelete: Cascade)
  
  attemptNumber   Int       @default(1)
  
  startedAt       DateTime  @default(now())
  submittedAt     DateTime?
  gradedAt        DateTime?
  
  timeSpentMs     Int       @default(0)
  
  score           Float?
  isPassed        Boolean?
  
  responses       QuestionResponse[]
  feedback        AssessmentFeedback?
  
  @@unique([userId, assessmentId, attemptNumber])
  @@index([userId])
  @@index([organizationId])
  @@index([isPassed])
}

model QuestionResponse {
  id              String    @id @default(cuid())
  questionId      String
  question        Question @relation(fields: [questionId], references: [id], onDelete: Cascade)
  
  attemptId       String
  attempt         AssessmentAttempt @relation(fields: [attemptId], references: [id], onDelete: Cascade)
  
  response        String    @db.Text  // User's answer
  
  isCorrect       Boolean?
  pointsAwarded   Float?
  
  gradedAt        DateTime?
  
  @@index([attemptId])
  @@index([questionId])
}

model AssessmentFeedback {
  id              String    @id @default(cuid())
  attemptId       String    @unique
  attempt         AssessmentAttempt @relation(fields: [attemptId], references: [id], onDelete: Cascade)
  
  overallFeedback String    @db.Text
  questionFeedback Json     // { question_id: "feedback text" }
  
  areasToImprove  String[]
  strengths       String[]
  
  createdAt       DateTime  @default(now())
}

model GradingRubric {
  id              String    @id @default(cuid())
  organizationId  String
  
  name            String
  description     String    @db.Text
  
  // Rubric format: { criteria: { name, levels: [{score, description}] } }
  criteria        Json
  maxScore        Int
  
  createdAt       DateTime  @default(now())
  
  @@index([organizationId])
}
```

---

## 3. Assessments Service

File: `src/assessments/assessments.service.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '../database/prisma.service';
import { EventsService } from '../events/events.service';

@Injectable()
export class AssessmentsService {
  private readonly logger = new Logger(AssessmentsService.name);

  constructor(
    private prisma: PrismaService,
    private events: EventsService,
  ) {}

  /**
   * Create an assessment with questions
   */
  async createAssessment(data: {
    organizationId: string;
    courseId: string;
    name: string;
    totalPoints: number;
    questions: any[];
  }) {
    const assessment = await this.prisma.assessment.create({
      data: {
        organizationId: data.organizationId,
        courseId: data.courseId,
        name: data.name,
        totalPoints: data.totalPoints,
        questions: {
          createMany: {
            data: data.questions.map((q, idx) => ({
              order: idx,
              type: q.type,
              stem: q.stem,
              choices: q.choices,
              correctAnswer: q.correctAnswer,
              points: q.points,
              rubric: q.rubric,
              codeTemplate: q.codeTemplate,
              codeTests: q.codeTests,
            })),
          },
        },
      },
      include: { questions: true },
    });

    this.logger.log(`Assessment ${assessment.id} created with ${data.questions.length} questions`);
    return assessment;
  }

  /**
   * Start an assessment attempt
   */
  async startAttempt(userId: string, organizationId: string, assessmentId: string) {
    // Check if previous attempt exists
    const previousAttempts = await this.prisma.assessmentAttempt.count({
      where: { userId, assessmentId },
    });

    const attempt = await this.prisma.assessmentAttempt.create({
      data: {
        userId,
        organizationId,
        assessmentId,
        attemptNumber: previousAttempts + 1,
      },
      include: { assessment: { include: { questions: true } } },
    });

    // Emit event
    await this.events.publish('assessment.started', {
      userId,
      assessmentId,
      attemptId: attempt.id,
    });

    return attempt;
  }

  /**
   * Submit assessment attempt (trigger grading)
   */
  async submitAttempt(
    attemptId: string,
    responses: Array<{ questionId: string; response: string }>,
  ) {
    const attempt = await this.prisma.assessmentAttempt.findUnique({
      where: { id: attemptId },
      include: { assessment: true },
    });

    if (!attempt) throw new Error(`Attempt ${attemptId} not found`);

    // Save responses
    for (const resp of responses) {
      await this.prisma.questionResponse.create({
        data: {
          questionId: resp.questionId,
          attemptId,
          response: resp.response,
        },
      });
    }

    // Mark submitted
    const updated = await this.prisma.assessmentAttempt.update({
      where: { id: attemptId },
      data: {
        submittedAt: new Date(),
      },
      include: { assessment: true },
    });

    // Emit event (grading service listens)
    await this.events.publish('assessment.submitted', {
      attemptId,
      assessmentId: attempt.assessmentId,
      userId: attempt.userId,
    });

    return updated;
  }

  /**
   * Get assessment with all questions
   */
  async getAssessment(assessmentId: string) {
    return this.prisma.assessment.findUnique({
      where: { id: assessmentId },
      include: { questions: true },
    });
  }

  /**
   * Get user's attempt
   */
  async getAttempt(attemptId: string) {
    return this.prisma.assessmentAttempt.findUnique({
      where: { id: attemptId },
      include: {
        assessment: true,
        responses: { include: { question: true } },
        feedback: true,
      },
    });
  }
}
```

---

## 4. Grading Engine

File: `src/grading/engines/auto-grader.service.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '../../database/prisma.service';
import { EventsService } from '../../events/events.service';

@Injectable()
export class AutoGraderService {
  private readonly logger = new Logger(AutoGraderService.name);

  constructor(
    private prisma: PrismaService,
    private events: EventsService,
  ) {}

  /**
   * Grade multiple choice and short answer questions automatically
   */
  async gradeAttempt(attemptId: string) {
    const attempt = await this.prisma.assessmentAttempt.findUnique({
      where: { id: attemptId },
      include: { responses: { include: { question: true } } },
    });

    if (!attempt) throw new Error(`Attempt not found`);

    let totalScore = 0;
    let maxScore = 0;

    for (const response of attempt.responses) {
      const question = response.question;
      maxScore += question.points;

      let isCorrect = false;
      let pointsAwarded = 0;

      // Auto-grade based on question type
      if (question.type === 'multiple_choice' || question.type === 'short_answer') {
        isCorrect = this.normalizeAnswer(response.response) === 
                    this.normalizeAnswer(question.correctAnswer);
        pointsAwarded = isCorrect ? question.points : 0;
      }

      // Update response
      await this.prisma.questionResponse.update({
        where: { id: response.id },
        data: {
          isCorrect,
          pointsAwarded,
          gradedAt: new Date(),
        },
      });

      totalScore += pointsAwarded;
    }

    // Finalize attempt
    const isPassed = totalScore >= attempt.assessment.passingScore;
    const finalAttempt = await this.prisma.assessmentAttempt.update({
      where: { id: attemptId },
      data: {
        score: totalScore,
        isPassed,
        gradedAt: new Date(),
      },
    });

    // Emit event
    await this.events.publish('assessment.graded', {
      attemptId,
      userId: attempt.userId,
      assessmentId: attempt.assessmentId,
      score: totalScore,
      maxScore,
      isPassed,
    });

    return finalAttempt;
  }

  private normalizeAnswer(answer: string): string {
    return answer
      .toLowerCase()
      .trim()
      .replace(/\s+/g, ' ')
      .replace(/[.,!?;:]/g, '');
  }
}
```

File: `src/grading/engines/code-executor.service.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class CodeExecutorService {
  private readonly logger = new Logger(CodeExecutorService.name);

  /**
   * Execute and grade coding submissions
   */
  async executeAndGrade(
    code: string,
    testCases: any[],
    language: string = 'javascript',
  ): Promise<{ passed: number; failed: number; score: number }> {
    // Use Docker sandbox for secure code execution
    // Or call external service like Judge0

    this.logger.log(`Executing code for ${testCases.length} test cases`);

    // Placeholder: call external grading service
    const result = await this.executeViaSandbox(code, testCases, language);

    return result;
  }

  private async executeViaSandbox(
    code: string,
    testCases: any[],
    language: string,
  ) {
    // TODO: Call sandbox service (e.g., Judge0 API)
    // Return { passed: 8, failed: 2, score: 80 }
    return { passed: 0, failed: 0, score: 0 };
  }
}
```

---

## 5. Event Handlers

File: `src/events/handlers/assessment-graded.handler.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { EventsService } from '../events.service';
import { NotificationService } from '../../notifications/notifications.service';

@Injectable()
export class AssessmentGradedHandler {
  private readonly logger = new Logger(AssessmentGradedHandler.name);

  constructor(
    private events: EventsService,
    private notifications: NotificationService,
  ) {
    this.setupSubscription();
  }

  private setupSubscription() {
    this.events.subscribe('assessment.graded', async (data) => {
      const { userId, assessmentId, score, isPassed } = data;

      this.logger.log(
        `Assessment graded for user ${userId}: ${score} (${isPassed ? 'passed' : 'failed'})`,
      );

      // Send notification
      await this.notifications.sendAssessmentResult({
        userId,
        assessmentId,
        score,
        isPassed,
      });
    });
  }
}
```

---

## 6. API Endpoints

File: `src/assessments/assessments.controller.ts`

```typescript
import {
  Controller,
  Post,
  Get,
  Body,
  Param,
  UseGuards,
} from '@nestjs/common';
import { AssessmentsService } from './assessments.service';
import { JwtGuard } from '../common/guards/jwt.guard';
import { CurrentUser } from '../common/decorators/current-user.decorator';

@Controller('api/assessments')
@UseGuards(JwtGuard)
export class AssessmentsController {
  constructor(private assessmentsService: AssessmentsService) {}

  /**
   * POST /api/assessments
   * Create assessment (instructor only)
   */
  @Post()
  async createAssessment(@Body() dto: any) {
    return this.assessmentsService.createAssessment(dto);
  }

  /**
   * GET /api/assessments/:id
   * Get assessment
   */
  @Get(':id')
  async getAssessment(@Param('id') id: string) {
    return this.assessmentsService.getAssessment(id);
  }

  /**
   * POST /api/assessments/:id/start
   * Start assessment attempt
   */
  @Post(':id/start')
  async startAttempt(
    @Param('id') assessmentId: string,
    @CurrentUser() userId: string,
    @CurrentUser('orgId') organizationId: string,
  ) {
    return this.assessmentsService.startAttempt(userId, organizationId, assessmentId);
  }

  /**
   * POST /api/assessments/attempts/:id/submit
   * Submit assessment
   */
  @Post('attempts/:id/submit')
  async submitAttempt(
    @Param('id') attemptId: string,
    @Body() dto: { responses: Array<{ questionId: string; response: string }> },
  ) {
    return this.assessmentsService.submitAttempt(attemptId, dto.responses);
  }

  /**
   * GET /api/assessments/attempts/:id
   * Get attempt with responses
   */
  @Get('attempts/:id')
  async getAttempt(@Param('id') id: string) {
    return this.assessmentsService.getAttempt(id);
  }
}
```

---

## 7. Wrapper Functions

File: `lumina-learning-hub/supabase/functions/submit-assessment/index.ts`

```typescript
import { handleWrapperRequest } from "../_shared/gateway-wrapper.ts";
import { corsHeaders } from "../_shared/cors.ts";

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }

  return handleWrapperRequest(req, {
    microserviceUrl:
      Deno.env.get("ASSESSMENT_SERVICE_URL") || "http://localhost:3005",
    microservicePath: "/api/assessments/attempts",
    fallbackRpcName: "submit_assessment_legacy",
    timeoutMs: 8000,
    enabled: Deno.env.get("WRAPPER_ENABLE_ASSESSMENT") !== "false",
  });
});
```

---

## 8. Docker Setup

File: `docker-compose.yml`

```yaml
version: '3.8'

services:
  assessment-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: assessment
      POSTGRES_PASSWORD: password
      POSTGRES_DB: assessment
    ports:
      - "5437:5432"
    volumes:
      - assessment-db-data:/var/lib/postgresql/data

  assessment-service:
    build: .
    ports:
      - "3005:3005"
    environment:
      DATABASE_URL: postgresql://assessment:password@assessment-db:5432/assessment
      NATS_URL: nats://nats:4222
    depends_on:
      - assessment-db

volumes:
  assessment-db-data:
```

---

## 9. Deployment Checklist

- [ ] PostgreSQL configured
- [ ] Question types tested (MC, SA, essays, coding)
- [ ] Auto-grading engine working
- [ ] Code sandbox integrated (if offering)
- [ ] Rubric grader working for essays
- [ ] Event handlers subscribed
- [ ] Wrapper functions deployed
- [ ] Fallback RPCs created
- [ ] Canary traffic routed
- [ ] Analytics reporting working

## Success Criteria

✅ Assessment starts in <500ms  
✅ Auto-grading completes in <2 seconds  
✅ Manual grading UI loads in <1 second  
✅ Feedback sent to student within 5 seconds  
✅ All question types graded correctly (100% auto-grade accuracy)

**Phase 5 is the gateway to course content. Enables foundation for Phase 6.**
