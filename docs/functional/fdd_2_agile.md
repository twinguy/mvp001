## Scheduling & Dispatch Module – Agile User Stories (`fdd_2_agile.md`)

This document decomposes the **Scheduling & Dispatch** module defined in `fdd_2.md` (and aligned with `functional.md`, `tooling.md`, and `overview.md`) into epics and user stories suitable for implementation on **Supabase (Postgres, Auth, Edge Functions)** with a **Next.js frontend on Vercel**.

Each story includes:
- **User story** (role / intent / value)
- **Acceptance Criteria**
- **Definition of Done (DoD)**

Story IDs are for reference (not prescriptive for tooling).

---

## Epic 1 – Dispatch Multi-Tenancy, Roles, and Foundational Conventions

Establish cross-cutting conventions required by `fdd_2.md` including tenancy (`org_id`), roles, and how identity is resolved for RLS and Edge Functions.

### Story DISP-001 – Confirm Dispatch Tenancy Model and `org_id` Resolution

**As a** platform architect  
**I want** a clear, documented tenancy strategy for Scheduling & Dispatch data  
**So that** all dispatch data is correctly isolated per organization and consistently queryable.

**Acceptance Criteria**
- [ ] A decision is documented confirming a multi-tenant model using `org_id` for all dispatch tables as described in `fdd_2.md` §2.3.
- [ ] A convention is documented for deriving `org_id` from the authenticated user (e.g., via a `profiles` table or JWT claims), including how Edge Functions obtain it.
- [ ] If any cross-org “service role” access is required (e.g., cron processors), the pattern is documented (service role key only on server).

**Definition of Done**
- [ ] Documentation exists in repo docs explaining tenancy and `org_id` resolution for dispatch flows.
- [ ] Tenancy conventions are reviewed for consistency with existing modules (CRM, work orders, portal).

---

### Story DISP-002 – Define Dispatch Roles and Permission Matrix

**As a** product owner  
**I want** a clear role/permission matrix for dispatch operations  
**So that** RLS policies, APIs, and UI restrictions are consistent.

**Acceptance Criteria**
- [ ] Roles are defined at minimum for: **admin**, **dispatcher**, **technician**, and (optional) **CSR**/**office staff** as referenced in `fdd_2.md` §2.3 and §4.1.
- [ ] For each role, the matrix documents permissions for: technicians, jobs, assignments, routes, calendar integrations, and notifications.
- [ ] Any “customer” access is explicitly defined as via portal-specific APIs/views (read-only) per `fdd_2.md` §2.3.

**Definition of Done**
- [ ] Permission matrix is committed in Markdown and linked from `fdd_2_agile.md`.
- [ ] Matrix is validated against the stories below (no missing areas).

---

## Epic 2 – Core Scheduling Data Model (Supabase Postgres Schema)

Implement the Scheduling & Dispatch domain tables described in `fdd_2.md` §3.2 with constraints, indexes, and referential integrity.

### Story DISP-003 – Implement `technician_profiles` Table

**As a** dispatcher  
**I want** a reliable technician profile table  
**So that** scheduling and assignment can target real technicians with capacity and metadata.

**Acceptance Criteria**
- [ ] A `technician_profiles` table exists matching `fdd_2.md` §3.2.1 (including `org_id`, `user_id`, `display_name`, capacity fields, and timestamps).
- [ ] Unique constraint exists for (`org_id`, `user_id`).
- [ ] Index exists on `org_id` (and any other indices from the design).

**Definition of Done**
- [ ] DDL is created, migrated, and version-controlled.
- [ ] Test data exists for at least 2 technicians in one org and at least 1 technician in a second org.

---

### Story DISP-004 – Implement `technician_skills` Table

**As a** dispatcher  
**I want** skills to be modeled per technician  
**So that** auto-scheduling can match jobs to qualified technicians.

**Acceptance Criteria**
- [ ] A `technician_skills` table exists matching `fdd_2.md` §3.2.2.
- [ ] Unique constraint exists on (`org_id`, `technician_id`, `skill_code`).
- [ ] FK to `technician_profiles.id` uses cascade on delete.

**Definition of Done**
- [ ] DDL is migrated and version-controlled.
- [ ] Seed data demonstrates at least 2 skill codes per technician and differing proficiency levels.

---

### Story DISP-005 – Implement `service_zones` Table

**As a** dispatcher  
**I want** service zones to be represented  
**So that** dispatch can assign work by territory and reduce drive time.

**Acceptance Criteria**
- [ ] A `service_zones` table exists matching `fdd_2.md` §3.2.3.
- [ ] Storage approach for zone geometry is decided and documented (`polygon` as `jsonb` vs PostGIS geometry), including trade-offs.
- [ ] Zone activation (`is_active`) is supported.

**Definition of Done**
- [ ] DDL is migrated and version-controlled.
- [ ] At least one example zone exists using the chosen geometry approach.

---

### Story DISP-006 – Implement `technician_service_zones` Table

**As a** dispatcher  
**I want** technicians mapped to service zones  
**So that** dispatch can restrict assignments to allowed territories.

**Acceptance Criteria**
- [ ] A `technician_service_zones` table exists matching `fdd_2.md` §3.2.4.
- [ ] Unique constraint exists on (`org_id`, `technician_id`, `service_zone_id`).
- [ ] FKs use cascade on delete.

**Definition of Done**
- [ ] DDL is migrated and version-controlled.
- [ ] Seed data shows a technician with multiple zones and one marked primary.

---

### Story DISP-007 – Implement `technician_shifts` Table

**As a** dispatcher  
**I want** shifts represented  
**So that** availability can be computed for scheduling and route optimization.

**Acceptance Criteria**
- [ ] A `technician_shifts` table exists matching `fdd_2.md` §3.2.5.
- [ ] Data constraints prevent invalid intervals (e.g., `ends_at` must be after `starts_at`) or the rule is clearly documented if deferred.
- [ ] Index exists per `fdd_2.md` (e.g., (`org_id`, `technician_id`, `starts_at`)).

**Definition of Done**
- [ ] DDL is migrated and version-controlled.
- [ ] Seed data includes a repeating shift scenario documented via `recurrence_rule` (even if recurrence expansion is deferred).

---

### Story DISP-008 – Implement `technician_time_off` Table

**As a** dispatcher  
**I want** time off represented  
**So that** scheduling avoids unavailable technicians.

**Acceptance Criteria**
- [ ] A `technician_time_off` table exists matching `fdd_2.md` §3.2.6.
- [ ] Data constraints prevent invalid intervals (or are documented as deferred).
- [ ] Index exists per `fdd_2.md` (e.g., (`org_id`, `technician_id`, `starts_at`)).

**Definition of Done**
- [ ] DDL is migrated and version-controlled.
- [ ] Seed data includes overlapping time-off and shift examples for validation later.

---

### Story DISP-009 – Implement `dispatch_jobs` Table

**As a** dispatcher  
**I want** a schedulable “dispatch job” record  
**So that** I can plan work independent of the full work-order detail model.

**Acceptance Criteria**
- [ ] A `dispatch_jobs` table exists matching `fdd_2.md` §3.2.7 including SLA fields, `priority`, `status`, and estimated duration.
- [ ] FK relationships exist to CRM concepts (`customers`, `customer_locations`) as referenced in `fdd_2.md` §3.3.
- [ ] Indexes exist per `fdd_2.md` (e.g., status/priority, SLA start).

**Definition of Done**
- [ ] DDL is migrated and version-controlled.
- [ ] Seed data includes unscheduled and scheduled jobs and at least one emergency priority job.

---

### Story DISP-010 – Implement `job_time_windows` Table

**As a** dispatcher or customer (via portal)  
**I want** selectable time windows on a job  
**So that** appointments can respect customer/dispatcher constraints.

**Acceptance Criteria**
- [ ] A `job_time_windows` table exists matching `fdd_2.md` §3.2.8.
- [ ] Job time windows are cascade-deleted with a `dispatch_job`.
- [ ] A documented convention exists for “selected window” behavior (`is_selected`), including how multiple windows are handled.

**Definition of Done**
- [ ] DDL is migrated and version-controlled.
- [ ] Seed data includes multiple windows for one job with one selected.

---

### Story DISP-011 – Implement `job_assignments` Table

**As a** dispatcher  
**I want** job assignments modeled with scheduled times and status  
**So that** I can dispatch work to technicians and track progress.

**Acceptance Criteria**
- [ ] A `job_assignments` table exists matching `fdd_2.md` §3.2.9.
- [ ] Assignment status enum supports all states listed (`assigned`, `accepted`, `declined`, `en_route`, `on_site`, `completed`, `no_show`, `canceled`).
- [ ] Indexes exist per `fdd_2.md`.

**Definition of Done**
- [ ] DDL is migrated and version-controlled.
- [ ] Seed data includes at least one multi-assignment job (crew scenario) if `required_crew_size` is used.

---

### Story DISP-012 – Implement `route_plans` Table

**As a** dispatcher  
**I want** daily route plans  
**So that** optimized sequences can be stored, reviewed, and executed.

**Acceptance Criteria**
- [ ] A `route_plans` table exists matching `fdd_2.md` §3.2.10.
- [ ] A uniqueness strategy is defined and documented for “one active route plan per tech per date” (design mentions filtered unique constraints as needed).
- [ ] `optimization_metadata` can store optimizer results in a versionable JSON shape.

**Definition of Done**
- [ ] DDL is migrated and version-controlled.
- [ ] Seed data includes at least one route plan in `draft` and one in `finalized`.

---

### Story DISP-013 – Implement `route_stops` Table

**As a** dispatcher  
**I want** ordered route stops  
**So that** the route plan can represent job order, travel segments, and breaks.

**Acceptance Criteria**
- [ ] A `route_stops` table exists matching `fdd_2.md` §3.2.11.
- [ ] `sequence` ordering is enforced (at minimum via uniqueness or clear application logic) per `route_plan_id`.
- [ ] Index exists per `fdd_2.md` (`route_plan_id`, `sequence`).

**Definition of Done**
- [ ] DDL is migrated and version-controlled.
- [ ] Seed data includes at least one route with depot start/end and multiple job stops.

---

### Story DISP-014 – Implement `calendar_integrations` Table

**As a** technician or dispatcher  
**I want** my calendar connection stored securely  
**So that** appointments can sync to my external calendar.

**Acceptance Criteria**
- [ ] A `calendar_integrations` table exists matching `fdd_2.md` §3.2.12.
- [ ] Token handling is explicitly documented as **not stored in plaintext** and managed via secure mechanisms (e.g., encryption strategy, secrets, vault).
- [ ] Provider enum includes at least `google` and `microsoft`.

**Definition of Done**
- [ ] DDL is migrated and version-controlled.
- [ ] Security approach for token storage is documented and reviewed.

---

### Story DISP-015 – Implement `calendar_events` Table

**As a** system  
**I want** a mapping between internal assignments and external calendar events  
**So that** sync can be incremental, auditable, and reversible.

**Acceptance Criteria**
- [ ] A `calendar_events` table exists matching `fdd_2.md` §3.2.13.
- [ ] `sync_direction` enum supports `internal_to_external`, `external_to_internal`, `bidirectional`.
- [ ] Index exists for provider event ID lookup per `fdd_2.md`.

**Definition of Done**
- [ ] DDL is migrated and version-controlled.
- [ ] Seed data includes at least one mapping row for a job assignment.

---

### Story DISP-016 – Implement `job_notifications` Table

**As a** system  
**I want** scheduling-related notifications queued and tracked  
**So that** reminders and status messages can be sent reliably.

**Acceptance Criteria**
- [ ] A `job_notifications` table exists matching `fdd_2.md` §3.2.14 including recipient and channel enums.
- [ ] Index exists per `fdd_2.md` for efficient pending lookup by send time and status.
- [ ] A documented convention exists for how `recipient_contact_id` is resolved for customers vs technicians vs dispatchers.

**Definition of Done**
- [ ] DDL is migrated and version-controlled.
- [ ] Seed data includes pending and sent notifications across multiple channels.

---

## Epic 3 – Security: RLS Policies and Least-Privilege Access

Implement Row-Level Security to enforce `org_id` isolation and role-based permissions per `fdd_2.md` §2.3 and §7.

### Story DISP-017 – Enable RLS on All Dispatch Tables

**As a** security-conscious engineer  
**I want** RLS enabled everywhere in the dispatch schema  
**So that** no table can leak data across tenants by accident.

**Acceptance Criteria**
- [ ] RLS is enabled for all dispatch tables created in Epic 2.
- [ ] A baseline tenant-scoping policy exists for each table requiring `org_id` match.
- [ ] Policies are testable with at least two org contexts.

**Definition of Done**
- [ ] Policies are version-controlled and deployed.
- [ ] Positive/negative tests (manual checklist acceptable) confirm cross-org access is blocked.

---

### Story DISP-018 – Implement Role-Based Access for Dispatchers/Admins

**As a** dispatcher/admin  
**I want** full access to dispatch objects within my org  
**So that** I can schedule, adjust, and optimize work.

**Acceptance Criteria**
- [ ] Dispatchers/admins can CRUD: technicians (config only), jobs, assignments, route plans/stops, time windows, and notifications in their org.
- [ ] Dispatchers/admins can read calendar integration states (without exposing sensitive tokens).

**Definition of Done**
- [ ] RLS policies enforce privileges and are documented.
- [ ] Attempted access outside org is blocked.

---

### Story DISP-019 – Implement Technician Self-Scope Access

**As a** technician  
**I want** to see only my schedule, assignments, and routes  
**So that** I can perform my day’s work without seeing other technicians’ data.

**Acceptance Criteria**
- [ ] Technicians can read `job_assignments`, `dispatch_jobs`, and `route_plans`/`route_stops` only for their own `technician_profiles.id`.
- [ ] Technicians can update only allowed fields (e.g., assignment status and ETA) per `fdd_2.md` §4.1 and §4.3.3.
- [ ] Technicians cannot modify technician configuration for others (skills/zones/shifts/time-off) unless explicitly granted.

**Definition of Done**
- [ ] RLS policies are implemented and tested with at least two technicians in the same org.
- [ ] “Allowed update fields” are documented.

---

### Story DISP-020 – Define Customer Read-Only Access Strategy for Portal Flows

**As a** platform architect  
**I want** a documented strategy for customer (portal) access  
**So that** customers can see their own appointments without broad dispatch access.

**Acceptance Criteria**
- [ ] Strategy is documented (e.g., portal-specific API endpoints, DB views, or separate auth role) that limits access to a customer’s own `dispatch_jobs` and time windows only.
- [ ] Customer access excludes technician lists, other customers, and route plans.

**Definition of Done**
- [ ] Documented approach is reviewed for security risks and alignment with portal TDD (Section 7).

---

## Epic 4 – Technician Configuration & Availability APIs

Provide CRUD endpoints described in `fdd_2.md` §4.2 for technician profiles, skills/zones, shifts, and time off.

### Story DISP-021 – Create and Update Technician Profiles API

**As a** dispatcher/admin  
**I want** APIs to create and update technician profiles  
**So that** dispatch can onboard technicians and manage scheduling-relevant attributes.

**Acceptance Criteria**
- [ ] Endpoints exist per `fdd_2.md` §4.2.1 (`POST /dispatch/technicians`, `PATCH /dispatch/technicians/:id`).
- [ ] Updates support changing `display_name`, active status, capacity fields, home base, and default zone.
- [ ] Authorization enforces dispatcher/admin-only modifications.

**Definition of Done**
- [ ] API request/response examples are documented in Markdown.
- [ ] Error handling covers invalid user IDs, cross-org access, and missing required fields.

---

### Story DISP-022 – Manage Technician Skills API

**As a** dispatcher/admin  
**I want** to manage technician skills  
**So that** auto-scheduling and suggestions can match jobs to capabilities.

**Acceptance Criteria**
- [ ] Skill management is available either as part of technician update or via dedicated endpoints (documented).
- [ ] Unique constraint behavior (no duplicate skill_code per technician) is respected and yields useful errors.

**Definition of Done**
- [ ] API behavior is documented, including upsert vs replace semantics.
- [ ] Basic tests verify add/update/remove of skills.

---

### Story DISP-023 – Manage Technician Service Zones API

**As a** dispatcher/admin  
**I want** to manage technician service zones  
**So that** assignments can respect geographic eligibility.

**Acceptance Criteria**
- [ ] Zone membership management exists either embedded in technician update or via dedicated endpoints (documented).
- [ ] Primary zone logic is documented (how to ensure at most one primary zone).

**Definition of Done**
- [ ] API behavior is documented and validated with sample data.
- [ ] Basic tests cover adding and removing zones.

---

### Story DISP-024 – Create and List Shifts API

**As a** dispatcher/admin  
**I want** to create and view technician shifts  
**So that** availability is maintained for scheduling.

**Acceptance Criteria**
- [ ] Endpoints exist per `fdd_2.md` §4.2.2 (`POST /dispatch/technicians/:id/shifts`, `GET /dispatch/technicians/:id/shifts`).
- [ ] Validation prevents negative-length shifts or documents the behavior if deferred.
- [ ] Optional warnings exist for overlaps (shift vs time-off) or are explicitly documented as deferred.

**Definition of Done**
- [ ] API documentation exists with example payloads.
- [ ] Tests cover shift creation and filtering by date range.

---

### Story DISP-025 – Create and List Time Off API

**As a** dispatcher/admin  
**I want** to create and view technician time off  
**So that** scheduling avoids unavailable time.

**Acceptance Criteria**
- [ ] Endpoints exist per `fdd_2.md` §4.2.2 (`POST /dispatch/technicians/:id/time_off`, `GET /dispatch/technicians/:id/time_off`).
- [ ] Validation prevents negative-length time-off intervals or documents the behavior if deferred.
- [ ] Data is org-scoped and technician-scoped.

**Definition of Done**
- [ ] API documentation exists with example payloads.
- [ ] Tests cover time-off creation and overlap scenarios (at least detected or documented).

---

## Epic 5 – Dispatch Job Lifecycle APIs (Create, Assign, Reschedule, Cancel)

Implement core workflows per `fdd_2.md` §4.3 including job creation, manual assignment, and rescheduling/canceling.

### Story DISP-026 – Create Dispatch Job API

**As a** dispatcher/CSR  
**I want** to create a dispatch job with scheduling constraints  
**So that** it can be scheduled and tracked through completion.

**Acceptance Criteria**
- [ ] Endpoint exists per `fdd_2.md` §4.3.1 (`POST /dispatch/jobs`) with support for priority, estimated duration, skills, crew size, SLA and/or time windows.
- [ ] Job creation optionally creates `job_time_windows` (requested or system defaults) as described in `fdd_2.md` §4.3.1.
- [ ] `is_customer_booked` is supported for portal-origin jobs.

**Definition of Done**
- [ ] API docs include request/response examples for (a) dispatcher-created job and (b) customer-booked job.
- [ ] Error cases are documented (invalid customer/location IDs, invalid enums).

---

### Story DISP-027 – List and Filter Dispatch Jobs API

**As a** dispatcher  
**I want** to list jobs by status/priority/date  
**So that** I can manage the dispatch queue.

**Acceptance Criteria**
- [ ] A list endpoint exists (e.g., `GET /dispatch/jobs`) supporting filters for status, priority, SLA window, and “unscheduled only”.
- [ ] Pagination is supported.
- [ ] Results include fields needed for schedule board cards (title, location, priority, status, estimated duration).

**Definition of Done**
- [ ] Documented query params and example responses exist.
- [ ] Performance expectations for typical org sizes are documented (target \< 500ms for common filters per `fdd_2.md` §8).

---

### Story DISP-028 – View Dispatch Job Details API

**As a** dispatcher  
**I want** a job details endpoint  
**So that** I can view constraints, time windows, and assignments in one place.

**Acceptance Criteria**
- [ ] An endpoint exists (e.g., `GET /dispatch/jobs/:id`) returning job data plus:
  - [ ] `job_time_windows` (selected vs unselected)
  - [ ] current `job_assignments` (active and canceled as configured)
- [ ] Authorization is enforced by role/RLS.

**Definition of Done**
- [ ] API is documented and verified with seeded data.
- [ ] Errors are consistent for not-found vs unauthorized.

---

### Story DISP-029 – Manual Assign Job to Technician API

**As a** dispatcher  
**I want** to manually assign a job to a technician at a time  
**So that** I can schedule work when automation is not used or needs override.

**Acceptance Criteria**
- [ ] Endpoint exists per `fdd_2.md` §4.3.2 (`POST /dispatch/jobs/:id/assign`).
- [ ] Validations include:
  - [ ] availability vs shifts/time off
  - [ ] overlap/double-booking checks
  - [ ] capacity constraints (e.g., max daily minutes / max concurrent jobs) or a documented initial “MVP rule set”
- [ ] `dispatch_jobs.status` updates according to assignment existence rules in `fdd_2.md` §4.3.2.

**Definition of Done**
- [ ] Endpoint returns a created assignment and any warnings (if applicable).
- [ ] Validation rules and known limitations are documented.

---

### Story DISP-030 – Update Assignment (Reschedule/Reassign) API

**As a** dispatcher  
**I want** to reschedule or reassign an existing job assignment  
**So that** changes can be made as conditions evolve.

**Acceptance Criteria**
- [ ] Endpoint exists per `fdd_2.md` §4.3.2 (`PATCH /dispatch/assignments/:id`).
- [ ] Rescheduling triggers:
  - [ ] updates to `dispatch_jobs.status` if needed
  - [ ] calendar sync updates (recorded in `calendar_events`) or flags for later sync
  - [ ] notification updates (e.g., reschedule notice) via `job_notifications` logic
- [ ] A strategy is documented for “manual locks” that prevent optimizer changes (referenced in `fdd_2.md` §4.4.3).

**Definition of Done**
- [ ] API docs include reschedule and reassign examples.
- [ ] Tests cover rescheduling to an unavailable time (rejected) and allowed cases.

---

### Story DISP-031 – Cancel/Remove Assignment API

**As a** dispatcher  
**I want** to cancel or remove an assignment  
**So that** jobs can be unscheduled or moved without deleting history.

**Acceptance Criteria**
- [ ] Endpoint exists per `fdd_2.md` §4.3.2 (`DELETE /dispatch/assignments/:id`).
- [ ] Cancel semantics are documented: hard delete vs soft cancel via status; history retention is defined.
- [ ] Cancel triggers calendar event cancellation behavior and appropriate notifications if configured.

**Definition of Done**
- [ ] Behavior is documented and consistent with downstream sync/notification processors.
- [ ] Tests validate the job’s status updates appropriately after assignment removal.

---

### Story DISP-032 – Update Dispatch Job Status API (Non-Assignment Statuses)

**As a** dispatcher  
**I want** to update job status for operational reasons  
**So that** cancellations and completion states reflect reality.

**Acceptance Criteria**
- [ ] A job update endpoint exists (e.g., `PATCH /dispatch/jobs/:id`) allowing status changes such as canceled (with reason/notes).
- [ ] Status transition rules are documented (e.g., cannot mark completed if no assignment completed unless explicitly allowed).

**Definition of Done**
- [ ] Status transition rules are documented and enforced or explicitly marked as a later hardening item.
- [ ] Tests cover at least one invalid transition.

---

## Epic 6 – Technician Mobile Hooks: Status & ETA Updates

Provide technician-facing endpoints described in `fdd_2.md` §4.3.3 and ensure they are secure and real-time friendly.

### Story DISP-033 – Technician Assignment Status Update API

**As a** technician  
**I want** to update my assignment status (accepted, en route, on site, completed)  
**So that** dispatch and customers can see progress in real time.

**Acceptance Criteria**
- [ ] Endpoint exists per `fdd_2.md` §4.3.3 (`PATCH /dispatch/assignments/:id/status`).
- [ ] Technicians can only update assignments that belong to them (RLS enforced).
- [ ] Status transitions are validated (documented state machine).

**Definition of Done**
- [ ] State machine is documented in Markdown.
- [ ] Tests confirm technicians cannot update others’ assignments.

---

### Story DISP-034 – Technician ETA Update API

**As a** technician  
**I want** to update ETA for my next job  
**So that** customers can be informed accurately.

**Acceptance Criteria**
- [ ] Endpoint exists per `fdd_2.md` §4.3.3 (`PATCH /dispatch/assignments/:id/eta`) updating `tech_eta_at`.
- [ ] ETA changes can trigger notification scheduling (e.g., “tech on the way”) per `fdd_2.md` §4.3.3 and §4.6.
- [ ] Rate-limiting or deduping strategy is documented to avoid notification spam.

**Definition of Done**
- [ ] Notification trigger rules for ETA changes are documented.
- [ ] Tests cover rapid repeated ETA updates (dedupe behavior at least documented).

---

### Story DISP-035 – Real-Time Subscriptions for Technician Views

**As a** technician  
**I want** real-time updates to my assignments and route plan  
**So that** changes made by dispatch appear immediately in the mobile app.

**Acceptance Criteria**
- [ ] Real-time subscriptions are planned and documented for `job_assignments`, `dispatch_jobs`, `route_plans`, and `route_stops` (org-scoped + technician-scoped).
- [ ] A strategy is documented for minimizing payload size and handling reconnect/backfill.

**Definition of Done**
- [ ] A “realtime channels” spec is documented (tables, filters, expected events).
- [ ] Security review confirms realtime filters do not leak cross-tech or cross-org data.

---

## Epic 7 – Auto-Scheduling and Route Optimization (Edge Functions)

Implement Edge Function orchestration described in `fdd_2.md` §4.4 and AI components in §5.

### Story DISP-036 – Define Routing/Mapping Provider Strategy

**As a** technical lead  
**I want** to select and document a mapping/routing provider and abstraction approach  
**So that** travel time and distance estimates are reliable and swappable.

**Acceptance Criteria**
- [ ] Candidate providers are evaluated (e.g., Google Maps, Mapbox, other) and one is selected or an abstraction is defined.
- [ ] The data needed (geocoded lat/long) is confirmed available via CRM locations.
- [ ] Rate limits, cost considerations, and caching strategy are documented.

**Definition of Done**
- [ ] Provider decision doc exists, including API keys/secret handling in Supabase.
- [ ] Caching approach is documented (even if implemented later).

---

### Story DISP-037 – Auto-Schedule Single Job Edge Function (Propose vs Commit)

**As a** dispatcher  
**I want** an auto-scheduler to propose or commit an assignment for a job  
**So that** scheduling can be accelerated and consistent.

**Acceptance Criteria**
- [ ] Endpoint exists per `fdd_2.md` §4.4.1 (`POST /dispatch/jobs/:id/auto_schedule`) with `mode=propose|commit`.
- [ ] Inputs considered include required skills, zone, time windows/SLA, existing assignments, shifts/time-off, and travel time estimates.
- [ ] Output includes a ranked recommendation and rationale (even minimal in MVP), and in commit mode writes `job_assignments` (and updates `job_time_windows` selection if applicable).

**Definition of Done**
- [ ] The algorithm’s initial heuristic is documented (even if later upgraded).
- [ ] Idempotency expectations are documented (same input should not create duplicates).

---

### Story DISP-038 – Suggest Technician Candidates for a Job (Scored List)

**As a** dispatcher  
**I want** a ranked list of candidate technicians for a job  
**So that** I can make informed manual decisions.

**Acceptance Criteria**
- [ ] An Edge Function exists per `fdd_2.md` §5.2 (`suggest_assignments_for_job`) returning scored candidates with rationales.
- [ ] Scoring inputs include at least skill match, proximity/zone match, and current load.
- [ ] Response format is stable and documented for UI consumption.

**Definition of Done**
- [ ] Example outputs are captured for at least 2 sample jobs.
- [ ] Logs include decision factors without exposing sensitive data.

---

### Story DISP-039 – Optimize Route for One Technician (Day-Level)

**As a** dispatcher  
**I want** to optimize a technician’s route for a given day  
**So that** travel time is minimized and schedules are feasible.

**Acceptance Criteria**
- [ ] Endpoint exists per `fdd_2.md` §4.4.2 (`POST /dispatch/technicians/:id/optimize_route`).
- [ ] Function builds a travel-time cost matrix (via provider) and produces an ordered sequence.
- [ ] The function can operate in **propose** mode (does not change assignment times) and **commit** mode (updates route tables and assignment sequencing/times) or documents the chosen approach.

**Definition of Done**
- [ ] Route output persistence strategy is implemented and documented (writes `route_plans`, `route_stops`, updates assignment fields if committing).
- [ ] Known limitations (e.g., breaks, time windows) are documented.

---

### Story DISP-040 – Bulk Daily Optimization for All Technicians

**As a** dispatcher  
**I want** to optimize routes for all technicians for a day  
**So that** planning can be done efficiently at scale.

**Acceptance Criteria**
- [ ] Endpoint exists per `fdd_2.md` §4.4.3 (`POST /dispatch/days/:date/optimize_all`).
- [ ] The function respects “manual locks” or excludes assignments that should not move (as referenced in `fdd_2.md` §4.4.3).
- [ ] Returns a summary of changes and estimated savings (time/distance).

**Definition of Done**
- [ ] Bulk optimization run is traceable via a run ID.
- [ ] Documentation explains how locks work and how dispatchers override.

---

### Story DISP-041 – Asynchronous Optimization Jobs and Progress Reporting

**As a** dispatcher  
**I want** long-running optimization to run asynchronously with progress indicators  
**So that** the UI stays responsive and operators can track completion.

**Acceptance Criteria**
- [ ] A job/run tracking approach is defined (table or logs) that can store status: queued/running/succeeded/failed with error details.
- [ ] Optimization endpoints can return quickly with a run ID.
- [ ] UI can poll or subscribe for run status.

**Definition of Done**
- [ ] Run tracking schema and API are documented.
- [ ] Failure states are observable and debuggable.

---

## Epic 8 – Emergency / Priority Job Handling

Implement emergency insertion workflow per `fdd_2.md` §4.4.4.

### Story DISP-042 – Insert Emergency Job Edge Function (Propose vs Commit)

**As a** dispatcher  
**I want** to insert an emergency job into existing schedules/routes  
**So that** urgent work is handled quickly while minimizing disruption.

**Acceptance Criteria**
- [ ] Endpoint exists per `fdd_2.md` §4.4.4 (`POST /dispatch/jobs/:id/insert_emergency`).
- [ ] Function evaluates current schedules/routes and proposes an insertion plan considering priority, skill/zone match, travel time, and SLA impact.
- [ ] In commit mode, route and assignment updates are applied and affected customers are queued for reschedule notifications.

**Definition of Done**
- [ ] The insertion strategy (heuristic) is documented, including “bump rules”.
- [ ] Audit trail requirements for changes are captured (who/why/what moved).

---

### Story DISP-043 – Reschedule Notification Generation for Disrupted Jobs

**As a** customer  
**I want** to be notified if my appointment is rescheduled due to emergencies  
**So that** I am not surprised by changes.

**Acceptance Criteria**
- [ ] When emergency insertion commits changes affecting other jobs/assignments, `job_notifications` entries are created for `reschedule_notice`.
- [ ] The notifications include enough context for templates: old vs new time windows, updated ETA if applicable.

**Definition of Done**
- [ ] Notification payload contract is documented for template rendering.
- [ ] Tests confirm at least one disrupted job creates a reschedule notification.

---

## Epic 9 – Calendar Integration (OAuth, Sync, Webhooks)

Implement calendar integration flows per `fdd_2.md` §4.5, including secure token handling and reconciliation behaviors.

### Story DISP-044 – Calendar Connect Endpoint (OAuth Initiation)

**As a** technician or dispatcher  
**I want** to connect my Google/Microsoft calendar  
**So that** assignments appear on my external calendar automatically.

**Acceptance Criteria**
- [ ] Endpoint exists per `fdd_2.md` §4.5.1 (`POST /dispatch/calendar/connect`).
- [ ] OAuth flow initiation and callback handling approach is defined and documented (redirect URLs, state validation, PKCE if used).
- [ ] Successful connection results in a `calendar_integrations` record with provider identity and scoped config.

**Definition of Done**
- [ ] OAuth security considerations are documented (CSRF/state, token storage, least scope).
- [ ] A manual test checklist exists for connecting a calendar in a dev environment.

---

### Story DISP-045 – Sync Internal Appointments to Calendar

**As a** technician  
**I want** my scheduled assignments to sync to my external calendar  
**So that** my day is visible alongside other commitments.

**Acceptance Criteria**
- [ ] Endpoint exists per `fdd_2.md` §4.5.2 (`POST /dispatch/calendar/sync`).
- [ ] Sync reads changed assignments and creates/updates/cancels provider events and `calendar_events` tracking rows.
- [ ] Sync is incremental and idempotent per assignment-provider event mapping.

**Definition of Done**
- [ ] Sync behavior is documented including how updates are detected (timestamps/status).
- [ ] Failures are retriable without duplicating events.

---

### Story DISP-046 – Calendar Provider Webhooks Ingestion

**As a** system  
**I want** to receive external calendar changes via webhooks  
**So that** schedule changes made outside the app can be reconciled.

**Acceptance Criteria**
- [ ] Webhook endpoints exist per `fdd_2.md` §4.5.3 (`POST /webhooks/calendar/google`, `POST /webhooks/calendar/microsoft`).
- [ ] Webhook authenticity verification strategy is defined and documented per provider.
- [ ] External changes map back to internal `calendar_events` and then to assignments/jobs.

**Definition of Done**
- [ ] Webhook verification and replay/duplicate handling are documented.
- [ ] At least one example “event moved” and “event canceled” flow is documented.

---

### Story DISP-047 – Calendar Reconciliation Modes (Apply vs Flag)

**As a** dispatcher/admin  
**I want** a configurable reconciliation strategy for external calendar edits  
**So that** we can decide whether external changes should automatically affect internal schedules.

**Acceptance Criteria**
- [ ] A documented org-level setting exists describing two modes:
  - [ ] Apply external updates to internal schedule (with notifications).
  - [ ] Flag for manual reconciliation (mark as `deleted_by_user` or similar).
- [ ] The chosen mode affects webhook processing behavior.

**Definition of Done**
- [ ] Setting is documented (and implemented in config storage if in scope).
- [ ] An audit note strategy is documented for schedule changes applied due to external edits.

---

## Epic 10 – Notifications & Reminder Orchestration

Implement scheduling notification generation and dispatch per `fdd_2.md` §4.6.

### Story DISP-048 – Schedule Standard Notifications for a Job

**As a** dispatcher/admin  
**I want** standard reminders scheduled for appointments  
**So that** customers and technicians receive timely updates.

**Acceptance Criteria**
- [ ] Endpoint exists per `fdd_2.md` §4.6.1 (`POST /dispatch/jobs/:id/schedule_notifications`).
- [ ] Rules for reminder timing are configurable (document MVP defaults: e.g., 24h and 2h before).
- [ ] Creates `job_notifications` rows for each channel/recipient type required.

**Definition of Done**
- [ ] Reminder rules and default values are documented.
- [ ] Tests confirm notifications are created at correct times relative to assignment.

---

### Story DISP-049 – Notification Processor (Cron) for Pending Sends

**As a** system  
**I want** a cron-based processor to send due notifications  
**So that** reminders are delivered reliably even if the UI is not active.

**Acceptance Criteria**
- [ ] An Edge Function exists per `fdd_2.md` §4.6.2 (`process_job_notifications`) designed to run periodically.
- [ ] Processor:
  - [ ] fetches `pending` notifications due now
  - [ ] resolves templates/content (via shared comms system or documented stub)
  - [ ] calls provider(s) and updates status (`sent`, `failed`) with metadata/errors
- [ ] Idempotency is addressed to avoid double sends.

**Definition of Done**
- [ ] Cron schedule and operational runbook are documented.
- [ ] Failure handling and retry policy are documented.

---

### Story DISP-050 – Notification Template Variables Contract (Scheduling Context)

**As a** communications integrator  
**I want** a stable contract for template variables used in scheduling notifications  
**So that** SMS/email/push templates can be authored consistently.

**Acceptance Criteria**
- [ ] A documented JSON payload shape exists for each `notification_type` including required fields (job title, customer name, address, window start/end, ETA, technician display name).
- [ ] PII handling rules are documented (what is safe to include in each channel).

**Definition of Done**
- [ ] Contract doc is committed and referenced from both scheduling and comms modules.

---

## Epic 11 – Dispatch Console UI (Next.js)

Build the dispatcher-facing UI described in `fdd_2.md` §6.1.

### Story DISP-051 – Dispatch Schedule Board (Timeline View)

**As a** dispatcher  
**I want** a timeline schedule board with drag-and-drop assignments  
**So that** I can quickly plan and adjust the day.

**Acceptance Criteria**
- [ ] A Next.js page/view exists that renders technicians (rows) and time (columns) and shows `job_assignments` as blocks.
- [ ] Drag-and-drop supports:
  - [ ] moving an assignment in time
  - [ ] reassigning to a different technician
- [ ] Drag actions call `PATCH /dispatch/assignments/:id` and reflect updates in UI.
- [ ] Basic states exist: loading, empty day, and error.

**Definition of Done**
- [ ] UI uses real-time updates or refresh to reflect changes.
- [ ] Usability notes documented (snap intervals, collision indicators).

---

### Story DISP-052 – Map-Based Dispatch View

**As a** dispatcher  
**I want** a map view showing jobs, technicians, and routes  
**So that** I can make geographically informed dispatch decisions.

**Acceptance Criteria**
- [ ] Map view displays:
  - [ ] job pins from `dispatch_jobs.location_id` lat/long
  - [ ] technician positions (integration placeholder acceptable; see open question in `fdd_2.md` §9)
  - [ ] route overlays from `route_plans` / `route_stops` when available
- [ ] Filters exist for zone, skill, status, and priority.

**Definition of Done**
- [ ] Map provider and licensing considerations are documented.
- [ ] Clear fallback behavior exists if technician location is not yet implemented.

---

### Story DISP-053 – Job Creation UI + Detail Drawer

**As a** dispatcher  
**I want** to create jobs and view job details in a drawer  
**So that** dispatch work can be done without losing context.

**Acceptance Criteria**
- [ ] UI supports creating a `dispatch_job` including priority, duration, skills, zone, SLA/time windows.
- [ ] Job details drawer shows linked CRM data (customer + location) and existing assignments/time windows.
- [ ] A CTA exists for manual assign and auto-schedule from the job drawer.

**Definition of Done**
- [ ] UI is validated against seeded data.
- [ ] Error handling is clear for missing CRM data or invalid inputs.

---

### Story DISP-054 – Capacity & Utilization View

**As a** dispatcher/manager  
**I want** a capacity view per tech/day  
**So that** I can avoid overscheduling and spot SLA risks.

**Acceptance Criteria**
- [ ] UI shows per technician:
  - [ ] available minutes (from shifts minus time-off)
  - [ ] scheduled minutes (from assignments)
  - [ ] utilization percentage and warnings
- [ ] SLA risk warnings appear for jobs near SLA deadlines.

**Definition of Done**
- [ ] Computation rules are documented (including assumptions).
- [ ] UI matches performance target for typical org sizes.

---

### Story DISP-055 – Optimization Actions UI (“Optimize Route”, “Optimize Day”)

**As a** dispatcher  
**I want** buttons to run optimization workflows  
**So that** route plans can be generated and improved quickly.

**Acceptance Criteria**
- [ ] UI supports triggering:
  - [ ] optimize one technician (`/dispatch/technicians/:id/optimize_route`)
  - [ ] optimize day (`/dispatch/days/:date/optimize_all`)
- [ ] UI shows progress and results summary (distance/time savings) when available.

**Definition of Done**
- [ ] Async run tracking integration (if implemented) is wired into UI.
- [ ] Errors are actionable (e.g., missing coordinates, provider failure).

---

## Epic 12 – Customer Portal and Booking Hooks (Scheduling-Adjacent)

Provide scheduling data surfaces needed for portal flows per `fdd_2.md` §6.2 and §3.3 without implementing the full portal UX.

### Story DISP-056 – Expose Appointment Time Windows for Portal Selection

**As a** customer (portal user)  
**I want** to view and select available time windows  
**So that** I can book an appointment that fits my schedule.

**Acceptance Criteria**
- [ ] A portal-safe API/view exists to list `job_time_windows` for a customer’s job(s).
- [ ] Customer can select a window (updates `is_selected` or equivalent) with authorization restricted to their own job.
- [ ] Selecting a window can trigger scheduling actions (e.g., auto-schedule) or is documented as a manual dispatcher step.

**Definition of Done**
- [ ] Portal-safe access control strategy is implemented or specified (no broad org access).
- [ ] API contract is documented for portal UI developers.

---

### Story DISP-057 – Customer Appointment Status + ETA Read Model

**As a** customer  
**I want** to see appointment status and technician ETA updates  
**So that** I know when to expect arrival.

**Acceptance Criteria**
- [ ] A portal-safe read endpoint returns job status plus assignment arrival window and current `tech_eta_at` where available.
- [ ] Real-time updates are supported or a polling strategy is documented.

**Definition of Done**
- [ ] Response model is documented.
- [ ] Security tests confirm customers cannot access others’ jobs.

---

## Epic 13 – Analytics & KPI Foundations (Scheduling Data Outputs)

Prepare scheduling-derived metrics described in `fdd_2.md` §5.3 for downstream Reporting & Analytics.

### Story DISP-058 – Define Scheduling KPI View Specifications

**As a** data/analytics engineer  
**I want** a documented spec for scheduling KPIs and their inputs  
**So that** reporting can be implemented consistently later.

**Acceptance Criteria**
- [ ] KPIs from `fdd_2.md` §5.3 are defined with formulas and source fields, including:
  - [ ] on-time arrival rate
  - [ ] utilization percentage
  - [ ] average travel time per job/tech
  - [ ] emergency response time
- [ ] Data quality requirements are documented (needed timestamps, statuses, completion signals).

**Definition of Done**
- [ ] KPI spec doc exists and references dispatch schema fields precisely.

---

### Story DISP-059 – Add Auditability Hooks for Schedule Changes

**As a** compliance-minded operator  
**I want** an audit trail of dispatch schedule changes  
**So that** we can trace who moved what and why.

**Acceptance Criteria**
- [ ] Audit approach is defined (table or logs) to capture:
  - [ ] who (user_id)
  - [ ] when
  - [ ] entity (assignment/job/route)
  - [ ] before/after times/tech
  - [ ] reason (optional)
- [ ] Captures changes from manual actions and automation (optimizer/emergency insert) with a run ID correlation when applicable.

**Definition of Done**
- [ ] Audit approach is documented and included in implementation backlog.
- [ ] Security/access rules for audit data are defined.

---

## Epic 14 – Reliability, Performance, and Observability

Meet non-functional requirements in `fdd_2.md` §8 and observability needs in §8.

### Story DISP-060 – Performance Baseline for Schedule Board Queries

**As a** technical lead  
**I want** schedule board queries to be fast and predictable  
**So that** dispatchers can operate without lag.

**Acceptance Criteria**
- [ ] Identify and document key query patterns for schedule board (assignments by day, jobs by status, technician availability).
- [ ] Define targets aligned to `fdd_2.md` §8 (e.g., \< 500ms for typical org sizes).
- [ ] Indexes and query strategies are validated against realistic test data volumes.

**Definition of Done**
- [ ] Performance notes are captured in repo docs.
- [ ] Known bottlenecks and future optimizations are documented.

---

### Story DISP-061 – Idempotency Strategy for Optimization and Notification Processing

**As a** reliability engineer  
**I want** idempotent scheduling automation  
**So that** retries and duplicate triggers do not corrupt schedules or double-send notifications.

**Acceptance Criteria**
- [ ] Idempotency keys/strategy are defined for:
  - [ ] auto-schedule job
  - [ ] optimize routes
  - [ ] emergency insert
  - [ ] notification processing
- [ ] Retry behaviors are documented for external provider failures (routing, calendar, messaging).

**Definition of Done**
- [ ] Idempotency and retry docs are committed.
- [ ] At least one simulated duplicate call scenario is documented with expected outcomes.

---

### Story DISP-062 – Structured Logging and Trace IDs for Dispatch Edge Functions

**As a** DevOps engineer  
**I want** structured logs with trace IDs for automation runs  
**So that** optimization and sync decisions can be debugged in production.

**Acceptance Criteria**
- [ ] All dispatch Edge Functions log:
  - [ ] run ID / trace ID
  - [ ] key inputs (redacted where sensitive)
  - [ ] decisions (selected technician, moved assignments)
  - [ ] errors with actionable context
- [ ] Logs correlate to persisted artifacts (`route_plans`, `job_assignments`, `calendar_events`, `job_notifications`) via IDs.

**Definition of Done**
- [ ] Logging conventions documented and applied consistently.
- [ ] PII redaction rules documented.

---

## Epic 15 – Documentation & API Reference

Ensure the Scheduling & Dispatch module is consumable by engineers and LLM-based task generators.

### Story DISP-063 – Scheduling & Dispatch Developer Documentation

**As a** developer  
**I want** clear module documentation for Scheduling & Dispatch  
**So that** I can implement and extend the module efficiently.

**Acceptance Criteria**
- [ ] Documentation covers:
  - [ ] schema overview and key relationships
  - [ ] endpoints and Edge Functions
  - [ ] RLS policy intent and role matrix
  - [ ] integration points with CRM, work orders, mobile, portal, communications
  - [ ] open questions from `fdd_2.md` §9 and planned resolutions

**Definition of Done**
- [ ] Docs are committed and linked from `docs/overview.md` or a module index.
- [ ] Another reviewer can follow the docs to understand the module without reading `fdd_2.md` end-to-end.

---

### Story DISP-064 – API Reference for Dispatch Endpoints

**As a** frontend engineer or integrator  
**I want** a concise API reference for dispatch endpoints  
**So that** I can consume them without reading backend code.

**Acceptance Criteria**
- [ ] A Markdown API reference exists for all endpoints in `fdd_2.md` §4 including auth requirements, request/response schemas, and examples.
- [ ] Error cases are documented (unauthorized, forbidden, validation errors, conflicts).
- [ ] “Propose vs commit” conventions (where supported) are documented.

**Definition of Done**
- [ ] API reference is version-controlled and linked from module docs.
- [ ] Reference is consistent with the stories in this document.

---

## Epic 16 – Integration Hooks, Change Triggers, and Real-Time Location (Dispatch-Adjacent)

Cover integration and trigger behaviors called out in `fdd_2.md` (§3.3, §4.4, §6.1, §9) that connect Scheduling & Dispatch to other modules and power map-based dispatching.

### Story DISP-065 – Work Order → Dispatch Job Synchronization Strategy

**As a** platform integrator  
**I want** a clear integration strategy between Work Orders and Dispatch Jobs  
**So that** scheduling stays in sync with operational work order creation/updates.

**Acceptance Criteria**
- [ ] A documented decision exists for `fdd_2.md` §9 “Work Order & Job Unification”:
  - [ ] either `dispatch_jobs` is separate and links to work orders via `related_work_order_id`, or
  - [ ] dispatch fields are unified into the work order table, with clear mapping.
- [ ] A documented event contract exists for work order events consumed by scheduling (created, updated, canceled, completed).
- [ ] At least one “happy path” synchronization flow is defined (e.g., work order created → dispatch job created/updated).

**Definition of Done**
- [ ] Integration contract doc is committed and referenced from both scheduling and work order docs.
- [ ] A backlog note exists for any required schema changes based on the unification decision.

---

### Story DISP-066 – Implement Work Order Event Handler Hook (Scheduling Consumer)

**As a** scheduling system  
**I want** to react to work order lifecycle events  
**So that** dispatch jobs are created/updated/canceled consistently without manual duplication.

**Acceptance Criteria**
- [ ] A scheduling-side handler is defined (Edge Function, webhook endpoint, or RPC trigger) that accepts work order event payloads.
- [ ] On relevant events:
  - [ ] creates or updates a `dispatch_jobs` record (if separate model), including priority, location, and constraints where available
  - [ ] cancels or completes dispatch jobs/assignments when work orders are canceled/completed (rules documented)
- [ ] Handler is idempotent with a documented dedupe key.

**Definition of Done**
- [ ] Event payload schema and handler behavior are documented with examples.
- [ ] Security model is documented (trusted caller, signing/secret, service role usage).

---

### Story DISP-067 – System-Suggested Time Windows Generation Rules

**As a** dispatcher or customer (portal user)  
**I want** the system to generate sensible default appointment windows  
**So that** bookings can be offered even when dispatchers don’t manually craft windows.

**Acceptance Criteria**
- [ ] A documented algorithm exists for generating `job_time_windows` when none are provided (e.g., next available business slots within SLA, zone, and capacity constraints).
- [ ] Generated windows are marked `source = system_suggested` and exactly one can be selected at a time per job (rule documented).
- [ ] Edge cases are addressed (SLA too tight, no availability): return “no slots” with reason(s).

**Definition of Done**
- [ ] Window generation rules are committed as documentation and referenced by job creation and portal stories.
- [ ] A test checklist exists covering at least 5 scheduling scenarios (tight SLA, no techs, time-off, zone mismatch, emergency priority).

---

### Story DISP-068 – Re-Optimization Triggers on Schedule/Availability Change

**As a** dispatcher  
**I want** the system to detect changes that invalidate schedules and prompt re-optimization  
**So that** routes stay feasible when jobs, assignments, or technician availability changes.

**Acceptance Criteria**
- [ ] A documented list of trigger events exists (e.g., assignment moved, technician time-off added, emergency inserted, job canceled).
- [ ] For each trigger, the system either:
  - [ ] automatically queues a re-optimization run, or
  - [ ] flags affected routes/days as “needs review” (with a clear UI indicator)
- [ ] Triggers avoid “thrash” (debounce/coalesce strategy documented).

**Definition of Done**
- [ ] Trigger strategy is documented (including debounce/coalesce rules).
- [ ] A runbook exists for dispatchers describing what happens after each change type.

---

### Story DISP-069 – SLA and Time-Window Conflict Detection (Warnings/Blocks)

**As a** dispatcher  
**I want** the system to warn or block scheduling that violates SLA/time-window constraints  
**So that** we minimize late arrivals and missed compliance windows.

**Acceptance Criteria**
- [ ] Assignment create/update validates constraints:
  - [ ] respects selected time window (if any)
  - [ ] respects SLA earliest start and latest completion fields
- [ ] A documented policy exists for: warn vs block (MVP acceptable: warn for soft constraints, block for hard constraints).
- [ ] UI displays SLA risk warnings in schedule board/capacity view.

**Definition of Done**
- [ ] Constraint policy is documented and referenced by assignment and optimization stories.
- [ ] At least one example scenario is documented showing a blocked change and a warned change.

---

### Story DISP-070 – Technician Location Ingestion and Storage for Map Dispatch

**As a** dispatcher  
**I want** near-real-time technician locations in the system  
**So that** the map-based dispatch board can show current positions and improve routing decisions.

**Acceptance Criteria**
- [ ] A documented approach exists for location ingestion (mobile app sends GPS updates) including:
  - [ ] update frequency guidelines
  - [ ] privacy constraints and retention
  - [ ] battery/network considerations
- [ ] A storage model is defined (e.g., `technician_locations` table with latest point per technician and optional history).
- [ ] Access rules ensure only dispatchers/admins can view all tech locations; technicians can view their own (as needed).

**Definition of Done**
- [ ] Location ingestion + storage design is committed and referenced by the map UI story (DISP-052).
- [ ] Security and privacy considerations are reviewed and documented.

---

### Story DISP-071 – Calendar Token Refresh and Expiration Handling

**As a** system  
**I want** calendar integrations to handle token expiry safely  
**So that** sync remains reliable over time without exposing secrets.

**Acceptance Criteria**
- [ ] A documented strategy exists for:
  - [ ] storing refresh tokens securely
  - [ ] refreshing access tokens before expiry
  - [ ] handling revoked/invalid tokens (mark integration inactive, notify user)
- [ ] Sync functions detect auth failures and update integration state accordingly.

**Definition of Done**
- [ ] Token refresh strategy is documented and referenced by calendar connect/sync stories.
- [ ] A troubleshooting guide exists for common calendar integration failures.

---

These epics and stories collectively provide comprehensive coverage for implementing **Scheduling & Dispatch (`fdd_2.md`)**, aligned with platform goals in `functional.md`, technical choices in `tooling.md`, and the repo documentation structure in `overview.md`.


