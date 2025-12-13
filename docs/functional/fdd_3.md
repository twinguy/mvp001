## Technical Design Document – Section 3: Work Order Management

## 1. Purpose & Scope

This Technical Design Document (TDD) defines the **Work Order Management** capabilities for the HVAC, Plumbing, and Electrical services platform described in `functional.md`. It is intended to be detailed enough for engineers and LLM-based task generators to break the module into concrete implementation tasks.

- **In-scope capabilities (from `functional.md`, §3 Work Order Management)**:
  - Digital work order creation, tracking, and completion.
  - Photo/video and document attachments for job documentation.
  - Customizable checklists and forms for on-site work.
  - AI-powered job categorization and prioritization.
  - Work order lifecycle management (status transitions, audit trail).
  - Technician- and customer-facing visibility into work orders.
- **Explicit design assumption**:
  - **Work Order Management and Scheduling/Dispatch are logically separated**:
    - This module owns the *work order* as a business object (scope, tasks, documentation, status).
    - The **Scheduling & Dispatch** module (TDD §2) owns calendars, time slots, route optimization, and technician assignment logic.
    - Integration is via shared entities (e.g., `work_order_visits`) and events, not embedded scheduling logic.
- **Out-of-scope for this TDD (but integrated)**:
  - Scheduling & Dispatch (calendars, route optimization, technician assignment).
  - Quoting & Invoicing (prices, payments, invoices).
  - Inventory & Asset Management (stock levels, asset lifecycle).
  - Marketing, CRM beyond what is needed to link customers and interactions.

The Work Order Management module must be deployable on **Supabase (PostgreSQL + Auth + Storage + Edge Functions)** and consumed primarily by a **Next.js web app on Vercel**, plus future mobile apps, in alignment with `tooling.md`.

## 2. Target Platform & High-Level Architecture

### 2.1 Platform Alignment (from `tooling.md`)

- **Backend: Supabase**
  - PostgreSQL for relational data (work orders, visits, checklists, etc.).
  - Supabase Auth for users (office staff, dispatchers, technicians).
  - Row-Level Security (RLS) for multi-tenant and role-based access control.
  - Supabase Storage for media/attachments (photos, videos, documents).
  - Supabase Edge Functions for:
    - Work order lifecycle orchestration and validation.
    - AI-powered classification and prioritization.
    - Integration with external services (SMS/email, mapping, AI providers).
    - Webhook/event handling from Scheduling, Inventory, CRM, etc.
- **Frontend: Vercel**
  - Next.js application (office/dispatcher UI and possibly technician web UI).
  - Uses Supabase JS client for auth, database, storage, and real-time updates.
  - Deployed via Vercel with CI/CD and environment separation (dev/stage/prod).
- **Mobile**
  - Future Flutter or React Native technician app using Supabase SDKs.
  - Shares same backend models and APIs (no separate work order backend).

### 2.2 Logical Components

1. **Data Layer (Supabase PostgreSQL)**  
   Core tables for:
   - `work_orders` – main work order record.
   - `work_order_visits` – specific scheduled visits/appointments tied to scheduling module.
   - `work_order_status_history` – audit trail of status transitions.
   - `work_order_tasks` – atomic tasks/steps within a work order (optional if checklists are used as tasks).
   - `checklist_templates`, `checklist_template_items` – reusable templates for job types.
   - `work_order_checklists`, `work_order_checklist_items` – instantiated checklists attached to work orders.
   - `work_order_attachments` – metadata for photos, videos, documents in Supabase Storage.
   - `work_order_notes` – free-form internal notes (or link to CRM interactions).
   - `job_categories`, `job_types` – controlled vocabularies for job classification.
   - `work_order_ai_suggestions` – storage of AI-generated categorizations and priorities.
   - `work_order_tags`, `work_order_tag_links` – labels on work orders (e.g., warranty, callback).

2. **Service Layer (Supabase Edge Functions + SQL/RPC)**  
   - Encapsulates complex workflows and validation:
     - Work order creation from multiple sources (phone, portal, quote, recurring maintenance plan).
     - Enforcement of lifecycle state machine and side-effects (e.g., when status becomes `ready_for_scheduling`).
     - AI classification/prioritization jobs triggered on creation or update.
     - Synchronization with Scheduling & Dispatch for visits.
     - Generation of checklists from templates by job type.

3. **API Layer**
   - **Direct data access** via Supabase client:
     - CRUD operations on relatively simple tables (e.g., `work_order_notes`, `work_order_attachments`).
   - **Higher-level orchestration** via Edge Functions (HTTP):
     - `create_work_order`, `update_work_order`, `change_work_order_status`.
     - `classify_and_prioritize_work_order`.
     - `generate_work_order_checklists`.
     - Event/listener endpoints for Scheduling, Inventory, CRM.

4. **UI Layer (Next.js + future mobile)**
   - Office/dispatcher web UI for creating, editing, tracking work orders.
   - Technician UI (web or mobile) for viewing assigned visits, updating status, filling checklists, capturing media.
   - Customer portal views for seeing job status and history (read-only slices of this module).

5. **Integrations**
   - **CRM**: Link work orders to customers, locations, and CRM interactions.
   - **Scheduling & Dispatch**: Coordinate via `work_order_visits` and events; that module controls time/assignment.
   - **Inventory & Asset Management**: Reserve and consume parts; associate assets/equipment on site.
   - **Quoting & Invoicing**: Create work orders from approved quotes; expose completion data for invoicing.
   - **Notifications**: Email/SMS/push notifications about work order events via communication services.

### 2.3 Multi-Tenancy & Access Control

Assume one Supabase project may serve multiple companies (tenants).

- **Tenancy key**: `org_id` on all work order–related tables (`orgs` defined as in other TDDs).
- **Role-based access**:
  - **Admin/Manager**: full CRUD on all work orders for their `org_id`.
  - **Dispatcher**: create/edit work orders, manage status, coordinate with scheduling.
  - **Technician**: read-only for assigned work orders/visits plus rights to update limited fields (status, checklist responses, attachments, tech notes, used materials).
  - **Customer (portal)**: read-only view of their own work orders (limited fields).
- **RLS policies** enforce:
  - `org_id` scoping for all roles.
  - Additional filters for technicians/customers based on assignment or customer relationship.

## 3. Domain Model & Data Design

### 3.1 Entity Overview

Primary domain concepts:

- **Work Order**: A job to be performed for a customer at one or more locations (e.g., "Install new HVAC unit", "Fix leaking pipe").
- **Visit**: A specific scheduled visit/appointment for a work order (may be multiple visits per work order).
- **Job Category / Job Type**: Standardized labels describing the nature of the work (e.g., HVAC > Installation, Plumbing > Emergency Leak).
- **Checklist Template**: Reusable template of steps and form fields for given job types (e.g., "HVAC Tune-Up Checklist").
- **Work Order Checklist**: Instantiated checklist(s) attached to a work order/visit, derived from templates or created ad hoc.
- **Attachments**: Photos, videos, documents, customer signatures, etc., linked to a work order or visit.
- **Notes**: Internal notes or comments about the work order.
- **Status & History**: Lifecycle state (draft, ready_for_scheduling, scheduled, in_progress, on_hold, completed, canceled) and its transitions with timestamps and actors.
- **AI Suggestions**: Machine-generated categorizations, priorities, recommended checklists, and notes.
- **Tags**: Labels used for searching and reporting (e.g., warranty, callback, high_value_customer).

### 3.2 Tables & Columns (Conceptual Schema)

> Note: types are conceptual; actual implementation uses Supabase/Postgres types and constraints. All tables include `created_at` and `updated_at` timestamps with defaults.

#### 3.2.1 `orgs`

Shared tenant table (if not already defined in another module).

- `id` (UUID, PK).
- `name` (text, not null).
- `created_at` (timestamptz, default now()).

#### 3.2.2 `work_orders`

Core work order entity.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `customer_id` (UUID, FK -> `customers.id`, not null).
- `location_id` (UUID, FK -> `customer_locations.id`, nullable, not null if location-based work is required).
- `external_ref` (text, nullable) – legacy system reference.
- `source` (enum: `phone`, `portal`, `crm`, `quote`, `maintenance_plan`, `other`).
- `title` (text, not null) – short description, e.g., "AC not cooling on 2nd floor".
- `description` (text, nullable) – detailed description/problem statement.
- `job_category_id` (UUID, FK -> `job_categories.id`, nullable).
- `job_type_id` (UUID, FK -> `job_types.id`, nullable).
- `status` (enum: `draft`, `pending_approval`, `ready_for_scheduling`, `scheduled`, `in_progress`, `on_hold`, `completed`, `canceled`).
- `priority` (enum: `low`, `normal`, `high`, `emergency`).
- `requested_start_window_start` (timestamptz, nullable) – customer preferred window start (for scheduling module).
- `requested_start_window_end` (timestamptz, nullable).
- `sla_due_at` (timestamptz, nullable) – computed from SLA rules.
- `origin_quote_id` (UUID, nullable, FK to quotes when defined).
- `origin_maintenance_plan_id` (UUID, nullable, FK to maintenance plan when defined).
- `created_by_user_id` (UUID, FK -> auth.users, nullable).
- `last_status_changed_at` (timestamptz, nullable).
- `is_billable` (boolean, default true).
- `metadata` (jsonb, nullable) – arbitrary data (e.g., building access instructions).

Indexes:
- `idx_work_orders_org_id_status`.
- `idx_work_orders_org_id_customer_id`.
- `idx_work_orders_org_id_priority`.
- `idx_work_orders_org_id_sla_due_at` (for SLA/overdue tracking).

#### 3.2.3 `job_categories`

Top-level classification for jobs.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `name` (text, not null) – e.g., "HVAC", "Plumbing", "Electrical".
- `description` (text, nullable).
- `is_system` (boolean, default false) – indicates built-in categories.
- `sort_order` (integer, default 0).

Unique constraint: (`org_id`, `name`).

#### 3.2.4 `job_types`

More granular job type within categories.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `job_category_id` (UUID, FK -> `job_categories.id`, not null).
- `code` (text, nullable) – internal code, e.g., "HVAC-TUNE-UP".
- `name` (text, not null).
- `description` (text, nullable).
- `default_priority` (enum: `low`, `normal`, `high`, `emergency`, nullable).
- `default_duration_minutes` (integer, nullable) – suggested visit duration (used by scheduling module).
- `default_checklist_template_id` (UUID, FK -> `checklist_templates.id`, nullable).

Unique constraint: (`org_id`, `job_category_id`, `name`).

#### 3.2.5 `work_order_visits`

Represents specific scheduled visits associated with a work order. This is the primary integration point with the Scheduling & Dispatch module.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `work_order_id` (UUID, FK -> `work_orders.id`, not null, on delete cascade).
- `scheduled_start_at` (timestamptz, nullable) – set by Scheduling module.
- `scheduled_end_at` (timestamptz, nullable).
- `technician_id` (UUID, FK -> auth.users, nullable) – assignment by Scheduling module.
- `crew_id` (UUID, nullable) – for multi-tech crews (optional, may be defined in Scheduling module).
- `status` (enum: `unscheduled`, `scheduled`, `en_route`, `on_site`, `completed`, `no_show`, `canceled`).
- `sequence_number` (integer, default 1) – for multi-visit jobs.
- `notes` (text, nullable).
- `created_by_user_id` (UUID, FK -> auth.users, nullable).
- `metadata` (jsonb, nullable) – route info, optimization scores, etc.

Indexes:
- `idx_work_order_visits_work_order_id`.
- `idx_work_order_visits_org_id_technician_id_scheduled_start_at`.

> Scheduling & Dispatch owns the business logic that sets `scheduled_*` fields and technicians; this module simply persists and exposes them.

#### 3.2.6 `work_order_status_history`

Audit of work order status transitions.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `work_order_id` (UUID, FK -> `work_orders.id`, not null, on delete cascade).
- `previous_status` (enum matching `work_orders.status`, nullable).
- `new_status` (enum matching `work_orders.status`, not null).
- `changed_by_user_id` (UUID, FK -> auth.users, nullable).
- `changed_at` (timestamptz, default now()).
- `reason` (text, nullable).
- `metadata` (jsonb, nullable) – context, such as channel (mobile, portal, API).

Indexes:
- `idx_work_order_status_history_work_order_id_changed_at`.

#### 3.2.7 `checklist_templates`

Reusable templates for checklists tied to job types/categories.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `name` (text, not null) – e.g., "HVAC Tune-Up Standard".
- `job_category_id` (UUID, FK -> `job_categories.id`, nullable).
- `job_type_id` (UUID, FK -> `job_types.id`, nullable).
- `description` (text, nullable).
- `is_active` (boolean, default true).
- `version` (integer, default 1) – for evolving templates.

Unique constraint: (`org_id`, `name`, `version`).

#### 3.2.8 `checklist_template_items`

Items/fields within a checklist template.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `template_id` (UUID, FK -> `checklist_templates.id`, not null, on delete cascade).
- `label` (text, not null) – e.g., "Check filter condition".
- `type` (enum: `checkbox`, `text`, `number`, `select`, `photo`, `temperature`, `pressure`, `reading`, `signature`).
- `options` (jsonb, nullable) – select options, ranges, units.
- `is_required` (boolean, default false).
- `sort_order` (integer, default 0).
- `default_value` (text, nullable).

Index: `idx_checklist_template_items_template_id_sort_order`.

#### 3.2.9 `work_order_checklists`

Instances of checklists attached to a work order or specific visit.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `work_order_id` (UUID, FK -> `work_orders.id`, not null, on delete cascade).
- `visit_id` (UUID, FK -> `work_order_visits.id`, nullable) – optional if checklist is visit-specific.
- `template_id` (UUID, FK -> `checklist_templates.id`, nullable).
- `name` (text, not null) – may default from template.
- `status` (enum: `not_started`, `in_progress`, `completed`).
- `created_by_user_id` (UUID, FK -> auth.users, nullable).
- `completed_by_user_id` (UUID, FK -> auth.users, nullable).
- `completed_at` (timestamptz, nullable).

Index: `idx_work_order_checklists_work_order_id`.

#### 3.2.10 `work_order_checklist_items`

Instantiated checklist items with technician responses.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `checklist_id` (UUID, FK -> `work_order_checklists.id`, not null, on delete cascade).
- `template_item_id` (UUID, FK -> `checklist_template_items.id`, nullable) – null if ad hoc item.
- `label` (text, not null).
- `type` (enum as in template).
- `is_required` (boolean, default false).
- `sort_order` (integer, default 0).
- `value_boolean` (boolean, nullable).
- `value_text` (text, nullable).
- `value_number` (numeric, nullable).
- `value_json` (jsonb, nullable) – for complex inputs.
- `attachment_id` (UUID, FK -> `work_order_attachments.id`, nullable) – for photo/signature fields.
- `recorded_by_user_id` (UUID, FK -> auth.users, nullable).
- `recorded_at` (timestamptz, nullable).

Index: `idx_work_order_checklist_items_checklist_id_sort_order`.

#### 3.2.11 `work_order_attachments`

Metadata for files stored in Supabase Storage.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `work_order_id` (UUID, FK -> `work_orders.id`, not null, on delete cascade).
- `visit_id` (UUID, FK -> `work_order_visits.id`, nullable).
- `uploaded_by_user_id` (UUID, FK -> auth.users, nullable).
- `storage_bucket` (text, not null) – e.g., `work-order-media`.
- `storage_path` (text, not null) – unique path/key in bucket.
- `file_name` (text, not null).
- `content_type` (text, nullable).
- `file_size_bytes` (bigint, nullable).
- `category` (enum: `before_photo`, `after_photo`, `video`, `document`, `signature`, `other`).
- `description` (text, nullable).

Index: `idx_work_order_attachments_work_order_id`.

#### 3.2.12 `work_order_notes`

Internal notes/comments.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `work_order_id` (UUID, FK -> `work_orders.id`, not null, on delete cascade).
- `visit_id` (UUID, FK -> `work_order_visits.id`, nullable).
- `author_user_id` (UUID, FK -> auth.users, nullable).
- `body` (text, not null).
- `is_internal_only` (boolean, default true) – controls visibility to customers.
- `created_at` (timestamptz, default now()).

Index: `idx_work_order_notes_work_order_id_created_at`.

> Optionally, these notes could be modeled as a subset of `crm_interactions`, but this TDD treats them as a work order–scoped construct with possible later linkage.

#### 3.2.13 `work_order_tags` and `work_order_tag_links`

Tagging for work orders.

`work_order_tags`:
- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `name` (text, not null).
- `description` (text, nullable).
- `color` (text, nullable).

Unique constraint: (`org_id`, `name`).

`work_order_tag_links`:
- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `work_order_id` (UUID, FK -> `work_orders.id`, not null, on delete cascade).
- `tag_id` (UUID, FK -> `work_order_tags.id`, not null, on delete cascade).
- `assigned_by_user_id` (UUID, FK -> auth.users, nullable).

Unique constraint: (`org_id`, `work_order_id`, `tag_id`).

#### 3.2.14 `work_order_ai_suggestions`

Stores AI-generated categorization, priority, and other suggestions.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `work_order_id` (UUID, FK -> `work_orders.id`, not null, on delete cascade).
- `suggested_job_category_id` (UUID, FK -> `job_categories.id`, nullable).
- `suggested_job_type_id` (UUID, FK -> `job_types.id`, nullable).
- `suggested_priority` (enum as in `work_orders.priority`, nullable).
- `suggested_checklist_template_ids` (uuid[] or jsonb, nullable).
- `confidence_score` (numeric, nullable).
- `explanation` (text, nullable) – human-readable explanation from AI.
- `provider` (text, nullable) – which AI provider/model.
- `status` (enum: `pending`, `applied`, `dismissed`, `expired`).
- `created_at` (timestamptz, default now()).
- `applied_at` (timestamptz, nullable).
- `applied_by_user_id` (UUID, FK -> auth.users, nullable).

Index: `idx_work_order_ai_suggestions_work_order_id_created_at`.

### 3.3 Relationships to Other Modules

- **CRM (TDD §1)**:
  - `work_orders.customer_id` -> `customers.id`.
  - `work_orders.location_id` -> `customer_locations.id`.
  - Status changes and completion events may create `crm_interactions` (e.g., "Job completed" notifications).
- **Scheduling & Dispatch (TDD §2)**:
  - `work_order_visits` is the primary shared entity.
  - Scheduling module controls `scheduled_*` fields and `technician_id`.
  - Work order module emits events when statuses change to/from `ready_for_scheduling`, `scheduled`, `in_progress`, `completed`.
- **Inventory & Asset Management (TDD §5)**:
  - Future tables like `work_order_materials`, `work_order_assets` will reference this module’s IDs.
  - Inventory module listens for work order completion to confirm usage.
- **Quoting & Invoicing (TDD §4)**:
  - `work_orders.origin_quote_id` links to quote records.
  - Completion data (labor time from visits, materials from checklists) feeds invoicing.
- **Technician Mobile App (TDD §6)**:
  - Uses `work_order_visits`, `work_order_checklists`, `work_order_attachments` for on-site work.
- **Customer Portal (TDD §7)**:
  - Exposes a filtered view of `work_orders`, `work_order_visits`, and selected attachments/notes.

## 4. API & Edge Function Design

This section defines logical APIs and flows. Implementation can use Supabase RPC (SQL functions) or Edge Functions (HTTP endpoints).

### 4.1 Authentication & Authorization

- All endpoints require:
  - Authenticated Supabase user (for staff/technicians) OR scoped customer token for portal access.
  - `org_id` derived from user profile or JWT claims.
- RLS ensures:
  - Users only access rows where `org_id` matches their organization.
  - Technicians only see work orders/visits assigned to them (or to their crew).
  - Customers only see work orders tied to their own `customer_id`.

### 4.2 Work Order Lifecycle APIs

#### 4.2.1 Create Work Order

- **Endpoint**: Edge Function `POST /work-orders`
- **Input**:
  - `customer_id`, `location_id`.
  - `title`, `description`.
  - Optional: `job_category_id`, `job_type_id`, `priority`, `requested_start_window_start/end`, `origin_quote_id`, `origin_maintenance_plan_id`.
  - Optional: initial notes, tags, and attachments (handled via separate upload flow or pre-signed URLs).
- **Behavior**:
  - Validate `customer_id` and `location_id` belong to caller’s `org_id`.
  - Insert row into `work_orders` with initial `status = draft` or `ready_for_scheduling` based on input.
  - Optionally trigger AI classification/prioritization Edge Function asynchronously.
  - Optionally generate default checklist(s) based on job type.
  - Emit internal event `work_order.created`.
- **Output**:
  - Newly created work order record with IDs.

#### 4.2.2 Update Work Order Details

- **Endpoint**: `PATCH /work-orders/:id`
- **Input**:
  - Partial updates: `title`, `description`, `job_category_id`, `job_type_id`, `priority`, `requested_start_window_*`, `is_billable`, metadata.
- **Behavior**:
  - Enforce that certain fields are immutable once in specific statuses (e.g., cannot change `customer_id` when `in_progress`).
  - If description or classification-related fields change significantly, optionally retrigger AI suggestions.
  - Emit `work_order.updated` event.

#### 4.2.3 Get Work Order Details

- **Endpoint**: `GET /work-orders/:id`
- **Behavior**:
  - Returns:
    - `work_orders` core fields.
    - Related `work_order_visits`.
    - Checklists and checklist items (optional pagination).
    - Attachments metadata.
    - Notes (subject to visibility).
    - Latest AI suggestions.
    - Status history.

Implementation:
- Single Edge Function aggregating via SQL joins or multiple Supabase client queries from frontend, depending on performance needs.

#### 4.2.4 List/Search Work Orders

- **Endpoint**: `GET /work-orders`
- **Query params**:
  - Text search (`q` for title/description/customer name).
  - Filters: `status`, `priority`, `job_category_id`, `job_type_id`, `customer_id`, `technician_id` (via visits), date ranges (created, scheduled, completed), `tag_id`.
  - Pagination (page, page_size) and sort options.
- **Behavior**:
  - Use indexes for common filters.
  - Join to `work_order_visits` for tech-specific filtering.

#### 4.2.5 Change Work Order Status

- **Endpoint**: `POST /work-orders/:id/status`
- **Input**:
  - `new_status`, optional `reason`.
- **Behavior**:
  - Validate allowed transitions via state machine rules (e.g., cannot go from `completed` back to `in_progress` without a callback flow).
  - Insert row into `work_order_status_history`.
  - Update `work_orders.status` and `last_status_changed_at`.
  - Potential side-effects:
    - When `new_status = ready_for_scheduling`: emit event for Scheduling module (e.g., `work_order.ready_for_scheduling`).
    - When `new_status = completed`: emit `work_order.completed` for CRM, invoicing, inventory.
  - Enforce role-based permissions (e.g., only managers can cancel completed work orders).

### 4.3 Visit Management APIs (Integration with Scheduling)

> Scheduling & Dispatch TDD defines how visits are created/assigned. Here we define the shared shape and simple operations, not scheduling algorithms.

#### 4.3.1 Get Work Order Visits

- **Endpoint**: `GET /work-orders/:id/visits`
- **Behavior**:
  - Returns `work_order_visits` records for given work order, ordered by `sequence_number` and `scheduled_start_at`.

#### 4.3.2 Technician Visit Status Updates

- **Endpoint**: `POST /visits/:id/status`
- **Input**:
  - `status` (e.g., `en_route`, `on_site`, `completed`, `no_show`), optional `notes`.
- **Behavior**:
  - Enforce that technician can only update visits assigned to them.
  - Update `work_order_visits.status`.
  - Optionally mirror certain transitions to work order status, e.g.:
    - First visit `on_site` -> set work order to `in_progress`.
    - All required visits `completed` -> set work order to `completed` (or `awaiting_review` if such status is added).
  - Emit events (e.g., for customer notifications).

### 4.4 Checklist & Form APIs

#### 4.4.1 Manage Checklist Templates (Back Office)

- **Endpoints**:
  - `POST /checklists/templates`
  - `PATCH /checklists/templates/:id`
  - `GET /checklists/templates`
  - `GET /checklists/templates/:id`
- **Behavior**:
  - CRUD operations on `checklist_templates` and nested `checklist_template_items`.
  - Role-based restriction (only admins/managers).

#### 4.4.2 Generate Work Order Checklists

- **Endpoint**: `POST /work-orders/:id/checklists/generate`
- **Input**:
  - Optional override: explicit `template_ids` to apply.
- **Behavior**:
  - Determine templates from:
    - `job_type.default_checklist_template_id`.
    - AI suggestions stored in `work_order_ai_suggestions`.
  - Create `work_order_checklists` and `work_order_checklist_items` from templates.

#### 4.4.3 Update Checklist Responses (Technician)

- **Endpoint**: `PATCH /checklists/:id/items`
- **Input**:
  - Array of updates: `{ item_id, value_* fields, attachment_id }`.
- **Behavior**:
  - Validate technician has access to the associated work order/visit.
  - Update items; mark checklist as `in_progress` or `completed` based on required fields.
  - Optionally, if certain items indicate safety/compliance issues, emit an alert event.

### 4.5 Attachments & Media APIs

- **Upload Flow**:
  - Client requests pre-signed URL or direct Supabase Storage upload for media.
  - After successful upload, client calls `POST /work-orders/:id/attachments` with metadata.

- **Endpoints**:
  - `POST /work-orders/:id/attachments`
  - `GET /work-orders/:id/attachments`
  - `DELETE /attachments/:id`

- **Behavior**:
  - Enforce media visibility rules (e.g., internal-only vs customer-visible).
  - Ensure Storage paths include `org_id` to enforce RLS-like semantics with signed URLs.

### 4.6 Integration & Event APIs

#### 4.6.1 Events Emitted by Work Order Module

Emitted via Edge Functions / message bus abstraction (implementation detail):

- `work_order.created`
- `work_order.updated`
- `work_order.status_changed`
- `work_order.ready_for_scheduling`
- `work_order.completed`

These events are consumed by:
- Scheduling module (for booking).
- CRM (for interaction logs).
- Inventory (for material usage prompts).
- Invoicing (for billing triggers).

#### 4.6.2 Events Consumed by Work Order Module

From Scheduling module:
- `visit.scheduled` – update `work_order_visits` with times and technician.
- `visit.rescheduled` – adjust schedule fields, notify stakeholders.
- `visit.canceled` – set visit `status = canceled`.

From Inventory module:
- `materials.usage_confirmed` – optional updates to work order summaries.

From CRM / Portal:
- `customer.request_created` – may be translated into a new draft work order.

### 4.7 Background Jobs & Maintenance

- Cron-style Edge Functions for:
  - **SLA monitoring**: find work orders where `sla_due_at` is near/past due, update status or create alerts.
  - **Stale work orders**: identify work orders stuck in certain statuses (e.g., `ready_for_scheduling` for too long).
  - **AI enrichment**: batch process work orders requiring (re)classification.

## 5. AI & Analytics Design

### 5.1 AI-Powered Job Categorization & Prioritization

- **Edge Function**: `classify_work_order`
  - **Inputs**:
    - Work order description, title, source channel.
    - Customer history: prior jobs, average ticket size, SLA tier.
    - Time of day, location, any structured pre-filled fields.
  - **Process**:
    - Calls external LLM/AI provider with prompt describing available `job_categories`, `job_types`, and `priority` levels.
    - Receives structured response with suggested category, type, priority, and explanation.
  - **Outputs**:
    - Insert row into `work_order_ai_suggestions`.
    - Optionally auto-apply if within confidence threshold and org settings allow.
  - **Configuration**:
    - Org-level settings: auto-apply vs suggest-only, mapping between AI labels and internal codes.

### 5.2 AI-Assisted Checklists & Notes

- **Edge Function**: `suggest_checklists_for_work_order`
  - Uses job type, history, and environment to propose checklist templates and additional items.
  - Writes template IDs and suggestions to `work_order_ai_suggestions.suggested_checklist_template_ids` or creates extra checklist items.

- **Edge Function**: `summarize_work_order`
  - After completion, aggregates checklist responses, notes, and attachments metadata.
  - Generates readable summary for customer invoices or internal reports.

### 5.3 Analytics & Reporting

- Derived views and materialized views for:
  - Work order volume by status, category, type, technician, time period.
  - SLA compliance rates (on-time vs late).
  - Rework/callback rate (work orders linked to previous ones via tags).
  - AI suggestion accuracy (comparison between suggestions and final chosen values).

Implementation tasks:
- Create SQL views (e.g., `vw_work_order_kpis`) and indexes for common reporting queries.

## 6. Frontend (Next.js / Technician App) UI/UX Design

### 6.1 Office / Dispatcher Web UI

- **Work Orders List Page**:
  - Filters by status, priority, category, type, SLA risk, technician (via visits).
  - Quick indicators for overdue SLAs, unscheduled work orders, high-priority jobs.
- **Work Order Detail Page**:
  - Header: customer, location, status, priority, SLA time, tags.
  - Tabs/sections:
    - Overview (description, AI suggestions, classification).
    - Schedule/Visits (visit list, including fields filled by Scheduling module).
    - Checklists (status and responses).
    - Attachments.
    - Notes & history (including status history).
- **Checklist Template Management**:
  - UI to define/edit templates and items (drag-and-drop ordering, required flags).

### 6.2 Technician UI (Web or Mobile)

- **My Jobs / Visits List**:
  - Shows upcoming visits with date/time, location, and job summary.
- **Visit Detail / Work Order Execution Screen**:
  - Start/complete visit, update visit status (`en_route`, `on_site`, etc.).
  - View customer/location, job description, priority.
  - Fill out checklists with offline support (client-side caching and sync).
  - Capture photos/videos and upload.
  - Add notes specific to the visit and mark selected notes as customer-visible.

### 6.3 Customer Portal UI (Read-Only)

- **Service History**:
  - List of past and current work orders with high-level status and dates.
- **Work Order Detail (Customer View)**:
  - Limited fields (title, description, dates, status, customer-visible attachments and notes).
  - Read-only checklist summary where appropriate.

### 6.4 Data Access Patterns

- Use Supabase JS client for:
  - Simple table reads/writes (notes, attachments metadata, checklist responses).
  - Real-time subscriptions for status updates on relevant work orders/visits.
- Use Edge Functions for:
  - State transitions and lifecycle enforcement.
  - AI functions.
  - Bulk operations and cross-module interactions.

## 7. Security, Privacy, and Compliance

### 7.1 RLS & Role Policies

- For all tables:
  - `org_id = current_setting('app.current_org_id')::uuid` (or equivalent Supabase pattern).
- Role-specific filters:
  - **Technicians**: `work_order_visits.technician_id = auth.uid()` or membership in relevant crew.
  - **Customers**: access only to work orders where `customer_id` matches their CRM customer record.
- Write operations:
  - Restrict status changes, cancellation, and template edits to certain roles.

### 7.2 Media & PII Protection

- Store media in dedicated Storage buckets with path patterns including `org_id` and `work_order_id`.
- Use signed URLs with short TTLs for media access; never expose raw bucket keys in frontend.
- Avoid storing sensitive PII in unstructured notes where possible; encourage structured fields.

### 7.3 Auditing

- Maintain full audit trails via:
  - `work_order_status_history`.
  - `work_order_notes` (immutable once written, or soft-edit approach with revision fields).
  - Access logs configurable at Edge Function level for sensitive operations.

## 8. Non-Functional Requirements

- **Performance**:
  - List/search operations for up to ~100k work orders per org should respond in \< 500–800ms with proper indexes.
  - Checklist and status updates from technicians should complete in \< 300ms under normal load.
- **Scalability**:
  - Supabase Pro tier recommended; sharding by `org_id` not required initially but schema should not assume global uniqueness beyond PKs.
  - AI calls should be asynchronous for large volumes; synchronous classification allowed for single work order creation.
- **Reliability**:
  - Idempotent status change endpoints (safe from double submissions).
  - Graceful degradation if AI services are unavailable (fall back to manual classification).
- **Offline Support**:
  - Technician clients should cache assigned work orders, checklists, and attachments metadata locally, syncing when connectivity is restored.
- **Observability**:
  - Logs and metrics for Edge Functions handling state transitions and AI calls.
  - Alerts for failed background jobs (e.g., SLA monitor).

## 9. Implementation Notes & Open Questions

- **Open Questions**:
  - Exact state machine transitions and whether intermediate states like `awaiting_parts`, `awaiting_customer_approval` should be in this module or in a separate workflow engine.
  - How strongly to couple work order completion to invoicing (e.g., must an invoice be created before status can be set to `completed`?).
  - Which AI provider(s) and models to use, and how to handle per-org customization of prompts and thresholds.
  - Final decision on representing work order notes as separate entities vs reusing `crm_interactions`.
- **Extension Points**:
  - Add `work_order_materials` and `work_order_labor_logs` once Inventory & Invoicing designs are finalized.
  - Support recurring/preventive maintenance schedules via templates that generate future work orders.
  - Deeper integration with IoT/connected equipment to automatically create work orders based on sensor events.

This TDD is structured to be consumed by an LLM or engineering team to generate concrete tasks such as DDL migrations, Edge Function implementations, RLS policy definitions, backend services, and frontend/mobile UI for comprehensive Work Order Management, while maintaining a clean separation from Scheduling & Dispatch logic.


