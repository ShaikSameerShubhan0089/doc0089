# Gateway Wrapper Plan: Edge Functions → Microservices

**Objective:** Incrementally convert Supabase edge functions into temporary gateway wrappers that delegate to emerging microservices while maintaining backward compatibility.

**Timeline:** Per phase (Phase 1 onwards)  
**Owner:** Services Team + DevOps

---

## 1. Architecture Overview

### Current State
```
Frontend (React SPA)
    ↓ supabase.functions.invoke('send-invitation', {...})
Supabase Edge Functions (Deno)
    ↓
PostgreSQL (monolith schema)
```

### Target State (Phase N)
```
Frontend (React SPA)
    ↓ gatewayRequest('/notification/send-invitation', {...})
API Gateway (Kong/Traefik)
    ↓
Notification Microservice (NestJS/FastAPI)
    ↓
Notification Service DB (isolated schema)

[LEGACY FALLBACK: If microservice unavailable]
    ↓ supabase.functions.invoke('send-invitation', {...})
Supabase Edge Function (Wrapper) → Calls microservice or falls back to RPC
    ↓
PostgreSQL
```

### Wrapper Pattern
Each edge function becomes a **thin proxy** that:
1. Validates auth and input
2. Tries to call the new microservice endpoint first
3. Falls back to local RPC or deprecated logic if service unreachable
4. Logs the routing decision for monitoring

---

## 2. Wrapper Pattern Implementation

### 2.1 Generic Wrapper Template (Deno Edge Function)

File: `supabase/functions/[service-name]/index.ts`

```typescript
// Example: Generic wrapper pattern for any edge function
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";
import { corsHeaders } from "../_shared/cors.ts";

export const supabase = createClient(
  Deno.env.get("SUPABASE_URL")!,
  Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
);

interface WrapperConfig {
  /** Microservice endpoint URL (e.g., http://notification-service:3000) */
  microserviceUrl: string;
  /** Path on microservice (e.g., /api/notifications/send-invitation) */
  microservicePath: string;
  /** Fallback RPC name if service unavailable */
  fallbackRpcName?: string;
  /** Timeout for microservice call in ms */
  timeoutMs?: number;
}

/**
 * Generic gateway wrapper for edge functions.
 * Routes to microservice, falls back to RPC if service is down.
 */
async function handleWrapperRequest(
  req: Request,
  config: WrapperConfig
): Promise<Response> {
  const token = req.headers.get("authorization")?.replace("Bearer ", "");

  if (!token) {
    return new Response(
      JSON.stringify({ error: "Unauthorized" }),
      { status: 401, headers: corsHeaders }
    );
  }

  try {
    // Extract request body
    let body: any = {};
    if (req.method !== "GET") {
      body = await req.json();
    }

    // Step 1: Try microservice first
    console.log(
      `[${config.microservicePath}] Attempting microservice call to ${config.microserviceUrl}${config.microservicePath}`
    );

    const serviceResponse = await callMicroservice(
      config.microserviceUrl,
      config.microservicePath,
      {
        ...body,
        // Pass auth context from token
      },
      token,
      config.timeoutMs || 10000
    );

    // Log the successful routing
    console.log(`[${config.microservicePath}] ✓ Routed to microservice (200 OK)`);

    return new Response(JSON.stringify(serviceResponse), {
      status: 200,
      headers: { ...corsHeaders, "Content-Type": "application/json" },
    });
  } catch (error) {
    const errorMsg = error instanceof Error ? error.message : String(error);

    console.warn(
      `[${config.microservicePath}] Microservice failed (${errorMsg}). Attempting fallback...`
    );

    // Step 2: Fallback to RPC or legacy logic
    if (config.fallbackRpcName) {
      try {
        const { data, error: rpcError } = await supabase.rpc(
          config.fallbackRpcName,
          body
        );

        if (rpcError) throw rpcError;

        console.log(
          `[${config.microservicePath}] ✓ Routed to fallback RPC (${config.fallbackRpcName})`
        );

        return new Response(JSON.stringify(data), {
          status: 200,
          headers: { ...corsHeaders, "Content-Type": "application/json" },
        });
      } catch (rpcError) {
        console.error(
          `[${config.microservicePath}] BOTH microservice and fallback RPC failed:`,
          rpcError
        );
        return new Response(
          JSON.stringify({
            error: "Service unavailable. Try again later.",
          }),
          { status: 503, headers: corsHeaders }
        );
      }
    } else {
      // No fallback available
      console.error(
        `[${config.microservicePath}] Microservice failed and no fallback configured`,
        error
      );
      return new Response(
        JSON.stringify({ error: "Service unavailable. Try again later." }),
        { status: 503, headers: corsHeaders }
      );
    }
  }
}

/**
 * Call a microservice endpoint with timeout and error handling.
 */
async function callMicroservice(
  baseUrl: string,
  path: string,
  body: any,
  token: string,
  timeoutMs: number
): Promise<any> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const response = await fetch(`${baseUrl}${path}`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${token}`,
        "X-Forwarded-By": "edge-wrapper",
      },
      body: JSON.stringify(body),
      signal: controller.signal,
    });

    clearTimeout(timeoutId);

    if (!response.ok) {
      const errorBody = await response.text();
      throw new Error(
        `Microservice returned ${response.status}: ${errorBody}`
      );
    }

    return await response.json();
  } catch (err) {
    clearTimeout(timeoutId);
    throw err;
  }
}

// Export for handler registration
export { handleWrapperRequest };
```

---

## 3. Phase-by-Phase Wrapper Implementation

### Phase 1: Notification Service

#### 3.1.1 Wrapper for `send-invitation`

File: `supabase/functions/send-invitation/index.ts`

```typescript
import { handleWrapperRequest } from "../_shared/gateway-wrapper.ts";
import { corsHeaders } from "../_shared/cors.ts";

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }

  return handleWrapperRequest(req, {
    microserviceUrl: Deno.env.get("NOTIFICATION_SERVICE_URL") ||
      "http://localhost:3001",
    microservicePath: "/api/notifications/send-invitation",
    fallbackRpcName: "send_invitation_legacy",
    timeoutMs: 8000,
  });
});
```

**Microservice Endpoint** (in `notification-service/src/routes/`):

```typescript
// POST /api/notifications/send-invitation
import { Router, Request, Response } from "express"; // or NestJS handlers

export async function sendInvitationHandler(req: Request, res: Response) {
  const { email, organizationId, inviterName, inviteeRole } = req.body;
  const userId = req.user.id; // From JWT middleware

  // Step 1: Validate input
  if (!email || !organizationId) {
    return res.status(400).json({ error: "Missing required fields" });
  }

  // Step 2: Check authorization (tenant_admin can only invite to own org)
  const callerOrg = await getUserOrganization(userId);
  if (callerOrg !== organizationId && !isAdmin(req.user)) {
    return res.status(403).json({ error: "Forbidden" });
  }

  // Step 3: Create invitation record
  const invitation = await db.invitations.create({
    email,
    organizationId,
    invitedBy: userId,
    role: inviteeRole,
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7 days
  });

  // Step 4: Send email (Resend, SendGrid, etc.)
  const emailResult = await emailService.send({
    to: email,
    subject: `You're invited to ${organizationId}`,
    template: "invitation",
    data: {
      inviterName,
      organizationId,
      invitationLink: `${process.env.FRONTEND_URL}/accept-invitation?token=${invitation.token}`,
    },
  });

  // Step 5: Emit event for other services
  await eventBus.publish("invitation.created", {
    invitationId: invitation.id,
    email,
    organizationId,
    role: inviteeRole,
  });

  // Step 6: Return response
  return res.status(201).json({
    invitationId: invitation.id,
    email,
    status: "sent",
    expiresAt: invitation.expiresAt,
  });
}
```

#### 3.1.2 Wrapper for `send-ticket-notification`

File: `supabase/functions/send-ticket-notification/index.ts`

```typescript
import { handleWrapperRequest } from "../_shared/gateway-wrapper.ts";
import { corsHeaders } from "../_shared/cors.ts";

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }

  return handleWrapperRequest(req, {
    microserviceUrl: Deno.env.get("NOTIFICATION_SERVICE_URL") ||
      "http://localhost:3001",
    microservicePath: "/api/notifications/send-ticket",
    fallbackRpcName: "send_ticket_notification_legacy",
    timeoutMs: 5000,
  });
});
```

---

### Phase 2: Gamification Service

#### 3.2.1 Wrapper for `track-milestone`

File: `supabase/functions/track-milestone/index.ts`

```typescript
import { handleWrapperRequest } from "../_shared/gateway-wrapper.ts";
import { corsHeaders } from "../_shared/cors.ts";

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }

  return handleWrapperRequest(req, {
    microserviceUrl: Deno.env.get("GAMIFICATION_SERVICE_URL") ||
      "http://localhost:3002",
    microservicePath: "/api/gamification/milestone/track",
    fallbackRpcName: "track_milestone_legacy",
    timeoutMs: 5000,
  });
});
```

#### 3.2.2 Wrapper for `check-achievements`

File: `supabase/functions/check-achievements/index.ts`

```typescript
import { handleWrapperRequest } from "../_shared/gateway-wrapper.ts";
import { corsHeaders } from "../_shared/cors.ts";

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }

  return handleWrapperRequest(req, {
    microserviceUrl: Deno.env.get("GAMIFICATION_SERVICE_URL") ||
      "http://localhost:3002",
    microservicePath: "/api/gamification/achievements/check",
    fallbackRpcName: "check_achievements_legacy",
    timeoutMs: 5000,
  });
});
```

---

### Phase 3: AI & Personalization Service

#### 3.3.1 Wrapper for `ai-tutor`

File: `supabase/functions/ai-tutor/index.ts`

```typescript
import { handleWrapperRequest } from "../_shared/gateway-wrapper.ts";
import { corsHeaders } from "../_shared/cors.ts";

Deno.serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }

  // Special case: AI tutor returns Server-Sent Events (streaming)
  // Wrapper must NOT buffer the entire response
  return handleStreamingWrapperRequest(req, {
    microserviceUrl: Deno.env.get("AI_SERVICE_URL") ||
      "http://localhost:3003",
    microservicePath: "/api/ai/tutor/chat",
    fallbackRpcName: "ai_tutor_legacy",
    timeoutMs: 30000, // Longer timeout for LLM generation
    isStreaming: true, // Enables streaming mode
  });
});

/**
 * Special wrapper for streaming responses (e.g., SSE from LLM).
 * Proxies chunks without buffering.
 */
async function handleStreamingWrapperRequest(
  req: Request,
  config: any
): Promise<Response> {
  const token = req.headers.get("authorization")?.replace("Bearer ", "");

  if (!token) {
    return new Response(JSON.stringify({ error: "Unauthorized" }), {
      status: 401,
      headers: corsHeaders,
    });
  }

  try {
    const body = await req.json();

    // Call microservice with streaming
    const serviceResponse = await fetch(
      `${config.microserviceUrl}${config.microservicePath}`,
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${token}`,
          "X-Forwarded-By": "edge-wrapper",
        },
        body: JSON.stringify(body),
      }
    );

    if (!serviceResponse.ok) {
      throw new Error(`Service returned ${serviceResponse.status}`);
    }

    // Proxy streaming response with SSE format
    const { readable, writable } = new TransformStream();

    // Copy headers (preserve SSE content-type)
    const responseHeaders = new Headers(serviceResponse.headers);
    responseHeaders.set("Access-Control-Allow-Origin", "*");

    // Stream response body
    if (serviceResponse.body) {
      serviceResponse.body.pipeTo(writable);
    }

    return new Response(readable, {
      status: 200,
      headers: responseHeaders,
    });
  } catch (error) {
    console.error("[ai-tutor] Streaming wrapper error:", error);
    return new Response(
      JSON.stringify({ error: "Service unavailable" }),
      { status: 503, headers: corsHeaders }
    );
  }
}
```

---

## 4. Environment Configuration

### 4.1 Secrets and Configuration

File: `supabase/.env.local` (or Secrets Manager in production)

```bash
# Microservice URLs (update as services are deployed)
NOTIFICATION_SERVICE_URL=https://notification-service.internal:3001
GAMIFICATION_SERVICE_URL=https://gamification-service.internal:3002
AI_SERVICE_URL=https://ai-service.internal:3003
ANALYTICS_SERVICE_URL=https://analytics-service.internal:3004
ASSESSMENT_SERVICE_URL=https://assessment-service.internal:3005
COURSE_SERVICE_URL=https://course-service.internal:3006
ENROLLMENT_SERVICE_URL=https://enrollment-service.internal:3007
IDENTITY_SERVICE_URL=https://identity-service.internal:3008

# Toggles for gradual migration
WRAPPER_ENABLE_NOTIFICATION=true
WRAPPER_ENABLE_GAMIFICATION=false
WRAPPER_ENABLE_AI=false
WRAPPER_ENABLE_ANALYTICS=false
WRAPPER_ENABLE_ASSESSMENT=false
WRAPPER_ENABLE_COURSE=false
WRAPPER_ENABLE_ENROLLMENT=false

# Legacy RPC fallback names
FALLBACK_RPC_SEND_INVITATION=send_invitation_legacy
FALLBACK_RPC_SEND_TICKET=send_ticket_notification_legacy
FALLBACK_RPC_TRACK_MILESTONE=track_milestone_legacy
```

### 4.2 Deno Configuration

File: `deno.json` (in supabase/functions root)

```json
{
  "tasks": {
    "dev": "deno run --allow-all --watch supabase/functions/_shared/server.ts"
  },
  "imports": {
    "std/": "https://deno.land/std@0.203.0/",
    "fresh": "https://deno.land/x/fresh@1.4/",
    "supabase": "https://esm.sh/@supabase/supabase-js@2"
  }
}
```

---

## 5. Monitoring & Observability

### 5.1 Logging Pattern

Each wrapper logs routing decisions:

```typescript
// Structured logging for observability
interface WrapperLog {
  timestamp: string;
  service: string;
  path: string;
  userId: string;
  routedTo: "microservice" | "fallback_rpc" | "error";
  latencyMs: number;
  statusCode: number;
}

function logWrapperDecision(log: WrapperLog) {
  console.log(JSON.stringify({
    level: "INFO",
    service: "edge-wrapper",
    ...log,
  }));
  
  // Also send to observability platform (e.g., Datadog, New Relic)
  // sendMetric("edge_wrapper.routing", log);
}
```

### 5.2 Metrics to Track

- **Routing Distribution**: % requests to microservice vs fallback
- **Latency**: p50, p95, p99 for each service
- **Error Rate**: % failures per service
- **Fallback Success Rate**: When fallback was used, did RPC succeed?

Example dashboard query:

```sql
-- Daily wrapper routing summary (pseudo-SQL)
SELECT
  DATE(timestamp) as day,
  service,
  routedTo,
  COUNT(*) as count,
  AVG(latencyMs) as avg_latency_ms,
  COUNT(CASE WHEN statusCode >= 500 THEN 1 END) / COUNT(*) as error_rate
FROM wrapper_logs
GROUP BY day, service, routedTo
ORDER BY day DESC, service;
```

---

## 6. Migration Checklist per Wrapper

### For Each Wrapper (e.g., `send-invitation`):

1. **Preparation Phase**
   - [ ] Microservice endpoint developed and tested in staging
   - [ ] Microservice contract tests passing
   - [ ] Wrapper code written (using template above)
   - [ ] Edge function converted to wrapper

2. **Staging Validation**
   - [ ] Environment variable set for microservice URL
   - [ ] Wrapper fallback test (microservice down → RPC called)
   - [ ] Latency acceptable (<2x original time)
   - [ ] Error scenarios tested

3. **Canary Deployment**
   - [ ] Deploy wrapper to production
   - [ ] Route 10% of traffic to wrapper (90% still direct RPC)
   - [ ] Monitor error rate and latency for 1 day
   - [ ] If stable, increase to 50%

4. **Full Rollout**
   - [ ] Increase traffic to 100%
   - [ ] Monitor for 1 week
   - [ ] Mark wrapper as "stable"
   - [ ] Schedule legacy RPC for decommissioning (phase final)

5. **Monitoring**
   - [ ] Dashboard created showing routing decisions
   - [ ] Alerts set for fallback failures
   - [ ] Weekly review of wrapper metrics

---

## 7. Rollback Procedure

If a microservice fails in production:

1. **Immediate action** (ops team):
   ```bash
   # Disable wrapper; force fallback
   export WRAPPER_ENABLE_<SERVICE>=false
   # OR manually update function code to skip microservice
   ```

2. **Investigate** (service team):
   - Check service logs for errors
   - Analyze recent code changes
   - Verify DB connectivity

3. **Fix and redeploy**:
   ```bash
   # Fix issue in microservice
   git commit -am "fix: resolve <issue>"
   docker build -t <service>:v1.0.1 .
   kubectl set image deployment/<service> <service>=<service>:v1.0.1
   kubectl rollout status deployment/<service>
   ```

4. **Re-enable wrapper**:
   ```bash
   export WRAPPER_ENABLE_<SERVICE>=true
   # Canary rollout again
   ```

---

## 8. Timeline and Dependencies

```
Phase 1 (Weeks 3–6): Notification
  └─ Wrapper for: send-invitation, send-ticket-notification
  └─ Fallback RPC: send_invitation_legacy, send_ticket_notification_legacy

Phase 2 (Weeks 7–10): Gamification
  └─ Wrapper for: track-milestone, check-achievements
  └─ Fallback RPC: track_milestone_legacy, check_achievements_legacy

Phase 3 (Weeks 11–16): AI & Personalization
  └─ Wrapper for: ai-tutor, generate-ai-roadmap, recommend-content, etc.
  └─ Fallback RPC: ai_tutor_legacy, etc.
  └─ Note: Streaming wrappers require special handling (see 3.3.1)

Phase 4 (Weeks 17–22): Analytics & Engagement
  └─ Wrapper for: generate-learning-summary, generate-nudge
  └─ Fallback RPC: generate_learning_summary_legacy, etc.

Phase 5 (Weeks 23–28): Assessment & Grading
  └─ Wrapper for: grade-assignment, generate-quiz
  └─ Fallback RPC: grade_assignment_legacy, etc.

Phase 6 (Weeks 29–36): Course + Enrollment
  └─ Wrapper for: enhance-content
  └─ Note: Course and enrollment are major services; no direct RPC fallback

Phase 7 (Weeks 37–40): Identity & Decommission
  └─ All edge functions deprecated
  └─ Direct microservice calls from frontend via gateway
```

---

## 9. Example: Complete `send-invitation` Wrapper Flow

### Request Flow

```
┌─ Frontend (React)
│  POST /auth/accept-invitation
│  body: { email, role, orgId }
│  header: Authorization: Bearer <token>
│
└─→ API Gateway
    │  Route to BFF endpoint
    │
    └─→ BFF Layer
        │  POST /api/bff/invitations/send
        │
        └─→ Supabase Edge Function (Wrapper)
            │  send-invitation/index.ts
            │  ├─ Extract token from header
            │  ├─ Parse request body
            │
            └─→ Try: Call Microservice
                │  POST http://notification-service:3001/api/notifications/send-invitation
                │  ├─ Add Authorization header
                │  ├─ Timeout: 8 seconds
                │
                └─→ Success (200)
                    return { invitationId, email, status, expiresAt }
                    [Emit: invitation.created event]

                └──→ Failure (timeout or 5xx)
                    │  Log to observability
                    │
                    └─→ Fallback: Call RPC
                        │  supabase.rpc('send_invitation_legacy', body)
                        │
                        └─→ Success (legacy path)
                            return { invitationId, email, status, expiresAt }
                            [Emit event via trigger]

                        └──→ Failure
                            return 503 Service Unavailable
                            [Alert ops team]
```

### Database State After Success

```sql
-- Notification Service DB
INSERT INTO invitations (id, email, organization_id, invited_by, role, created_at, expires_at)
VALUES ('inv_abc123', 'user@example.com', 'org_xyz', 'user_id_123', 'instructor', NOW(), NOW() + INTERVAL 7 DAYS);

-- Event Bus
PUBLISH "invitation.created" {
  "invitationId": "inv_abc123",
  "email": "user@example.com",
  "organizationId": "org_xyz",
  "role": "instructor",
  "timestamp": "2026-03-29T10:00:00Z"
}

-- Analytics (async)
[Notification Service] → [Analytics Service] (event consumer)
INSERT INTO engagement_events (service, action, user_id, metadata)
VALUES ('notification', 'invitation_sent', 'user_id_123', { inviteEmail: '...' });
```

---

## 10. Troubleshooting Guide

| Symptom | Cause | Solution |
|---------|-------|----------|
| Wrapper returns 503 | Microservice down + RPC failure | Check microservice health; verify RPC exists; check token |
| High latency (5-10s) | Microservice timeout → fallback RPC slow | Increase timeout; profile RPC; consider async fallback |
| Events not emitted | Microservice success but event bus down | Check event bus connectivity; add DLQ for failed publishes |
| Inconsistent data | Dual data in old + new DB | Run reconciliation job; validate with checksums |
| Legacy RPC missing | RPC not created before wrapper deployed | Create fallback RPC; deploy wrapper after |

---

## 11. End of Life: Removing Wrappers

After microservice is stable for 4+ weeks, remove wrapper:

```typescript
// Option A: Delete wrapper function entirely
rm supabase/functions/send-invitation/index.ts

// Option B: Replace with simple redirect (if needed for backward compat)
// POST /send-invitation → redirect to /api/notifications/send-invitation
```

Update frontend to call microservice directly via gateway:

```typescript
// Old path (via wrapper)
const { data } = await supabase.functions.invoke('send-invitation', { body: {...} });

// New path (direct)
const data = await gatewayRequest('/notification/send-invitation', { body: {...} });
```

Archive RPC function:

```sql
-- Rename RPC to indicate deprecation
ALTER FUNCTION send_invitation_legacy RENAME TO send_invitation_legacy_DEPRECATED;
-- Or drop if no longer needed
-- DROP FUNCTION send_invitation_legacy;
```

---

## Summary

This **Gateway Wrapper Plan** enables:

✅ **Gradual migration** of edge functions without big-bang cutover  
✅ **Fallback safety** via legacy RPC paths  
✅ **Observability** of routing decisions  
✅ **Easy rollback** if microservice fails  
✅ **Unknown unknowns handling** with comprehensive error paths  

**Start with Phase 0 (gateway launch) and Phase 1 (notification wrappers) to validate the pattern before scaling to other services.**
