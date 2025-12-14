# Technical Design Document – Epic 3: Security: RLS Policies and Least-Privilege Access

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 3 – Security: RLS Policies and Least-Privilege Access
- **Source**: Derived from `fdd_2_agile.md` Epic 3 (Stories DISP-017 through DISP-020)
- **Reference Documents**: 
  - `fdd_2.md` (Functional Design Document for Scheduling & Dispatch)
  - `fdd_2_agile.md` (Agile User Stories)
  - `tdd_2_epic_1.md` (Dispatch Epic 1 for tenancy/role conventions)
  - `tdd_2_epic_2.md` (Dispatch Epic 2 for table schemas)
- **Target Platform**: Supabase (PostgreSQL 15+)
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing Row-Level Security (RLS) policies for all Scheduling & Dispatch tables. It covers:

- RLS enablement for all 14 dispatch tables
- Tenant isolation policies (org_id filtering)
- Role-based access policies (admin, dispatcher, technician, CSR, customer)
- Field-level update restrictions for technicians
- Customer portal access strategy
- Testing and validation requirements

All RLS policies are designed to enforce least-privilege access, ensuring users can only access data within their organization and according to their role permissions. This epic assumes Epic 1 (tenancy/roles) and Epic 2 (tables) are complete.

---

## 2. Prerequisites

### 2.1 Required Components

Before implementing Epic 3, ensure:

1. **Epic 1 Complete**: Helper functions exist:
   - `get_user_org_id()` - Returns authenticated user's org_id
   - `get_user_role()` - Returns authenticated user's role
   - `is_user_technician(technician_profile_id UUID)` - Checks if user is a technician

2. **Epic 2 Complete**: All 14 dispatch tables exist:
   - `technician_profiles`
   - `technician_skills`
   - `service_zones`
   - `technician_service_zones`
   - `technician_shifts`
   - `technician_time_off`
   - `dispatch_jobs`
   - `job_time_windows`
   - `job_assignments`
   - `route_plans`
   - `route_stops`
   - `calendar_integrations`
   - `calendar_events`
   - `job_notifications`

3. **Required Tables**:
   - `orgs` table exists
   - `profiles` table exists with `user_id`, `org_id`, `role` columns
   - `customers` table exists (from CRM)
   - `customer_locations` table exists (from CRM)
   - `customer_contacts` table exists (from CRM, for notifications)

### 2.2 Helper Functions Reference

These functions are assumed to exist from Epic 1:

```sql
-- Get user's org_id
CREATE OR REPLACE FUNCTION get_user_org_id()
RETURNS UUID AS $$
  SELECT org_id FROM profiles WHERE user_id = auth.uid();
$$ LANGUAGE sql SECURITY DEFINER STABLE;

-- Get user's role
CREATE OR REPLACE FUNCTION get_user_role()
RETURNS TEXT AS $$
  SELECT role FROM profiles WHERE user_id = auth.uid();
$$ LANGUAGE sql SECURITY DEFINER STABLE;

-- Check if user is a technician
CREATE OR REPLACE FUNCTION is_user_technician(technician_profile_id UUID)
RETURNS BOOLEAN AS $$
  SELECT EXISTS (
    SELECT 1 FROM technician_profiles
    WHERE id = technician_profile_id
      AND user_id = auth.uid()
      AND org_id = get_user_org_id()
  );
$$ LANGUAGE sql SECURITY DEFINER STABLE;
```

---

## 3. RLS Policy Patterns

### 3.1 Policy Naming Convention

Policies follow this naming pattern:
- `<table>_<role>_<operation>` for role-specific policies
- `<table>_tenant_isolation` for baseline tenant isolation

Examples:
- `technician_profiles_admin_dispatcher_full` - Admin/dispatcher full access
- `job_assignments_technician_read_own` - Technician read own assignments
- `dispatch_jobs_tenant_isolation` - Baseline tenant isolation

### 3.2 Policy Structure Pattern

Each table follows this structure:

1. **Enable RLS**: `ALTER TABLE <table> ENABLE ROW LEVEL SECURITY;`
2. **Tenant Isolation Policy**: Baseline org_id filtering (if needed)
3. **Admin/Dispatcher Policies**: Full CRUD access within org
4. **Technician Policies**: Self-scoped read/update access
5. **CSR Policies**: Limited read/create access
6. **Customer Policies**: Portal-specific read access (if applicable)

### 3.3 Role Hierarchy

Policies are ordered by privilege level:
- **Admin/Dispatcher**: Highest privilege (often combined)
- **Technician**: Self-scoped access
- **CSR**: Limited access
- **Customer**: Portal read-only access

---

## 4. RLS Policies by Table

### 4.1 `technician_profiles` Table Policies

**Permission Matrix** (from Epic 1):
- Admin: CRUD
- Dispatcher: CRUD
- Technician: R (self only)
- CSR: R
- Customer: N

**DDL**:

```sql
-- Enable RLS
ALTER TABLE technician_profiles ENABLE ROW LEVEL SECURITY;

-- Admin/Dispatcher: Full CRUD access within org
CREATE POLICY "technician_profiles_admin_dispatcher_full"
ON technician_profiles FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
);

-- Technician: Read own profile only
CREATE POLICY "technician_profiles_technician_read_own"
ON technician_profiles FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  user_id = auth.uid()
);

-- CSR: Read-only access within org
CREATE POLICY "technician_profiles_csr_read"
ON technician_profiles FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
);
```

**Policy Notes**:
- Technicians can only read their own profile (via `user_id = auth.uid()`)
- CSR has read-only access for scheduling context
- No customer access (customers don't need technician data)

---

### 4.2 `technician_skills` Table Policies

**Permission Matrix**:
- Admin: CRUD
- Dispatcher: CRUD
- Technician: R (self only)
- CSR: R
- Customer: N

**DDL**:

```sql
-- Enable RLS
ALTER TABLE technician_skills ENABLE ROW LEVEL SECURITY;

-- Admin/Dispatcher: Full CRUD access within org
CREATE POLICY "technician_skills_admin_dispatcher_full"
ON technician_skills FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
);

-- Technician: Read own skills only
CREATE POLICY "technician_skills_technician_read_own"
ON technician_skills FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  technician_id IN (
    SELECT id FROM technician_profiles 
    WHERE user_id = auth.uid() AND org_id = get_user_org_id()
  )
);

-- CSR: Read-only access within org
CREATE POLICY "technician_skills_csr_read"
ON technician_skills FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
);
```

**Policy Notes**:
- Technicians can read skills for their own technician profile
- Skills are managed by admin/dispatcher only

---

### 4.3 `service_zones` Table Policies

**Permission Matrix**:
- Admin: CRUD
- Dispatcher: CRUD
- Technician: R
- CSR: R
- Customer: N

**DDL**:

```sql
-- Enable RLS
ALTER TABLE service_zones ENABLE ROW LEVEL SECURITY;

-- Admin/Dispatcher: Full CRUD access within org
CREATE POLICY "service_zones_admin_dispatcher_full"
ON service_zones FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
);

-- Technician: Read-only access within org (needed for route context)
CREATE POLICY "service_zones_technician_read"
ON service_zones FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician'
);

-- CSR: Read-only access within org
CREATE POLICY "service_zones_csr_read"
ON service_zones FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
);
```

**Policy Notes**:
- Technicians need read access to see zone information for routes
- Zones are managed by admin/dispatcher only

---

### 4.4 `technician_service_zones` Table Policies

**Permission Matrix**:
- Admin: CRUD
- Dispatcher: CRUD
- Technician: R (self only)
- CSR: R
- Customer: N

**DDL**:

```sql
-- Enable RLS
ALTER TABLE technician_service_zones ENABLE ROW LEVEL SECURITY;

-- Admin/Dispatcher: Full CRUD access within org
CREATE POLICY "technician_service_zones_admin_dispatcher_full"
ON technician_service_zones FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
);

-- Technician: Read own zone assignments only
CREATE POLICY "technician_service_zones_technician_read_own"
ON technician_service_zones FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  technician_id IN (
    SELECT id FROM technician_profiles 
    WHERE user_id = auth.uid() AND org_id = get_user_org_id()
  )
);

-- CSR: Read-only access within org
CREATE POLICY "technician_service_zones_csr_read"
ON technician_service_zones FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
);
```

---

### 4.5 `technician_shifts` Table Policies

**Permission Matrix**:
- Admin: CRUD
- Dispatcher: CRUD
- Technician: R (self only)
- CSR: R
- Customer: N

**DDL**:

```sql
-- Enable RLS
ALTER TABLE technician_shifts ENABLE ROW LEVEL SECURITY;

-- Admin/Dispatcher: Full CRUD access within org
CREATE POLICY "technician_shifts_admin_dispatcher_full"
ON technician_shifts FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
);

-- Technician: Read own shifts only
CREATE POLICY "technician_shifts_technician_read_own"
ON technician_shifts FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  technician_id IN (
    SELECT id FROM technician_profiles 
    WHERE user_id = auth.uid() AND org_id = get_user_org_id()
  )
);

-- CSR: Read-only access within org (for scheduling context)
CREATE POLICY "technician_shifts_csr_read"
ON technician_shifts FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
);
```

---

### 4.6 `technician_time_off` Table Policies

**Permission Matrix**:
- Admin: CRUD
- Dispatcher: CRUD
- Technician: CRUD (self only)
- CSR: R
- Customer: N

**DDL**:

```sql
-- Enable RLS
ALTER TABLE technician_time_off ENABLE ROW LEVEL SECURITY;

-- Admin/Dispatcher: Full CRUD access within org
CREATE POLICY "technician_time_off_admin_dispatcher_full"
ON technician_time_off FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
);

-- Technician: Full CRUD for own time-off only
CREATE POLICY "technician_time_off_technician_self"
ON technician_time_off FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  technician_id IN (
    SELECT id FROM technician_profiles 
    WHERE user_id = auth.uid() AND org_id = get_user_org_id()
  )
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  technician_id IN (
    SELECT id FROM technician_profiles 
    WHERE user_id = auth.uid() AND org_id = get_user_org_id()
  )
);

-- CSR: Read-only access within org
CREATE POLICY "technician_time_off_csr_read"
ON technician_time_off FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
);
```

**Policy Notes**:
- Technicians can manage their own time-off requests (create, read, update, delete)
- This is the only technician configuration table where technicians have write access

---

### 4.7 `dispatch_jobs` Table Policies

**Permission Matrix**:
- Admin: CRUD
- Dispatcher: CRUD
- Technician: R (assigned jobs only)
- CSR: CRU (limited update)
- Customer: R (own jobs only, via portal)

**DDL**:

```sql
-- Enable RLS
ALTER TABLE dispatch_jobs ENABLE ROW LEVEL SECURITY;

-- Admin/Dispatcher: Full CRUD access within org
CREATE POLICY "dispatch_jobs_admin_dispatcher_full"
ON dispatch_jobs FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
);

-- Technician: Read assigned jobs only
CREATE POLICY "dispatch_jobs_technician_read_assigned"
ON dispatch_jobs FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  id IN (
    SELECT dispatch_job_id FROM job_assignments
    WHERE technician_id IN (
      SELECT id FROM technician_profiles 
      WHERE user_id = auth.uid() AND org_id = get_user_org_id()
    )
  )
);

-- CSR: Create and read within org, limited update
CREATE POLICY "dispatch_jobs_csr_create_read"
ON dispatch_jobs FOR INSERT
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
);

CREATE POLICY "dispatch_jobs_csr_read"
ON dispatch_jobs FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
);

CREATE POLICY "dispatch_jobs_csr_update_limited"
ON dispatch_jobs FOR UPDATE
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
  -- Note: Application layer should restrict CSR updates to non-critical fields
  -- (e.g., notes, description) but not status, priority, assignments
);
```

**CSR Update Restrictions**:
- CSR can update: `description`, `notes_internal`, `title`
- CSR cannot update: `status`, `priority`, `sla_start_at`, `sla_end_at`, `related_work_order_id`
- These restrictions are enforced in application layer (Edge Functions/API), not RLS

**Customer Portal Access**: See Section 5 for portal-specific strategy.

---

### 4.8 `job_time_windows` Table Policies

**Permission Matrix**:
- Admin: CRUD
- Dispatcher: CRUD
- Technician: R (assigned jobs only)
- CSR: CRU
- Customer: RU (select window, via portal)

**DDL**:

```sql
-- Enable RLS
ALTER TABLE job_time_windows ENABLE ROW LEVEL SECURITY;

-- Admin/Dispatcher: Full CRUD access within org
CREATE POLICY "job_time_windows_admin_dispatcher_full"
ON job_time_windows FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
);

-- Technician: Read windows for assigned jobs only
CREATE POLICY "job_time_windows_technician_read_assigned"
ON job_time_windows FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  dispatch_job_id IN (
    SELECT dispatch_job_id FROM job_assignments
    WHERE technician_id IN (
      SELECT id FROM technician_profiles 
      WHERE user_id = auth.uid() AND org_id = get_user_org_id()
    )
  )
);

-- CSR: Create, read, update within org
CREATE POLICY "job_time_windows_csr_full"
ON job_time_windows FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
);
```

**Customer Portal Access**: See Section 5 for portal-specific strategy.

---

### 4.9 `job_assignments` Table Policies

**Permission Matrix**:
- Admin: CRUD
- Dispatcher: CRUD
- Technician: RU (status/ETA only, own assignments)
- CSR: R
- Customer: R (own jobs, via portal)

**DDL**:

```sql
-- Enable RLS
ALTER TABLE job_assignments ENABLE ROW LEVEL SECURITY;

-- Admin/Dispatcher: Full CRUD access within org
CREATE POLICY "job_assignments_admin_dispatcher_full"
ON job_assignments FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
);

-- Technician: Read own assignments
CREATE POLICY "job_assignments_technician_read_own"
ON job_assignments FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  technician_id IN (
    SELECT id FROM technician_profiles 
    WHERE user_id = auth.uid() AND org_id = get_user_org_id()
  )
);

-- Technician: Update status and ETA only (own assignments)
CREATE POLICY "job_assignments_technician_update_limited"
ON job_assignments FOR UPDATE
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  technician_id IN (
    SELECT id FROM technician_profiles 
    WHERE user_id = auth.uid() AND org_id = get_user_org_id()
  )
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  technician_id IN (
    SELECT id FROM technician_profiles 
    WHERE user_id = auth.uid() AND org_id = get_user_org_id()
  )
  -- Note: Application layer must restrict updates to status and tech_eta_at only
  -- RLS cannot enforce field-level restrictions, so this is enforced in API/Edge Functions
);

-- CSR: Read-only access within org
CREATE POLICY "job_assignments_csr_read"
ON job_assignments FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
);
```

**Technician Update Restrictions**:
- Technicians can update: `status`, `tech_eta_at`
- Technicians cannot update: `scheduled_start_at`, `scheduled_end_at`, `technician_id`, `assigned_by_user_id`, `arrival_window_start`, `arrival_window_end`, `sequence_in_route`
- These restrictions are enforced in application layer (Edge Functions/API), not RLS

**Status Transition Validation**: Application layer should enforce valid state transitions (e.g., `assigned` -> `accepted` -> `en_route` -> `on_site` -> `completed`).

---

### 4.10 `route_plans` Table Policies

**Permission Matrix**:
- Admin: CRUD
- Dispatcher: CRUD
- Technician: R (self only)
- CSR: R
- Customer: N

**DDL**:

```sql
-- Enable RLS
ALTER TABLE route_plans ENABLE ROW LEVEL SECURITY;

-- Admin/Dispatcher: Full CRUD access within org
CREATE POLICY "route_plans_admin_dispatcher_full"
ON route_plans FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
);

-- Technician: Read own route plans only
CREATE POLICY "route_plans_technician_read_own"
ON route_plans FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  technician_id IN (
    SELECT id FROM technician_profiles 
    WHERE user_id = auth.uid() AND org_id = get_user_org_id()
  )
);

-- CSR: Read-only access within org
CREATE POLICY "route_plans_csr_read"
ON route_plans FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
);
```

---

### 4.11 `route_stops` Table Policies

**Permission Matrix**:
- Admin: CRUD
- Dispatcher: CRUD
- Technician: RU (actual times only, own routes)
- CSR: R
- Customer: N

**DDL**:

```sql
-- Enable RLS
ALTER TABLE route_stops ENABLE ROW LEVEL SECURITY;

-- Admin/Dispatcher: Full CRUD access within org
CREATE POLICY "route_stops_admin_dispatcher_full"
ON route_stops FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
);

-- Technician: Read own route stops
CREATE POLICY "route_stops_technician_read_own"
ON route_stops FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  route_plan_id IN (
    SELECT id FROM route_plans
    WHERE technician_id IN (
      SELECT id FROM technician_profiles 
      WHERE user_id = auth.uid() AND org_id = get_user_org_id()
    )
  )
);

-- Technician: Update actual arrival/departure times only (own routes)
CREATE POLICY "route_stops_technician_update_limited"
ON route_stops FOR UPDATE
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  route_plan_id IN (
    SELECT id FROM route_plans
    WHERE technician_id IN (
      SELECT id FROM technician_profiles 
      WHERE user_id = auth.uid() AND org_id = get_user_org_id()
    )
  )
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  route_plan_id IN (
    SELECT id FROM route_plans
    WHERE technician_id IN (
      SELECT id FROM technician_profiles 
      WHERE user_id = auth.uid() AND org_id = get_user_org_id()
    )
  )
  -- Note: Application layer must restrict updates to actual_arrival_at and actual_departure_at only
);

-- CSR: Read-only access within org
CREATE POLICY "route_stops_csr_read"
ON route_stops FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
);
```

**Technician Update Restrictions**:
- Technicians can update: `actual_arrival_at`, `actual_departure_at`
- Technicians cannot update: `planned_arrival_at`, `planned_departure_at`, `sequence`, `latitude`, `longitude`, `travel_time_minutes_from_prev`, `distance_km_from_prev`
- These restrictions are enforced in application layer

---

### 4.12 `calendar_integrations` Table Policies

**Permission Matrix**:
- Admin: CRUD (but tokens not readable)
- Dispatcher: CRUD (self only)
- Technician: CRUD (self only)
- CSR: N
- Customer: N

**DDL**:

```sql
-- Enable RLS
ALTER TABLE calendar_integrations ENABLE ROW LEVEL SECURITY;

-- Admin: Full CRUD access within org (but tokens encrypted)
CREATE POLICY "calendar_integrations_admin_full"
ON calendar_integrations FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'admin'
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() = 'admin'
);

-- Dispatcher/Technician: Full CRUD for own integrations only
CREATE POLICY "calendar_integrations_user_self"
ON calendar_integrations FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('dispatcher', 'technician') AND
  user_id = auth.uid()
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() IN ('dispatcher', 'technician') AND
  user_id = auth.uid()
);
```

**Security Notes**:
- Tokens (`access_token`, `refresh_token`) are encrypted and never exposed via RLS
- Users can only manage their own calendar integrations
- Admin can view integration status but tokens are encrypted (application layer handles decryption)

**Token Access Restriction**: Create a view or function that excludes token columns for non-admin users:

```sql
-- View for reading calendar integrations without tokens
CREATE VIEW calendar_integrations_public AS
SELECT 
  id,
  org_id,
  user_id,
  provider,
  provider_account_id,
  expires_at,
  calendar_id,
  scope,
  is_active,
  metadata,
  created_at,
  updated_at
FROM calendar_integrations;

-- RLS on view (reuses table policies)
ALTER VIEW calendar_integrations_public SET (security_invoker = true);
```

---

### 4.13 `calendar_events` Table Policies

**Permission Matrix**:
- Admin: CRUD
- Dispatcher: CRUD
- Technician: R (own events)
- CSR: R
- Customer: N

**DDL**:

```sql
-- Enable RLS
ALTER TABLE calendar_events ENABLE ROW LEVEL SECURITY;

-- Admin/Dispatcher: Full CRUD access within org
CREATE POLICY "calendar_events_admin_dispatcher_full"
ON calendar_events FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
);

-- Technician: Read own calendar events
CREATE POLICY "calendar_events_technician_read_own"
ON calendar_events FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  calendar_integration_id IN (
    SELECT id FROM calendar_integrations
    WHERE user_id = auth.uid() AND org_id = get_user_org_id()
  )
);

-- CSR: Read-only access within org
CREATE POLICY "calendar_events_csr_read"
ON calendar_events FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
);
```

---

### 4.14 `job_notifications` Table Policies

**Permission Matrix**:
- Admin: CRUD
- Dispatcher: CRUD
- Technician: R (related assignments)
- CSR: R
- Customer: R (own jobs, via portal)

**DDL**:

```sql
-- Enable RLS
ALTER TABLE job_notifications ENABLE ROW LEVEL SECURITY;

-- Admin/Dispatcher: Full CRUD access within org
CREATE POLICY "job_notifications_admin_dispatcher_full"
ON job_notifications FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
)
WITH CHECK (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
);

-- Technician: Read notifications for own assignments
CREATE POLICY "job_notifications_technician_read_related"
ON job_notifications FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  (
    job_assignment_id IN (
      SELECT id FROM job_assignments
      WHERE technician_id IN (
        SELECT id FROM technician_profiles 
        WHERE user_id = auth.uid() AND org_id = get_user_org_id()
      )
    ) OR
    dispatch_job_id IN (
      SELECT dispatch_job_id FROM job_assignments
      WHERE technician_id IN (
        SELECT id FROM technician_profiles 
        WHERE user_id = auth.uid() AND org_id = get_user_org_id()
      )
    )
  )
);

-- CSR: Read-only access within org
CREATE POLICY "job_notifications_csr_read"
ON job_notifications FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
);
```

**Customer Portal Access**: See Section 5 for portal-specific strategy.

---

## 5. Customer Portal Access Strategy

### 5.1 Overview

Customers accessing via portal need read-only access to their own appointments. However, direct table access via RLS is not recommended for security reasons. Instead, use portal-specific APIs/views.

### 5.2 Strategy: Portal-Specific Database Views

**Decision**: Create read-only database views that filter customer data, then grant access via RLS on views.

**Rationale**:
- Views provide controlled access to specific columns
- RLS on views is simpler than complex customer policies on base tables
- Views can join related data (jobs, time windows, assignments) in one query
- Easier to maintain and audit

### 5.3 Implementation

#### 5.3.1 Customer Portal View for Jobs

```sql
-- View for customer portal: own jobs with time windows
CREATE VIEW customer_portal_jobs AS
SELECT 
  dj.id,
  dj.org_id,
  dj.customer_id,
  dj.title,
  dj.description,
  dj.job_type,
  dj.priority,
  dj.status,
  dj.estimated_duration_minutes,
  dj.sla_start_at,
  dj.sla_end_at,
  dj.is_customer_booked,
  dj.created_at,
  dj.updated_at,
  -- Time windows
  COALESCE(
    json_agg(
      json_build_object(
        'id', jtw.id,
        'window_start', jtw.window_start,
        'window_end', jtw.window_end,
        'source', jtw.source,
        'is_selected', jtw.is_selected
      )
    ) FILTER (WHERE jtw.id IS NOT NULL),
    '[]'::json
  ) AS time_windows
FROM dispatch_jobs dj
LEFT JOIN job_time_windows jtw ON jtw.dispatch_job_id = dj.id
WHERE dj.customer_id IN (
  SELECT customer_id FROM customers 
  WHERE id IN (
    SELECT customer_id FROM customer_contacts
    WHERE id IN (
      SELECT contact_id FROM portal_user_contacts
      WHERE user_id = auth.uid()
    )
  )
)
GROUP BY dj.id, dj.org_id, dj.customer_id, dj.title, dj.description, 
         dj.job_type, dj.priority, dj.status, dj.estimated_duration_minutes,
         dj.sla_start_at, dj.sla_end_at, dj.is_customer_booked,
         dj.created_at, dj.updated_at;

-- Enable RLS on view
ALTER VIEW customer_portal_jobs SET (security_invoker = true);

-- RLS Policy: Customers can only see their own jobs
CREATE POLICY "customer_portal_jobs_read_own"
ON customer_portal_jobs FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'customer' AND
  customer_id IN (
    SELECT customer_id FROM customers 
    WHERE id IN (
      SELECT customer_id FROM customer_contacts
      WHERE id IN (
        SELECT contact_id FROM portal_user_contacts
        WHERE user_id = auth.uid()
      )
    )
  )
);
```

**Note**: `portal_user_contacts` is a mapping table linking portal users (`auth.users.id`) to customer contacts. This table should be created in the Portal module.

#### 5.3.2 Customer Portal View for Assignments

```sql
-- View for customer portal: assignments for own jobs
CREATE VIEW customer_portal_assignments AS
SELECT 
  ja.id,
  ja.org_id,
  ja.dispatch_job_id,
  ja.scheduled_start_at,
  ja.scheduled_end_at,
  ja.arrival_window_start,
  ja.arrival_window_end,
  ja.status,
  ja.tech_eta_at,
  ja.created_at,
  ja.updated_at,
  -- Technician info (display name only)
  tp.display_name AS technician_name
FROM job_assignments ja
JOIN technician_profiles tp ON tp.id = ja.technician_id
WHERE ja.dispatch_job_id IN (
  SELECT id FROM dispatch_jobs
  WHERE customer_id IN (
    SELECT customer_id FROM customers 
    WHERE id IN (
      SELECT customer_id FROM customer_contacts
      WHERE id IN (
        SELECT contact_id FROM portal_user_contacts
        WHERE user_id = auth.uid()
      )
    )
  )
);

-- Enable RLS on view
ALTER VIEW customer_portal_assignments SET (security_invoker = true);

-- RLS Policy: Customers can only see assignments for their own jobs
CREATE POLICY "customer_portal_assignments_read_own"
ON customer_portal_assignments FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'customer'
);
```

#### 5.3.3 Customer Portal Function for Window Selection

```sql
-- Function for customers to select time windows (via portal API)
CREATE OR REPLACE FUNCTION customer_select_time_window(
  p_job_id UUID,
  p_window_id UUID
)
RETURNS BOOLEAN AS $$
DECLARE
  v_customer_id UUID;
BEGIN
  -- Verify customer owns the job
  SELECT customer_id INTO v_customer_id
  FROM dispatch_jobs
  WHERE id = p_job_id
    AND org_id = get_user_org_id()
    AND customer_id IN (
      SELECT customer_id FROM customers 
      WHERE id IN (
        SELECT customer_id FROM customer_contacts
        WHERE id IN (
          SELECT contact_id FROM portal_user_contacts
          WHERE user_id = auth.uid()
        )
      )
    );
  
  IF v_customer_id IS NULL THEN
    RETURN false;
  END IF;
  
  -- Unselect other windows for this job
  UPDATE job_time_windows
  SET is_selected = false
  WHERE dispatch_job_id = p_job_id
    AND org_id = get_user_org_id()
    AND id != p_window_id;
  
  -- Select the specified window
  UPDATE job_time_windows
  SET is_selected = true
  WHERE id = p_window_id
    AND dispatch_job_id = p_job_id
    AND org_id = get_user_org_id();
  
  RETURN true;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Grant execute to authenticated users (portal will validate role)
GRANT EXECUTE ON FUNCTION customer_select_time_window(UUID, UUID) TO authenticated;
```

### 5.4 Alternative Strategy: Portal-Specific Edge Functions

**If views are not preferred**, use Edge Functions that:
1. Validate customer role and job ownership
2. Query base tables with service role key
3. Return filtered results

**Example**:
```typescript
// Edge Function: Get customer's appointments
export async function getCustomerAppointments(req: Request) {
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return new Response('Unauthorized', { status: 401 });
  
  // Verify customer role
  const { data: profile } = await supabase
    .from('profiles')
    .select('org_id, role')
    .eq('user_id', user.id)
    .single();
  
  if (profile?.role !== 'customer') {
    return new Response('Forbidden', { status: 403 });
  }
  
  // Get customer ID from portal mapping
  const { data: portalMapping } = await supabase
    .from('portal_user_contacts')
    .select('contact_id')
    .eq('user_id', user.id)
    .single();
  
  // Query jobs using service role (bypasses RLS)
  const supabaseAdmin = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
  );
  
  const { data: jobs } = await supabaseAdmin
    .from('dispatch_jobs')
    .select('*, job_time_windows(*)')
    .eq('org_id', profile.org_id)
    .eq('customer_id', portalMapping.contact_id);
  
  return new Response(JSON.stringify(jobs), {
    headers: { 'Content-Type': 'application/json' }
  });
}
```

### 5.5 Recommended Approach

**Decision**: Use **portal-specific Edge Functions** (Section 5.4) rather than views.

**Rationale**:
- Simpler security model (no complex view RLS)
- Easier to add business logic (e.g., filtering by date range)
- Better performance (can optimize queries)
- Aligns with Supabase best practices (Edge Functions for API endpoints)

**Implementation**: Portal module will implement Edge Functions for customer access. RLS on base tables does not need customer policies.

---

## 6. Field-Level Update Restrictions

### 6.1 Technician Update Restrictions

Technicians have limited update permissions. These restrictions are enforced in **application layer** (Edge Functions/API), not RLS, because RLS cannot enforce field-level restrictions.

#### 6.1.1 `job_assignments` Update Restrictions

**Allowed Fields**:
- `status` - Assignment status updates
- `tech_eta_at` - ETA updates

**Restricted Fields**:
- `scheduled_start_at` - Dispatcher-only
- `scheduled_end_at` - Dispatcher-only
- `technician_id` - Dispatcher-only (reassignment)
- `assigned_by_user_id` - System-only
- `arrival_window_start` - Dispatcher-only
- `arrival_window_end` - Dispatcher-only
- `sequence_in_route` - Optimizer-only
- `is_primary_technician` - Dispatcher-only

**Implementation Pattern** (Edge Function):
```typescript
async function updateAssignmentStatus(
  assignmentId: string,
  status: string,
  eta?: string
) {
  // Verify technician owns assignment
  const { data: assignment } = await supabase
    .from('job_assignments')
    .select('technician_id')
    .eq('id', assignmentId)
    .single();
  
  const { data: techProfile } = await supabase
    .from('technician_profiles')
    .select('id')
    .eq('user_id', auth.user.id)
    .single();
  
  if (assignment?.technician_id !== techProfile?.id) {
    throw new Error('Unauthorized');
  }
  
  // Update only allowed fields
  const updates: any = { status };
  if (eta) updates.tech_eta_at = eta;
  
  const { error } = await supabase
    .from('job_assignments')
    .update(updates)
    .eq('id', assignmentId);
  
  if (error) throw error;
}
```

#### 6.1.2 `route_stops` Update Restrictions

**Allowed Fields**:
- `actual_arrival_at` - Actual arrival time
- `actual_departure_at` - Actual departure time

**Restricted Fields**:
- All other fields (planned times, sequence, coordinates, etc.)

**Implementation**: Similar pattern to `job_assignments`.

### 6.2 CSR Update Restrictions

CSR can update `dispatch_jobs` but only non-critical fields:

**Allowed Fields**:
- `title`
- `description`
- `notes_internal`

**Restricted Fields**:
- `status`
- `priority`
- `sla_start_at`
- `sla_end_at`
- `related_work_order_id`
- `service_zone_id`
- `required_skills`
- `required_crew_size`

**Implementation**: Enforce in Edge Functions/API layer.

---

## 7. Testing Requirements

### 7.1 Multi-Tenancy Tests

**Test Cases**:

1. **Cross-Org Isolation**:
   - User from Org A cannot see data from Org B
   - Queries filtered by `org_id` return only user's org data
   - RLS policies block cross-org access

2. **`org_id` Enforcement**:
   - All policies include `org_id = get_user_org_id()` check
   - Attempts to insert/update with wrong `org_id` are blocked

### 7.2 Role-Based Access Tests

**Test Cases**:

1. **Admin/Dispatcher Access**:
   - Can CRUD all dispatch entities in their org
   - Cannot access other orgs' data
   - Can manage technician configuration

2. **Technician Access**:
   - Can read own assignments, routes, shifts
   - Can update own assignment status and ETA
   - Cannot modify other technicians' data
   - Cannot create/delete jobs or assignments
   - Can manage own time-off

3. **CSR Access**:
   - Can create jobs and read schedules
   - Can update limited fields on jobs
   - Cannot modify assignments or routes
   - Cannot modify technician configuration

4. **Customer Access** (via portal):
   - Can read own jobs and appointments (via portal APIs)
   - Cannot access other customers' data
   - Cannot modify scheduling data

### 7.3 Field-Level Update Tests

**Test Cases**:

1. **Technician Updates**:
   - Can update `job_assignments.status` and `tech_eta_at`
   - Cannot update `scheduled_start_at`, `scheduled_end_at`, `technician_id`
   - Can update `route_stops.actual_arrival_at` and `actual_departure_at`
   - Cannot update `route_stops.planned_arrival_at`, `sequence`

2. **CSR Updates**:
   - Can update `dispatch_jobs.title`, `description`, `notes_internal`
   - Cannot update `dispatch_jobs.status`, `priority`, `sla_start_at`

### 7.4 RLS Policy Coverage Tests

**Test Cases**:

1. **Policy Coverage**:
   - All dispatch tables have RLS enabled
   - Policies exist for all expected access patterns
   - Policies correctly filter by `org_id`

2. **Policy Correctness**:
   - Admin/dispatcher policies allow full access
   - Technician policies restrict to self-scoped data
   - CSR policies allow appropriate read/create access
   - Customer policies (if implemented) restrict to own data

### 7.5 Test Data Requirements

Create test data for:
- At least 2 organizations
- At least 2 users per role per org (where applicable)
- At least 2 technicians per org
- Jobs assigned to different technicians
- Cross-org data to verify isolation

---

## 8. Migration Strategy

### 8.1 Migration Order

RLS policies should be created after tables exist (Epic 2 complete):

1. **Verify Helper Functions**: Ensure `get_user_org_id()`, `get_user_role()`, `is_user_technician()` exist
2. **Enable RLS**: Enable RLS on all 14 dispatch tables
3. **Create Policies**: Create policies in dependency order (tables with FKs first)
4. **Test Policies**: Verify policies work correctly

### 8.2 Migration File Structure

```
supabase/migrations/
  20240101000016_enable_rls_technician_profiles.sql
  20240101000017_enable_rls_technician_skills.sql
  20240101000018_enable_rls_service_zones.sql
  20240101000019_enable_rls_technician_service_zones.sql
  20240101000020_enable_rls_technician_shifts.sql
  20240101000021_enable_rls_technician_time_off.sql
  20240101000022_enable_rls_dispatch_jobs.sql
  20240101000023_enable_rls_job_time_windows.sql
  20240101000024_enable_rls_job_assignments.sql
  20240101000025_enable_rls_route_plans.sql
  20240101000026_enable_rls_route_stops.sql
  20240101000027_enable_rls_calendar_integrations.sql
  20240101000028_enable_rls_calendar_events.sql
  20240101000029_enable_rls_job_notifications.sql
```

**Alternative**: Single migration file with all RLS policies (easier to review).

### 8.3 Rollback Strategy

To rollback RLS policies:
1. Drop all policies: `DROP POLICY IF EXISTS <policy_name> ON <table>;`
2. Disable RLS: `ALTER TABLE <table> DISABLE ROW LEVEL SECURITY;`

**Note**: Rollback will remove all security, so use with caution.

---

## 9. Performance Considerations

### 9.1 RLS Policy Performance

- **Helper Functions**: `get_user_org_id()` and `get_user_role()` are called frequently
- **Optimization**: Ensure `profiles.user_id` has index (from Epic 1)
- **Caching**: Consider caching user role/org_id in JWT claims (optional optimization)

### 9.2 Query Performance

- **Subqueries**: Policies with subqueries (e.g., technician self-scope) may impact performance
- **Indexes**: Ensure foreign keys are indexed (from Epic 2)
- **Testing**: Use `EXPLAIN ANALYZE` to verify policies don't degrade query performance

### 9.3 Policy Complexity

- **Keep Simple**: Avoid complex subqueries where possible
- **Use Indexes**: Policies should use indexed columns (`org_id`, `user_id`, `technician_id`)
- **Monitor**: Monitor query performance after enabling RLS

---

## 10. Security Considerations

### 10.1 RLS Enforcement

- **Requirement**: All dispatch tables MUST have RLS enabled
- **Validation**: Test that RLS blocks unauthorized access
- **Documentation**: Document which tables have RLS and which don't (if any exceptions)

### 10.2 Helper Function Security

- **`SECURITY DEFINER`**: Helper functions use `SECURITY DEFINER` to bypass RLS on `profiles` table
- **Risk**: Ensure helper functions don't expose sensitive data
- **Mitigation**: Functions only return `org_id` and `role`, not full profile data

### 10.3 Field-Level Restrictions

- **RLS Limitation**: RLS cannot enforce field-level update restrictions
- **Solution**: Enforce in application layer (Edge Functions/API)
- **Documentation**: Document which restrictions are enforced where

### 10.4 Token Security

- **Calendar Tokens**: Tokens in `calendar_integrations` are encrypted
- **RLS**: RLS policies don't expose tokens (application layer handles encryption/decryption)
- **Access**: Only users can manage their own integrations (admin can view status but not tokens)

---

## 11. Implementation Checklist

### Story DISP-017: Enable RLS on All Dispatch Tables

- [ ] **Enable RLS**:
  - [ ] Enable RLS on `technician_profiles`
  - [ ] Enable RLS on `technician_skills`
  - [ ] Enable RLS on `service_zones`
  - [ ] Enable RLS on `technician_service_zones`
  - [ ] Enable RLS on `technician_shifts`
  - [ ] Enable RLS on `technician_time_off`
  - [ ] Enable RLS on `dispatch_jobs`
  - [ ] Enable RLS on `job_time_windows`
  - [ ] Enable RLS on `job_assignments`
  - [ ] Enable RLS on `route_plans`
  - [ ] Enable RLS on `route_stops`
  - [ ] Enable RLS on `calendar_integrations`
  - [ ] Enable RLS on `calendar_events`
  - [ ] Enable RLS on `job_notifications`

- [ ] **Create Baseline Policies**:
  - [ ] Tenant isolation policy for each table (if needed)
  - [ ] Verify policies filter by `org_id`

- [ ] **Testing**:
  - [ ] Test cross-org access is blocked
  - [ ] Test policies work with at least 2 org contexts
  - [ ] Document test results

### Story DISP-018: Implement Role-Based Access for Dispatchers/Admins

- [ ] **Admin/Dispatcher Policies**:
  - [ ] Full CRUD policies for all 14 tables
  - [ ] Policies enforce `org_id` filtering
  - [ ] Policies check role (`admin` or `dispatcher`)

- [ ] **Calendar Integration Access**:
  - [ ] Admin can read calendar integration status (without tokens)
  - [ ] Dispatcher can manage own calendar integrations only

- [ ] **Testing**:
  - [ ] Verify admin/dispatcher can CRUD all entities in their org
  - [ ] Verify access outside org is blocked
  - [ ] Document test results

### Story DISP-019: Implement Technician Self-Scope Access

- [ ] **Technician Read Policies**:
  - [ ] Read own `job_assignments`
  - [ ] Read assigned `dispatch_jobs`
  - [ ] Read own `route_plans` and `route_stops`
  - [ ] Read own `technician_profiles`, `technician_skills`, `technician_service_zones`, `technician_shifts`

- [ ] **Technician Update Policies**:
  - [ ] Update own `job_assignments.status` and `tech_eta_at`
  - [ ] Update own `route_stops.actual_arrival_at` and `actual_departure_at`
  - [ ] Full CRUD on own `technician_time_off`

- [ ] **Field-Level Restrictions**:
  - [ ] Document allowed update fields for technicians
  - [ ] Implement Edge Function/API restrictions for field-level updates
  - [ ] Test restrictions work correctly

- [ ] **Testing**:
  - [ ] Test with at least 2 technicians in same org
  - [ ] Verify technicians cannot see other technicians' data
  - [ ] Verify technicians can update only allowed fields
  - [ ] Document test results

### Story DISP-020: Define Customer Read-Only Access Strategy for Portal Flows

- [ ] **Strategy Documentation**:
  - [ ] Document portal access strategy (Edge Functions recommended)
  - [ ] Document customer data access patterns
  - [ ] Document security considerations

- [ ] **Implementation** (if views chosen):
  - [ ] Create `customer_portal_jobs` view
  - [ ] Create `customer_portal_assignments` view
  - [ ] Create `customer_select_time_window()` function
  - [ ] Create RLS policies on views

- [ ] **Alternative Implementation** (if Edge Functions chosen):
  - [ ] Document Edge Function patterns for customer access
  - [ ] Note: Portal module will implement Edge Functions

- [ ] **Security Review**:
  - [ ] Review strategy for security risks
  - [ ] Align with portal TDD (Section 7)
  - [ ] Document review results

---

## 12. Appendix: Complete RLS Migration Script

A complete migration script combining all RLS policies is provided in a separate file: `supabase/migrations/YYYYMMDDHHMMSS_enable_dispatch_rls_policies.sql`

**Note**: This TDD provides the specification; the actual migration script should be generated from these specifications and tested in a development environment before applying to production.

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 3 – Security: RLS Policies and Least-Privilege Access. All RLS policies are designed to enforce tenant isolation and role-based access control, ensuring users can only access data within their organization and according to their role permissions.

**Next Steps**: After completing Epic 3, proceed to Epic 4 (Technician Configuration & Availability APIs) which will implement the API endpoints that use these RLS policies.

