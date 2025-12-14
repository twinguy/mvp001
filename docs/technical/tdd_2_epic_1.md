# Technical Design Document – Epic 1: Dispatch Multi-Tenancy, Roles, and Foundational Conventions

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 1 – Dispatch Multi-Tenancy, Roles, and Foundational Conventions
- **Source**: Derived from `fdd_2_agile.md` Epic 1 (Stories DISP-001 and DISP-002)
- **Reference Documents**: 
  - `fdd_2.md` (Functional Design Document for Scheduling & Dispatch)
  - `fdd_2_agile.md` (Agile User Stories)
  - `functional.md` (Platform Capabilities)
  - `tooling.md` (Platform Choices: Supabase + Vercel)
  - `tdd_1_epic_1.md` (CRM Epic 1 TDD for reference on tenancy patterns)
- **Target Platform**: Supabase (PostgreSQL 15+, Auth, Edge Functions)
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing the foundational multi-tenancy and role-based access control conventions required by the Scheduling & Dispatch module. It establishes:

- Multi-tenant data isolation strategy using `org_id`
- `org_id` resolution patterns for authenticated users and Edge Functions
- Role definitions and permission matrix for dispatch operations
- Cross-cutting conventions that all subsequent dispatch stories will follow

All specifications are designed to be directly implementable and consistent with the CRM module's tenancy approach (`tdd_1_epic_1.md`), ensuring platform-wide consistency.

---

## 2. Multi-Tenancy Foundation

### 2.1 Organizational Model Alignment

**Decision**: The Dispatch module reuses the same `orgs` table and tenancy model established by the CRM module (`tdd_1_epic_1.md` §2.1). No separate organizational model is created.

**Assumption**: The `orgs` table exists with the following structure:

```sql
CREATE TABLE IF NOT EXISTS orgs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**If `orgs` does not exist**: Create it using the DDL above before implementing any dispatch tables.

### 2.2 `org_id` Convention for Dispatch Tables

- **Column Name**: `org_id` (consistent across ALL dispatch tables)
- **Data Type**: `UUID` (references `orgs.id`)
- **Nullability**: `NOT NULL` on all dispatch tables requiring tenancy
- **Foreign Key**: `REFERENCES orgs(id) ON DELETE CASCADE` (cascade delete ensures cleanup)
- **RLS Integration**: All RLS policies will filter by `org_id` matching the authenticated user's organization

### 2.3 `org_id` Resolution Patterns

#### 2.3.1 User Profile-Based Resolution (Primary Pattern)

**Assumption**: A `profiles` table exists (or will be created) that links `auth.users` to organizations:

```sql
CREATE TABLE IF NOT EXISTS profiles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  role TEXT NOT NULL, -- e.g., 'admin', 'dispatcher', 'technician', 'csr'
  display_name TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT profiles_user_id_unique UNIQUE (user_id)
);

CREATE INDEX idx_profiles_user_id ON profiles(user_id);
CREATE INDEX idx_profiles_org_id ON profiles(org_id);
```

**Resolution Pattern**:
- For authenticated users: Query `profiles.org_id` WHERE `profiles.user_id = auth.uid()`
- Store in a helper function or RLS policy context

#### 2.3.2 JWT Claims-Based Resolution (Alternative Pattern)

If `profiles` table is not available or for Edge Functions, use JWT claims:

**Pattern**: Set `app.current_org_id` in JWT claims via middleware or Edge Function context

**Implementation**:
- Edge Functions extract `org_id` from JWT custom claims: `event.user.app_metadata.org_id` or `event.user.user_metadata.org_id`
- RLS policies use: `current_setting('app.current_org_id', true)::uuid`

**Note**: Supabase Auth JWT can include custom claims set via:
- Database triggers on `profiles` table updates
- Auth hooks (Edge Functions)
- Admin API calls

#### 2.3.3 Edge Function Resolution Strategy

**For Supabase Edge Functions**:

1. **From Request Context**:
   ```typescript
   // In Edge Function
   const userId = event.user?.id;
   const { data: profile } = await supabase
     .from('profiles')
     .select('org_id, role')
     .eq('user_id', userId)
     .single();
   
   const orgId = profile?.org_id;
   ```

2. **From JWT Claims** (if set):
   ```typescript
   const orgId = event.user?.app_metadata?.org_id || 
                 event.user?.user_metadata?.org_id;
   ```

3. **Service Role Access** (for cron/background jobs):
   - Use Supabase service role key (server-side only)
   - Query `orgs` table directly or use org-specific configuration
   - Document which Edge Functions require service role and why

### 2.4 Cross-Org Data Isolation Rules

**Enforcement Strategy**:

1. **Database Level**:
   - All dispatch tables include `org_id` column
   - Foreign keys enforce `org_id` consistency where applicable
   - RLS policies enforce `org_id` filtering on all queries

2. **Application Level**:
   - All API endpoints validate `org_id` matches authenticated user's org
   - Edge Functions explicitly filter by `org_id` in all queries
   - Frontend never exposes `org_id` as a user-controllable parameter

3. **Query Pattern Example**:
   ```sql
   -- Correct: Always filter by org_id
   SELECT * FROM dispatch_jobs 
   WHERE org_id = current_setting('app.current_org_id')::uuid
     AND status = 'scheduled';
   
   -- Incorrect: Missing org_id filter (will be blocked by RLS)
   SELECT * FROM dispatch_jobs WHERE status = 'scheduled';
   ```

### 2.5 Service Role Access Pattern

**Use Case**: Background jobs, cron processors, and system-level operations that need to operate across organizations or without user context.

**Pattern**:
- Use Supabase service role key (stored as environment variable, never exposed to frontend)
- Document which operations require service role:
  - Notification processor cron jobs
  - Calendar sync background jobs
  - Bulk optimization runs (if cross-org)
- Service role operations must still respect `org_id` boundaries unless explicitly documented as cross-org

**Example**:
```typescript
// Edge Function using service role
const supabaseAdmin = createClient(
  Deno.env.get('SUPABASE_URL')!,
  Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')! // Service role key
);

// Still filter by org_id for multi-tenant operations
const { data } = await supabaseAdmin
  .from('job_notifications')
  .select('*')
  .eq('org_id', targetOrgId) // Explicit org_id filter
  .eq('status', 'pending')
  .lte('scheduled_send_at', new Date().toISOString());
```

---

## 3. Role Definitions and Permission Matrix

### 3.1 Role Enumeration

The Dispatch module defines the following roles (aligned with `fdd_2.md` §2.3):

#### 3.1.1 `admin`
- **Description**: Full platform access, including dispatch configuration and all scheduling operations
- **Typical Users**: Company owners, operations managers, system administrators
- **Scope**: Organization-wide

#### 3.1.2 `dispatcher`
- **Description**: Primary scheduling operator, manages daily dispatch operations
- **Typical Users**: Dispatch coordinators, scheduling staff
- **Scope**: Organization-wide scheduling operations

#### 3.1.3 `technician`
- **Description**: Field technician who receives assignments and updates job status
- **Typical Users**: Field service technicians
- **Scope**: Self-scoped (own assignments, routes, schedule)

#### 3.1.4 `csr` (Customer Service Representative)
- **Description**: Office staff who can create jobs and view schedules but with limited modification rights
- **Typical Users**: Customer service representatives, office staff
- **Scope**: Job creation and read-only schedule access

#### 3.1.5 `customer` (Portal User)
- **Description**: External customer accessing via portal (read-only appointment access)
- **Typical Users**: End customers booking appointments
- **Scope**: Own appointments only (via portal-specific APIs/views)

**Note**: Role definitions may be extended in the future. The permission matrix below defines current scope.

### 3.2 Permission Matrix Specification

The following matrix defines permissions for each role across dispatch entities. Permissions are:
- **CRUD**: Create, Read, Update, Delete
- **R**: Read-only
- **RU**: Read and Update (no create/delete)
- **N**: No access
- **C**: Conditional (documented in notes)

#### 3.2.1 Technician Profiles (`technician_profiles`)

| Role | Create | Read | Update | Delete | Notes |
|------|--------|------|--------|--------|-------|
| `admin` | ✅ | ✅ | ✅ | ✅ | Full access |
| `dispatcher` | ✅ | ✅ | ✅ | ✅ | Full access |
| `technician` | ❌ | ✅ (self) | ❌ | ❌ | Can read own profile only |
| `csr` | ❌ | ✅ | ❌ | ❌ | Read-only |
| `customer` | ❌ | ❌ | ❌ | ❌ | No access |

**Implementation Notes**:
- Technicians can read their own profile via `technician_profiles.user_id = auth.uid()`
- Updates to technician profiles (skills, zones, shifts) are restricted to admin/dispatcher

#### 3.2.2 Technician Skills (`technician_skills`)

| Role | Create | Read | Update | Delete | Notes |
|------|--------|------|--------|--------|-------|
| `admin` | ✅ | ✅ | ✅ | ✅ | Full access |
| `dispatcher` | ✅ | ✅ | ✅ | ✅ | Full access |
| `technician` | ❌ | ✅ (self) | ❌ | ❌ | Can read own skills only |
| `csr` | ❌ | ✅ | ❌ | ❌ | Read-only |
| `customer` | ❌ | ❌ | ❌ | ❌ | No access |

#### 3.2.3 Service Zones (`service_zones`)

| Role | Create | Read | Update | Delete | Notes |
|------|--------|------|--------|--------|-------|
| `admin` | ✅ | ✅ | ✅ | ✅ | Full access |
| `dispatcher` | ✅ | ✅ | ✅ | ✅ | Full access |
| `technician` | ❌ | ✅ | ❌ | ❌ | Read-only (needed for route context) |
| `csr` | ❌ | ✅ | ❌ | ❌ | Read-only |
| `customer` | ❌ | ❌ | ❌ | ❌ | No access |

#### 3.2.4 Technician Service Zones (`technician_service_zones`)

| Role | Create | Read | Update | Delete | Notes |
|------|--------|------|--------|--------|-------|
| `admin` | ✅ | ✅ | ✅ | ✅ | Full access |
| `dispatcher` | ✅ | ✅ | ✅ | ✅ | Full access |
| `technician` | ❌ | ✅ (self) | ❌ | ❌ | Can read own zone assignments only |
| `csr` | ❌ | ✅ | ❌ | ❌ | Read-only |
| `customer` | ❌ | ❌ | ❌ | ❌ | No access |

#### 3.2.5 Technician Shifts (`technician_shifts`)

| Role | Create | Read | Update | Delete | Notes |
|------|--------|------|--------|--------|-------|
| `admin` | ✅ | ✅ | ✅ | ✅ | Full access |
| `dispatcher` | ✅ | ✅ | ✅ | ✅ | Full access |
| `technician` | ❌ | ✅ (self) | ❌ | ❌ | Can read own shifts only |
| `csr` | ❌ | ✅ | ❌ | ❌ | Read-only (for scheduling context) |
| `customer` | ❌ | ❌ | ❌ | ❌ | No access |

#### 3.2.6 Technician Time Off (`technician_time_off`)

| Role | Create | Read | Update | Delete | Notes |
|------|--------|------|--------|--------|-------|
| `admin` | ✅ | ✅ | ✅ | ✅ | Full access |
| `dispatcher` | ✅ | ✅ | ✅ | ✅ | Full access |
| `technician` | ✅ (self) | ✅ (self) | ✅ (self) | ✅ (self) | Can manage own time-off requests |
| `csr` | ❌ | ✅ | ❌ | ❌ | Read-only |
| `customer` | ❌ | ❌ | ❌ | ❌ | No access |

**Implementation Notes**:
- Technicians can create/update/delete their own time-off records
- Approval workflow may be added later (currently self-service)

#### 3.2.7 Dispatch Jobs (`dispatch_jobs`)

| Role | Create | Read | Update | Delete | Notes |
|------|--------|------|--------|--------|-------|
| `admin` | ✅ | ✅ | ✅ | ✅ | Full access |
| `dispatcher` | ✅ | ✅ | ✅ | ✅ | Full access |
| `technician` | ❌ | ✅ (assigned) | ❌ | ❌ | Can read jobs they are assigned to |
| `csr` | ✅ | ✅ | ✅ (limited) | ❌ | Can create and update non-critical fields |
| `customer` | ❌ | ✅ (own) | ❌ | ❌ | Portal: read own jobs only |

**Implementation Notes**:
- CSR updates may be restricted to non-scheduling fields (e.g., notes, description) but not status/priority/assignments
- Customer access is via portal-specific APIs/views, not direct table access

#### 3.2.8 Job Time Windows (`job_time_windows`)

| Role | Create | Read | Update | Delete | Notes |
|------|--------|------|--------|--------|-------|
| `admin` | ✅ | ✅ | ✅ | ✅ | Full access |
| `dispatcher` | ✅ | ✅ | ✅ | ✅ | Full access |
| `technician` | ❌ | ✅ (assigned) | ❌ | ❌ | Can read windows for assigned jobs |
| `csr` | ✅ | ✅ | ✅ | ❌ | Can create/update windows |
| `customer` | ❌ | ✅ (own) | ✅ (select) | ❌ | Portal: can select from available windows |

**Implementation Notes**:
- Customer can update `is_selected` flag for their own jobs (via portal API)
- Selection logic must ensure only one window is selected per job

#### 3.2.9 Job Assignments (`job_assignments`)

| Role | Create | Read | Update | Delete | Notes |
|------|--------|------|--------|--------|-------|
| `admin` | ✅ | ✅ | ✅ | ✅ | Full access |
| `dispatcher` | ✅ | ✅ | ✅ | ✅ | Full access |
| `technician` | ❌ | ✅ (self) | ✅ (status/ETA) | ❌ | Can update status and ETA only |
| `csr` | ❌ | ✅ | ❌ | ❌ | Read-only |
| `customer` | ❌ | ✅ (own) | ❌ | ❌ | Portal: read own assignments only |

**Implementation Notes**:
- Technicians can update:
  - `status` (with state machine validation)
  - `tech_eta_at` (ETA updates)
- Technicians cannot update:
  - `scheduled_start_at`, `scheduled_end_at` (dispatcher-only)
  - `technician_id` (reassignment)
  - `assigned_by_user_id`

#### 3.2.10 Route Plans (`route_plans`)

| Role | Create | Read | Update | Delete | Notes |
|------|--------|------|--------|--------|-------|
| `admin` | ✅ | ✅ | ✅ | ✅ | Full access |
| `dispatcher` | ✅ | ✅ | ✅ | ✅ | Full access |
| `technician` | ❌ | ✅ (self) | ❌ | ❌ | Can read own route plans only |
| `csr` | ❌ | ✅ | ❌ | ❌ | Read-only |
| `customer` | ❌ | ❌ | ❌ | ❌ | No access |

#### 3.2.11 Route Stops (`route_stops`)

| Role | Create | Read | Update | Delete | Notes |
|------|--------|------|--------|--------|-------|
| `admin` | ✅ | ✅ | ✅ | ✅ | Full access |
| `dispatcher` | ✅ | ✅ | ✅ | ✅ | Full access |
| `technician` | ❌ | ✅ (own route) | ✅ (actual times) | ❌ | Can update actual arrival/departure times |
| `csr` | ❌ | ✅ | ❌ | ❌ | Read-only |
| `customer` | ❌ | ❌ | ❌ | ❌ | No access |

**Implementation Notes**:
- Technicians can update `actual_arrival_at` and `actual_departure_at` for their own route stops
- Cannot modify `planned_arrival_at`, `planned_departure_at`, or `sequence`

#### 3.2.12 Calendar Integrations (`calendar_integrations`)

| Role | Create | Read | Update | Delete | Notes |
|------|--------|------|--------|--------|-------|
| `admin` | ✅ | ✅ (limited) | ✅ | ✅ | Full access, but tokens not readable |
| `dispatcher` | ✅ (self) | ✅ (self) | ✅ (self) | ✅ (self) | Own calendar connections only |
| `technician` | ✅ (self) | ✅ (self) | ✅ (self) | ✅ (self) | Own calendar connections only |
| `csr` | ❌ | ❌ | ❌ | ❌ | No access |
| `customer` | ❌ | ❌ | ❌ | ❌ | No access |

**Implementation Notes**:
- Tokens (`access_token`, `refresh_token`) are encrypted and never exposed via RLS or API responses
- Users can only manage their own calendar integrations
- Admin can view integration status but not tokens

#### 3.2.13 Calendar Events (`calendar_events`)

| Role | Create | Read | Update | Delete | Notes |
|------|--------|------|--------|--------|-------|
| `admin` | ✅ | ✅ | ✅ | ✅ | Full access |
| `dispatcher` | ✅ | ✅ | ✅ | ✅ | Full access |
| `technician` | ❌ | ✅ (own) | ❌ | ❌ | Can read own calendar events |
| `csr` | ❌ | ✅ | ❌ | ❌ | Read-only |
| `customer` | ❌ | ❌ | ❌ | ❌ | No access |

#### 3.2.14 Job Notifications (`job_notifications`)

| Role | Create | Read | Update | Delete | Notes |
|------|--------|------|--------|--------|-------|
| `admin` | ✅ | ✅ | ✅ | ✅ | Full access |
| `dispatcher` | ✅ | ✅ | ✅ | ✅ | Full access |
| `technician` | ❌ | ✅ (related) | ❌ | ❌ | Can read notifications for their assignments |
| `csr` | ❌ | ✅ | ❌ | ❌ | Read-only |
| `customer` | ❌ | ✅ (own) | ❌ | ❌ | Portal: read own job notifications |

**Implementation Notes**:
- Notifications are typically system-generated
- Manual creation is admin/dispatcher only
- Customers can read notifications for their own jobs via portal

### 3.3 Permission Matrix Summary Table

| Entity | Admin | Dispatcher | Technician | CSR | Customer |
|--------|-------|------------|------------|-----|----------|
| **Technician Config** |
| `technician_profiles` | CRUD | CRUD | R (self) | R | N |
| `technician_skills` | CRUD | CRUD | R (self) | R | N |
| `technician_service_zones` | CRUD | CRUD | R (self) | R | N |
| `technician_shifts` | CRUD | CRUD | R (self) | R | N |
| `technician_time_off` | CRUD | CRUD | CRUD (self) | R | N |
| **Jobs & Scheduling** |
| `dispatch_jobs` | CRUD | CRUD | R (assigned) | CRU | R (own) |
| `job_time_windows` | CRUD | CRUD | R (assigned) | CRU | RU (select) |
| `job_assignments` | CRUD | CRUD | RU (status/ETA) | R | R (own) |
| **Routing** |
| `route_plans` | CRUD | CRUD | R (self) | R | N |
| `route_stops` | CRUD | CRUD | RU (actual times) | R | N |
| **Calendar** |
| `calendar_integrations` | CRUD | CRUD (self) | CRUD (self) | N | N |
| `calendar_events` | CRUD | CRUD | R (own) | R | N |
| **Notifications** |
| `job_notifications` | CRUD | CRUD | R (related) | R | R (own) |

**Legend**: C=Create, R=Read, U=Update, D=Delete, (self)=own records only, (assigned)=assigned jobs only, (own)=customer's own records

### 3.4 Role Resolution Pattern

**Implementation**: Roles are stored in the `profiles` table (`profiles.role` column).

**Resolution in RLS Policies**:
```sql
-- Helper function to get user's role
CREATE OR REPLACE FUNCTION get_user_role()
RETURNS TEXT AS $$
  SELECT role FROM profiles WHERE user_id = auth.uid();
$$ LANGUAGE sql SECURITY DEFINER;

-- Helper function to get user's org_id
CREATE OR REPLACE FUNCTION get_user_org_id()
RETURNS UUID AS $$
  SELECT org_id FROM profiles WHERE user_id = auth.uid();
$$ LANGUAGE sql SECURITY DEFINER;
```

**Usage in RLS Policies**:
```sql
-- Example: Dispatcher can read all jobs in their org
CREATE POLICY "dispatchers_read_jobs"
ON dispatch_jobs FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher', 'csr')
);

-- Example: Technician can read own assignments
CREATE POLICY "technicians_read_own_assignments"
ON job_assignments FOR SELECT
USING (
  org_id = get_user_org_id() AND
  (
    get_user_role() IN ('admin', 'dispatcher') OR
    (get_user_role() = 'technician' AND technician_id IN (
      SELECT id FROM technician_profiles WHERE user_id = auth.uid()
    ))
  )
);
```

### 3.5 Edge Function Authorization Pattern

**Pattern**: Edge Functions must validate user role and org_id before performing operations.

**Example**:
```typescript
// Edge Function authorization helper
async function authorizeUser(
  supabase: SupabaseClient,
  userId: string,
  requiredRoles: string[]
): Promise<{ orgId: string; role: string } | null> {
  const { data: profile, error } = await supabase
    .from('profiles')
    .select('org_id, role')
    .eq('user_id', userId)
    .single();
  
  if (error || !profile) {
    return null;
  }
  
  if (!requiredRoles.includes(profile.role)) {
    return null;
  }
  
  return { orgId: profile.org_id, role: profile.role };
}

// Usage in Edge Function
const auth = await authorizeUser(supabase, event.user.id, ['admin', 'dispatcher']);
if (!auth) {
  return new Response(JSON.stringify({ error: 'Unauthorized' }), { status: 403 });
}

// Use auth.orgId for all queries
const { data } = await supabase
  .from('dispatch_jobs')
  .select('*')
  .eq('org_id', auth.orgId);
```

---

## 4. Database Schema Requirements

### 4.1 Required Tables (Dependencies)

Before implementing dispatch tables, ensure these tables exist:

1. **`orgs`** (from CRM module or create if missing)
2. **`profiles`** (user-to-org mapping with roles)
3. **`auth.users`** (Supabase Auth table)

### 4.2 Helper Functions

Create these database functions to support RLS policies:

#### 4.2.1 `get_user_org_id()`

```sql
CREATE OR REPLACE FUNCTION get_user_org_id()
RETURNS UUID AS $$
  SELECT org_id FROM profiles WHERE user_id = auth.uid();
$$ LANGUAGE sql SECURITY DEFINER STABLE;

-- Grant execute to authenticated users
GRANT EXECUTE ON FUNCTION get_user_org_id() TO authenticated;
```

**Purpose**: Returns the authenticated user's `org_id` for use in RLS policies.

**Security**: `SECURITY DEFINER` allows the function to access `profiles` table even if RLS would otherwise block it. `STABLE` indicates the function doesn't modify data.

#### 4.2.2 `get_user_role()`

```sql
CREATE OR REPLACE FUNCTION get_user_role()
RETURNS TEXT AS $$
  SELECT role FROM profiles WHERE user_id = auth.uid();
$$ LANGUAGE sql SECURITY DEFINER STABLE;

-- Grant execute to authenticated users
GRANT EXECUTE ON FUNCTION get_user_role() TO authenticated;
```

**Purpose**: Returns the authenticated user's role for use in RLS policies.

#### 4.2.3 `is_user_technician(technician_profile_id UUID)`

```sql
CREATE OR REPLACE FUNCTION is_user_technician(technician_profile_id UUID)
RETURNS BOOLEAN AS $$
  SELECT EXISTS (
    SELECT 1 FROM technician_profiles
    WHERE id = technician_profile_id
      AND user_id = auth.uid()
      AND org_id = get_user_org_id()
  );
$$ LANGUAGE sql SECURITY DEFINER STABLE;

-- Grant execute to authenticated users
GRANT EXECUTE ON FUNCTION is_user_technician(UUID) TO authenticated;
```

**Purpose**: Checks if the authenticated user is the technician associated with a given `technician_profiles.id`.

**Usage**: Used in RLS policies to allow technicians to access their own records.

### 4.3 RLS Policy Template Pattern

All dispatch tables will follow this RLS pattern:

1. **Enable RLS**: `ALTER TABLE <table> ENABLE ROW LEVEL SECURITY;`
2. **Tenant Isolation Policy**: Always filter by `org_id`
3. **Role-Based Policies**: Separate policies for each role's permissions

**Example Template**:
```sql
-- Enable RLS
ALTER TABLE <table_name> ENABLE ROW LEVEL SECURITY;

-- Tenant isolation + admin/dispatcher full access
CREATE POLICY "<table>_admin_dispatcher_full"
ON <table_name> FOR ALL
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
);

-- Technician self-scoped access (if applicable)
CREATE POLICY "<table>_technician_self"
ON <table_name> FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'technician' AND
  -- Add technician-specific filter here
);

-- CSR read-only access (if applicable)
CREATE POLICY "<table>_csr_read"
ON <table_name> FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() = 'csr'
);
```

---

## 5. Edge Function Conventions

### 5.1 `org_id` Extraction Pattern

All Edge Functions must extract and validate `org_id`:

```typescript
// Standard pattern for Edge Functions
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

Deno.serve(async (req) => {
  const authHeader = req.headers.get('Authorization');
  if (!authHeader) {
    return new Response(JSON.stringify({ error: 'Unauthorized' }), { status: 401 });
  }

  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_ANON_KEY')!,
    { global: { headers: { Authorization: authHeader } } }
  );

  // Get authenticated user
  const { data: { user }, error: userError } = await supabase.auth.getUser();
  if (userError || !user) {
    return new Response(JSON.stringify({ error: 'Unauthorized' }), { status: 401 });
  }

  // Resolve org_id from profile
  const { data: profile, error: profileError } = await supabase
    .from('profiles')
    .select('org_id, role')
    .eq('user_id', user.id)
    .single();

  if (profileError || !profile) {
    return new Response(JSON.stringify({ error: 'User profile not found' }), { status: 403 });
  }

  const orgId = profile.org_id;
  const userRole = profile.role;

  // Use orgId for all subsequent queries
  // ...
});
```

### 5.2 Role Validation Pattern

```typescript
// Validate role before proceeding
function requireRole(role: string, allowedRoles: string[]): boolean {
  return allowedRoles.includes(role);
}

// Usage
if (!requireRole(userRole, ['admin', 'dispatcher'])) {
  return new Response(JSON.stringify({ error: 'Insufficient permissions' }), { status: 403 });
}
```

### 5.3 Service Role Pattern (Cron Jobs)

```typescript
// For background jobs that run without user context
const supabaseAdmin = createClient(
  Deno.env.get('SUPABASE_URL')!,
  Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')! // Service role key
);

// Still filter by org_id when processing org-specific data
const orgIds = await getActiveOrgIds(supabaseAdmin); // Helper to get all orgs

for (const orgId of orgIds) {
  // Process notifications for this org
  const { data } = await supabaseAdmin
    .from('job_notifications')
    .select('*')
    .eq('org_id', orgId)
    .eq('status', 'pending')
    .lte('scheduled_send_at', new Date().toISOString());
  
  // Process notifications...
}
```

---

## 6. Frontend Conventions

### 6.1 `org_id` Handling

**Rule**: Frontend never sends `org_id` as a user-controllable parameter. It is always derived server-side.

**Pattern**:
```typescript
// Frontend: Use Supabase client (org_id is handled by RLS)
const { data } = await supabase
  .from('dispatch_jobs')
  .select('*')
  .eq('status', 'scheduled');
// RLS automatically filters by user's org_id

// Edge Function calls: No org_id in request body
const response = await fetch('/functions/v1/auto_schedule_job', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${session.access_token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    job_id: jobId // No org_id here
  })
});
```

### 6.2 Role-Based UI Rendering

```typescript
// Get user role from profile
const { data: profile } = await supabase
  .from('profiles')
  .select('role')
  .eq('user_id', user.id)
  .single();

const userRole = profile?.role;

// Conditionally render UI based on role
{userRole === 'admin' || userRole === 'dispatcher' ? (
  <Button onClick={handleOptimizeRoute}>Optimize Route</Button>
) : null}

{userRole === 'technician' ? (
  <TechnicianDashboard />
) : (
  <DispatcherDashboard />
)}
```

---

## 7. Integration Points

### 7.1 CRM Module Integration

- **Shared `orgs` table**: Dispatch reuses CRM's `orgs` table
- **Shared `profiles` table**: User-to-org mapping is shared
- **Customer/Location references**: `dispatch_jobs` references `customers` and `customer_locations` from CRM

### 7.2 Auth Module Integration

- **Supabase Auth**: All users authenticated via `auth.users`
- **JWT Claims**: Custom claims can be set for `org_id` and `role` (optional optimization)
- **Profile Table**: Links `auth.users.id` to `orgs.id` and role

### 7.3 Future Module Integrations

- **Work Orders**: `dispatch_jobs.related_work_order_id` will reference work order table
- **Portal**: Customer role access via portal-specific APIs (not direct table access)
- **Mobile App**: Technician role access via mobile APIs with same RLS policies

---

## 8. Documentation Requirements

### 8.1 Developer Documentation

Create documentation covering:

1. **Tenancy Model**:
   - How `org_id` is derived
   - How to query data scoped to user's org
   - How to use service role for background jobs

2. **Role System**:
   - Role definitions and use cases
   - Permission matrix reference
   - How to add new roles (if needed)

3. **RLS Policy Patterns**:
   - Template for creating new RLS policies
   - Helper functions reference
   - Testing RLS policies

4. **Edge Function Patterns**:
   - Authorization pattern
   - `org_id` extraction pattern
   - Service role usage

### 8.2 API Documentation

Document:
- All endpoints require authentication
- `org_id` is never a request parameter (derived server-side)
- Role requirements for each endpoint
- Error responses for unauthorized/forbidden access

### 8.3 Security Documentation

Document:
- RLS policy coverage (all tables have RLS enabled)
- Service role key security (never expose to frontend)
- Token encryption for calendar integrations
- Customer portal access patterns (separate from internal APIs)

---

## 9. Testing Requirements

### 9.1 Multi-Tenancy Tests

**Test Cases**:

1. **Cross-Org Isolation**:
   - User from Org A cannot see data from Org B
   - Queries filtered by `org_id` return only user's org data
   - RLS policies block cross-org access

2. **`org_id` Resolution**:
   - User profile correctly resolves `org_id`
   - Edge Functions correctly extract `org_id`
   - Service role operations respect `org_id` boundaries

### 9.2 Role-Based Access Tests

**Test Cases**:

1. **Admin/Dispatcher Access**:
   - Can CRUD all dispatch entities in their org
   - Cannot access other orgs' data

2. **Technician Access**:
   - Can read own assignments, routes, shifts
   - Can update own assignment status and ETA
   - Cannot modify other technicians' data
   - Cannot create/delete jobs or assignments

3. **CSR Access**:
   - Can create jobs and read schedules
   - Cannot modify assignments or routes
   - Cannot modify technician configuration

4. **Customer Access** (via portal):
   - Can read own jobs and appointments
   - Cannot access other customers' data
   - Cannot modify scheduling data

### 9.3 RLS Policy Tests

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

### 9.4 Edge Function Authorization Tests

**Test Cases**:

1. **Authentication**:
   - Unauthenticated requests are rejected
   - Invalid tokens are rejected

2. **Authorization**:
   - Role checks work correctly
   - `org_id` extraction works correctly
   - Unauthorized roles are rejected

3. **Service Role**:
   - Service role can access data (for cron jobs)
   - Service role operations respect `org_id` boundaries

---

## 10. Implementation Checklist

### Story DISP-001: Confirm Dispatch Tenancy Model and `org_id` Resolution

- [ ] **Document Tenancy Decision**:
  - [ ] Document reuse of `orgs` table from CRM module
  - [ ] Document `org_id` column convention for all dispatch tables
  - [ ] Document `org_id` derivation pattern (profile-based vs JWT claims)

- [ ] **Implement Helper Functions**:
  - [ ] Create `get_user_org_id()` function
  - [ ] Create `get_user_role()` function
  - [ ] Create `is_user_technician()` function (if needed)
  - [ ] Grant execute permissions to authenticated users

- [ ] **Document Edge Function Patterns**:
  - [ ] Document `org_id` extraction pattern for Edge Functions
  - [ ] Document service role usage pattern for cron jobs
  - [ ] Provide code examples

- [ ] **Document Frontend Patterns**:
  - [ ] Document that frontend never sends `org_id` as parameter
  - [ ] Document RLS automatic filtering

- [ ] **Create Documentation**:
  - [ ] Tenancy model documentation in repo docs
  - [ ] `org_id` resolution guide
  - [ ] Integration guide for other modules

- [ ] **Validation**:
  - [ ] Review tenancy conventions against CRM module for consistency
  - [ ] Verify helper functions work correctly
  - [ ] Test `org_id` resolution in Edge Function context

### Story DISP-002: Define Dispatch Roles and Permission Matrix

- [ ] **Define Roles**:
  - [ ] Document `admin` role definition
  - [ ] Document `dispatcher` role definition
  - [ ] Document `technician` role definition
  - [ ] Document `csr` role definition
  - [ ] Document `customer` role definition (portal access)

- [ ] **Create Permission Matrix**:
  - [ ] Define permissions for `technician_profiles`
  - [ ] Define permissions for `technician_skills`
  - [ ] Define permissions for `service_zones`
  - [ ] Define permissions for `technician_service_zones`
  - [ ] Define permissions for `technician_shifts`
  - [ ] Define permissions for `technician_time_off`
  - [ ] Define permissions for `dispatch_jobs`
  - [ ] Define permissions for `job_time_windows`
  - [ ] Define permissions for `job_assignments`
  - [ ] Define permissions for `route_plans`
  - [ ] Define permissions for `route_stops`
  - [ ] Define permissions for `calendar_integrations`
  - [ ] Define permissions for `calendar_events`
  - [ ] Define permissions for `job_notifications`

- [ ] **Document Permission Matrix**:
  - [ ] Create Markdown table with all permissions
  - [ ] Document implementation notes for each entity
  - [ ] Document conditional permissions (e.g., technician self-scoped)

- [ ] **Create RLS Policy Templates**:
  - [ ] Template for admin/dispatcher full access
  - [ ] Template for technician self-scoped access
  - [ ] Template for CSR read/create access
  - [ ] Template for customer portal access (if applicable)

- [ ] **Document Edge Function Authorization**:
  - [ ] Document role validation pattern
  - [ ] Provide code examples for authorization checks

- [ ] **Create Documentation**:
  - [ ] Permission matrix document (Markdown)
  - [ ] Link from `fdd_2_agile.md`
  - [ ] RLS policy creation guide

- [ ] **Validation**:
  - [ ] Review permission matrix against all stories in `fdd_2_agile.md`
  - [ ] Verify no missing permission areas
  - [ ] Validate permission matrix covers all dispatch entities

---

## 11. Migration Strategy

### 11.1 Prerequisites

Before implementing Epic 1, ensure:

1. **`orgs` table exists**:
   ```sql
   -- Verify or create orgs table
   CREATE TABLE IF NOT EXISTS orgs (
     id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
     name TEXT NOT NULL,
     created_at TIMESTAMPTZ NOT NULL DEFAULT now()
   );
   ```

2. **`profiles` table exists**:
   ```sql
   -- Verify or create profiles table
   CREATE TABLE IF NOT EXISTS profiles (
     id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
     user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
     org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
     role TEXT NOT NULL,
     display_name TEXT,
     created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
     updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
     CONSTRAINT profiles_user_id_unique UNIQUE (user_id)
   );
   
   CREATE INDEX IF NOT EXISTS idx_profiles_user_id ON profiles(user_id);
   CREATE INDEX IF NOT EXISTS idx_profiles_org_id ON profiles(org_id);
   ```

### 11.2 Migration Order

1. **Create helper functions** (if `profiles` table exists):
   - `get_user_org_id()`
   - `get_user_role()`
   - `is_user_technician()` (optional, for later use)

2. **Document conventions** (no database changes):
   - Tenancy model documentation
   - Permission matrix documentation

### 11.3 Migration Files

Recommended migration file structure:

```
supabase/migrations/
  20240101000000_create_dispatch_helper_functions.sql
```

**Note**: Epic 1 does not create dispatch tables. It only establishes helper functions and documentation. Tables will be created in Epic 2.

---

## 12. Seed Data Requirements

### 12.1 Test Organizations

Create at least two test organizations for multi-tenancy validation:

```sql
INSERT INTO orgs (id, name) VALUES
  ('00000000-0000-0000-0000-000000000001', 'Test HVAC Company A'),
  ('00000000-0000-0000-0000-000000000002', 'Test HVAC Company B')
ON CONFLICT (id) DO NOTHING;
```

### 12.2 Test Users and Profiles

Create test users with different roles:

```sql
-- Note: Actual user creation happens via Supabase Auth API
-- Profiles are created after users exist

-- Admin user for Org A
INSERT INTO profiles (user_id, org_id, role, display_name) VALUES
  ('<admin-user-id-from-auth>', '00000000-0000-0000-0000-000000000001', 'admin', 'Admin User A')
ON CONFLICT (user_id) DO NOTHING;

-- Dispatcher user for Org A
INSERT INTO profiles (user_id, org_id, role, display_name) VALUES
  ('<dispatcher-user-id-from-auth>', '00000000-0000-0000-0000-000000000001', 'dispatcher', 'Dispatcher User A')
ON CONFLICT (user_id) DO NOTHING;

-- Technician user for Org A
INSERT INTO profiles (user_id, org_id, role, display_name) VALUES
  ('<technician-user-id-from-auth>', '00000000-0000-0000-0000-000000000001', 'technician', 'Technician User A')
ON CONFLICT (user_id) DO NOTHING;

-- CSR user for Org A
INSERT INTO profiles (user_id, org_id, role, display_name) VALUES
  ('<csr-user-id-from-auth>', '00000000-0000-0000-0000-000000000001', 'csr', 'CSR User A')
ON CONFLICT (user_id) DO NOTHING;

-- Admin user for Org B (for cross-org isolation testing)
INSERT INTO profiles (user_id, org_id, role, display_name) VALUES
  ('<admin-user-id-org-b-from-auth>', '00000000-0000-0000-0000-000000000002', 'admin', 'Admin User B')
ON CONFLICT (user_id) DO NOTHING;
```

**Note**: User IDs must be obtained from Supabase Auth after creating users via the Auth API or dashboard.

---

## 13. Performance Considerations

### 13.1 Helper Function Performance

- **`get_user_org_id()` and `get_user_role()`**: Called frequently in RLS policies
- **Optimization**: Ensure `profiles.user_id` has an index (created above)
- **Caching**: Consider caching user role/org_id in JWT claims for Edge Functions (optional optimization)

### 13.2 RLS Policy Performance

- **Index Usage**: Ensure RLS policies use indexed columns (`org_id`, `user_id`)
- **Policy Complexity**: Keep policies simple; avoid complex subqueries where possible
- **Testing**: Use `EXPLAIN ANALYZE` to verify RLS policies don't degrade query performance

---

## 14. Security Considerations

### 14.1 RLS Enforcement

- **Requirement**: All dispatch tables MUST have RLS enabled
- **Validation**: Test that RLS blocks unauthorized access
- **Documentation**: Document which tables have RLS and which don't (if any exceptions)

### 14.2 Helper Function Security

- **`SECURITY DEFINER`**: Helper functions use `SECURITY DEFINER` to bypass RLS on `profiles` table
- **Risk**: Ensure helper functions don't expose sensitive data
- **Mitigation**: Functions only return `org_id` and `role`, not full profile data

### 14.3 Service Role Key Security

- **Storage**: Service role key stored as environment variable, never in code or frontend
- **Access**: Only Edge Functions running server-side can access service role key
- **Documentation**: Document which Edge Functions use service role and why

---

## 15. Open Questions and Future Considerations

### 15.1 JWT Claims Optimization

**Question**: Should `org_id` and `role` be stored in JWT claims for performance?

**Decision**: Initially use database lookup via helper functions. If performance becomes an issue, add JWT claims via Auth hooks.

**Implementation Note**: JWT claims can be set via database triggers on `profiles` table updates or Auth hooks.

### 15.2 Role Hierarchy

**Question**: Should roles have a hierarchy (e.g., admin inherits dispatcher permissions)?

**Decision**: Initially implement explicit role checks. If hierarchy is needed later, add helper function `has_permission(required_role, user_role)`.

### 15.3 Multi-Org Users

**Question**: Can a user belong to multiple organizations?

**Decision**: Initially assume one user = one org (via `profiles.user_id` unique constraint). If multi-org support is needed, modify `profiles` to allow multiple rows per user and add `org_id` selection in UI.

---

## 16. Appendix: Complete SQL Migration Script

### 16.1 Helper Functions Migration

```sql
-- Migration: Create Dispatch Helper Functions for Epic 1
-- File: supabase/migrations/YYYYMMDDHHMMSS_create_dispatch_helper_functions.sql

-- Prerequisites: orgs and profiles tables must exist

-- Function: Get authenticated user's org_id
CREATE OR REPLACE FUNCTION get_user_org_id()
RETURNS UUID AS $$
  SELECT org_id FROM profiles WHERE user_id = auth.uid();
$$ LANGUAGE sql SECURITY DEFINER STABLE;

-- Grant execute to authenticated users
GRANT EXECUTE ON FUNCTION get_user_org_id() TO authenticated;

-- Function: Get authenticated user's role
CREATE OR REPLACE FUNCTION get_user_role()
RETURNS TEXT AS $$
  SELECT role FROM profiles WHERE user_id = auth.uid();
$$ LANGUAGE sql SECURITY DEFINER STABLE;

-- Grant execute to authenticated users
GRANT EXECUTE ON FUNCTION get_user_role() TO authenticated;

-- Function: Check if user is a specific technician (for self-scoped access)
CREATE OR REPLACE FUNCTION is_user_technician(technician_profile_id UUID)
RETURNS BOOLEAN AS $$
  SELECT EXISTS (
    SELECT 1 FROM technician_profiles
    WHERE id = technician_profile_id
      AND user_id = auth.uid()
      AND org_id = get_user_org_id()
  );
$$ LANGUAGE sql SECURITY DEFINER STABLE;

-- Grant execute to authenticated users
GRANT EXECUTE ON FUNCTION is_user_technician(UUID) TO authenticated;

-- Add comment for documentation
COMMENT ON FUNCTION get_user_org_id() IS 'Returns the authenticated user''s org_id for use in RLS policies';
COMMENT ON FUNCTION get_user_role() IS 'Returns the authenticated user''s role for use in RLS policies';
COMMENT ON FUNCTION is_user_technician(UUID) IS 'Checks if the authenticated user is the technician associated with the given technician_profile_id';
```

### 16.2 Rollback Migration

```sql
-- Rollback: Drop Dispatch Helper Functions
-- File: supabase/migrations/YYYYMMDDHHMMSS_drop_dispatch_helper_functions.sql

DROP FUNCTION IF EXISTS is_user_technician(UUID);
DROP FUNCTION IF EXISTS get_user_role();
DROP FUNCTION IF EXISTS get_user_org_id();
```

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 1 – Dispatch Multi-Tenancy, Roles, and Foundational Conventions. All specifications are designed to be directly consumable by LLM-based code generators, with exact patterns, conventions, and implementation details defined.

**Next Steps**: After completing Epic 1, proceed to Epic 2 (Core Scheduling Data Model) which will create the dispatch tables using the tenancy and role conventions established here.

