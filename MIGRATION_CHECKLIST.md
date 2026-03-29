# Lumina Microservices Migration Checklist

**Status: ACTIVE MIGRATION**  
Last Updated: 2026-03-29  
Target Completion: Q4 2026

---

## Quick Status Dashboard

| Phase | Service | Status | Duration | Risk | Owner |
|-------|---------|--------|----------|------|-------|
| 0 | API Gateway + BFF | 🟠 In Planning | 2 weeks | Low | Platform |
| 1 | Notification | ⬜ Not Started | 4 weeks | Low | Services |
| 2 | Gamification | ⬜ Not Started | 4 weeks | Low | Services |
| 3 | AI & Personalization | ⬜ Not Started | 6 weeks | Medium | ML/Services |
| 4 | Analytics & Engagement | ⬜ Not Started | 6 weeks | Medium | Analytics |
| 5 | Assessment & Grading | ⬜ Not Started | 6 weeks | Medium | Services |
| 6 | Course + Enrollment | ⬜ Not Started | 8 weeks | High | Core Services |
| 7 | Identity Finalization | ✅ DONE | - | - | Auth |

---

## Phase 0: API Gateway + BFF Foundation

**Timeline:** Weeks 1–2  
**Owner:** Platform Team  
**Risk Level:** Low

### Infrastructure Setup
- [ ] Provision API Gateway infrastructure (Kong, Traefik, or AWS API Gateway)
  - [ ] Namespace and service discovery configured
  - [ ] TLS/mTLS certificates issued
  - [ ] Rate limiting rules in place
  - [ ] CORS policy configured
- [ ] Setup BFF (Backend-for-Frontend) layer
  - [ ] Docker image created and pushed to registry
  - [ ] Kubernetes deployment manifests created
  - [ ] Service mesh integration (Istio/Linkerd) enabled
  - [ ] Health checks configured (/health, /ready)

### Frontend Configuration
- [ ] Environment variables defined and documented
  - [ ] `VITE_GATEWAY_URL` set for non-prod
  - [ ] `VITE_PROXY_SUPABASE_VIA_GATEWAY` flag available
  - [ ] Feature flag system in place for gradual rollout
- [ ] Gateway client updated
  - [ ] `gatewayRequest` function tested
  - [ ] Fallback behavior validated
  - [ ] Timeout tuning completed
  - [ ] Error handling enhanced with retries

### Testing and Validation
- [ ] Load testing on gateway (1k req/sec baseline)
- [ ] Latency benchmarks captured (target: <200ms p99)
- [ ] Fallback to direct Supabase verified
- [ ] Auth token propagation working
- [ ] Integration tests passing

### Sign-Off
- [ ] Tech review completed
- [ ] Staging deployment successful
- [ ] Production rollout plan finalized

---

## Phase 1: Notification Service

**Timeline:** Weeks 3–6  
**Owner:** Services Team  
**Risk Level:** Low  
**Dependencies:** Phase 0 (API Gateway)

### Service Development
- [ ] Create notification service repository (or monorepo package)
  - [ ] NestJS/FastAPI base boilerplate
  - [ ] Prisma schema migration complete
  - [ ] Docker image and registry setup
  - [ ] Kubernetes manifests created
- [ ] Database schema migration
  - [ ] `notifications` table migrated
  - [ ] `scheduled_nudges` table migrated
  - [ ] `notification_preferences` table migrated
  - [ ] `support_tickets` table migrated
- [ ] Service endpoints implemented
  - [ ] `GET /notifications` (list user notifications)
  - [ ] `POST /notifications` (create)
  - [ ] `PATCH /notifications/:id` (mark as read)
  - [ ] `DELETE /notifications/:id`
  - [ ] `GET /preferences` (notification settings)

### Edge Function Migration
- [ ] Migrate `send-invitation` logic to service
  - [ ] Email template handling
  - [ ] Tenant context validation
- [ ] Migrate `send-ticket-notification` to service
  - [ ] Support ticket workflow
- [ ] Migrate `auth-email-hook` to service (deprecate Supabase hook)

### Event Integration
- [ ] Event consumer setup
  - [ ] Subscribe to `achievement.unlocked` events
  - [ ] Subscribe to `assessment.graded` events
  - [ ] Subscribe to `user.created` events
- [ ] Event handler logic
  - [ ] Notification creation for each event
  - [ ] Email dispatch via Resend or SendGrid
  - [ ] Idempotency key deduplication

### Dual-Write and Cutover
- [ ] Dual-write implementation (old + new DB)
  - [ ] Write path in BFF routes to both
  - [ ] Read path routed to notification service
  - [ ] Validation scripts for row count matching
- [ ] 2-week validation period
  - [ ] Checksum verification (daily)
  - [ ] Manual spot checks
  - [ ] User feedback collection
- [ ] Cutover complete
  - [ ] Old notification data archived
  - [ ] Supabase RLS policies deprecated

### Frontend Migration
- [ ] `useNotifications` hook updated
  - [ ] Points to gateway endpoint
  - [ ] Fallback to service API if gateway unavailable
- [ ] UI components tested
  - [ ] NotificationBell component verified
  - [ ] Real-time notification display working

### Testing and Deployment
- [ ] Unit tests (90%+ coverage)
- [ ] Integration tests (Testcontainers)
- [ ] Contract tests against BFF
- [ ] Staging deployment and validation
- [ ] Canary deployment (5% traffic, 1 day)
- [ ] Full production rollout

### Sign-Off
- [ ] Performance SLOs met (p99 <200ms)
- [ ] Error rate tracking dashboard created
- [ ] Runbook and incident response plan documented

---

## Phase 2: Gamification Service

**Timeline:** Weeks 7–10  
**Owner:** Services Team  
**Risk Level:** Low  
**Dependencies:** Phase 0, Phase 1

### Service Development
- [ ] Create gamification service
  - [ ] Base scaffolding and Docker setup
  - [ ] Prisma schema for gamification tables
  - [ ] API endpoints for XP, achievements, leaderboards
- [ ] Database schema migration
  - [ ] `achievements` table
  - [ ] `user_achievements` table
  - [ ] `user_experience` table
  - [ ] `xp_transactions` table (audit log)
  - [ ] `daily_challenges` table
  - [ ] `confidence_boosters` table
  - [ ] `learning_milestones` table
  - [ ] `user_milestones` table

### Service Endpoints
- [ ] `GET /me/xp` (user experience points)
- [ ] `GET /me/achievements` (unlocked achievements)
- [ ] `POST /xp/award` (internal endpoint from event consumer)
- [ ] `GET /leaderboard` (global/org-level rankings)
- [ ] `GET /challenges/daily` (today's challenges)
- [ ] `POST /me/challenges/:id/complete`

### Event Integration
- [ ] Event consumers
  - [ ] `lesson.completed` → Award XP
  - [ ] `enrollment.created` → Award enrollment XP
  - [ ] `assessment.graded` → Award assessment XP
  - [ ] `achievement.unlocked` → Dispatch to notification service
- [ ] Redis leaderboard cache
  - [ ] Sorted set for rankings
  - [ ] Cache invalidation on XP change
  - [ ] TTL strategy (refresh every 5 min)

### Real-Time Updates (WebSocket)
- [ ] WebSocket handler for live XP/leaderboard
  - [ ] Namespace per organization
  - [ ] Broadcast on XP award
  - [ ] User-specific achievement unlock notifications
- [ ] Frontend WebSocket client in hooks

### Dual-Write and Cutover
- [ ] Dual-write in BFF to old + new DB
- [ ] 2-week validation period
- [ ] Cutover after validation

### Frontend Migration
- [ ] `useRewards` hook refactored
- [ ] `useAchievements` hook refactored
- [ ] Dashboard leaderboard components updated
- [ ] Achievement unlock popup triggered via WebSocket

### Testing and Deployment
- [ ] Unit tests for XP calculation logic
- [ ] Integration tests with event consumer
- [ ] Load testing on leaderboard endpoint (1M users stress test)
- [ ] Staging and canary rollout

### Sign-Off
- [ ] Leaderboard response time <100ms (p99)
- [ ] XP awarded within 5 seconds of event

---

## Phase 3: AI & Personalization Service

**Timeline:** Weeks 11–16  
**Owner:** ML + Services Team  
**Risk Level:** Medium  
**Dependencies:** Phase 0, Phase 1

### Infrastructure
- [ ] GPU node pool provisioned (K8s)
  - [ ] NVIDIA GPU drivers installed
  - [ ] TensorRT or vLLM runtime set up
  - [ ] Model registry (HuggingFace Hub or local)
- [ ] Service autoscaling
  - [ ] Min 1 replica (GPU expensive), max 3
  - [ ] Metrics-based scaling (GPU utilization)

### Service Development
- [ ] Python/FastAPI microservice
  - [ ] Async request handling
  - [ ] GPU memory management
  - [ ] Model loading/caching on startup
- [ ] Database schema
  - [ ] `career_roadmaps` table
  - [ ] `roadmap_skills` table
  - [ ] `skill_stages` table
  - [ ] `ai_content_recommendations` table
  - [ ] `ai_roadmap_customizations` table
  - [ ] `learning_insights` table
  - [ ] `learning_goals` table

### Service Endpoints
- [ ] `POST /roadmap/generate` (career roadmap)
- [ ] `POST /tutor/chat` (AI tutor with SSE streaming)
- [ ] `POST /recommendations/content` (learning resources)
- [ ] `POST /recommendations/path` (next lesson path)
- [ ] `POST /insights/generate` (learning analytics)
- [ ] `POST /behavior/analyze` (user learning pattern)

### LLM Model Integration
- [ ] Model selection and licensing
  - [ ] Specify base model (GPT-3.5, Llama 2, etc.)
  - [ ] Cost estimation per request
  - [ ] Latency targets (SSE response start within 2s)
- [ ] Context management
  - [ ] User learning history chunking
  - [ ] Token counting and trimming
  - [ ] Prompt engineering for consistency

### Event Integration
- [ ] Event producer: `roadmap.generated`
  - [ ] Consumed by Progress service
  - [ ] Consumed by Notification service
- [ ] Event consistency checks

### Streaming Response (SSE)
- [ ] Server-sent events for AI tutor
  - [ ] Token-by-token streaming
  - [ ] Proper error handling mid-stream
  - [ ] Client-side abort support

### Dual-Write and Cutover
- [ ] Careful shadow mode (read only from new, write both)
- [ ] 2-week validation period
- [ ] Full cutover

### Frontend Migration
- [ ] `useCareerRoadmap` hook updated
- [ ] `useInstructorAI` hook updated
- [ ] AI tutor UI with streaming display
- [ ] Error boundaries for LLM failures

### Testing and Deployment
- [ ] Unit tests for prompt templates
- [ ] Integration tests with sample data
- [ ] Load testing (100 concurrent AI requests)
- [ ] Latency profiling (target: first token <2s)
- [ ] Staging canary with reduced traffic initially (1%)

### Sign-Off
- [ ] Cost per request baseline captured
- [ ] GPU utilization monitoring dashboard
- [ ] Incident runbook for model failures

---

## Phase 4: Analytics & Engagement Service

**Timeline:** Weeks 17–22  
**Owner:** Analytics Team  
**Risk Level:** Medium  
**Dependencies:** Phase 0, Phase 1, Phase 2

### Infrastructure
- [ ] TimescaleDB or ClickHouse setup
  - [ ] Time-series optimized schema
  - [ ] Retention policy (1 year default)
  - [ ] Backup and recovery plan
- [ ] Event ingestion pipeline
  - [ ] Kafka or RabbitMQ broker
  - [ ] Message buffer (at-least-once semantics)
  - [ ] DLQ for failed messages

### Service Development
- [ ] Analytics microservice
  - [ ] Event ingestion endpoint
  - [ ] Query and aggregation endpoints
  - [ ] Real-time dashboard WebSocket
- [ ] Database schema
  - [ ] `engagement_events` (high-volume, partitioned by date)
  - [ ] `user_behavior_patterns` (materialized aggregates)
  - [ ] `learning_analytics_snapshots` (daily snapshots)

### Service Endpoints
- [ ] `POST /events/ingest` (event stream)
- [ ] `GET /analytics/dashboard` (summary stats)
- [ ] `GET /user/:id/behavior` (user learning pattern)
- [ ] `GET /cohort/:id/engagement` (class-level metrics)
- [ ] WebSocket `/live-dashboard` (streaming updates)

### Event Consumer
- [ ] Subscribe to ALL event types for analytics
  - [ ] `user.created`, `enrollment.created`, `lesson.completed`, etc.
  - [ ] Transform and aggregate in real time
  - [ ] Store in TimescaleDB

### Dashboards
- [ ] Dashboard service endpoints
  - [ ] Learning progress by cohort
  - [ ] Engagement heatmap
  - [ ] XP distribution (Gamification integration)
  - [ ] AI recommendation effectiveness

### Dual-Write and Cutover
- [ ] Parallel ingestion for 2-week period
- [ ] Event comparison and validation
- [ ] Cutover after validation confirmed

### Frontend Migration
- [ ] `useLearningAnalytics` hook refactored
- [ ] `useEngagement` hook refactored
- [ ] Dashboard components updated to use new API
- [ ] Real-time chart refresh via WebSocket

### Testing and Deployment
- [ ] Load testing (10k events/sec ingestion)
- [ ] Query performance benchmarks
- [ ] Staging with production-like traffic volume
- [ ] Canary deployment

### Sign-Off
- [ ] Event ingestion latency <500ms (p99)
- [ ] Dashboard load time <1s
- [ ] Data accuracy validated (spot checks)

---

## Phase 5: Assessment & Grading Service

**Timeline:** Weeks 23–28  
**Owner:** Services Team  
**Risk Level:** Medium  
**Dependencies:** Phase 0, Course service API contract

### Service Development
- [ ] Assessment microservice
  - [ ] REST API for assignments and grades
  - [ ] Async grading job queue
- [ ] Database schema
  - [ ] `assignments` table
  - [ ] `assignment_submissions` table
  - [ ] `user_skill_assessments` table
  - [ ] `assessment_question_responses` table

### Service Endpoints
- [ ] `GET /assignments` (list for course/student)
- [ ] `POST /assignments` (create, instructor only)
- [ ] `POST /submissions` (student submits)
- [ ] `GET /submissions/:id/grade` (automatic or manual)
- [ ] `PATCH /submissions/:id/grade` (manual grading)
- [ ] `GET /user/:id/skills` (skill assessment results)

### Course Service Integration
- [ ] API contract testing
  - [ ] Validate lesson_id exists in Course service
  - [ ] Fetch course metadata for context
  - [ ] Handle course deletion (cascade or orphan check)

### Automatic Grading
- [ ] Grading engine for quiz/objective questions
  - [ ] Multiple choice, true/false, matching
  - [ ] Score calculation with weighted rubric
- [ ] Async grading for essays (human review)
  - [ ] Queue system for instructor notifications
  - [ ] Status tracking (pending, graded)

### Event Integration
- [ ] Producer: `assessment.graded`
  - [ ] Consumed by Gamification (XP award)
  - [ ] Consumed by Analytics
  - [ ] Consumed by Notification (grade email)

### Dual-Write and Cutover
- [ ] Dual-write to old (RLS) + new service
- [ ] 2-week validation
- [ ] Cutover

### Frontend Migration
- [ ] `useAssignmentSubmission` hook refactored
- [ ] AssignmentPlayer component updated
- [ ] GradingInterface component updated

### Testing and Deployment
- [ ] Unit tests for grading logic
- [ ] Integration tests with Course service API
- [ ] Contract tests (ensure course data availability)
- [ ] Staging and canary rollout

### Sign-Off
- [ ] Grading time baseline captured (target: <2s)
- [ ] Incident plan for service unavailability

---

## Phase 6: Course Management + Enrollment/Progress

**Timeline:** Weeks 29–36  
**Owner:** Core Services Team  
**Risk Level:** High  
**Dependencies:** All prior phases

### Service Development (Part A: Course Management)
- [ ] Course microservice
  - [ ] REST + GraphQL endpoints (content has rich relationships)
  - [ ] Content versioning (drafts, published)
  - [ ] Publishing workflow approval
- [ ] Database schema
  - [ ] `courses` table
  - [ ] `modules` table
  - [ ] `lessons` table
  - [ ] `quizzes` table
  - [ ] `quiz_questions` table
  - [ ] `content_drafts` table
  - [ ] `content_suggestions` table
  - [ ] `curated_resources` table

### Service Development (Part B: Enrollment/Progress)
- [ ] Enrollment microservice
  - [ ] Student enrollment lifecycle
  - [ ] Progress tracking and calculations
  - [ ] Learning streak management
  - [ ] Real-time WebSocket for `lesson_progress`
- [ ] Database schema
  - [ ] `enrollments` table
  - [ ] `lesson_progress` table
  - [ ] `learning_streaks` table
  - [ ] `learning_path_nodes` table

### Service Endpoints (Course)
- [ ] `GET /courses` (catalog, paginated)
- [ ] `POST /courses` (create, instructor)
- [ ] `GET /courses/:id` (with modules, lessons)
- [ ] `PATCH /courses/:id` (update)
- [ ] `POST /courses/:id/publish` (workflow)
- [ ] GraphQL endpoint for rich queries

### Service Endpoints (Enrollment)
- [ ] `GET /me/enrollments` (student's courses)
- [ ] `POST /enrollments` (student enrolls)
- [ ] `GET /enrollments/:id/progress` (detailed progress)
- [ ] `PATCH /lessons/:id/progress` (track progress)
- [ ] WebSocket `/progress/:enrollment_id` (real-time)

### Event Integration
- [ ] Producer: `enrollment.created`
  - [ ] Consumed by Progress (init tracking)
  - [ ] Consumed by Gamification (XP)
  - [ ] Consumed by Analytics
  - [ ] Consumed by Notification
- [ ] Producer: `lesson.completed`
  - [ ] Consumed by Gamification, Analytics, AI

### Dual-Write and Cutover (Extended)
- [ ] Dual-write for 4 weeks (higher risk)
  - [ ] Tests for eventual consistency
  - [ ] Reconciliation job runs hourly
  - [ ] Checksum validation per batch
- [ ] Gradual traffic shift
  - [ ] Week 1: 10% reads from new service
  - [ ] Week 2: 50% reads from new service
  - [ ] Week 3–4: 90% reads, monitor errors
  - [ ] Cutover on week 5

### Frontend Migration
- [ ] Major hooks updated:
  - [ ] `useCourses` → gateway
  - [ ] `useEnrollment` → gateway
  - [ ] `useProgress` → gateway
  - [ ] `useLearningPath` → gateway
- [ ] Component updates (20+ pages):
  - [ ] CourseCard, CourseDetail
  - [ ] LessonPlayer
  - [ ] MyCourses dashboard
  - [ ] Course editor (if instructor)

### Testing and Deployment
- [ ] Extensive contract tests with dependent services
- [ ] Chaos testing (simulate service failures)
- [ ] Load testing (10k concurrent enrolled users)
- [ ] High-traffic staging environment replication
- [ ] Slow canary rollout (1%, 5%, 25%, 50%, 100%)

### Sign-Off
- [ ] Zero downtime during cutover
- [ ] Reconciliation job success rate >99%
- [ ] User-facing latency maintained <500ms (p99)

---

## Phase 7: Identity Service Finalization & Decommission

**Timeline:** Weeks 37–40  
**Owner:** Auth Team  
**Risk Level:** High  
**Dependencies:** All prior phases complete

### Identity Service Completion
- [ ] Clerk integration hardened
  - [ ] All OAuth2 providers active
  - [ ] MFA enforcement configurable per org
  - [ ] Session management robust (timeouts, revocation)
- [ ] Token strategy finalized
  - [ ] JWT payload agreed upon (org_id, roles, permissions)
  - [ ] Token lifetime set (15 min access, 7 day refresh)
  - [ ] Claim validation in all services implemented

### Supabase Decommission
- [ ] Data export and backup
  - [ ] All user data exported to new identity DB
  - [ ] Archival backup to cold storage
- [ ] Auth path cutover
  - [ ] Remove Supabase GoTrue dependency
  - [ ] Direct Supabase client calls removed from frontend
  - [ ] All auth routed through identity service + gateway
- [ ] RLS policies deprecated
  - [ ] Document legacy RLS logic for reference
  - [ ] Verify no RLS queries remain

### Frontend Cleanup
- [ ] Remove direct supabase.auth calls
  - [ ] All auth via `useAuth` hook only
  - [ ] `supabase` client used only for fallback (if any)
- [ ] Remove Supabase client SDK (if fully migrated)
  - [ ] Or keep minimal version for other integrations

### Validation and Sign-Off
- [ ] Full platform audit
  - [ ] No direct auth service calls outside gateway
  - [ ] All 8 services healthy and responsive
  - [ ] Data consistency across all DBs verified
  - [ ] Event bus delivering messages reliably
- [ ] Performance baseline met
  - [ ] API latency SLOs met
  - [ ] Database query performance stable
  - [ ] Event delivery latency <5s (p99)
- [ ] Team training
  - [ ] Runbooks and incident response tested
  - [ ] On-call engineer trained
  - [ ] Documentation complete and accessible

### Final Decommission
- [ ] Supabase project archived or deleted
- [ ] DNS records cleaned up
- [ ] Cost tracking updated for new infrastructure
- [ ] Retrospective performed

---

## Cross-Phase Tracking

### Continuous Validation
- [ ] Nightly data consistency checks (all services)
- [ ] Weekly alert threshold review
- [ ] Bi-weekly architecture review meeting
- [ ] Monthly cost and performance analysis

### Documentation
- [ ] API documentation updated per phase
- [ ] Runbooks created for each service
- [ ] Architecture decision records (ADRs) logged
- [ ] Knowledge base wiki maintained

### Team Communication
- [ ] Daily standup (15 min)
- [ ] Weekly steering committee (1 hr)
- [ ] Bi-weekly all-hands update
- [ ] Slack channels: #migration-status, #incidents, #test-results

### Rollback Procedures
- [ ] Each phase has documented rollback steps
- [ ] Test rollback monthly
- [ ] At-a-glance decision tree in runbook

---

## Overall Success Criteria

- ✅ Zero production incidents caused by migration
- ✅ User-facing latency does not increase >10%
- ✅ Cost reduction of 20% post-migration
- ✅ Deployment frequency increases from weekly to daily
- ✅ MTTR (mean time to recovery) improves by 40%
- ✅ Engineer velocity increases (independent service releases)
- ✅ All SLOs maintained (99.95% uptime target)

---

## Key Contacts

| Role | Name | Slack | Email |
|------|------|-------|-------|
| Migration Lead | [Your Name] | @migration-lead | lead@example.com |
| Platform Eng | [Platform Owner] | @platform | platform@example.com |
| Backend Services | [Backend Lead] | @backend | backend@example.com |
| ML/AI Team | [ML Lead] | @ml-team | ml@example.com |
| Analytics | [Analytics Lead] | @analytics | analytics@example.com |
| DevOps/Infrastructure | [DevOps Lead] | @devops | devops@example.com |

---

## Appendix: Quick Ref Commands

### Check phase status
```bash
# View migration status dashboard
kubectl get svc -l phase=<phase-number>
```

### View logs for service
```bash
docker logs <service-name>
# or
kubectl logs -f svc/<service-name>
```

### Run data validation
```bash
# Compare row counts (example)
psql -h old-db -c "SELECT COUNT(*) FROM notifications" && \
psql -h new-service-db -c "SELECT COUNT(*) FROM notifications"
```

### Monitor events
```bash
# View events being published
nats sub -s nats://localhost:4222 '>'
```
