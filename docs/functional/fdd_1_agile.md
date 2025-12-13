## CRM Module – Agile User Stories (`fdd_1_agile.md`)

This document decomposes the **CRM module** defined in `fdd_1.md` (and aligned with `functional.md`, `tooling.md`, and `overview.md`) into epics and user stories suitable for implementation on **Supabase (Postgres, Auth, Storage, Edge Functions)** with a **Next.js frontend on Vercel**.

Each story includes:
- **User story** (role / intent / value)
- **Acceptance Criteria**
- **Definition of Done (DoD)**

Story IDs are for reference (not prescriptive for tooling).

---

## Epic 1 – CRM Core Data Model & Supabase Schema

Implement the CRM data model in Supabase Postgres, including multi-tenancy, core entities, and supporting tables as described in `fdd_1.md` §3.

### Story CRM-001 – Define CRM Multi-Tenancy & Org Model

**As a** platform administrator  
**I want** a clear organizational/tenant model for CRM data  
**So that** all CRM records are scoped correctly per company and enforce data isolation.

**Acceptance Criteria**
- [ ] An `orgs` (or equivalent) table exists (or clear decision is documented to reuse a global org/account model) with at minimum: `id`, `name`, `created_at`.
- [ ] All CRM tables defined in `fdd_1.md` that require tenancy include an `org_id` column referencing `orgs.id`.
- [ ] A convention is documented for how `org_id` is derived from the authenticated user (e.g., via `profiles` table or JWT claims).
- [ ] Example queries demonstrate that CRM data from different orgs cannot be mixed inadvertently (e.g., joins must always be org-scoped).

**Definition of Done**
- [ ] DDL for `orgs` (if new) is created, migrated, and version-controlled.
- [ ] All CRM table definitions clearly include `org_id` where required.
- [ ] Documentation exists in the CRM design/README describing tenancy assumptions and how `org_id` is set and used.
- [ ] Reviewed and approved by at least one engineer or architect for alignment with broader platform tenancy strategy.

---

### Story CRM-002 – Implement `customers` Table

**As a** CSR or sales user  
**I want** a `customers` table that represents individuals and organizations  
**So that** I can manage core customer records in a centralized database.

**Acceptance Criteria**
- [ ] A `customers` table exists with columns at least matching `fdd_1.md` §3.2.2 (id, org_id, external_ref, type, name, first_name, last_name, company_name, primary_location_id, primary_contact_id, email, phone, status, lifecycle_stage, source, preferred_language, notes, created_at, updated_at).
- [ ] Appropriate enums are defined for `type`, `status`, and `lifecycle_stage` according to the design.
- [ ] Indexes are created for `org_id` and common search fields (e.g., name, email, phone) as specified.
- [ ] Constraints ensure referential integrity for `primary_location_id` and `primary_contact_id` (nullable but, if present, must reference valid rows).
- [ ] Creating, updating, and deleting a customer record via SQL scripts or Supabase console works as expected without violating constraints.

**Definition of Done**
- [ ] DDL is implemented and stored in migrations or schema files under version control.
- [ ] Basic seed or test data for `customers` is created for at least one sample org.
- [ ] Schema is visible and verified in Supabase dashboard.
- [ ] Schema decisions that diverge from `fdd_1.md` (if any) are documented with rationale.

---

### Story CRM-003 – Implement `customer_locations` Table

**As a** dispatcher or technician  
**I want** to store multiple billing and service locations per customer  
**So that** jobs can be scheduled and routed to the correct physical address.

**Acceptance Criteria**
- [ ] A `customer_locations` table exists with columns at least matching `fdd_1.md` §3.2.3 (id, org_id, customer_id, label, type, address_line1, address_line2, city, state, postal_code, country, latitude, longitude, is_primary, created_at, updated_at).
- [ ] `customer_id` is a required FK to `customers.id` with cascade on delete.
- [ ] `type` enum is implemented (`billing`, `service`, `both`).
- [ ] `is_primary` is enforced so that at most one primary location per customer and org can be set for a given `type` (documented or constrained as appropriate).
- [ ] Indexes exist for `customer_id` and optionally (`latitude`, `longitude`) if required for routing later.

**Definition of Done**
- [ ] DDL for `customer_locations` is implemented, migrated, and version-controlled.
- [ ] At least one customer in test data has multiple locations with different `type` values.
- [ ] Verified that deleting a customer cascades to its locations.
- [ ] Any decisions about geo-indexing strategy are documented.

---

### Story CRM-004 – Implement `customer_contacts` Table

**As a** CSR  
**I want** to store multiple contact channels per customer (email, phone, messaging)  
**So that** I can reach customers through their preferred communication methods.

**Acceptance Criteria**
- [ ] A `customer_contacts` table exists with fields matching `fdd_1.md` §3.2.4 (id, org_id, customer_id, type, value, is_primary, is_verified, opt_in_marketing, opt_in_transactional, preferred_channel, notes, created_at, updated_at).
- [ ] The `type` enum includes at least: `email`, `mobile`, `phone`, `fax`, `whatsapp`, `telegram`, `portal`.
- [ ] Indexes exist on `customer_id` and any unique constraints needed (e.g., per-org unique email if desired).
- [ ] Business rules on `is_primary` and `preferred_channel` are defined (e.g., at most one primary contact per type per customer) and implemented or documented.
- [ ] Sample contact data exists for test customers demonstrating multiple types and opt-in flags.

**Definition of Done**
- [ ] DDL is implemented, migrated, and version-controlled.
- [ ] Sample queries (or Supabase UI checks) confirm that contacts link correctly to customers and orgs.
- [ ] Any partial unique index decisions are captured in documentation.

---

### Story CRM-005 – Implement `crm_preferences` Table

**As a** compliance-conscious operator  
**I want** customer-level communication preferences and do-not-contact flags  
**So that** we respect opt-outs and legal requirements for outreach.

**Acceptance Criteria**
- [ ] A `crm_preferences` table exists with fields matching `fdd_1.md` §3.2.5 (id, org_id, customer_id, do_not_contact, do_not_email, do_not_sms, do_not_call, preferred_contact_window_start, preferred_contact_window_end, notes, created_at, updated_at).
- [ ] `customer_id` is unique in `crm_preferences` (one row per customer), enforced via a unique constraint.
- [ ] For at least one test customer, preferences are set such that some channels are disabled.
- [ ] Queries demonstrate that joining `customers` with `crm_preferences` yields correct per-customer preference data.

**Definition of Done**
- [ ] DDL is created, migrated, and version-controlled.
- [ ] The intended interpretation of each flag (e.g., `do_not_contact` overrides channel-specific flags) is documented.
- [ ] Sample data illustrates edge cases (e.g., do-not-contact vs more granular flags).

---

### Story CRM-006 – Implement `crm_interactions` Table

**As a** CSR or manager  
**I want** a detailed interaction log for each customer  
**So that** I can see complete communication history across channels.

**Acceptance Criteria**
- [ ] A `crm_interactions` table exists with fields matching `fdd_1.md` §3.2.6 (id, org_id, customer_id, location_id, related_work_order_id, related_quote_id, channel, direction, subject, summary, body, metadata, sentiment, created_by_user_id, occurred_at, created_at).
- [ ] Enums for `channel` and `direction` are defined as specified.
- [ ] FKs exist to `customers`, `customer_locations`, and future work-order/quote tables (placeholders allowed if other modules not implemented yet, with clear TODOs).
- [ ] Indexes are created on `customer_id` and (`org_id`, `occurred_at`) as specified.
- [ ] Test data includes at least one interaction for multiple channels and directions per sample customer.

**Definition of Done**
- [ ] DDL is implemented, migrated, and version-controlled.
- [ ] Verified in Supabase that interaction records link to customers and, where present, locations.
- [ ] Any temporary FK strategies (e.g., nullable references until other modules exist) are documented.

---

### Story CRM-007 – Implement `crm_followups` Table

**As a** CSR or sales rep  
**I want** scheduled follow-ups tied to customers  
**So that** I don’t miss important reminders and outreach tasks.

**Acceptance Criteria**
- [ ] A `crm_followups` table exists with fields matching `fdd_1.md` §3.2.7 (id, org_id, customer_id, assigned_to_user_id, title, description, due_at, status, priority, origin, related_interaction_id, related_work_order_id, created_by_user_id, completed_at, completion_notes, created_at, updated_at).
- [ ] Enums for `status`, `priority`, and `origin` are defined as specified.
- [ ] `due_at` is required and stored as timestamptz.
- [ ] Indexes exist on (`org_id`, `due_at`) and (`assigned_to_user_id`, `due_at`).
- [ ] Sample data includes follow-ups in different statuses, priorities, and origins.

**Definition of Done**
- [ ] DDL is implemented, migrated, and version-controlled.
- [ ] Verified via queries that filtering by assignee and due date returns expected follow-ups.
- [ ] Retention or archival strategy for old completed follow-ups is at least documented (even if not yet implemented).

---

### Story CRM-008 – Implement `crm_tags` and `crm_customer_tags` Tables

**As a** CRM power user  
**I want** flexible tags on customers  
**So that** I can group and filter customers by arbitrary labels (e.g., VIP, warranty).

**Acceptance Criteria**
- [ ] `crm_tags` table exists with fields matching `fdd_1.md` §3.2.8 (id, org_id, name, description, color, created_at).
- [ ] `crm_customer_tags` table exists with fields matching §3.2.8 (id, org_id, customer_id, tag_id, assigned_by_user_id, created_at).
- [ ] A unique constraint exists on (`org_id`, `name`) for `crm_tags`.
- [ ] A unique constraint exists on (`org_id`, `customer_id`, `tag_id`) for `crm_customer_tags`.
- [ ] Sample data includes multiple tags and customers tagged with them.

**Definition of Done**
- [ ] DDL for both tables is implemented, migrated, and version-controlled.
- [ ] Queries demonstrate retrieval of tags per customer and customers per tag.
- [ ] Any color format conventions (e.g., hex) are documented.

---

### Story CRM-009 – Implement `crm_segments` and `crm_segment_members` Tables

**As a** marketing or operations manager  
**I want** to define and persist customer segments  
**So that** I can target groups of customers for campaigns and analysis.

**Acceptance Criteria**
- [ ] `crm_segments` table exists with fields matching `fdd_1.md` §3.2.9 (id, org_id, name, description, type, definition, ai_prompt, ai_explanation, is_active, last_computed_at, created_by_user_id, created_at, updated_at).
- [ ] `crm_segment_members` table exists with fields matching §3.2.10 (id, org_id, segment_id, customer_id, score, metadata, created_at).
- [ ] Enums for segment `type` include `static`, `rule_based`, and `ai_generated`.
- [ ] Unique constraint exists on (`org_id`, `segment_id`, `customer_id`) for `crm_segment_members`.
- [ ] At least one sample static and one rule-based segment is represented in test data, with corresponding members.

**Definition of Done**
- [ ] DDL for both tables is implemented, migrated, and version-controlled.
- [ ] Sample queries show how to retrieve all members of a segment and all segments for a customer.
- [ ] Any schema considerations for storing `definition` and `metadata` JSON are documented (e.g., allowed shape, versioning).

---

### Story CRM-010 – Implement `crm_message_templates` Table

**As a** CRM/marketing admin  
**I want** templated messages for follow-ups and campaigns  
**So that** communications are consistent and efficient.

**Acceptance Criteria**
- [ ] `crm_message_templates` table exists with fields matching `fdd_1.md` §3.2.11 (id, org_id, name, channel, subject, body, variables, is_system, created_by_user_id, created_at, updated_at).
- [ ] Enum for `channel` includes `email`, `sms`, `phone_script`, and `portal_message`.
- [ ] `body` supports placeholder variables (e.g., `{{customer.first_name}}`) with at least a documented convention.
- [ ] At least one system template (`is_system = true`) and one org-defined template exist in test data.

**Definition of Done**
- [ ] DDL is implemented, migrated, and version-controlled.
- [ ] Documentation explains supported variable syntax and validation expectations (even if not fully implemented).
- [ ] Example queries show how templates can be retrieved by channel and org.

---

### Story CRM-011 – Implement `crm_automation_rules` and `crm_automation_runs` Tables

**As a** CRM administrator  
**I want** to define and track automation rules for CRM workflows  
**So that** follow-ups and other actions can happen automatically.

**Acceptance Criteria**
- [ ] `crm_automation_rules` table exists with fields matching `fdd_1.md` §3.2.12 (id, org_id, name, description, is_enabled, trigger_type, event_type, time_offset_minutes, segment_id, conditions, actions, created_by_user_id, created_at, updated_at).
- [ ] Enum for `trigger_type` includes `event`, `time_based`, `segment_membership`.
- [ ] Validation rules are documented (and enforced where possible) for field combinations (e.g., `event_type` required when `trigger_type = event`).
- [ ] `crm_automation_runs` table exists with fields matching §3.2.13 (id, org_id, rule_id, customer_id, trigger_context, status, error_message, started_at, completed_at), with index on (`rule_id`, `started_at`).
- [ ] Sample rules and runs exist in test data to support downstream development and UI.

**Definition of Done**
- [ ] DDL for both tables is implemented, migrated, and version-controlled.
- [ ] Example JSON shape for `actions` and `conditions` is documented.
- [ ] Verified via queries that runs can be filtered by rule, org, status, and date ranges.

---

### Story CRM-012 – Define Indexing & Performance Strategy for CRM Tables

**As a** backend engineer  
**I want** appropriate indexes and performance considerations for CRM tables  
**So that** search, listing, and reporting are responsive for typical dataset sizes.

**Acceptance Criteria**
- [ ] For each CRM table, required indexes from `fdd_1.md` are implemented (e.g., search fields, foreign keys, time-based filters).
- [ ] At least one performance target is documented (e.g., \< 500 ms for common list/search operations up to 50k customers per org).
- [ ] Example explain/analyze output is captured for a representative query (e.g., customer search, interactions by date).
- [ ] Any deferred or optional indexes (e.g., geo indexes) are explicitly listed with rationale.

**Definition of Done**
- [ ] Index DDLs are included in migrations and verified in Supabase.
- [ ] Test data of realistic scale (or scaled-down approximation) is used to validate at least one key query.
- [ ] Performance considerations and tradeoffs are recorded in technical notes or README.

---

## Epic 2 – Authentication, Authorization & RLS Policies

Ensure that CRM data is securely scoped per organization and role, aligned with Supabase Auth and RLS as described in `fdd_1.md` §2.3 and §7.

### Story CRM-013 – Define CRM User Roles & Profile Schema

**As a** system administrator  
**I want** a clear role model for CRM users  
**So that** permissions can be enforced consistently across all CRM features.

**Acceptance Criteria**
- [ ] Roles are defined and documented for at least: admin, manager, dispatcher, technician, sales/CSR.
- [ ] A `profiles` (or equivalent) table exists linking Supabase `auth.users` to `org_id` and `role`.
- [ ] Sample users for each role exist in a non-production environment for testing.
- [ ] Mapping from user JWT to `org_id` and `role` is understood and documented for backend and frontend consumers.

**Definition of Done**
- [ ] DDL for `profiles` (if not already present) is implemented and migrated.
- [ ] Documentation in CRM README describes roles and their intended permissions.
- [ ] Roles are verified using Supabase’s auth and row-level security testing where possible.

---

### Story CRM-014 – Implement RLS Base Policies for Multi-Tenancy

**As a** security-conscious architect  
**I want** RLS policies that enforce org scoping  
**So that** users can only see and modify CRM data for their organization.

**Acceptance Criteria**
- [ ] For each CRM table with `org_id`, RLS is enabled.
- [ ] Base policies ensure that:
  - [ ] Select/insert/update/delete operations are restricted to rows where `org_id` matches the user’s `org_id` (from `profiles` or claims).
- [ ] Policies are tested using at least two different orgs to confirm isolation.
- [ ] Where applicable, system/Edge Function service roles are allowed to bypass RLS using secure patterns (e.g., service key, not exposed to client).

**Definition of Done**
- [ ] RLS policies are defined, version-controlled, and deployed to Supabase.
- [ ] Manual or automated tests verify positive and negative cases (e.g., cross-org access is blocked).
- [ ] Any intentional exceptions (e.g., global admin access) are explicitly documented.

---

### Story CRM-015 – Implement Role-Based RLS Policies for CRM Tables

**As a** product owner  
**I want** role-based access control over CRM operations  
**So that** different user types have appropriate permissions (e.g., techs read-only).

**Acceptance Criteria**
- [ ] For each CRM table, role-based policies are defined consistent with `fdd_1.md` §7.1 guidelines (e.g., only certain roles can modify automation rules, segments, or preferences).
- [ ] Technicians have read-only access to a limited subset of customer data as defined by the product requirements (e.g., no marketing preferences edits).
- [ ] CSRs/sales roles can fully manage core CRM records (customers, contacts, follow-ups, tags) within their org.
- [ ] Attempted unauthorized operations by users of insufficient roles are blocked by RLS and loggable.

**Definition of Done**
- [ ] RLS policies for role-based access are implemented and version-controlled.
- [ ] Test cases exist for each role covering allowed and disallowed operations.
- [ ] Policy comments or documentation explain the rationale for each table’s role rules.

---

## Epic 3 – Customer Management APIs & Workflows

Expose customer-centric operations through Supabase RPC functions and/or Edge Functions as outlined in `fdd_1.md` §4.2.

### Story CRM-016 – Create Customer (Edge Function or RPC)

**As a** CSR  
**I want** to create a new customer with primary location and contact in a single action  
**So that** onboarding a new customer is fast and consistent.

**Acceptance Criteria**
- [ ] An authenticated API endpoint (Edge Function `POST /crm/customers` or equivalent RPC) accepts input matching §4.2.1 (type, name, optional first/last, company_name, primary contact, primary location, initial tags).
- [ ] Endpoint inserts a row into `customers`, with associated `customer_locations`, `customer_contacts`, and default `crm_preferences`.
- [ ] `primary_location_id` and `primary_contact_id` are correctly set on the `customers` row.
- [ ] Initial tags are created/linked via `crm_tags` and `crm_customer_tags` where necessary.
- [ ] Errors for invalid input (e.g., missing required fields) return useful messages and do not leave partial data.

**Definition of Done**
- [ ] Endpoint is implemented, authenticated, and respects RLS/role rules.
- [ ] Happy path and basic error path tests exist (manual or automated).
- [ ] Example request/response payloads are documented for frontend use.

---

### Story CRM-017 – Update Customer & Nested Data

**As a** CSR  
**I want** to update customer fields, locations, and contacts  
**So that** customer records remain accurate over time.

**Acceptance Criteria**
- [ ] An authenticated API endpoint (e.g., `PATCH /crm/customers/:id`) supports partial updates to customer fields.
- [ ] Nested updates for `customer_locations` and `customer_contacts` are supported per design (add/update/delete) or, if not implemented yet, limitations are clearly documented.
- [ ] Updating key customer details can optionally log a `crm_interactions` record of type `note` when configured.
- [ ] Attempting to update a customer from a different org is rejected by RLS.

**Definition of Done**
- [ ] Endpoint is implemented and aligned with the data model and RLS.
- [ ] Test cases cover updates to core fields, contacts, and locations, including invalid attempts.
- [ ] API documentation includes examples of partial update payloads.

---

### Story CRM-018 – Get Customer Details View API

**As a** CSR or manager  
**I want** a single endpoint to fetch a full customer profile  
**So that** I can see customer info, preferences, recent interactions, and follow-ups in one place.

**Acceptance Criteria**
- [ ] An authenticated endpoint (e.g., `GET /crm/customers/:id`) returns a composed view including:
  - [ ] Core customer fields.
  - [ ] Locations.
  - [ ] Contacts.
  - [ ] Preferences.
  - [ ] Recent interactions (paged or limited).
  - [ ] Upcoming follow-ups.
- [ ] Pagination/limit parameters are supported for interactions and follow-ups.
- [ ] Response format is versioned or structured in a way that future fields can be added without breaking clients.

**Definition of Done**
- [ ] Endpoint is implemented, secured, and documented.
- [ ] Sample responses for at least two test customers are captured for UI development.
- [ ] Performance of the composed query is validated with indexes in place.

---

### Story CRM-019 – Search & List Customers API

**As a** CSR  
**I want** to search and filter the customer list  
**So that** I can quickly find the right customer when handling calls or tasks.

**Acceptance Criteria**
- [ ] An endpoint (e.g., `GET /crm/customers`) supports:
  - [ ] A `q` parameter for searching by name, email, phone, and optionally address.
  - [ ] Filters for `status`, `lifecycle_stage`, `tag`, and `segment_id`.
  - [ ] Pagination (limit/offset or cursor-based).
- [ ] Results are scoped to the user’s org via RLS.
- [ ] Sorting options (e.g., by name, created_at) are supported and documented.

**Definition of Done**
- [ ] Endpoint implementation is complete, authenticated, and RLS-compliant.
- [ ] Performance is validated on a sample dataset for typical query patterns.
- [ ] Example query strings and responses are documented for frontend integration.

---

## Epic 4 – Interaction & Communication Logging

Enable logging and retrieval of communication history as per `fdd_1.md` §4.3.

### Story CRM-020 – Log Interaction API

**As a** CSR or system integration  
**I want** an API to log communications and notes as interactions  
**So that** the full history with each customer is captured.

**Acceptance Criteria**
- [ ] An authenticated endpoint (e.g., `POST /crm/interactions`) accepts input matching §4.3.1 (customer_id, channel, direction, subject, summary, body, metadata, occurred_at, related_work_order_id, location_id).
- [ ] Validations ensure `customer_id` belongs to the caller’s org and channel/direction are valid enum values.
- [ ] Records are inserted into `crm_interactions` with timestamps and link to `created_by_user_id` where applicable.
- [ ] Service/system-generated interactions can be created without a user ID when appropriate, using a documented pattern.

**Definition of Done**
- [ ] Endpoint is implemented, tested for typical and error cases, and respects RLS.
- [ ] Example usages are documented for both manual UI logging and external integrations (e.g., email/SMS webhooks).

---

### Story CRM-021 – Fetch Interaction History API

**As a** CSR or manager  
**I want** to view a timeline of interactions for a customer  
**So that** I can understand context before engaging with them.

**Acceptance Criteria**
- [ ] An endpoint (e.g., `GET /crm/customers/:id/interactions`) supports:
  - [ ] Pagination.
  - [ ] Optional filters by `channel`, date range, and `sentiment`.
- [ ] Results are ordered by `occurred_at` descending by default.
- [ ] RLS ensures only interactions from the user’s org are returned, and only for accessible customers.

**Definition of Done**
- [ ] Endpoint is implemented and documented.
- [ ] Test data demonstrates multiple channels and sentiments in the timeline.
- [ ] Performance and pagination behavior is validated.

---

## Epic 5 – Follow-Ups & Reminders

Support manual creation, listing, and completion of follow-ups as per `fdd_1.md` §4.4.

### Story CRM-022 – Create Manual Follow-Up API

**As a** CSR or salesperson  
**I want** to create a follow-up task for a customer  
**So that** I remember to contact them at the right time.

**Acceptance Criteria**
- [ ] An authenticated endpoint (e.g., `POST /crm/followups`) accepts input matching §4.4.1 (customer_id, title, description, due_at, priority, optional assigned_to_user_id, origin=manual).
- [ ] A new row is inserted into `crm_followups` with `status = pending` and correct `origin`.
- [ ] Confirms that `assigned_to_user_id` (if provided) belongs to the same org and exists in the profiles/users table.
- [ ] Returns the created follow-up, including generated ID and timestamps.

**Definition of Done**
- [ ] Endpoint is implemented, authenticated, and tested for valid and invalid scenarios.
- [ ] Documented request/response shapes for frontend usage.

---

### Story CRM-023 – List & Filter Follow-Ups API

**As a** CSR or manager  
**I want** a dashboard-friendly API for upcoming follow-ups  
**So that** I can manage workloads and priorities.

**Acceptance Criteria**
- [ ] An endpoint (e.g., `GET /crm/followups`) supports filters:
  - [ ] `assigned_to_user_id`.
  - [ ] Date range (e.g., due between X and Y).
  - [ ] `status` (pending, completed, canceled, expired).
  - [ ] Pagination.
- [ ] Responses include enough info to render a follow-up card (e.g., title, due_at, priority, customer name).
- [ ] RLS ensures users only see follow-ups within their org; optional role logic (e.g., techs only see their own follow-ups) is respected and documented.

**Definition of Done**
- [ ] Endpoint is implemented and secured.
- [ ] Example filters and queries are documented.
- [ ] At least one test scenario validates filter combinations and pagination.

---

### Story CRM-024 – Complete/Cancel Follow-Up API

**As a** CSR or assignee  
**I want** to mark follow-ups as done or canceled  
**So that** my dashboard reflects current work.

**Acceptance Criteria**
- [ ] An endpoint (e.g., `PATCH /crm/followups/:id`) allows changing `status` and adding `completion_notes` when closing a follow-up.
- [ ] When status transitions to `completed`, `completed_at` is set automatically.
- [ ] Status transitions are validated (e.g., cannot move from `canceled` back to `pending` without explicit admin rules).
- [ ] RLS and role checks ensure only authorized users can modify a follow-up.

**Definition of Done**
- [ ] Endpoint is implemented and tested for valid and invalid status changes.
- [ ] Business rules about who can modify which follow-ups (e.g., assignee vs manager) are documented.

---

## Epic 6 – Segmentation & Targeting

Implement management of segments and membership as per `fdd_1.md` §4.5 (rule-based and AI-generated).

### Story CRM-025 – Create & Update Segments API

**As a** CRM/marketing manager  
**I want** to create and update customer segments  
**So that** I can save target groups for ongoing use.

**Acceptance Criteria**
- [ ] Authenticated endpoints (e.g., `POST /crm/segments` and `PATCH /crm/segments/:id`) allow creation and modification of segments, including `name`, `description`, `type`, and `definition` or `ai_prompt`.
- [ ] For `rule_based` segments, the API validates that `definition` is syntactically valid according to a documented schema.
- [ ] For `ai_generated` segments, creating or updating a segment triggers an asynchronous AI job (or schedules one) to compute members.
- [ ] Only users with appropriate roles (e.g., manager/admin) can create or update segments.

**Definition of Done**
- [ ] Endpoints are implemented, authenticated, and protected by role-based RLS.
- [ ] The expected JSON shape for `definition` and `ai_prompt` is documented.
- [ ] Test cases cover creating each segment type and handling invalid definitions.

---

### Story CRM-026 – Compute Members for Rule-Based Segments

**As a** backend engineer  
**I want** to compute and persist members of rule-based segments  
**So that** they can be reused efficiently for campaigns and dashboards.

**Acceptance Criteria**
- [ ] A backend function (SQL or Edge Function) exists to compute `crm_segment_members` for `rule_based` segments based on their `definition`.
- [ ] Computing a segment is idempotent: re-running for the same segment replaces or updates membership as expected without duplicates.
- [ ] The function respects `org_id` and RLS constraints, never leaking data across orgs.
- [ ] At least one example `rule_based` segment definition is implemented and verified via tests or queries.

**Definition of Done**
- [ ] Computation logic is implemented, version-controlled, and documented.
- [ ] Test data demonstrates correct membership for at least one non-trivial rule.
- [ ] Performance is acceptable for expected customer volumes (document any limits).

---

### Story CRM-027 – Get Segment Members API

**As a** CRM user  
**I want** an API to list members of a segment  
**So that** I can review and act on the customers in that group.

**Acceptance Criteria**
- [ ] An authenticated endpoint (e.g., `GET /crm/segments/:id/members`) returns:
  - [ ] Member `customer_id` and core customer summary fields (name, status, lifecycle_stage, tags).
  - [ ] Optional `score` and `metadata` fields for AI segments.
- [ ] Supports pagination and optional filters (e.g., min score for AI segments if relevant).
- [ ] RLS ensures only segments and members from the caller’s org are visible.

**Definition of Done**
- [ ] Endpoint is implemented, tested with multiple segment types, and documented.
- [ ] Example responses are shared with frontend for UI integration.

---

### Story CRM-028 – Recompute AI or Rule-Based Segment Members

**As a** CRM admin  
**I want** to manually trigger segment recomputation  
**So that** segments stay up to date after data changes.

**Acceptance Criteria**
- [ ] An endpoint (e.g., `POST /crm/segments/:id/recompute`) invokes:
  - [ ] Rule-based membership recomputation logic for `rule_based` segments.
  - [ ] AI-based segmentation logic for `ai_generated` segments.
- [ ] Status or progress of recomputation is available (e.g., via logs or optional tracking table/fields like `last_computed_at`).
- [ ] Appropriate role checks ensure only authorized users can trigger recomputation.

**Definition of Done**
- [ ] Endpoint and underlying recomputation flows are implemented and documented.
- [ ] Test cases cover recomputing segments with existing members and verifying updated membership.

---

## Epic 7 – Automation Engine

Enable automation rules and execution as per `fdd_1.md` §4.6.

### Story CRM-029 – Manage Automation Rules API

**As a** CRM admin  
**I want** CRUD APIs for automation rules  
**So that** I can configure automated follow-ups and actions.

**Acceptance Criteria**
- [ ] Authenticated endpoints exist for:
  - [ ] Creating `crm_automation_rules` (`POST /crm/automation/rules`).
  - [ ] Updating rules (`PATCH /crm/automation/rules/:id`).
  - [ ] Listing rules (`GET /crm/automation/rules`).
- [ ] Input validation enforces required fields based on `trigger_type` (event, time_based, segment_membership) as per §4.6.1.
- [ ] Only appropriate roles (e.g., admin/manager) can manage rules.
- [ ] Rules can be enabled/disabled via `is_enabled`.

**Definition of Done**
- [ ] Endpoints are implemented, authenticated, and protected by RLS/roles.
- [ ] Example rule JSON, including `actions` and `conditions`, is documented.
- [ ] Tests validate rule creation/update failures for invalid combinations.

---

### Story CRM-030 – Event-Triggered Automation Handler

**As a** CRM system  
**I want** to execute automation rules when relevant events occur  
**So that** follow-ups and other actions happen without manual intervention.

**Acceptance Criteria**
- [ ] An Edge Function (e.g., `handle_event_trigger`) exists and can be called by other modules (e.g., work orders) with an event payload (e.g., `work_order_completed`, `quote_sent`).
- [ ] Function looks up active `crm_automation_rules` with `trigger_type = event` and matching `event_type`.
- [ ] For each applicable rule, the function evaluates `conditions` (if any) and creates corresponding actions (e.g., `crm_followups`, messages, tags) and a `crm_automation_runs` record.
- [ ] Error scenarios (e.g., invalid rule config) are logged and stored in `crm_automation_runs.error_message`.

**Definition of Done**
- [ ] Event handler function is implemented, deployed, and secured (e.g., only callable from trusted sources).
- [ ] At least one end-to-end test (manual or automated) shows a simulated event resulting in a follow-up and a logged run.
- [ ] Documentation explains expected event payload schema and integration points for other modules.

---

### Story CRM-031 – Time-Based Automation Processor

**As a** CRM system  
**I want** a scheduled job to process time-based automation rules  
**So that** follow-ups and actions occur at scheduled offsets after events.

**Acceptance Criteria**
- [ ] An Edge Function (e.g., `process_time_based_automations`) runs on a cron schedule (e.g., every 5 minutes) in Supabase.
- [ ] Function finds applicable `crm_automation_rules` with `trigger_type = time_based` and evaluates which customers/events are due based on `time_offset_minutes` and stored context.
- [ ] For each due rule, the function executes actions and logs `crm_automation_runs` entries with status.
- [ ] Idempotency is addressed so that the same time window is not processed multiple times for the same customer/rule combination.

**Definition of Done**
- [ ] Function and schedule are configured, deployed, and documented.
- [ ] At least one test scenario demonstrates a rule becoming due and actions executing correctly.
- [ ] Idempotency strategy is clearly documented and tested.

---

### Story CRM-032 – Segment Membership Automation Processor

**As a** CRM system  
**I want** to trigger actions based on segment membership  
**So that** customers in certain segments receive automated attention.

**Acceptance Criteria**
- [ ] An Edge Function (e.g., `process_segment_automations`) periodically evaluates `crm_automation_rules` with `trigger_type = segment_membership`.
- [ ] For each rule, function identifies customers in the target segment(s) and, based on conditions, performs configured actions (e.g., create follow-ups, send messages, tag customers).
- [ ] Runs are logged in `crm_automation_runs`, including status and any errors.
- [ ] Safeguards exist to avoid over-triggering actions (e.g., configurable limits per run or per customer).

**Definition of Done**
- [ ] Logic is implemented and deployed, with clear documentation on scheduling and limits.
- [ ] Test scenarios cover at least one segment-based automation from rule to actions.

---

## Epic 8 – AI & Analytics Features

Implement AI-driven segmentation, sentiment analysis, and follow-up suggestions as per `fdd_1.md` §5.

### Story CRM-033 – AI-Driven Customer Segmentation Function

**As a** CRM manager  
**I want** AI to generate customer segments based on a high-level prompt  
**So that** I can discover useful customer groups without manual rule-writing.

**Acceptance Criteria**
- [ ] An Edge Function exists to implement §5.1:
  - [ ] Accepts segment ID and uses `crm_segments.ai_prompt`, schema description, and aggregated customer metrics as inputs to an AI API.
  - [ ] Receives and validates outputs (e.g., per-customer scores or rules) and updates `crm_segments.definition`, `ai_explanation`, and `crm_segment_members`.
- [ ] Errors from the AI provider are handled gracefully, with status communicated (e.g., via logs and segment flags/fields).
- [ ] Privacy and security concerns for sending data to the AI provider are documented and respected (e.g., aggregated vs raw PII).

**Definition of Done**
- [ ] Function is implemented and integrated with at least one AI provider (or a mock in early stages).
- [ ] Example runs on test data produce a plausible AI-generated segment with explanation and members.
- [ ] Limitations and costs of AI calls are documented.

---

### Story CRM-034 – Sentiment Analysis for Interactions

**As a** manager  
**I want** sentiment scores for customer interactions  
**So that** I can identify unhappy customers and prioritize follow-ups.

**Acceptance Criteria**
- [ ] An Edge Function `analyze_interaction_sentiment` implements §5.2:
  - [ ] Takes interaction body/summary as input.
  - [ ] Calls AI API to classify sentiment into `positive`, `neutral`, or `negative`.
  - [ ] Updates `crm_interactions.sentiment` for the interaction.
- [ ] A trigger or background mechanism ensures new interactions are analyzed automatically (e.g., DB trigger calling Edge Function or scheduled batch).
- [ ] Failure modes (e.g., AI timeout) result in safe defaults and are logged; they do not block interaction creation.

**Definition of Done**
- [ ] Sentiment analysis function and invocation mechanism are implemented and documented.
- [ ] Test interactions demonstrate correct sentiment classification for at least basic cases.
- [ ] Any costs or rate limits for the AI provider are documented.

---

### Story CRM-035 – AI-Based Follow-Up Suggestions

**As a** CSR or manager  
**I want** AI-suggested follow-ups based on a customer’s history  
**So that** I can proactively engage customers at the right time.

**Acceptance Criteria**
- [ ] An Edge Function `suggest_followups_for_customer` implements §5.3:
  - [ ] Takes `customer_id` and fetches relevant history (interactions, work orders, quotes).
  - [ ] Calls an AI API to recommend follow-up actions and timing.
  - [ ] Creates `crm_followups` records with `origin = ai_recommendation`.
- [ ] API access (e.g., an endpoint) allows UI to request suggestions for a given customer with appropriate role checks.
- [ ] Safeguards exist to prevent excessive or duplicate AI-generated follow-ups (e.g., thresholds per customer).

**Definition of Done**
- [ ] Function, supporting queries, and optional endpoint are implemented and documented.
- [ ] At least one test customer scenario shows AI-suggested follow-ups being generated and stored.
- [ ] Any explicit limitations (e.g., beta feature, not for all orgs) are documented.

---

## Epic 9 – Frontend (Next.js) CRM UI

Build Next.js UI components and pages to expose CRM features as per `fdd_1.md` §6 and platform choices in `tooling.md`.

### Story CRM-036 – Customers List Page

**As a** CSR  
**I want** a customers list view with search and filters  
**So that** I can quickly find and select the right customer record.

**Acceptance Criteria**
- [ ] A Next.js page (e.g., `/crm/customers`) displays:
  - [ ] Search bar supporting name/email/phone query.
  - [ ] Filters for status, lifecycle_stage, tags, and segments.
  - [ ] Table or card list with key details (name, type, primary contact, tags, lifecycle stage).
- [ ] Page consumes the `GET /crm/customers` API and renders paginated results.
- [ ] Loading, empty state, and basic error states are handled gracefully.

**Definition of Done**
- [ ] UI is consistent with overall design system (if defined).
- [ ] Role-based access to the page is enforced (e.g., only authenticated CRM roles).
- [ ] Verified in a dev environment against sample data.

---

### Story CRM-037 – Customer Detail Page

**As a** CSR or manager  
**I want** a detailed customer view with tabs for overview, activity, segments, and notes  
**So that** I can manage and understand the customer in one place.

**Acceptance Criteria**
- [ ] A Next.js page (e.g., `/crm/customers/[id]`) shows:
  - [ ] Summary header with name, lifecycle stage, status, tags, primary contact, and preferred channel.
  - [ ] Tabs/sections: Overview (locations, preferences), Activity (interactions + follow-ups), Segments (membership + scores), Notes.
- [ ] Page uses the "Get Customer Details" API (CRM-018) and related endpoints for timeline and follow-ups.
- [ ] Inline editing (or clearly scoped edit flows) are available for customer core details, locations, contacts, and tags where appropriate.

**Definition of Done**
- [ ] UI behavior for each tab is implemented and tested with real data.
- [ ] Access control is enforced (e.g., techs see limited fields vs CSRs/admins).
- [ ] Page performance is acceptable (no obvious blocking UI while loading).

---

### Story CRM-038 – Follow-Ups Dashboard Page

**As a** CSR or manager  
**I want** a dedicated view of upcoming and overdue follow-ups  
**So that** I can manage my work and team workload.

**Acceptance Criteria**
- [ ] A Next.js page (e.g., `/crm/followups`) displays:
  - [ ] List of follow-ups with filters for assignee, status, priority, and date range.
  - [ ] Visual indicators for overdue and high-priority items.
  - [ ] Quick actions to complete or reschedule a follow-up.
- [ ] Page consumes `GET /crm/followups` and `PATCH /crm/followups/:id` APIs.
- [ ] Role-based visibility rules are respected (e.g., techs only see their own, managers can see team).

**Definition of Done**
- [ ] UI and interactions are implemented and verified against the backend.
- [ ] UX for marking tasks complete or rescheduling is intuitive and responsive.

---

### Story CRM-039 – Segments Management UI

**As a** CRM admin/manager  
**I want** to manage segments and inspect their members  
**So that** I can define and validate my targeting strategy.

**Acceptance Criteria**
- [ ] A Next.js page (e.g., `/crm/segments`) lists segments with type, member count, active status, and last computed timestamp.
- [ ] A segment detail page (e.g., `/crm/segments/[id]`) includes:
  - [ ] Segment metadata (name, description, type).
  - [ ] Editor for `definition` or `ai_prompt` (depending on type).
  - [ ] Members list (using CRM-027 API).
  - [ ] Action to trigger recompute (CRM-028).
- [ ] Only users with appropriate roles can create/edit segments and trigger recomputes.

**Definition of Done**
- [ ] UI flows for creating, editing, and reviewing segments are functional.
- [ ] Error handling is implemented (e.g., invalid definitions, recompute failures).

---

### Story CRM-040 – Automation Rules Management UI

**As a** CRM admin  
**I want** a UI for managing automation rules and viewing recent runs  
**So that** I can configure, monitor, and troubleshoot automations.

**Acceptance Criteria**
- [ ] A Next.js page (e.g., `/crm/automation`) lists rules with trigger types, enabled status, and basic description.
- [ ] A rule detail view allows editing trigger configuration, conditions, and actions (within scope of early implementation).
- [ ] A section shows recent `crm_automation_runs` for the rule, with status and error messages.
- [ ] Role-based access is enforced such that only admin/manager roles can modify rules.

**Definition of Done**
- [ ] UI is implemented to the level supported by the backend rule model.
- [ ] Valid and invalid configurations lead to clear user feedback.

---

### Story CRM-041 – Reusable CRM UI Components

**As a** frontend engineer  
**I want** reusable CRM UI components  
**So that** CRM pages have consistent UX and are easy to maintain.

**Acceptance Criteria**
- [ ] Components are created for at least:
  - [ ] Customer avatar + badge.
  - [ ] Tag selector and tag chips.
  - [ ] Interaction timeline item.
  - [ ] Follow-up card.
- [ ] Components are documented (e.g., Storybook or internal docs) with usage examples.
- [ ] Components handle loading and error states appropriately when data is not yet available.

**Definition of Done**
- [ ] Components are integrated into relevant pages (customers list/detail, follow-ups, segments).
- [ ] Basic unit or visual tests (if applicable) ensure components render correctly with sample props.

---

## Epic 10 – Security, Privacy & Compliance

Implement privacy, opt-out enforcement, and audit logging as per `fdd_1.md` §7.

### Story CRM-042 – Enforce Communication Preferences in Automations & Messaging

**As a** compliance-conscious operator  
**I want** automations and messaging to respect customer preference flags  
**So that** we honor opt-outs and reduce regulatory risk.

**Acceptance Criteria**
- [ ] Automation flows (event/time-based/segment-based) and any messaging utilities check:
  - [ ] `crm_preferences` flags (`do_not_contact`, `do_not_email`, `do_not_sms`, `do_not_call`).
  - [ ] `customer_contacts` opt-in flags for marketing/transactional communications.
- [ ] If a communication would violate preferences, the action is skipped and, where appropriate, logged as `skipped` in `crm_automation_runs`.
- [ ] Manual messaging UIs display clear warnings when attempting to send to opted-out customers/contacts.

**Definition of Done**
- [ ] Preference-check logic is centralized and documented for reuse.
- [ ] Test scenarios verify that prohibited communications are not sent.

---

### Story CRM-043 – CRM Data Access Audit Logging

**As a** security/compliance officer  
**I want** audit logs for sensitive CRM data changes  
**So that** we can investigate incidents and meet compliance requirements.

**Acceptance Criteria**
- [ ] Changes to key CRM tables (e.g., `customers`, `crm_preferences`) are logged to an audit trail (e.g., separate audit table or Supabase logs) with:
  - [ ] Who made the change (user ID).
  - [ ] When it occurred.
  - [ ] What changed (at least high-level).
- [ ] Access to audit logs is restricted to appropriate roles.
- [ ] Retention policy for audit data is documented (even if not fully automated yet).

**Definition of Done**
- [ ] Triggers or Edge Functions implementing audit logging are in place and tested.
- [ ] Documentation describes how to query audit logs and which events are captured.

---

## Epic 11 – Non-Functional Requirements & Observability

Address performance, scalability, reliability, and observability as per `fdd_1.md` §8.

### Story CRM-044 – Performance Baseline & Query Optimization

**As a** technical lead  
**I want** to baseline and optimize key CRM queries  
**So that** user-facing operations remain fast as data grows.

**Acceptance Criteria**
- [ ] Identify and document at least 3–5 critical CRM queries (e.g., customer search, interactions by customer, follow-ups dashboard).
- [ ] Measure response times on a representative dataset and capture results.
- [ ] Apply optimizations where needed (indexes, query changes) to meet target response times (e.g., \< 500 ms for typical loads).
- [ ] Document remaining known performance constraints or future optimization areas.

**Definition of Done**
- [ ] Performance measurements and optimization decisions are stored in repo docs.
- [ ] Confirmed via testing that after optimizations, observed performance meets or approaches targets.

---

### Story CRM-045 – Reliability & Idempotency for Automations

**As a** reliability engineer  
**I want** automation actions to be idempotent and resilient  
**So that** repeated processing or transient errors do not cause duplicate or incorrect outcomes.

**Acceptance Criteria**
- [ ] For each automation pathway (event, time-based, segment-based), idempotency behavior is defined (e.g., deduping by rule-customer-event key).
- [ ] Implementation prevents duplicate follow-ups or messages for the same logical trigger.
- [ ] Failed runs can be safely retried without corrupting data.
- [ ] `crm_automation_runs` status values are used consistently to reflect lifecycle of a run.

**Definition of Done**
- [ ] Idempotency logic is implemented and tested with simulated duplicate events.
- [ ] Documentation explains the idempotency keys and retry strategies.

---

### Story CRM-046 – Logging & Monitoring for CRM Edge Functions

**As a** DevOps engineer  
**I want** structured logging and basic monitoring for CRM Edge Functions  
**So that** issues can be diagnosed quickly in production.

**Acceptance Criteria**
- [ ] All CRM Edge Functions (segmentation, sentiment, follow-ups, automations) log key events:
  - [ ] Inputs (with sensitivity/PII considerations).
  - [ ] Decisions taken (e.g., number of follow-ups created).
  - [ ] Errors and warnings.
- [ ] Logs are accessible via Supabase or external observability tooling, with clear naming conventions.
- [ ] At least basic alerting criteria are defined (even if not yet fully wired to an alerting system).

**Definition of Done**
- [ ] Logging patterns are consistent across functions and documented.
- [ ] At least one simulated error path is verified to produce useful logs.

---

## Epic 12 – Documentation & Developer Experience

Ensure developers and LLM-based task generators can work effectively with the CRM module.

### Story CRM-047 – CRM Developer Documentation

**As a** developer  
**I want** clear documentation for the CRM module  
**So that** I can understand and extend its capabilities confidently.

**Acceptance Criteria**
- [ ] A CRM-specific README or docs section explains:
  - [ ] Data model overview (tables, key relationships).
  - [ ] Key APIs and Edge Functions.
  - [ ] RLS and role semantics.
  - [ ] Integration points with other modules (e.g., work orders, quoting).
- [ ] References to `fdd_1.md`, `functional.md`, and `tooling.md` are included for context.
- [ ] Onboarding instructions for setting up test data and running CRM-related services locally are documented.

**Definition of Done**
- [ ] Documentation is committed, reviewed, and kept in sync with the implemented schema and APIs.
- [ ] At least one other engineer or reviewer confirms that the docs are sufficient to start working on CRM tasks.

---

### Story CRM-048 – API Reference for CRM Endpoints

**As a** frontend engineer or integrator  
**I want** a concise API reference for CRM endpoints  
**So that** I can consume them without reading all backend code.

**Acceptance Criteria**
- [ ] An API reference (Markdown or OpenAPI/JSON) exists for all CRM endpoints defined in this document (customers, interactions, follow-ups, segments, automation, AI).
- [ ] Each endpoint includes method, path, authentication requirements, request schema, response schema, and example payloads.
- [ ] Any important error codes or business rules are called out.

**Definition of Done**
- [ ] API reference is stored in the repo and kept versioned.
- [ ] Reference is linked from CRM README and discoverable by developers.

---

These epics and stories collectively provide comprehensive coverage for implementing the **CRM module (fdd_1.md)**, aligned with the overall platform capabilities and technology choices described in `functional.md`, `tooling.md`, and `overview.md`.


