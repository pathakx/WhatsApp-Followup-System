# PRD.md — AyuDevices Doctor Outreach & Relationship Automation Platform

**Document version:** 1.0  
**Status:** Final product requirements specification for a new build  
**Prepared:** 26 October 2026  
**Product owner:** AyuDevices  
**Primary use case:** Specialty-based doctor outreach, product education, follow-up, and conversation management for AyuSynk 2+  
**Document purpose:** Defines what to build, how it should behave, what is in scope, and how completion will be verified. It is not a record of an existing implementation.

---

## 1. Product Overview

### 1.1 Vision

Build a web application that helps AyuDevices manage doctor relationships and run configurable, specialty-based communication pipelines. An operator should be able to add a doctor using their name, mobile number, and specialty; select or automatically assign the right pipeline; configure approved messages and media; preview the whole sequence; and run it with controlled scheduling and persistent execution state.

The platform should manage the full communication lifecycle: doctor records, consent evidence, specialty segmentation, reusable templates, media assets, campaign execution, message and delivery states, incoming replies, opt-outs, follow-up tasks, human review, analytics, and audit history.

### 1.2 Primary outcome

Reduce manual work in sending appropriate product information and follow-ups to doctors while keeping the operator in control of message content, timing, business claims, and exceptional situations.

### 1.3 Product rules

1. **The owner controls the content.** The system must not invent product claims, prices, offers, discounts, testimonials, clinical claims, or promises. Operators supply and approve message templates, assets, and offers.
2. **Specialty chooses a pipeline, not a medical conclusion.** Specialty-based personalization is for communication organization and content selection only. It must not infer patient diagnoses or provide clinical advice.
3. **Consent is a hard gate.** No outbound communication may be sent when eligibility or the required permission cannot be verified.
4. **One shared send gate.** Campaigns, direct sends, follow-ups, and AI-assisted replies must pass through the same authorization, recipient, and consent checks.
5. **Persist before and after actions.** Campaign steps, message attempts, incoming messages, opt-outs, and approvals must survive application restarts.
6. **Never guess delivery.** Distinguish queued, submitted, sent, delivered, read, failed, and unknown states according to what the configured provider actually reports.
7. **Human control is always available.** The operator can pause a pipeline, pause a campaign, stop all outbound automation, or take over a conversation.
8. **Compliance is not a rate-limit workaround.** Do not build number rotation, stealth automation, CAPTCHA bypass, or other mechanisms intended to evade WhatsApp controls.
9. **Fail closed.** If provider status, consent status, recipient identity, or required campaign configuration is uncertain, block the action and show a useful reason.
10. **Build an MVP first.** Reliable configured pipelines and message tracking come before autonomous AI replies and optional integrations.

---

## 2. Goals and Non-Goals

### 2.1 Goals

- Create and maintain doctor records with name, normalized phone number, specialty, tags, consent records, and communication history.
- Import doctor lists from CSV/XLSX with validation, duplicate detection, and a preview before saving.
- Configure one or more reusable communication pipelines for each specialty or audience segment.
- Let the owner write and approve all message copy and attach images, brochures, PDFs, videos, links, and other supported media.
- Assign a pipeline automatically from the doctor’s specialty, with a clear override and preview.
- Schedule ordered message steps with configurable timing, business hours, stop conditions, and a maximum number of follow-ups.
- Execute messages through a supported WhatsApp integration with visible provider status and cost-related metadata where available.
- Receive and persist inbound messages when the selected provider supports webhooks or inbox access.
- Pause or cancel pending outbound steps when the doctor replies, opts out, or requires human attention, according to pipeline configuration.
- Track campaign and per-doctor progress and recover safely after a restart.
- Offer AI assistance for categorization, summaries, and draft replies only after the core deterministic workflow is reliable.
- Provide dashboard, inbox, pipeline editor, campaign tracking, consent management, audit logs, and system health controls.

### 2.2 Non-goals for the first release

- Building a complete ERP, inventory, delivery, billing, or order-management system.
- Automatically negotiating prices, approving discounts, or making binding commercial commitments.
- Giving clinical recommendations, interpreting clinical audio, diagnosing patients, or replacing a healthcare professional’s judgement.
- Sending unlimited promotional follow-ups to non-responders.
- Circumventing WhatsApp product limits or enforcement.
- Guaranteeing that WhatsApp will deliver every message or that an account will never be restricted.
- Training a custom foundation model.
- Building integrations with every CRM, email, calendar, and telephony system in the MVP.

### 2.3 Product success metrics

The dashboard must be capable of reporting at least:

- Doctors imported, valid, duplicate, and rejected.
- Doctors eligible and ineligible to message, with reason categories.
- Active, paused, completed, cancelled, and failed campaigns.
- Pipeline step completion by specialty and campaign.
- Messages queued, submitted, delivered, read, failed, and in unknown state, where the provider exposes those events.
- Incoming conversations and replies.
- Follow-ups due, cancelled, and overdue.
- Opt-outs and opt-out processing latency.
- Human-review queue size and age.
- AI-assisted drafts approved, edited, rejected, and escalated, if AI is enabled.
- Estimated messaging spend when provider pricing data is available; otherwise show provider-reported billing data as unavailable, not estimated fact.

The product must not claim that a delivered or read message means the recipient is interested or that a sale occurred.

---

## 3. Users and Permissions

### 3.1 Owner / Administrator

Can manage users, doctors, consent evidence, products, pricing facts, templates, assets, specialty pipelines, campaigns, AI settings, provider integrations, audit logs, exports, and global stop controls.

### 3.2 Sales operator

Can view assigned doctors and conversations, update permitted CRM fields, create and preview campaigns when authorized, send eligible approved templates, process replies, schedule follow-ups, and request human review. They cannot override consent, approve their own restricted actions if an approval policy forbids it, or change provider credentials.

### 3.3 Reviewer / Sales manager

Can review message templates and drafts, approve or reject campaign launches and sensitive replies, assign conversations, review opt-out events, and view performance reports. Permissions must be configurable rather than hard-coded to one person.

### 3.4 Background workers

Can run already-authorized scheduled jobs, synchronize provider statuses, process webhooks, and update campaign state. Workers must use the same application-level policy services as user-initiated actions and must not bypass access controls or sending checks.

### 3.5 Permission rules

- Apply role-based access control on both the UI and backend.
- Protect administrative and provider configuration actions.
- Record actor identity for important changes and sends.
- Use least-privilege access for workers and integrations.
- Keep secrets out of source code, logs, and exported data.

---

## 4. Primary User Journeys

### 4.1 Add one doctor and assign a specialty pipeline

1. The operator selects **Add Doctor**.
2. The operator enters name, mobile number including country code, and specialty; optional fields can include city, product interest, notes, tags, and lead source.
3. The application normalizes and validates the phone number and checks for a duplicate.
4. The operator records the source and evidence of permission to receive the relevant communication. If permission is unknown or insufficient, the doctor is saved but marked ineligible to message.
5. The application proposes the default pipeline mapped to the selected specialty.
6. The operator can preview the pipeline, content, media, expected schedule, and any provider-template requirements.
7. The operator authorizes enrollment. The application rechecks eligibility before creating a runnable enrollment.
8. The scheduler runs steps when due, subject to current consent, provider rules, campaign state, business hours, and stop conditions.
9. Every step records its state and result. An inbound reply may pause the pipeline or move the conversation to the inbox depending on configured rules.

### 4.2 Import many doctors

1. Upload CSV or XLSX.
2. Map source columns to supported fields.
3. Preview normalized records, duplicates, invalid numbers, missing specialties, missing consent evidence, and conflicting values.
4. Choose a safe import policy: skip duplicate, update approved non-consent fields, or send conflicts for review. Imports must not overwrite consent or opt-out state silently.
5. Save valid records and produce a downloadable import report.
6. Importing a contact list must not automatically start outbound messages. A separate campaign or explicit enrollment action is required.

### 4.3 Configure a specialty pipeline

1. Select **Pipelines** and create or edit a pipeline.
2. Choose a specialty or segment, pipeline name, objective, and activation status.
3. Add ordered steps. Each step uses an existing approved template or a draft that must be approved before production use.
4. Select supported attachments or media assets; configure timing and stop conditions.
5. Define whether an inbound reply pauses the sequence, whether follow-ups stop when a doctor engages, and which intents require a human.
6. Validate all missing assets, placeholders, templates, approvals, and timing rules.
7. Preview the sequence using sample doctor data and approve a new version.
8. Changes create a new version. Enrolled doctors remain on their pinned version unless a manager explicitly migrates them.

### 4.4 Run and monitor a campaign

1. Create a campaign from an approved pipeline version.
2. Select doctors using filters or an imported cohort.
3. Display eligible, ineligible, duplicate, opted-out, and unknown-consent counts before launch.
4. Show a content and schedule preview plus any official provider template requirements and expected provider costs if known.
5. Require an explicit launch confirmation from an authorized user.
6. Create one enrollment per eligible doctor and a persisted step execution record for each upcoming action.
7. Permit pause, resume, cancel, and per-doctor exclusion.
8. Recheck eligibility immediately before each outbound attempt.
9. Display progress, errors, uncertain sends, inbound replies, and opt-outs.

### 4.5 Handle an inbound reply

1. Receive the inbound event from the provider, or create a manual inbox entry if the configured channel does not support automatic inbound events.
2. Deduplicate the event and associate it with a doctor using the provider sender identity and normalized number.
3. Persist the incoming message before processing it.
4. Check deterministic opt-out patterns first.
5. If the message is an opt-out, persist it, block future covered sends, cancel pending jobs, and surface the event.
6. Otherwise classify the reply as a simple product question, request for brochure/demo, pricing or discount inquiry, objection, positive interest, request to call, complaint, clinical question, or unknown.
7. In MVP, route the reply to the inbox and allow the operator to use approved quick replies. AI classification/drafting is an optional later capability.
8. Never send an AI-generated answer when the system lacks approved facts or the message requires human judgment.

---

## 5. Scope and Delivery Priorities

### P0 — MVP foundation (must ship first)

- Authentication and role-based permissions.
- Doctor CRM and consent-event records.
- CSV/XLSX import preview and validation.
- Specialty and segment management.
- Template and media library.
- Pipeline builder with ordered steps and versioning.
- Campaign creation, approval, launch, pause, resume, and cancellation.
- Persistent scheduler and per-doctor pipeline execution.
- A supported WhatsApp sending adapter. Production unattended sends must use the official WhatsApp Business Platform/Cloud API or an explicitly approved provider integration.
- Centralized send-eligibility and message-template validation.
- Provider status webhook handling where supported.
- Inbound inbox when supported by the provider.
- Opt-out detection and immediate suppression of pending follow-ups.
- Audit history, error reporting, application health, backup, and recovery.

### P1 — Sales operations and assisted intelligence

- Conversation inbox with assignment and notes.
- Configurable follow-up policies and task reminders.
- Human-review queue.
- Knowledge base of approved product facts, offers, FAQs, and documents.
- AI intent classification, conversation summaries, and draft replies, initially draft-only.
- Low-risk auto-replies behind per-intent switches, quality evaluation, and explicit owner authorization.
- Analytics and scheduled daily summaries.

### P2 — Integrations and optimization

- External CRM synchronization, including Zoho if selected.
- Google Sheets/Excel synchronization and exports.
- Calendar-based demo scheduling.
- Order, inventory, quotation, and invoice lookups where authoritative systems exist.
- Additional language support.
- Advanced campaign cohort analysis and experiments.
- Workflow integration through a controlled internal engine or n8n.

Do not start P1/P2 work at the expense of P0 correctness. All P1/P2 features must use the same policy enforcement, audit, and persistence layers.

---

## 6. Functional Requirements

### 6.1 Doctor CRM

Each doctor record must contain:

- Unique internal identifier.
- Full name.
- Normalized phone number in international format.
- Specialty ID and display name.
- City/region, if provided.
- Lead source and acquisition date, if known.
- Tags, product interest, and non-sensitive relationship notes.
- Assigned owner/operator.
- Current relationship stage (for example: new, contacted, engaged, demo requested, follow-up, converted, not interested, archived).
- Current consent eligibility and a link to its event history.
- Last inbound and outbound interaction timestamps.
- Next scheduled action.
- Created/updated timestamps and actor references.

Requirements:

- Add, view, edit, archive, restore where permitted, search, filter, sort, and export doctor records.
- Search by name and normalized phone number.
- Detect duplicate phone numbers before insert.
- Use a database uniqueness rule for normalized phone number where appropriate; support an explicitly reviewed merge flow rather than creating silent duplicates.
- Treat consent and opt-out events as separate auditable records, not fields that can be casually overwritten by imports.
- Allow specialty values to be managed by administrators and preserve an `Other / Unclassified` option.
- Do not collect patient records or patient health data as part of doctor relationship management.

### 6.2 Specialty and segmentation

Provide configurable segments for:

- Specialty.
- Region/city.
- Product interest.
- Relationship stage.
- Campaign enrollment and outcome.
- Assigned operator.
- Last interaction and next task.
- Consent eligibility.

Initial specialty choices should include:

- General Physician.
- Pediatrician.
- Cardiologist.
- Pulmonologist.
- Surgeon.
- Anesthesiologist.
- Orthopedic.
- Emergency Medicine.
- Gynecologist.
- Medical Student.
- Professor / Faculty.
- Other / Unclassified.

Administrators must be able to add, rename, deactivate, and map specialties to default pipelines without code changes. Deactivating a specialty must not delete existing records or historical campaign data.

### 6.3 Consent and communication eligibility

This is a system-wide P0 capability.

Maintain a single authoritative communication-eligibility record supported by an append-only consent-event history. Suggested states:

- `opted_in` — valid permission for the applicable channel and purpose is documented.
- `opted_out` — a request to stop some or all covered communication has been recorded.
- `pending_review` — evidence or the scope of permission needs review.
- `unknown` — no sufficient evidence is stored.

Store, where applicable:

- Channel and communication purpose/category.
- Consent status.
- Timestamp, source, method, and evidence reference.
- Notice/version shown to the recipient, if applicable.
- Actor or process that recorded the event.
- Opt-out scope, timestamp, source, original message, and processing result.
- Renewed consent evidence, if the person later re-consents.

Rules:

1. `unknown` and `pending_review` are ineligible for promotional outbound sending.
2. All outbound send paths must check the latest status immediately before sending.
3. If consent storage or policy evaluation is unavailable, fail closed.
4. An import cannot mark a contact `opted_in` without configured evidence and an authorized operation.
5. An opt-out received in or outside WhatsApp must be recorded according to the supported workflow and suppress covered future messages.
6. When an opt-out is detected, cancel or suppress queued campaign steps and follow-up tasks as soon as possible, and recheck all remaining jobs at send time.
7. Ambiguous opt-out language pauses promotional automation and enters review; do not send another sales pitch while it is unresolved.
8. Do not automatically re-enable an opted-out doctor. Require documented renewed permission and an auditable authorized action.
9. Keep minimum necessary suppression data so an opted-out contact is not accidentally reimported and messaged again.
10. The consent interface must show why a contact is eligible or blocked.

The application must support review of applicable local law and platform policy with the business owner. This PRD is not legal advice.

### 6.4 Template and content library

Operators must be able to create reusable templates and content assets.

Template fields:

- Name and internal ID.
- Channel/provider.
- Communication category/purpose.
- Language.
- Message body or media-template configuration.
- Approved placeholders.
- Attachments/media references.
- Review status: draft, pending approval, approved, rejected, archived.
- Provider template name/ID and provider approval status where applicable.
- Version, owner, reviewer, and timestamps.
- Optional specialty/product tags.

Supported placeholder examples:

- `{doctor_name}`
- `{specialty}`
- `{company_name}`
- `{product_name}`
- `{approved_offer}`
- `{booking_link}`
- `{operator_name}`

Rules:

- Render a preview using sample data before approval.
- Block activation if a required placeholder is missing or unresolved.
- Escape or validate placeholder values so imported data cannot inject template markup or controls.
- Show whether a template is application-approved and, separately, whether the provider has approved it.
- Do not treat approval in this application as provider approval.
- Do not silently modify live approved template content. Create a new version and require the appropriate review/approval.
- Preserve the exact approved message and the version used in the outbound audit record.
- The owner supplies final copy, media, claims, prices, offers, and validity dates. The system may suggest drafts only if enabled, but all claims must be grounded in approved facts and reviewed before activation.

### 6.5 Media library

Support provider-compatible images, videos, PDFs/documents, and links, subject to provider size and format limits.

Required capabilities:

- Upload, view, rename, tag, version, and archive assets.
- Store file type, size, checksum, upload time, and owner.
- Validate media against provider constraints before a campaign is launched.
- Show a preview or a safe file description.
- Block missing or invalid assets from active pipeline steps.
- Avoid exposing public file URLs unless intended and access-controlled.
- Keep credentials, patient data, and unrelated personal data out of media assets.
- Track asset references so an asset used by a live or historical campaign is not physically removed without a controlled archive/retention operation.

### 6.6 Specialty pipeline builder

A pipeline is a versioned, ordered sequence of communication steps that can be assigned to a doctor or group.

Pipeline fields:

- Name, description, purpose, specialty/segment mapping, and owner.
- Draft/approved/active/archived state.
- Version number and change history.
- Default business-hour policy and timezone.
- Default stop conditions and maximum duration.
- Approval information.

Supported step types for the MVP:

- Send approved text or provider-approved template.
- Send supported media/document template.
- Wait for a configured duration or until an allowed time window.
- Create a human task/reminder.
- Pause and await a reply or operator decision.
- End successfully.

Each step contains:

- Step order and internal identifier.
- Step name and purpose.
- Approved template/content reference and pinned version.
- Attachment reference(s), if supported.
- Delay from prior step or scheduling rule.
- Applicable business hours and timezone.
- Required consent purpose/category.
- Conditions to continue, skip, pause, or terminate.
- Retry policy and handling for uncertain outcomes.
- Human approval requirement.
- Optional branch conditions based on deterministic status or reviewed reply classification.

Rules:

- Pipelines are configured in the UI; new specialties or sequences must not require code changes.
- A doctor has at most one active enrollment per pipeline version and campaign unless an authorized duplicate enrollment is explicitly allowed.
- Preview all steps in order and calculate a human-readable schedule before launch.
- Validate every step before activation.
- An edited pipeline creates a new version. Each enrollment pins a version so it cannot change unexpectedly mid-sequence.
- A manager may explicitly migrate an enrollment to a new pipeline version, with an audit event and a preview of changed steps.
- Do not send a step when any template, consent, schedule, provider, or asset condition fails.

### 6.7 Starter pipeline blueprint

The application should ship with **disabled/draft starter blueprints**, not pre-approved marketing content. The owner must add and approve actual copy/assets and obtain any needed provider template approvals.

Suggested sequence stages, configurable per specialty:

1. **Introduction / greeting** — approved first-contact template where legally and platform-permitted.
2. **Product overview** — brief approved introduction to AyuSynk 2+.
3. **Specialty-relevant use cases** — only content supplied and approved by the owner for that specialty.
4. **Media / demonstration** — approved product image, brochure, PDF, or video.
5. **Product details** — approved feature facts and limitations.
6. **Price or offer** — only when the current price/offer, dates, terms, and availability are confirmed in the product/offer records.
7. **Interest check / call-to-action** — an approved request for the doctor to reply, request a demo, or ask for details.
8. **Follow-up 1** — only if permitted by consent, channel policy, configured frequency, and contact status.
9. **Follow-up 2** — optional, limited, and disabled by default until the owner approves the policy.
10. **Stop / close** — complete the enrollment, preserve history, and create a human task if needed.

This is a workflow blueprint, not a requirement to send every stage. Each pipeline can use fewer or different steps. Configure stop rules so a reply, opt-out, complaint, demo request, conversion, manual pause, or maximum follow-up count can prevent irrelevant later messages.

### 6.8 Campaign management

Campaigns define the audience and runtime policy for enrolling doctors in an approved pipeline version.

Required features:

- Create, edit draft, duplicate, preview, validate, approve, schedule, launch, pause, resume, cancel, and archive.
- Choose a pinned pipeline version and audience filters.
- Import recipients through the validated import workflow.
- Preview total audience and counts by consent state, specialty, missing fields, duplicate enrollment, and provider eligibility.
- Require authorized launch confirmation.
- Show a per-doctor progress table with current step, next step, next due time, last message state, last reply, and block/failure reason.
- Pause or cancel the entire campaign, selected doctors, or one enrollment.
- Export campaign results with safe field selection.
- Ensure a campaign resume does not repeat completed steps.
- Record who authorized a campaign and which template/pipeline versions were in use.

Campaign lifecycle states:

- `draft`
- `pending_approval`
- `scheduled`
- `running`
- `paused`
- `completed`
- `cancelled`
- `failed` (campaign-level failure that requires intervention)

Per-doctor enrollment states:

- `pending`
- `active`
- `waiting`
- `paused`
- `needs_review`
- `blocked_consent`
- `blocked_configuration`
- `completed`
- `cancelled`
- `failed`

### 6.9 Scheduler and execution engine

Use a persistent job queue and database-backed state. Do not rely on in-memory timers as the only source of truth.

For every due step:

1. Lock or claim the job safely so concurrent workers cannot execute it twice.
2. Confirm campaign and enrollment are still active.
3. Check current opt-out and consent state.
4. Check destination phone/recipient identity and required provider mapping.
5. Check business hours, current schedule, rate/policy caps, and provider window/template constraints.
6. Validate template and asset versions and required fields.
7. Verify required approvals.
8. Create a durable attempt record before calling the provider.
9. Submit through the messaging adapter using an idempotency key if supported.
10. Persist the provider result and timestamps.
11. Update the pipeline only after a known result or explicit uncertain-state reconciliation.
12. Schedule the next step only when the configured transition conditions are satisfied.

Reliability requirements:

- Handle crashes and application restarts.
- Deduplicate webhook events and jobs.
- Use database transactions, unique constraints, and locks where required.
- Use bounded retries with backoff for retryable transient errors.
- Never automatically retry a potentially sent message when the provider outcome is uncertain unless the adapter can safely reconcile it. Mark it `uncertain` and request provider status or human review.
- Separate provider errors from application errors.
- Avoid parallel user-interface messaging sessions if the chosen provider requires serialized access.
- Provide dead-letter or failed-job review for exhausted retries.

### 6.10 Messaging adapter and WhatsApp integration

Implement a provider abstraction so business workflows are independent from the transport. The production implementation should target the official WhatsApp Business Platform / Cloud API or a vetted provider that exposes the needed supported capabilities.

The adapter should define capabilities such as:

- Send text/template/media messages where permitted.
- Validate provider template status and required variables.
- Receive inbound messages and status webhooks.
- Verify webhook authenticity.
- Query available message status when the provider supports it.
- Report provider errors in a structured format.
- Health-check credentials and configuration without exposing secrets.
- Normalize provider message IDs and statuses into internal records.

Important platform constraints must be modeled explicitly:

- Business-initiated messaging on the WhatsApp Business Platform must use an approved message template when required by the current policy; the application must not assume it can send arbitrary free-form messages at any time.
- Free-form replies are subject to the provider/platform's active customer-service window and current rules.
- Provider templates can be rejected, paused, or have their status changed. Block sends that require a template which is not currently usable.
- Consent/permission must be documented and opt-out requests respected.
- The system must expose delivery status only to the extent the selected provider returns trustworthy status events.
- Provider limits, eligible message types, media constraints, and pricing must be fetched/configured from the provider's current documentation rather than hard-coded permanently.

Do not implement stealth browser automation, session-cookie scraping, CAPTCHA bypass, OTP automation, number rotation, or attempts to evade detection or platform limits. Browser automation against the WhatsApp Business App/Web is not a dependable supported production API and must not be treated as ban-proof. If a future manual-assist mode is built, it must clearly identify the operator-controlled action and must not claim verified sending/delivery unless that is confirmed by a supported source.

The adapter interface should permit a future provider change without rewriting pipeline and consent logic.

### 6.11 Inbox and conversation management

P0 should support inbound messages when available through the configured provider. If inbound webhooks are unavailable, the UI must show that automatic inbox monitoring is unavailable instead of silently implying that replies are being monitored.

Inbox requirements:

- Conversation list with doctor, last message, timestamp, unread count, owner, and status.
- Conversation timeline showing inbound and outbound messages, attachments, send status, AI drafts if enabled, operator notes, and linked campaign steps.
- Assign/unassign an operator.
- Mark read, mark resolved, reopen, add internal notes, and create follow-up tasks.
- Pause/resume automated outreach for one conversation.
- Human-takeover mode that prevents AI auto-replies until explicitly re-enabled.
- Display recent campaign context and next scheduled step.
- Search/filter by specialty, owner, unread state, intent, opt-out, campaign, and review status.
- Persist event order and deduplicate inbound events.
- Avoid displaying AI reasoning as fact; show a short explanation and source references when AI is used.

### 6.12 Incoming replies and opt-out processing

Process clear opt-outs before sales intent handling. Support clear forms such as “STOP,” “unsubscribe,” “don't contact me,” “remove my number,” and equivalent configured language patterns. AI classification can assist with broader phrasing but cannot delay a clear deterministic opt-out.

On clear opt-out:

1. Persist the original inbound event and an opt-out event.
2. Update the authoritative eligibility state transactionally.
3. Cancel or suppress pending promotional campaign steps and follow-ups.
4. Pause active promotional automation for the conversation.
5. Add the contact to suppression filters used by every sending path.
6. Display the result in the consent centre and audit log.
7. Optionally send a concise confirmation only where permitted and configured.

When sender matching is ambiguous, block further promotional messages from that sender while resolving identity. If an inbound message indicates a complaint, legal threat, sensitive issue, uncertain meaning, discount negotiation, or clinical question, route it for human review.

### 6.13 Follow-up task management

- Create manual and automated follow-up tasks.
- Support due date/time, assigned operator, reason, linked doctor/conversation/campaign, status, and notes.
- Allow rescheduling, completion, cancellation, and reassignment.
- Notify operators of upcoming and overdue tasks in the application.
- Enforce configurable business hours, maximum follow-up count, minimum spacing, and a maximum campaign duration.
- Cancel/suppress message-based follow-ups after opt-out, conversion if configured, explicit disinterest, manual stop, or a blocking reply.
- Do not interpret silence as permission for unlimited follow-ups.
- Recheck eligibility and pipeline state immediately before any follow-up message is sent.

### 6.14 Approved product facts and knowledge base (P1)

Maintain a controlled source of truth for product information used in templates and AI drafts.

Support structured facts such as product name, approved features, supported use cases, current approved price, offer start/end dates, eligibility terms, booking terms, shipping terms, product documents, and version timestamps.

Requirements:

- Facts have a source, owner, effective date, review date, and approval state.
- Outdated offers cannot be used in new drafts or campaigns.
- Conflicting or missing facts trigger a review instead of an invented answer.
- Vector retrieval may help find documents, but structured product facts remain authoritative for prices, dates, and terms.
- AI output must cite the internal source records used for product-specific factual claims.

### 6.15 AI assistance (P1, disabled by default)

AI is optional. The deterministic CRM, pipeline engine, template sending, consent enforcement, and campaign tracking must function when AI is disabled or unavailable.

Initial AI use cases:

- Intent classification.
- Conversation summary.
- Suggested reply from approved facts and documents.
- Suggested next task or pipeline transition.
- Language detection and translation assistance.

Rules:

- Begin in draft-only mode.
- Do not automatically send AI-generated answers by default.
- Auto-reply permissions must be separately configurable by intent and require explicit owner activation after testing.
- Enforce a confidence threshold and structured output schema.
- Ground product claims in approved knowledge.
- Never invent prices, discounts, availability, certifications, clinical benefits, study results, delivery dates, or commitments.
- Price negotiation, special discount, complaint, refund, clinical question, legal issue, ambiguous intent, low confidence, and unsupported questions require a human.
- AI must not access arbitrary shell, SQL, filesystem, or browser actions. It may propose only allowlisted structured actions.
- The execution layer independently validates every proposed action.
- Include global AI disable, draft-only mode, per-intent controls, per-conversation pause, human takeover, and an emergency stop.
- If the model fails or returns invalid output, keep the message in review and do not send.

Ollama may be supported as a local model adapter to reduce recurring inference cost, but model selection must be configurable. The product must not require local LLM inference for the core MVP.

### 6.16 Dashboard and pages

The first release should include these pages:

1. **Overview:** system status, active campaigns, messages by state, pending reviews, upcoming follow-ups, opt-outs, and recent errors.
2. **Doctors:** CRM table, search/filters, add/edit flow, consent reason, specialty, history, and import/export.
3. **Inbox:** inbound/outbound timeline, assignments, notes, stop/pause controls, and human takeover.
4. **Pipelines:** specialty mapping, ordered step editor, template/media selection, branch/stop rules, preview, validation, approval, and version history.
5. **Campaigns:** create, audience selection, consent eligibility preview, schedule, launch confirmation, progress, failures, pause/resume/cancel, and export.
6. **Templates:** message drafts, approved application content, provider template mapping/status, placeholders, version history, and reviewer approvals.
7. **Media:** uploads, tags, format/size validation, preview, archive, and usage references.
8. **Consent Centre:** eligible/ineligible counts, consent records, evidence, opt-out events, suppression, and re-consent audit trail.
9. **Tasks / Scheduler:** due jobs, follow-ups, failed jobs, uncertain attempts, retry/reconcile actions, and operator tasks.
10. **Knowledge Base (P1):** approved product facts and source documents.
11. **AI Review (P1):** incoming cases, classification, sources, draft, proposed action, approval/edit/reject, and review history.
12. **Analytics:** campaign and pipeline performance, message states, opt-outs, reply rate, task completion, and provider cost data where available.
13. **Settings:** users, roles, specialties, business hours, messaging policy, provider configuration, AI controls, retention, notifications, and emergency stop.
14. **Audit Log:** searchable chronological record of key actions, configuration changes, approvals, sends, blocks, webhooks, and administrative events.

### 6.17 Notifications and alerts

- Show in-app alerts for provider disconnection, webhook failures, stuck jobs, recurring send failures, pending approvals, uncertain send outcomes, and opt-out processing errors.
- Allow configuration for operational email/notification integrations in a later phase.
- Alerts must include the affected job/campaign and a practical next action.
- Do not put access tokens, full message bodies, or unnecessary personal data into notifications.

### 6.18 Reporting and analytics

- Campaign report with audience size, eligible/ineligible counts, completion, per-step outcomes, replies, opt-outs, failures, and dates.
- Pipeline report by specialty and pipeline version.
- Contact report with relationship stage and last/next activity.
- Operational report with queue depth, job failures, webhook processing lag, provider health, and uncertain outcomes.
- AI report (P1) with intent distribution, draft approval/edit/rejection, escalations, and evaluated quality.
- Export must honor role permissions and omit data not required for the selected report.
- Clearly distinguish provider-confirmed events from application estimates and internal workflow states.

---

## 7. Data Model Requirements

Use a relational database such as PostgreSQL as the authoritative store. The following are logical entities; implementation names may vary, but the relationships and invariants must be preserved.

### 7.1 Core entities

**User**
- id, name, email/login, role, status, created_at, updated_at.

**Doctor**
- id, name, normalized_phone, specialty_id, city/region, lead_source, assigned_user_id, relationship_stage, tags, notes, last_inbound_at, last_outbound_at, next_action_at, archived_at, timestamps.

**Specialty**
- id, display_name, active, default_pipeline_id, timestamps.

**ConsentEvent**
- id, doctor_id or unresolved_sender_key, channel, purpose, event_type, status_after_event, source, evidence_ref, original_message_ref, actor_id, occurred_at, recorded_at, metadata.

**MessageTemplate**
- id, name, channel, purpose/category, language, body/configuration, approval_state, provider_template_id, provider_status, owner_id, reviewer_id, current_version_id, timestamps.

**TemplateVersion**
- id, template_id, version, content_snapshot, placeholder_schema, attachment_refs, approved_by, approved_at, created_at.

**MediaAsset**
- id, file_name, mime_type, size_bytes, checksum, storage_ref, status, owner_id, metadata, created_at, archived_at.

**Pipeline**
- id, name, purpose, specialty/segment mapping, status, current_version_id, owner_id, created_at, updated_at.

**PipelineVersion**
- id, pipeline_id, version, configuration_snapshot, validation_result, approved_by, approved_at, created_at.

**PipelineStep** (or versioned step configuration)
- id, pipeline_version_id, step_order, step_type, template_version_id, media_refs, delay/configuration, conditions, retry_policy, approval_requirement.

**Campaign**
- id, name, pipeline_version_id, audience_definition, status, timezone, scheduled_at, launched_by, approved_by, started_at, ended_at, timestamps.

**Enrollment**
- id, campaign_id, doctor_id, pipeline_version_id, status, current_step_id/order, next_due_at, paused_reason, started_at, completed_at, timestamps.

**Message**
- id, doctor_id, conversation_id, direction, channel, provider_message_id, template_version_id, body_snapshot, attachment_refs, state, error_code, error_detail_safe, created_at, submitted_at, delivered_at, read_at, failed_at, status_source.

**MessageAttempt**
- id, message_id, idempotency_key, attempt_number, request_metadata_redacted, result_metadata_redacted, state, started_at, completed_at.

**Conversation**
- id, doctor_id, provider_conversation_ref, assigned_user_id, status, automation_paused, human_takeover, last_message_at, created_at, updated_at.

**InboundEvent**
- id, provider_event_id, provider_message_id, sender_key, payload_hash, received_at, processed_at, process_state, safe_error.

**FollowUpTask**
- id, doctor_id, conversation_id, enrollment_id, owner_id, due_at, task_type, status, reason, notes, created_by, timestamps.

**ReviewItem** (P1)
- id, conversation_id, inbound_message_id, intent, confidence, reason, retrieved_sources, draft, proposed_action, status, reviewer_id, timestamps.

**KnowledgeItem** (P1)
- id, item_type, title, structured_value/document_ref, source, status, effective_from, effective_until, approved_by, reviewed_at, timestamps.

**AuditEvent**
- id, actor_type, actor_id, action, object_type, object_id, occurred_at, result, correlation_id, redacted_metadata.

**ProviderConfiguration**
- id, provider_type, display_name, secret_reference, status, capabilities, last_health_check_at, safe_diagnostics.

### 7.2 Integrity requirements

- Normalize phone numbers before matching and deduplication.
- Add database constraints and indexes for common searches and state transitions.
- Enforce one authoritative consent record/state per relevant doctor/channel/purpose while keeping consent events append-only.
- Use foreign keys or equivalent integrity guarantees for references to doctors, campaigns, pipeline versions, templates, and messages.
- Keep immutable content snapshots for messages actually submitted.
- Ensure idempotency keys are unique in the scope required to prevent duplicate provider requests.
- Do not store credentials in regular application tables as plain text; use a secret manager or encrypted secret storage.
- Use migrations. Do not use runtime schema mutation as a substitute for reviewed migrations.

---

## 8. Message and Pipeline State Model

### 8.1 Message states

Use internal states that preserve uncertainty:

- `draft`
- `queued`
- `sending`
- `submitted` — provider accepted the request; not necessarily delivered.
- `sent` — only when a provider event/documentation defines this meaning.
- `delivered`
- `read`
- `failed`
- `uncertain` — request may have been accepted but the final result is unknown.
- `skipped`
- `cancelled`

Not every provider reports all states. The UI must show unsupported statuses as unavailable, not as false certainty.

### 8.2 Step states

- `pending`
- `ready`
- `running`
- `waiting`
- `completed`
- `skipped`
- `blocked`
- `failed`
- `uncertain`
- `cancelled`

### 8.3 State transition rules

- A step cannot progress to `completed` merely because an API request was started.
- An uncertain send must not be blindly resent.
- A cancelled or blocked step cannot restart without an explicit authorized action and fresh eligibility validation.
- If consent changes while a message is queued, the send gate blocks it at execution time even if it was eligible when queued.
- A new pipeline version must not silently alter in-flight enrollments.
- Every manual override must record actor, reason, prior state, and new state.

---

## 9. Application Architecture

### 9.1 Recommended implementation stack

Use this stack unless there is a documented technical reason to change it:

- **Frontend:** React + TypeScript using a modern component-based UI framework such as Next.js or Vite. Choose one and avoid running two frontends.
- **Backend API:** Python + FastAPI.
- **Primary database:** PostgreSQL.
- **Background jobs:** Celery with Redis or an equivalent durable job system with explicit retry, deduplication, and monitoring. Do not use only FastAPI background tasks for long-running campaigns.
- **File storage:** local development storage behind a storage interface; production object storage or a secure equivalent.
- **Authentication:** proven session/JWT implementation with secure cookie/token handling, password hashing, and role checks.
- **AI adapter (P1):** optional Ollama/local model and/or configurable provider adapter.
- **Deployment:** Docker-based local and production configuration, environment-specific settings, health checks, and documented migrations/backups.

Use a modular monolith initially: one application codebase with clear modules and separate background worker processes. Do not create microservices without evidence that deployment or scaling requires them.

### 9.2 Backend modules

Organize code into clear modules such as:

- `auth` and `users`
- `doctors` and `specialties`
- `consent` and `suppression`
- `templates` and `media`
- `pipelines` and `pipeline_versions`
- `campaigns` and `enrollments`
- `scheduler` and `jobs`
- `messaging` and `providers`
- `webhooks` and `inbox`
- `followups` and `tasks`
- `knowledge` and `ai_assistance` (P1)
- `analytics` and `reports`
- `audit` and `system_health`

All business logic must live in testable services, not only in API route handlers or frontend components.

### 9.3 Key architecture boundaries

- Frontend talks to backend APIs; it does not call provider APIs with secret credentials.
- Pipeline scheduling uses a persistent job system and the database as source of truth.
- All outbound transports implement one messaging adapter interface.
- Every send request passes through a shared `OutboundEligibilityService` and policy validation.
- Webhooks are authenticated, persisted, deduplicated, and processed idempotently.
- AI can propose a structured action, but normal application services validate and execute it.
- Structured price/offer/product facts are authoritative in the database; vector search is supplementary.
- The application remains usable for CRM and pipeline configuration if AI is down.

### 9.4 Core API capabilities

Exact endpoint names may vary, but the API must provide equivalent capabilities:

- Auth: sign in/out, current user, authorized user administration.
- Doctors: create/list/read/update/archive, filter/search, import preview/commit, export, duplicate resolution.
- Specialties: list/create/update/deactivate and map default pipelines.
- Consent: retrieve eligibility, record verified consent, record opt-out, review consent events, inspect block reasons.
- Templates/media: CRUD, upload/validation, versioning, approval, provider status.
- Pipelines: CRUD, step editing, validation, preview, approval, version history, activation.
- Campaigns: draft, audience preview, validate, approve, schedule, launch, pause, resume, cancel, progress, result export.
- Messages/inbox: conversation list/detail, send approved reply, assign, note, pause, human takeover, resolve.
- Tasks: create/list/update/complete/cancel/reschedule follow-ups.
- Webhooks: provider inbound messages and message status events with signature verification and deduplication.
- System: readiness/liveness health, provider health, job queue state, audit search, emergency stop.
- AI (P1): classify, summarize, draft, review, approve/reject, per-intent enablement.

Protect all routes with authentication, authorization, validation, rate controls where relevant, and audit requirements.

---

## 10. Security, Privacy, and Reliability

### 10.1 Authentication and authorization

- Hash passwords with a current, purpose-built password hashing algorithm.
- Use secure, HttpOnly cookies or an equally secure documented session design.
- Implement CSRF protection where cookie-based sessions require it.
- Rate-limit authentication and sensitive administrative actions.
- Require explicit authorization for campaign launches, consent overrides, provider credential changes, and AI auto-reply activation.
- Log access to sensitive administrative records at an appropriate level.

### 10.2 Secrets and provider security

- Store access tokens, API keys, signing secrets, and database credentials in environment variables for local development and a secret manager or secure deployment secret store for production.
- Do not expose provider secrets to the browser.
- Verify webhook signatures according to the provider's current specification.
- Redact secrets and personal data from logs.
- Rotate credentials through a documented process.

### 10.3 Personal data handling

- Collect only the information needed for permitted business communication and relationship management.
- Support configurable retention and deletion/archive policies consistent with applicable requirements.
- Keep an opt-out suppression record sufficient to prevent accidental re-contact.
- Restrict data exports to authorized roles and record sensitive exports.
- Provide a documented backup, restore, and deletion procedure.
- Do not place doctor personal data into external AI services unless the configured provider, contractual terms, and business policy explicitly permit it.

### 10.4 Reliability and observability

- Structured application logs with correlation IDs.
- Health checks for API, database, Redis/job queue, file storage, messaging provider, webhook processing, and optional AI provider.
- Metrics for queue depth, job age, retries, send latency, webhook lag, errors, and provider status.
- Graceful behavior when external providers are unavailable.
- Backup/restore procedure and a tested recovery path.
- Configurable global emergency stop that blocks new outbound messages while leaving CRM and audit functions accessible.
- No silent failures: stuck or failed jobs must surface in the UI.

---

## 11. Compliance and Messaging Policies

This section defines product behavior, not legal advice. The business owner must confirm that message content, consent collection, data handling, and use cases comply with applicable law and the provider's current policies.

- Use the official WhatsApp Business Platform/Cloud API or a vetted integration for supported unattended production sending.
- Obtain and store suitable permission for the channel and message purpose before promotional messaging.
- Respect opt-out requests and maintain a suppression mechanism across all campaigns and manual/AI send paths.
- Use approved provider message templates for business-initiated messaging whenever current platform policy requires them.
- Track the customer-service window and template requirement based on current provider rules rather than assuming all free-form messages are allowed.
- Provide an obvious route to human support and do not let automation trap a recipient in a loop.
- Make it possible to inspect message content and audience before launch.
- Configure maximum contacts per campaign, maximum messages per contact, time windows, and quiet hours as business safety controls—not as a guarantee against provider restrictions.
- Do not build controls intended to hide automation, rotate numbers, evade policy enforcement, or bypass technical restrictions.
- Keep records of template approval status, consent events, launch approvals, opt-outs, message attempts, and operator overrides.
- Recheck current official documentation before release because provider requirements, capabilities, and prices can change.

---

## 12. Error Handling and Edge Cases

The application must explicitly handle at least the following cases:

1. Invalid or unsupported phone number.
2. Duplicate normalized phone number.
3. Missing name or specialty mapping.
4. Missing, unknown, expired, or conflicting consent evidence.
5. Opted-out doctor reappears through CSV import.
6. Imported data attempts to overwrite consent or opt-out state.
7. Pipeline has an unapproved template or missing asset.
8. Provider template is rejected, paused, or no longer approved.
9. Provider credentials expire or become invalid.
10. Provider is unavailable or returns a transient error.
11. Provider accepts a send but the application times out before receiving confirmation.
12. Delivery/read status is not provided by the provider.
13. Duplicate or out-of-order webhook event.
14. Inbound sender cannot be matched confidently to a doctor.
15. Opt-out arrives while a send is queued or a worker has claimed a job.
16. Doctor replies while later pipeline steps are scheduled.
17. Campaign is paused or cancelled during execution.
18. Worker crashes during an outbound attempt.
19. User changes a pipeline after some enrollments have started.
20. Asset is archived after being referenced by a live enrollment.
21. Placeholder value is missing or contains unexpected markup.
22. AI returns invalid JSON, an unsupported claim, or low-confidence classification.
23. AI provider is unavailable.
24. Two workers attempt the same scheduled job.
25. Database, Redis, storage, or webhook processing becomes unavailable.
26. Operator loses permission or session while a sensitive action is pending.
27. Campaign launch is attempted while emergency stop is active.
28. The selected provider does not support a required media type or inbound event.
29. The campaign would exceed configured frequency/follow-up limits.
30. An offer expires after campaign creation but before a later step is due.

Each case must produce a safe state, a useful operator-visible reason, and an audit record where appropriate. It must not cause silent message duplication or an unauthorized send.

---

## 13. Testing Strategy and Acceptance Criteria

### 13.1 Unit tests

Cover:

- Phone normalization and validation.
- Duplicate detection and import mapping.
- Consent eligibility for each state and purpose.
- Opt-out precedence over campaign eligibility.
- Pipeline schedule calculation and timezone handling.
- Template placeholder validation/rendering.
- Pipeline version pinning.
- Campaign/enrollment state transitions.
- Retry policy and uncertain outcome handling.
- Inbound webhook deduplication.
- Role permissions and required audit events.
- AI structured-output validation if enabled.

### 13.2 Integration tests

Cover:

- Database migrations and constraints.
- CSV/XLSX import preview and commit.
- Scheduler claiming jobs without duplicate execution.
- Messaging adapter against a mocked provider and, in a separate staging setup, provider sandbox/test facilities when available.
- Provider status webhook processing and signature validation.
- Inbound message persistence and opt-out handling.
- Pause/resume/cancel behavior.
- Crash/restart recovery of scheduled work.
- Human approval and audit workflow.

### 13.3 End-to-end tests

At minimum:

**Scenario A — eligible doctor pipeline:** create a test doctor with suitable recorded consent, map a specialty, build/approve a pipeline using test content, preview, launch a test campaign, run one or more steps through a mocked provider, and verify expected state transitions.

**Scenario B — unknown consent:** create/import a doctor with unknown consent and attempt to enroll. The application must block sending and explain the reason.

**Scenario C — opt-out race:** queue a step, record an opt-out before execution, then run the worker. No outbound send may occur; the job must be suppressed and audited.

**Scenario D — restart recovery:** schedule a step, stop and restart worker/API processes, and verify there is no lost or duplicated send.

**Scenario E — uncertain send:** simulate a timeout after a provider may have accepted the message. The application must mark the result uncertain and must not blindly retry.

**Scenario F — inbound reply:** inject a webhook, verify deduplication, conversation association, message persistence, and configured pause/review behavior.

**Scenario G — template unavailable:** simulate rejected/paused provider template and verify the relevant step is blocked before the send attempt.

**Scenario H — version change:** edit a live pipeline and verify existing enrollments remain pinned to the previous approved version unless explicitly migrated.

**Scenario I — unauthorized action:** attempt campaign launch, consent override, or AI auto-reply enablement with an insufficient role and verify rejection plus security audit.

**Scenario J — emergency stop:** enable the global stop and verify no campaign, follow-up, direct send, or AI send passes the common outbound gate.

### 13.4 Release acceptance criteria

P0 is accepted only when:

- A user can create and manage doctors and specialty values.
- Imports are previewed and validated before saving.
- Unknown or opted-out contacts cannot receive promotional messages.
- A specialty can map to a configurable pipeline without code changes.
- The owner can build, validate, approve, and preview a pipeline with ordered steps and attachments.
- Campaigns can be launched only by authorized users after an eligibility preview.
- Scheduled steps survive restarts and do not duplicate under tested concurrency.
- Every outbound path uses the same eligibility and policy checks.
- Message/template/pipeline versions are auditable.
- The interface distinguishes submitted, delivered, read, failed, and uncertain states correctly for the provider's capabilities.
- Opt-outs suppress pending messages and remain suppressed across imports and restarts.
- Human pause/takeover and global emergency stop work.
- Provider/webhook failures are visible to operators.
- Core security and recovery scenarios pass automated tests.
- Deployment, configuration, migrations, backup, restore, and troubleshooting are documented.

P1 AI functionality is accepted separately and must not be used as a reason to skip P0 requirements.

---

## 14. Implementation Plan

### Phase 1 — Foundations

- Create application structure, environment configuration, authentication, and RBAC.
- Set up PostgreSQL migrations, Redis/job queue, logging, and health checks.
- Implement doctor, specialty, consent event, and audit models/services.
- Build doctor list/detail/edit and import preview/commit.

**Exit criteria:** CRM is persistent, role-protected, tested, and consent evidence is auditable.

### Phase 2 — Content and pipeline builder

- Implement media library and template editor with versioning.
- Implement pipeline configuration, validation, previews, specialty mapping, and approval.
- Implement pipeline version pinning and starter blueprint configuration.

**Exit criteria:** The owner can configure a complete specialty sequence with their own copy and assets without code edits.

### Phase 3 — Messaging provider and consent gate

- Implement the provider adapter and staging configuration.
- Implement common outbound eligibility service and policy checks.
- Implement message attempts, idempotency, provider status normalization, and webhook handling.
- Implement template capability/status validation.

**Exit criteria:** A test message can be sent through a supported provider in a controlled environment and every send is logged and consent-checked.

### Phase 4 — Campaign scheduler and execution

- Implement campaigns, audiences, enrollments, persistent scheduled jobs, retry/reconciliation rules, pause/resume/cancel, and recovery.
- Add campaign progress dashboard and operational error views.
- Add concurrency and restart tests.

**Exit criteria:** A campaign executes a configured pipeline safely, with no duplicate step execution in tested conditions.

### Phase 5 — Inbox, opt-outs, follow-ups

- Implement inbound messages, conversation timeline, assignment, operator tasks, and manual pause/takeover.
- Implement deterministic opt-out handling and pending-job suppression.
- Implement configurable follow-up policies and quiet hours.

**Exit criteria:** Incoming replies and opt-outs are persisted, opt-outs block all covered future outbound activity, and operators can handle conversations.

### Phase 6 — Analytics and operational hardening

- Add campaign, pipeline, and reliability reporting.
- Add backups, restoration test, retention policies, alerts, and deployment runbooks.
- Complete security review and full end-to-end test suite.

**Exit criteria:** P0 release criteria pass in a staging environment.

### Phase 7 — AI assistance (optional P1)

- Add approved knowledge store and product-fact review.
- Add intent classification and grounded draft generation in draft-only mode.
- Add review queue and quality measurement.
- Consider selected low-risk auto-replies only after separate approval and test evidence.

**Exit criteria:** AI remains disabled by default; only specifically approved intents can be enabled, with source traceability, fallback, and human takeover.

### Phase 8 — Optional integrations (P2)

- Add selected CRM, calendar, email, workflow, or order integrations only after the core platform is stable.
- Preserve consent, audit, and authorization rules across all external workflows.

---

## 15. Deployment and Operations

### 15.1 Local development

Provide:

- `.env.example` containing variable names and safe placeholder values only.
- Docker Compose or an equally straightforward setup for the web/API, PostgreSQL, Redis/job queue, and worker.
- Database migration commands.
- Seed data using fake doctors, fake phone numbers, and non-production message templates.
- Mock provider mode that can test full workflows without sending real messages.
- Clear setup, test, lint, and run commands.

### 15.2 Production setup

Document:

- Supported runtime and dependencies.
- Required environment variables and secret storage.
- Database migrations and rollback approach.
- Worker and scheduler process deployment.
- Webhook URL, TLS, signature verification, and provider setup.
- File storage and backups.
- Health checks, logging, monitoring, and alert setup.
- Backup restoration, provider outage, credential rotation, and campaign recovery.
- How to enable/disable AI and how to test safely before activation.

### 15.3 Environments

Separate local, staging, and production configuration. Production credentials and data must never be copied into test fixtures. Staging must use a mock or provider-approved test configuration and must not send to real doctors without a deliberate, documented test authorization.

---

## 16. Configuration Requirements

Provide documented configuration for:

- Database and job queue connections.
- File storage.
- Authentication/security settings.
- Messaging provider credentials and webhook verification.
- Default timezone and business hours.
- Campaign message/follow-up limits and quiet hours.
- Consent purposes/categories.
- Retry limits and retryable error classes.
- AI provider/model settings, with AI disabled by default.
- Logging level and redaction rules.
- Retention and backup settings.
- Emergency stop state and authorized control.

Secrets must not be included in `.env.example`, frontend builds, API responses, logs, or source control.

---

## 17. Explicit Decisions and Assumptions

1. The product is a **new build**. Do not assume an existing repository, database schema, Playwright implementation, or prior UI exists.
2. The core workflow is configurable specialty-based outreach and follow-up, not unrestricted autonomous AI.
3. The operator supplies the actual message templates, videos, photos, brochures, offers, and schedule. Starter blueprints remain drafts until configured and approved.
4. Specialty mapping and pipeline changes must be possible from the application UI.
5. The production unattended messaging target is the official WhatsApp Business Platform/Cloud API or a vetted provider. A browser script is not the default production transport.
6. A lower-cost manual-assisted workflow may be considered later, but it cannot be represented as unattended, provider-verified sending unless there is a trustworthy supported signal.
7. WhatsApp delivery is not guaranteed; the application must accurately report provider states and account for policy changes.
8. AI features are optional and disabled by default. MVP operations must work without LLMs.
9. PostgreSQL + FastAPI + React/TypeScript + a persistent background job system are the recommended baseline. Alternatives require a documented reason and must meet the same reliability requirements.
10. The owner is responsible for confirming current legal, privacy, consent, provider, and product-claim requirements before production launch.

---

## 18. Definition of Done

The initial production-ready version is done when all of the following are true:

- The application runs locally from documented instructions and can be deployed to staging.
- Authentication and permissions are active.
- Doctor records, specialty records, consent events, templates, media references, pipeline versions, campaigns, enrollments, messages, and audit events are persisted.
- The operator can add a doctor, map specialty to a pipeline, preview the sequence, and launch an authorized campaign.
- The system never sends to an unknown/blocked/opted-out contact through a supported outbound path.
- The system checks message/provider policy immediately before sending.
- Pipeline scheduling and recovery are persistent and tested.
- Uncertain sends are surfaced and are not blindly retried.
- Inbound messages and provider statuses are deduplicated when the provider supports them.
- Opt-outs cancel/suppress pending covered outbound communication and remain effective after restart and re-import.
- An operator can pause a conversation, campaign, or all outbound automation.
- AI is disabled by default and cannot bypass the same policies if later enabled.
- Critical actions are auditable.
- The acceptance tests in Section 13 pass.
- There is no fake delivery/read status, hard-coded live pricing, invented product content, or undocumented dependency on a personal browser session.
- Setup, migration, testing, deployment, recovery, and troubleshooting documentation is complete.

**Product definition:** AyuDevices is a specialty-based doctor relationship platform with configurable communication pipelines, provider-supported messaging, consent enforcement, persistent scheduling, inbox/workflow management, and optional grounded AI assistance. Its priority is reliable and controlled communication—not unrestricted automation.

---

## 19. Reference Links

The following official documentation should be checked during implementation and again before release because platform requirements can change:

- WhatsApp Business Messaging Policy: https://whatsappbusiness.com/policy/
- WhatsApp Business Terms and Conditions: https://whatsappbusiness.com/terms-and-conditions/
- WhatsApp Business Platform / Cloud API documentation: https://developers.facebook.com/docs/whatsapp/

In particular, validate opt-in, opt-out, approved template, customer-service-window, webhook, media, quality, and pricing requirements against the current provider documentation. This PRD does not hard-code a permanent provider price or guarantee that any sending method will avoid account restrictions.
