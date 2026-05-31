# Orion Agent Integration — Implementation Plan

**Date:** 2026-05-29  
**Status:** Plan complete, Hermes skill created, ready for Orion implementation  
**Repo:** `orion-scheduler-assistant` (at `/srv/hermes/orion-scheduler-assistant` or wherever it lives)  
**Hermes skill:** Already created at `~/.hermes/skills/productivity/orion-agent/SKILL.md` — no action needed from Krio/Codex

---

## 1. Overview

Add a `POST /agent/task` endpoint to Orion's backend so Hermes (Lyra) can create tasks in Hammad's Orion schedule. The task lands in the sync store and flows to the mobile app on the next sync pull.

**Auth model:** A single shared secret (`ORION_AGENT_API_KEY`) set on both Orion's backend and in Hermes' config. No per-user tokens needed — the endpoint takes a `userEmail` field to scope the task to the right Orion account.

---

## 2. Architecture

```
Hermes (Lyra)
    │
    │  POST /agent/task
    │  Header: X-Orion-Agent-Key: <secret>
    │  Body: { userEmail, title, ... }
    ▼
Orion Backend (services/api)
    │
    │  1. Validate X-Orion-Agent-Key against ORION_AGENT_API_KEY
    │  2. Parse request body (userEmail + title required)
    │  3. Look up user by email via AuthStore.findUserByEmail()
    │  4. Build a Task object with crypto.randomUUID()
    │  5. Push into SyncStore.push() with deviceId="hermes-agent"
    │  6. Return { ok: true, task } or { ok: false, error }
    ▼
Sync Store (SQLite: sync_tasks table)
    │
    │  Mobile app pulls on next sync
    ▼
Orion Mobile App — task appears in Inbox/Today
```

---

## 3. Files to Create

### 3.1 `packages/shared/src/agent.ts` (NEW)

This is the shared contract — types and a parser for the agent task creation request. The backend imports from here, and Hermes' skill can reference these types for documentation.

```typescript
import type { Task } from "./productivity";

/** Fields a Hermes agent can supply when creating a task.
 *  Only title is required; everything else has sensible defaults. */
export type AgentCreateTaskRequest = {
  /** Orion user email to scope the task to. */
  userEmail: string;
  /** Task title. Required. */
  title: string;
  /** Optional longer description / notes. */
  description?: string | null;
  /** Priority: 1 (P1/highest) – 4 (P4/lowest). Defaults to 3. */
  priority?: number;
  /** ISO date string e.g. "2026-05-30". Defaults to null (no due date). */
  dueDate?: string | null;
  /** HH:MM time string e.g. "14:00". Defaults to null. */
  dueTime?: string | null;
  /** ISO date string for deadline. Defaults to null. */
  deadlineDate?: string | null;
  /** Estimated duration in minutes. Defaults to null. */
  durationMinutes?: number | null;
  /** Project ID to assign the task to. Defaults to null (inbox). */
  projectId?: string | null;
  /** Label names. Defaults to []. */
  labels?: string[];
};

export type AgentCreateTaskResponse =
  | { ok: true; task: Task }
  | { ok: false; error: string };

export function parseAgentCreateTaskRequest(
  value: unknown
): AgentCreateTaskRequest | null {
  if (!isRecord(value)) return null;

  const userEmail = asTrimmedString(value.userEmail);
  const title = asTrimmedString(value.title);
  if (!userEmail || !title) return null;

  const description = asNullableString(value.description) ?? null;
  const priority = clampPriority(asNumber(value.priority));
  const dueDate = asNullableDateString(value.dueDate) ?? null;
  const dueTime = asNullableTimeString(value.dueTime) ?? null;
  const deadlineDate = asNullableDateString(value.deadlineDate) ?? null;
  const durationMinutes = asNullablePositiveInt(value.durationMinutes) ?? null;
  const projectId = asNullableString(value.projectId) ?? null;
  const labels = normalizeStringArray(value.labels);

  return {
    userEmail,
    title,
    description,
    priority,
    dueDate,
    dueTime,
    deadlineDate,
    durationMinutes,
    projectId,
    labels,
  };
}

// -- helpers (keep these private to this module) ---------------------------

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null;
}

function asTrimmedString(value: unknown): string | null {
  if (typeof value !== "string") return null;
  const trimmed = value.trim();
  return trimmed.length > 0 ? trimmed : null;
}

function asNullableString(value: unknown): string | null {
  if (value === null || value === undefined) return null;
  return typeof value === "string" ? value : null;
}

function asNullableDateString(value: unknown): string | null {
  if (typeof value !== "string") return null;
  if (/^\d{4}-\d{2}-\d{2}$/.test(value.trim())) return value.trim();
  return null;
}

function asNullableTimeString(value: unknown): string | null {
  if (typeof value !== "string") return null;
  if (/^\d{2}:\d{2}(:\d{2})?$/.test(value.trim())) return value.trim();
  return null;
}

function asNumber(value: unknown): number | null {
  if (typeof value !== "number" || !Number.isFinite(value)) return null;
  return value;
}

function asNullablePositiveInt(value: unknown): number | null {
  const n = asNumber(value);
  if (n === null || n < 1 || !Number.isInteger(n)) return null;
  return n;
}

function clampPriority(value: number | null): number {
  if (value === null) return 3;
  const clamped = Math.round(value);
  if (clamped < 1) return 1;
  if (clamped > 4) return 4;
  return clamped;
}

function normalizeStringArray(value: unknown): string[] {
  if (!Array.isArray(value)) return [];
  return value
    .filter(
      (item): item is string =>
        typeof item === "string" && item.trim().length > 0
    )
    .map((item) => item.trim());
}
```

### 3.2 `services/api/src/agentService.ts` (NEW)

Business logic: resolves user by email, builds a Task, pushes into the sync store.

```typescript
import { randomUUID } from "node:crypto";
import {
  type AgentCreateTaskRequest,
  type AgentCreateTaskResponse,
  type Task,
  type TaskPriority,
} from "@orion/shared";
import { getSyncStore } from "./syncStore";
import { getAuthStore } from "./authStore";

/**
 * Creates a task in the Orion sync store on behalf of a Hermes agent.
 *
 * The task flows to the user's mobile app on the next sync pull.
 *
 * Requires `ORION_AGENT_API_KEY` to be set and match the request header.
 * The `userEmail` in the request identifies which Orion user the task
 * belongs to.
 */
export function createAgentTask(
  request: AgentCreateTaskRequest
): AgentCreateTaskResponse {
  const user = getAuthStore().findUserByEmail(request.userEmail);
  if (!user) {
    return {
      ok: false,
      error: `No Orion user found with email "${request.userEmail}".`,
    };
  }

  const now = new Date().toISOString();
  const task = buildTask(request, now);

  getSyncStore().push({
    userId: user.id,
    deviceId: "hermes-agent",
    changes: {
      tasks: [task],
      projects: [],
      labels: [],
      settings: [],
      activity: [
        {
          id: randomUUID(),
          entityType: "task",
          entityId: task.id,
          action: "created_by_agent",
          payload: JSON.stringify({ source: "hermes" }),
          createdAt: now,
          updatedAt: now,
        },
      ],
      tombstones: [],
    },
  });

  return { ok: true, task };
}

function buildTask(
  request: AgentCreateTaskRequest,
  now: string
): Task {
  const priority = (request.priority ?? 3) as TaskPriority;

  return {
    id: randomUUID(),
    title: request.title,
    description: request.description ?? null,
    completed: false,
    projectId: request.projectId ?? null,
    sectionId: null,
    parentTaskId: null,
    labels: request.labels ?? [],
    priority,
    dueDate: request.dueDate ?? null,
    dueTime: request.dueTime ?? null,
    deadlineDate: request.deadlineDate ?? null,
    durationMinutes: request.durationMinutes ?? null,
    recurrenceRule: null,
    createdAt: now,
    updatedAt: now,
    completedAt: null,
    sortOrder: 0,
  };
}
```

---

## 4. Files to Modify

### 4.1 `packages/shared/src/index.ts`

Add one line at the end, before the closing (or wherever the exports are):

```typescript
export * from "./agent";
```

Full file after edit:

```typescript
export * from "./productivity";
export * from "./dateHelpers";
export * from "./scheduling";
export * from "./planning";
export * from "./calendarPlanning";
export * from "./voiceDump";
export * from "./aiAssist";
export * from "./sync";
export * from "./syncMerge";
export * from "./auth";
export * from "./calendar";
export * from "./agent";   // <-- ADD THIS LINE
```

### 4.2 `services/api/src/authStore.ts`

The `findUserByEmail` method is currently `private`. We need a **public** version for `agentService.ts` to call.

**Step A:** Rename the private method from `findUserByEmail` to `findUserRowByEmail` (it returns the raw DB row, not the clean `AuthUser` type). Find this line (~line 250):

```typescript
  private findUserByEmail(email: string): AuthUserRow | null {
```

Change `findUserByEmail` → `findUserRowByEmail`.

**Step B:** Update the two internal callers inside `signup()` and `login()` to use the new name:

In `signup()` (~line 99):
```typescript
// CHANGE THIS:
    const existing = this.findUserByEmail(normalizedEmail);
// TO:
    const existing = this.findUserRowByEmail(normalizedEmail);
```

In `login()` (~line 129):
```typescript
// CHANGE THIS:
    const user = this.findUserByEmail(normalizedEmail);
// TO:
    const user = this.findUserRowByEmail(normalizedEmail);
```

**Step C:** Add the new **public** method right after `logout()` (after ~line 189, before `authenticateAccessToken`):

```typescript
  findUserByEmail(email: string): AuthUser | null {
    const normalizedEmail = normalizeAuthEmail(email);
    const row = this.findUserRowByEmail(normalizedEmail);
    if (!row) return null;
    return {
      id: row.id,
      email: normalizeAuthEmail(row.email),
      createdAt: row.created_at,
    };
  }
```

This returns the clean `AuthUser` type (imported from `@orion/shared`), not the raw DB row.

### 4.3 `services/api/src/server.ts`

Three changes:

**A) Add imports** at the top of the file:

Find the `@orion/shared` import block (~lines 3-36) and add these two items:
```typescript
  parseAgentCreateTaskRequest,
  type AgentCreateTaskResponse
```

Find the local imports (~line 65, after `import { loadApiEnvFile } from "./envLoader";`) and add:
```typescript
import { createAgentTask } from "./agentService";
```

**B) Add the `/agent/task` route handler** — insert right after the `/calendar/events` route block (after its closing `}`), before the `sendJson(res, 404, { error: "Not found." });` line:

```typescript
  if (requestUrl.pathname === "/agent/task" && req.method === "POST") {
    try {
      const agentKey = getAgentApiKey(req);
      if (!agentKey || !isValidAgentKey(agentKey)) {
        sendJson(res, 401, { error: "Missing or invalid agent API key." });
        return;
      }
      const body = await readJsonBody(req);
      const parsed = parseAgentCreateTaskRequest(body);
      if (!parsed) {
        sendJson(res, 400, {
          error: "Body must include a valid userEmail and title."
        });
        return;
      }
      const responsePayload: AgentCreateTaskResponse = createAgentTask(parsed);
      if (responsePayload.ok) {
        sendJson(res, 201, responsePayload);
      } else {
        const status = responsePayload.error.includes("No Orion user")
          ? 404
          : 400;
        sendJson(res, status, responsePayload);
      }
      return;
    } catch (error) {
      if (error instanceof RequestValidationError) {
        sendJson(res, error.statusCode, { error: error.message });
        return;
      }
      const message = error instanceof Error ? error.message : "Unexpected server error.";
      sendJson(res, 500, { error: message });
      return;
    }
  }
```

**C) Add helper functions** at the bottom of the file (after `escapeHtml`):

```typescript
function getAgentApiKey(req: IncomingMessage): string | null {
  const header = req.headers["x-orion-agent-key"];
  if (!header || typeof header !== "string") return null;
  const trimmed = header.trim();
  return trimmed.length > 0 ? trimmed : null;
}

function isValidAgentKey(key: string): boolean {
  const configured = process.env.ORION_AGENT_API_KEY?.trim();
  if (!configured) return false;
  return key === configured;
}
```

**D) Update CORS headers** to allow the custom header. Find `setCorsHeaders` (~line 731) and change:

```typescript
// CHANGE THIS:
  res.setHeader("access-control-allow-headers", "content-type, authorization");
// TO:
  res.setHeader("access-control-allow-headers", "content-type, authorization, x-orion-agent-key");
```

### 4.4 `services/api/.env.example`

Add at the end of the file:

```
# Hermes agent integration: API key for Hermes → Orion task creation.
# Set a long random value and configure it in Hermes as well.
ORION_AGENT_API_KEY=
```

---

## 5. Environment Variables

| Variable | Where | Purpose |
|---|---|---|
| `ORION_AGENT_API_KEY` | `services/api/.env` | Shared secret. Generate with: `node -e "console.log(require('crypto').randomBytes(32).toString('base64url'))"` |
| `EXPO_PUBLIC_API_BASE_URL` | `apps/mobile/.env` | Already exists. Hermes needs to know the Orion backend URL. |

---

## 6. Hermes Skill — `orion-agent`

This skill **already exists** at `~/.hermes/skills/productivity/orion-agent/SKILL.md`. It was created by Lyra during the planning session. Krio/Codex should NOT recreate it — it's ready to use. Load it via `skill_view(name='orion-agent')`.

### 6.1 What the Skill Tells Lyra

The skill gives Lyra everything needed to turn natural-language requests ("add X to Orion for Friday") into API calls:

- **Trigger phrases** — "add to Orion", "schedule this", "create task", "remind me"
- **Endpoint** — `POST {ORION_API_BASE_URL}/agent/task`
- **Auth** — `X-Orion-Agent-Key` header (NOT bearer token)
- **Request schema** — all 10 fields with types, defaults, and validation rules
- **Response handling** — 201/400/401/404/500, what each means, how to recover
- **Date parsing** — `date -I -d "tomorrow"` for converting natural language to ISO dates
- **Priority mapping** — "urgent"→1, "high"→2, "medium"→3, "low"→4
- **Curl recipe** — exact working command pattern
- **Pitfalls** — 7 documented mistakes (wrong auth header, date format, stale sync, etc.)
- **Config** — three values from `~/.hermes/config.yaml` or env vars

### 6.2 Request Lifecycle (what Lyra does)

```
User says: "Add 'Review Q3 deck' to Orion for Friday at 2pm, high priority"
    │
    ▼
1. Lyra loads orion-agent skill (skill_view)
    │
    ▼
2. Gathers config: ORION_API_BASE_URL, ORION_AGENT_API_KEY, ORION_USER_EMAIL
   If any missing → asks user once
    │
    ▼
3. Parses natural language:
   title     = "Review Q3 deck"
   dueDate   = date -I -d "friday"        → "2026-06-05"
   dueTime   = "2pm"                      → "14:00"
   priority  = "high"                     → 2
    │
    ▼
4. Builds JSON body:
   {
     "userEmail": "hammad@...",
     "title": "Review Q3 deck",
     "priority": 2,
     "dueDate": "2026-06-05",
     "dueTime": "14:00"
   }
  (Only includes fields the user specified — no extra nulls)
    │
    ▼
5. Calls via terminal():
   curl -s -X POST "${ORION_API_BASE_URL}/agent/task" \
     -H "Content-Type: application/json" \
     -H "X-Orion-Agent-Key: ${ORION_AGENT_API_KEY}" \
     -d '{"userEmail":"hammad@...","title":"Review Q3 deck","priority":2,...}'
    │
    ▼
6. Handles response:
   201 → "✅ Created: 'Review Q3 deck' (P2, due Fri Jun 5 at 14:00). 
          Task ID: a1b2c3... — it'll appear in Orion on next sync."
   401 → "The agent API key doesn't match. Check ORION_AGENT_API_KEY 
          in Orion's .env and Hermes' config match."
   404 → "No Orion user with that email. Have you signed up in the 
          Orion app yet?"
   400 → "Missing required fields — I need at least a title."
   500 → "Orion backend error. Check the API logs."
```

### 6.3 Curl Recipe (what Lyra actually runs)

```bash
curl -s -X POST "${ORION_API_BASE_URL}/agent/task" \
  -H "Content-Type: application/json" \
  -H "X-Orion-Agent-Key: ${ORION_AGENT_API_KEY}" \
  -d '{
    "userEmail": "'"${ORION_USER_EMAIL}"'",
    "title": "Review Q3 roadmap deck",
    "priority": 2,
    "dueDate": "2026-06-05",
    "dueTime": "14:00"
  }' | jq .
```

`| jq .` is optional but helps with readable output. If `jq` is not available, Lyra parses the raw JSON.

### 6.4 Date Parsing Reference

Lyra uses `date` CLI to convert natural language:

```bash
date -I                     # → 2026-05-29 (today)
date -I -d "tomorrow"       # → 2026-05-30
date -I -d "friday"         # → 2026-06-05 (this Friday)
date -I -d "next monday"    # → 2026-06-08
date -I -d "2 weeks"        # → 2026-06-12
date -I -d "2026-12-25"     # → 2026-12-25 (already ISO, passthrough)
```

### 6.5 Config Location

The skill reads three values. In `~/.hermes/config.yaml`:

```yaml
orion:
  api_base_url: "http://localhost:3000"
  agent_api_key: "orion-agent-secret-change-me"
  user_email: "hammad@example.com"
```

Or as environment variables: `ORION_API_BASE_URL`, `ORION_AGENT_API_KEY`, `ORION_USER_EMAIL`.

### 6.6 Error Recovery Table

| Error | HTTP | Meaning | Lyra's action |
|-------|------|---------|---------------|
| Missing/invalid API key | 401 | `ORION_AGENT_API_KEY` doesn't match | Tell user to verify keys match on both sides |
| Bad body | 400 | `userEmail` or `title` missing | Fix the JSON — Lyra's bug, retry |
| User not found | 404 | Email doesn't match any Orion account | Ask user to verify email or sign up in Orion |
| Connection refused | — | Orion backend not running | Tell user to run `npm run api` in Orion repo |
| Parse error | — | Response wasn't valid JSON | Report raw output, suggest backend may be down |
| Network timeout | — | Backend unreachable | Check URL, network, suggest trying again |

### 6.7 What Lyra Should NOT Do

- ❌ Use `Authorization: Bearer` — this endpoint uses `X-Orion-Agent-Key`
- ❌ Send dates like "tomorrow" or "Friday" in JSON — always `YYYY-MM-DD`
- ❌ Retry on 401 (bad key) or 404 (unknown email) without user input
- ❌ Create tasks without confirming with the user (unless they said "just do it")
- ❌ Read tasks from Orion — READ is not implemented yet, only CREATE

---

## 7. Data Flow (ASCII)

```
┌─────────────┐     POST /agent/task      ┌──────────────────┐
│  Hermes     │ ──────────────────────────▶│  Orion Backend   │
│  (Lyra)     │   X-Orion-Agent-Key        │  (services/api)  │
│             │   {userEmail, title, ...}  │                  │
└─────────────┘                            │  1. validate key │
                                           │  2. parse body   │
                                           │  3. findUser by  │
                                           │     email         │
                                           │  4. build Task    │
                                           │  5. syncStore     │
                                           │     .push()       │
                                           └────────┬─────────┘
                                                    │
                                           ┌────────▼─────────┐
                                           │   Sync Store     │
                                           │   (SQLite)       │
                                           │   sync_tasks     │
                                           │   sync_changes   │
                                           └────────┬─────────┘
                                                    │ GET /sync/pull
                                           ┌────────▼─────────┐
                                           │  Orion Mobile    │
                                           │  App (Expo)      │
                                           │  Task appears    │
                                           │  in Inbox/Today  │
                                           └──────────────────┘
```

---

## 8. Testing Checklist

After implementing, verify:

- [ ] `npm run typecheck` passes (all workspaces)
- [ ] `npm run lint` passes
- [ ] `npm run test` passes
- [ ] Start backend: `npm run api`
- [ ] Set `ORION_AGENT_API_KEY` in `services/api/.env` (generate a random key)
- [ ] **No key → 401:** `curl -X POST localhost:3000/agent/task -H 'Content-Type: application/json' -d '{"userEmail":"test@test.com","title":"test"}'` returns 401
- [ ] **Wrong key → 401:** Same but with `-H 'X-Orion-Agent-Key: wrong'`
- [ ] **Missing fields → 400:** With valid key, send `{}` or `{"userEmail":"x"}`
- [ ] **Unknown user → 404:** With valid key, send `{"userEmail":"nobody@nowhere.com","title":"test"}`
- [ ] **Happy path → 201:** Create a real Orion user first (sign up via mobile), then send a task for that email with valid key
- [ ] **Sync pull delivers task:** After creating a task via agent, pull sync for that user (`GET /sync/pull` with bearer auth) — task should appear
- [ ] **Mobile app shows task:** After sync pull, task should appear in Orion mobile app

---

## 9. Design Decisions & Edge Cases

| Decision | Rationale |
|---|---|
| API key auth (not bearer) | Hermes isn't an Orion user — it's a service. No user session to manage. |
| `userEmail` in body (not header/deduced) | Explicit. Hermes knows whose schedule it's writing to. |
| Resolving user by email | `AuthStore` already has email lookups. No new DB schema needed. |
| `deviceId = "hermes-agent"` | Consistent attribution in activity logs. The mobile app sees where tasks came from. |
| Priority defaults to 3 (P3) | Neutral. Hermes can override. |
| No recurrence support in agent API | Keep MVP simple. Recurrence can be added later if needed. |
| No label creation — names only | Labels passed as string names. If Orion supports label resolution later, this API stays forward-compatible. For MVP, labels are just stored on the task. |
| Activity log entry `created_by_agent` | Distinguishable from user-created tasks in activity history. |
| 201 for created, 404 for unknown user | Follows REST semantics. Unknown email is "not found", bad body is "bad request". |

---

## 10. Hermes Configuration

After the Orion backend is deployed, configure Hermes to talk to it. Add to `~/.hermes/config.yaml` or environment:

```yaml
# In Hermes config — exact location depends on Hermes setup
orion:
  api_base_url: "http://localhost:3000"   # or deployed URL
  agent_api_key: "<generated-secret>"     # matches ORION_AGENT_API_KEY in Orion
  user_email: "hammad@example.com"        # Hammad's Orion account email
```

Then the `orion-agent` skill (see section 6) will use these values.
