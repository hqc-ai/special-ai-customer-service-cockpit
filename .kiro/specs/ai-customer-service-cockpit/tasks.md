# Implementation Plan: AI Customer Service Cockpit

## Overview

This plan is divided into three sections:

1. **Architecture & Documentation** — repo scaffold, schema, and contracts that everything else builds on (tasks 1–4).
2. **MVP Implementation** — the minimal end-to-end working system: widget, AI loop, guardrails, approval workflow, intervention levels, cockpit, leads, and basic security (tasks 5–22).
3. **Future Production Hardening** — features deferred after MVP is validated: vector search, Redis, PII encryption, push notifications, scaling, CSV export, and advanced PBT (tasks 23–30).

Sub-tasks use decimal notation (e.g., 1.1, 1.2). Items marked with `*` are optional and can be skipped for a faster first pass.

---

## Tasks

### Section 1: Architecture & Documentation

- [ ] 1. Set up monorepo project scaffold
  - [ ] 1.1 Initialize pnpm workspace root with `package.json` and workspace config
    - Create `pnpm-workspace.yaml` listing `packages/*`
    - Add root-level scripts: `dev`, `build`, `test`, `lint`
    - _Requirements: 14.5_
  - [ ] 1.2 Create four package directories with skeleton config files
    - `packages/widget`, `packages/cockpit`, `packages/api-server`, `packages/orchestrator`
    - Each package gets its own `package.json`, `tsconfig.json` (TS packages), or `pyproject.toml` (orchestrator)
    - _Requirements: 14.5_
  - [ ] 1.3 Add Docker Compose for local development
    - Define services: `api-server`, `orchestrator`, `postgres` (no Redis for MVP)
    - Add named volumes for Postgres data persistence
    - Wire environment variables for inter-service URLs and DB connection strings
    - _Requirements: 14.5_
  - [ ]* 1.4 Configure GitHub Actions CI pipeline skeleton
    - Create `.github/workflows/ci.yml` with lint + test jobs for each package
    - _Requirements: 14.5_

- [ ] 2. Define shared types and interfaces
  - [ ] 2.1 Create `packages/api-server/src/types/index.ts` with canonical shared types
    - `DecisionState` interface (all 15 fields from Req 4.1)
    - `ConversationStatus` enum (8 statuses from Req 15.2)
    - `InterventionLevel` type (1–5), `LeadStage`, `GuardrailId`, `AuditEventType`, `MessageActor`, `ApprovalStatus`
    - `VALID_TRANSITIONS` map (state machine from Req 15.2)
    - _Requirements: 4.1, 6.1, 12.1, 15.2_
  - [ ] 2.2 Create Python Pydantic model `packages/orchestrator/src/models/decision_state.py`
    - Mirror the TypeScript `DecisionState` interface as a Pydantic v2 model
    - Add validators for `confidence_level` in [0.0, 1.0] and `lead_stage` enum
    - _Requirements: 4.1, 4.7_

- [ ] 3. Write all PostgreSQL schema migrations
  - [ ] 3.1 Create migration for `visitors` and `admins` tables
    - Use `node-pg-migrate` format; include all columns, constraints, and indexes from the design schema
    - _Requirements: 13.1, 1.3_
  - [ ] 3.2 Create migration for `conversations` and `messages` tables
    - Include `conversation_status` and `message_actor` enum types; add all indexes
    - _Requirements: 15.1, 2.1_
  - [ ] 3.3 Create migration for `decision_states` table
    - Include `UNIQUE (conversation_id, version)` and `confidence_level CHECK (0.0–1.0)`
    - _Requirements: 4.1, 4.5_
  - [ ] 3.4 Create migration for `admin_instructions` and `approval_requests` tables
    - Include `approval_status` enum type and all indexes
    - _Requirements: 6.1, 7.2_
  - [ ] 3.5 Create migration for `leads` table
    - Use `TEXT` for phone and email for MVP; include `UNIQUE (conversation_id)` constraint
    - _Requirements: 10.3_
  - [ ] 3.6 Create migration for `knowledge_base_items` table
    - Use `TEXT` for content; omit `vector` column for MVP; include `kb_status` and `kb_category` enum types
    - _Requirements: 8.1, 8.6_
  - [ ] 3.7 Create migration for `audit_logs` table
    - Include `audit_event_type` enum with all 13 values
    - Add PostgreSQL RULEs to block UPDATE and DELETE (append-only enforcement)
    - Add all three indexes from design
    - _Requirements: 12.1, 12.2_

- [ ] 4. Document API and event contracts
  - [ ] 4.1 Create `docs/api-reference.md` listing all REST endpoints
    - Cover all routes: conversations, messages, approvals, intervention, leads, KB items, auth
    - Include method, path, auth requirement, request/response shape for each
    - _Requirements: 14.4_
  - [ ] 4.2 Document Socket.IO event contract
    - List all events for `/visitor` and `/cockpit` namespaces with payload shapes
    - _Requirements: 2.1, 2.2_

---

### Section 2: MVP Implementation

#### 2-A: API Server Foundation

- [ ] 5. Bootstrap `api-server` (Node.js + Fastify)
  - [ ] 5.1 Create `packages/api-server/src/server.ts` — Fastify app entry point
    - Register JSON body parser, CORS plugin, and error handler; add `GET /health`
    - Wire `packages/api-server/src/db/pool.ts` PostgreSQL connection pool
    - _Requirements: 14.5_
  - [ ] 5.2 Implement JWT authentication middleware
    - Create `packages/api-server/src/middleware/authMiddleware.ts`
    - Validate `Authorization: Bearer <token>` header; extract `adminId`, `role`, `permissions[]`; return 401 on missing or expired token
    - _Requirements: 13.1_
  - [ ] 5.3 Implement RBAC middleware
    - Create `packages/api-server/src/middleware/rbacMiddleware.ts`
    - Export `requirePermission(permission)` factory; return 403 if permission not in token
    - _Requirements: 13.2_
  - [ ] 5.4 Implement input sanitization middleware
    - Create `packages/api-server/src/middleware/inputSanitizer.ts`
    - Strip HTML tags and script content via `sanitize-html`; normalize Unicode (NFC); enforce 4000-char max
    - _Requirements: 13.6_
  - [ ] 5.5 Implement rate-limiting middleware
    - Create `packages/api-server/src/middleware/rateLimiter.ts`
    - 30 messages/minute per `conversationId` on visitor endpoints; 5 attempts/minute per IP on login
    - Return HTTP 429 with `Retry-After` header on violation
    - _Requirements: 13.5_
  - [ ]* 5.6 Write unit tests for auth, RBAC, sanitization, and rate-limiter middleware
    - Expired token → 401; missing permission → 403; script tags stripped; 31st message/min → 429
    - _Requirements: 13.1, 13.2, 13.5, 13.6_

- [ ] 6. Implement admin authentication endpoints
  - [ ] 6.1 Implement `POST /api/v1/auth/login`
    - Verify credentials against `admins` table (bcrypt); issue 15-min access token JWT + 7-day refresh cookie
    - Write login success/failure event to `audit_logs` with IP and timestamp
    - _Requirements: 13.1, 13.8_
  - [ ] 6.2 Implement `POST /api/v1/auth/refresh` and `POST /api/v1/auth/logout`
    - Refresh: validate refresh cookie; issue new access token; return 401 on invalid/expired cookie
    - Logout: require Admin JWT; clear refresh cookie
    - _Requirements: 13.1_
  - [ ]* 6.3 Write unit tests for auth endpoints
    - Correct credentials → token; wrong credentials → 401; expired cookie → 401
    - _Requirements: 13.1, 13.8_

#### 2-B: Conversation & Message REST API

- [ ] 7. Implement conversation management endpoints
  - [ ] 7.1 Implement `POST /api/v1/conversations` (no auth — visitor)
    - Upsert `visitors` row by `anonymous_id`; create `conversations` row with `status = 'ai_handling'`
    - Issue 1-hour `visitorToken` JWT scoped to `conversationId`; return `{ conversationId, visitorToken }`
    - _Requirements: 1.2, 1.3, 15.1_
  - [ ] 7.2 Implement `GET /api/v1/conversations` (Admin JWT)
    - Paginated list with status, visitor_id, created_at, intervention_level; support `?status=` filter
    - _Requirements: 5.1_
  - [ ] 7.3 Implement `GET /api/v1/conversations/:id` (Admin JWT)
    - Return conversation joined with current (max-version) Decision_State
    - _Requirements: 5.3_
  - [ ] 7.4 Implement `PATCH /api/v1/conversations/:id/status` (Admin JWT)
    - Validate transition against `VALID_TRANSITIONS`; reject invalid transitions with HTTP 400
    - Write `conversation_closed` audit event if transitioning to `closed`
    - _Requirements: 15.2, 15.5_
  - [ ] 7.5 Implement `GET /api/v1/conversations/:id/messages` and `GET /api/v1/conversations/:id/audit`
    - Messages: ordered by `created_at`; support `?since=<ISO>` for missed-message sync
    - Audit: return all audit log entries ordered by `created_at`
    - _Requirements: 2.4, 5.10, 12.4_

- [ ] 8. Implement message, admin-note, and Decision_State endpoints
  - [ ] 8.1 Implement `POST /api/v1/conversations/:id/messages` (visitor token, rate-limited)
    - Validate `visitorToken` scoped to `conversationId`; INSERT message row (`actor = 'visitor'`)
    - Write `visitor_message_received` audit event (must persist before returning); forward to Orchestrator
    - _Requirements: 1.4, 12.1, 12.5_
  - [ ] 8.2 Implement `POST /api/v1/conversations/:id/admin-notes` (Admin JWT + `add_admin_notes`)
    - Insert message row (`is_internal = true`); insert `admin_instructions` row; write `admin_note_added` audit event
    - Forward to Orchestrator; update `intervention_level` to `max(current, 2)` and status to `admin_guided` if applicable
    - _Requirements: 5.6, 6.3, 12.1_
  - [ ] 8.3 Implement `POST /api/v1/conversations/:id/messages/admin` (Admin JWT + `activate_takeover`)
    - Only allowed when status = `admin_takeover`; insert message (`actor = 'admin'`, `is_internal = false`)
    - Write audit event; emit `ai:message` socket event to visitor room
    - _Requirements: 6.6, 1.8_
  - [ ] 8.4 Implement Decision_State read endpoints
    - `GET /api/v1/conversations/:id/decision-state` — max-version row
    - `GET /api/v1/conversations/:id/decision-state/history` — all versions ordered ASC
    - _Requirements: 4.1, 4.5_

#### 2-C: State Machine, Audit, and Decision State Services

- [ ] 9. Implement conversation state machine and core services
  - [ ] 9.1 Implement `conversationStateMachine.ts`
    - Create `packages/api-server/src/stateMachine/conversationStateMachine.ts`
    - Encode `VALID_TRANSITIONS` map; export `assertValidTransition(from, to)` and `transitionStatus(conversationId, to, actorId, db)`
    - `transitionStatus` wraps status update + audit log write in a single DB transaction; rolls back entirely if audit log INSERT fails
    - _Requirements: 15.2, 15.3, 15.5_
  - [ ]* 9.2 Write unit tests for the state machine
    - Every valid transition succeeds; every invalid pair throws `InvalidTransitionError`
    - Audit log failure rolls back status change
    - _Requirements: 15.2, 15.3, 15.5_
  - [ ] 9.3 Implement `auditLogService.ts`
    - Create `packages/api-server/src/services/auditLogService.ts`
    - Export `appendAuditEvent(...)` — INSERT only; confirm `rowCount === 1` before resolving; throw on failure
    - _Requirements: 12.1, 12.2, 12.5_
  - [ ] 9.4 Implement `decisionStateService.ts`
    - Atomic update: BEGIN → SELECT MAX(version) FOR UPDATE → INSERT new version → INSERT audit event → COMMIT
    - Only emit socket event after COMMIT; do not emit on failure
    - Export `getCurrentDecisionState(conversationId, db)`
    - _Requirements: 4.2, 4.5_
  - [ ]* 9.5 Write unit tests for `decisionStateService`
    - Each call increments version by 1; DB failure on audit INSERT rolls back transaction
    - _Requirements: 4.2, 4.5_

#### 2-D: Real-time Layer (Socket.IO)

- [ ] 10. Implement Socket.IO real-time layer
  - [ ] 10.1 Set up Socket.IO server on `api-server`
    - Attach Socket.IO to the Fastify HTTP server; configure `/visitor` and `/cockpit` namespaces
    - Add auth middleware to each namespace: `/visitor` validates `visitorToken`; `/cockpit` validates Admin JWT
    - _Requirements: 2.1_
  - [ ] 10.2 Implement `/visitor` namespace handlers
    - Create `packages/api-server/src/socket/visitorNamespace.ts`
    - On connect: join room `conv:{conversationId}`; handle `visitor:message` event
    - Expose `emitToVisitor(conversationId, event, payload)` helper
    - _Requirements: 2.6_
  - [ ] 10.3 Implement `/cockpit` namespace handlers
    - Create `packages/api-server/src/socket/cockpitNamespace.ts`
    - On connect: record admin as observer; update conversation status to `admin_observing` if currently `ai_handling`
    - Expose `emitToCockpit(conversationId, event, payload)` helper
    - _Requirements: 5.2, 6.2_
  - [ ] 10.4 Wire Decision_State push to cockpit on update
    - After `decisionStateService` commits, call `emitToCockpit(conversationId, 'decision_state:updated', state)`
    - Retry 3× in 5-second window; emit `decision_state:stale` if all attempts fail
    - _Requirements: 2.2, 4.2_

#### 2-E: Orchestrator (Python + LangGraph)

- [ ] 11. Bootstrap `orchestrator` and LangGraph graph
  - [ ] 11.1 Create `packages/orchestrator/src/main.py` FastAPI entry point
    - Register routers for `/orchestrate/message`, `/orchestrate/admin-note`, `/orchestrate/approval-decision`, `/orchestrate/takeover`, `/orchestrate/assist`
    - Configure asyncpg PostgreSQL connection
    - _Requirements: 3.1_
  - [ ] 11.2 Implement LangGraph `orchestrator_graph.py`
    - Define `StateGraph` nodes: `receive_message → classify_intent → retrieve_knowledge → check_guardrails → generate_response → post_generation_guardrails → route_response`
    - Route edges: `GUARDRAIL_VIOLATED → handle_violation`, `REQUIRES_APPROVAL → create_approval_request`, `SAFE → send_response`
    - Add `update_decision_state → update_audit_log` tail nodes
    - _Requirements: 3.1, 3.2, 3.3_
  - [ ] 11.3 Implement `classify_intent` and `retrieve_knowledge` nodes
    - `classify_intent`: call OpenAI JSON-mode to classify intent; update `Decision_State.intent`
    - `retrieve_knowledge` (MVP): use PostgreSQL `to_tsvector` / `plainto_tsquery` full-text search on approved KB items; return top-5 results
    - _Requirements: 3.1, 8.2, 8.7_

- [ ] 12. Implement guardrail engine in Orchestrator
  - [ ] 12.1 Implement `patterns.py` and `guardrail_engine.py`
    - Define `GUARDRAIL_PATTERNS` dict with compiled regex lists for GUARDRAIL-REG, GUARDRAIL-LEGAL, GUARDRAIL-AVAILABILITY, GUARDRAIL-PRICING, GUARDRAIL-PAYMENT
    - `check_all_guardrails(text)` returns list of all triggered guardrail IDs
    - GUARDRAIL-FABRICATION: string containment check against retrieved KB context documents
    - _Requirements: 9.1–9.6_
  - [ ] 12.2 Implement `sensitive_topics.py`
    - Keyword match on visitor message for: legal dispute, formal complaint, medical advice, financial advice
    - If matched: skip LLM call; route to `create_approval_request`
    - _Requirements: 9.7_
  - [ ]* 12.3 Write unit tests for guardrail patterns and engine
    - Each `GUARDRAIL-*` pattern tested with known-bad and known-safe inputs
    - Sensitive topic detection fires on known trigger phrases
    - _Requirements: 9.1–9.7_


- [ ] 13. Implement response generation and approval request creation
  - [ ] 13.1 Implement `generate_response` LangGraph node
    - Compose prompt: system prompt + KB context + conversation history + admin notes + Decision_State
    - Call OpenAI GPT-4o with JSON-mode structured output; parse response text and action category
    - _Requirements: 3.2, 3.3_
  - [ ] 13.2 Implement `create_approval_request` LangGraph node
    - INSERT into `approval_requests` with proposed text, action category, and Decision_State snapshot
    - Set Redis-less MVP TTL via a DB `created_at` + cron check (no Redis required for MVP)
    - Return `{ approvalRequestId, requiresApproval: true }` to api-server
    - _Requirements: 7.2, 3.7_
  - [ ] 13.3 Implement `handle_violation` LangGraph node
    - Build escalation message for visitor ("This topic requires admin assistance")
    - Update Decision_State: add triggered guardrail to `risk_flags`, add action to `restricted_actions`
    - Return escalation text as the response
    - _Requirements: 3.4, 9.1–9.6_
  - [ ] 13.4 Implement `/orchestrate/admin-note` and `/orchestrate/approval-decision` endpoints
    - `admin-note`: inject note into conversation context; return updated Decision_State fields
    - `approval-decision`: approve/edit/reject; return `{ responseText?, auditEvent }`
    - _Requirements: 5.6, 7.4, 7.5, 7.6_
  - [ ] 13.5 Implement `/orchestrate/takeover` and `/orchestrate/assist` endpoints
    - `takeover`: activate or release admin control; return Decision_State diff and audit event
    - `assist`: run intent classification + KB retrieval only; return suggestion text for admin
    - _Requirements: 6.6, 3.9_

#### 2-F: Approval Workflow and Notification

- [ ] 14. Implement approval workflow endpoints and basic notifications
  - [ ] 14.1 Implement `GET /api/v1/conversations/:id/approvals`
    - Return all approval requests for the conversation ordered by `created_at`
    - _Requirements: 7.2_
  - [ ] 14.2 Implement `POST /api/v1/approvals/:id/approve` (Admin JWT + `approve_requests`)
    - Accept `{ editedResponse? }`; call Orchestrator `/orchestrate/approval-decision`
    - Deliver response to visitor via socket; log approval event in audit log after confirmed delivery
    - Retry delivery up to 3× at 10-second intervals; log delivery failure if all retries exhausted
    - _Requirements: 7.4, 7.5_
  - [ ] 14.3 Implement `POST /api/v1/approvals/:id/reject` (Admin JWT + `approve_requests`)
    - Call Orchestrator with `decision = 'reject'` and rejection reason
    - Log `approval_request_rejected` audit event separately from approval events
    - _Requirements: 7.6_
  - [ ] 14.4 Implement approval timeout enforcement (cron, no Redis)
    - Add a PostgreSQL cron job or `setInterval` task that queries `approval_requests` where `status = 'pending'` and `created_at < now() - interval '15 minutes'`
    - Auto-reject each expired request: update status to `auto_rejected`, write audit event with reason `"timeout"`, notify AI via Orchestrator
    - Send escalation alert socket event to cockpit at 5-minute mark
    - _Requirements: 7.8, 7.9_
  - [ ] 14.5 Implement in-app notification delivery via Socket.IO
    - Emit `notification:new_conversation` within 5s of conversation creation
    - Emit `approval:created` within 10s of approval request creation
    - Emit `approval:alert` escalation at 2-minute and 5-minute marks for unacknowledged conversations
    - _Requirements: 11.1, 11.2, 11.4_

#### 2-G: Admin Intervention Levels

- [ ] 15. Implement intervention level management
  - [ ] 15.1 Implement `POST /api/v1/conversations/:id/intervention` (Admin JWT)
    - Accept `{ level: 1–5, constraint?: string }`; validate level is 1–5
    - Apply max-of-levels rule when multiple admins are active (query active sessions)
    - Write `decision_constraint_applied` audit event if level = 3; write intervention audit event for all changes
    - _Requirements: 6.1, 6.4, 6.9, 6.10_
  - [ ] 15.2 Implement `POST /api/v1/conversations/:id/takeover` (Admin JWT + `activate_takeover`)
    - `action: 'activate'`: transition status to `admin_takeover`; suspend Decision_Constraints (mark suspended, do not remove); cancel pending approvals; notify AI via Orchestrator
    - `action: 'release'`: present confirmation response; restore suspended constraints; transition status back to `ai_handling`; write audit event
    - _Requirements: 6.6, 6.8, 7.7_
  - [ ]* 15.3 Write unit tests for intervention level enforcement
    - Concurrent admins at different levels → effective level = max; Admin_Note never lowers level below 2
    - Level 5 activate → constraints suspended; Level 5 release → constraints restored intact
    - _Requirements: 6.1, 6.3, 6.4_

#### 2-H: Chat Widget (Frontend)

- [ ] 16. Build Chat_Widget (Vanilla TypeScript + Shadow DOM)
  - [ ] 16.1 Implement `packages/widget/src/index.ts` entry point
    - Read `data-site`, `data-primary-color`, `data-position` from script element
    - Create `<div id="csw-root">` appended to `document.body`; attach Shadow DOM; apply CSS custom properties from config
    - _Requirements: 1.1, 14.1, 14.2, 14.3_
  - [ ] 16.2 Implement `LauncherButton` and `ChatPanel` components
    - Launcher: floating button (bottom-right by default); click opens ChatPanel
    - ChatPanel: message list area, text input, send button, typing indicator area
    - _Requirements: 1.4, 1.7_
  - [ ] 16.3 Implement `VisitorSocket.ts` and `ReconnectManager.ts`
    - Connect to Socket.IO `/visitor` namespace with `visitorToken`
    - Implement exponential backoff: base 1s, doubles per attempt, max 30s, max 10 attempts
    - On reconnect: call `GET /messages?since=:lastReceivedAt`; display missed messages before accepting new ones
    - After 10 failed attempts: stop retrying, show error message, display manual retry button
    - _Requirements: 2.3, 2.4, 2.7_
  - [ ] 16.4 Implement `conversationsApi.ts` contact info collection and format validation
    - `POST /conversations` on widget open; store `conversationId` + `visitorToken` in `localStorage`
    - Validate phone (E.164-like) and email (RFC 5322 simple regex) before accepting input
    - _Requirements: 1.2, 1.5_
  - [ ] 16.5 Implement conversation closure handling
    - On `conversation:closed` socket event: display closure message; disable input field immediately
    - _Requirements: 1.9_
  - [ ]* 16.6 Write unit tests for widget
    - Message displayed within 500ms of send; typing indicator shown/hidden correctly; input disabled on close
    - Email/phone validation rejects bad formats; admin takeover — visitor cannot distinguish sender
    - _Requirements: 1.4, 1.5, 1.8, 1.9_

#### 2-I: Admin Cockpit (React SPA)

- [ ] 17. Build Admin Cockpit SPA (React 18 + Vite)
  - [ ] 17.1 Bootstrap React app with routing and auth guard
    - Configure React Router: `/login`, `/cockpit` (protected), `/cockpit/:conversationId`
    - Auth guard: redirect to `/login` if no valid access token
    - _Requirements: 5.1, 13.1_
  - [ ] 17.2 Implement Conversation List page
    - Subscribe to `conversation:new` and `conversation:status` socket events; update list in real time without page refresh
    - Display: status badge (8 labels), visitor identity (if collected), elapsed time
    - _Requirements: 5.1, 5.2, 5.4_
  - [ ] 17.3 Implement `useCockpitSocket` hook
    - Connect to Socket.IO `/cockpit` namespace with Admin JWT on mount
    - Subscribe to: `conversation:new`, `conversation:status`, `decision_state:updated`, `approval:created`, `approval:alert`, `decision_state:stale`
    - _Requirements: 2.2, 5.2, 5.5_
  - [ ] 17.4 Implement Conversation Detail page — Window A (Chat Transcript)
    - Render `MessageList` (all public messages); show typing indicator on AI responses
    - Show `AdminNoteInput` (always visible); show `AdminMessageInput` only when status = `admin_takeover`
    - Handle `decision_state:stale` — show stale indicator in Window B header
    - _Requirements: 5.3, 5.6, 6.7_
  - [ ] 17.5 Implement Window B — J Space panel
    - Render all 15 Decision_State fields as structured card layout
    - Display `confidence_level` as progress bar (0.0–1.0)
    - Show active `risk_flags` with colour-coded badges; show `restricted_actions` list
    - Auto-refresh on `decision_state:updated` socket event without Admin interaction
    - _Requirements: 4.1, 4.3, 5.3, 5.5_
  - [ ] 17.6 Implement Approval Panel in Window B
    - List pending Approval_Requests with proposed response text, action category
    - Approve (sends as-is), Edit + Approve (text area), Reject (requires reason), Takeover button
    - After approval/rejection: remove from panel; update conversation status badge
    - _Requirements: 5.7, 7.4, 7.5, 7.6, 7.7_
  - [ ] 17.7 Implement Lead Info panel and Audit Log tab
    - Lead Info: show visitor name, phone, email, company, service_interest, lead_stage; hide contact fields if Admin lacks `view_contact_data` permission
    - Audit Log tab: chronological list of all audit events for the selected conversation
    - _Requirements: 5.9, 5.10, 5.11_
  - [ ] 17.8 Implement Intervention Controls — Takeover button and return-to-AI flow
    - Takeover button: calls `POST /conversations/:id/takeover { action: 'activate' }`; button changes to "Return to AI"
    - Return to AI: show confirmation modal; on confirm: call `POST /conversations/:id/takeover { action: 'release' }`
    - _Requirements: 5.8, 6.8_
  - [ ] 17.9 Implement `useRbac` hook and RBAC-gated UI elements
    - Parse `permissions[]` from decoded JWT; expose `hasPermission(p)` predicate
    - Gate: Approval Panel → `approve_requests`; Takeover → `activate_takeover`; KB management → `manage_knowledge_base`; contact fields → `view_contact_data`
    - _Requirements: 5.11, 13.2_

#### 2-J: Knowledge Base Management

- [ ] 18. Implement Knowledge Base API and seeding
  - [ ] 18.1 Implement KB CRUD endpoints (Admin JWT + `manage_knowledge_base`)
    - `POST /api/v1/kb/items` — create item with status `draft`
    - `PATCH /api/v1/kb/items/:id` — update content/category; reset status to `draft`
    - `POST /api/v1/kb/items/:id/publish` — set status to `approved`; record `approved_by` and `approved_at`
    - `DELETE /api/v1/kb/items/:id` — set status to `archived` (no hard delete)
    - `GET /api/v1/kb/items` — list all items with status filter
    - _Requirements: 8.1, 8.2, 8.5, 8.6_
  - [ ] 18.2 Create seed file with initial 5SOffice KB content
    - One approved item per category: service_descriptions, faqs, pricing_policy, contract_invoice_policy, deposit_booking_policy, escalation_rules, legal_disclaimers
    - _Requirements: 8.1_

#### 2-K: Lead Management

- [ ] 19. Implement lead management
  - [ ] 19.1 Implement lead auto-creation in Orchestrator
    - When Visitor provides name, phone, or email in conversation: call `POST /api/v1/leads` from Orchestrator
    - If Lead already exists for this `conversation_id`: PATCH existing record (no duplicate)
    - _Requirements: 10.1, 10.4_
  - [ ] 19.2 Implement Lead REST endpoints
    - `GET /api/v1/leads` with filters: `?lead_stage=`, `?date_from=`, `?date_to=`
    - `GET /api/v1/leads/:id` — full lead detail
    - `PATCH /api/v1/leads/:id` — update `admin_status` and `lead_stage`
    - _Requirements: 10.3, 10.6_
  - [ ] 19.3 Implement lead status review on conversation close
    - On `PATCH /conversations/:id/status` to `closed`: prompt in cockpit UI (5-minute auto-finalize with last known `lead_stage`)
    - _Requirements: 10.5_

#### 2-L: Inactivity Timeout and Conversation Lifecycle

- [ ] 20. Implement conversation inactivity auto-close
  - [ ] 20.1 Implement inactivity timeout background task
    - Add `setInterval` (every 5 minutes) in `api-server` that queries conversations where `status NOT IN ('closed', 'converted_to_lead', 'follow_up_needed')` and last message `created_at < now() - interval '$INACTIVITY_TIMEOUT_SECONDS seconds'`
    - For each: call `transitionStatus(id, 'closed', 'system', db)`; emit `conversation:closed` to visitor socket
    - Default `INACTIVITY_TIMEOUT_SECONDS = 1800` (30 min), configurable via env var
    - _Requirements: 15.4_

#### 2-M: Basic Security (MVP-required)

- [ ] 21. Apply MVP-required security measures
  - [ ] 21.1 Enforce TLS in production
    - Configure Fastify to redirect HTTP → HTTPS; add `Strict-Transport-Security` header
    - Widget: only transmit data over WSS when host page is HTTPS (check `window.location.protocol`)
    - _Requirements: 1.10, 13.4_
  - [ ] 21.2 Secure JWT configuration
    - Use HS256 with a secret from environment variable (minimum 32 chars); set `exp` on all tokens
    - Admin access token: 15 min; visitor token: 1 hour; refresh token cookie: `HttpOnly; Secure; SameSite=Strict`
    - _Requirements: 13.1_

#### 2-N: Integration Smoke Tests

- [ ]* 22. Write end-to-end integration smoke tests
  - [ ]* 22.1 Happy-path chat flow
    - Visitor opens widget → AI responds → Decision_State appears in cockpit → audit log contains both events
    - _Requirements: 1.2, 3.1, 4.2, 12.1_
  - [ ]* 22.2 Guardrail enforcement smoke test
    - Visitor asks for price confirmation → GUARDRAIL-PRICING fires → escalation message sent → risk_flag in Decision_State
    - _Requirements: 9.4, 3.4_
  - [ ]* 22.3 Approval workflow smoke test
    - AI generates approval-gated response → Approval_Request created → Admin approves → message delivered → audit log updated
    - _Requirements: 7.1, 7.2, 7.4_
  - [ ]* 22.4 Admin takeover smoke test
    - Admin activates takeover → AI stops responding → Admin message reaches visitor → return-to-AI restores constraints
    - _Requirements: 6.6, 6.8_
  - [ ]* 22.5 Inactivity timeout smoke test
    - Conversation idle for configured timeout → auto-closed → visitor socket receives `conversation:closed`
    - _Requirements: 15.4_

---

### Section 3: Future Production Hardening

- [ ] 23. Replace full-text KB search with pgvector semantic search
  - Add `vector(1536)` column to `knowledge_base_items`; generate embeddings on publish via OpenAI `text-embedding-3-small`
  - Replace `to_tsvector` retrieval with cosine similarity query using configurable threshold (default 0.70)
  - Add IVFFlat index; implement 0.90 confidence threshold for escalation offer
  - _Requirements: 8.3, 8.7, 8.8_

- [ ] 24. Add Redis for session cache, pub/sub, and approval TTL
  - Replace `setInterval` cron with Redis keyspace TTL notifications for approval auto-reject
  - Add `@socket.io/redis-adapter` for multi-node Socket.IO fan-out
  - Use Redis as session store for visitor tokens
  - _Requirements: 2.5, 7.8, 7.9_

- [ ] 25. Encrypt PII at rest with pgcrypto
  - Add `pgcrypto` extension; migrate `phone` and `email` columns in `leads` to `BYTEA` using `pgp_sym_encrypt`
  - Store encryption key in secrets manager (never in code or `.env` committed to version control)
  - Gate decryption on `view_contact_data` RBAC permission
  - _Requirements: 13.7_

- [ ] 26. Add browser push notifications (Web Push API)
  - Integrate `web-push` library; store VAPID public/private key pair in secrets
  - Store admin push subscriptions in `admins` table; deliver notifications when tab is not in focus
  - _Requirements: 11.3, 11.6_

- [ ] 27. Add Nginx reverse proxy and production rate limiting
  - Configure Nginx TLS termination, HTTP → HTTPS redirect, and rate limiting (30 msg/min per visitor, 5/min login)
  - Replace Fastify-level rate limiter with Nginx `limit_req_zone`
  - _Requirements: 13.5_

- [ ] 28. Add property-based tests (Hypothesis + fast-check)
  - Implement all 20 correctness properties defined in design.md using Hypothesis (Python) and fast-check (TypeScript)
  - Minimum 100 iterations per property; tag each test with the property ID it verifies
  - _Requirements: All_

- [ ] 29. Add CSV lead export
  - Implement `GET /api/v1/leads/export` for `system_owner` role
  - Stream CSV with columns: visitor_id, name, phone, email, company, service_interest, lead_stage, admin_status, conversation_id, created_at
  - Support filters: `?lead_stage=`, `?date_from=`, `?date_to=`, `?service_interest=`
  - _Requirements: 10.7_

- [ ] 30. Add Zalo OA and Telegram/Slack admin notifications
  - Implement webhook receiver for Zalo OA messages; route into Conversation model
  - Add Telegram bot and/or Slack webhook for admin alerts (new conversation, approval needed)
  - _Requirements: Future phase_

---

## Task Dependency Graph

```json
{
  "waves": [
    {
      "wave": 1,
      "tasks": [1, 2, 3, 4],
      "description": "Architecture & Documentation — scaffold, types, schema, API contracts"
    },
    {
      "wave": 2,
      "tasks": [5, 11],
      "description": "Bootstrap — api-server foundation and orchestrator bootstrap run in parallel"
    },
    {
      "wave": 3,
      "tasks": [6, 7, 8, 9, 12, 16],
      "description": "Core services — auth, conversation API, message API, state machine, guardrail engine, chat widget (all parallelizable after wave 2)"
    },
    {
      "wave": 4,
      "tasks": [10, 13, 18],
      "description": "Real-time layer, response generation & approval creation, KB endpoints — depends on wave 3"
    },
    {
      "wave": 5,
      "tasks": [14, 15, 17, 19, 20],
      "description": "Approval workflow, intervention levels, cockpit SPA, lead management, inactivity timeout — depends on wave 4"
    },
    {
      "wave": 6,
      "tasks": [21],
      "description": "MVP security hardening — depends on auth (task 6) and widget (task 16)"
    },
    {
      "wave": 7,
      "tasks": [22],
      "description": "Integration smoke tests — depends on all MVP tasks (5–21) complete"
    },
    {
      "wave": 8,
      "tasks": [23, 24, 25, 26, 27, 28, 29, 30],
      "description": "Future production hardening — all tasks in this wave are independent of each other and can be done in any order after wave 7"
    }
  ]
}
```

---

## Notes

- **Do not implement tasks 23–30 until MVP (tasks 1–22) is deployed and validated** in a staging environment.
- Tasks marked `*` are optional for the first deployment pass; they improve confidence but are not blockers.
- **Nginx** (task 27): not required for MVP. Fastify's built-in rate limiting and TLS (task 21) are sufficient for early staging. Add Nginx before production traffic.
- **Redis** (task 24): not required for MVP. The `setInterval`-based approval timeout and single-node Socket.IO are adequate for MVP scale (< 50 concurrent sessions).
- **pgvector** (task 23): not required for MVP. PostgreSQL full-text search (`to_tsvector`) is sufficient until the KB grows beyond ~500 items or semantic accuracy becomes a user-visible problem.
- **PII encryption** (task 25): deferred to post-MVP. Store phone/email as plain `TEXT` in MVP; migrate to `pgcrypto` before handling real customer data at scale.
- **Property-based testing** (task 28): deferred to post-MVP. Unit tests (tasks 9.2, 9.5, 12.3, 15.3, 16.6) provide sufficient coverage for the MVP release.
- The `INACTIVITY_TIMEOUT_SECONDS` environment variable defaults to `1800` (30 minutes). Override per environment as needed.
- All secrets (JWT secret, OpenAI API key, encryption key) must be stored in environment variables or a secrets manager — never committed to version control.
- The widget is fully isolated via Shadow DOM and must not pollute the host page's global `window` namespace beyond the `window.__csw` initialization key.
