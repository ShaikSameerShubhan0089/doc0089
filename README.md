# Lumina Migration Review & Recommended Development Plan

Date: 2026-03-29

## 1. Overview

This document provides a complete migration assessment for the Lumina monolith (`lumina-learning-hub`) and existing microservice (`lumina-auth-service`) against the provided Pathwisse microservices migration plan (`pathwisse_migration_plan.pdf`).

Goals:
- Validate current implementation against the migration plan.
- Recommend a practical, low-risk microservices migration roadmap.
- Deliver concrete tasks and schedule for phase completion.

## 2. Current Code State Summary

### 2.1 lumina-learning-hub (monolith)

- React + Vite + Supabase client.
- Supabase direct SQL calls in many hooks and pages (e.g., `useEnrollment`, `useAuth`, `TenantManager`, `CourseEditor`, etc.).
- Edge functions live in `supabase/functions/` (one-to-one with plan functions such as `send-invitation`, `track-milestone`, `ai-tutor`, etc.).
- Gateway support exists:
  - `src/lib/gateway-client.ts` has `gatewayRequest` / `withGatewayFallback`.
  - `src/hooks/useAuth.tsx` attempts gateway first, then Supabase RPC fallback.
  - `src/integrations/supabase/client.ts` includes `VITE_PROXY_SUPABASE_VIA_GATEWAY` switch.

### 2.2 lumina-auth-service (microservice)

- NestJS + Prisma + Clerk integration.
- Implements identity, user roles, callback events and access control.
- Key services:
  - `UserService` (getMe, listUsers, primary role logic)
  - `RoleService` (assign/remove roles, hierarchy checks, audit log, Clerk metadata sync)
- This matches Pathwisse extracted Identity service.

## 3. Pathwisse Migration Plan Review

### 3.1 Plan content summary

- 8 phases over 40 weeks following the Strangler Fig Pattern.
- Services:
  1. Identity
  2. Course Management
  3. Enrollment & Progress
  4. Assessment & Grading
  5. AI & Personalization
  6. Analytics & Engagement
  7. Gamification
  8. Notification
- Infrastructure: API Gateway, Service Mesh (Istio/Linkerd), Event Bus (NATS/RabbitMQ/Kafka), K8s, DB per service.
- Data migration with dual-write and checksum validation.
- Frontend upgrade with service-aware API client and hook adapters.

### 3.2 Gap analysis vs implementation

- Identity service is implemented (`lumina-auth-service`).
- API gateway path exists partially via frontend + `gateway-client`.
- Notification, Gamification, AI, Analytics not yet separate services but edge functions already exist.
- Monolith dependency remains (direct Supabase calls in UI) there is “gradual migration path”.

## 4. Recommended Development Plan (Enhanced)

### 4.1 Guiding principles

- Stay fully backwards-compatible (dual route) until final.
- Use `featureFlag` by service-phase for traffic switching.
- Deploy small measurable increments (no big bang).
- Fail-safe path: supabase continues if gateway/microservices fail.
- Keep developers productive with current path while migrating.

### 4.2 Active service extraction priority

1. Identity (already done; treat as canonical proof-of-concept).
2. Notification (lowest coupling, quick win).
3. Gamification (event-driven, moderate risk).
4. AI & Personalization (needs GPU compute & separate scaling).
5. Analytics & Engagement (high volume, needs specialized DB).
6. Assessment & Grading (moderately coupled to course model).
7. Course management + Enrollment/Progress (core domain, highest risk; perform near project completion after other dependencies are stable).

### 4.3 Suggested roadmap (12-month, 3-month sprints)

- Sprint 0 (2 weeks): API Gateway + BFF + auth token strategy
- Sprint 1 (4 weeks): Notification service implementation + keep supabase compatibility
- Sprint 2 (4 weeks): Gamification service + events + leaderboard caching
- Sprint 3 (6 weeks): AI service migration & sandbox with GPU endpoint
- Sprint 4 (6 weeks): Analytics ingestion + time-series store + event consumer
- Sprint 5 (6 weeks): Assessment service + test contract with course microservice API
- Sprint 6 (8 weeks): Course + Enrollment service extraction, dual-write, cutover
- Sprint 7 (4 weeks): Decommission supabase direct SQL and finalize

### 4.4 Frontend migration guidance

- Convert hooks to use `gatewayClient` first; maintain old interface.
- Build base wrapper `src/services` with unified calls.
- Add `useBackendPreference` to toggle between gateway and direct `supabase` for stage.
- Replace direct `supabase.from()` calls in priorities:
  - Authentication + user context
  - Enrollment progress
  - Course CRUD
  - Assessment workflows
- Keep edge functions running as transitional compatibility until microservice endpoints are 100% stable.

### 4.5 Data migration improvements

- Add an outbox event table for write synchronization.
- Add change-data-capture (CDC) for important business tables.
- “Mailman” reconciliation background job every 5 min.
- Phase key point: dual-write including old and new service during 2-4 weeks.

### 4.6 Cross-cutting concerns (must always be in place)

- Observability: OpenTelemetry + traces + structured logs + centralized aggregator.
- Security: JWT claim-based tenant and role enforcement; service-to-service mTLS.
- Resilience: Circuit breakers, timeouts, retries, DLQs.
- Infrastructure as code: Terraform/CloudFormation for all resources.
- CI/CD: per-service pipeline, contract checks, canary deploys.

## 5. Detailed phase chapter

[**Full bullet roadmap and technical steps by phase** available in the attached slide (below).]

**Phase 0**
- Configure API gateway (Kong or Traefik) with routing rules.
- Deploy BFF endpoints to proxy existing Supabase queries.
- Frontend config: `VITE_PROXY_SUPABASE_VIA_GATEWAY=true`.

**Phase 1: Notification**
- Create new `notification-service` repo/module (or monorepo package) with REST API.
- Migrate DB schema + existing notification related edge functions.
- Start event consumer for `achievement.unlocked` and `assessment.graded`.

**Phase 2: Gamification**
- Create gamification service and event consumers.
- Migrate XP/achievement tables and domain logic from edge functions.

**Phase 3: AI/Personalization**
- Build Python/FastAPI AI service with existing edge function logic.
- Migrate AI-related tables and output to event bus.

**Phase 4: Assessment & Grading**
- Move grading workflows to dedicated service; reuse courses API for lesson validation.

**Phase 5: Analytics & Engagement**
- Implement stream ingestion and storage in TimescaleDB/ClickHouse.
- Build dashboards endpoint and live WebSocket.

**Phase 6: Course + Enrollment**
- Final heavy migration; extensive API contracts and migration scripts.

**Phase 7: Identity decommission**
- Finalize and remove Supabase GoTrue path.
- Move to complete auth service (Clerk + local JWT tokens as needed).

## 6. Technical action items (with implementation notes)

1. Add one common adapter: `src/services/gateway-api.ts`
2. Refactor `useAuth` and `useEnrollment` to optional `gateway` mode.
3. Add `migration.status` field in DB to track live service ownership.
4. Battle test dual-write with a non-prod cutover exercise.
5. Add route-level contract tests for each extracted service.
6. Pipe edge function calls to microservices in `supabase/functions` as a canary.

## 7. Risks & mitigations

- Data inconsistency in dual writes: use idempotent upsert, reconciliation jobs.
- Latency from API hops: API Gateway cache + Redis near service.
- Complexity from many independent services: enforce standards, strong docs.
- Temporary cost increase: monitor autoscale and enforce scheduler.

## 8. Deliverables & Documents

- `migration-plan-analysis.md` (this file)
- `migration-plan-analysis.pdf` (generated)
- `migration-plan-analysis.docx` (generated)

---

### Appendix: relevant files reviewed

- lumina-auth-service/src/user/user.service.ts
- lumina-auth-service/src/role/role.service.ts
- lumina-learning-hub/src/integrations/supabase/client.ts
- lumina-learning-hub/src/lib/gateway-client.ts
- lumina-learning-hub/src/hooks/useAuth.tsx
- lumina-learning-hub/src/hooks/useEnrollment.ts
- lumina-learning-hub/supabase/functions/**/*
- pathwisse_migration_plan.pdf
