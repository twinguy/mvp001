# Technical Design Document – Epic 0: Platform Identity Model & Tenancy Primitives

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 0 – Platform Identity Model & Tenancy Primitives (Required by CRM)
- **Source**: Derived from `fdd_0_agile.md` Epic 0 (Stories AUTH-000 through AUTH-003)
- **Reference Documents**: 
  - `fdd_0.md` (Platform Foundation Requirements)
  - `fdd_0_agile.md` (Agile User Stories)
  - `functional.md` (Platform Capabilities)
  - `tooling.md` (Platform Choices: Supabase + Vercel)
- **Target Platform**: Supabase (PostgreSQL 15+), Next.js 14+ (App Router), TypeScript
- **Purpose**: Comprehensive technical specification for LLM-based code generation
- **Prerequisites**: Supabase project initialized, local development environment configured

---

## 1. Overview

This document provides complete technical specifications for implementing the foundational identity and tenancy primitives required by all modules (especially CRM). It covers:

- Tenancy strategy and account model decisions
- `orgs` table schema and lifecycle management
- `profiles` table schema linking Supabase Auth users to organizations and roles
- Role model with complete enum definition
- Helper functions for deriving `org_id` and role from authenticated users
- Row-Level Security (RLS) policies for tenant isolation
- Auth context conventions for consistent access control across modules
- Migration strategy and testing requirements

All specifications are designed to be directly implementable via SQL migrations in Supabase, with exact DDL, functions, policies, and patterns defined.

**Critical Dependency**: This epic must be completed before Epic 1 (Authentication flows) and all CRM modules, as they depend on the `orgs` and `profiles` foundation.

---

## 2. Prerequisites

### 2.1 Required Components

Before implementing Epic 0, ensure:

1. **Supabase Project Setup**:
   - Supabase project created (cloud or local via CLI)
   - Database access configured
   - Migration system initialized (`supabase init` if using CLI)

2. **Development Environment**:
   - PostgreSQL 15+ (via Supabase)
   - Supabase CLI installed (for local development)
   - Database client access (Supabase Studio or psql)

3. **No Dependencies**: This epic has no prerequisites and establishes the foundation for all other epics.

### 2.2 Project Structure

This epic creates the following database objects:

```
supabase/migrations/
  20240101000000_create_orgs_table.sql
  20240101000001_create_user_role_enum.sql
  20240101000002_create_profiles_table.sql
  20240101000003_create_helper_functions.sql
  20240101000004_enable_rls_policies.sql
```

---

## 3. Story AUTH-000: Define Tenancy Strategy and Account Model

### 3.1 Tenancy Decision

**Decision**: Single Supabase project, multi-tenant by `org_id`

**Rationale**:
- Cost-effective: Single project reduces infrastructure overhead
- Simpler operations: One database to manage, backup, and monitor
- Easier development: Local development with one Supabase instance
- Scalability: PostgreSQL handles multi-tenancy efficiently with RLS
- Alignment with Supabase best practices: RLS is designed for this pattern

**Alternative Considered**: One Supabase project per tenant
- **Rejected because**: Higher operational complexity, cost multiplication, harder local development

### 3.2 Tenancy Model Specifications

#### 3.2.1 Core Principles

1. **Tenant Isolation**: Every data row belongs to exactly one organization via `org_id` column
2. **RLS Enforcement**: Row-Level Security policies enforce tenant boundaries at the database level
3. **No Cross-Tenant Access**: Users can only access data within their organization (except break-glass support access)
4. **Consistent Naming**: All tenant-scoped tables use `org_id` column name (UUID, FK to `orgs.id`)

#### 3.2.2 Naming Conventions

**Table-Level Conventions**:
- All tenant-scoped tables MUST include `org_id UUID NOT NULL REFERENCES orgs(id)`
- `org_id` column name is consistent across all tables
- Foreign key constraint: `REFERENCES orgs(id) ON DELETE RESTRICT` (prevents accidental org deletion)
- Exception: `orgs` table itself does not have `org_id` (it is the root tenant entity)

**Column Naming Standards**:
- `org_id`: Organization identifier (UUID, FK to `orgs.id`)
- `user_id`: User identifier (UUID, FK to `auth.users.id`) - used in `profiles` table
- `id`: Primary key (UUID, typically `gen_random_uuid()`)
- `created_at`: Timestamp of creation (TIMESTAMPTZ, DEFAULT now())
- `updated_at`: Timestamp of last update (TIMESTAMPTZ, DEFAULT now())

**Table Naming Standards**:
- Plural nouns: `orgs`, `profiles`, `customers`, `locations`
- Snake_case: All table and column names use snake_case
- Descriptive: Table names clearly indicate their purpose

#### 3.2.3 RLS Strategy

**Base Pattern for All Tenant-Scoped Tables**:

```sql
-- Enable RLS
ALTER TABLE <table_name> ENABLE ROW LEVEL SECURITY;

-- Base SELECT policy: Users can only see rows in their org
CREATE POLICY "<table_name>_select_org"
ON <table_name> FOR SELECT
USING (org_id = get_user_org_id());

-- Base INSERT policy: Users can only insert rows in their org
CREATE POLICY "<table_name>_insert_org"
ON <table_name> FOR INSERT
WITH CHECK (org_id = get_user_org_id());

-- Base UPDATE policy: Users can only update rows in their org
CREATE POLICY "<table_name>_update_org"
ON <table_name> FOR UPDATE
USING (org_id = get_user_org_id())
WITH CHECK (org_id = get_user_org_id());

-- Base DELETE policy: Users can only delete rows in their org
CREATE POLICY "<table_name>_delete_org"
ON <table_name> FOR DELETE
USING (org_id = get_user_org_id());
```

**Key Points**:
- All policies use `get_user_org_id()` helper function (defined in Story AUTH-003)
- Policies are additive: Base org-scoping + role-based restrictions
- Service role bypasses RLS (documented exception for Edge Functions)

#### 3.2.4 Onboarding Implications

**Org Creation Flow**:
1. User signs up via Supabase Auth (creates `auth.users` record)
2. User completes onboarding form (collects org name)
3. System creates `orgs` record
4. System creates `profiles` record linking user to org with role `owner`
5. User can now access the platform within their org context

**Idempotency**: Onboarding must be idempotent (refresh/retry does not create duplicate orgs)

#### 3.2.5 Support/Admin Access (Break-Glass)

**Policy**: Support access is NOT granted by default. Future Epic 5 (AUTH-052) will define break-glass patterns.

**Current State**: No cross-org access is possible. All access is org-scoped.

**Future Considerations**:
- Support users may have a special role or flag
- Support access will require explicit opt-in and audit logging
- Support access will be time-limited and revocable

### 3.3 Documentation Requirements

**Deliverables**:
- [ ] Tenancy decision documented in this TDD
- [ ] Naming conventions documented
- [ ] RLS strategy pattern documented
- [ ] Onboarding flow implications documented
- [ ] Reference from CRM Epic 1 stories

---

## 4. Story AUTH-001: Define `orgs` (Accounts) Concept and Lifecycle

### 4.1 Organization Concept

**Purpose**: An organization (`org`) represents a tenant company account. All data in the platform is scoped to an organization.

**Core Attributes**:
- Unique identifier (`id`: UUID)
- Organization name (`name`: TEXT, required)
- Creation timestamp (`created_at`: TIMESTAMPTZ)
- Soft delete flag (`is_active`: BOOLEAN, for deactivation)

**Future Extensibility**: The schema is designed to accommodate future fields:
- `address` (TEXT)
- `timezone` (TEXT, e.g., 'America/New_York')
- `billing_email` (TEXT)
- `subscription_status` (TEXT, enum)
- `trial_ends_at` (TIMESTAMPTZ)
- `owner_user_id` (UUID, FK to `profiles.id`)

### 4.2 `orgs` Table Schema

**DDL**:

```sql
-- Create orgs table
CREATE TABLE IF NOT EXISTS orgs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Constraints
  CONSTRAINT chk_orgs_name_not_empty CHECK (length(trim(name)) > 0)
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_orgs_name ON orgs(name);
CREATE INDEX IF NOT EXISTS idx_orgs_is_active ON orgs(is_active) WHERE is_active = true;
CREATE INDEX IF NOT EXISTS idx_orgs_created_at ON orgs(created_at);

-- Trigger for updated_at
CREATE OR REPLACE FUNCTION update_orgs_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_orgs_updated_at
  BEFORE UPDATE ON orgs
  FOR EACH ROW
  EXECUTE FUNCTION update_orgs_updated_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key, unique identifier |
| `name` | TEXT | NO | - | Organization name (trimmed, non-empty) |
| `is_active` | BOOLEAN | NO | `true` | Soft delete flag (false = deactivated) |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- Organization name must be non-empty (after trimming)
- `is_active = false` prevents new data creation (enforced in application logic)
- `is_active = false` does NOT delete existing data (soft delete)
- Organizations cannot be hard-deleted if they have associated `profiles` (RESTRICT FK)

### 4.3 Organization Lifecycle

#### 4.3.1 Creation Flow

**Happy Path**:
1. User completes sign-up (Epic 1)
2. User completes onboarding form:
   - Input: Organization name
   - Validation: Name is required, non-empty, max length (e.g., 255 chars)
3. Server Action creates org:
   ```sql
   INSERT INTO orgs (name) VALUES ('Acme HVAC Services') RETURNING id;
   ```
4. Server Action creates profile:
   ```sql
   INSERT INTO profiles (id, org_id, role) 
   VALUES (auth.uid(), <org_id>, 'owner') 
   RETURNING *;
   ```
5. User redirected to app home

**Idempotency Handling**:
- Check if user already has a profile before creating org
- If profile exists, redirect to app home (skip org creation)
- Use database transaction to ensure atomicity

**Error Cases**:
- Duplicate org name: Allowed (names are not unique globally)
- Invalid name: Validation error, show to user
- Database error: Log error, show generic message to user

#### 4.3.2 Update Flow

**Allowed Updates**:
- Organization name (by owner/admin)
- Future: Address, timezone, billing email

**Update Pattern**:
```sql
UPDATE orgs 
SET name = $1, updated_at = now()
WHERE id = $2 AND is_active = true
RETURNING *;
```

**Authorization**: Only users with role `owner` or `admin` can update org details (enforced in application logic or RLS)

#### 4.3.3 Deactivation Flow (Soft Delete)

**Policy**: Organizations are soft-deleted, not hard-deleted.

**Deactivation Process**:
1. Set `is_active = false`:
   ```sql
   UPDATE orgs 
   SET is_active = false, updated_at = now()
   WHERE id = $1;
   ```
2. Application logic prevents:
   - New data creation for deactivated orgs
   - User login for users in deactivated orgs
   - New team member invites
3. Existing data remains accessible (for reactivation or data export)

**Authorization**: Only `owner` role can deactivate org (enforced in application logic)

**Reactivation**:
- Set `is_active = true` (same authorization required)
- All existing data becomes accessible again

**Hard Delete** (Future, Not MVP):
- Hard delete would require:
  - Cascade delete all related data
  - Export data before deletion
  - Confirmation from multiple owners
  - Audit logging
- **Not implemented in MVP**: Use soft delete only

### 4.4 RLS Policies for `orgs` Table

**Policy**: Users can only read their own organization.

```sql
-- Enable RLS
ALTER TABLE orgs ENABLE ROW LEVEL SECURITY;

-- Policy: Users can read their own organization
CREATE POLICY "orgs_select_own"
ON orgs FOR SELECT
USING (
  id = get_user_org_id()
);

-- Policy: Owners/admins can update their organization
CREATE POLICY "orgs_update_own"
ON orgs FOR UPDATE
USING (
  id = get_user_org_id() AND
  EXISTS (
    SELECT 1 FROM profiles
    WHERE id = auth.uid()
    AND org_id = orgs.id
    AND role IN ('owner', 'admin')
    AND is_active = true
  )
)
WITH CHECK (
  id = get_user_org_id() AND
  -- Prevent deactivating if last owner
  (OLD.is_active = true OR NEW.is_active = true OR
   EXISTS (
     SELECT 1 FROM profiles
     WHERE org_id = orgs.id
     AND role = 'owner'
     AND is_active = true
     AND id != auth.uid()
   ))
);
```

**Note**: INSERT policy is not needed for `orgs` table (orgs are created via Server Actions with service role or explicit authorization check).

### 4.5 Documentation Requirements

**Deliverables**:
- [ ] `orgs` table schema documented with DDL
- [ ] Lifecycle flows documented (create, update, deactivate)
- [ ] RLS policies implemented
- [ ] Reference from CRM Epic 1 (CRM-001) story

---

## 5. Story AUTH-002: Define User Profile Mapping (`profiles`) and Role Model

### 5.1 Profile Concept

**Purpose**: The `profiles` table links Supabase Auth users (`auth.users`) to organizations and roles. It extends Supabase Auth with platform-specific metadata.

**Core Attributes**:
- User ID (`id`: UUID, FK to `auth.users.id`)
- Organization ID (`org_id`: UUID, FK to `orgs.id`)
- Role (`role`: user_role_enum)
- Active status (`is_active`: BOOLEAN)
- Contact information (name, email, phone)
- Timestamps (created_at, updated_at, last_login_at)

**Relationship Model**: 
- **MVP Decision**: Single org per user (one profile per user)
- **Rationale**: Simpler implementation, clearer UX, sufficient for MVP
- **Future Consideration**: Multi-org support would require active-org selection UI

### 5.2 Role Model

#### 5.2.1 Role Definitions

| Role | Description | Key Permissions (Platform-Level) |
|------|-------------|----------------------------------|
| `owner` | Organization owner | Full access + billing management + org settings |
| `admin` | System administrator | Full access except billing ownership transfer |
| `manager` | Operations manager | CRM management + reporting + limited platform admin |
| `csr` | Customer service rep | CRM customers/interactions/followups; no billing/team |
| `dispatcher` | Dispatch coordinator | Dispatch console access; limited CRM read |
| `technician` | Field technician | Limited customer read + assigned work visibility |
| `viewer` | Read-only user | Read-only access to CRM and other modules |

**Role Hierarchy** (for permission inheritance):
- `owner` > `admin` > `manager` > `csr`/`dispatcher`/`technician` > `viewer`

**Note**: Detailed permission matrix per module will be defined in Epic 5 (AUTH-050).

#### 5.2.2 Role Enum Definition

**DDL**:

```sql
-- Create user_role_enum
CREATE TYPE user_role_enum AS ENUM (
  'owner',
  'admin',
  'manager',
  'csr',
  'dispatcher',
  'technician',
  'viewer'
);
```

**Business Rules**:
- All users must have exactly one role
- Role cannot be NULL
- Role changes require `owner` or `admin` privileges (enforced in application logic)
- Role changes are audited (future Epic 7)

### 5.3 `profiles` Table Schema

**DDL**:

```sql
-- Create profiles table
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
  ),
  CONSTRAINT chk_profiles_phone_format CHECK (
    phone IS NULL OR phone ~* '^\+?[1-9]\d{1,14}$'
  )
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_profiles_org_id ON profiles(org_id);
CREATE INDEX IF NOT EXISTS idx_profiles_role ON profiles(role);
CREATE INDEX IF NOT EXISTS idx_profiles_org_id_role ON profiles(org_id, role);
CREATE INDEX IF NOT EXISTS idx_profiles_email ON profiles(email) WHERE email IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_profiles_is_active ON profiles(is_active) WHERE is_active = true;
CREATE INDEX IF NOT EXISTS idx_profiles_last_login_at ON profiles(last_login_at) WHERE last_login_at IS NOT NULL;

-- Composite index for common query pattern (org + active users)
CREATE INDEX IF NOT EXISTS idx_profiles_org_active ON profiles(org_id, is_active) WHERE is_active = true;
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
| `phone` | TEXT | YES | NULL | User's phone number (E.164 format recommended) |
| `avatar_url` | TEXT | YES | NULL | URL to user's avatar image (Supabase Storage) |
| `is_active` | BOOLEAN | NO | `true` | Whether user account is active |
| `last_login_at` | TIMESTAMPTZ | YES | NULL | Last login timestamp |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Profile creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- One profile per user (enforced by PK)
- `org_id` cannot be deleted if users exist (RESTRICT on delete)
- Email format validation if provided (regex pattern)
- Phone format validation if provided (E.164 format recommended)
- `is_active = false` should block login (enforced in application logic)
- Email is denormalized for convenience but should sync with `auth.users.email`

#### 5.3.1 Email Sync Strategy

**Option 1: Database Trigger** (Requires superuser privileges):
```sql
-- Note: This requires superuser access to auth.users table
-- Alternative: Handle in application code or Edge Function
CREATE OR REPLACE FUNCTION sync_profile_email()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE profiles
  SET email = NEW.email, updated_at = now()
  WHERE id = NEW.id;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

**Option 2: Application-Level Sync** (Recommended for MVP):
- Sync email in Server Actions when user updates email via Supabase Auth
- Use Supabase Auth webhook or handle in email change flow

**MVP Decision**: Use Option 2 (application-level sync) to avoid superuser requirements.

#### 5.3.2 Updated_at Trigger

```sql
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

### 5.4 Profile Lifecycle

#### 5.4.1 Creation Flow

**Context**: Profile creation happens during onboarding (after org creation).

**Process**:
1. User signs up (creates `auth.users` record)
2. User completes onboarding (creates `orgs` record)
3. System creates `profiles` record:
   ```sql
   INSERT INTO profiles (id, org_id, role, email, first_name, last_name)
   VALUES (
     auth.uid(),
     $org_id,
     'owner',
     (SELECT email FROM auth.users WHERE id = auth.uid()),
     $first_name,
     $last_name
   )
   RETURNING *;
   ```

**Idempotency**: Check if profile exists before creating (onboarding flow handles this).

#### 5.4.2 Update Flow

**Self-Update** (User updates own profile):
- Allowed fields: `first_name`, `last_name`, `phone`, `avatar_url`
- Forbidden fields: `id`, `org_id`, `role` (require admin/owner)

**Admin-Update** (Admin updates team member profile):
- Allowed fields: All fields except `id` and `org_id`
- Role changes require `owner` or `admin` role

#### 5.4.3 Deactivation Flow

**Process**:
1. Set `is_active = false`:
   ```sql
   UPDATE profiles
   SET is_active = false, updated_at = now()
   WHERE id = $user_id AND org_id = get_user_org_id();
   ```
2. Application logic prevents login for deactivated users
3. Existing data remains (soft delete)

**Authorization**: Only `owner` or `admin` can deactivate users (except themselves).

**Protection**: Cannot deactivate last `owner` in org (enforced in application logic).

### 5.5 RLS Policies for `profiles` Table

```sql
-- Enable RLS
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

-- Policy: Users can read their own profile
CREATE POLICY "profiles_select_own"
ON profiles FOR SELECT
USING (id = auth.uid());

-- Policy: Users can read profiles in their organization
CREATE POLICY "profiles_select_org"
ON profiles FOR SELECT
USING (
  org_id = get_user_org_id() AND
  EXISTS (
    SELECT 1 FROM profiles p
    WHERE p.id = auth.uid()
    AND p.org_id = profiles.org_id
    AND p.is_active = true
  )
);

-- Policy: Users can update their own profile (limited fields)
CREATE POLICY "profiles_update_own"
ON profiles FOR UPDATE
USING (id = auth.uid())
WITH CHECK (
  id = auth.uid() AND
  org_id = OLD.org_id AND  -- Prevent org_id changes
  role = OLD.role          -- Prevent role changes
);

-- Policy: Admins/owners can update profiles in their org
CREATE POLICY "profiles_update_org"
ON profiles FOR UPDATE
USING (
  org_id = get_user_org_id() AND
  EXISTS (
    SELECT 1 FROM profiles p
    WHERE p.id = auth.uid()
    AND p.org_id = profiles.org_id
    AND p.role IN ('owner', 'admin')
    AND p.is_active = true
  )
)
WITH CHECK (
  org_id = get_user_org_id() AND
  -- Prevent removing last owner
  (OLD.role != 'owner' OR NEW.role = 'owner' OR
   EXISTS (
     SELECT 1 FROM profiles
     WHERE org_id = profiles.org_id
     AND role = 'owner'
     AND is_active = true
     AND id != profiles.id
   ))
);

-- Policy: Service role can insert profiles (for onboarding)
-- Note: In practice, this is handled via Server Actions with explicit checks
-- This policy allows direct inserts if needed (e.g., Edge Functions)
CREATE POLICY "profiles_insert_service"
ON profiles FOR INSERT
WITH CHECK (true);  -- Service role bypasses RLS, so this is for explicit service role usage

-- Policy: Admins/owners can deactivate profiles (via is_active update)
-- Handled by profiles_update_org policy above
```

**Note**: INSERT policy for normal users is not needed (profiles created via Server Actions with service role or explicit authorization).

### 5.6 Multi-Org User Policy (MVP Decision)

**Decision**: Single org per user (MVP)

**Rationale**:
- Simpler implementation
- Clearer UX (no org switching needed)
- Sufficient for MVP use cases

**Implementation**:
- One `profiles` record per user (enforced by PK)
- No active-org selection UI needed
- No org switching logic needed

**Future Consideration** (Post-MVP):
- Support multiple `profiles` records per user (one per org)
- Add `is_primary` flag or active org selection
- Add org switching UI
- Update RLS policies to support multi-org context

### 5.7 Documentation Requirements

**Deliverables**:
- [ ] `profiles` table schema documented with DDL
- [ ] Role enum defined and documented
- [ ] Profile lifecycle flows documented
- [ ] RLS policies implemented
- [ ] Multi-org policy decision documented
- [ ] Reference from CRM stories describing how CRM tables use `org_id`

---

## 6. Story AUTH-003: Define Auth Context Conventions

### 6.1 Overview

This story defines the standard conventions for deriving authentication context (user ID, org ID, role) across all layers of the application: database RLS policies, Edge Functions, and Next.js Server Actions/Components.

### 6.2 Core Principles

1. **Never Trust Client-Provided `org_id`**: Always derive from authenticated user's profile
2. **Single Source of Truth**: `profiles` table is the authoritative source for org/role mapping
3. **Consistent Patterns**: Use the same helper functions/patterns across all layers
4. **Security First**: Fail closed (return NULL/error) if context cannot be determined

### 6.3 Database-Level Conventions (RLS Policies)

#### 6.3.1 Helper Functions

**Purpose**: Provide consistent, secure access to user context in RLS policies.

**Function: `get_user_org_id()`**

```sql
-- Get current user's organization ID
CREATE OR REPLACE FUNCTION get_user_org_id()
RETURNS UUID AS $$
BEGIN
  RETURN (
    SELECT org_id
    FROM profiles
    WHERE id = auth.uid()
    AND is_active = true
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;
```

**Usage in RLS Policies**:
```sql
CREATE POLICY "table_select_org"
ON table_name FOR SELECT
USING (org_id = get_user_org_id());
```

**Behavior**:
- Returns `UUID` if user has active profile
- Returns `NULL` if user has no profile or profile is inactive
- Uses `SECURITY DEFINER` to bypass RLS on `profiles` table (necessary for RLS policies)
- Marked `STABLE` for query optimization

**Function: `get_user_role()`**

```sql
-- Get current user's role
CREATE OR REPLACE FUNCTION get_user_role()
RETURNS user_role_enum AS $$
BEGIN
  RETURN (
    SELECT role
    FROM profiles
    WHERE id = auth.uid()
    AND is_active = true
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;
```

**Usage in RLS Policies**:
```sql
CREATE POLICY "table_update_admin_only"
ON table_name FOR UPDATE
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('owner', 'admin')
);
```

**Function: `is_user_in_org(org_uuid UUID)`**

```sql
-- Check if current user belongs to specified org
CREATE OR REPLACE FUNCTION is_user_in_org(org_uuid UUID)
RETURNS BOOLEAN AS $$
BEGIN
  RETURN (
    SELECT EXISTS (
      SELECT 1
      FROM profiles
      WHERE id = auth.uid()
      AND org_id = org_uuid
      AND is_active = true
    )
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;
```

**Usage**: For complex RLS policies requiring org membership checks.

**Function: `has_role(role_name user_role_enum)`**

```sql
-- Check if current user has specific role
CREATE OR REPLACE FUNCTION has_role(role_name user_role_enum)
RETURNS BOOLEAN AS $$
BEGIN
  RETURN (
    SELECT EXISTS (
      SELECT 1
      FROM profiles
      WHERE id = auth.uid()
      AND role = role_name
      AND is_active = true
    )
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;
```

**Usage**: For role-based RLS policies.

**Function: `has_any_role(role_names user_role_enum[])`**

```sql
-- Check if current user has any of the specified roles
CREATE OR REPLACE FUNCTION has_any_role(role_names user_role_enum[])
RETURNS BOOLEAN AS $$
BEGIN
  RETURN (
    SELECT EXISTS (
      SELECT 1
      FROM profiles
      WHERE id = auth.uid()
      AND role = ANY(role_names)
      AND is_active = true
    )
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;
```

**Usage**: For policies allowing multiple roles.

#### 6.3.2 RLS Policy Patterns

**Pattern 1: Base Org-Scoping** (Most Common):
```sql
CREATE POLICY "table_select_org"
ON table_name FOR SELECT
USING (org_id = get_user_org_id());
```

**Pattern 2: Role-Based Access**:
```sql
CREATE POLICY "table_update_admin_only"
ON table_name FOR UPDATE
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('owner', 'admin')
);
```

**Pattern 3: Self + Org Access**:
```sql
CREATE POLICY "table_select_self_or_org"
ON table_name FOR SELECT
USING (
  user_id = auth.uid() OR
  (org_id = get_user_org_id() AND get_user_role() IN ('admin', 'manager'))
);
```

**Pattern 4: Conditional Based on Role**:
```sql
CREATE POLICY "table_select_role_based"
ON table_name FOR SELECT
USING (
  org_id = get_user_org_id() AND
  (
    get_user_role() IN ('admin', 'manager', 'csr') OR
    (get_user_role() = 'technician' AND <additional_condition>)
  )
);
```

### 6.4 Edge Functions Conventions

#### 6.4.1 Context Extraction Pattern

**Standard Pattern**:
```typescript
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

Deno.serve(async (req) => {
  // Create Supabase client with service role (bypasses RLS)
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL') ?? '',
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? ''
  );

  // Extract auth token from request
  const authHeader = req.headers.get('Authorization');
  if (!authHeader) {
    return new Response(JSON.stringify({ error: 'Unauthorized' }), { status: 401 });
  }

  const token = authHeader.replace('Bearer ', '');
  
  // Verify token and get user
  const { data: { user }, error: authError } = await supabase.auth.getUser(token);
  if (authError || !user) {
    return new Response(JSON.stringify({ error: 'Unauthorized' }), { status: 401 });
  }

  // Get user's profile (org_id and role)
  const { data: profile, error: profileError } = await supabase
    .from('profiles')
    .select('org_id, role, is_active')
    .eq('id', user.id)
    .single();

  if (profileError || !profile || !profile.is_active) {
    return new Response(JSON.stringify({ error: 'Forbidden' }), { status: 403 });
  }

  // Use profile.org_id and profile.role for authorization
  const orgId = profile.org_id;
  const role = profile.role;

  // Continue with function logic...
});
```

#### 6.4.2 Helper Function Pattern

**Create reusable helper**:
```typescript
// lib/edge-functions/auth-context.ts
export async function getAuthContext(
  supabase: SupabaseClient,
  authHeader: string | null
): Promise<{ userId: string; orgId: string; role: string } | null> {
  if (!authHeader) return null;

  const token = authHeader.replace('Bearer ', '');
  const { data: { user }, error: authError } = await supabase.auth.getUser(token);
  
  if (authError || !user) return null;

  const { data: profile, error: profileError } = await supabase
    .from('profiles')
    .select('org_id, role, is_active')
    .eq('id', user.id)
    .single();

  if (profileError || !profile || !profile.is_active) return null;

  return {
    userId: user.id,
    orgId: profile.org_id,
    role: profile.role,
  };
}
```

### 6.5 Next.js Server Actions Conventions

#### 6.5.1 Standard Pattern

**Server Action Template**:
```typescript
'use server';

import { createClient } from '@/lib/supabase/server';
import { redirect } from 'next/navigation';

export async function serverActionName(params: unknown) {
  // Get Supabase client (with user session)
  const supabase = await createClient();

  // Get authenticated user
  const { data: { user }, error: authError } = await supabase.auth.getUser();
  if (authError || !user) {
    return { error: 'Unauthorized' };
  }

  // Get user's profile (org_id and role)
  const { data: profile, error: profileError } = await supabase
    .from('profiles')
    .select('org_id, role, is_active')
    .eq('id', user.id)
    .single();

  if (profileError || !profile || !profile.is_active) {
    return { error: 'Forbidden' };
  }

  // Use profile.org_id and profile.role for authorization
  const orgId = profile.org_id;
  const role = profile.role;

  // Validate input (never trust client-provided org_id)
  // If params include org_id, verify it matches user's org
  if (params.org_id && params.org_id !== orgId) {
    return { error: 'Forbidden' };
  }

  // Continue with action logic...
  // Always use orgId from profile, never from params
}
```

#### 6.5.2 Helper Function Pattern

**Create reusable helper**:
```typescript
// lib/auth/context.ts
import { createClient } from '@/lib/supabase/server';

export interface AuthContext {
  userId: string;
  orgId: string;
  role: string;
}

export async function getAuthContext(): Promise<AuthContext | null> {
  const supabase = await createClient();

  const { data: { user }, error: authError } = await supabase.auth.getUser();
  if (authError || !user) return null;

  const { data: profile, error: profileError } = await supabase
    .from('profiles')
    .select('org_id, role, is_active')
    .eq('id', user.id)
    .single();

  if (profileError || !profile || !profile.is_active) return null;

  return {
    userId: user.id,
    orgId: profile.org_id,
    role: profile.role,
  };
}
```

**Usage in Server Actions**:
```typescript
'use server';

import { getAuthContext } from '@/lib/auth/context';

export async function serverActionName(params: unknown) {
  const context = await getAuthContext();
  if (!context) {
    return { error: 'Unauthorized' };
  }

  // Use context.orgId and context.role
  // Never trust params.org_id
}
```

### 6.6 Next.js Client Components Conventions

#### 6.6.1 Standard Pattern

**Client Component Template**:
```typescript
'use client';

import { useEffect, useState } from 'react';
import { createClient } from '@/lib/supabase/client';

export function ClientComponent() {
  const [orgId, setOrgId] = useState<string | null>(null);
  const [role, setRole] = useState<string | null>(null);

  useEffect(() => {
    async function loadContext() {
      const supabase = createClient();
      
      const { data: { user } } = await supabase.auth.getUser();
      if (!user) return;

      const { data: profile } = await supabase
        .from('profiles')
        .select('org_id, role')
        .eq('id', user.id)
        .single();

      if (profile) {
        setOrgId(profile.org_id);
        setRole(profile.role);
      }
    }

    loadContext();
  }, []);

  // Use orgId and role for UI logic
  // Never use client-provided org_id values
}
```

#### 6.6.2 Custom Hook Pattern

**Create reusable hook**:
```typescript
// hooks/use-auth-context.ts
'use client';

import { useEffect, useState } from 'react';
import { createClient } from '@/lib/supabase/client';

export interface AuthContext {
  userId: string | null;
  orgId: string | null;
  role: string | null;
  isLoading: boolean;
}

export function useAuthContext(): AuthContext {
  const [context, setContext] = useState<AuthContext>({
    userId: null,
    orgId: null,
    role: null,
    isLoading: true,
  });

  useEffect(() => {
    async function loadContext() {
      const supabase = createClient();
      
      const { data: { user } } = await supabase.auth.getUser();
      if (!user) {
        setContext({ userId: null, orgId: null, role: null, isLoading: false });
        return;
      }

      const { data: profile } = await supabase
        .from('profiles')
        .select('org_id, role')
        .eq('id', user.id)
        .single();

      setContext({
        userId: user.id,
        orgId: profile?.org_id ?? null,
        role: profile?.role ?? null,
        isLoading: false,
      });
    }

    loadContext();
  }, []);

  return context;
}
```

**Usage**:
```typescript
'use client';

import { useAuthContext } from '@/hooks/use-auth-context';

export function ClientComponent() {
  const { orgId, role, isLoading } = useAuthContext();

  if (isLoading) return <div>Loading...</div>;
  if (!orgId) return <div>Not authenticated</div>;

  // Use orgId and role
}
```

### 6.7 Security Rules Summary

**Critical Rules**:
1. ✅ **Always derive `org_id` from authenticated user's profile**
2. ✅ **Never trust client-provided `org_id` in Server Actions**
3. ✅ **Always verify user has active profile before allowing access**
4. ✅ **Use helper functions consistently across all layers**
5. ✅ **Fail closed: Return error if context cannot be determined**
6. ✅ **Service role bypasses RLS but should still validate org_id explicitly**

**Anti-Patterns to Avoid**:
- ❌ Using `params.org_id` directly without verification
- ❌ Storing `org_id` in JWT claims (unnecessary, use profiles table)
- ❌ Bypassing profile lookup "for performance" (security risk)
- ❌ Allowing inactive users to access data

### 6.8 Documentation Requirements

**Deliverables**:
- [ ] Helper functions documented with DDL
- [ ] RLS policy patterns documented
- [ ] Edge Functions conventions documented
- [ ] Server Actions conventions documented
- [ ] Client Components conventions documented
- [ ] Security rules documented
- [ ] Reference from all CRM stories using access control

---

## 7. Migration Strategy

### 7.1 Migration Order

**Critical**: Migrations must be applied in this exact order:

1. **Create `orgs` table** (no dependencies)
2. **Create `user_role_enum`** (no dependencies)
3. **Create `profiles` table** (depends on `orgs` and `user_role_enum`)
4. **Create helper functions** (depends on `profiles` table)
5. **Enable RLS policies** (depends on helper functions)

### 7.2 Migration Files

#### 7.2.1 Migration 1: Create `orgs` Table

**File**: `supabase/migrations/20240101000000_create_orgs_table.sql`

```sql
-- Create orgs table
CREATE TABLE IF NOT EXISTS orgs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_orgs_name_not_empty CHECK (length(trim(name)) > 0)
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_orgs_name ON orgs(name);
CREATE INDEX IF NOT EXISTS idx_orgs_is_active ON orgs(is_active) WHERE is_active = true;
CREATE INDEX IF NOT EXISTS idx_orgs_created_at ON orgs(created_at);

-- Trigger for updated_at
CREATE OR REPLACE FUNCTION update_orgs_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_orgs_updated_at
  BEFORE UPDATE ON orgs
  FOR EACH ROW
  EXECUTE FUNCTION update_orgs_updated_at();

-- Enable RLS
ALTER TABLE orgs ENABLE ROW LEVEL SECURITY;

-- RLS Policies (basic - will be enhanced after helper functions exist)
-- Note: These policies will be updated in migration 5 after helper functions exist
CREATE POLICY "orgs_select_own"
ON orgs FOR SELECT
USING (true);  -- Temporary: Will be updated to use get_user_org_id()
```

#### 7.2.2 Migration 2: Create `user_role_enum`

**File**: `supabase/migrations/20240101000001_create_user_role_enum.sql`

```sql
-- Create user_role_enum
CREATE TYPE user_role_enum AS ENUM (
  'owner',
  'admin',
  'manager',
  'csr',
  'dispatcher',
  'technician',
  'viewer'
);

-- Add comment
COMMENT ON TYPE user_role_enum IS 'User roles for platform access control';
```

#### 7.2.3 Migration 3: Create `profiles` Table

**File**: `supabase/migrations/20240101000002_create_profiles_table.sql`

```sql
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
  ),
  CONSTRAINT chk_profiles_phone_format CHECK (
    phone IS NULL OR phone ~* '^\+?[1-9]\d{1,14}$'
  )
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_profiles_org_id ON profiles(org_id);
CREATE INDEX IF NOT EXISTS idx_profiles_role ON profiles(role);
CREATE INDEX IF NOT EXISTS idx_profiles_org_id_role ON profiles(org_id, role);
CREATE INDEX IF NOT EXISTS idx_profiles_email ON profiles(email) WHERE email IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_profiles_is_active ON profiles(is_active) WHERE is_active = true;
CREATE INDEX IF NOT EXISTS idx_profiles_last_login_at ON profiles(last_login_at) WHERE last_login_at IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_profiles_org_active ON profiles(org_id, is_active) WHERE is_active = true;

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

#### 7.2.4 Migration 4: Create Helper Functions

**File**: `supabase/migrations/20240101000003_create_helper_functions.sql`

```sql
-- Get current user's organization ID
CREATE OR REPLACE FUNCTION get_user_org_id()
RETURNS UUID AS $$
BEGIN
  RETURN (
    SELECT org_id
    FROM profiles
    WHERE id = auth.uid()
    AND is_active = true
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;

-- Get current user's role
CREATE OR REPLACE FUNCTION get_user_role()
RETURNS user_role_enum AS $$
BEGIN
  RETURN (
    SELECT role
    FROM profiles
    WHERE id = auth.uid()
    AND is_active = true
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;

-- Check if current user belongs to specified org
CREATE OR REPLACE FUNCTION is_user_in_org(org_uuid UUID)
RETURNS BOOLEAN AS $$
BEGIN
  RETURN (
    SELECT EXISTS (
      SELECT 1
      FROM profiles
      WHERE id = auth.uid()
      AND org_id = org_uuid
      AND is_active = true
    )
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;

-- Check if current user has specific role
CREATE OR REPLACE FUNCTION has_role(role_name user_role_enum)
RETURNS BOOLEAN AS $$
BEGIN
  RETURN (
    SELECT EXISTS (
      SELECT 1
      FROM profiles
      WHERE id = auth.uid()
      AND role = role_name
      AND is_active = true
    )
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;

-- Check if current user has any of the specified roles
CREATE OR REPLACE FUNCTION has_any_role(role_names user_role_enum[])
RETURNS BOOLEAN AS $$
BEGIN
  RETURN (
    SELECT EXISTS (
      SELECT 1
      FROM profiles
      WHERE id = auth.uid()
      AND role = ANY(role_names)
      AND is_active = true
    )
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;

-- Add comments
COMMENT ON FUNCTION get_user_org_id() IS 'Returns the organization ID for the current authenticated user';
COMMENT ON FUNCTION get_user_role() IS 'Returns the role for the current authenticated user';
COMMENT ON FUNCTION is_user_in_org(UUID) IS 'Checks if the current user belongs to the specified organization';
COMMENT ON FUNCTION has_role(user_role_enum) IS 'Checks if the current user has the specified role';
COMMENT ON FUNCTION has_any_role(user_role_enum[]) IS 'Checks if the current user has any of the specified roles';
```

#### 7.2.5 Migration 5: Enable RLS Policies

**File**: `supabase/migrations/20240101000004_enable_rls_policies.sql`

```sql
-- Update orgs RLS policies (replace temporary policy)
DROP POLICY IF EXISTS "orgs_select_own" ON orgs;

CREATE POLICY "orgs_select_own"
ON orgs FOR SELECT
USING (id = get_user_org_id());

CREATE POLICY "orgs_update_own"
ON orgs FOR UPDATE
USING (
  id = get_user_org_id() AND
  EXISTS (
    SELECT 1 FROM profiles
    WHERE id = auth.uid()
    AND org_id = orgs.id
    AND role IN ('owner', 'admin')
    AND is_active = true
  )
)
WITH CHECK (
  id = get_user_org_id() AND
  -- Prevent deactivating if last owner
  (OLD.is_active = true OR NEW.is_active = true OR
   EXISTS (
     SELECT 1 FROM profiles
     WHERE org_id = orgs.id
     AND role = 'owner'
     AND is_active = true
     AND id != auth.uid()
   ))
);

-- Enable RLS on profiles table
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

-- Profiles RLS policies
CREATE POLICY "profiles_select_own"
ON profiles FOR SELECT
USING (id = auth.uid());

CREATE POLICY "profiles_select_org"
ON profiles FOR SELECT
USING (
  org_id = get_user_org_id() AND
  EXISTS (
    SELECT 1 FROM profiles p
    WHERE p.id = auth.uid()
    AND p.org_id = profiles.org_id
    AND p.is_active = true
  )
);

CREATE POLICY "profiles_update_own"
ON profiles FOR UPDATE
USING (id = auth.uid())
WITH CHECK (
  id = auth.uid() AND
  org_id = OLD.org_id AND
  role = OLD.role
);

CREATE POLICY "profiles_update_org"
ON profiles FOR UPDATE
USING (
  org_id = get_user_org_id() AND
  EXISTS (
    SELECT 1 FROM profiles p
    WHERE p.id = auth.uid()
    AND p.org_id = profiles.org_id
    AND p.role IN ('owner', 'admin')
    AND p.is_active = true
  )
)
WITH CHECK (
  org_id = get_user_org_id() AND
  -- Prevent removing last owner
  (OLD.role != 'owner' OR NEW.role = 'owner' OR
   EXISTS (
     SELECT 1 FROM profiles
     WHERE org_id = profiles.org_id
     AND role = 'owner'
     AND is_active = true
     AND id != profiles.id
   ))
);

-- Service role can insert profiles (for onboarding via Server Actions)
CREATE POLICY "profiles_insert_service"
ON profiles FOR INSERT
WITH CHECK (true);
```

### 7.3 Applying Migrations

**Local Development**:
```bash
supabase db reset  # Applies all migrations
```

**Production**:
```bash
supabase db push  # Applies pending migrations
```

### 7.4 Rollback Strategy

**Note**: Supabase migrations are forward-only by default. For rollbacks:
1. Create new migration to reverse changes
2. Or restore from backup
3. Or manually reverse DDL statements

**Best Practice**: Test migrations thoroughly in local/staging before production.

---

## 8. Testing Requirements

### 8.1 Unit Tests (Database Functions)

**Test: `get_user_org_id()`**
- [ ] Returns correct org_id for authenticated user with profile
- [ ] Returns NULL for authenticated user without profile
- [ ] Returns NULL for inactive profile
- [ ] Returns NULL for unauthenticated user

**Test: `get_user_role()`**
- [ ] Returns correct role for authenticated user with profile
- [ ] Returns NULL for authenticated user without profile
- [ ] Returns NULL for inactive profile

**Test: `has_role()`**
- [ ] Returns true for user with matching role
- [ ] Returns false for user with different role
- [ ] Returns false for user without profile

### 8.2 Integration Tests (RLS Policies)

**Test: Org Isolation**
- [ ] User in Org A cannot see Org B's data
- [ ] User in Org A cannot insert data into Org B
- [ ] User in Org A cannot update Org B's data
- [ ] User in Org A cannot delete Org B's data

**Test: Profile Access**
- [ ] User can read own profile
- [ ] User can read profiles in their org
- [ ] User cannot read profiles in other orgs
- [ ] User can update own profile (limited fields)
- [ ] Admin can update profiles in their org
- [ ] User cannot update profiles in other orgs

**Test: Org Access**
- [ ] User can read their own org
- [ ] User cannot read other orgs
- [ ] Owner/admin can update their org
- [ ] User cannot update other orgs

### 8.3 End-to-End Tests

**Test: Onboarding Flow**
- [ ] User signs up → creates auth.users record
- [ ] User completes onboarding → creates orgs record
- [ ] System creates profiles record with role 'owner'
- [ ] User can access app with correct org context

**Test: Multi-Tenant Isolation**
- [ ] Create two orgs (Org A, Org B)
- [ ] Create users in each org
- [ ] Verify users can only access their org's data
- [ ] Verify RLS policies prevent cross-org access

### 8.4 Test Data Setup

**Seed Data Script** (`supabase/seed.sql`):

```sql
-- Create test organizations
INSERT INTO orgs (id, name) VALUES
  ('00000000-0000-0000-0000-000000000001', 'Test HVAC Company A'),
  ('00000000-0000-0000-0000-000000000002', 'Test HVAC Company B')
ON CONFLICT (id) DO NOTHING;

-- Note: Profiles are created via actual user sign-up flow
-- Test profiles will be created when test users sign up
```

### 8.5 Manual Testing Checklist

**Story AUTH-000**:
- [ ] Tenancy decision documented
- [ ] Naming conventions documented
- [ ] RLS strategy documented

**Story AUTH-001**:
- [ ] `orgs` table created
- [ ] Can create org via SQL
- [ ] Can update org name
- [ ] Can deactivate org (soft delete)
- [ ] RLS policies prevent cross-org access

**Story AUTH-002**:
- [ ] `user_role_enum` created with all roles
- [ ] `profiles` table created
- [ ] Can create profile via SQL
- [ ] Can update profile
- [ ] Can deactivate profile
- [ ] RLS policies enforce org isolation

**Story AUTH-003**:
- [ ] Helper functions created
- [ ] Helper functions return correct values
- [ ] RLS policies use helper functions correctly
- [ ] Server Actions pattern documented
- [ ] Edge Functions pattern documented

---

## 9. TypeScript Type Definitions

### 9.1 Database Types

**File**: `types/database.ts` (generated via `supabase gen types`)

After running migrations, generate types:
```bash
supabase gen types typescript --local > types/database.ts
```

### 9.2 Application Types

**File**: `types/auth.ts`:

```typescript
export type UserRole = 
  | 'owner'
  | 'admin'
  | 'manager'
  | 'csr'
  | 'dispatcher'
  | 'technician'
  | 'viewer';

export interface Profile {
  id: string;
  org_id: string;
  role: UserRole;
  first_name: string | null;
  last_name: string | null;
  email: string | null;
  phone: string | null;
  avatar_url: string | null;
  is_active: boolean;
  last_login_at: string | null;
  created_at: string;
  updated_at: string;
}

export interface Organization {
  id: string;
  name: string;
  is_active: boolean;
  created_at: string;
  updated_at: string;
}

export interface AuthContext {
  userId: string;
  orgId: string;
  role: UserRole;
}
```

---

## 10. Definition of Done Checklist

### Story AUTH-000: Tenancy Strategy
- [ ] Tenancy decision documented (single project, multi-tenant by org_id)
- [ ] Naming conventions documented
- [ ] RLS strategy pattern documented
- [ ] Onboarding implications documented
- [ ] Support access policy documented (future consideration)
- [ ] Reference added to CRM Epic 1 stories

### Story AUTH-001: Orgs Table
- [ ] `orgs` table created with all required fields
- [ ] Indexes created
- [ ] Triggers created (updated_at)
- [ ] RLS policies implemented
- [ ] Lifecycle flows documented (create, update, deactivate)
- [ ] Migration file created and tested

### Story AUTH-002: Profiles Table
- [ ] `user_role_enum` created with all roles
- [ ] `profiles` table created with all required fields
- [ ] Indexes created
- [ ] Triggers created (updated_at)
- [ ] RLS policies implemented
- [ ] Profile lifecycle flows documented
- [ ] Multi-org policy decision documented (single org per user)
- [ ] Migration file created and tested

### Story AUTH-003: Auth Context Conventions
- [ ] Helper functions created (`get_user_org_id`, `get_user_role`, etc.)
- [ ] RLS policy patterns documented
- [ ] Edge Functions conventions documented
- [ ] Server Actions conventions documented
- [ ] Client Components conventions documented
- [ ] Security rules documented
- [ ] Migration file created and tested

### Overall Epic 0
- [ ] All migrations applied successfully
- [ ] RLS policies tested (org isolation verified)
- [ ] Helper functions tested
- [ ] TypeScript types generated
- [ ] Documentation complete
- [ ] Ready for Epic 1 (Authentication flows)

---

## 11. References

- **Source Stories**: `docs/functional/fdd_0_agile.md` Epic 0
- **Platform Docs**: `docs/technical/tooling.md`
- **Supabase Docs**: https://supabase.com/docs/guides/auth/row-level-security
- **Next.js Docs**: https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations

---

## 12. Appendix: Complete SQL Scripts

### 12.1 Complete Migration Script (All-in-One)

For reference, here is the complete SQL script combining all migrations:

```sql
-- ============================================
-- Epic 0: Platform Identity Model & Tenancy Primitives
-- Complete Migration Script
-- ============================================

-- Migration 1: Create orgs table
CREATE TABLE IF NOT EXISTS orgs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_orgs_name_not_empty CHECK (length(trim(name)) > 0)
);

CREATE INDEX IF NOT EXISTS idx_orgs_name ON orgs(name);
CREATE INDEX IF NOT EXISTS idx_orgs_is_active ON orgs(is_active) WHERE is_active = true;
CREATE INDEX IF NOT EXISTS idx_orgs_created_at ON orgs(created_at);

CREATE OR REPLACE FUNCTION update_orgs_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_orgs_updated_at
  BEFORE UPDATE ON orgs
  FOR EACH ROW
  EXECUTE FUNCTION update_orgs_updated_at();

-- Migration 2: Create user_role_enum
CREATE TYPE user_role_enum AS ENUM (
  'owner',
  'admin',
  'manager',
  'csr',
  'dispatcher',
  'technician',
  'viewer'
);

COMMENT ON TYPE user_role_enum IS 'User roles for platform access control';

-- Migration 3: Create profiles table
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
  ),
  CONSTRAINT chk_profiles_phone_format CHECK (
    phone IS NULL OR phone ~* '^\+?[1-9]\d{1,14}$'
  )
);

CREATE INDEX IF NOT EXISTS idx_profiles_org_id ON profiles(org_id);
CREATE INDEX IF NOT EXISTS idx_profiles_role ON profiles(role);
CREATE INDEX IF NOT EXISTS idx_profiles_org_id_role ON profiles(org_id, role);
CREATE INDEX IF NOT EXISTS idx_profiles_email ON profiles(email) WHERE email IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_profiles_is_active ON profiles(is_active) WHERE is_active = true;
CREATE INDEX IF NOT EXISTS idx_profiles_last_login_at ON profiles(last_login_at) WHERE last_login_at IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_profiles_org_active ON profiles(org_id, is_active) WHERE is_active = true;

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

-- Migration 4: Create helper functions
CREATE OR REPLACE FUNCTION get_user_org_id()
RETURNS UUID AS $$
BEGIN
  RETURN (
    SELECT org_id
    FROM profiles
    WHERE id = auth.uid()
    AND is_active = true
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;

CREATE OR REPLACE FUNCTION get_user_role()
RETURNS user_role_enum AS $$
BEGIN
  RETURN (
    SELECT role
    FROM profiles
    WHERE id = auth.uid()
    AND is_active = true
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;

CREATE OR REPLACE FUNCTION is_user_in_org(org_uuid UUID)
RETURNS BOOLEAN AS $$
BEGIN
  RETURN (
    SELECT EXISTS (
      SELECT 1
      FROM profiles
      WHERE id = auth.uid()
      AND org_id = org_uuid
      AND is_active = true
    )
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;

CREATE OR REPLACE FUNCTION has_role(role_name user_role_enum)
RETURNS BOOLEAN AS $$
BEGIN
  RETURN (
    SELECT EXISTS (
      SELECT 1
      FROM profiles
      WHERE id = auth.uid()
      AND role = role_name
      AND is_active = true
    )
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;

CREATE OR REPLACE FUNCTION has_any_role(role_names user_role_enum[])
RETURNS BOOLEAN AS $$
BEGIN
  RETURN (
    SELECT EXISTS (
      SELECT 1
      FROM profiles
      WHERE id = auth.uid()
      AND role = ANY(role_names)
      AND is_active = true
    )
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;

COMMENT ON FUNCTION get_user_org_id() IS 'Returns the organization ID for the current authenticated user';
COMMENT ON FUNCTION get_user_role() IS 'Returns the role for the current authenticated user';
COMMENT ON FUNCTION is_user_in_org(UUID) IS 'Checks if the current user belongs to the specified organization';
COMMENT ON FUNCTION has_role(user_role_enum) IS 'Checks if the current user has the specified role';
COMMENT ON FUNCTION has_any_role(user_role_enum[]) IS 'Checks if the current user has any of the specified roles';

-- Migration 5: Enable RLS policies
ALTER TABLE orgs ENABLE ROW LEVEL SECURITY;

DROP POLICY IF EXISTS "orgs_select_own" ON orgs;

CREATE POLICY "orgs_select_own"
ON orgs FOR SELECT
USING (id = get_user_org_id());

CREATE POLICY "orgs_update_own"
ON orgs FOR UPDATE
USING (
  id = get_user_org_id() AND
  EXISTS (
    SELECT 1 FROM profiles
    WHERE id = auth.uid()
    AND org_id = orgs.id
    AND role IN ('owner', 'admin')
    AND is_active = true
  )
)
WITH CHECK (
  id = get_user_org_id() AND
  (OLD.is_active = true OR NEW.is_active = true OR
   EXISTS (
     SELECT 1 FROM profiles
     WHERE org_id = orgs.id
     AND role = 'owner'
     AND is_active = true
     AND id != auth.uid()
   ))
);

ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "profiles_select_own"
ON profiles FOR SELECT
USING (id = auth.uid());

CREATE POLICY "profiles_select_org"
ON profiles FOR SELECT
USING (
  org_id = get_user_org_id() AND
  EXISTS (
    SELECT 1 FROM profiles p
    WHERE p.id = auth.uid()
    AND p.org_id = profiles.org_id
    AND p.is_active = true
  )
);

CREATE POLICY "profiles_update_own"
ON profiles FOR UPDATE
USING (id = auth.uid())
WITH CHECK (
  id = auth.uid() AND
  org_id = OLD.org_id AND
  role = OLD.role
);

CREATE POLICY "profiles_update_org"
ON profiles FOR UPDATE
USING (
  org_id = get_user_org_id() AND
  EXISTS (
    SELECT 1 FROM profiles p
    WHERE p.id = auth.uid()
    AND p.org_id = profiles.org_id
    AND p.role IN ('owner', 'admin')
    AND p.is_active = true
  )
)
WITH CHECK (
  org_id = get_user_org_id() AND
  (OLD.role != 'owner' OR NEW.role = 'owner' OR
   EXISTS (
     SELECT 1 FROM profiles
     WHERE org_id = profiles.org_id
     AND role = 'owner'
     AND is_active = true
     AND id != profiles.id
   ))
);

CREATE POLICY "profiles_insert_service"
ON profiles FOR INSERT
WITH CHECK (true);
```

---

**End of Document**

