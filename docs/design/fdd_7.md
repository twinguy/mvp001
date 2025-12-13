# Technical Design Document – Section 7: Customer Portal

## 1. Purpose & Scope

This Technical Design Document (TDD) defines the **Customer Portal** capabilities for the HVAC, Plumbing, and Electrical services platform described in `functional.md`. It is intended to be detailed enough for engineers and LLM-based task generators to decompose the Customer Portal module into concrete implementation tasks.

- **Source functional requirements (from `functional.md`, §7 Customer Portal)**:
  - Online booking and appointment management.
  - Service history and documentation access.
  - Real-time job status updates.
  - Secure messaging with technicians.
  - AI-powered self-service support (chatbots, FAQs).
- **Additional derived requirements (to create a complete, production-ready portal)**:
  - Customer authentication, account linking, and profile management.
  - Management of notification and communication preferences.
  - Viewing and managing upcoming appointments (reschedule/cancel within business rules).
  - Viewing quotes, invoices, and payments (when those modules are implemented).
  - Access to warranties, membership plans, and documents related to work performed.
  - Multi-tenant support (each service company’s customers access only their own data).

The Customer Portal must be implemented on **Supabase (PostgreSQL + Auth + Storage + Edge Functions)** and exposed primarily via a **web application (Next.js) hosted on Vercel**, in alignment with `tooling.md`. Mobile access may be via responsive web or dedicated apps using the same backend.

Out-of-scope for this document (but integrated with):
- Internal staff-facing UI (CSR, dispatcher, technician console).
- Detailed data model for Scheduling & Dispatch, Work Orders, Quoting & Invoicing, Inventory, etc. (defined in their own TDDs).

---

## 2. Target Platform & High-Level Architecture

### 2.1 Platform Alignment (from `tooling.md`)

- **Backend: Supabase**
  - PostgreSQL database (managed).
  - Supabase Auth for user authentication for both staff and customers.
  - Row-Level Security (RLS) for multi-tenancy and role-based access.
  - Supabase Storage for documents (invoices, quotes, attachments, manuals).
  - Supabase Edge Functions for:
    - Booking orchestration (validating availability, creating booking requests and/or work orders).
    - Real-time job status updates fan-out (e.g., notifying portal clients).
    - Self-service AI assistant orchestration (calling LLM and vector search APIs).
    - Notification orchestration (email/SMS/push) for appointment reminders and updates.
- **Frontend: Vercel**
  - Next.js web application (customer-facing portal).
  - Uses Supabase JS client for Auth, DB access (via PostgREST/RPC), and real-time subscriptions.
  - Deployed via Vercel with CI/CD and environment-specific configuration (dev/stage/prod).
- **Mobile (future option)**
  - Flutter or React Native app sharing the same Supabase backend.
  - Supabase mobile SDKs used to mirror core portal capabilities (booking, history, messaging, AI chat).

### 2.2 Logical Components

1. **Customer Portal Auth & Account Linking**
   - Customer account creation and login (password-based, OAuth optional).
   - Linking portal user to internal `customers` record via verified identifiers (email, phone, invite token, or booking reference).
   - Support for multiple contacts under the same customer (e.g., property managers).

2. **Booking & Appointment Management**
   - Online request and booking flow.
   - Appointment detail view, rescheduling, and cancellation within policy constraints.
   - Integration with Scheduling & Dispatch module (jobs/slots/technician assignment).

3. **Service History & Documentation**
   - List of past jobs and visits for a customer/location.
   - Access to attached documents: work orders, photos, invoices, quotes, manuals, warranties, membership details.
   - Download/print capability for PDFs and generated documents.

4. **Real-Time Job Status**
   - Tracking of current/next appointment status (e.g., scheduled, en route, on-site, completed).
   - Optional technician ETA and map view (if GPS/location is available).
   - Real-time updates using Supabase real-time channels or polling fallback.

5. **Secure Messaging with Technicians/Office**
   - Threaded conversations between the customer and office/technicians.
   - Unified with CRM interaction logging (leveraging `crm_interactions` where appropriate).
   - Typing indicators, message read status (optional).

6. **AI-Powered Self-Service Support**
   - Chat-style interface for troubleshooting, FAQs, and guidance (before or after booking).
   - Knowledge base content management on the back-office side; AI retrieval/response at runtime.
   - Session transcripts linked to customer records to assist CSRs and technicians.

7. **Settings & Preferences**
   - Profile, addresses, and contact management (surfaced from core CRM entities with appropriate permissions).
   - Notification preferences (channels, quiet hours, opt-in/opt-out).
   - Security (password change, MFA where supported, device/session management).

### 2.3 Multi-Tenancy & Access Control

Assume one Supabase project serves multiple companies (tenants) or at least supports role-based segmentation within a single organization:

- **Tenancy Key**: `org_id` present on all portal-related tables and core domain tables (e.g., `customers`, `work_orders`).
- **RLS Policies**:
  - End users only see rows where `org_id` matches the `org_id` of the company they belong to.
  - Portal customers can only access:
    - Their own `customer` record (and associated locations/jobs).
    - Data for which they are explicitly authorized (e.g., shared access for property managers).
  - Staff users (CSR, dispatcher, technician) have broader access via separate roles and policies.
- **Roles**:
  - `portal_customer` – end users of the portal.
  - `portal_manager` – customer-side admin (e.g., property manager) with permissions to manage multiple properties and contacts.
  - Internal roles (admin/CSR/tech) are defined elsewhere but must be considered for messaging and history display logic.

---

## 3. Domain Model & Data Design

### 3.1 Entities Overview

Key entities for the Customer Portal (reusing core entities where possible):

- **Customer** (`customers`) – canonical customer record (individual or organization).
- **Customer Location** (`customer_locations`) – properties or units where service is performed.
- **Work Order / Job** (`work_orders` or equivalent) – scheduled service visit/job, including status and technician assignment.
- **Quote & Invoice** (`quotes`, `invoices` or equivalent) – financial documents related to work.
- **Portal Account** – mapping between Supabase Auth user and `customer`/contact.
- **Portal Session / Audit** – record of important actions performed by portal users.
- **Portal Booking Request** – initial service request coming from the portal prior to full job creation or scheduling.
- **Portal Thread** – conversation context for secure messaging.
- **Portal Message** – individual messages within a thread (often backed by `crm_interactions`).
- **Notification Preferences** – per-customer and per-contact channel/quiet-hour settings.
- **Knowledge Base (KB)** – structured FAQs, articles, and documents used for AI retrieval.
- **AI Conversation** – chat sessions between the customer and AI assistant.
- **File & Document Links** – references to Supabase Storage objects for invoices, photos, manuals, etc.

> Note: Some entities like `customers`, `customer_locations`, and `crm_interactions` are defined in other TDDs; here we specify how the portal builds on them and add portal-specific tables.

### 3.2 Tables & Columns (Portal-Specific)

The following schema is conceptual; exact types and constraints should be implemented in PostgreSQL for Supabase.

#### 3.2.1 `portal_accounts`

Maps Supabase auth users to customers and defines portal-level roles.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `user_id` (UUID, FK -> `auth.users.id`, unique, not null).
- `customer_id` (UUID, FK -> `customers.id`, not null, on delete restrict).
- `primary_location_id` (UUID, FK -> `customer_locations.id`, nullable).
- `role` (enum: `customer`, `property_manager`, `billing_contact`, `admin`).
- `status` (enum: `active`, `invited`, `suspended`, `closed`).
- `invited_via` (enum: `self_signup`, `email_invite`, `csr_created`, nullable).
- `last_login_at` (timestamptz, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Indexes/constraints:
- Unique constraint: (`org_id`, `user_id`).
- Index on (`org_id`, `customer_id`).

#### 3.2.2 `portal_audit_events`

Tracks key actions performed by portal users for audit/compliance and debugging.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `portal_account_id` (UUID, FK -> `portal_accounts.id`, not null).
- `event_type` (text, not null) – e.g., `login`, `logout`, `booking_created`, `booking_canceled`, `profile_updated`.
- `event_payload` (jsonb, nullable) – structured meta (before/after diffs, entities affected).
- `ip_address` (inet, nullable).
- `user_agent` (text, nullable).
- `occurred_at` (timestamptz, default now()).

Indexes:
- `idx_portal_audit_events_org_id_occurred_at`.

#### 3.2.3 `portal_booking_requests`

Represents service requests initiated via the portal before they are fully accepted/converted to scheduled jobs.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `portal_account_id` (UUID, FK -> `portal_accounts.id`, not null).
- `customer_id` (UUID, FK -> `customers.id`, not null).
- `location_id` (UUID, FK -> `customer_locations.id`, not null).
- `preferred_service_date` (date, nullable).
- `preferred_time_window_start` (time, nullable).
- `preferred_time_window_end` (time, nullable).
- `flexible_date` (boolean, default false).
- `service_type` (text, not null) – e.g., `hvac_repair`, `plumbing_emergency`, `electrical_install`.
- `problem_description` (text, not null).
- `attachments` (jsonb, nullable) – references to Supabase Storage objects (photos/videos).
- `source_channel` (enum: `web_portal`, `mobile_app`, `ai_assistant`).
- `status` (enum: `submitted`, `under_review`, `approved`, `scheduled`, `rejected`, `canceled_by_customer`).
- `related_work_order_id` (UUID, FK -> `work_orders.id`, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Indexes:
- `idx_portal_booking_requests_customer_id`.
- `idx_portal_booking_requests_org_id_status_created_at`.

#### 3.2.4 `portal_notification_preferences`

Extends core CRM preferences with portal-specific notification settings.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `portal_account_id` (UUID, FK -> `portal_accounts.id`, unique, not null).
- `notify_booking_updates_email` (boolean, default true).
- `notify_booking_updates_sms` (boolean, default true).
- `notify_marketing_email` (boolean, default false).
- `notify_marketing_sms` (boolean, default false).
- `notify_technician_enroute` (boolean, default true).
- `quiet_hours_start` (time, nullable).
- `quiet_hours_end` (time, nullable).
- `timezone` (text, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

#### 3.2.5 `portal_threads`

Conversation threads used for secure messaging between portal users and the office/technicians.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `customer_id` (UUID, FK -> `customers.id`, not null).
- `location_id` (UUID, FK -> `customer_locations.id`, nullable).
- `related_work_order_id` (UUID, FK -> `work_orders.id`, nullable).
- `subject` (text, not null).
- `status` (enum: `open`, `pending_customer`, `pending_internal`, `closed`).
- `created_by_portal_account_id` (UUID, FK -> `portal_accounts.id`, nullable).
- `created_by_user_id` (UUID, FK -> `auth.users.id`, nullable) – for threads initiated by staff.
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Indexes:
- `idx_portal_threads_org_id_customer_id`.
- `idx_portal_threads_org_id_related_work_order_id`.

#### 3.2.6 `portal_messages`

Individual messages within a portal thread. To avoid duplication, each message may also map to a `crm_interactions` record.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `thread_id` (UUID, FK -> `portal_threads.id`, not null, on delete cascade).
- `crm_interaction_id` (UUID, FK -> `crm_interactions.id`, nullable).
- `sender_type` (enum: `portal_customer`, `internal_user`, `system`).
- `sender_portal_account_id` (UUID, FK -> `portal_accounts.id`, nullable).
- `sender_user_id` (UUID, FK -> `auth.users.id`, nullable).
- `body` (text, not null).
- `attachments` (jsonb, nullable) – references to Storage.
- `sentiment` (enum: `positive`, `neutral`, `negative`, nullable) – from AI analysis.
- `sent_at` (timestamptz, default now()).
- `read_at` (timestamptz, nullable).

Indexes:
- `idx_portal_messages_thread_id_sent_at`.

#### 3.2.7 `kb_articles`

Knowledge base articles powering FAQs and AI retrieval.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, nullable) – null for global/shared content.
- `slug` (text, not null).
- `title` (text, not null).
- `body_markdown` (text, not null).
- `category` (text, nullable) – e.g., `hvac`, `billing`, `portal_usage`.
- `tags` (text[], nullable).
- `is_published` (boolean, default false).
- `created_by_user_id` (UUID, FK -> `auth.users.id`, nullable).
- `updated_by_user_id` (UUID, FK -> `auth.users.id`, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Constraints:
- Unique: (`org_id`, `slug`).

#### 3.2.8 `kb_article_chunks`

Vectorized chunks of KB content for semantic search (requires pgvector or similar).

- `id` (UUID, PK).
- `article_id` (UUID, FK -> `kb_articles.id`, not null, on delete cascade).
- `chunk_index` (integer, not null).
- `content` (text, not null).
- `embedding` (vector, not null) – dimension depends on chosen model (e.g., 1536).
- `created_at` (timestamptz, default now()).

Indexes:
- Vector index on `embedding` for similarity search.

#### 3.2.9 `portal_ai_conversations`

Represents a self-service AI chat session in the portal.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `portal_account_id` (UUID, FK -> `portal_accounts.id`, nullable) – nullable for anonymous pre-login sessions.
- `customer_id` (UUID, FK -> `customers.id`, nullable).
- `context_type` (enum: `general_support`, `pre_booking`, `active_job`, `billing_question`).
- `related_work_order_id` (UUID, FK -> `work_orders.id`, nullable).
- `status` (enum: `active`, `ended`, `escalated`).
- `escalated_to_thread_id` (UUID, FK -> `portal_threads.id`, nullable) – link to human conversation if escalated.
- `started_at` (timestamptz, default now()).
- `ended_at` (timestamptz, nullable).

Indexes:
- `idx_portal_ai_conversations_org_id_portal_account_id`.

#### 3.2.10 `portal_ai_messages`

Messages within an AI conversation (both user and assistant).

- `id` (UUID, PK).
- `conversation_id` (UUID, FK -> `portal_ai_conversations.id`, not null, on delete cascade).
- `role` (enum: `user`, `assistant`, `system`).
- `content` (text, not null).
- `tokens_used` (integer, nullable).
- `metadata` (jsonb, nullable) – model name, retrieval hits, etc.
- `created_at` (timestamptz, default now()).

Indexes:
- `idx_portal_ai_messages_conversation_id_created_at`.

### 3.3 Relationships to Other Modules

- **CRM**:
  - `portal_accounts.customer_id` references `customers`.
  - `portal_messages.crm_interaction_id` allows portal messages to be tracked in CRM history.
- **Scheduling & Dispatch / Work Orders**:
  - `portal_booking_requests.related_work_order_id` points to the ultimately scheduled job.
  - `portal_threads.related_work_order_id` ties conversations to a specific job.
  - Real-time status updates consume job status events produced by this module.
- **Quoting & Invoicing**:
  - Portal will surface quotes and invoices associated with a customer; exact FKs (e.g., `quote_id`, `invoice_id`) will be modeled in those modules but referenced in portal UI.
- **Technician Mobile App**:
  - Messaging threads shared between technicians and customers through `portal_threads` / `portal_messages`.
- **Marketing / Notifications**:
  - `portal_notification_preferences` feeds into marketing and transactional message orchestration.

---

## 4. API & Edge Function Design

This section defines logical APIs and flows. Actual implementation may use:
- Supabase-generated REST endpoints and RPC functions.
- Supabase Edge Functions (HTTP endpoints) for higher-level orchestration and integrations.

### 4.1 Authentication & Authorization

- Use Supabase Auth for customer portal users (`auth.users` records with role/claims indicating `portal_customer`).
- A `profiles` or similar table (defined centrally) should mark:
  - `org_id`.
  - `role` (e.g., `portal_customer`, `internal_user`).
- RLS ensures:
  - Portal users read/write data only for their `org_id`.
  - Portal users see only records related to their own `customer_id` and permitted locations.

### 4.2 Portal Account Lifecycle

#### 4.2.1 Account Creation / Self-Signup

- **Endpoint**: Edge Function `POST /portal/signup`
- **Inputs**:
  - Email, password (or OAuth token).
  - Basic profile: name, phone.
  - Optional: invite code, existing job or quote reference to auto-link customer.
- **Behavior**:
  - Create Supabase Auth user.
  - Create/lookup `customers` and `customer_contacts` records based on email/phone.
  - Create `portal_accounts` row linked to `customer_id`.
  - Initialize `portal_notification_preferences`.
  - Send verification email (via Supabase or custom function).
- **Outputs**:
  - Auth token (on success).
  - Minimal profile and portal account info.

#### 4.2.2 Account Linking (Existing Customer Invited by Office)

- **Endpoint**: `POST /portal/accept-invite`
- **Inputs**:
  - Invite token.
  - Password or OAuth link.
- **Behavior**:
  - Validate invite token maps to an existing `customer_id`.
  - Create Supabase Auth user and `portal_accounts` if not already present.
  - Mark `portal_accounts.status` = `active`.

#### 4.2.3 Authentication & Session Management

- Leverage Supabase client SDK for:
  - Login: `POST /auth/v1/token?grant_type=password` or OAuth provider endpoints.
  - Logout and token refresh.
- Record key events in `portal_audit_events` via triggers or Edge Functions.

### 4.3 Booking & Appointment Management APIs

#### 4.3.1 Create Booking Request

- **Endpoint**: Edge Function `POST /portal/booking-requests`
- **Inputs**:
  - `location_id` (or full address to create a new location).
  - `service_type`, `problem_description`.
  - Preferred date/time window.
  - Attachments (optionally uploaded first to Storage).
- **Behavior**:
  - Validate customer ownership of location.
  - Insert into `portal_booking_requests`.
  - Optionally trigger internal notification (email/SMS/Slack) to CSR.
  - Optionally auto-generate a tentative job in Scheduling module based on availability rules.
- **Outputs**:
  - Booking request object with status `submitted` or `under_review`.

#### 4.3.2 View Upcoming & Past Appointments

- **Endpoint**: `GET /portal/appointments`
- **Filters**:
  - Date range (default: upcoming + recent past).
  - Status (scheduled, completed, canceled).
- **Behavior**:
  - Query jobs/work orders for the customer and locations they can access.
  - Include status, technician (name + avatar), scheduled windows, and job type.

#### 4.3.3 Reschedule / Cancel Appointment

- **Endpoint**: `POST /portal/appointments/:id/reschedule`
  - Inputs: new preferred date/time, optional reason.
  - Behavior: Validate business rules (e.g., cannot reschedule within X hours of appointment). Call Scheduling module to update job.
- **Endpoint**: `POST /portal/appointments/:id/cancel`
  - Behavior: Apply cancellation policy, update job status, and log in CRM/portal audit.

### 4.4 Service History & Documentation APIs

#### 4.4.1 List Service History

- **Endpoint**: `GET /portal/service-history`
- **Behavior**:
  - Return paginated list of past jobs linked to the customer.
  - Include summary fields (date, location, service type, technician).

#### 4.4.2 Job Detail & Documents

- **Endpoint**: `GET /portal/jobs/:id`
- **Behavior**:
  - Return job details, technician notes (filtered for customer-safe content), parts used, etc.
  - Include URLs to:
    - Invoices and quotes (from invoices/quotes storage).
    - Attachments (photos/videos).
    - Manuals/warranties from Storage or KB.

### 4.5 Real-Time Job Status APIs

#### 4.5.1 Subscribe to Job Status Updates

- Use Supabase real-time channels or row-level subscriptions:
  - Channel: `realtime:work_orders` filtered by customer’s `org_id` and `customer_id`.
- Portal client subscribes to:
  - Status changes (e.g., `scheduled` → `en_route` → `on_site` → `completed`).
  - ETA updates.

#### 4.5.2 Fallback Polling Endpoint

- **Endpoint**: `GET /portal/appointments/:id/status`
- Returns current status, ETA, and last updated timestamp.

### 4.6 Secure Messaging APIs

#### 4.6.1 Create or Open Thread

- **Endpoint**: `POST /portal/threads`
- **Inputs**:
  - `subject`, `initial_message`, optional `related_work_order_id`.
- **Behavior**:
  - Create `portal_threads` row.
  - Create initial `portal_messages` row.
  - Optionally create a `crm_interactions` row of type `portal_message`.

#### 4.6.2 Send Message

- **Endpoint**: `POST /portal/threads/:thread_id/messages`
- **Inputs**:
  - `body`, optional attachments.
- **Behavior**:
  - Insert `portal_messages` row.
  - Optionally insert/append `crm_interactions` record.
  - Trigger internal notification (e.g., to the assigned CSR or technician).

#### 4.6.3 List Threads & Messages

- **Endpoints**:
  - `GET /portal/threads` – list threads for current customer.
  - `GET /portal/threads/:id/messages` – paginated list of messages.

### 4.7 AI Self-Service APIs

#### 4.7.1 Start AI Conversation

- **Endpoint**: `POST /portal/ai/conversations`
- **Inputs**:
  - `context_type`.
  - Optional `related_work_order_id`.
- **Behavior**:
  - Create `portal_ai_conversations` row.
  - Optionally seed system messages (policies, disclaimers).

#### 4.7.2 Send Message to AI

- **Endpoint**: Edge Function `POST /portal/ai/conversations/:id/messages`
- **Inputs**:
  - User message text.
- **Behavior**:
  - Store user message in `portal_ai_messages`.
  - Build retrieval query:
    - Embed user message.
    - Perform vector search on `kb_article_chunks`.
  - Call external LLM with retrieved context and relevant customer/job context (as allowed by privacy rules).
  - Store assistant message and usage metadata.
  - Return AI response to client.
  - Optionally offer escalation to human (creating a `portal_thread`).

#### 4.7.3 Escalate AI Conversation to Human

- **Endpoint**: `POST /portal/ai/conversations/:id/escalate`
- **Behavior**:
  - Create a `portal_threads` row summarizing the AI conversation.
  - Link `portal_ai_conversations.escalated_to_thread_id`.
  - Notify CSR queue.

### 4.8 Settings & Preferences APIs

- **Endpoints**:
  - `GET /portal/settings` – retrieve profile, locations, notification preferences (read-only aggregated view).
  - `PATCH /portal/settings/profile` – update name, phone, etc. (with optional verification).
  - `PATCH /portal/settings/notifications` – update `portal_notification_preferences`.
  - `PATCH /portal/settings/security` – change password, enable/disable MFA (via Supabase Auth).

---

## 5. UI/UX Views & Components

The portal will be implemented as a Next.js app with responsive design. Key views/components, which should be reflected in the routing and component structure:

- **Authentication**
  - Login page (`/login`).
  - Signup/invite acceptance page (`/signup`, `/invite/:token`).
  - Password reset / forgot password pages.

- **Dashboard (`/`)**
  - Summary cards: next appointment, open threads, recent AI conversations, outstanding invoices (when available).
  - Quick actions: “Book Service”, “Message Us”, “Ask the Assistant”.

- **Booking**
  - New booking flow (`/book`):
    - Step 1: Select service type.
    - Step 2: Select or add property/location.
    - Step 3: Describe problem and upload photos.
    - Step 4: Pick time window (based on availability).
    - Confirmation screen with booking ID and status.

- **Appointments**
  - Upcoming appointments list (`/appointments`).
  - Appointment detail page (`/appointments/:id`) with:
    - Status timeline, technician info, ETA.
    - Embedded messaging thread related to the job.

- **Service History**
  - History list (`/history`).
  - Job detail (`/history/:id`) mirroring job detail but with greater emphasis on past notes, documents, and invoices.

- **Messaging**
  - Thread list (`/messages`).
  - Thread detail (`/messages/:id`), real-time updates, attachments, and seen indicators.

- **AI Assistant**
  - AI chat page (`/assistant`).
  - Embedded AI widget in booking flow and dashboard (for contextual help).

- **Settings**
  - Profile (`/settings/profile`).
  - Notifications (`/settings/notifications`).
  - Security (`/settings/security`).

Each view should be driven by typed client-side API hooks (e.g., using React Query or SWR) hitting the APIs defined above, with strict type alignment to Supabase schemas.

---

## 6. Background Jobs & Automation

- **Booking Request Processing**
  - Periodic job or event-based trigger to:
    - Move `portal_booking_requests` from `submitted` to `under_review` to `approved/scheduled`.
    - Ensure `related_work_order_id` is populated when scheduling completes.

- **Notifications**
  - Edge Functions triggered on:
    - New booking requests.
    - Appointment status changes.
    - New portal messages from staff or customers.
  - Respect `portal_notification_preferences` and CRM-level `do_not_contact` flags.

- **AI KB Maintenance**
  - Batch jobs for embedding new or updated `kb_articles` into `kb_article_chunks`.
  - Optional scheduled re-embedding if model/version changes.

---

## 7. Security, Privacy, and Compliance

- **Authentication**
  - All portal APIs require authenticated Supabase JWT (except basic public endpoints like viewing marketing KB content).
  - MFA support where configured in Supabase Auth.

- **Authorization**
  - RLS enforces `org_id` and `customer_id` boundaries.
  - Edge Functions re-validate authorization based on `portal_accounts`.

- **Data Protection**
  - Use HTTPS everywhere (enforced by Vercel and Supabase).
  - No sensitive fields (card numbers, full SSNs) stored in plain text.
  - For payment data, rely on PCI-compliant third-party (e.g., Stripe) with tokenization.

- **Privacy**
  - Allow customers to control marketing opt-ins.
  - Log access to sensitive documents (invoices, detailed job notes) via `portal_audit_events`.

---

## 8. Non-Functional Requirements

- **Performance**
  - Typical portal interactions (dashboard load, list views) should complete within 300–500 ms backend response times under normal load.
  - Real-time updates should propagate within a few seconds of status changes.

- **Scalability**
  - Design for hundreds of concurrent portal users per tenant and thousands overall.
  - Use pagination and lazy loading for lists (history, threads, AI conversations).

- **Reliability**
  - Ensure idempotent booking and messaging endpoints (safe retries).
  - Use logical soft-deletion or status flags for critical entities where appropriate.

- **Observability**
  - Log errors and key events from Edge Functions.
  - Maintain metrics on AI usage, message volume, booking conversion, etc.

---

## 9. Open Questions / Future Enhancements

These items can be turned into backlog epics or subtasks by an LLM:

- Decide on the exact canonical job/work order schema and how portal-specific fields map to it.
- Define a shared `profiles` and `orgs` schema across modules, if not already standardized.
- Determine which job notes are customer-visible vs. internal-only and how to enforce that separation.
- Decide on the initial AI model/provider(s) and embedding configuration (dimensions, cost limits, latency SLOs).
- Add support for push notifications via mobile apps or browser push.
- Implement support for multiple organizations per portal account (e.g., property manager working with multiple service providers), if needed.


