# IMPLEMENTATION_PLAN.md — AyuDevices Doctor Outreach & Relationship Automation Platform

**Document version:** 1.0  
**Status:** Build plan for a new system  
**Prepared:** 26 October 2026  
**Related specifications:** `PRD.md`, `ARCHITECTURE.md`, `DATABASE.md`  
**Purpose:** Define what to implement, in what order, the prerequisites for each phase, and how to prove a phase is complete.

---

## 1. How to Use This Plan

This document is the implementation sequence for the product described in `PRD.md`. Use `ARCHITECTURE.md` for component boundaries and `DATABASE.md` for persistence, constraints, transactions, and state definitions.

Implement one phase at a time. Each phase must produce working software, automated tests, updated documentation, and a demonstrable acceptance result before work proceeds to the next dependent phase.

Do not attempt to generate the entire application in one large AI-IDE request. Large one-shot implementations make it difficult to verify migrations, authorization, sending safeguards, and failure recovery.

### 1.1 Source-of-truth order

When making implementation decisions:

1. `PRD.md` defines required product behavior and release scope.
2. `ARCHITECTURE.md` defines service boundaries and runtime design.
3. `DATABASE.md` defines data integrity and persistence rules.
4. This plan defines execution order and phase exit gates.
5. If the documents conflict or leave an important behavior ambiguous, stop that implementation slice, record the conflict, and propose a documented decision. Do not silently weaken a safeguard.

### 1.2 Non-negotiable rules in every phase

- Consent, recipient eligibility, provider policy, and current automation controls are enforced by backend services—not just the UI.
- All outbound message paths use one shared messaging/eligibility boundary.
- Unknown, unavailable, or inconsistent permission data means **do not send**.
- Imported contacts are not automatically enrolled in a campaign.
- Doctor specialty selects a communication pipeline; it does not support clinical inference or diagnosis.
- Message content, pricing, offers, discounts, and product claims come from operator-approved sources.
- A queue job or HTTP request being accepted is not proof of message delivery.
- Persist state before and after external side effects and handle uncertain outcomes explicitly.
- No stealth automation, phone-number rotation, CAPTCHA bypass, ban-evasion logic, or rate-limit workarounds.
- AI is optional and disabled by default until the deterministic foundation is tested.
- Never commit API keys, provider credentials, session cookies, database passwords, or production personal data.

---

## 2. Delivery Strategy

Build the system in this dependency order:

1. Project foundation and development workflow.
2. Authentication, authorization, and database migrations.
3. Doctor CRM and safe data import.
4. Consent and communication eligibility.
5. Template and media library.
6. Versioned specialty pipeline builder.
7. Provider integration in sandbox/test mode.
8. Shared outbound messaging and send gate.
9. Durable campaign execution and scheduler.
10. Inbox, inbound events, opt-outs, and human takeover.
11. Follow-up tasks and notifications.
12. Analytics, operations, security hardening, and pilot release.
13. Optional AI assistance.
14. Optional external integrations and optimization.

The first usable release should reliably run approved, configured pipelines with human oversight. AI replies, advanced integrations, and speculative automation must not delay or bypass this foundation.

---

## 3. Scope Boundaries

### 3.1 MVP scope (must be completed before a live pilot)

- User authentication and role-based authorization.
- Doctor records, specialty assignment, search, tags, and import preview.
- Consent evidence, opt-out state, audit history, and a centralized eligibility check.
- Approved templates, media metadata, and safe asset storage.
- Versioned pipeline definitions and specialty-to-pipeline mapping.
- Campaign creation, audience preview, approval, scheduling, pause/resume/cancel, and per-doctor execution tracking.
- A supported WhatsApp integration behind a provider-neutral adapter.
- Inbound event handling when supported by the chosen provider.
- Persistent inbox and conversation history.
- Follow-up scheduling with caps and cancellation rules.
- Human takeover, emergency stop, error review, audit trail, monitoring, backups, and recovery tests.
- Automated tests and documented deployment/runbooks.

### 3.2 Later scope (not required for the first live pilot)

- AI-generated drafts and automatic replies.
- Retrieval-augmented answers from approved product knowledge.
- Additional languages beyond the initial configured templates.
- Zoho/other CRM synchronization, calendar, email, n8n, inventory, quotations, invoices, or order integrations.
- Advanced lead scoring and optimization recommendations.
- Multi-tenant SaaS features unless explicitly approved as a product requirement.

### 3.3 Explicit non-goals

- Automatically negotiating or approving discounts.
- Generating unsupported product or clinical claims.
- Diagnosing patients or providing clinical decision-making.
- Sending unlimited follow-ups to silent recipients.
- Claiming to guarantee delivery or prevent account restriction.
- Bypassing WhatsApp restrictions or permission mechanisms.

---

## 4. Phase 0 — Product Decisions and Project Setup

### Objective

Create a reproducible software project with clear configuration, quality checks, and documented decisions before implementing business behavior.

### Tasks

- [x] Create the repository and initial branch strategy.
- [x] Add `PRD.md`, `ARCHITECTURE.md`, `DATABASE.md`, and this plan at the project root.
- [x] Create a short `README.md` covering prerequisites, startup, tests, migrations, and environment configuration.
- [x] Add `.gitignore` for Python virtual environments, frontend dependencies/build output, local databases, media, logs, caches, and `.env` files.
- [x] Add a safe `.env.example` listing variable names and non-secret example values only.
- [x] Choose supported Python and Node versions and pin compatible dependency ranges/lockfiles.
- [x] Create backend and frontend skeletons with health/status endpoints and a minimal page.
- [x] Add formatting, linting, type-checking, and test commands.
- [x] Add a development Docker Compose configuration for PostgreSQL and Redis.
- [x] Add structured logging with request/correlation IDs and redaction rules.
- [x] Record architecture decisions for authentication, queue technology, object storage, WhatsApp provider, and deployment target.
- [x] Create CI to install dependencies, run linters/type checks/tests, and validate migrations on a disposable database.

### Deliverables

- Reproducible local setup.
- Minimal frontend and FastAPI application that start successfully.
- CI pipeline with baseline checks.
- Written decisions and environment setup guide.

### Exit gate

- [x] A new developer can follow the README to start the application locally.
- [x] A clean checkout passes the documented quality checks.
- [x] No secret or real doctor data is tracked in version control.
- [x] CI runs on a clean environment and fails when a deliberately failing test is introduced.

---

## 5. Phase 1 — Database Foundation, Authentication, and Authorization

### Objective

Establish secure identities and the schema/migration workflow before adding business data.

### Backend tasks

- [ ] Configure SQLAlchemy models and database sessions according to `DATABASE.md`.
- [ ] Configure Alembic migrations and migration review conventions.
- [ ] Implement the initial schema for users, roles/permissions (or the approved RBAC representation), audit events, and global automation controls.
- [ ] Implement user sign-in/session handling using a proven authentication approach; do not create custom password cryptography.
- [ ] Hash passwords using a maintained password-hashing implementation if local passwords are used.
- [ ] Implement authorization dependencies for routes and service-layer actions.
- [ ] Add secure account bootstrap/invitation or administrator creation flow.
- [ ] Configure secure cookie/session behavior or bearer-token handling appropriate to the selected frontend setup.
- [ ] Add login/logout and failed-login audit events without logging credentials.
- [ ] Add health endpoints that distinguish application liveness from database/queue readiness.
- [ ] Add database connection and transaction tests.

### Frontend tasks

- [ ] Build the application shell and navigation structure.
- [ ] Add sign-in and sign-out states.
- [ ] Add unauthorized, forbidden, session-expired, loading, and error states.
- [ ] Hide inaccessible controls in the UI while relying on backend authorization as the enforcement layer.

### Exit gate

- [ ] Unauthenticated requests cannot read or mutate protected data.
- [ ] Users with insufficient permissions are rejected at the backend.
- [ ] Administrative actions produce audit events.
- [ ] Migrations create the schema from an empty database and can be tested in CI.
- [ ] Secrets are obtained from environment/secret configuration and never embedded in source.

---

## 6. Phase 2 — Doctor CRM and Specialty Reference Data

### Objective

Create the authoritative doctor directory on which consent, pipeline selection, and campaigns depend.

### Backend tasks

- [ ] Implement specialty reference data and administrator management or controlled seed data.
- [ ] Implement doctor create/read/update/archive operations.
- [ ] Normalize and validate phone numbers into E.164 form before saving.
- [ ] Enforce the approved uniqueness rule for normalized phone numbers.
- [ ] Implement doctor search, pagination, filtering, sorting, and detail retrieval.
- [ ] Implement tags/segments if required by the initial PRD scope.
- [ ] Add a doctor interaction timeline contract, even if early entries contain only manual/audit activity.
- [ ] Ensure archived/deleted records do not inadvertently make an opted-out number eligible again.
- [ ] Add safe audit events for create/update/archive and sensitive field changes.

### Frontend tasks

- [ ] Build the Doctors list with search, filters, pagination, and clear empty/error states.
- [ ] Build Add/Edit Doctor forms.
- [ ] Show normalized phone number, specialty, consent/eligibility summary, and next action when available.
- [ ] Show validation feedback for malformed numbers and duplicates.
- [ ] Add Doctor Detail view with profile and interaction timeline.

### Acceptance criteria

- [ ] A user can create, update, search, view, and archive a doctor.
- [ ] Duplicate normalized phone numbers are rejected or explicitly resolved according to the documented policy.
- [ ] Invalid phone numbers cannot be enrolled or sent to the provider.
- [ ] Specialty is stored as a validated reference, not arbitrary uncontrolled text where a reference is required.
- [ ] CRM APIs and services have unit and integration tests.

---

## 7. Phase 3 — CSV/XLSX Import and Safe Segmentation

### Objective

Allow users to add many doctors without corrupting existing records or implicitly starting outreach.

### Backend tasks

- [ ] Implement import batch and row persistence.
- [ ] Parse the supported input formats with file size, row count, and content-type limits.
- [ ] Implement column mapping and row-level validation.
- [ ] Normalize phone numbers and validate specialties/tags.
- [ ] Detect duplicates within the file and against existing records.
- [ ] Flag missing/unknown consent information for review rather than inferring permission.
- [ ] Define explicit duplicate strategies: skip, update only approved non-consent fields, or require manual resolution.
- [ ] Never let spreadsheet input silently overwrite consent or opt-out fields.
- [ ] Commit import rows in bounded transactions and report partial failures clearly.
- [ ] Produce an import report with counts and reason codes.

### Frontend tasks

- [ ] Build upload and column-mapping interface.
- [ ] Build preview screen showing valid, invalid, duplicate, conflicting, and missing-consent rows.
- [ ] Require explicit confirmation before commit.
- [ ] Show import progress and downloadable results.
- [ ] Allow row-level conflict review.

### Acceptance criteria

- [ ] Uploading/importing contacts never starts a campaign automatically.
- [ ] Duplicate and malformed records are visible before commit.
- [ ] The user can download a report explaining rejected/skipped rows.
- [ ] Consent and opt-out state cannot be overwritten through a generic import.
- [ ] Large files are processed in bounded, observable jobs rather than blocking an API request indefinitely.

---

## 8. Phase 4 — Consent, Eligibility, and Global Controls

### Objective

Implement the policy enforcement layer before enabling any outbound messaging.

### Backend tasks

- [ ] Implement current consent state scoped by channel and purpose, as specified in `DATABASE.md`.
- [ ] Implement append-only consent event history with actor, timestamp, source, scope, and evidence reference where available.
- [ ] Implement explicit transitions for opt-in, opt-out, unknown, and pending-review states.
- [ ] Implement an eligibility service that returns both decision and machine-readable reason codes.
- [ ] Check latest consent, purpose, recipient validity, global stop state, conversation/campaign state, scheduled time, and provider restrictions as relevant.
- [ ] Make the policy service fail closed when required data is missing or cannot be read.
- [ ] Add a global emergency stop and pause controls with audited changes.
- [ ] Add per-campaign and per-doctor suppression rules.
- [ ] Ensure all planned sending code will call this service rather than implementing local checks.
- [ ] Add race-condition tests for opt-out versus scheduled-send execution.
- [ ] Add a route/service that previews eligibility without sending.

### Frontend tasks

- [ ] Build the Consent Centre.
- [ ] Display consent state, scope, source, recorded time, and eligibility reason.
- [ ] Require authorized evidence/confirmation for changes to consent.
- [ ] Display global automation state prominently.
- [ ] Provide a clear emergency stop and a deliberate confirmation for resuming outbound work.

### Acceptance criteria

- [ ] Unknown or unavailable consent blocks sending.
- [ ] Every decision provides an explainable reason code.
- [ ] Changes to consent create historical events and audit entries.
- [ ] Opt-out state cannot be removed by ordinary doctor edits or import operations.
- [ ] Tests demonstrate that a concurrent opt-out prevents a not-yet-authorized message from proceeding.

**No live outbound sending may be enabled before this phase passes its exit gate.**

---

## 9. Phase 5 — Message Templates, Approved Facts, and Media Library

### Objective

Allow the owner to control and approve message content and assets before those items can be used by pipelines.

### Backend tasks

- [ ] Implement template records and immutable template versions.
- [ ] Support approved placeholders such as `{doctor_name}` and `{specialty}` using a defined allowlist.
- [ ] Validate missing/unknown placeholders before approval or campaign launch.
- [ ] Implement draft, review, approved, archived, and superseded content states as defined in the product model.
- [ ] Store media metadata in PostgreSQL and file bytes in private object storage or an approved local storage adapter for development.
- [ ] Validate file extension, MIME type, size, storage path, and safe download permissions.
- [ ] Use random/object IDs for storage keys; do not expose private filesystem paths.
- [ ] Implement attachment linking to template versions and preserve media references in sent-message snapshots.
- [ ] Provide a preview/render service that does not send messages.
- [ ] Add approval/audit history for content and pricing/offer facts.
- [ ] Keep authoritative price/offer facts structured and versioned where required; never let AI create or change them.

### Frontend tasks

- [ ] Build template list, editor, preview, version history, and approval actions.
- [ ] Build media upload/list/detail and usage references.
- [ ] Show file validation and upload status.
- [ ] Preview rendered placeholders against sample doctor data.
- [ ] Make it clear which assets and templates are approved for production.

### Acceptance criteria

- [ ] A draft or rejected template cannot be used for a live campaign.
- [ ] The final rendered content and referenced attachment versions can be traced.
- [ ] Unknown placeholders block approval or launch with a useful error.
- [ ] Private media cannot be fetched by an unauthorized user.
- [ ] No product feature, price, discount, clinical claim, or testimonial is generated from model assumptions.

---

## 10. Phase 6 — Specialty Pipeline Builder and Versioning

### Objective

Let operators define a sequential communication workflow per specialty without hard-coding each specialty into application code.

### Backend tasks

- [ ] Implement pipeline and immutable pipeline-version records.
- [ ] Implement ordered pipeline steps referencing approved templates/media and configurable wait conditions.
- [ ] Implement specialty-to-default-pipeline mapping with optional override rules.
- [ ] Validate pipeline steps, references, timing, supported content types, maximum follow-ups, business hours, and stop conditions.
- [ ] Implement draft/approved/published/archived pipeline lifecycle according to the agreed schema.
- [ ] Publish a new version rather than mutating a version already referenced by campaigns.
- [ ] Store step behavior such as pause on reply, stop on opt-out, stop on human takeover, or continue only under an explicitly configured condition.
- [ ] Build a rendered schedule preview for an example doctor.
- [ ] Seed illustrative specialty/pipeline records only as editable examples; no unapproved copy or claims should be sent by default.

### Frontend tasks

- [ ] Build pipeline list and specialty mapping page.
- [ ] Build step editor with drag/reorder or explicit ordering controls.
- [ ] Allow template and media selection.
- [ ] Configure delay, business-hour window, stop conditions, and follow-up cap.
- [ ] Show a full timeline preview before publishing.
- [ ] Show version history and which campaigns use each version.

### Acceptance criteria

- [ ] A specialty can be mapped to an approved pipeline without a code change.
- [ ] An operator can preview a pipeline in order before publishing.
- [ ] Publishing changes a version ID and does not rewrite in-flight campaign behavior.
- [ ] A pipeline cannot be published with missing templates, invalid timing, unapproved content, or unsupported conditions.
- [ ] Tests cover default selection, override, version pinning, and invalid pipeline configuration.

---

## 11. Phase 7 — WhatsApp Provider Adapter (Sandbox/Test Mode First)

### Objective

Integrate a supported messaging provider behind a stable internal contract before campaign scheduling is enabled.

### Provider selection tasks

- [ ] Choose a supported provider/official WhatsApp Business Platform path based on current business needs, account eligibility, inbound webhook support, template requirements, status events, cost, and operational support.
- [ ] Document provider capabilities and limits; do not assume unsupported features.
- [ ] Create a provider adapter contract for send text/template/media, verify configuration/health, process provider message IDs, and map statuses into internal states.
- [ ] Implement provider sandbox/test-double support so all core flows can be tested without contacting real doctors.
- [ ] Store credentials using environment variables or a secret manager and use provider-configuration records only for non-secret metadata or secret references.
- [ ] Implement signature verification for inbound webhooks where supported by the provider.
- [ ] Validate payload size, event type, provider account, timestamp, and idempotency key.
- [ ] Persist inbound events before asynchronous processing.
- [ ] Map provider statuses to the internal model without claiming that a status means more than the provider confirms.
- [ ] Record provider request IDs and safe error details while redacting secrets and unnecessary personal data.
- [ ] Add provider timeout, retryability, rate-limit response, auth failure, invalid template, and malformed payload tests.

### Frontend tasks

- [ ] Build provider configuration status view without exposing secrets.
- [ ] Add a test-connection or sandbox test action where provider capabilities allow it.
- [ ] Surface provider health and actionable errors.

### Acceptance criteria

- [ ] Provider transport logic is isolated from CRM and campaign code.
- [ ] Core application tests can run with a fake provider.
- [ ] Webhooks are authenticated/validated where supported and are deduplicated.
- [ ] Provider IDs and internal IDs remain distinct.
- [ ] No provider credentials appear in logs, API responses, or frontend bundles.

---

## 12. Phase 8 — Centralized Outbound Messaging and Safe Send Gate

### Objective

Build one tested entry point through which every outbound message must pass.

### Backend tasks

- [ ] Implement a messaging application service used by manual sends, campaign steps, follow-ups, and future AI-assisted replies.
- [ ] Require an explicit recipient, conversation/purpose, approved content snapshot, source operation, and idempotency key.
- [ ] Re-check permission immediately before calling the provider, not only when a campaign is created.
- [ ] Re-check global stop state, current consent, purpose, recipient validity, conversation and campaign state, send-window/template rules, and any required human approval.
- [ ] Revalidate the rendered content and attachment references.
- [ ] Persist the message record and an attempt record before/around the provider call using the transaction strategy specified in `DATABASE.md`.
- [ ] Distinguish internal job status, provider submission state, and provider delivery/read status.
- [ ] Handle timeouts and ambiguous responses as `uncertain` or equivalent; never blindly retry an uncertain send.
- [ ] Add an operator reconciliation path for uncertain outcomes using provider status lookup/webhook evidence or manual review.
- [ ] Implement a durable outbox or equivalent mechanism where the database is the source of truth for queued work.
- [ ] Record audit details including who initiated the send, recipient, template/version, purpose, policy result, provider reference, and outcome.

### Acceptance criteria

- [ ] No direct API route or worker calls the provider while bypassing the shared message service.
- [ ] A blocked eligibility decision produces no provider call.
- [ ] Retries do not create a second send for the same idempotency key when safe deduplication is available.
- [ ] Uncertain sends are visible and require reconciliation rather than blind retry.
- [ ] UI shows accurate state labels and does not convert `submitted` to `delivered` without provider evidence.

---

## 13. Phase 9 — Campaign Creation and Audience Snapshot

### Objective

Let users authorize a specific audience and specific pipeline version before executing messages.

### Backend tasks

- [ ] Implement campaign lifecycle and approval state.
- [ ] Implement recipient selection by explicit doctors, specialty, tag, or approved saved segment.
- [ ] Snapshot selected recipients and key configuration at campaign authorization time.
- [ ] Exclude or visibly flag doctors whose consent/eligibility is not confirmed; never silently make them eligible.
- [ ] Resolve the approved pipeline version and persist the version ID used by the campaign.
- [ ] Validate required content versions, provider configuration, business-hour policy, maximum follow-ups, and campaign limits.
- [ ] Provide an eligibility preview/count before launch and show exclusions by reason.
- [ ] Require explicit campaign authorization after reviewing recipients, messages, timing, and policy warnings.
- [ ] Ensure a campaign snapshot does not freeze permission: current eligibility must be checked again at send time.

### Frontend tasks

- [ ] Build campaign creation wizard.
- [ ] Add audience filters and recipient preview.
- [ ] Show eligible, blocked, duplicate, and unknown-consent counts.
- [ ] Show full per-recipient step sequence and schedule.
- [ ] Add save draft, request approval, authorize, schedule, and cancel actions based on user permissions.

### Acceptance criteria

- [ ] The user can inspect the precise recipient set and content schedule before launch.
- [ ] Importing doctors does not start campaigns.
- [ ] Campaigns pin approved pipeline/template versions.
- [ ] Current consent is rechecked on execution even when the audience snapshot showed eligibility.

---

## 14. Phase 10 — Durable Scheduler and Campaign Execution Engine

### Objective

Execute published pipelines predictably and recover safely across restarts, network failures, and repeated worker delivery.

### Backend tasks

- [ ] Configure the selected task queue and scheduler (as specified in `ARCHITECTURE.md`).
- [ ] Implement durable enrollment and step-execution records.
- [ ] Claim due jobs with database-safe locking/leases and concurrency limits.
- [ ] Persist state transitions and attempt outcomes.
- [ ] Use idempotency keys and uniqueness constraints to prevent duplicate work.
- [ ] Respect business hours, configured delays, provider constraints, daily/follow-up limits, and global/campaign/doctor pause state.
- [ ] Before each step, check whether the doctor replied, opted out, was put on hold, or had the enrollment cancelled.
- [ ] Use centralized send gate for actual sends.
- [ ] Implement retries only for known-safe retryable errors with bounded backoff and a maximum attempt count.
- [ ] Keep uncertain external outcomes separate from ordinary retryable failures.
- [ ] Implement pause/resume/cancel semantics for campaign and enrollment scopes.
- [ ] Recover due jobs after worker restart without resetting completed step state.
- [ ] Provide failed-job and uncertain-job reconciliation tooling.
- [ ] Ensure queue redelivery cannot bypass database state checks.

### Frontend tasks

- [ ] Build campaign detail and per-doctor progress view.
- [ ] Show pending, scheduled, sending, submitted, delivered/read (when supported), failed, uncertain, skipped, cancelled, and completed states accurately.
- [ ] Add pause/resume/cancel controls with confirmation and permission checks.
- [ ] Display retryability and reason codes for failures.
- [ ] Provide an operator workflow for resolving uncertain attempts.

### Acceptance criteria

- [ ] A campaign survives application and worker restarts.
- [ ] Pause prevents newly claimed steps from sending after the pause is observed.
- [ ] Cancelled/opted-out recipients do not receive subsequent campaign steps.
- [ ] Repeated scheduler execution does not duplicate completed steps.
- [ ] Failures are visible and actionable; no job remains silently stuck.
- [ ] A provider timeout does not trigger a blind duplicate send.

---

## 15. Phase 11 — Inbox, Incoming Events, and Opt-Out Processing

### Objective

Receive and persist replies, associate them with doctors/conversations, and stop or pause automated sequences when required.

### Backend tasks

- [ ] Implement conversations and normalized message records.
- [ ] Process authenticated provider webhooks or supported provider inbox events.
- [ ] Persist raw/minimized event metadata before downstream processing.
- [ ] Deduplicate inbound event IDs and provider message IDs.
- [ ] Match the sender to a doctor using normalized phone number and provider account context.
- [ ] Create a review item instead of guessing when matching is ambiguous.
- [ ] Update provider-reported message statuses asynchronously and idempotently.
- [ ] Implement deterministic detection for clear opt-out phrases in supported languages; use AI only later for ambiguous phrasing.
- [ ] For an opt-out, persist the event, update authoritative consent, cancel or suppress pending work, and block further promotional messages.
- [ ] For uncertain opt-out wording, pause promotional automation pending review.
- [ ] Implement a documented reply behavior for each pipeline: pause, continue under a defined condition, or hand over to the inbox.
- [ ] Add replay/reconciliation for inbound events after temporary failures.

### Frontend tasks

- [ ] Build inbox/conversation list with unread, assigned, paused, and needs-review filters.
- [ ] Build chronological message timeline with accurate status.
- [ ] Add assign, mark handled, pause automation, human takeover, and resume controls.
- [ ] Show why a pipeline was paused and what event triggered the change.
- [ ] Show opt-out processing outcomes and unresolved sender review items.

### Acceptance criteria

- [ ] Inbound events are durable and deduplicated.
- [ ] A clear opt-out is reflected in authoritative consent and suppresses pending messages.
- [ ] An inbound reply can pause the pipeline based on configured policy.
- [ ] An ambiguous sender or ambiguous opt-out does not trigger unsafe outbound automation.
- [ ] Human takeover blocks automated responses until explicitly resumed by an authorized user.

---

## 16. Phase 12 — Follow-Ups, Notifications, and Task Management

### Objective

Support explicit, bounded follow-up tasks that stay connected to conversation and campaign state.

### Backend tasks

- [ ] Implement follow-up task schema and lifecycle.
- [ ] Support due date/time, owner, doctor/conversation reference, reason, and source campaign/step.
- [ ] Implement create, reschedule, complete, cancel, and overdue handling.
- [ ] Enforce business hours and maximum follow-up limits.
- [ ] Cancel/suppress tasks after opt-out, completed sale/status where represented, manual stop, or relevant reply policy.
- [ ] Route any automated follow-up through the centralized send gate.
- [ ] Add in-app notifications and optional email notifications if configured.
- [ ] Avoid duplicate tasks when the same event is processed more than once.

### Frontend tasks

- [ ] Build task list/calendar-style view as appropriate to initial scope.
- [ ] Add due, overdue, completed, cancelled, and assigned filters.
- [ ] Add reschedule/complete/cancel controls.
- [ ] Show links to the originating conversation and campaign step.

### Acceptance criteria

- [ ] Follow-ups remain durable across restarts.
- [ ] A follow-up does not send after an applicable opt-out or pause.
- [ ] Replayed events do not create duplicate follow-ups.
- [ ] Follow-up volume is bounded and configurable.

---

## 17. Phase 13 — Dashboard, Analytics, Audit Search, and Operational Controls

### Objective

Give operators visibility into campaign performance, blocked actions, failures, costs where known, and system health.

### Backend tasks

- [ ] Implement campaign/step counts and per-specialty pipeline metrics.
- [ ] Implement import validation counts and consent eligibility counts.
- [ ] Implement message state counts using provider-reported statuses where available.
- [ ] Implement follow-up due/cancelled/overdue metrics.
- [ ] Implement opt-out processing latency and review-queue metrics.
- [ ] Implement audit-event search with access controls and pagination.
- [ ] Add health checks for database, task queue, scheduler heartbeat, provider connection/configuration, and storage.
- [ ] Add alerts for provider failures, rising uncertain sends, stuck jobs, webhook processing lag, expired credentials, and backup failures.
- [ ] Show messaging spend only if authoritative provider pricing/billing information is available; otherwise display unavailable.
- [ ] Avoid interpreting delivery/read events as purchase intent or sales conversions.

### Frontend tasks

- [ ] Build operational overview dashboard.
- [ ] Build audit log filters and event detail view.
- [ ] Build system health and failed/uncertain job views.
- [ ] Provide export for permitted reports while applying data minimization and permission rules.

### Acceptance criteria

- [ ] Operators can locate active campaigns, pending reviews, failed jobs, uncertain sends, due follow-ups, and provider/system health.
- [ ] Metrics reconcile with underlying records and have documented definitions.
- [ ] Logs and analytics avoid exposing secrets or unnecessary personal data.
- [ ] Every administrative or consequential action can be traced to a user or explicit system actor.

---

## 18. Phase 14 — Security, Reliability, and Production Readiness

### Objective

Prove that the MVP can be operated safely before live outreach is enabled at meaningful volume.

### Tasks

- [ ] Review authentication, authorization, and session handling.
- [ ] Review database least privilege and migration permissions.
- [ ] Verify secure media access and upload limits.
- [ ] Verify CORS, CSRF protections where applicable, rate limits on sensitive APIs, input validation, and request size limits.
- [ ] Add dependency and container vulnerability checks.
- [ ] Redact phone numbers/message content from operational logs where full content is not required.
- [ ] Test concurrent sends, consent changes, global stop, and campaign state transitions.
- [ ] Test provider downtime, webhook replay, database outage, queue outage, worker crash, storage failure, and app restart.
- [ ] Test backup restoration into a clean environment.
- [ ] Document recovery procedure for uncertain messages and provider status reconciliation.
- [ ] Add data-retention, archive, deletion, export, and access-request procedures appropriate to the operating context.
- [ ] Add alert routing and an operator runbook.
- [ ] Configure TLS, secret rotation, production database backups, and environment-specific configuration.
- [ ] Perform a limited pilot with an explicitly approved test audience and low-risk template content.
- [ ] Collect operator feedback and fix workflow blockers before wider usage.

### Mandatory live-pilot checklist

- [ ] Consent and eligibility gate tests pass.
- [ ] The official/provider-approved messaging path is configured and verified.
- [ ] The provider's applicable business and messaging policies have been reviewed by the responsible operator.
- [ ] Approved templates and assets are available in the required provider format.
- [ ] Test recipients and test data are separated from real campaigns.
- [ ] Global emergency stop has been exercised successfully.
- [ ] Pause, resume, cancel, opt-out, and human takeover are tested end to end.
- [ ] Duplicate and uncertain-send recovery has been tested.
- [ ] Backups and restoration are verified.
- [ ] An operator can identify and resolve failures from the UI/runbook.

### Exit gate

Do not begin the live pilot while any critical consent bypass, duplicate-send bug, secret exposure, unverified provider setup, or unrecoverable campaign state exists.

---

## 19. Phase 15 — Optional AI Assistance (P1, Disabled by Default)

### Prerequisites

Phases 0–14 must pass. AI must not be used as a substitute for consent, policy, recipient verification, database constraints, or deterministic workflow state.

### Backend tasks

- [ ] Implement approved product facts and document ingestion/versioning.
- [ ] Add optional vector retrieval for approved documents if semantic retrieval is needed.
- [ ] Keep authoritative prices, offers, claims, policies, and inventory facts in structured sources.
- [ ] Integrate the selected local or hosted model behind an AI service interface.
- [ ] Implement intent classification and conversation summarization with schema-validated outputs.
- [ ] Implement source-grounded reply drafts and include retrieved source references internally for review.
- [ ] Define confidence/abstention behavior and tests for missing or conflicting facts.
- [ ] Implement human-review queue with approve, edit-and-approve, reject, and escalate paths.
- [ ] Implement per-intent automation permissions; draft-only is the default.
- [ ] Allow low-risk automatic responses only after explicit administrator enablement and evaluation.
- [ ] Keep discounts, price negotiation, complaints, refunds, clinical questions, unsupported claims, and consequential actions in human review.
- [ ] Prevent model output from calling arbitrary tools, SQL, shell commands, or unrestricted browser operations.
- [ ] Record model/version, retrieved source IDs, prompt/config version, review outcome, and safe diagnostic metadata.
- [ ] Ensure AI-generated messages go through the same send gate as all other messages.

### Frontend tasks

- [ ] Build knowledge-base management and indexing status.
- [ ] Build AI draft/review queue and display evidence/source references.
- [ ] Provide global AI enable/disable, draft-only mode, per-intent switches, and confidence configuration.
- [ ] Show clear escalation reasons and permit human takeover.

### Acceptance criteria

- [ ] AI is disabled by default in production configuration.
- [ ] Unsupported or conflicting knowledge causes abstention/escalation rather than invention.
- [ ] AI cannot modify consent, authorize special discounts, change official pricing, or bypass the policy gate.
- [ ] All externally visible AI messages are logged and pass normal authorization/eligibility checks.
- [ ] AI is evaluated on a representative test set before any auto-reply intent is enabled.

---

## 20. Phase 16 — Optional Integrations and Optimization (P2)

Implement only after the MVP is stable and there is a verified business need.

Possible slices:

- [ ] External CRM synchronization.
- [ ] Google Calendar/demo scheduling.
- [ ] Gmail or other email workflows.
- [ ] Spreadsheet exports and scheduled reports.
- [ ] n8n or other approved workflow triggers.
- [ ] Order, invoice, quotation, or inventory status integration.
- [ ] Additional locales and language-specific template variants.
- [ ] Advanced analytics and operational summaries.

For each integration:

1. Define the authoritative source for each synced field.
2. Define authentication scopes and secret storage.
3. Define matching and duplicate rules.
4. Define conflict behavior and retry semantics.
5. Add audit events, observability, and integration tests.
6. Ensure external events cannot bypass consent or the centralized send gate.
7. Ship behind a feature flag until the integration is verified.

### Exit gate

The integration is independently disableable, recoverable, auditable, and tested for duplicate/replayed events. Failure of an optional integration does not corrupt the core doctor/consent/campaign database.

---

## 21. Cross-Phase Engineering Standards

Every phase must follow these practices.

### 21.1 Backend conventions

- Keep route handlers thin; put rules in testable domain services.
- Use typed request/response schemas and stable API error codes.
- Use database transactions around related state changes.
- Use repositories or a consistent data-access convention; avoid cross-module ad hoc queries.
- Validate inputs on the server even when forms validate in the browser.
- Add pagination and explicit limits to list APIs.
- Keep provider-specific payloads inside adapters and normalized internal models.
- Make background jobs idempotent because queues may deliver a task more than once.

### 21.2 Frontend conventions

- Implement loading, empty, success, validation, forbidden, and failure states.
- Do not trust hidden buttons as authorization.
- Do not expose secrets or directly send through a provider from the browser.
- Show the difference between queued, submitted, delivered, read, failed, and uncertain states.
- Use explicit confirmation for campaign launch, destructive actions, and global resume.
- Make blocked actions explainable and actionable.

### 21.3 Database and migration conventions

- Every schema change is a reviewed migration.
- Test migrations from a clean database and from the immediately previous schema.
- Apply foreign keys, unique constraints, checks, and indexes in the database where appropriate.
- Use UTC timestamps.
- Do not rewrite immutable content/pipeline versions that an existing campaign references.
- Preserve consent, opt-out, provider event, message attempt, and audit history according to retention policy.

### 21.4 Testing convention

For each feature, implement tests with the feature—not as a cleanup task at the end. A phase is not complete merely because the page renders or a happy-path manual demo works.

---

## 22. Required Test Matrix

| Area | Required scenarios | Minimum test level |
|---|---|---|
| Authentication | Unauthenticated, insufficient role, expired session, disabled user | API integration + E2E |
| Doctor CRM | Valid record, malformed phone, duplicate phone, archive, search/filter | Unit + API integration |
| Imports | Missing column, bad number, duplicate rows, conflicting record, consent columns in file, partial failure | Unit + integration |
| Consent | Unknown, opted out, purpose mismatch, missing store, concurrent opt-out | Unit + integration + concurrency |
| Templates | Draft rejected at launch, unknown placeholder, unapproved asset, version snapshot | Unit + integration |
| Pipelines | Specialty default, override, invalid step, publish new version, existing campaign remains pinned | Unit + integration |
| Provider | Auth failure, invalid template, rate limit, timeout, duplicate webhook, malformed callback | Adapter tests + integration |
| Messaging | Eligibility blocked, global stop, duplicate idempotency key, uncertain outcome, provider status reconciliation | Unit + integration |
| Campaigns | Preview, approval, pause, resume, cancel, one doctor fails, restart recovery | Integration + E2E |
| Scheduler | Due jobs, business hours, duplicate task delivery, lease expiry, worker crash | Integration + concurrency |
| Inbox | Inbound message saved, sender match ambiguity, duplicate event, human takeover | Integration + E2E |
| Opt-outs | Clear opt-out, alternate supported phrase, ambiguous phrase, pending-job cancellation, post-opt-out send blocked | Unit + integration + E2E |
| Follow-ups | Due, delayed, rescheduled, cancelled after reply/opt-out, duplicate task generation | Unit + integration |
| Media | Oversized upload, invalid MIME, unauthorized read, missing object, asset snapshot | Unit + integration |
| Analytics | Counts reconcile with source records, unavailable billing remains unavailable | Integration |
| Security | Secret redaction, access restrictions, request validation, unauthorized media, audit access | Integration + security checks |
| Recovery | Database restart, queue outage, provider outage, restore from backup | Integration + operational exercise |
| AI (P1) | Missing fact, conflicting facts, prompt injection in source docs, low confidence, restricted intent | Unit + evaluation suite |

---

## 23. Release Gates

### Gate A — Internal development demo

Allowed after Phases 0–3. Use synthetic or approved test data. No live campaigns.

### Gate B — Consent-safe test mode

Allowed after Phase 4. Verify every planned outbound route is blocked when consent is missing/unknown. Use provider test doubles/sandbox.

### Gate C — End-to-end campaign simulation

Allowed after Phases 5–10. Demonstrate pipeline preview, campaign authorization, step execution against a fake provider, pause/resume/cancel, restart recovery, and uncertain outcomes.

### Gate D — Inbound/opt-out simulation

Allowed after Phase 11. Replay inbound events and prove deduplication, sender matching, opt-out persistence, and pending-step suppression.

### Gate E — Live limited pilot

Allowed only after Phase 14's mandatory checklist passes. Use approved provider configuration, approved content, a documented and authorized audience, and conservative operation. Monitor every send and resolve all uncertain outcomes.

### Gate F — AI-assisted pilot

Allowed only after Phase 15 tests, evaluation, and explicit feature-level approvals. Draft-only should be used first. Enable automatic replies only for explicitly approved low-risk intents.

---

## 24. Definition of Done for Any Feature

A feature is considered complete only when all applicable items pass:

- [ ] Requirement and UX behavior are defined.
- [ ] API contract and authorization are implemented.
- [ ] Data schema/migration is reviewed where required.
- [ ] Server-side validation and business rules exist.
- [ ] Auditability and safe logging are included for consequential actions.
- [ ] Unit tests cover normal and edge cases.
- [ ] Integration tests cover persistence and external boundaries.
- [ ] UI handles loading, empty, validation, blocked, and failure states.
- [ ] Recovery/idempotency behavior is defined for background or external actions.
- [ ] Documentation and configuration examples are updated.
- [ ] CI passes.
- [ ] The implementation has been exercised against the acceptance criteria.
- [ ] The developer report states what was implemented, what was tested, and any remaining limitations.

Do not mark a feature complete because code was generated, files exist, or a page looks finished.

---

## 25. Working Instructions for an AI IDE

Use the following operating procedure throughout the project:

1. Read `PRD.md`, `ARCHITECTURE.md`, `DATABASE.md`, and `IMPLEMENTATION_PLAN.md` before implementing a phase.
2. Identify the current phase, its prerequisites, deliverables, and exit gate.
3. Propose a small implementation slice with the exact files/modules, migration changes, and tests needed.
4. Implement only that slice and preserve the architecture boundaries.
5. Run relevant unit, integration, lint, type, and migration tests.
6. Fix regressions before beginning another slice.
7. Update documentation and a phase checklist with verified status.
8. Do not remove tests, relax authorization, bypass consent, suppress errors, or fabricate successful provider outcomes to make the build pass.
9. Use a fake provider or sandbox until the live-pilot gate is explicitly satisfied.
10. If information is missing, do not invent business content or external credentials. Add a clear configuration field or mark the feature blocked pending owner input.
11. Keep AI capabilities behind feature flags and disabled by default.
12. At the end of each work session, report: files changed, functionality implemented, commands/tests executed, results, unresolved issues, and the next smallest step.

### Recommended first AI-IDE task

> Read all four specification files. Implement **Phase 0 only**. Create the project skeleton, local development setup, configuration example, code-quality commands, CI checks, basic FastAPI health endpoints, and minimal frontend shell. Do not implement doctor messaging, campaign sending, AI replies, or live provider integration yet. Run the setup and tests, then report verified results and any blockers.

After Phase 0 passes, ask the IDE to implement Phase 1. Continue one phase at a time.

---

## 26. Final Delivery Checklist

Before considering the MVP ready for normal operation:

- [ ] Core user journeys in `PRD.md` work end to end.
- [ ] Doctor imports are explicit and never trigger outbound messages on their own.
- [ ] Specialty-based pipelines are configured by data, versioned, previewable, and auditable.
- [ ] Approved templates and attachments are used consistently.
- [ ] Every outbound path passes the centralized eligibility/send gate.
- [ ] Consent, opt-outs, pauses, human takeover, and the global stop are enforced across queued and scheduled work.
- [ ] Campaigns persist and recover safely after restarts.
- [ ] Duplicate events and duplicate job delivery are handled idempotently.
- [ ] Provider-reported message states are displayed accurately.
- [ ] Uncertain external sends can be reconciled without blindly duplicating messages.
- [ ] Inbox, follow-ups, reviews, and audit history work.
- [ ] Metrics and health signals are available to operators.
- [ ] Security review, backup restore, and failure-recovery exercises are complete.
- [ ] Deployment, configuration, troubleshooting, and incident runbooks exist.
- [ ] Critical and high-severity defects have been resolved or explicitly accepted with documented mitigation.
- [ ] The responsible owner has approved the live pilot and its scope.

**The implementation priority is correctness and recoverability first, useful automation second, and autonomous AI last.**
