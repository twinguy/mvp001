# Technical Design Document – Section 2: Scheduling & Dispatch

## 1. Purpose & Scope

This Technical Design Document (TDD) defines the **Scheduling & Dispatch** capabilities for the HVAC, Plumbing, and Electrical services platform described in `functional.md`. It provides sufficient technical detail for engineers (and LLM-based task generators) to decompose the Scheduling & Dispatch module into concrete implementation tasks.

- **In-scope capabilities (from `functional.md`, §2 Scheduling & Dispatch)**:
  - Real-time job scheduling and technician dispatch.
  - Route optimization using AI.
  - Calendar integration (Google, Outlook, etc.).
  - Automated appointment reminders (SMS, email, push).
  - Emergency/priority job handling.
- **Additional derived capabilities needed for a production-ready module**:
  - Technician capacity & availability management (shifts, time-off, on-call rotations).
  - Skill- and zone-based assignment (tech skills, certifications, service areas).
  - SLA/compliance windows and appointment time-slot handling.
  - Real-time status updates and map-based dispatch board.
  - Rescheduling & re-optimization when jobs or availability change.
  - Integration hooks with Work Order Management, Technician Mobile App, and Customer Portal.
- **Out-of-scope (defined in other TDDs, but referenced here)**:
  - Detailed Work Order domain (Section 3 TDD).
  - Full Technician Mobile App UX (Section 6 TDD).
  - Customer Portal booking flows (Section 7 TDD).
  - Marketing/notification templates and global communication engines (Sections 8, 10).

The Scheduling & Dispatch module must run on **Supabase (PostgreSQL + Auth + Edge Functions)** and expose functionality to a **Next.js web application on Vercel**, consistent with `tooling.md`.

## 2. Target Platform & High-Level Architecture

### 2.1 Platform Alignment (from `tooling.md`)

- **Backend**: Supabase
  - PostgreSQL for primary data (jobs, assignments, technician availability, route plans, calendar sync state).
  - Supabase Auth for users (dispatchers, technicians, admins).
  - Row-Level Security (RLS) for multi-tenant and role-based access.
  - Supabase Edge Functions for:
    - Scheduling & dispatch orchestration (e.g., auto-assign, auto-reschedule).
    - AI route optimization calls to external APIs.
    - Calendar provider webhooks and synchronization.
    - Reminder and notification orchestration (calling email/SMS/push providers).
- **Frontend**: Vercel
  - Next.js web application.
  - Supabase JS client for real-time data (job status, technician locations).
  - Edge Function integration for optimization and calendar/webhook flows.
- **Mobile (Technician App)**:
  - Flutter or React Native app using Supabase SDKs.
  - Real-time subscription to job assignments, job status, and route plans.

### 2.2 Logical Components

1. **Scheduling Data Layer (PostgreSQL)**:
   - Core entities:
     - Technicians & capabilities: `technician_profiles`, `technician_skills`, `technician_service_zones`, `technician_shifts`, `technician_time_off`.
     - Jobs and appointments: `dispatch_jobs`, `job_assignments`, `job_time_windows`.
     - Routing: `route_plans`, `route_stops`.
     - Calendar integration: `calendar_integrations`, `calendar_events`.
     - Notifications: `job_notifications` (scheduling-related), with hooks into shared messaging infrastructure.
2. **Scheduling Service Layer (Edge Functions + SQL)**:
   - Encapsulates complex workflows:
     - Auto-scheduling / re-scheduling jobs (capacity, skills, zones, travel time).
     - AI-driven route optimization.
     - Emergency job insertion and rebalancing.
     - Calendar synchronization with Google/Outlook.
     - Reminder generation and dispatch.
3. **API Layer**:
   - **Direct DB / RPC** via Supabase for simple CRUD (e.g., maintaining shifts, time off, basic job metadata).
   - **HTTP Edge Functions** for higher-level actions:
     - `auto_schedule_job`, `reoptimize_day`, `insert_emergency_job`, `sync_calendar`, etc.
4. **UI Layer (Next.js Dispatch Console)**:
   - Scheduler/dispatcher views:
     - Drag-and-drop schedule board (timeline and map).
     - Technician availability and capacity views.
     - Route visualization and what-if planning tools.
5. **Integrations**:
   - Mapping & routing APIs (e.g., Google Maps, Mapbox, or other routing services).
   - Calendar providers (Google Calendar, Outlook/Microsoft 365).
   - Messaging providers for reminders (SMS/email/push).

### 2.3 Multi-Tenancy & Access Control

Assume a multi-tenant Supabase project with `org_id` used across tables:

- Every scheduling-related table includes an `org_id` column (FK -> `orgs.id` as defined in other modules).
- **Dispatchers/Admins**:
  - Full CRUD on jobs, assignments, shifts, and routes within their `org_id`.
- **Technicians**:
  - Read-only access to their schedule, jobs, and routes (`job_assignments` filtered by technician).
  - Ability to update status fields (`accepted`, `en_route`, `on_site`, `completed`, etc.) and ETAs.
- **Customers** (via portal):
  - Limited read-only access to their own booked appointments (through separate portal APIs/views).

RLS policies must enforce `org_id` isolation and role-based restrictions (e.g., technicians cannot modify other technicians’ assignments).

## 3. Domain Model & Data Design

### 3.1 Conceptual Overview

Key concepts:

- **Technician**: A user who performs jobs in the field; has skills, certifications, service zones, and an availability calendar (shifts, time off, on-call).
- **Job**: A field visit to perform work at a customer location (linked to a Work Order). Has an estimated duration, priority, SLA window, and scheduling state.
- **Job Assignment**: A specific technician scheduled for a job with start/end times and status (assigned, accepted, in-progress, completed).
- **Route Plan**: An ordered set of stops (jobs and travel segments) for a technician for a given day.
- **Time Window**: Constraints on when a job can occur (e.g., 8–10AM arrival window, SLA deadlines).
- **Calendar Integration**: The mapping of internal jobs/assignments to external calendar events and the tokens/configuration needed to sync.
- **Notifications**: Scheduling-specific reminder/notification events triggered before, during, and after appointments.

### 3.2 Tables & Columns (Conceptual Schema)

> Note: Types and constraints should be implemented as Supabase/Postgres DDL. Names and relationships are designed to be LLM-friendly but can be refined during implementation.

#### 3.2.1 `technician_profiles`

Scheduling-view of technicians (may be extended/consumed by Technician Mobile App module).

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `user_id` (UUID, FK -> `auth.users.id`, unique, not null).
- `display_name` (text, not null).
- `employment_type` (enum: `employee`, `contractor`, `subcontractor`).
- `is_active` (boolean, default true).
- `home_base_location_id` (UUID, FK -> `customer_locations.id` or separate `locations` table; nullable).
- `default_service_zone_id` (UUID, FK -> `service_zones.id`, nullable).
- `max_daily_work_minutes` (integer, nullable; default capacity).
- `max_concurrent_jobs` (integer, default 1) – for overlapping assignments rules.
- `vehicle_type` (enum: `van`, `truck`, `car`, `other`, nullable).
- `metadata` (jsonb, nullable) – e.g., certifications, union info.
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Indexes:
- `idx_technician_profiles_org_id`.
- Unique index on (`org_id`, `user_id`).

#### 3.2.2 `technician_skills`

Normalized mapping from technicians to skills needed for job matching.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `technician_id` (UUID, FK -> `technician_profiles.id`, not null, on delete cascade).
- `skill_code` (text, not null) – e.g., `hvac_install`, `hvac_service`, `plumbing_drain`, `electrical_panel`.
- `proficiency_level` (enum: `junior`, `mid`, `senior`, `expert`).
- `is_primary` (boolean, default false).
- `created_at` (timestamptz, default now()).

Unique constraint:
- (`org_id`, `technician_id`, `skill_code`).

#### 3.2.3 `service_zones`

Service areas or territories for dispatch optimization.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `name` (text, not null).
- `description` (text, nullable).
- `polygon` (geometry or jsonb, nullable) – shape representing zone.
- `postal_codes` (text[], nullable) – alternative to polygon.
- `is_active` (boolean, default true).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

#### 3.2.4 `technician_service_zones`

Many-to-many link between technicians and service zones.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `technician_id` (UUID, FK -> `technician_profiles.id`, not null, on delete cascade).
- `service_zone_id` (UUID, FK -> `service_zones.id`, not null, on delete cascade).
- `is_primary` (boolean, default false).
- `created_at` (timestamptz, default now()).

Unique constraint:
- (`org_id`, `technician_id`, `service_zone_id`).

#### 3.2.5 `technician_shifts`

Represents scheduled working time blocks for technicians (availability).

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `technician_id` (UUID, FK -> `technician_profiles.id`, not null).
- `starts_at` (timestamptz, not null).
- `ends_at` (timestamptz, not null).
- `shift_type` (enum: `regular`, `on_call`, `overtime`, `training`).
- `recurrence_rule` (text, nullable) – iCal RRULE string for repeating shifts (optional).
- `is_active` (boolean, default true).
- `created_by_user_id` (UUID, FK -> `auth.users.id`, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Indexes:
- `idx_technician_shifts_org_id_technician_id_starts_at`.

#### 3.2.6 `technician_time_off`

Blocks of time when the technician is unavailable.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `technician_id` (UUID, FK -> `technician_profiles.id`, not null).
- `starts_at` (timestamptz, not null).
- `ends_at` (timestamptz, not null).
- `reason` (enum: `vacation`, `sick`, `personal`, `other`).
- `notes` (text, nullable).
- `created_by_user_id` (UUID, FK -> `auth.users.id`, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Indexes:
- `idx_technician_time_off_org_id_technician_id_starts_at`.

#### 3.2.7 `dispatch_jobs`

Represents a schedulable job/visit, typically linked to a Work Order but focused on scheduling attributes.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `related_work_order_id` (UUID, nullable) – FK to work order table (defined in Work Order TDD).
- `customer_id` (UUID, FK -> `customers.id`, not null).
- `location_id` (UUID, FK -> `customer_locations.id`, not null).
- `title` (text, not null) – short description shown in schedule board.
- `description` (text, nullable).
- `job_type` (text, nullable) – e.g., `maintenance`, `install`, `repair`, `inspection`.
- `priority` (enum: `low`, `normal`, `high`, `emergency`).
- `status` (enum: `unscheduled`, `scheduled`, `dispatched`, `in_progress`, `completed`, `canceled`).
- `estimated_duration_minutes` (integer, not null, default 60).
- `required_skills` (text[] or jsonb, nullable) – skill codes required.
- `required_crew_size` (integer, default 1).
- `service_zone_id` (UUID, FK -> `service_zones.id`, nullable).
- `sla_start_at` (timestamptz, nullable) – earliest allowed start (SLA).
- `sla_end_at` (timestamptz, nullable) – latest allowed completion (SLA).
- `is_customer_booked` (boolean, default false) – true if booked via portal.
- `notes_internal` (text, nullable).
- `created_by_user_id` (UUID, FK -> `auth.users.id`, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Indexes:
- `idx_dispatch_jobs_org_id_status_priority`.
- `idx_dispatch_jobs_org_id_sla_start_at`.

#### 3.2.8 `job_time_windows`

Appointment windows presented to/selected by customers or chosen by dispatchers.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `dispatch_job_id` (UUID, FK -> `dispatch_jobs.id`, not null, on delete cascade).
- `window_start` (timestamptz, not null).
- `window_end` (timestamptz, not null).
- `source` (enum: `system_suggested`, `dispatcher_selected`, `customer_selected`).
- `is_selected` (boolean, default false).
- `created_at` (timestamptz, default now()).

Indexes:
- `idx_job_time_windows_org_id_dispatch_job_id`.

#### 3.2.9 `job_assignments`

Mapping between jobs and technicians/crews with scheduled times and status.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `dispatch_job_id` (UUID, FK -> `dispatch_jobs.id`, not null, on delete cascade).
- `technician_id` (UUID, FK -> `technician_profiles.id`, not null).
- `assigned_by_user_id` (UUID, FK -> `auth.users.id`, nullable).
- `scheduled_start_at` (timestamptz, not null).
- `scheduled_end_at` (timestamptz, not null).
- `arrival_window_start` (timestamptz, nullable) – for customer comms.
- `arrival_window_end` (timestamptz, nullable).
- `status` (enum: `assigned`, `accepted`, `declined`, `en_route`, `on_site`, `completed`, `no_show`, `canceled`).
- `tech_eta_at` (timestamptz, nullable) – dynamic ETA.
- `sequence_in_route` (integer, nullable) – relative order in technician’s route.
- `is_primary_technician` (boolean, default true).
- `notes` (text, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Indexes:
- `idx_job_assignments_org_id_technician_id_scheduled_start_at`.
- `idx_job_assignments_org_id_dispatch_job_id`.

#### 3.2.10 `route_plans`

Daily or time-bounded route plan for a technician, typically optimized by AI.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `technician_id` (UUID, FK -> `technician_profiles.id`, not null).
- `date` (date, not null) – canonical date for the route.
- `status` (enum: `draft`, `proposed`, `finalized`, `in_progress`, `completed`, `canceled`).
- `optimization_strategy` (enum: `time_minimization`, `distance_minimization`, `priority_first`, `balanced`).
- `optimization_metadata` (jsonb, nullable) – details returned by optimizer (e.g., total distance, cost).
- `generated_by` (enum: `manual`, `rule_engine`, `ai_optimizer`).
- `generated_by_user_id` (UUID, FK -> `auth.users.id`, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Unique constraint:
- (`org_id`, `technician_id`, `date`, `status` filtered to active statuses, as needed).

#### 3.2.11 `route_stops`

Individual stops in a route, mapped to assignments or navigation waypoints.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `route_plan_id` (UUID, FK -> `route_plans.id`, not null, on delete cascade).
- `job_assignment_id` (UUID, FK -> `job_assignments.id`, nullable) – null for non-job stops (e.g., depot).
- `stop_type` (enum: `depot_start`, `job`, `break`, `fuel`, `depot_end`, `other`).
- `sequence` (integer, not null).
- `latitude` (numeric, nullable).
- `longitude` (numeric, nullable).
- `planned_arrival_at` (timestamptz, nullable).
- `planned_departure_at` (timestamptz, nullable).
- `actual_arrival_at` (timestamptz, nullable).
- `actual_departure_at` (timestamptz, nullable).
- `travel_time_minutes_from_prev` (integer, nullable).
- `distance_km_from_prev` (numeric, nullable).
- `notes` (text, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Indexes:
- `idx_route_stops_org_id_route_plan_id_sequence`.

#### 3.2.12 `calendar_integrations`

OAuth/connection config for external calendars per user/org.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `user_id` (UUID, FK -> `auth.users.id`, not null).
- `provider` (enum: `google`, `microsoft`).
- `provider_account_id` (text, not null).
- `access_token` (text, encrypted/managed via secrets, not plain).
- `refresh_token` (text, encrypted, nullable if using provider account-level config).
- `expires_at` (timestamptz, nullable).
- `calendar_id` (text, not null) – target calendar.
- `scope` (text, nullable).
- `is_active` (boolean, default true).
- `metadata` (jsonb, nullable).
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

#### 3.2.13 `calendar_events`

Mapping from internal jobs/assignments to external calendar events.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `calendar_integration_id` (UUID, FK -> `calendar_integrations.id`, not null).
- `job_assignment_id` (UUID, FK -> `job_assignments.id`, nullable).
- `dispatch_job_id` (UUID, FK -> `dispatch_jobs.id`, nullable).
- `provider_event_id` (text, not null).
- `status` (enum: `scheduled`, `updated`, `canceled`, `deleted_by_user`).
- `last_synced_at` (timestamptz, nullable).
- `sync_direction` (enum: `internal_to_external`, `external_to_internal`, `bidirectional`).
- `metadata` (jsonb, nullable) – original payload snapshot.
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Indexes:
- `idx_calendar_events_org_id_provider_event_id`.

#### 3.2.14 `job_notifications`

Scheduling-related reminders and notifications for jobs and assignments.

- `id` (UUID, PK).
- `org_id` (UUID, FK -> `orgs.id`, not null).
- `dispatch_job_id` (UUID, FK -> `dispatch_jobs.id`, nullable).
- `job_assignment_id` (UUID, FK -> `job_assignments.id`, nullable).
- `recipient_type` (enum: `customer`, `technician`, `dispatcher`).
- `recipient_contact_id` (UUID, nullable) – FK to `customer_contacts.id` or user contact; resolved by service.
- `channel` (enum: `sms`, `email`, `push`, `in_app`).
- `notification_type` (enum: `booking_confirmation`, `pre_appointment_reminder`, `same_day_reminder`, `tech_on_the_way`, `reschedule_notice`, `cancellation_notice`).
- `scheduled_send_at` (timestamptz, not null).
- `sent_at` (timestamptz, nullable).
- `status` (enum: `pending`, `sent`, `failed`, `canceled`).
- `error_message` (text, nullable).
- `metadata` (jsonb, nullable) – template variables, provider IDs.
- `created_at` (timestamptz, default now()).
- `updated_at` (timestamptz, default now()).

Indexes:
- `idx_job_notifications_org_id_scheduled_send_at_status`.

### 3.3 Relationships to Other Modules

- **CRM (Section 1)**:
  - `dispatch_jobs.customer_id` -> `customers.id`.
  - `dispatch_jobs.location_id` -> `customer_locations.id` (for service location, routing).
- **Work Order Management (Section 3)**:
  - `dispatch_jobs.related_work_order_id` -> work order table (to be defined).
  - Work orders emit events (created, updated, completed) consumed by scheduling automation.
- **Technician Mobile App (Section 6)**:
  - Technicians consume `job_assignments` and `route_plans` via mobile APIs and Supabase realtime.
- **Customer Portal (Section 7)**:
  - Customers select from `job_time_windows` and view `dispatch_jobs` status & ETAs (read-only).
- **Communication Module (Sections 8/10)**:
  - `job_notifications` may integrate with a shared messaging engine and template system.

## 4. API & Edge Function Design

This section describes logical APIs; implementation may use:

- Supabase Postgres RPC functions (for transactional query/update).
- Supabase Edge Functions (HTTP endpoints) for orchestration, AI calls, external APIs, and webhooks.

### 4.1 Authentication & Authorization

- All Scheduling & Dispatch APIs require an authenticated Supabase user (JWT).
- `org_id` is resolved from user profile or JWT claims and enforced via RLS.
- Role/permission assumptions:
  - **Dispatcher/Admin**: full scheduling control.
  - **Technician**: read-only on their own assignments/routes; may update status and ETA.
  - **Customer**: limited to their own jobs via portal APIs (no direct schedule manipulation).

### 4.2 Technician Availability & Configuration APIs

#### 4.2.1 Manage Technician Profiles

- **Endpoints**:
  - `POST /dispatch/technicians`
  - `PATCH /dispatch/technicians/:id`
  - `GET /dispatch/technicians`
  - `GET /dispatch/technicians/:id`
- **Behavior**:
  - CRUD operations on `technician_profiles`, `technician_skills`, and `technician_service_zones`.
  - Enforce `org_id` and role-based permissions.

#### 4.2.2 Manage Shifts & Time Off

- **Endpoints**:
  - `POST /dispatch/technicians/:id/shifts`
  - `GET /dispatch/technicians/:id/shifts`
  - `POST /dispatch/technicians/:id/time_off`
  - `GET /dispatch/technicians/:id/time_off`
- **Behavior**:
  - Insert/update `technician_shifts` and `technician_time_off`.
  - Validations:
    - No invalid negative-length intervals.
    - Optional warnings on overlapping shifts vs time-off.

### 4.3 Job Creation, Scheduling & Dispatch APIs

#### 4.3.1 Create Dispatch Job

- **Endpoint**: `POST /dispatch/jobs`
- **Input**:
  - `customer_id`, `location_id`.
  - `title`, `description`, `job_type`.
  - `priority`, `estimated_duration_minutes`.
  - Optional `related_work_order_id`.
  - Optional `required_skills`, `required_crew_size`, `service_zone_id`.
  - Optional requested time windows or SLA (`sla_start_at`, `sla_end_at`).
  - `is_customer_booked` flag and booking source (portal vs CSR).
- **Behavior**:
  - Insert into `dispatch_jobs`.
  - Optionally create `job_time_windows` based on requested windows or system defaults.
  - Trigger auto-scheduling if configured (see 4.4).

#### 4.3.2 Manual Assignment & Rescheduling

- **Endpoints**:
  - `POST /dispatch/jobs/:id/assign` – create one or more `job_assignments`.
  - `PATCH /dispatch/assignments/:id` – update assignment times, technician, status.
  - `DELETE /dispatch/assignments/:id` – cancel/remove assignment.
- **Behavior**:
  - Validate technician availability (shifts, time-off).
  - Check for double-booking and capacity constraints.
  - Update `dispatch_jobs.status` based on assignments (e.g., `scheduled` when at least one active assignment exists).
  - Generate/update `job_notifications` and `calendar_events` as needed.

#### 4.3.3 Technician Status Updates (Mobile)

- **Endpoints**:
  - `PATCH /dispatch/assignments/:id/status`
  - `PATCH /dispatch/assignments/:id/eta`
- **Behavior**:
  - Technicians update their assignment `status` (`accepted`, `en_route`, `on_site`, `completed`, etc.).
  - Techs can update `tech_eta_at`; this can trigger updated customer notifications (e.g., "tech on the way").

### 4.4 Auto-Scheduling & AI Route Optimization APIs

#### 4.4.1 Auto-Schedule Single Job

- **Endpoint**: `POST /dispatch/jobs/:id/auto_schedule`
- **Behavior**:
  - Inputs: job constraints (skills, priority, time windows, SLA, location), technician pool, availability, existing assignments.
  - Edge Function:
    - Queries relevant technicians/availability/time-off and travel times (via routing API).
    - Applies heuristic or optimization algorithm to find best assignment/time.
    - Creates `job_assignments` and optionally updates `job_time_windows` to reflect selected arrival window.
  - Returns proposed or committed schedule (based on parameter like `mode = propose` vs `mode = commit`).

#### 4.4.2 Optimize Technician Day / Route

- **Endpoint**: `POST /dispatch/technicians/:id/optimize_route`
- **Input**:
  - Date, list of target `dispatch_job_id`s or `job_assignment_id`s (optional; default: all open assignments for that day).
  - Optimization strategy (`time_minimization`, `distance_minimization`, `priority_first`, `balanced`).
- **Behavior**:
  - Edge Function builds cost matrix (travel time, distance) using mapping API.
  - Calls an optimization engine (external API or internal OR-Tools-like logic).
  - Writes/updates:
    - `route_plans` (one per technician and date).
    - `route_stops` (ordered stops).
    - `job_assignments.sequence_in_route`, `scheduled_start_at`/`scheduled_end_at` (if committing).

#### 4.4.3 Bulk Daily Optimization

- **Endpoint**: `POST /dispatch/days/:date/optimize_all`
- **Behavior**:
  - For each active technician, compute or update `route_plans` and `route_stops`.
  - Respect locks (e.g., assignments manually locked by dispatcher should not move).
  - Return summary (distance, estimated time savings).

#### 4.4.4 Emergency/Priority Job Insertion

- **Endpoint**: `POST /dispatch/jobs/:id/insert_emergency`
- **Behavior**:
  - Edge Function evaluates current in-progress or scheduled routes.
  - Inserts emergency job into the best possible slot considering:
    - Priority level.
    - Technician skill/zone.
    - Current route sequences and travel times.
    - SLAs of existing jobs (avoid violating high-priority SLAs).
  - May:
    - Re-sequence some `route_stops`.
    - Propose or automatically adjust `job_assignments` times.
    - Generate reschedule notifications for affected customers.

### 4.5 Calendar Integration APIs

#### 4.5.1 Connect Calendar

- **Endpoint**: `POST /dispatch/calendar/connect`
- **Behavior**:
  - Initiates OAuth flow with provider (Google, Microsoft).
  - Stores/updates credentials and configuration in `calendar_integrations`.

#### 4.5.2 Sync Appointments to Calendar

- **Endpoint**: `POST /dispatch/calendar/sync`
- **Behavior**:
  - Edge Function:
    - Reads pending or changed `job_assignments` and `dispatch_jobs`.
    - For each technician/user with active calendar integration, creates/updates `calendar_events`.
    - Calls provider APIs to create/update/cancel events.
    - Updates `calendar_events.status` and `last_synced_at`.

#### 4.5.3 Provider Webhooks

- **Endpoints**:
  - `POST /webhooks/calendar/google`
  - `POST /webhooks/calendar/microsoft`
- **Behavior**:
  - Process external changes (e.g., event moved/canceled by technician outside app).
  - Map provider event IDs back to `calendar_events` and then `job_assignments`/`dispatch_jobs`.
  - Depending on org settings, either:
    - Apply updates to internal schedule (and notify dispatchers); or
    - Mark events as `deleted_by_user` and prompt for manual reconciliation.

### 4.6 Notifications & Reminders APIs

#### 4.6.1 Schedule Standard Reminders

- **Endpoint**: `POST /dispatch/jobs/:id/schedule_notifications`
- **Behavior**:
  - Based on org-level rules (e.g., 24h before, 2h before, "tech on way"), compute reminder times.
  - Insert `job_notifications` records per channel/recipient.

#### 4.6.2 Notification Dispatcher (Internal)

- **Edge Function (Cron)**: `process_job_notifications`
- **Behavior**:
  - Periodically (e.g., every minute) query `job_notifications` where `status = 'pending'` and `scheduled_send_at <= now()`.
  - For each:
    - Resolve template & content (likely via shared messaging service).
    - Call SMS/email/push providers.
    - Update `status`, `sent_at`, and `error_message` as needed.

## 5. AI & Analytics Design

### 5.1 AI Route Optimization

- Implement Edge Functions that orchestrate calls to:
  - Mapping/routing APIs for distance and travel time estimates.
  - Optimization logic (external service or internal algorithm) to minimize total travel time, minimize lateness, and respect constraints.
- **Inputs**:
  - Technician home base or current location.
  - List of jobs with locations, estimated durations, time windows, priorities.
  - Technician constraints (shifts, time-off, capacity).
- **Outputs**:
  - For each technician:
    - Ranked sequences of jobs with planned arrival/departure times.
    - Aggregate metrics (total distance, drive time, number of jobs).
  - Persisted as `route_plans` and `route_stops`.

### 5.2 AI-Based Job Assignment Suggestions

- Edge Function `suggest_assignments_for_job`:
  - Given `dispatch_job_id`, compute ranked technician candidates.
  - Use:
    - Historical job performance, first-time fix rate.
    - Skill match and proximity.
    - Current load for the day.
  - Output: list of candidate technicians with scores and rationales (for UI).

### 5.3 Analytics & KPIs

- Use analytics views or materialized views (defined elsewhere) that aggregate:
  - Average travel time per job/technician.
  - Schedule adherence and on-time arrival rates.
  - Utilization (%) of technicians vs capacity.
  - Emergency job response times.
- These metrics feed into the Reporting & Analytics module (Section 9) but rely on scheduling data structures.

## 6. Frontend (Vercel/Next.js) UI/UX Design

### 6.1 Dispatcher Views

- **Schedule Board (Timeline View)**:
  - Horizontal day/week view with technicians on vertical axis and time horizontally.
  - Visual blocks for `job_assignments` (color-coded by status/priority).
  - Drag-and-drop to change assignment times or technicians; triggers `PATCH /dispatch/assignments/:id`.
- **Map-Based Dispatch View**:
  - Map with technician positions (from mobile app; defined elsewhere) and job locations.
  - Visual overlay of `route_plans` and `route_stops`.
  - Filter controls (zone, skill, status, priority).
- **Job Creation & Detail Drawer**:
  - Create new `dispatch_jobs` (and associated `job_time_windows`).
  - Show linked work order, CRM info, and schedule options.
- **Capacity & Utilization View**:
  - Per-day/per-tech view of available vs scheduled hours.
  - Warnings for over-scheduling or SLA risks.

### 6.2 Technician & Customer UX Hooks

- **Technician Mobile**:
  - "My Day" list using `job_assignments` and `route_plans`.
  - Buttons to update status and ETAs.
- **Customer Portal**:
  - Appointment selection UI using `job_time_windows`.
  - Real-time appointment status and ETA updates (read-only).

### 6.3 Frontend Data Access Patterns

- Use Supabase JS client for:
  - Real-time subscriptions to `job_assignments`, `dispatch_jobs`, and `route_plans` (org-scoped).
  - Basic CRUD for technician configuration, shifts, and time-off.
- Use Edge Functions for:
  - Auto-scheduling, route optimization.
  - Calendar connect/sync flows.
  - Bulk operations (optimize day, insert emergency, batch reschedule).

## 7. Security, Privacy, and Compliance

- **RLS**:
  - All scheduling tables enforce `org_id` equality with current user’s org.
  - Technicians can only read/update `job_assignments` where `technician_id` maps to their own `technician_profiles` row.
- **Least Privilege**:
  - Dispatchers/Admins have broader scheduling rights; regular office staff may have limited roles (e.g., can create jobs but not edit others' assignments).
- **Data Protection**:
  - External provider tokens in `calendar_integrations` must be stored securely (encrypted, not directly readable by frontend).
  - PII for customers (addresses, coordinates) is already covered by CRM; ensure scheduling accesses it via secure queries.
- **Auditability**:
  - Optional audit logs (separate table or extensions) for schedule changes: who moved which job, when, from/to what times.

## 8. Non-Functional Requirements

- **Performance**:
  - Schedule board queries should respond in \< 500ms for typical orgs (dozens of technicians, hundreds of daily jobs).
  - Route optimization may be longer-running; design as async jobs with progress indicators.
- **Scalability**:
  - Support at least:
    - 100+ technicians per org.
    - 1,000+ jobs per day per org.
  - Avoid N+1 query patterns in Edge Functions; rely on bulk queries and batch updates.
- **Reliability**:
  - All scheduling/optimization operations should be idempotent where possible (e.g., deterministic updates for the same input).
  - Robust error handling for external APIs (routing, calendar, messaging) with retries and fallback behavior.
- **Observability**:
  - Edge Functions must log key decisions (e.g., why certain technician was selected).
  - Include tracing IDs to relate optimization runs to resulting `route_plans` and `job_assignments`.

## 9. Implementation Notes & Open Questions

- **Tenancy Model**:
  - Confirm whether each Supabase project is single-tenant or multi-tenant; this affects RLS strategy but not schema shape.
- **Work Order & Job Unification**:
  - Decide whether `dispatch_jobs` and the Work Order table are the same physical table or linked via `related_work_order_id`.
  - If unified, scheduling fields may move into the Work Order schema and this TDD becomes a subset of that design.
- **External Providers**:
  - Choose mapping/routing provider(s) and calendar API libraries.
  - Choose messaging providers for SMS/email/push (or use a general message bus designed in another module).
- **Real-Time Location**:
  - Decide how technician GPS/location is ingested and stored (e.g., `technician_locations` table); only partial hints are given here.
- **Constraint Tuning**:
  - Define default business rules: max travel distance per day, late arrival tolerance, whether emergency jobs can bump scheduled jobs without manual approval.

This TDD is intended to be consumed by an LLM or engineering team to generate concrete implementation tasks, including DDL for scheduling-related tables, Edge Functions for auto-scheduling and integration, RLS policies, and frontend components/pages for the dispatch console, technician scheduling, and calendar/notification flows.


