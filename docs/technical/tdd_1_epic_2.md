# Technical Design Document – Epic 2: Authentication, Authorization & RLS Policies

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 2 – Authentication, Authorization & RLS Policies
- **Source**: Derived from `fdd_1_agile.md` Epic 2 (Stories CRM-013 through CRM-015)
- **Reference Documents**: 
  - `fdd_1.md` (Functional Design Document for CRM, §2.3 and §7)
  - `fdd_1_agile.md` (Agile User Stories)
  - `tdd_1_epic_1.md` (CRM Core Data Model - prerequisite)
  - `functional.md` (Platform Capabilities)
  - `tooling.md` (Platform Choices: Supabase + Vercel)
- **Target Platform**: Supabase (PostgreSQL 15+ with Row-Level Security)
- **Purpose**: Comprehensive technical specification for LLM-based code generation
- **Prerequisites**: Epic 1 (CRM Core Data Model) must be completed first

---

## 1. Overview

This document provides complete technical specifications for implementing authentication, authorization, and Row-Level Security (RLS) policies for the CRM module in Supabase. It covers:

- User roles and profile schema integration with Supabase Auth
- Multi-tenancy enforcement via RLS base policies
- Role-based access control (RBAC) policies for all CRM tables
- Helper functions for JWT claims and org_id resolution
- Service role patterns for Edge Functions
- Testing strategies for security validation

All specifications are designed to be directly implementable via SQL migrations in Supabase, with exact policy definitions, helper functions, and security patterns defined.

---

## 2. User Roles & Profile Schema

### 2.1 Role Definitions

The CRM module defines the following user roles with hierarchical permissions:

| Role | Description | Permissions Level |
|------|-------------|-------------------|
| `admin` | System administrator | Full access to all CRM data and configuration within their org |
| `manager` | Operations manager | Full access to CRM data; can manage segments, automations, and preferences |
| `dispatcher` | Dispatch coordinator | Read/write access to customers, locations, interactions, follow-ups; read-only for segments/automations |
| `technician` | Field technician | Read-only access to limited customer data (name, location, contact); no access to preferences, segments, automations |
| `csr` / `sales` | Customer service rep / Sales | Full CRUD on customers, contacts, locations, interactions, follow-ups, tags; read-only for segments/automations |

**Note**: `csr` and `sales` are treated as equivalent roles with identical permissions.

### 2.2 Role Enum

**DDL**:

```sql
CREATE TYPE user_role_enum AS ENUM (
  'admin',
  'manager',
  'dispatcher',
  'technician',
  'csr',
  'sales'
);
```

**Usage**: `profiles.role`

**Business Rules**:
- All users must have exactly one role
- Role cannot be NULL
- Role changes require admin privileges (enforced in application logic or via RLS)

---

### 2.3 Profiles Table

**Purpose**: Links Supabase `auth.users` to organizations and roles. This table extends Supabase Auth with CRM-specific metadata.

**DDL**:

```sql
CREATE TABLE IF NOT EXISTS profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE RESTRICT,
  role user_role_enum NOT NULL,
  first_name TEXT,
  last_name TEXT,
  email TEXT, -- Denormalized from auth.users.email for convenience
  phone TEXT,
  avatar_url TEXT,
  is_active BOOLEAN NOT NULL DEFAULT true,
  last_login_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Constraints
  CONSTRAINT chk_profiles_email_format CHECK (
    email IS NULL OR email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
  )
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_profiles_org_id ON profiles(org_id);
CREATE INDEX IF NOT EXISTS idx_profiles_role ON profiles(role);
CREATE INDEX IF NOT EXISTS idx_profiles_org_id_role ON profiles(org_id, role);
CREATE INDEX IF NOT EXISTS idx_profiles_email ON profiles(email) WHERE email IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_profiles_is_active ON profiles(is_active) WHERE is_active = true;

-- Unique constraint: one profile per user (enforced by PK, but explicit for clarity)
-- Note: Supabase Auth ensures one auth.users record per user, so PK constraint is sufficient

-- Trigger to sync email from auth.users (optional, can be handled in application)
CREATE OR REPLACE FUNCTION sync_profile_email()
RETURNS TRIGGER AS $$
BEGIN
  -- Update profile email when auth.users email changes
  UPDATE profiles
  SET email = NEW.email, updated_at = now()
  WHERE id = NEW.id;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Note: Trigger on auth.users requires superuser privileges
-- Alternative: Handle email sync in application code or Edge Function

-- Trigger for updated_at
CREATE OR REPLACE FUNCTION update_profiles_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_profiles_updated_at
  BEFORE UPDATE ON profiles
  FOR EACH ROW
  EXECUTE FUNCTION update_profiles_updated_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | - | Primary key, FK to `auth.users.id` |
| `org_id` | UUID | NO | - | FK to `orgs.id` (user's organization) |
| `role` | user_role_enum | NO | - | User's role |
| `first_name` | TEXT | YES | NULL | User's first name |
| `last_name` | TEXT | YES | NULL | User's last name |
| `email` | TEXT | YES | NULL | Denormalized email (synced from `auth.users`) |
| `phone` | TEXT | YES | NULL | User's phone number |
| `avatar_url` | TEXT | YES | NULL | URL to user's avatar image |
| `is_active` | BOOLEAN | NO | `true` | Whether user account is active |
| `last_login_at` | TIMESTAMPTZ | YES | NULL | Last login timestamp |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Profile creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- One profile per user (enforced by PK)
- `org_id` cannot be deleted if users exist (RESTRICT on delete)
- Email format validation if provided
- `is_active = false` should block login (enforced in application logic or RLS)

**RLS on Profiles Table**:

```sql
-- Enable RLS
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

-- Users can read their own profile
CREATE POLICY "Users can read own profile"
  ON profiles
  FOR SELECT
  USING (auth.uid() = id);

-- Users can update their own profile (limited fields)
CREATE POLICY "Users can update own profile"
  ON profiles
  FOR UPDATE
  USING (auth.uid() = id)
  WITH CHECK (
    auth.uid() = id AND
    -- Prevent users from changing their own org_id or role
    (OLD.org_id = NEW.org_id) AND
    (OLD.role = NEW.role)
  );

-- Admins can read all profiles in their org
CREATE POLICY "Admins can read org profiles"
  ON profiles
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM profiles p
      WHERE p.id = auth.uid()
      AND p.org_id = profiles.org_id
      AND p.role IN ('admin', 'manager')
      AND p.is_active = true
    )
  );

-- Service role can bypass RLS (for Edge Functions)
-- Note: Service role access is automatic when using service_role key, no policy needed
```

---

### 2.4 Helper Functions for JWT Claims and Org Resolution

These functions extract `org_id` and `role` from the authenticated user's profile for use in RLS policies.

#### 2.4.1 `get_user_org_id()` Function

**Purpose**: Returns the `org_id` of the currently authenticated user.

**DDL**:

```sql
CREATE OR REPLACE FUNCTION get_user_org_id()
RETURNS UUID
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  user_org_id UUID;
BEGIN
  SELECT org_id INTO user_org_id
  FROM profiles
  WHERE id = auth.uid()
    AND is_active = true;
  
  RETURN user_org_id;
END;
$$;
```

**Usage**: `WHERE org_id = get_user_org_id()`

**Security Notes**:
- `SECURITY DEFINER` allows the function to access `profiles` table even if RLS would block direct access
- `STABLE` indicates the function returns the same result for the same input within a transaction
- Returns `NULL` if user is not authenticated or profile doesn't exist

#### 2.4.2 `get_user_role()` Function

**Purpose**: Returns the `role` of the currently authenticated user.

**DDL**:

```sql
CREATE OR REPLACE FUNCTION get_user_role()
RETURNS user_role_enum
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  user_role user_role_enum;
BEGIN
  SELECT role INTO user_role
  FROM profiles
  WHERE id = auth.uid()
    AND is_active = true;
  
  RETURN user_role;
END;
$$;
```

**Usage**: `WHERE get_user_role() IN ('admin', 'manager')`

**Security Notes**:
- Same security pattern as `get_user_org_id()`
- Returns `NULL` if user is not authenticated or profile doesn't exist

#### 2.4.3 `is_user_in_org(org_uuid UUID)` Function

**Purpose**: Checks if the authenticated user belongs to a specific organization.

**DDL**:

```sql
CREATE OR REPLACE FUNCTION is_user_in_org(org_uuid UUID)
RETURNS BOOLEAN
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  RETURN EXISTS (
    SELECT 1
    FROM profiles
    WHERE id = auth.uid()
      AND org_id = org_uuid
      AND is_active = true
  );
END;
$$;
```

**Usage**: `WHERE is_user_in_org(org_id)`

**Security Notes**:
- Useful for complex RLS policies that need to check org membership
- Returns `false` if user is not authenticated

#### 2.4.4 `has_role(required_roles user_role_enum[])` Function

**Purpose**: Checks if the authenticated user has one of the specified roles.

**DDL**:

```sql
CREATE OR REPLACE FUNCTION has_role(required_roles user_role_enum[])
RETURNS BOOLEAN
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  RETURN EXISTS (
    SELECT 1
    FROM profiles
    WHERE id = auth.uid()
      AND role = ANY(required_roles)
      AND is_active = true
  );
END;
$$;
```

**Usage**: `WHERE has_role(ARRAY['admin', 'manager']::user_role_enum[])`

**Security Notes**:
- Efficient for checking multiple roles
- Returns `false` if user is not authenticated or doesn't have required role

---

### 2.5 JWT Claims Integration

**Supabase JWT Structure**:

Supabase automatically includes user metadata in JWT claims. To include `org_id` and `role` in JWT for frontend use:

**Option 1: Custom Claims via Database Function** (Recommended for Edge Functions)

Create a function that sets custom claims:

```sql
CREATE OR REPLACE FUNCTION set_user_claims()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  user_profile profiles%ROWTYPE;
BEGIN
  SELECT * INTO user_profile
  FROM profiles
  WHERE id = NEW.id;
  
  -- Set custom claims in JWT (requires Supabase Auth configuration)
  -- This is typically handled via Supabase Auth hooks or Edge Functions
  -- For now, document that claims are read from profiles table via helper functions
  
  RETURN NEW;
END;
$$;
```

**Option 2: Read from Profiles Table** (Current Implementation)

Frontend and backend read `org_id` and `role` from `profiles` table using helper functions or direct queries (with RLS).

**Frontend Pattern** (Next.js):

```typescript
// Example: Get user org_id and role
const { data: profile } = await supabase
  .from('profiles')
  .select('org_id, role')
  .eq('id', user.id)
  .single();
```

**Backend Pattern** (Edge Functions):

```typescript
// Example: Get user org_id from JWT and profiles
const { data: { user } } = await supabase.auth.getUser();
const { data: profile } = await supabase
  .from('profiles')
  .select('org_id, role')
  .eq('id', user.id)
  .single();
```

---

## 3. RLS Base Policies for Multi-Tenancy

### 3.1 RLS Enablement

Enable RLS on all CRM tables that contain `org_id`. RLS is disabled by default in PostgreSQL; it must be explicitly enabled.

**General Pattern**:

```sql
ALTER TABLE <table_name> ENABLE ROW LEVEL SECURITY;
```

### 3.2 Base Policy Pattern

For each CRM table, create base policies that enforce `org_id` scoping:

**SELECT Policy**:
```sql
CREATE POLICY "<table_name>_select_own_org"
  ON <table_name>
  FOR SELECT
  USING (org_id = get_user_org_id());
```

**INSERT Policy**:
```sql
CREATE POLICY "<table_name>_insert_own_org"
  ON <table_name>
  FOR INSERT
  WITH CHECK (org_id = get_user_org_id());
```

**UPDATE Policy**:
```sql
CREATE POLICY "<table_name>_update_own_org"
  ON <table_name>
  FOR UPDATE
  USING (org_id = get_user_org_id())
  WITH CHECK (org_id = get_user_org_id());
```

**DELETE Policy**:
```sql
CREATE POLICY "<table_name>_delete_own_org"
  ON <table_name>
  FOR DELETE
  USING (org_id = get_user_org_id());
```

### 3.3 Complete RLS Policies for Each CRM Table

#### 3.3.1 `customers` Table RLS Policies

```sql
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;

-- SELECT: Users can read customers in their org
CREATE POLICY "customers_select_own_org"
  ON customers
  FOR SELECT
  USING (org_id = get_user_org_id());

-- INSERT: Users can create customers in their org
CREATE POLICY "customers_insert_own_org"
  ON customers
  FOR INSERT
  WITH CHECK (org_id = get_user_org_id());

-- UPDATE: Users can update customers in their org
CREATE POLICY "customers_update_own_org"
  ON customers
  FOR UPDATE
  USING (org_id = get_user_org_id())
  WITH CHECK (org_id = get_user_org_id());

-- DELETE: Users can delete customers in their org
CREATE POLICY "customers_delete_own_org"
  ON customers
  FOR DELETE
  USING (org_id = get_user_org_id());
```

#### 3.3.2 `customer_locations` Table RLS Policies

```sql
ALTER TABLE customer_locations ENABLE ROW LEVEL SECURITY;

CREATE POLICY "customer_locations_select_own_org"
  ON customer_locations
  FOR SELECT
  USING (org_id = get_user_org_id());

CREATE POLICY "customer_locations_insert_own_org"
  ON customer_locations
  FOR INSERT
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "customer_locations_update_own_org"
  ON customer_locations
  FOR UPDATE
  USING (org_id = get_user_org_id())
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "customer_locations_delete_own_org"
  ON customer_locations
  FOR DELETE
  USING (org_id = get_user_org_id());
```

#### 3.3.3 `customer_contacts` Table RLS Policies

```sql
ALTER TABLE customer_contacts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "customer_contacts_select_own_org"
  ON customer_contacts
  FOR SELECT
  USING (org_id = get_user_org_id());

CREATE POLICY "customer_contacts_insert_own_org"
  ON customer_contacts
  FOR INSERT
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "customer_contacts_update_own_org"
  ON customer_contacts
  FOR UPDATE
  USING (org_id = get_user_org_id())
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "customer_contacts_delete_own_org"
  ON customer_contacts
  FOR DELETE
  USING (org_id = get_user_org_id());
```

#### 3.3.4 `crm_preferences` Table RLS Policies

```sql
ALTER TABLE crm_preferences ENABLE ROW LEVEL SECURITY;

CREATE POLICY "crm_preferences_select_own_org"
  ON crm_preferences
  FOR SELECT
  USING (org_id = get_user_org_id());

CREATE POLICY "crm_preferences_insert_own_org"
  ON crm_preferences
  FOR INSERT
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_preferences_update_own_org"
  ON crm_preferences
  FOR UPDATE
  USING (org_id = get_user_org_id())
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_preferences_delete_own_org"
  ON crm_preferences
  FOR DELETE
  USING (org_id = get_user_org_id());
```

#### 3.3.5 `crm_interactions` Table RLS Policies

```sql
ALTER TABLE crm_interactions ENABLE ROW LEVEL SECURITY;

CREATE POLICY "crm_interactions_select_own_org"
  ON crm_interactions
  FOR SELECT
  USING (org_id = get_user_org_id());

CREATE POLICY "crm_interactions_insert_own_org"
  ON crm_interactions
  FOR INSERT
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_interactions_update_own_org"
  ON crm_interactions
  FOR UPDATE
  USING (org_id = get_user_org_id())
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_interactions_delete_own_org"
  ON crm_interactions
  FOR DELETE
  USING (org_id = get_user_org_id());
```

#### 3.3.6 `crm_followups` Table RLS Policies

```sql
ALTER TABLE crm_followups ENABLE ROW LEVEL SECURITY;

CREATE POLICY "crm_followups_select_own_org"
  ON crm_followups
  FOR SELECT
  USING (org_id = get_user_org_id());

CREATE POLICY "crm_followups_insert_own_org"
  ON crm_followups
  FOR INSERT
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_followups_update_own_org"
  ON crm_followups
  FOR UPDATE
  USING (org_id = get_user_org_id())
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_followups_delete_own_org"
  ON crm_followups
  FOR DELETE
  USING (org_id = get_user_org_id());
```

#### 3.3.7 `crm_tags` Table RLS Policies

```sql
ALTER TABLE crm_tags ENABLE ROW LEVEL SECURITY;

CREATE POLICY "crm_tags_select_own_org"
  ON crm_tags
  FOR SELECT
  USING (org_id = get_user_org_id());

CREATE POLICY "crm_tags_insert_own_org"
  ON crm_tags
  FOR INSERT
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_tags_update_own_org"
  ON crm_tags
  FOR UPDATE
  USING (org_id = get_user_org_id())
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_tags_delete_own_org"
  ON crm_tags
  FOR DELETE
  USING (org_id = get_user_org_id());
```

#### 3.3.8 `crm_customer_tags` Table RLS Policies

```sql
ALTER TABLE crm_customer_tags ENABLE ROW LEVEL SECURITY;

CREATE POLICY "crm_customer_tags_select_own_org"
  ON crm_customer_tags
  FOR SELECT
  USING (org_id = get_user_org_id());

CREATE POLICY "crm_customer_tags_insert_own_org"
  ON crm_customer_tags
  FOR INSERT
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_customer_tags_update_own_org"
  ON crm_customer_tags
  FOR UPDATE
  USING (org_id = get_user_org_id())
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_customer_tags_delete_own_org"
  ON crm_customer_tags
  FOR DELETE
  USING (org_id = get_user_org_id());
```

#### 3.3.9 `crm_segments` Table RLS Policies

```sql
ALTER TABLE crm_segments ENABLE ROW LEVEL SECURITY;

CREATE POLICY "crm_segments_select_own_org"
  ON crm_segments
  FOR SELECT
  USING (org_id = get_user_org_id());

CREATE POLICY "crm_segments_insert_own_org"
  ON crm_segments
  FOR INSERT
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_segments_update_own_org"
  ON crm_segments
  FOR UPDATE
  USING (org_id = get_user_org_id())
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_segments_delete_own_org"
  ON crm_segments
  FOR DELETE
  USING (org_id = get_user_org_id());
```

#### 3.3.10 `crm_segment_members` Table RLS Policies

```sql
ALTER TABLE crm_segment_members ENABLE ROW LEVEL SECURITY;

CREATE POLICY "crm_segment_members_select_own_org"
  ON crm_segment_members
  FOR SELECT
  USING (org_id = get_user_org_id());

CREATE POLICY "crm_segment_members_insert_own_org"
  ON crm_segment_members
  FOR INSERT
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_segment_members_update_own_org"
  ON crm_segment_members
  FOR UPDATE
  USING (org_id = get_user_org_id())
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_segment_members_delete_own_org"
  ON crm_segment_members
  FOR DELETE
  USING (org_id = get_user_org_id());
```

#### 3.3.11 `crm_message_templates` Table RLS Policies

```sql
ALTER TABLE crm_message_templates ENABLE ROW LEVEL SECURITY;

CREATE POLICY "crm_message_templates_select_own_org"
  ON crm_message_templates
  FOR SELECT
  USING (org_id = get_user_org_id());

CREATE POLICY "crm_message_templates_insert_own_org"
  ON crm_message_templates
  FOR INSERT
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_message_templates_update_own_org"
  ON crm_message_templates
  FOR UPDATE
  USING (org_id = get_user_org_id())
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_message_templates_delete_own_org"
  ON crm_message_templates
  FOR DELETE
  USING (org_id = get_user_org_id());
```

#### 3.3.12 `crm_automation_rules` Table RLS Policies

```sql
ALTER TABLE crm_automation_rules ENABLE ROW LEVEL SECURITY;

CREATE POLICY "crm_automation_rules_select_own_org"
  ON crm_automation_rules
  FOR SELECT
  USING (org_id = get_user_org_id());

CREATE POLICY "crm_automation_rules_insert_own_org"
  ON crm_automation_rules
  FOR INSERT
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_automation_rules_update_own_org"
  ON crm_automation_rules
  FOR UPDATE
  USING (org_id = get_user_org_id())
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_automation_rules_delete_own_org"
  ON crm_automation_rules
  FOR DELETE
  USING (org_id = get_user_org_id());
```

#### 3.3.13 `crm_automation_runs` Table RLS Policies

```sql
ALTER TABLE crm_automation_runs ENABLE ROW LEVEL SECURITY;

CREATE POLICY "crm_automation_runs_select_own_org"
  ON crm_automation_runs
  FOR SELECT
  USING (org_id = get_user_org_id());

CREATE POLICY "crm_automation_runs_insert_own_org"
  ON crm_automation_runs
  FOR INSERT
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_automation_runs_update_own_org"
  ON crm_automation_runs
  FOR UPDATE
  USING (org_id = get_user_org_id())
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "crm_automation_runs_delete_own_org"
  ON crm_automation_runs
  FOR DELETE
  USING (org_id = get_user_org_id());
```

### 3.4 Service Role Bypass Pattern

**Important**: Supabase service role (used by Edge Functions) automatically bypasses RLS when using the `service_role` key. No explicit policy is needed.

**Edge Function Pattern**:

```typescript
// Edge Function using service role (bypasses RLS)
import { createClient } from '@supabase/supabase-js';

const supabaseAdmin = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!, // Service role key bypasses RLS
  {
    auth: {
      autoRefreshToken: false,
      persistSession: false
    }
  }
);

// This query bypasses RLS
const { data } = await supabaseAdmin
  .from('customers')
  .select('*')
  .eq('org_id', targetOrgId);
```

**Security Notes**:
- Service role key must NEVER be exposed to client-side code
- Service role key should only be used in Edge Functions or server-side code
- Always validate `org_id` in Edge Function logic before performing operations

---

## 4. Role-Based RLS Policies

### 4.1 Permission Matrix

The following matrix defines what operations each role can perform on each CRM table:

| Table | admin | manager | dispatcher | technician | csr/sales |
|-------|-------|---------|------------|------------|-----------|
| `customers` | CRUD | CRUD | CRUD | R (limited) | CRUD |
| `customer_locations` | CRUD | CRUD | CRUD | R (limited) | CRUD |
| `customer_contacts` | CRUD | CRUD | CRUD | R (limited) | CRUD |
| `crm_preferences` | CRUD | CRUD | - | - | R |
| `crm_interactions` | CRUD | CRUD | CRUD | R | CRUD |
| `crm_followups` | CRUD | CRUD | CRUD | R (own only) | CRUD |
| `crm_tags` | CRUD | CRUD | CRUD | R | CRUD |
| `crm_customer_tags` | CRUD | CRUD | CRUD | R | CRUD |
| `crm_segments` | CRUD | CRUD | R | - | R |
| `crm_segment_members` | CRUD | CRUD | R | - | R |
| `crm_message_templates` | CRUD | CRUD | R | - | R |
| `crm_automation_rules` | CRUD | CRUD | R | - | R |
| `crm_automation_runs` | CRUD | CRUD | R | - | R |

**Legend**:
- **CRUD**: Create, Read, Update, Delete
- **R**: Read-only
- **R (limited)**: Read-only with limited fields (see §4.2.4)
- **R (own only)**: Read-only for own assigned records
- **-**: No access

### 4.2 Role-Based Policy Implementation

Role-based policies are layered on top of base org-scoping policies. They use `get_user_role()` or `has_role()` functions to check permissions.

#### 4.2.1 Admin and Manager Policies

**Pattern**: Admin and manager have full CRUD access (already covered by base policies). No additional restrictions needed.

**Note**: If stricter admin-only operations are needed (e.g., only admins can delete segments), add additional policies:

```sql
-- Example: Only admins can delete segments
CREATE POLICY "crm_segments_delete_admin_only"
  ON crm_segments
  FOR DELETE
  USING (
    org_id = get_user_org_id() AND
    get_user_role() = 'admin'
  );
```

#### 4.2.2 Dispatcher Policies

**Pattern**: Dispatchers have CRUD on most tables but read-only on segments, automations, and preferences.

**Implementation**: Base policies already allow CRUD. Add restrictive policies for read-only tables:

```sql
-- crm_segments: Read-only for dispatchers
DROP POLICY IF EXISTS "crm_segments_select_own_org" ON crm_segments;
CREATE POLICY "crm_segments_select_own_org"
  ON crm_segments
  FOR SELECT
  USING (
    org_id = get_user_org_id() AND
    (has_role(ARRAY['admin', 'manager', 'dispatcher', 'csr', 'sales']::user_role_enum[]) OR
     get_user_role() IS NOT NULL) -- Allow all authenticated users to read
  );

-- crm_segments: Only admin/manager can insert/update/delete
CREATE POLICY "crm_segments_modify_admin_manager"
  ON crm_segments
  FOR ALL
  USING (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager']::user_role_enum[])
  )
  WITH CHECK (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager']::user_role_enum[])
  );
```

**Note**: The above pattern replaces base policies. A cleaner approach is to keep base policies and add restrictive policies that take precedence.

**Better Pattern**: Use policy precedence (more restrictive policies evaluated first):

```sql
-- Keep base SELECT policy
-- Add restrictive INSERT/UPDATE/DELETE policies

CREATE POLICY "crm_segments_insert_admin_manager"
  ON crm_segments
  FOR INSERT
  WITH CHECK (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager']::user_role_enum[])
  );

CREATE POLICY "crm_segments_update_admin_manager"
  ON crm_segments
  FOR UPDATE
  USING (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager']::user_role_enum[])
  )
  WITH CHECK (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager']::user_role_enum[])
  );

CREATE POLICY "crm_segments_delete_admin_manager"
  ON crm_segments
  FOR DELETE
  USING (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager']::user_role_enum[])
  );
```

#### 4.2.3 Technician Policies

**Pattern**: Technicians have read-only access to limited customer data and no access to preferences, segments, or automations.

**Limited Fields for Technicians**:

Technicians can only read:
- `customers`: `id`, `name`, `first_name`, `last_name`, `company_name`, `status`
- `customer_locations`: All fields (needed for routing)
- `customer_contacts`: `type`, `value` (no opt-in flags)

**Implementation**: Use column-level security or views. Since PostgreSQL RLS doesn't support column-level restrictions directly, use a view or restrict in application logic.

**Option 1: Views for Technicians** (Recommended)

```sql
-- Create view for technician customer access
CREATE VIEW technician_customers AS
SELECT 
  id,
  org_id,
  name,
  first_name,
  last_name,
  company_name,
  status,
  created_at
FROM customers;

-- Enable RLS on view (requires security definer function)
CREATE POLICY "technician_customers_select_own_org"
  ON technician_customers
  FOR SELECT
  USING (
    org_id = get_user_org_id() AND
    get_user_role() = 'technician'
  );
```

**Option 2: Application-Level Filtering** (Simpler)

Keep base RLS policies and filter columns in application code based on role.

**Technician Policies**:

```sql
-- crm_preferences: No access for technicians
DROP POLICY IF EXISTS "crm_preferences_select_own_org" ON crm_preferences;
CREATE POLICY "crm_preferences_select_own_org"
  ON crm_preferences
  FOR SELECT
  USING (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager', 'csr', 'sales']::user_role_enum[])
  );

-- crm_segments: No access for technicians
DROP POLICY IF EXISTS "crm_segments_select_own_org" ON crm_segments;
CREATE POLICY "crm_segments_select_own_org"
  ON crm_segments
  FOR SELECT
  USING (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager', 'dispatcher', 'csr', 'sales']::user_role_enum[])
  );

-- crm_automation_rules: No access for technicians
DROP POLICY IF EXISTS "crm_automation_rules_select_own_org" ON crm_automation_rules;
CREATE POLICY "crm_automation_rules_select_own_org"
  ON crm_automation_rules
  FOR SELECT
  USING (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager', 'dispatcher', 'csr', 'sales']::user_role_enum[])
  );

-- crm_followups: Technicians can only read their own assigned follow-ups
CREATE POLICY "crm_followups_select_technician_own"
  ON crm_followups
  FOR SELECT
  USING (
    org_id = get_user_org_id() AND
    get_user_role() = 'technician' AND
    assigned_to_user_id = auth.uid()
  );

-- Technicians cannot insert/update/delete follow-ups
CREATE POLICY "crm_followups_modify_no_technician"
  ON crm_followups
  FOR ALL
  USING (
    org_id = get_user_org_id() AND
    get_user_role() != 'technician'
  )
  WITH CHECK (
    org_id = get_user_org_id() AND
    get_user_role() != 'technician'
  );
```

#### 4.2.4 CSR/Sales Policies

**Pattern**: CSR/Sales have full CRUD on customers, contacts, locations, interactions, follow-ups, and tags. Read-only on segments, automations, and preferences.

**Implementation**: Base policies already allow CRUD. Add restrictive policies for read-only tables (similar to dispatcher pattern):

```sql
-- crm_preferences: CSR/Sales can read but not modify
CREATE POLICY "crm_preferences_read_csr_sales"
  ON crm_preferences
  FOR SELECT
  USING (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager', 'csr', 'sales']::user_role_enum[])
  );

-- crm_preferences: Only admin/manager can modify
CREATE POLICY "crm_preferences_modify_admin_manager"
  ON crm_preferences
  FOR INSERT
  WITH CHECK (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager']::user_role_enum[])
  );

CREATE POLICY "crm_preferences_update_admin_manager"
  ON crm_preferences
  FOR UPDATE
  USING (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager']::user_role_enum[])
  )
  WITH CHECK (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager']::user_role_enum[])
  );

CREATE POLICY "crm_preferences_delete_admin_manager"
  ON crm_preferences
  FOR DELETE
  USING (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager']::user_role_enum[])
  );
```

### 4.3 Complete Role-Based Policies by Table

#### 4.3.1 `crm_preferences` Role-Based Policies

```sql
-- Drop base policies (will be replaced)
DROP POLICY IF EXISTS "crm_preferences_select_own_org" ON crm_preferences;
DROP POLICY IF EXISTS "crm_preferences_insert_own_org" ON crm_preferences;
DROP POLICY IF EXISTS "crm_preferences_update_own_org" ON crm_preferences;
DROP POLICY IF EXISTS "crm_preferences_delete_own_org" ON crm_preferences;

-- SELECT: Admin, manager, CSR, sales can read
CREATE POLICY "crm_preferences_select_role_based"
  ON crm_preferences
  FOR SELECT
  USING (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager', 'csr', 'sales']::user_role_enum[])
  );

-- INSERT/UPDATE/DELETE: Only admin and manager
CREATE POLICY "crm_preferences_modify_admin_manager"
  ON crm_preferences
  FOR ALL
  USING (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager']::user_role_enum[])
  )
  WITH CHECK (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager']::user_role_enum[])
  );
```

#### 4.3.2 `crm_segments` Role-Based Policies

```sql
-- Drop base policies
DROP POLICY IF EXISTS "crm_segments_select_own_org" ON crm_segments;
DROP POLICY IF EXISTS "crm_segments_insert_own_org" ON crm_segments;
DROP POLICY IF EXISTS "crm_segments_update_own_org" ON crm_segments;
DROP POLICY IF EXISTS "crm_segments_delete_own_org" ON crm_segments;

-- SELECT: All roles except technician
CREATE POLICY "crm_segments_select_role_based"
  ON crm_segments
  FOR SELECT
  USING (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager', 'dispatcher', 'csr', 'sales']::user_role_enum[])
  );

-- INSERT/UPDATE/DELETE: Only admin and manager
CREATE POLICY "crm_segments_modify_admin_manager"
  ON crm_segments
  FOR ALL
  USING (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager']::user_role_enum[])
  )
  WITH CHECK (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager']::user_role_enum[])
  );
```

#### 4.3.3 `crm_automation_rules` Role-Based Policies

```sql
-- Drop base policies
DROP POLICY IF EXISTS "crm_automation_rules_select_own_org" ON crm_automation_rules;
DROP POLICY IF EXISTS "crm_automation_rules_insert_own_org" ON crm_automation_rules;
DROP POLICY IF EXISTS "crm_automation_rules_update_own_org" ON crm_automation_rules;
DROP POLICY IF EXISTS "crm_automation_rules_delete_own_org" ON crm_automation_rules;

-- SELECT: All roles except technician
CREATE POLICY "crm_automation_rules_select_role_based"
  ON crm_automation_rules
  FOR SELECT
  USING (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager', 'dispatcher', 'csr', 'sales']::user_role_enum[])
  );

-- INSERT/UPDATE/DELETE: Only admin and manager
CREATE POLICY "crm_automation_rules_modify_admin_manager"
  ON crm_automation_rules
  FOR ALL
  USING (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager']::user_role_enum[])
  )
  WITH CHECK (
    org_id = get_user_org_id() AND
    has_role(ARRAY['admin', 'manager']::user_role_enum[])
  );
```

#### 4.3.4 `crm_followups` Role-Based Policies

```sql
-- Add technician-specific policy (technicians can only read their own)
CREATE POLICY "crm_followups_select_technician_own"
  ON crm_followups
  FOR SELECT
  USING (
    org_id = get_user_org_id() AND
    get_user_role() = 'technician' AND
    assigned_to_user_id = auth.uid()
  );

-- Restrict technicians from modifying follow-ups
-- Base policies already allow CRUD for other roles, so no additional restrictions needed
-- Technicians will be blocked by the absence of a matching INSERT/UPDATE/DELETE policy
-- when their role is 'technician'
```

**Note**: Since base policies use `org_id = get_user_org_id()`, technicians will be blocked from INSERT/UPDATE/DELETE because they don't match the base policy conditions when combined with role checks. However, to be explicit:

```sql
-- Ensure technicians cannot insert/update/delete
CREATE POLICY "crm_followups_modify_no_technician"
  ON crm_followups
  FOR INSERT
  WITH CHECK (
    org_id = get_user_org_id() AND
    get_user_role() != 'technician'
  );

CREATE POLICY "crm_followups_update_no_technician"
  ON crm_followups
  FOR UPDATE
  USING (
    org_id = get_user_org_id() AND
    get_user_role() != 'technician'
  )
  WITH CHECK (
    org_id = get_user_org_id() AND
    get_user_role() != 'technician'
  );

CREATE POLICY "crm_followups_delete_no_technician"
  ON crm_followups
  FOR DELETE
  USING (
    org_id = get_user_org_id() AND
    get_user_role() != 'technician'
  );
```

---

## 5. Migration Strategy

### 5.1 Migration Order

Migrations must be executed in the following order:

1. **Create `user_role_enum`** (if not exists)
2. **Create `profiles` table** (if not exists)
3. **Create helper functions** (`get_user_org_id()`, `get_user_role()`, `is_user_in_org()`, `has_role()`)
4. **Enable RLS on all CRM tables** (from Epic 1)
5. **Create base RLS policies** (org-scoping)
6. **Create role-based RLS policies** (replace or supplement base policies)
7. **Create test users and profiles** (seed data)

### 5.2 Migration File Structure

Recommended migration file naming:

```
supabase/migrations/
  20240102000000_create_user_roles_and_profiles.sql
  20240102000001_create_rls_helper_functions.sql
  20240102000002_enable_rls_on_crm_tables.sql
  20240102000003_create_base_rls_policies.sql
  20240102000004_create_role_based_rls_policies.sql
  20240102000005_seed_test_users.sql
```

### 5.3 Rollback Strategy

Each migration should be idempotent (use `CREATE OR REPLACE`, `DROP POLICY IF EXISTS`, etc.). For rollback:

- Create corresponding `down` migrations that drop policies and functions in reverse order
- Test rollback on non-production environments

---

## 6. Seed Data Requirements

### 6.1 Test Organizations

Create at least two test organizations for isolation testing:

```sql
INSERT INTO orgs (id, name) VALUES
  ('00000000-0000-0000-0000-000000000001', 'Test Org 1'),
  ('00000000-0000-0000-0000-000000000002', 'Test Org 2')
ON CONFLICT (id) DO NOTHING;
```

### 6.2 Test Users

Create test users via Supabase Auth API (not SQL), then create profiles:

**Note**: User creation must be done via Supabase Auth API or dashboard. Profiles can be created via SQL after users exist.

```sql
-- Example: Create profiles for existing auth users
-- Replace user IDs with actual auth.users.id values from Supabase Auth

-- Org 1 users
INSERT INTO profiles (id, org_id, role, email, first_name, last_name) VALUES
  -- Admin user
  ('11111111-1111-1111-1111-111111111111', '00000000-0000-0000-0000-000000000001', 'admin', 'admin@org1.com', 'Admin', 'User'),
  -- Manager user
  ('22222222-2222-2222-2222-222222222222', '00000000-0000-0000-0000-000000000001', 'manager', 'manager@org1.com', 'Manager', 'User'),
  -- CSR user
  ('33333333-3333-3333-3333-333333333333', '00000000-0000-0000-0000-000000000001', 'csr', 'csr@org1.com', 'CSR', 'User'),
  -- Technician user
  ('44444444-4444-4444-4444-444444444444', '00000000-0000-0000-0000-000000000001', 'technician', 'tech@org1.com', 'Tech', 'User')
ON CONFLICT (id) DO NOTHING;

-- Org 2 users
INSERT INTO profiles (id, org_id, role, email, first_name, last_name) VALUES
  -- Admin user
  ('55555555-5555-5555-5555-555555555555', '00000000-0000-0000-0000-000000000002', 'admin', 'admin@org2.com', 'Admin', 'User')
ON CONFLICT (id) DO NOTHING;
```

**Important**: User IDs must match actual `auth.users.id` values. Use Supabase Auth API to create users first, then create profiles.

---

## 7. Testing Strategy

### 7.1 Multi-Tenancy Isolation Tests

**Test Case 1**: User from Org 1 cannot see Org 2 data

```sql
-- As Org 1 user
SET LOCAL request.jwt.claim.sub = '11111111-1111-1111-1111-111111111111';
SELECT COUNT(*) FROM customers; -- Should only return Org 1 customers

-- As Org 2 user
SET LOCAL request.jwt.claim.sub = '55555555-5555-5555-5555-555555555555';
SELECT COUNT(*) FROM customers; -- Should only return Org 2 customers
```

**Test Case 2**: User cannot insert data for different org

```sql
-- As Org 1 user, try to insert Org 2 customer
SET LOCAL request.jwt.claim.sub = '11111111-1111-1111-1111-111111111111';
INSERT INTO customers (org_id, name, type, status, lifecycle_stage)
VALUES ('00000000-0000-0000-0000-000000000002', 'Hacker', 'individual', 'active', 'customer');
-- Should fail with RLS policy violation
```

### 7.2 Role-Based Access Tests

**Test Case 3**: Technician cannot read preferences

```sql
-- As technician
SET LOCAL request.jwt.claim.sub = '44444444-4444-4444-4444-444444444444';
SELECT * FROM crm_preferences; -- Should return empty or error
```

**Test Case 4**: CSR cannot modify segments

```sql
-- As CSR
SET LOCAL request.jwt.claim.sub = '33333333-3333-3333-3333-333333333333';
UPDATE crm_segments SET name = 'Hacked' WHERE id = '...';
-- Should fail with RLS policy violation
```

**Test Case 5**: Technician can only read own follow-ups

```sql
-- As technician
SET LOCAL request.jwt.claim.sub = '44444444-4444-4444-4444-444444444444';
SELECT * FROM crm_followups; -- Should only return follow-ups assigned to this technician
```

### 7.3 Automated Testing

Create test functions or use Supabase testing framework:

```sql
-- Example test function
CREATE OR REPLACE FUNCTION test_rls_isolation()
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
  org1_count INTEGER;
  org2_count INTEGER;
BEGIN
  -- Test as Org 1 user
  PERFORM set_config('request.jwt.claim.sub', '11111111-1111-1111-1111-111111111111', true);
  SELECT COUNT(*) INTO org1_count FROM customers;
  
  -- Test as Org 2 user
  PERFORM set_config('request.jwt.claim.sub', '55555555-5555-5555-5555-555555555555', true);
  SELECT COUNT(*) INTO org2_count FROM customers;
  
  IF org1_count = org2_count THEN
    RETURN 'FAIL: Isolation broken';
  ELSE
    RETURN 'PASS: Isolation working';
  END IF;
END;
$$;
```

---

## 8. Security Considerations

### 8.1 Helper Function Security

- All helper functions use `SECURITY DEFINER` to bypass RLS when reading `profiles`
- Functions are marked `STABLE` for query optimization
- Functions return `NULL` for unauthenticated users (fail-safe)

### 8.2 Policy Evaluation Order

PostgreSQL evaluates RLS policies in creation order. More restrictive policies should be created first or use explicit policy names that ensure correct evaluation.

**Best Practice**: Create base policies first, then add restrictive role-based policies. PostgreSQL will evaluate all matching policies and allow access if ANY policy allows it (for SELECT) or require ALL policies to allow (for INSERT/UPDATE/DELETE with `WITH CHECK`).

### 8.3 Service Role Security

- Service role key must NEVER be exposed to client-side code
- Service role should only be used in Edge Functions or server-side code
- Always validate `org_id` in Edge Function logic before operations
- Log all service role operations for audit

### 8.4 Policy Performance

- Helper functions (`get_user_org_id()`, `get_user_role()`) are called for every row evaluation
- Consider caching results in session variables if performance becomes an issue
- Monitor query performance with `EXPLAIN ANALYZE`

---

## 9. Documentation Requirements

### 9.1 Role Permissions Documentation

Document in CRM README:
- Role definitions and hierarchy
- Permission matrix (table from §4.1)
- How to assign roles to users
- How roles are enforced (RLS policies)

### 9.2 JWT and Claims Documentation

Document:
- How `org_id` and `role` are derived (from `profiles` table)
- Frontend pattern for reading user org/role
- Backend pattern for Edge Functions
- Service role usage patterns

### 9.3 RLS Policy Documentation

Document:
- Which tables have RLS enabled
- Base policy pattern (org-scoping)
- Role-based policy exceptions
- How to test RLS policies

---

## 10. Implementation Checklist

### Story CRM-013: Define CRM User Roles & Profile Schema
- [ ] `user_role_enum` created with all roles
- [ ] `profiles` table created with `org_id` and `role` columns
- [ ] Helper functions created (`get_user_org_id()`, `get_user_role()`, etc.)
- [ ] Sample users created for each role in test org
- [ ] Documentation describes roles and permissions
- [ ] JWT claims pattern documented

### Story CRM-014: Implement RLS Base Policies for Multi-Tenancy
- [ ] RLS enabled on all CRM tables
- [ ] Base SELECT policies created (org-scoping)
- [ ] Base INSERT policies created (org-scoping)
- [ ] Base UPDATE policies created (org-scoping)
- [ ] Base DELETE policies created (org-scoping)
- [ ] Policies tested with two orgs (isolation verified)
- [ ] Service role bypass pattern documented
- [ ] Any exceptions (global admin) documented

### Story CRM-015: Implement Role-Based RLS Policies
- [ ] Role-based policies created for `crm_preferences` (admin/manager only modify)
- [ ] Role-based policies created for `crm_segments` (admin/manager only modify, technician no access)
- [ ] Role-based policies created for `crm_automation_rules` (admin/manager only modify, technician no access)
- [ ] Role-based policies created for `crm_followups` (technician read own only)
- [ ] Test cases created for each role
- [ ] Policy comments/documentation explain rationale
- [ ] Unauthorized operations are blocked and logged

---

## 11. Appendix: Complete SQL Migration Scripts

### 11.1 Create Roles and Profiles

```sql
-- File: 20240102000000_create_user_roles_and_profiles.sql

-- Create role enum
CREATE TYPE user_role_enum AS ENUM (
  'admin',
  'manager',
  'dispatcher',
  'technician',
  'csr',
  'sales'
);

-- Create profiles table
CREATE TABLE IF NOT EXISTS profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE RESTRICT,
  role user_role_enum NOT NULL,
  first_name TEXT,
  last_name TEXT,
  email TEXT,
  phone TEXT,
  avatar_url TEXT,
  is_active BOOLEAN NOT NULL DEFAULT true,
  last_login_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_profiles_email_format CHECK (
    email IS NULL OR email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
  )
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_profiles_org_id ON profiles(org_id);
CREATE INDEX IF NOT EXISTS idx_profiles_role ON profiles(role);
CREATE INDEX IF NOT EXISTS idx_profiles_org_id_role ON profiles(org_id, role);
CREATE INDEX IF NOT EXISTS idx_profiles_email ON profiles(email) WHERE email IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_profiles_is_active ON profiles(is_active) WHERE is_active = true;

-- Trigger for updated_at
CREATE OR REPLACE FUNCTION update_profiles_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_profiles_updated_at
  BEFORE UPDATE ON profiles
  FOR EACH ROW
  EXECUTE FUNCTION update_profiles_updated_at();

-- RLS on profiles
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can read own profile"
  ON profiles
  FOR SELECT
  USING (auth.uid() = id);

CREATE POLICY "Users can update own profile"
  ON profiles
  FOR UPDATE
  USING (auth.uid() = id)
  WITH CHECK (
    auth.uid() = id AND
    (OLD.org_id = NEW.org_id) AND
    (OLD.role = NEW.role)
  );

CREATE POLICY "Admins can read org profiles"
  ON profiles
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM profiles p
      WHERE p.id = auth.uid()
      AND p.org_id = profiles.org_id
      AND p.role IN ('admin', 'manager')
      AND p.is_active = true
    )
  );
```

### 11.2 Create Helper Functions

```sql
-- File: 20240102000001_create_rls_helper_functions.sql

CREATE OR REPLACE FUNCTION get_user_org_id()
RETURNS UUID
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  user_org_id UUID;
BEGIN
  SELECT org_id INTO user_org_id
  FROM profiles
  WHERE id = auth.uid()
    AND is_active = true;
  
  RETURN user_org_id;
END;
$$;

CREATE OR REPLACE FUNCTION get_user_role()
RETURNS user_role_enum
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  user_role user_role_enum;
BEGIN
  SELECT role INTO user_role
  FROM profiles
  WHERE id = auth.uid()
    AND is_active = true;
  
  RETURN user_role;
END;
$$;

CREATE OR REPLACE FUNCTION is_user_in_org(org_uuid UUID)
RETURNS BOOLEAN
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  RETURN EXISTS (
    SELECT 1
    FROM profiles
    WHERE id = auth.uid()
      AND org_id = org_uuid
      AND is_active = true
  );
END;
$$;

CREATE OR REPLACE FUNCTION has_role(required_roles user_role_enum[])
RETURNS BOOLEAN
LANGUAGE plpgsql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  RETURN EXISTS (
    SELECT 1
    FROM profiles
    WHERE id = auth.uid()
      AND role = ANY(required_roles)
      AND is_active = true
  );
END;
$$;
```

### 11.3 Enable RLS and Create Base Policies

```sql
-- File: 20240102000002_enable_rls_on_crm_tables.sql
-- File: 20240102000003_create_base_rls_policies.sql

-- Enable RLS on all CRM tables
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
ALTER TABLE customer_locations ENABLE ROW LEVEL SECURITY;
ALTER TABLE customer_contacts ENABLE ROW LEVEL SECURITY;
ALTER TABLE crm_preferences ENABLE ROW LEVEL SECURITY;
ALTER TABLE crm_interactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE crm_followups ENABLE ROW LEVEL SECURITY;
ALTER TABLE crm_tags ENABLE ROW LEVEL SECURITY;
ALTER TABLE crm_customer_tags ENABLE ROW LEVEL SECURITY;
ALTER TABLE crm_segments ENABLE ROW LEVEL SECURITY;
ALTER TABLE crm_segment_members ENABLE ROW LEVEL SECURITY;
ALTER TABLE crm_message_templates ENABLE ROW LEVEL SECURITY;
ALTER TABLE crm_automation_rules ENABLE ROW LEVEL SECURITY;
ALTER TABLE crm_automation_runs ENABLE ROW LEVEL SECURITY;

-- Base policies for each table (example for customers, repeat for all tables)
CREATE POLICY "customers_select_own_org"
  ON customers
  FOR SELECT
  USING (org_id = get_user_org_id());

CREATE POLICY "customers_insert_own_org"
  ON customers
  FOR INSERT
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "customers_update_own_org"
  ON customers
  FOR UPDATE
  USING (org_id = get_user_org_id())
  WITH CHECK (org_id = get_user_org_id());

CREATE POLICY "customers_delete_own_org"
  ON customers
  FOR DELETE
  USING (org_id = get_user_org_id());

-- Repeat for all other CRM tables...
```

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 2 – Authentication, Authorization & RLS Policies. All specifications are designed to be directly consumable by LLM-based code generators, with exact SQL syntax, policy definitions, helper functions, and security patterns defined.

