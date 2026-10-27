# ARCHITECTURE.md — AyuDevices Doctor Outreach & Relationship Automation Platform

**Document version:** 1.0  
**Companion specification:** `PRD.md`  
**Architecture status:** Proposed target architecture for a new build  
**Primary deployment shape:** Modular monolith + independent background worker processes  
**Primary channel:** WhatsApp through the official WhatsApp Business Platform/Cloud API or a vetted provider adapter

---

## 1. Purpose and Architecture Principles

This document describes how the system is divided into components, how requests and events move through it, which component owns each responsibility, and how the system preserves consent, auditability, and reliable campaign execution.

The architecture supports this end-to-end workflow:

1. A user creates/imports doctors and records communication permission.
2. The system maps a doctor’s specialty to a configured, approved pipeline.
3. An operator reviews the audience, content, schedule, and provider requirements.
4. An authorized user launches the campaign.
5. Persistent workers execute steps when due.
6. Each outbound send passes the same policy and recipient-eligibility gate.
7. Provider results and incoming messages are persisted and deduplicated.
8. Replies, opt-outs, failures, or human intervention can pause or cancel future steps.
9. The dashboard presents current progress and an auditable history.

### 1.1 Non-negotiable principles

- **PostgreSQL is the source of truth.** Redis is used for job transport, caching, and short-lived coordination—not as the authoritative store for campaigns, consent, conversations, or message outcomes.
- **Modular monolith first.** Keep business rules in one backend codebase with explicit module boundaries. Run API and worker processes separately. Split services only when measured operational needs justify it.
- **One outbound gate.** Direct sends, campaigns, scheduled steps, follow-ups, and AI-assisted replies must call the same eligibility and policy service.
- **Persist state before external side effects.** Create durable execution/message-attempt records before calling a messaging provider.
- **Do not assume exactly-once delivery.** Use idempotency where supported and track uncertain outcomes; never blindly resend an uncertain request.
- **Events are at-least-once.** Webhooks and background jobs may be delivered more than once. Handlers must be idempotent.
- **Consent always wins.** A queued job is not permission to send. Re-check consent and other send rules immediately before the provider request.
- **AI is advisory.** AI may classify, summarize, retrieve facts, or draft content; application code enforces permissions and validates every proposed action.
- **Provider-neutral core.** Keep provider-specific request formats, statuses, limits, and capabilities behind adapters.
- **Fail closed and explain why.** Missing policy data, uncertain recipient identity, an unavailable consent check, or invalid campaign configuration blocks sending and gives the operator an actionable reason.
- **No stealth or evasion mechanisms.** Do not implement number rotation, CAPTCHA bypass, or technical controls intended to evade provider enforcement.

---

## 2. System Context

### 2.1 People and external systems

| Actor / system | Responsibility | Interaction with AyuDevices |
|---|---|---|
| Owner / administrator | Configures users, content, provider, policies, campaigns, and emergency stop | Web application |
| Sales operator | Manages doctors, reviews conversations, runs authorized campaigns, and schedules follow-ups | Web application |
| Reviewer / manager | Approves restricted templates, campaign launches, and sensitive responses | Web application |
| Doctor / recipient | Receives permitted messages and replies through WhatsApp | WhatsApp client |
| WhatsApp provider | Accepts outbound requests, emits inbound messages and status events where supported | Provider adapter + verified webhooks |
| Object storage | Stores approved media and uploaded documents | Storage adapter |
| Optional AI provider / Ollama | Produces classifications, summaries, and draft replies when enabled | AI adapter; never bypasses policy layer |
| Optional external CRM | Synchronizes approved fields when configured in a later phase | Integration adapter |

### 2.2 System boundary

AyuDevices owns doctor records, consent evidence, specialty mappings, approved content snapshots, pipeline versions, campaign execution state, tasks, conversation metadata, review decisions, and audit history.

The WhatsApp provider owns its delivery infrastructure and provider-specific message identifiers/status events. AyuDevices must not present a provider submission as confirmed delivery unless the provider explicitly reports delivery.

AI models do not own authoritative pricing, offer validity, consent, permissions, or execution decisions. The relational database and application services remain authoritative.

---

## 3. High-Level Component Diagram

```mermaid
flowchart TB
    User[Owner / Sales Operator / Reviewer] --> Web[React + TypeScript Web App]
    Web --> API[FastAPI Application]

    API --> Auth[Auth & RBAC]
    API --> CRM[Doctor CRM & Consent]
    API --> Content[Templates, Media & Pipeline Builder]
    API --> Campaign[Campaign & Enrollment Service]
    API --> Inbox[Conversation & Follow-up Service]
    API --> Analytics[Reporting & Audit]

    CRM --> PG[(PostgreSQL)]
    Content --> PG
    Campaign --> PG
    Inbox --> PG
    Analytics --> PG
    Content --> Storage[(Object Storage)]

    Campaign --> Queue[(Redis / Durable Job Queue)]
    Queue --> Worker[Campaign / Scheduler Workers]
    Worker --> Policy[Outbound Eligibility + Policy Gate]
    Policy --> PG
    Worker --> Messaging[Messaging Service]
    Messaging --> Adapter[WhatsApp Provider Adapter]
    Adapter --> WA[WhatsApp Business Platform / Approved Provider]
    WA --> Webhook[Verified Webhook Endpoint]
    Webhook --> PG
    Webhook --> Events[Webhook / Event Worker]
    Events --> Inbox
    Events --> Policy

    Inbox -. P1 optional .-> AI[AI Service Adapter]
    AI --> KB[Approved Knowledge Retrieval]
    KB --> PG
    KB --> Storage

    API --> Audit[Audit Event Service]
    Worker --> Audit
    Webhook --> Audit
```

The diagram is logical. Modules can live in the same Python package and deployment image while retaining these boundaries. PostgreSQL remains authoritative; Redis is not a substitute for durable application state.

---

## 4. Recommended Technology Stack

Use the stack specified by `PRD.md` unless a written architecture decision record (ADR) justifies a change.

| Layer | Choice | Notes |
|---|---|---|
| Web UI | React + TypeScript; choose either Vite or Next.js | Do not maintain two frontends. Use a consistent component system, forms, validation, and query/cache strategy. |
| API | Python + FastAPI | Typed request/response schemas, OpenAPI, service-layer business logic. |
| Database | PostgreSQL | Authoritative transactional storage, foreign keys, unique constraints, and migrations. |
| Job queue | Celery + Redis, or an equivalent durable queue | Use persistent task state in PostgreSQL; queue messages are wake-up signals, not the only record of work. |
| Scheduler | Persistent scheduler that creates due jobs | Must recover due work after restart; avoid relying on in-process timers. |
| Media | Storage interface; local filesystem for development, object storage for production | Store file bytes outside PostgreSQL where practical, with metadata/checksum in the database. |
| WhatsApp | Provider-neutral adapter; official WhatsApp Business Platform/Cloud API or vetted provider | Select an initial provider before implementation. Do not pretend every provider has identical capabilities. |
| Authentication | Secure server-managed sessions or documented JWT strategy | Prefer secure HttpOnly cookies for a same-origin web app; include CSRF protections as applicable. |
| AI (P1) | Adapter to optional Ollama/local model or approved hosted provider | Draft-only by default, per-intent authorization, strict schemas, timeouts, and source grounding. |
| Local development | Docker Compose | API, worker, scheduler, PostgreSQL, Redis, and optional local storage/AI service. |
| Production | Dockerized services with managed PostgreSQL/Redis and object storage where available | Separate API, worker, scheduler, and webhook ingress roles for independent operations. |

### 4.1 Avoid premature complexity

Do not begin with microservices, Kubernetes, multiple databases, or a custom workflow platform unless deployment requirements demonstrate the need. Use interfaces around provider, storage, AI, and queue integrations to preserve future options.

---

## 5. Runtime Components and Responsibilities

### 5.1 Web application

Responsible for:

- Doctor CRUD, import preview, duplicate resolution, and specialty assignment.
- Consent evidence entry and display of block reasons.
- Template/media library and pipeline editing.
- Campaign audience preview, approval, launch, pause/resume/cancel, and progress.
- Inbox, conversation assignment, human takeover, and follow-up tasks.
- Review queue, audit search, analytics, settings, and system health.

The browser must never hold WhatsApp access tokens, database credentials, webhook secrets, or object-storage master credentials. It must not call the provider directly for outbound sending.

### 5.2 FastAPI application

The API layer should be intentionally thin:

1. Authenticate the user or service.
2. Authorize the requested action.
3. Validate and normalize input.
4. Call a domain service.
5. Return a typed response or a stable error code.

API route handlers must not contain campaign scheduling algorithms, eligibility rules, provider-specific transport logic, or complex state transitions. Those belong in testable service modules.

### 5.3 Domain services

| Module / service | Owns | Must not own |
|---|---|---|
| `auth` | Login/session lifecycle, user status, roles and permission checks | Campaign business rules |
| `doctors` | Doctor profiles, normalization, duplicate handling, import/export | Consent decision by itself |
| `specialties` | Specialty dictionary and default-pipeline mapping | Editing a live enrollment’s pinned version silently |
| `consent` | Consent events, current eligibility projection, opt-out and suppression | Sending messages directly |
| `templates` | Template content, variables, language, approval and version snapshots | Provider delivery status |
| `media` | Upload validation, file metadata, storage references and lifecycle | Campaign authorization |
| `pipelines` | Pipeline configuration, validation, versioning and preview | Sending messages directly |
| `campaigns` | Campaign setup, audience snapshot, approvals, enrollments, pause/resume/cancel | Direct provider calls |
| `scheduler` | Finding and enqueueing due work; job recovery | Policy bypass or source-of-truth state only in Redis |
| `execution` | One pipeline step’s state transition and orchestration | Unvalidated arbitrary AI actions |
| `policy` | Central outbound eligibility and policy decision | UI-specific behavior |
| `messaging` | Provider-neutral send request, message attempts, outcome normalization | Overriding consent or campaign state |
| `providers.whatsapp` | Provider API calls, provider ID mapping, capability metadata | Doctor/pipeline domain decisions |
| `webhooks` | Signature validation, event persistence, deduplication and dispatch | Long AI tasks in the HTTP request lifecycle |
| `conversations` | Thread metadata, inbound/outbound timeline, assignments and human takeover | Claiming delivery beyond provider evidence |
| `followups` | Task lifecycle and due scheduling | Sending without the shared execution gate |
| `audit` | Append-only important-operation trail | Storing secrets or unnecessary full payloads |
| `analytics` | Read models and aggregated reports | Changing transactional business state |
| `knowledge` (P1) | Approved facts/documents and retrieval | Source of truth for structured price/offer validity |
| `ai_assistance` (P1) | Intent, summary, draft, structured action proposal | Direct arbitrary side effects or self-granted permissions |
| `integrations` (P2) | External CRM/calendar/sheets adapters | Bypassing core consent or audit services |

### 5.4 Background processes

Run separate process roles from the same codebase:

- **API process:** authenticated CRUD and orchestration endpoints.
- **Campaign worker:** processes a due pipeline step and handles controlled retries.
- **Scheduler process:** periodically finds due steps/tasks and enqueues them persistently.
- **Webhook ingress:** validates provider signatures and stores incoming events quickly. It may be the same FastAPI app with isolated endpoint limits initially.
- **Webhook/event worker:** processes stored inbound/status events asynchronously.
- **Maintenance worker:** cleanup, scheduled summaries, reconciliation, and retention jobs as required.

Do not create a separate worker pool for every feature in the MVP. Separate queue routing by task type when it improves isolation or operations.

---

## 6. Logical Module and Repository Layout

One reasonable starting layout is:

```text
ayudevices/
├── apps/
│   └── web/                         # React + TypeScript
├── backend/
│   ├── app/
│   │   ├── main.py                  # FastAPI app factory / router assembly
│   │   ├── api/
│   │   │   ├── dependencies.py
│   │   │   ├── errors.py
│   │   │   └── v1/                   # Versioned route modules
│   │   ├── core/                     # Settings, logging, auth primitives
│   │   ├── db/                       # Session, models, repositories, migrations config
│   │   ├── modules/
│   │   │   ├── auth/
│   │   │   ├── users/
│   │   │   ├── doctors/
│   │   │   ├── specialties/
│   │   │   ├── consent/
│   │   │   ├── templates/
│   │   │   ├── media/
│   │   │   ├── pipelines/
│   │   │   ├── campaigns/
│   │   │   ├── execution/
│   │   │   ├── messaging/
│   │   │   │   └── providers/
│   │   │   │       └── whatsapp/
│   │   │   ├── webhooks/
│   │   │   ├── conversations/
│   │   │   ├── followups/
│   │   │   ├── audit/
│   │   │   ├── analytics/
│   │   │   ├── knowledge/            # P1
│   │   │   ├── ai_assistance/         # P1
│   │   │   └── integrations/          # P2
│   │   ├── workers/                  # Task definitions / queue setup
│   │   └── tests/
│   │       ├── unit/
│   │       ├── integration/
│   │       └── e2e/
│   ├── alembic/                     # Reviewed database migrations
│   └── pyproject.toml
├── infra/
│   ├── docker/
│   ├── compose.yaml
│   └── deployment/                  # Environment templates/runbooks; no secrets
├── docs/
│   ├── ADRs/
│   └── runbooks/
├── .env.example
├── PRD.md
└── ARCHITECTURE.md
```

This is a proposed organization, not a requirement to create empty directories for every future module on day one. Create modules as their phase begins, while keeping current boundaries clear.

Within each backend module, prefer a structure such as:

```text
module_name/
├── router.py          # API boundary, if needed
├── schemas.py         # Input/output validation models
├── models.py          # ORM/database entities when appropriate
├── repository.py      # Data access and query composition
├── service.py         # Business rules and use cases
├── policies.py        # Module-local validation/policy helpers when useful
└── tests/
```

Avoid circular imports. Share cross-cutting policies through explicit interfaces rather than calling another module’s internal ORM functions directly.

---

## 7. Data Architecture

### 7.1 PostgreSQL as the source of truth

Use migrations for all schema changes. Store authoritative records and immutable history in PostgreSQL. Apply database constraints in addition to application-level validation.

Core entity groups:

- **Identity:** `users`, roles/permissions, sessions or refresh-token records as appropriate.
- **CRM:** `doctors`, `specialties`, tags, relationship stages, import batches and import rows.
- **Consent:** current eligibility projection and append-only `consent_events`.
- **Content:** `message_templates`, `template_versions`, `media_assets`.
- **Automation config:** `pipelines`, `pipeline_versions`, versioned pipeline steps.
- **Execution:** `campaigns`, `enrollments`, step execution records, scheduled jobs, follow-up tasks.
- **Messaging:** `conversations`, `messages`, `message_attempts`, inbound webhook/event records.
- **Operations:** `audit_events`, provider configurations/references, health snapshots/alerts if persisted.
- **P1 intelligence:** `knowledge_items`, document/chunk metadata, review items and AI action proposals.

### 7.2 Important relationships

```mermaid
erDiagram
    SPECIALTY ||--o{ DOCTOR : classifies
    SPECIALTY o|--o{ PIPELINE : defaults_to
    DOCTOR ||--o{ CONSENT_EVENT : has
    PIPELINE ||--o{ PIPELINE_VERSION : versions
    PIPELINE_VERSION ||--|{ PIPELINE_STEP : contains
    MESSAGE_TEMPLATE ||--o{ TEMPLATE_VERSION : versions
    TEMPLATE_VERSION o|--o{ PIPELINE_STEP : used_by
    CAMPAIGN }o--|| PIPELINE_VERSION : pins
    CAMPAIGN ||--o{ ENROLLMENT : includes
    DOCTOR ||--o{ ENROLLMENT : enrolled_in
    ENROLLMENT ||--o{ STEP_EXECUTION : progresses_through
    DOCTOR ||--o{ CONVERSATION : has
    CONVERSATION ||--o{ MESSAGE : contains
    MESSAGE ||--o{ MESSAGE_ATTEMPT : attempts
    DOCTOR ||--o{ FOLLOW_UP_TASK : receives
    CONVERSATION o|--o{ FOLLOW_UP_TASK : contextualizes
    USER ||--o{ AUDIT_EVENT : performs
```

Names shown in this diagram are logical; use `PRD.md` section 7 as the complete minimum entity specification.

### 7.3 Pipeline configuration vs execution state

Keep configuration separate from runtime state:

- `Pipeline` is the mutable editorial object.
- `PipelineVersion` is an immutable approved snapshot.
- `PipelineStep` belongs to a specific version and describes intended behavior.
- `Campaign` pins one approved version and an audience definition/snapshot.
- `Enrollment` represents one doctor’s participation in the campaign.
- `StepExecution` represents one actual or planned step for one enrollment, including state, due time, attempts, and result.

Do not store per-doctor progress by mutating the pipeline definition. Editing a pipeline creates a new version. Existing enrollments stay on the version they started with unless a specifically authorized migration is performed.

### 7.4 Message content snapshots

Before a message is submitted to a provider, persist the final rendered body and the exact asset references/content version used. Store template/version identifiers alongside the immutable rendered snapshot. A later template edit must not rewrite message history.

### 7.5 Media storage

Store media bytes in a storage backend and metadata in PostgreSQL. Metadata should include MIME type, byte size, checksum, upload owner, scan/validation state, storage key, and archive state.

Do not expose raw object-store paths as authorization. Use signed, short-lived download/upload URLs or backend-mediated file access with role checks. Validate extensions and actual content type; impose configurable file-size limits; scan uploads where the selected environment supports it.

### 7.6 Indexing and deletion

Index fields used for doctor search, status filtering, due jobs, campaign progress, provider IDs, event deduplication, and chronological timelines. Preserve suppression/opt-out data as required to prevent accidental re-contact. Implement a documented retention and deletion policy instead of hard-deleting related records without analysis.

---

## 8. Campaign Execution Architecture

### 8.1 Durable execution model

Treat PostgreSQL state as the durable workflow ledger. The queue only schedules work. Every work item should identify an execution record and be safe to deliver more than once.

Suggested unique execution identity:

`step_execution_id` + one attempt idempotency key for the provider operation.

Use database transactions, unique constraints, and row-level locking or equivalent concurrency control to prevent two workers claiming the same due step. Redis locks may reduce contention but must not be the only correctness mechanism.

### 8.2 Scheduling loop

1. Scheduler queries for due `pending`/`ready` step executions whose `next_due_at <= now` and whose campaign/enrollment is still active.
2. It claims records in a short transaction and enqueues identifiers, not full private message payloads.
3. Worker loads the record and current authoritative related state.
4. Worker validates campaign, enrollment, pinned pipeline version, quiet hours, follow-up limits, asset/template state, global emergency stop, and required provider capabilities.
5. Worker obtains a fresh eligibility decision immediately before the outbound attempt.
6. Worker persists a `Message` and `MessageAttempt` in a pre-send state, with an idempotency key and correlation ID.
7. Worker calls the `MessagingService` and provider adapter.
8. Worker persists the observed result and advances the pipeline state only according to the result semantics.
9. The next step is scheduled only if the current result and stop conditions permit it.
10. The dashboard reads persisted state and displays blocked/failed/uncertain reasons.

The implementation may combine steps 6–8 in service calls, but the persisted ordering and failure semantics must remain intact.

### 8.3 Execution state transitions

```mermaid
stateDiagram-v2
    [*] --> pending
    pending --> ready: due and campaign active
    ready --> running: worker claims atomically
    running --> completed: policy passed and step completed
    running --> blocked: consent/policy/configuration blocks send
    running --> failed: definite failure
    running --> uncertain: request outcome ambiguous
    running --> waiting: awaiting provider status or configured delay
    waiting --> ready: next action is due
    pending --> cancelled: campaign/enrollment cancelled
    ready --> cancelled: campaign/enrollment cancelled
    pending --> skipped: condition no longer applies
    blocked --> pending: authorized resolution + fresh validation
    failed --> pending: authorized retry after classification
    uncertain --> waiting: reconciliation required
```

Only permit transitions through a state-transition service. Every manual retry, unblocking action, or override must capture actor and reason.

### 8.4 Retries and uncertain results

- Retry only failures classified as safe/transient by the provider adapter and configured policy.
- Use bounded exponential backoff with jitter for suitable transient failures, respecting provider guidance and current policy.
- Do not retry invalid recipients, missing consent, rejected content, revoked templates, or other permanent policy/configuration failures until corrected.
- If the request may have reached the provider but confirmation is missing, set `uncertain` and reconcile against provider events/API where supported. Do not blindly send a second copy.
- If provider idempotency is supported, pass a stable idempotency key. If it is not supported, document the residual duplicate risk and require reconciliation for ambiguous results.
- Pause the execution queue when the provider connection is unhealthy or repeated errors breach a configured circuit-breaker threshold.

### 8.5 Pause, resume, and cancel semantics

- Pausing a campaign stops new execution claims for that campaign; an already-running provider request may not be cancellable.
- Cancelling a campaign marks unstarted steps cancelled and prevents new claims. Do not claim that an already-submitted message was recalled.
- Pausing one enrollment affects only that doctor’s future steps.
- Human takeover pauses configured automation for the conversation and cancels or defers relevant future follow-ups.
- Resume requires a fresh check of campaign status, doctor eligibility, time windows, and content validity.
- Global emergency stop blocks all new outbound provider calls regardless of queued tasks.

---

## 9. Outbound Eligibility and Policy Gate

This is a critical architectural component, not a frontend validation detail.

### 9.1 All outbound paths must converge here

```mermaid
flowchart TD
    Request[Direct send / campaign step / follow-up / AI-approved reply] --> AuthZ[Authorization check]
    AuthZ --> Global{Emergency stop active?}
    Global -- Yes --> Block[Block + reason + audit]
    Global -- No --> Doctor[Resolve doctor and recipient]
    Doctor --> Consent[Read current consent and suppression state]
    Consent --> Eligible{Eligible for this channel, purpose and content?}
    Eligible -- No --> Block
    Eligible -- Yes --> Campaign[Check campaign/enrollment/conversation state]
    Campaign --> Content[Validate approved template, version, assets and facts]
    Content --> Window[Validate provider messaging window/template rules]
    Window --> Limits[Validate business hours and frequency/follow-up limits]
    Limits --> Provider[Persist attempt then call provider adapter]
    Provider --> Result[Persist outcome and audit]
```

### 9.2 Minimum gate inputs

- Authenticated actor or authorized worker identity.
- Doctor ID and normalized target number.
- Channel and communication purpose.
- Current consent status and evidence reference where required.
- Opt-out/suppression status and relevant event timestamp.
- Campaign/enrollment/conversation status.
- Approved content/version and provider-template status if applicable.
- Provider session health and capability requirements.
- Applicable messaging-window rules.
- Configured business hours and frequency limits.
- Global emergency-stop state.
- Idempotency/execution identity.

### 9.3 Gate behavior

- Return a structured decision: `allowed` or `blocked`, a stable reason code, and safe operator-facing detail.
- Re-evaluate against the authoritative database at execution time.
- Fail closed when required state cannot be checked.
- Record blocked events and their reason. Do not leak secrets or unnecessary personal data in the response.
- Never allow an AI model, campaign configuration, import file, or UI action to override this service.
- For race conditions, minimize the gap between the final eligibility check and the provider call. Use appropriate database locking/transaction design around internal state, but do not hold long-running database transactions open during network calls.
- If an opt-out arrives after a provider request has already been accepted, stop future sends immediately and accurately reflect that the in-flight message may not be retractable.

---

## 10. WhatsApp Provider Adapter and Webhook Architecture

### 10.1 Provider-neutral interface

Define a Python interface/abstract base with operations conceptually similar to:

```python
class MessagingProvider:
    def capabilities(self) -> "ProviderCapabilities": ...
    def validate_template(self, template_ref: str) -> "TemplateValidationResult": ...
    def send_text(self, request: "OutboundMessageRequest") -> "ProviderResult": ...
    def send_media(self, request: "OutboundMessageRequest") -> "ProviderResult": ...
    def send_template(self, request: "OutboundMessageRequest") -> "ProviderResult": ...
    def verify_webhook(self, headers, raw_body: bytes) -> bool: ...
    def parse_webhook(self, headers, raw_body: bytes) -> list["NormalizedProviderEvent"]: ...
    def lookup_message(self, provider_message_id: str) -> "ProviderMessageStatus | None": ...
    def health_check(self) -> "ProviderHealth": ...
```

This is an interface sketch, not a drop-in implementation. Adapt method signatures to the chosen provider SDK and documented API.

The adapter must normalize provider-specific status/events into internal types and record provider capabilities. Do not assume all integrations support media, delivery/read receipts, inbound webhooks, template status, message lookup, or idempotency.

### 10.2 Outbound path

1. Policy gate authorizes the send.
2. Messaging service persists message and attempt records.
3. Provider adapter converts the normalized request to provider-specific format.
4. The request includes supported correlation/idempotency metadata.
5. The response is normalized and persisted.
6. The message timeline is updated from future verified status events when available.

A provider API acknowledgment means only what the provider documents it means. Do not translate `accepted` into `delivered`.

### 10.3 Webhook ingress

Webhook handling should be quick and robust:

1. Read the raw request body.
2. Verify the signature/authenticity using the provider’s current procedure.
3. Validate the request shape and provider/account identity.
4. Compute or extract a stable event ID/idempotency key.
5. Insert the event into a durable inbound-events table with a unique constraint.
6. Return the provider-required success response after durable acceptance.
7. Process the event asynchronously.

If verification or durable storage fails, return the appropriate non-success result specified by the provider and emit an operational alert. Never acknowledge a webhook as safely accepted before it is durably stored unless the provider integration’s contract explicitly makes another design necessary.

### 10.4 Inbound event processor

- Deduplicate by provider event ID when available, supplemented by provider message ID/event type and payload hash as appropriate.
- Handle status events and inbound messages idempotently.
- Tolerate out-of-order status events; use allowed transitions and retain raw/normalized event history safely.
- Resolve sender to a doctor only when the match is sufficiently confident.
- Persist inbound message before triggering classification, AI, or follow-up logic.
- Run opt-out detection early and cancel/suppress pending jobs as soon as possible.
- If sender identity is ambiguous, create a review case and block promotional automation for the unresolved sender.

### 10.5 Provider migration

The domain model must not rely on a provider’s message ID as the primary application message ID. Store both the internal UUID and provider-specific identifiers. Provider changes require an adapter and configuration migration, not a redesign of campaign/pipeline logic.

---

## 11. Specialty Pipelines, Templates, and Content

### 11.1 Specialty routing

`Specialty` maps to a default `Pipeline` or approved pipeline selection rule. When a doctor is created/imported:

1. Normalize their specialty to a canonical specialty record.
2. Look up the currently active default pipeline mapping.
3. Propose the mapped pipeline to the operator.
4. Require a valid approved version and a separate enrollment/launch action before any message is scheduled.

If no mapping exists, leave the doctor unassigned and show a configuration warning. Never fall back to a random pipeline.

### 11.2 Pipeline step definition

A versioned step configuration should include, as relevant:

- Stable step ID and order.
- Type: `send_template`, `send_media`, `wait`, `create_task`, `condition`, or another explicitly supported type.
- Template version reference and/or media references.
- Relative delay or absolute schedule configuration.
- Business-hour/time-zone policy.
- Required approvals and provider template constraints.
- Conditions: reply received, interest status changed, demo scheduled, opt-out, or another whitelisted condition.
- Stop/skip behavior.
- Retry classification and maximum attempts.
- Safe operator-facing description.

Do not permit arbitrary Python, SQL, shell code, or unrestricted expressions inside step configuration. Conditions should be declarative and evaluated by tested application code.

### 11.3 Template rendering

- Allow only registered placeholders such as `doctor_name`, `specialty`, approved product/offer fields, and approved links.
- Define a schema and fallback behavior for each placeholder.
- Escape or safely encode data appropriate to the output context.
- Block messages with required missing placeholders; do not send unresolved literal tokens.
- Validate message length, attachment compatibility, language, and provider template rules.
- Preview rendered content before launch and store the immutable rendering snapshot per send.

### 11.4 Offers and product facts

Prices, offer validity, approved feature wording, availability claims, and discount permissions must be stored as structured, versioned facts or approved template content. A step must check whether an offer is still valid when it becomes due. AI retrieval may help find facts but must not replace structured source-of-truth checks.

### 11.5 Media lifecycle

An asset referenced by an approved pipeline version must not disappear silently when archived. Block archive/delete if a live version depends on it, or preserve a durable immutable asset reference until the dependent work is complete. Show the operator what active versions and campaigns use the asset.

---

## 12. Import and Audience Selection Architecture

### 12.1 Import flow

```mermaid
flowchart LR
    Upload[Upload CSV/XLSX] --> Parse[Parse + validate file]
    Parse --> Normalize[Normalize phone/specialty/fields]
    Normalize --> Match[Match duplicates and conflicts]
    Match --> Preview[Import preview with row-level errors]
    Preview --> Decision{Operator confirms?}
    Decision -- No --> Discard[Discard import draft]
    Decision -- Yes --> Commit[Transactional import commit]
    Commit --> Report[Persist import report]
```

### 12.2 Import rules

- Enforce size and row limits, supported encodings/formats, and spreadsheet formula safety on exports.
- Normalize phone numbers consistently before duplicate matching.
- Do not automatically enroll imported doctors in a campaign.
- Do not let a CSV/XLSX row overwrite the authoritative consent or opt-out state without a separate verified consent workflow.
- Preserve row-level outcomes: imported, updated, duplicate skipped, rejected, conflict requiring review.
- Store import batch metadata and provide a downloadable result report.

### 12.3 Audience snapshots

A campaign must record the selection criteria and the resolved audience snapshot used at approval/launch. The exact implementation may use campaign membership rows plus a compact filter definition. The snapshot supports auditing what was intended; execution-time consent and eligibility are still rechecked for every send.

---

## 13. Inbound Conversations and Follow-Ups

### 13.1 Conversation timeline

Use a normalized `Conversation` and `Message` model, not a single mutable text field. Each timeline entry must identify direction, channel, timestamps, provider identifiers where relevant, internal state, and source of status. Keep attachments and delivery events linked to their message.

### 13.2 Reply behavior

Inbound reply policies are configurable per pipeline, for example:

- Pause future promotional steps whenever a reply arrives.
- Pause only when the reply indicates interest, a question, opt-out, or need for human handling.
- Continue only after the operator resolves the conversation, where policy allows.

Default conservatively: pause the individual’s promotional sequence when a reply indicates engagement or requires a response. Do not continue a generic sales sequence while the conversation is waiting for human attention.

### 13.3 Follow-up task model

Follow-ups are durable records with due time, owner, reason, source conversation/enrollment, status, and cancellation reason. A task being due does not imply authorization to send. If it will send a message, it must use the normal campaign execution and eligibility path.

### 13.4 Opt-out processor

Implement clear opt-out patterns deterministically for supported languages and keep a conservative fallback for ambiguous phrases. When opt-out is clear:

1. Persist the raw/normalized inbound event and consent event.
2. Update the authoritative current eligibility projection in a transaction.
3. Suppress all eligible-to-send queries immediately.
4. Cancel or block future scheduled steps and follow-ups.
5. Ensure queued jobs fail their last-moment send gate.
6. Write an audit event and operational metric.
7. Display the result to the operator.

AI must not be required to honor an unambiguous opt-out. For uncertain sender matching, block outbound promotional actions until reviewed.

---

## 14. AI Assistance Architecture (P1, Disabled by Default)

AI is optional and must not be on the critical path for basic CRM, template editing, pipeline configuration, or campaign operation.

### 14.1 AI service boundary

Expose bounded operations such as:

- `classify_inbound_message`
- `summarize_conversation`
- `retrieve_approved_facts`
- `draft_reply`
- `propose_structured_action`

Use typed inputs and outputs, timeouts, output schema validation, redaction, trace/correlation IDs, and a provider health check. A model failure must not prevent manual inbox handling.

### 14.2 Safe request path

```mermaid
flowchart TD
    Inbound[Persist inbound message] --> Optout[Deterministic opt-out check]
    Optout -->|Clear opt-out| Suppress[Update consent and cancel future steps]
    Optout -->|Not an opt-out| Classify[Classify intent if AI enabled]
    Classify --> Retrieve[Retrieve approved knowledge and structured facts]
    Retrieve --> Draft[Generate draft with source references]
    Draft --> Validate[Validate schema, claims, sources and allowed intent]
    Validate --> Review[Human review queue by default]
    Review --> Approve{Authorized approval?}
    Approve -- No --> Hold[Keep draft; no outbound side effect]
    Approve -- Yes --> Gate[Shared outbound policy gate]
    Gate --> Send[Messaging service]
```

### 14.3 AI restrictions

- Draft-only by default.
- AI outputs must never execute arbitrary code, SQL, browser actions, or unrestricted workflows.
- Use a structured, enumerated action schema. The application independently authorizes each proposed action.
- AI cannot alter consent, official prices, offer expiry, approved product claims, or permissions.
- No special discount, negotiation, complaint, refund request, clinical question, unsupported claim, or uncertain case may be auto-sent in the MVP.
- Every draft should retain the approved knowledge sources used and the model/configuration version necessary for audit.
- If the approved knowledge base cannot answer, the model must say the information is unavailable or route to a human; it must not invent.
- External model use must follow the configured data-handling policy. Do not send personal data to an AI provider unless the owner has authorized it and its terms meet policy.
- Per-intent auto-send may be enabled only after explicit authorization, evaluation, monitoring, and reliable rollback controls.

---

## 15. API Design

### 15.1 API conventions

- Prefix application endpoints with `/api/v1`.
- Use JSON for ordinary CRUD/commands, multipart upload for files, and provider-specific webhook formats only on webhook endpoints.
- Define Pydantic request/response schemas and stable error codes.
- Use pagination for lists, filtering through validated fields, and deterministic sorting.
- Use idempotency keys for commands that create campaigns, enrollments, messages, or other potentially repeated external actions.
- Use optimistic concurrency/version numbers for editing templates and pipelines when concurrent edits are possible.
- Keep long-running work asynchronous. Return a job/campaign ID and expose its durable status.
- Do not return secrets or provider raw payloads by default.

### 15.2 Logical endpoint groups

| Domain | Example capabilities |
|---|---|
| Auth/users | sign in/out, current user, role/user management |
| Doctors | CRUD/archive, search, import preview/commit, export, duplicate review |
| Specialties | list/manage specialties, assign default pipeline |
| Consent | current eligibility, record consent event, process/review opt-out, block reason |
| Templates/media | CRUD, upload, version/approval, provider template status, archive dependency check |
| Pipelines | create/edit, add/reorder steps, validate, render preview, version/approve/activate |
| Campaigns | create, audience preview, validation, approval, launch, pause/resume/cancel, progress/results |
| Messages/inbox | conversations, timeline, eligible send, assignment, notes, pause automation, takeover, resolve |
| Follow-ups | create/list/complete/cancel/reschedule tasks |
| Webhooks | provider verification, inbound messages, message status events |
| Audit/system | audit search, health/readiness, provider health, queue health, emergency stop |
| AI (P1) | classify, summarize, draft, review, approve/reject, per-intent settings |

This is a capability map, not a commitment to the exact route names. Route implementations should be generated from a consistent OpenAPI contract and documented as they are built.

### 15.3 Stable error structure

Return errors in a consistent structure, for example:

```json
{
  "error": {
    "code": "OUTBOUND_BLOCKED_CONSENT_UNKNOWN",
    "message": "This doctor is not eligible for outbound messaging.",
    "request_id": "correlation-id",
    "details": {
      "action_required": "Record and verify the required consent before sending."
    }
  }
}
```

Never include secrets, raw access tokens, or unnecessary doctor information in error details.

---

## 16. Security, Privacy, and Audit Architecture

### 16.1 Authorization layers

Enforce permissions in the backend service layer, not only through hidden buttons in the UI. Every route requires authentication except specifically designed health checks and provider webhook verification endpoints. Workers use a distinct service identity with least privilege.

Sensitive actions include:

- Campaign launch and approval.
- Provider credential update.
- Consent correction/renewal and any manual block resolution.
- Template approval or activation.
- Enabling any AI auto-send capability.
- Bulk export.
- Emergency stop reset.

### 16.2 Audit events

Use an append-only audit service/table for high-value changes. Each event should carry:

- Actor type and ID (user, worker, provider, system).
- Action and result.
- Object type and ID.
- Timestamp.
- Correlation/request ID.
- Reason or approval record where relevant.
- Redacted metadata describing state changes.

Record campaign launches, pauses/resumes/cancellations, template/pipeline approvals, sends/blocks/failures, opt-outs, consent updates, import commits, permission changes, provider configuration changes, AI approvals, and emergency-stop actions.

Do not log secrets, unnecessary full message bodies, or raw provider payloads indiscriminately. If raw event retention is necessary for reconciliation, store it in a restricted table with retention controls and redact credentials.

### 16.3 Personal data and files

- Restrict data by role and log sensitive exports.
- Encrypt traffic in transit and use encrypted storage/backups in production.
- Apply retention periods and a documented deletion/archive process.
- Keep an opt-out/suppression record sufficient to avoid future contact, even if a full CRM profile is archived/deleted as allowed by policy.
- Upload and download assets only through authenticated, authorized routes or short-lived signed URLs.
- Keep `.env` files out of source control; commit only `.env.example` with safe placeholders.

---

## 17. Reliability, Observability, and Recovery

### 17.1 Observability signals

Provide structured logs and metrics for:

- API latency and error rate.
- PostgreSQL connectivity, saturation, and transaction failures.
- Redis/queue availability, queue depth, oldest queued job, and worker heartbeat.
- Due job age and number of stuck `running` tasks.
- Provider health, request latency, rate-limit errors, authentication errors, and failure classes.
- Webhook verification failures, ingestion count, duplicates, processing lag, and processing errors.
- Messages by internal state and provider-reported state.
- Opt-out detection/processing latency.
- Campaign pause and emergency-stop status.
- AI health and review backlog when enabled.

Every user action and async job should propagate a correlation ID through logs and related records.

### 17.2 Health checks

Expose separate endpoints or probes for:

- **Liveness:** process is running.
- **Readiness:** dependencies necessary for serving the relevant operation are available.
- **Provider health:** credential/capability checks that do not accidentally send a real message.
- **Worker health:** worker heartbeat and queue processing age.

A provider outage should not necessarily mark the CRM API as unavailable. The UI should show that CRM works while messaging is degraded.

### 17.3 Recovery loop

On startup and periodically:

1. Find stale `running` step executions/attempts.
2. Determine if the operation was definitely not started, definitely failed, or may have been submitted.
3. Requeue only safe, known-not-submitted work.
4. Mark ambiguous requests `uncertain` and reconcile them using provider events/status lookup where possible.
5. Alert operators about unresolved cases.

Do not convert all stale records back to `pending`; that would create duplicate-send risk.

### 17.4 Backups and restore

- Automate PostgreSQL backups with defined retention.
- Back up essential media metadata and ensure object storage lifecycle/versioning or backup policy is documented.
- Define recovery point and recovery time objectives before production launch.
- Test restoring a backup in a separate environment and verify campaign/message/consent relationships.
- Document Redis rebuild behavior: queue messages may be reconstructed from authoritative database state where designed; the database cannot be reconstructed from Redis.

---

## 18. Deployment Topology

### 18.1 Local development

Use Docker Compose to provide:

- `web`
- `api`
- `worker`
- `scheduler`
- `postgres`
- `redis`
- optional local object-storage emulator
- optional Ollama service for P1 development

Use migrations to initialize the schema. Provide seed data strictly for development/testing; no real doctor data or provider secrets in a public repository.

### 18.2 Production topology

```mermaid
flowchart TD
    Browser[User Browser] --> HTTPS[HTTPS Reverse Proxy / Ingress]
    HTTPS --> Web[Web Frontend]
    HTTPS --> API[API Instances]
    HTTPS --> Hook[Webhook Ingress]
    API --> PG[(Managed PostgreSQL)]
    Hook --> PG
    API --> Redis[(Managed Redis / Queue)]
    Scheduler[Scheduler Process] --> PG
    Scheduler --> Redis
    Redis --> Workers[Worker Processes]
    Workers --> PG
    Workers --> Provider[WhatsApp Provider]
    Provider --> Hook
    API --> Objects[(Object Storage)]
    Workers --> Objects
    API -. optional .-> AI[AI Provider / Ollama]
    Workers -. optional .-> AI
```

### 18.3 Operational separation

- Deploy API, worker, and scheduler as separately restartable process types, even if they initially share the same image.
- Ensure only one effective scheduler lease/leader creates duplicate schedules; alternatively make job creation safely idempotent across multiple schedulers.
- Keep webhook ingress responsive; do not perform lengthy LLM calls inside the webhook HTTP request.
- Scale workers horizontally only with atomic job claims and idempotency safeguards.
- Keep provider request concurrency configurable and compatible with the provider’s documented limits.
- Use managed services when practical for production database, queue, and object storage, while keeping a documented self-hosted option if cost requires it.

---

## 19. Configuration and Secrets

Use typed application settings, validated at startup. Keep development and production configurations separate.

Example environment variable groups (names are illustrative; finalize with implementation):

```dotenv
APP_ENV=development
APP_BASE_URL=http://localhost:3000
API_BASE_URL=http://localhost:8000
SECRET_KEY=replace-with-local-secret
DATABASE_URL=postgresql+psycopg://app:app@postgres:5432/ayudevices
REDIS_URL=redis://redis:6379/0
MEDIA_STORAGE_BACKEND=local
MEDIA_LOCAL_PATH=/app/media
WHATSAPP_PROVIDER=cloud_api
WHATSAPP_ACCESS_TOKEN=load-from-secret-store
WHATSAPP_PHONE_NUMBER_ID=replace-me
WHATSAPP_BUSINESS_ACCOUNT_ID=replace-me
WHATSAPP_WEBHOOK_VERIFY_TOKEN=load-from-secret-store
WHATSAPP_APP_SECRET=load-from-secret-store
AI_ENABLED=false
AI_PROVIDER=none
AI_MODEL=none
OUTBOUND_EMERGENCY_STOP=false
```

This example intentionally uses placeholders. Do not commit production values. Use a secret manager in production, rotate credentials when exposed, and redact values from logs and support bundles. The exact provider fields depend on the adapter selected.

Important settings should be split between:

- Deployment secrets: credentials, signing keys, database URLs.
- Operational configuration: queue settings, retention, provider adapter and capability values.
- Business policy: allowed hours, follow-up caps, approved intent switches, emergency stop.

Changes to high-risk business policy require permission checks and audit records. Emergency stop should be stored durably and also be readable quickly by workers; cache it only with a design that prevents stale cache from allowing sends after stop activation.

---

## 20. Testing Architecture

Testing must prove boundaries and failure behavior, not only happy-path screens.

### 20.1 Unit tests

- Phone normalization and duplicate matching.
- Consent eligibility and suppression decisions.
- Pipeline validation, version pinning, and placeholder rendering.
- State-transition validation.
- Follow-up due-time and business-hours calculations.
- Provider status normalization.
- Webhook signature parser, event deduplication, and ordering rules.
- AI output schema validation and claim-source requirements (P1).

### 20.2 Integration tests

- API + PostgreSQL migrations and constraints.
- Campaign launch creating persistent audience memberships/enrollments.
- Concurrent workers cannot claim the same step in a way that causes a duplicate send.
- Consent changing after enqueue but before execution blocks the send.
- Opt-out cancels/suppresses pending steps and follow-ups.
- Provider success/failure/timeout mapping.
- Duplicate inbound webhook does not duplicate a message or task.
- Worker crash and recovery preserve uncertainty correctly.
- Provider adapter mocks simulate missing delivery receipts, expired credentials, rate limits, and unsupported media.

### 20.3 End-to-end tests

- Add eligible doctor → map specialty → preview pipeline → launch campaign → simulate provider status → verify timeline/progress.
- Import file → preview duplicates/invalid rows → commit → verify that no campaign sends automatically.
- Receive opt-out while a later step is queued → verify that no subsequent send occurs.
- Receive an engaged reply → verify pipeline pause and inbox assignment policy.
- Pause/cancel campaign during execution and verify safe state handling.
- Enable emergency stop while jobs are queued and verify all new sends are blocked.
- Verify the UI accurately shows provider-submitted vs delivered vs unknown states.

Use mocked/sandbox provider accounts for automated tests. Do not send test messages to real doctors as part of CI.

---

## 21. Implementation Sequence and Dependency Graph

Build in dependency order rather than enabling all features simultaneously.

### Phase 1 — Application foundation

- Repository setup, environments, auth/RBAC, PostgreSQL migrations, structured logging, frontend shell, health checks.
- Doctor and specialty modules.
- Audit event service and import-preview foundation.

**Exit gate:** users and doctor records persist, permission tests pass, migrations/backups are documented.

### Phase 2 — Approved content and pipeline configuration

- Media storage adapter, templates/template versions, specialty mapping, pipeline builder, validation, preview, approval, and immutable version snapshots.

**Exit gate:** a pipeline cannot be activated with missing/unapproved content, invalid placeholders, or unsupported provider capabilities.

### Phase 3 — Messaging adapter and policy gate

- Select provider, implement sandbox adapter, centralized eligibility service, consent events, template validation, webhook signature verification, normalized provider outcomes.

**Exit gate:** every outbound entry point goes through the shared gate; all negative consent tests block sends.

### Phase 4 — Durable campaign execution

- Campaign and audience snapshots, enrollments, per-step execution, scheduler, worker queue, idempotency, retry classification, uncertain-state recovery, pause/resume/cancel.

**Exit gate:** concurrent-worker, crash-recovery, provider-timeout, and duplicate-event tests pass.

### Phase 5 — Inbox, opt-outs, and follow-ups

- Persist inbound messages, inbox UI, sender matching, deterministic opt-out handler, task management, conversation pause/human takeover.

**Exit gate:** clear opt-outs suppress all future covered outbound paths; replies affect configured pipeline behavior.

### Phase 6 — Analytics and production hardening

- Campaign reports, health panels, alerts, backup/restore rehearsal, data retention, provider runbooks, security review, load and end-to-end tests.

**Exit gate:** operational issues are visible; restore and emergency-stop behaviors are verified.

### Phase 7 — AI assistance (optional P1)

- Approved knowledge items, retrieval, intent classification, summaries, draft-only reply generation, review queue, quality evaluation.

**Exit gate:** AI failure leaves core workflow usable; unsupported facts are escalated; no AI send bypass exists.

### Phase 8 — Optional integrations (P2)

- External CRM synchronization, calendar, spreadsheets, order/inventory systems, extra languages, workflow integration.

**Exit gate:** each integration has explicit ownership of fields, authorization, retries, conflict handling, and audit coverage.

---

## 22. Architecture Decisions to Record (ADRs)

Create short records in `docs/ADRs/` for decisions that materially affect implementation. At minimum, decide and document:

1. **Frontend framework:** Vite or Next.js; choose one.
2. **WhatsApp provider:** official Cloud API or vetted provider, capabilities, template workflow, status events, costs, and webhook contract.
3. **Queue/scheduler:** Celery + Redis or equivalent; persistence, retry semantics, and scheduler leader/duplicate strategy.
4. **Authentication:** server session vs JWT, cookie/CSRF strategy, and role model.
5. **Media storage:** local development and production object-storage choice, upload scanning, and signed URLs.
6. **Consent evidence model:** how consent is collected, represented, reviewed, and retained for each channel/purpose.
7. **AI deployment and data policy:** local vs hosted model, permitted data, model evaluation, and approved intents.
8. **Retention and deletion:** message/media/log retention, backup lifecycle, and suppression record treatment.

Do not silently change an ADR or the PRD’s security/consent invariants. Update the decision record with the reason and implications.

---

## 23. Definition of Architectural Done

The architecture is implemented sufficiently for an initial production pilot when:

- Web UI calls the backend API; provider secrets never reach the browser.
- Core business logic is in testable services rather than UI components or route handlers.
- PostgreSQL stores authoritative doctor, consent, pipeline-version, campaign, execution, message, and audit state.
- Pipeline and template versions are immutable once approved and referenced by live campaigns.
- Every send path uses the same last-moment eligibility/policy gate.
- Workers and webhooks tolerate duplicate delivery without repeating application side effects.
- The system handles provider timeouts as uncertain when the outcome cannot be established.
- Opt-outs suppress pending future outbound work and survive restarts.
- Campaign pause/cancel and global emergency stop have defined, tested semantics.
- Provider capability differences are explicit and shown accurately in the UI.
- Observability, health checks, error alerts, backups, and restore procedures exist.
- Automated tests cover the major state transitions, negative consent paths, webhook deduplication, and crash recovery.
- Optional AI can be disabled without disabling the basic CRM and campaign workflow.

The governing principle is **reliable, policy-controlled execution with traceable outcomes**, not unrestricted automation.
