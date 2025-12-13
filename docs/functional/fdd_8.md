## 8. Communication & Collaboration – Technical Design (Supabase + Vercel)

This document specifies the technical design for **Section 8 – Communication & Collaboration** of the HVAC/Plumbing/Electrical field service platform, assuming:

- **Backend**: Supabase (PostgreSQL, Auth, Storage, Realtime, Edge Functions)
- **Frontend**: React/Next.js on Vercel (or similar modern SPA/SSR framework)
- **Mobile**: React Native or Flutter app consuming the same Supabase backend

The goal is to define the **data model, APIs, realtime behavior, integrations, security, UX flows, and observability** in enough detail that an LLM can decompose this into granular implementation tasks.

---

### 8.1 Scope & User Roles

Communication & collaboration features cover all **messaging and coordination** between:

- **Customer**: Homeowner/property manager using web customer portal or mobile app
- **Technician**: Field technician using mobile app
- **Dispatcher/CSR**: Back-office staff using operations dashboard
- **Manager/Admin**: Supervisors and business owners

Core channels:

- **System Notifications**: Status changes (job created, scheduled, on-the-way, completed), reminders, payment events
- **Conversations**:
  - Customer ↔ Dispatcher/CSR
  - Customer ↔ Technician (with business controls)
  - Internal Team Chat (dispatcher ↔ techs; tech ↔ tech; group channels per job/crew)
- **External Channels**:
  - SMS
  - Email
  - Push notifications (mobile)
  - Optional in-app chat widgets

AI capabilities:

- **Message classification & routing**
- **Sentiment analysis and escalation**
- **Suggested replies & templates**
- **Summaries of long threads per job/customer**

---

### 8.2 Core Functional Requirements

- **Integrated messaging**:
  - Send/receive messages via:
    - In-app chat (web & mobile)
    - SMS (customer phone)
    - Email (customer email)
  - Messages are **normalized into a single conversation model**, regardless of channel.
- **Conversation types**:
  - **Job Conversations**: Scoped to a specific work order / job.
  - **Customer Conversations**: General (non-job-specific) threads with a customer.
  - **Internal Channels**: Team/department/group channels (e.g., “Dispatch Room”, “HVAC Techs”).
  - **Direct Messages (DMs)**: Between two internal users.
- **Group chat**:
  - Multiple participants (internal + external, depending on type).
  - Join/leave membership tracking.
  - Role-specific capabilities (e.g., tech cannot see private management comments unless allowed).
- **Notifications**:
  - Configurable notification events (job scheduled, tech en route, arrival window changes, estimate ready, invoice due, payment received, etc.).
  - Multi-channel delivery (SMS, email, push, in-app).
  - User-level preferences (opt-in/out per type/channel where legally allowed).
- **AI features**:
  - **Sentiment tagging** on customer-facing messages.
  - **Risk & escalation flags** when negative sentiment or escalation keywords detected.
  - **Suggested reply templates** based on message context and company tone.
  - **Conversation/job summaries** for internal users.

---

### 8.3 Data Model (Supabase / PostgreSQL)

> Note: Table names assume an existing core schema with `users`, `customers`, and `jobs` tables.

#### 8.3.1 Enum Types

Define PostgreSQL enums (or lookup tables):

- `message_channel`:
  - `in_app`
  - `sms`
  - `email`
  - `push`
  - `system`
  - `integration` (e.g., 3rd-party chat widget)

- `conversation_type`:
  - `job`
  - `customer`
  - `internal_channel`
  - `direct_message`

- `message_direction`:
  - `inbound`
  - `outbound`

- `sentiment_label`:
  - `positive`
  - `neutral`
  - `negative`
  - `mixed`
  - `unknown`

#### 8.3.2 Tables

**Table: `conversations`**

- `id` (UUID, PK, default `gen_random_uuid()`)
- `type` (`conversation_type`, NOT NULL)
- `job_id` (FK → `jobs.id`, nullable; required for `type = 'job'`)
- `customer_id` (FK → `customers.id`, nullable; required for `type = 'customer'` or `type = 'job'` if customer exists)
- `internal_channel_key` (text, nullable; unique slug for internal channels, e.g., `dispatch_room`)
- `created_by_user_id` (FK → `users.id`, nullable for system-created)
- `title` (text, nullable; e.g., "Job #1234 - Smith A/C Repair")
- `is_archived` (boolean, default false)
- `created_at` (timestamptz, default `now()`)
- `updated_at` (timestamptz, default `now()`)

Indexes:

- btree on `job_id`, `customer_id`, `type`
- partial index on `internal_channel_key` where not null

**Table: `conversation_participants`**

- `id` (UUID, PK)
- `conversation_id` (FK → `conversations.id`, ON DELETE CASCADE)
- `user_id` (FK → `users.id`, nullable; internal user)
- `customer_id` (FK → `customers.id`, nullable; external customer)
- `role` (text, e.g., `owner`, `member`, `viewer`)
- `is_muted` (boolean, default false)
- `last_read_message_id` (FK → `messages.id`, nullable)
- `created_at` (timestamptz, default `now()`)

Constraints:

- Either `user_id` OR `customer_id` must be non-null (not both null).
- Unique (`conversation_id`, `user_id`) where `user_id` not null.
- Unique (`conversation_id`, `customer_id`) where `customer_id` not null.

**Table: `messages`**

- `id` (UUID, PK)
- `conversation_id` (FK → `conversations.id`, ON DELETE CASCADE)
- `sender_user_id` (FK → `users.id`, nullable; internal sender)
- `sender_customer_id` (FK → `customers.id`, nullable; external sender)
- `sender_display_name` (text, optional denormalized field)
- `body` (text, nullable if message has only attachments/system template)
- `channel` (`message_channel`, NOT NULL)
- `direction` (`message_direction`, NOT NULL)
- `is_internal_note` (boolean, default false; not visible to customers)
- `sentiment` (`sentiment_label`, default `unknown`)
- `sentiment_score` (numeric, nullable; -1.0..1.0)
- `requires_follow_up` (boolean, default false)
- `escalation_level` (smallint, default 0; e.g., 0=none,1=CSR,2=Manager)
- `external_message_id` (text, nullable; provider message ID for SMS/email)
- `metadata` (jsonb, default '{}'::jsonb; store provider payload, tags, etc.)
- `created_at` (timestamptz, default `now()`)
- `delivered_at` (timestamptz, nullable)
- `read_at` (timestamptz, nullable, for customer side)

Indexes:

- btree on `conversation_id, created_at`
- btree on `external_message_id`
- gin index on `metadata` if needed

**Table: `message_attachments`**

- `id` (UUID, PK)
- `message_id` (FK → `messages.id`, ON DELETE CASCADE)
- `storage_object_path` (text, NOT NULL; Supabase Storage key)
- `file_name` (text)
- `mime_type` (text)
- `size_bytes` (bigint)
- `created_at` (timestamptz, default `now()`)

**Table: `notification_templates`**

- `id` (UUID, PK)
- `key` (text, unique, e.g., `job_scheduled_customer_sms`)
- `channel` (`message_channel`, NOT NULL)
- `locale` (text, default `en-US`)
- `subject_template` (text, nullable; for email)
- `body_template` (text, NOT NULL; plain text or markdown)
- `is_active` (boolean, default true)
- `created_at` (timestamptz, default `now()`)
- `updated_at` (timestamptz, default `now()`)

**Table: `notification_events`**

- `id` (UUID, PK)
- `key` (text, unique, e.g., `job_created`, `job_scheduled`, `job_status_changed`, `invoice_sent`)
- `description` (text)
- `default_channel_config` (jsonb; e.g., `{ "sms": true, "email": true, "push": false }`)
- `created_at` (timestamptz, default `now()`)

**Table: `user_notification_preferences`**

- `id` (UUID, PK)
- `user_id` (FK → `users.id`)
- `event_key` (FK → `notification_events.key`)
- `channel_overrides` (jsonb; e.g., `{ "sms": false, "email": true }`)
- `created_at` (timestamptz, default `now()`)
- `updated_at` (timestamptz, default `now()`)

**Table: `customer_notification_preferences`**

- `id` (UUID, PK)
- `customer_id` (FK → `customers.id`)
- `event_key` (FK → `notification_events.key`)
- `channel_overrides` (jsonb)
- `created_at` (timestamptz, default `now()`)
- `updated_at` (timestamptz, default `now()`)

**Table: `conversation_summaries`**

- `id` (UUID, PK)
- `conversation_id` (FK → `conversations.id`, unique)
- `summary_text` (text, NOT NULL)
- `last_message_id` (FK → `messages.id`; summary covers up to this message)
- `generated_by` (text, e.g., `gpt-4.1`)
- `created_at` (timestamptz, default `now()`)

---

### 8.4 Authentication, Authorization & RLS

Supabase Auth is assumed for internal users and customers, with JWT-based access.

#### 8.4.1 Authorization Rules (Conceptual)

- **Internal users**:
  - Can read/write conversations where:
    - They are a participant OR
    - They have a role with global/channel-level access (e.g., dispatchers can see all job & internal channels; managers see all).
  - Can read/write internal notes (`is_internal_note = true`).
  - Cannot impersonate customers; outbound customer messages are clearly labeled.

- **Customers**:
  - Can access:
    - Conversations where they are a participant (e.g., primary customer on a job).
    - Only **non-internal** messages (`is_internal_note = false`).
  - Can send messages to allowed conversations (job/customer threads scoped to their account).

#### 8.4.2 RLS Policies (High-Level)

Example policy sketches (not full SQL):

- On `conversations`:
  - Allow select for internal users if:
    - User is `conversation_participants.user_id`, OR
    - User has role `dispatcher` or `manager` or `admin` (role info from JWT).
  - Allow select for customers if:
    - JWT `customer_id` present AND customer participates (via `conversation_participants.customer_id`) or conversation’s `customer_id` matches.

- On `messages`:
  - Select:
    - For internal users: same rules as conversations.
    - For customers: same rules AND `is_internal_note = false`.
  - Insert:
    - For internal users: must be participant or have role for channel.
    - For customers: must be participant and `is_internal_note = false`.

- On `conversation_participants`:
  - Insert/update allowed only for internal roles with appropriate permissions (e.g., dispatcher, admin).

Exact RLS SQL should be defined with Supabase’s `auth.jwt()` claims (e.g., `role`, `user_id`, `customer_id`).

---

### 8.5 Realtime Messaging & Presence

Use **Supabase Realtime** on the `messages` table:

- Frontend (web/mobile) subscribes to:
  - `messages` where `conversation_id` in list of conversations the user participates in.
  - Optionally `conversation_participants` for presence/membership changes.

Client behavior:

- Upon receiving a new `message` event:
  - Append to local conversation thread.
  - Update unread counts (per `conversation_participants.last_read_message_id`).
  - Trigger sound/visual indication if conversation is active or not.

Presence (optional, phase 2):

- Use a separate Supabase Realtime channel (or Presence feature if available) keyed by `conversation_id` with:
  - `user_id`
  - `is_typing` flag
  - `last_seen_at`

---

### 8.6 External Channel Integrations

> Exact providers can be pluggable (e.g., Twilio for SMS, SendGrid/Postmark for email, Firebase/APNS/FCM for push). Here we define generic hooks.

#### 8.6.1 SMS

- Outbound:
  - Edge Function `send_sms_message`:
    - Input: `conversation_id`, `body`, optional `customer_id`.
    - Resolves customer phone from `customers` table.
    - Calls SMS provider API.
    - Writes a `messages` row with:
      - `channel = 'sms'`
      - `direction = 'outbound'`
      - `external_message_id` from provider.
  - Triggered from UI when user sends a message choosing SMS channel or via notifications engine.

- Inbound:
  - Webhook Edge Function `sms_inbound_webhook`:
    - Verifies provider signature.
    - Parses sender phone, message content, provider message ID.
    - Maps phone to a `customer` and appropriate `conversation` (existing job/customer thread or create new).
    - Inserts `messages` row:
      - `channel = 'sms'`
      - `direction = 'inbound'`
      - `sender_customer_id` set.
    - Optionally runs AI sentiment/triage.

#### 8.6.2 Email

- Outbound:
  - Edge Function `send_email_message`:
    - Input: `conversation_id`, `subject`, `body`, `to_email` (or infer from customer).
    - Calls email provider API with tracking headers or references to tie replies back.
    - Inserts `messages` row with `channel = 'email'`, `direction = 'outbound'`.

- Inbound:
  - Provider webhook (or inbound email parsing) to Edge Function `email_inbound_webhook`:
    - Validates authenticity (signatures, tokens).
    - Uses `to` / `reply-to` / custom headers to resolve related conversation.
    - Inserts `messages` row with `channel = 'email'`, `direction = 'inbound'`, sets `sender_customer_id`.

#### 8.6.3 Push Notifications

- Use mobile push provider (Firebase Cloud Messaging / APNS) managed by the mobile app backend integration:
  - Supabase tables store device tokens per user.
  - Edge Function `send_push_notification`:
    - Input: `user_id`, `title`, `body`, `deep_link` (e.g., conversation URL).
    - Sends push to all registered tokens for that user.
    - Logs to an internal `push_logs` table (optional).

---

### 8.7 Notification Orchestration

Centralized orchestration uses **Edge Functions** subscribed to domain events (via db triggers or application events).

#### 8.7.1 Event Sources

Typical domain events (not exhaustive):

- `job_created`
- `job_scheduled`
- `job_status_changed`
- `job_technician_assigned`
- `technician_en_route`
- `technician_arrived`
- `estimate_ready`
- `invoice_sent`
- `payment_received`

Implementation pattern:

- For each relevant table (e.g., `jobs`, `invoices`, `payments`), create database triggers or application-level event emitters that:
  - Insert into a simple `domain_events` table, OR
  - Directly invoke a Supabase Edge Function via HTTP RPC.

Example `domain_events` columns:

- `id` (UUID, PK)
- `event_key` (text, references `notification_events.key`)
- `payload` (jsonb; contains `job_id`, `customer_id`, etc.)
- `created_at` (timestamptz)

#### 8.7.2 Orchestrator Function

Edge Function `notification_orchestrator`:

- Input: `event_key`, payload (`job_id`, `customer_id`, etc.).
- Steps:
  1. Resolve affected entities (job, customer, technician).
  2. Determine **target audience** (customer, tech, dispatcher, manager).
  3. For each audience, evaluate:
     - `default_channel_config` from `notification_events`.
     - Overrides from `user_notification_preferences` or `customer_notification_preferences`.
  4. For each channel (SMS/email/push/in-app) that is enabled:
     - Load `notification_templates` by (`event_key`, `channel`, `locale`).
     - Render content (string interpolation, e.g., job date/time, address).
     - Call respective sending functions (`send_sms_message`, `send_email_message`, `send_push_notification`) or create in-app `messages`.

Notifications that should appear in the conversation feed:

- For example `job_status_changed` to customer:
  - `notification_orchestrator` **also** inserts a `messages` row into the appropriate `conversation` with:
    - `channel = 'system'`
    - `direction = 'outbound'`
    - `is_internal_note = false`

---

### 8.8 AI & LLM Integration

> Actual LLM provider may be OpenAI, Anthropic, or another; design Edge Functions so provider is abstracted via environment variables and internal service layer.

#### 8.8.1 Sentiment Analysis

Edge Function `analyze_message_sentiment`:

- Trigger:
  - Database trigger on `messages` **after insert** where:
    - `direction = 'inbound'` AND
    - `sender_customer_id` is not null AND
    - `channel` in (`in_app`, `sms`, `email`).
- Steps:
  1. Fetch the new message text and basic context (customer, job if any).
  2. Call external sentiment model (or internal classification).
  3. Update `messages.sentiment` and `messages.sentiment_score`.
  4. If sentiment is `negative` and score below a configured threshold:
     - Set `requires_follow_up = true`, `escalation_level` accordingly.

#### 8.8.2 Escalation Rules

Edge Function `process_escalations` (can be part of the same function as sentiment):

- When a message is flagged with `requires_follow_up = true`:
  - Notify relevant roles (dispatcher/manager) via:
    - In-app message (internal note in same conversation).
    - Optional SMS/email to on-call manager for severe cases.
  - Optionally create a **task** or **ticket** in a `tasks` table for follow-up.

#### 8.8.3 Suggested Replies

Edge Function `suggest_replies`:

- API route callable from frontend for a given `message_id` or recent conversation window.
- Steps:
  1. Retrieve last N messages from the conversation.
  2. Construct prompt with:
     - Conversation history.
     - Company guidelines (tone, policies).
     - Any job/customer context.
  3. Call LLM with system instructions to output 2–3 suggested replies.
  4. Return suggestions to client; **do not auto-send**.

Frontend behavior:

- When CSR/dispatcher opens a conversation with an inbound message:
  - UI calls `suggest_replies`.
  - Shows suggestions as clickable chips; on click, populates text area for editing.

#### 8.8.4 Conversation Summaries

Edge Function `generate_conversation_summary`:

- Trigger:
  - Manual (user clicks “Summarize” in UI) **or**
  - Scheduled (e.g., summarize conversations with more than N messages or older than X days).
- Steps:
  1. Fetch all messages since `conversation_summaries.last_message_id` (or entire conversation if no summary yet).
  2. Send condensed history to LLM with instructions:
     - Summarize key issues, decisions, commitments, and sentiment trends.
  3. Insert/update `conversation_summaries` with:
     - `summary_text`
     - `last_message_id` = latest message ID included.

Use cases:

- Quick intake for a manager taking over a difficult customer.
- Short summary attached to job record for post-job review.

---

### 8.9 Frontend UX Flows (Web & Mobile)

This section defines key screens and interactions; exact implementation left to UI engineers.

#### 8.9.1 Conversation List

- **Internal dashboard**:
  - Left-hand panel: list of conversations with:
    - Title (job/customer name), last message preview, timestamp.
    - Badges for:
      - Unread count.
      - Sentiment (color-coded; red for negative).
      - Escalation flag (icon).
  - Filters:
    - `All`, `My Conversations`, `Job`, `Customer`, `Internal`, `Escalated`.

- **Customer portal / mobile**:
  - List of job/customer conversations relevant to that customer.
  - Show job reference (e.g., “A/C Repair on 2025-06-10”).

Data fetching:

- API to fetch conversations for current user/customer with pagination and filter parameters.

#### 8.9.2 Conversation View

- Components:
  - Header:
    - Conversation title, participants (avatars).
    - For job-type: quick access to job status, address, scheduled time.
  - Message list:
    - Bubbles or threaded view, grouped by date.
    - Sender labels (customer vs internal staff vs system).
    - Badges for internal notes (only visible to internal users).
    - Attachment previews (images, PDFs, etc.).
  - Composer:
    - Text input.
    - Channel selector (if user can choose between in-app/SMS/email; default to in-app + mirrored to SMS if configured).
    - Attach file button (links to Supabase Storage).
    - For CSRs: show suggested replies (chips) when an inbound message is selected.

Interactions:

- Typing indicator updates presence channel.
- When user sends message:
  - Call messages API (insert row, optionally plus channel-specific send function).
  - Optimistically add to UI.

#### 8.9.3 Job Detail Page Integration

- Job detail view should show:
  - Embedded **job conversation** panel for that job.
  - Timeline of key notifications (system messages).

Data:

- Use `conversations` with `job_id = current_job.id`.

---

### 8.10 Observability, Logging & Auditing

- **Message logs**:
  - Every message is stored in `messages` with timestamps and sender metadata.
  - External provider IDs allow matching to provider logs.
- **Notification audits**:
  - For critical notifications (e.g., job scheduled, invoice sent), store:
    - Which channels were attempted.
    - Delivery status and timestamps (updated via provider webhooks).
- **Error handling**:
  - Edge Functions should:
    - Log errors to a centralized logging solution (e.g., Supabase logs + external).
    - Mark failed notifications in metadata for re-try.
- **Security/audit**:
  - Track who changed notification templates and preferences (`created_at`, `updated_at`, `created_by`, `updated_by` where applicable).

---

### 8.11 Non-Functional Requirements

- **Performance**:
  - Typing-to-message-send latency: target < 500ms for in-app messages.
  - Realtime updates to appear for other participants within ~1–2 seconds.
- **Reliability**:
  - At-least-once delivery semantics for notifications (idempotent webhooks and senders).
  - Graceful degradation if AI services are unavailable (no blocking of base messaging).
- **Compliance & Privacy**:
  - Respect customer notification preferences and legal requirements (e.g., consent for SMS).
  - Store communication history securely with access control and encryption at rest (Supabase Postgres).
- **Extensibility**:
  - Integration-agnostic service layer for SMS/email/push (swappable providers via environment configs).
  - AI functions designed with pluggable model providers.

---

### 8.12 LLM Task Decomposition Hints

For an LLM generating implementation tasks, consider breaking work down into at least the following areas:

- **Database & RLS**:
  - Create enums and tables defined in 8.3.
  - Implement Supabase RLS policies for conversations, messages, participants, and preferences.
- **Edge Functions**:
  - Messaging senders (`send_sms_message`, `send_email_message`, `send_push_notification`).
  - Webhooks (`sms_inbound_webhook`, `email_inbound_webhook`).
  - Orchestrator (`notification_orchestrator`).
  - AI-related functions (`analyze_message_sentiment`, `suggest_replies`, `generate_conversation_summary`).
- **Frontend**:
  - Conversation list & conversation view components (web + mobile).
  - Job detail page integration.
  - Notification preference UI.
  - Wiring Realtime subscriptions for messages and presence.
- **Integrations & DevOps**:
  - Configure providers (Twilio/SendGrid/etc.) and secrets.
  - Set up logging, monitoring, and error alerting for Edge Functions.

This design should be treated as the contract for implementing Communication & Collaboration features on top of the Supabase-based platform.


