# Technical Design Document – Section 1: Customer Relationship Management (CRM)

## 1. Purpose & Scope

This Technical Design Document (TDD) defines the **Customer Relationship Management (CRM)** capabilities for the HVAC, Plumbing, and Electrical services platform described in `functional.md`. It provides enough technical detail for engineers (and LLM-based task generators) to decompose the CRM module into implementation tasks.

- **In-scope capabilities (from functional.md, §1 CRM)**:
  - Centralized customer database.
  - Contact history, preferences, and communication logs.
  - Automated follow-ups and reminders.
  - AI-driven customer segmentation and targeting.
- **Out-of-scope (referenced but not designed here)**:
  - Scheduling & Dispatch (functional.md §2).
  - Work Order Management (§3).
  - Quoting & Invoicing (§4).
  - Other modules (Inventory, Technician App, Customer Portal, etc.).

The CRM module must integrate cleanly with all other modules while being **deployable on Supabase (PostgreSQL + Auth + Storage + Edge Functions)** and **fronted by a web application hosted on Vercel**, as defined in `tooling.md`.

## 2. Target Platform & High-Level Architecture

### 2.1 Platform Alignment (from `tooling.md`)

- **Backend**: Supabase
  - PostgreSQL database (managed).
  - Supabase Auth for user authentication and basic user profiles.
  - Row-Level Security (RLS) for multi-tenant and role-based access.
  - Supabase Storage for attachments (e.g., documents tied to customers).
  - Supabase Edge Functions (serverless) for:
    - Automated follow-up scheduling and sending.
    - Webhook handling (e.g., from email/SMS providers).
    - AI/ML orchestration (calling external AI APIs for segmentation, scoring).
- **Frontend**: Vercel
  - Next.js web application.
  - Supabase JS client (for DB, Auth, Storage, real-time).
  - Deployed via Vercel with CI/CD.
- **Mobile** (future):
  - Flutter or React Native apps using Supabase SDKs.
  - Leverage the same Supabase backend (no separate CRM backend).

### 2.2 Logical Components

1. **CRM Data Layer (PostgreSQL on Supabase)**:
   - Core entities: `customers`, `customer_locations`, `customer_contacts`, `crm_interactions`, `crm_followups`, `crm_segments`, `crm_segment_members`, `crm_preferences`, `crm_tags`, `crm_customer_tags`, `crm_message_templates`, `crm_automation_rules`, `crm_automation_runs`.
2. **CRM Service Layer (Supabase Edge Functions + SQL)**:
   - Encapsulates complex operations:
     - Bulk operations (e.g., add many customers to a segment).
     - AI-driven segmentation jobs.
     - Follow-up generation and dispatch.
     - Integration with external communication channels.
3. **CRM API Layer**:
   - Two access patterns:
     - **Direct**: Frontend uses Supabase client to call Postgres RPC functions and queries.
     - **Indirect**: Frontend calls Edge Functions over HTTPS for higher-level actions (e.g., "Generate AI segment").
4. **UI Layer (Next.js on Vercel)**:
   - CSR/SSR pages and components for customer management, interactions, follow-ups, and segmentation.
5. **Integrations**:
   - Messaging providers (SMTP/email, SMS, possibly push via mobile app).
   - AI provider(s) (LLM and/or vector DB) called from Edge Functions.

### 2.3 Multi-Tenancy & Access Control

Assuming one Supabase project serves multiple companies (tenants) OR at least supports clean role-based access within a single company:

- **Tenancy key**: `org_id` on all CRM tables (or `account_id` if broader account concept is introduced elsewhere).
- **RLS policies**:
  - Users can only access rows where `org_id` matches the `org_id` associated with their Supabase user profile.
  - Role-based policies (admin, manager, dispatcher, technician, sales/CSR) restricting what actions they can perform on CRM data (e.g., techs see limited customer info).

## 3. Domain Model & Data Design

### 3.1 Entities Overview

Core concepts for CRM:

- **Customer**: Person or organization receiving services.
- **Location**: Physical service locations associated with a customer (billing vs service addresses).
- **Contact Info**: One or more phone numbers/emails/messaging endpoints.
- **Interaction/Activity**: Logged calls, emails, SMS, portal messages, notes; optionally linked to jobs/quotes/invoices.
- **Follow-Up**: Future-dated reminder/action to contact a customer.
- **Preferences**: Customer communication preferences (channels, do-not-contact flags, time windows).
- **Tags**: Free-form or pre-defined labels on customers (e.g., VIP, warranty, commercial).
- **Segment**: Saved group definition (rule-based or static) for targeting (e.g., "customers with HVAC units older than 10 years").
- **AI Artifacts**: Segment generation jobs, scores, and explanations.

### 3.2 Tables & Columns

The following schema is conceptual; exact types and constraints should be applied as Supabase Postgres DDL.

#### 3.2.1 `orgs` (if not defined elsewhere)

Used only for tenancy scoping; if defined in another module, reuse instead.

- `id` (UUID, PK).
- `name` (text, not null).
- `created_at` (timestamptz, default now()).

#### 3.2.2 `customers`

Represents a logical customer entity (individual or organization).

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `external_ref` (text, nullable) – reference to legacy system if migrated.
- `type` (enum: `individual`, `company`).
- `name` (text, not null) – full name or company name.
- `first_name` (text, nullable) – for individuals.
- `last_name` (text, nullable).
- `company_name` (text, nullable) – for B2B contacts.
- `primary_location_id` (UUID, FK -> `customer_locations.id`, nullable).
- `primary_contact_id` (UUID, FK -> `customer_contacts.id`, nullable).
- `email` (text, nullable) – convenience, can mirror a primary contact.
- `phone` (text, nullable) – convenience, can mirror a primary contact.
- `status` (enum: `active`, `prospect`, `inactive`, `blacklisted`).
- `lifecycle_stage` (enum: `lead`, `opportunity`, `customer`, `former_customer`).
- `source` (text, nullable) – e.g., `web`, `phone`, `referral`, `ad_campaign`.
- `preferred_language` (text, nullable, e.g., `en`, `es`).
- `notes` (text, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Indexes:
- `idx_customers_org_id` on (`org_id`).
- Composite indexes for search (e.g., `name`, `email`, `phone`).

#### 3.2.3 `customer_locations`

Stores billing and service locations for customers.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `customer_id` (UUID, FK -> `customers.id`, not null, on delete cascade).
- `label` (text, nullable) – e.g., "Home", "Office", "Plant 1".
- `type` (enum: `billing`, `service`, `both`).
- `address_line1` (text, not null).
- `address_line2` (text, nullable).
- `city` (text, not null).
- `state` (text, nullable).
- `postal_code` (text, nullable).
- `country` (text, nullable, default `US`).
- `latitude` (numeric, nullable).
- `longitude` (numeric, nullable).
- `is_primary` (boolean, default false).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Indexes:
- `idx_customer_locations_customer_id`.
- Geo index on (`latitude`, `longitude`) if needed for routing integration.

#### 3.2.4 `customer_contacts`

Multiple communication channels per customer.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `customer_id` (UUID, FK -> `customers.id`, not null, on delete cascade).
- `type` (enum: `email`, `mobile`, `phone`, `fax`, `whatsapp`, `telegram`, `portal`).
- `value` (text, not null) – e.g., email address or phone number.
- `is_primary` (boolean, default false).
- `is_verified` (boolean, default false).
- `opt_in_marketing` (boolean, default true).
- `opt_in_transactional` (boolean, default true).
- `preferred_channel` (boolean, default false) – marks favorite channel for contact.
- `notes` (text, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Indexes:
- `idx_customer_contacts_customer_id`.
- Optional unique partial indexes (e.g., per org per email).

#### 3.2.5 `crm_preferences`

High-level preferences and privacy for a customer.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `customer_id` (UUID, FK -> `customers.id`, unique, not null, on delete cascade).
- `do_not_contact` (boolean, default false).
- `do_not_email` (boolean, default false).
- `do_not_sms` (boolean, default false).
- `do_not_call` (boolean, default false).
- `preferred_contact_window_start` (time, nullable).
- `preferred_contact_window_end` (time, nullable).
- `notes` (text, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

#### 3.2.6 `crm_interactions`

Tracks communication and contact history.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `customer_id` (UUID, FK -> `customers.id`, not null).
- `location_id` (UUID, FK -> `customer_locations.id`, nullable).
- `related_work_order_id` (UUID, nullable, FK to work order table when defined).
- `related_quote_id` (UUID, nullable).
- `channel` (enum: `phone_inbound`, `phone_outbound`, `email_inbound`, `email_outbound`, `sms_inbound`, `sms_outbound`, `portal_message`, `note`, `in_person`).
- `direction` (enum: `inbound`, `outbound`, `system_generated`, nullable).
- `subject` (text, nullable).
- `summary` (text, nullable).
- `body` (text, nullable) – full email or message body where applicable.
- `metadata` (jsonb, nullable) – provider-specific IDs, message IDs, call duration, etc.
- `sentiment` (enum: `positive`, `neutral`, `negative`, nullable) – from AI sentiment analysis.
- `created_by_user_id` (UUID, FK -> auth.users, nullable for system-generated).
- `occurred_at` (timestamptz, not null).
- `created_at` (timestamptz, default now()).

Indexes:
- `idx_crm_interactions_customer_id`.
- `idx_crm_interactions_org_id_occurred_at`.

#### 3.2.7 `crm_followups`

Represents scheduled follow-ups/reminders.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `customer_id` (UUID, FK -> `customers.id`, not null).
- `assigned_to_user_id` (UUID, FK -> auth.users, nullable).
- `title` (text, not null).
- `description` (text, nullable).
- `due_at` (timestamptz, not null).
- `status` (enum: `pending`, `completed`, `canceled`, `expired`).
- `priority` (enum: `low`, `medium`, `high`).
- `origin` (enum: `manual`, `system_rule`, `ai_recommendation`).
- `related_interaction_id` (UUID, FK -> `crm_interactions.id`, nullable).
- `related_work_order_id` (UUID, nullable).
- `created_by_user_id` (UUID, FK -> auth.users, nullable for system-generated).
- `completed_at` (timestamptz, nullable).
- `completion_notes` (text, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Indexes:
- `idx_crm_followups_org_id_due_at`.
- `idx_crm_followups_assigned_to_user_id_due_at`.

#### 3.2.8 `crm_tags` & `crm_customer_tags`

Generic labeling of customers.

`crm_tags`:
- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `name` (text, not null).
- `description` (text, nullable).
- `color` (text, nullable, e.g., hex).
- `created_at` (timestamptz, default now()).

Unique constraint: (`org_id`, `name`).

`crm_customer_tags`:
- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `customer_id` (UUID, FK -> `customers.id`, not null, on delete cascade).
- `tag_id` (UUID, FK -> `crm_tags.id`, not null, on delete cascade).
- `assigned_by_user_id` (UUID, FK -> auth.users, nullable).
- `created_at` (timestamptz, default now()).

Unique constraint: (`org_id`, `customer_id`, `tag_id`).

#### 3.2.9 `crm_segments`

Definition of segments.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `name` (text, not null).
- `description` (text, nullable).
- `type` (enum: `static`, `rule_based`, `ai_generated`).
- `definition` (jsonb, nullable) – rule definitions for `rule_based`.
- `ai_prompt` (text, nullable) – prompt used for AI-generated segments.
- `ai_explanation` (text, nullable) – human-readable explanation returned by AI.
- `is_active` (boolean, default true).
- `last_computed_at` (timestamptz, nullable).
- `created_by_user_id` (UUID, FK -> auth.users, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

#### 3.2.10 `crm_segment_members`

Members of a segment.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `segment_id` (UUID, FK -> `crm_segments.id`, not null, on delete cascade).
- `customer_id` (UUID, FK -> `customers.id`, not null, on delete cascade).
- `score` (numeric, nullable) – optional ranking from AI (e.g., propensity to buy).
- `metadata` (jsonb, nullable).
- `created_at` (timestamptz, default now()).

Unique constraint: (`org_id`, `segment_id`, `customer_id`).

#### 3.2.11 `crm_message_templates`

Templates for follow-ups and campaigns (CRM side; Marketing module may extend).

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `name` (text, not null).
- `channel` (enum: `email`, `sms`, `phone_script`, `portal_message`).
- `subject` (text, nullable) – email subject, etc.
- `body` (text, not null) – templated body with variables, e.g., `{{customer.first_name}}`.
- `variables` (jsonb, nullable) – list of supported variables.
- `is_system` (boolean, default false) – locked templates.
- `created_by_user_id` (UUID, FK -> auth.users, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

#### 3.2.12 `crm_automation_rules`

Defines automation logic for follow-ups and CRM actions.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `name` (text, not null).
- `description` (text, nullable).
- `is_enabled` (boolean, default true).
- `trigger_type` (enum: `event`, `time_based`, `segment_membership`).
- `event_type` (text, nullable) – e.g., `work_order_completed`, `quote_sent`.
- `time_offset_minutes` (integer, nullable) – for time-based follow-up after an event.
- `segment_id` (UUID, FK -> `crm_segments.id`, nullable).
- `conditions` (jsonb, nullable) – additional filter conditions.
- `actions` (jsonb, not null) – structured representation of actions (e.g., create follow-up, send email).
- `created_by_user_id` (UUID, FK -> auth.users, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

#### 3.2.13 `crm_automation_runs`

Tracks executions of automation rules.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `rule_id` (UUID, FK -> `crm_automation_rules.id`, not null).
- `customer_id` (UUID, FK -> `customers.id`, nullable).
- `trigger_context` (jsonb, nullable) – payload that caused the run.
- `status` (enum: `pending`, `success`, `failed`, `skipped`).
- `error_message` (text, nullable).
- `started_at` (timestamptz, default now()).
- `completed_at` (timestamptz, nullable).

Indexes:
- `idx_crm_automation_runs_rule_id_started_at`.

### 3.3 Relationships to Other Modules

- **Work Orders**:
  - `crm_interactions.related_work_order_id` references work orders.
  - `crm_followups.related_work_order_id` references upcoming or past jobs.
- **Quoting & Invoicing**:
  - Future FK columns (e.g., `related_quote_id`, `related_invoice_id`) to link CRM activities.
- **Scheduling & Dispatch**:
  - `customer_locations` will be consumed by dispatch for routing.

## 4. API & Edge Function Design

This section defines **logical APIs**. Actual implementation may use:
- Supabase Postgres functions (RPC).
- Supabase Edge Functions (HTTP endpoints).

### 4.1 Authentication & Authorization

- All APIs require:
  - Authenticated Supabase user (JWT).
  - `org_id` resolved from user profile table (`profiles`) or JWT claims.
- RLS ensures users can only access data for their `org_id`.
- Additional role checks:
  - CSR/Sales/Admin can fully manage CRM data.
  - Technicians may have read-only access to limited customer info.

### 4.2 Customer Management APIs

#### 4.2.1 Create Customer

- **Endpoint**: Edge Function `POST /crm/customers`
- **Input**:
  - `type`, `name`, optional `first_name`, `last_name`, `company_name`.
  - Primary contact info (email, phone).
  - Primary location data.
  - Initial tags.
- **Behavior**:
  - Insert into `customers`.
  - Create `customer_locations` and `customer_contacts` as needed.
  - Set `primary_location_id` and `primary_contact_id`.
  - Apply default `crm_preferences`.
- **Output**:
  - Created customer with nested contacts/locations.

Alternatively, some clients may directly call Supabase with:
- RPC `crm_create_customer(...)` implemented in Postgres.

#### 4.2.2 Update Customer

- **Endpoint**: `PATCH /crm/customers/:id`
- **Input**: Partial updates for customer fields, plus nested updates (optional) for contacts/locations.
- **Behavior**:
  - Validate `org_id` ownership via RLS.
  - Update rows; optionally log a `crm_interactions` record of type `note` for significant changes.

#### 4.2.3 Get Customer Details

- **Endpoint**: `GET /crm/customers/:id`
- **Data**:
  - Customer core fields.
  - Locations.
  - Contacts.
  - Preferences.
  - Recent interactions (paged).
  - Upcoming follow-ups.

Implementation:
- Either a single Edge Function that composes multiple queries OR a Postgres view + joins consumed by frontend.

#### 4.2.4 Search/List Customers

- **Endpoint**: `GET /crm/customers`
- **Query params**:
  - `q` (search term for name/email/phone/address).
  - Filters: `status`, `lifecycle_stage`, `tag`, `segment_id`, pagination.
- **Behavior**:
  - Use indexed columns.
  - Apply RLS automatically.

### 4.3 Interaction & Communication Logs APIs

#### 4.3.1 Log Interaction

- **Endpoint**: `POST /crm/interactions`
- **Input**:
  - `customer_id`, `channel`, `direction`, `subject`, `summary`, `body`, `metadata`, `occurred_at`.
  - Optional `related_work_order_id`, `location_id`.
- **Behavior**:
  - Insert into `crm_interactions`.
  - Optionally enqueue AI sentiment analysis & classification via async job (Edge Function).

#### 4.3.2 Fetch Interaction History

- **Endpoint**: `GET /crm/customers/:id/interactions`
- **Query params**: pagination, `channel`, date range, `sentiment`.
- **Behavior**:
  - Query `crm_interactions` filtered by `customer_id` and `org_id`.

### 4.4 Follow-Ups & Reminders APIs

#### 4.4.1 Create Manual Follow-Up

- **Endpoint**: `POST /crm/followups`
- **Input**:
  - `customer_id`, `title`, `description`, `due_at`, `priority`, optional `assigned_to_user_id`, `origin = manual`.
- **Behavior**:
  - Insert into `crm_followups`.
  - Optionally schedule push/email reminders for the assignee.

#### 4.4.2 List Upcoming Follow-Ups

- **Endpoint**: `GET /crm/followups`
- **Query params**: `assigned_to_user_id`, date range, `status`, pagination.

#### 4.4.3 Complete/Cancel Follow-Up

- **Endpoint**: `PATCH /crm/followups/:id`
- **Input**: status change, optional completion notes.

### 4.5 Segmentation & Targeting APIs

#### 4.5.1 Create/Update Segment

- **Endpoint**: `POST /crm/segments` / `PATCH /crm/segments/:id`
- **Input**:
  - `name`, `description`, `type`.
  - For `rule_based`: `definition` (jsonb rule structure).
  - For `ai_generated`: `ai_prompt`.
- **Behavior**:
  - Persist definition.
  - For `rule_based`, compute members via SQL.
  - For `ai_generated`, call AI Edge Function to:
    - Interpret `ai_prompt`.
    - Generate query or scoring criteria.
    - Populate `crm_segments` and `crm_segment_members`.

#### 4.5.2 Get Segment Members

- **Endpoint**: `GET /crm/segments/:id/members`
- **Behavior**:
  - Query `crm_segment_members` joined with `customers`.

#### 4.5.3 Trigger AI Segmentation (On-Demand)

- **Endpoint**: `POST /crm/segments/:id/recompute`
- **Behavior**:
  - Edge Function fetches segment definition.
  - For `ai_generated`, (re)calls AI provider to re-score all or filtered customers.
  - Updates `crm_segment_members` and `last_computed_at`.

### 4.6 Automation & Follow-Up Rules APIs

#### 4.6.1 Manage Automation Rules

- **Endpoints**:
  - `POST /crm/automation/rules`
  - `PATCH /crm/automation/rules/:id`
  - `GET /crm/automation/rules`
- **Behavior**:
  - CRUD of `crm_automation_rules` with validation:
    - If `trigger_type = event`, `event_type` is required.
    - If `trigger_type = time_based`, `time_offset_minutes` is required.
    - If `trigger_type = segment_membership`, `segment_id` is required.

#### 4.6.2 Automation Execution (Internal)

- **Edge Functions / Schedulers**:
  - `handle_event_trigger` – triggered by events from other modules (e.g., work order completed).
  - `process_time_based_automations` – scheduled cron (e.g., every 5 minutes) to evaluate `time_based` rules.
  - `process_segment_automations` – scheduled cron to act on `segment_membership` triggers.
- **Actions** (stored in `crm_automation_rules.actions`):
  - `create_followup` – inserts into `crm_followups`.
  - `send_message` – initiates email/SMS via integration.
  - `tag_customer` – adds `crm_customer_tags`.

## 5. AI & Analytics Design

### 5.1 AI-Driven Customer Segmentation

- Implemented via Edge Functions invoking an external AI API.
- **Inputs** to AI:
  - Schema description (customers, locations, interactions, tags).
  - Prompt and constraints from `crm_segments.ai_prompt`.
  - Aggregated customer metrics (e.g., number of jobs, average spend, days since last service).
- **Outputs**:
  - Segment definition (machine-readable rules or per-customer scores).
  - Human-readable explanation (`ai_explanation`).
- **Storage**:
  - Rules/scores stored in `crm_segments.definition` & `crm_segment_members`.

### 5.2 Sentiment Analysis for Interactions

- Edge Function `analyze_interaction_sentiment`:
  - Triggered on `INSERT` into `crm_interactions` (via database trigger + call to Edge Function, or via async polling).
  - Calls AI API to classify `sentiment`.
  - Updates `crm_interactions.sentiment`.
- Aggregations:
  - Used for analytics and potential triggers (e.g., negative sentiment -> create follow-up).

### 5.3 AI-Based Follow-Up Suggestions

- Edge Function `suggest_followups_for_customer`:
  - Inputs: `customer_id`, history of work orders, quotes, interactions.
  - Outputs: list of recommended follow-up actions and timing.
  - Writes suggestions as `crm_followups` with `origin = ai_recommendation`.

## 6. Frontend (Vercel/Next.js) UI/UX Design

### 6.1 Pages & Views

- **Customers List Page**:
  - Search bar (name/email/phone).
  - Filters (status, lifecycle_stage, tags, segments).
  - Table or card view with key details and badges for tags.
- **Customer Detail Page**:
  - Summary header: name, lifecycle stage, status, tags, preferred contact info.
  - Tabs or sections:
    - **Overview** – primary info, locations, preferences.
    - **Activity** – timeline of `crm_interactions` and `crm_followups`.
    - **Segments** – segments this customer belongs to and scores.
    - **Notes** – internal notes.
- **Follow-Ups Dashboard**:
  - List of upcoming follow-ups, filter by assignee, status, priority.
  - Quick actions to complete/reschedule.
- **Segments Management Page**:
  - List of segments with type (static/rule-based/AI), member counts, status.
  - Detail page per segment with definition editor and member list.
- **Automation Rules Page**:
  - List of automations, enable/disable toggles.
  - Rule detail with trigger, conditions, actions and recent runs.

### 6.2 UI Components

- Reusable components:
  - Customer avatar + badge.
  - Tag selector.
  - Interaction timeline item.
  - Follow-up card.
  - Segment rule builder UI (for non-technical users).

### 6.3 Frontend Data Access Patterns

- Use Supabase JS client for:
  - CRUD on simple tables (`customers`, `customer_locations`, `customer_contacts`, `crm_tags`).
  - Realtime subscriptions (optional) for updates to `crm_followups`.
- Use Edge Functions for:
  - AI-driven operations (segments, suggestions).
  - Bulk operations and automations (to avoid complex logic in frontend).

## 7. Security, Privacy, and Compliance

### 7.1 RLS & Roles

- Define roles in user profile table (e.g., `role` column).
- Implement RLS policies for each table:
  - `org_id = current_setting('app.current_org_id')::uuid`.
  - Additional role checks for write operations (only certain roles can update automation rules, etc.).

### 7.2 Data Protection

- PII (names, addresses, phone, email) stored in Supabase Postgres with encryption at rest (platform-provided).
- Access via HTTPS only.
- Ensure audit logging for:
  - Changes to `customers`, `crm_preferences`.
  - Access to CRM data for debugging and compliance.

### 7.3 Opt-Out & Consent Management

- Enforce `crm_preferences` and `customer_contacts.opt_in_*` flags in:
  - Automation engines (rules must respect do-not-contact flags).
  - Manual message UIs (warn if sending to opted-out contacts).

## 8. Non-Functional Requirements

- **Performance**:
  - Search results should return within a target of \< 500ms for typical dataset sizes (e.g., up to 50k customers per org).
  - Use indexes for frequently filtered/search columns.
- **Scalability**:
  - Supabase Pro tier recommended for production.
  - Long-running AI jobs executed asynchronously, with progress tracking.
- **Reliability**:
  - Follow-ups should be idempotent; avoid duplicate creation from repeated events.
  - Automation runs should log failures and allow retry.
- **Observability**:
  - Edge Functions should log key events and errors.
  - Provide dashboards (later) that aggregate CRM metrics.

## 9. Implementation Notes & Open Questions

- **Open Questions** (to be resolved before implementation or left for future iterations):
  - Final decision on tenancy model: true multi-tenant vs single-tenant per Supabase project.
  - Specific external providers for email/SMS and AI (e.g., which LLM and API).
  - Extent of real-time features in CRM (e.g., live updates vs periodic refresh).
- **Extension Points**:
  - Deep integration with Marketing Automation module for campaigns using `crm_segments`.
  - Customer Portal personalization based on CRM segments and preferences.

This TDD is intended to be consumed by an LLM or engineering team to generate concrete implementation tasks, including DDL creation, Edge Function scaffolding, RLS policy definitions, backend logic, and frontend components/pages for the CRM module.


