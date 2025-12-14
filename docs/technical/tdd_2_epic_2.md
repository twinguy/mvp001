# Technical Design Document – Epic 2: Core Scheduling Data Model (Supabase Postgres Schema)

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 2 – Core Scheduling Data Model (Supabase Postgres Schema)
- **Source**: Derived from `fdd_2_agile.md` Epic 2 (Stories DISP-003 through DISP-016)
- **Reference Documents**: 
  - `fdd_2.md` (Functional Design Document for Scheduling & Dispatch)
  - `fdd_2_agile.md` (Agile User Stories)
  - `tdd_2_epic_1.md` (Dispatch Epic 1 for tenancy/role conventions)
  - `tdd_1_epic_1.md` (CRM Epic 1 for schema patterns)
- **Target Platform**: Supabase (PostgreSQL 15+)
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing the core Scheduling & Dispatch data model in Supabase PostgreSQL. It covers all 14 database tables required for Epic 2, including:

- Technician configuration tables (profiles, skills, zones, shifts, time-off)
- Job and assignment tables (dispatch_jobs, job_time_windows, job_assignments)
- Routing tables (route_plans, route_stops)
- Calendar integration tables (calendar_integrations, calendar_events)
- Notification table (job_notifications)

All specifications are designed to be directly implementable via SQL migrations in Supabase, with exact data types, constraints, indexes, and relationships defined. This epic assumes Epic 1 (tenancy and roles) is complete.

---

## 2. Prerequisites

### 2.1 Required Tables

Before implementing Epic 2, ensure these tables exist:

1. **`orgs`** (from CRM module or Epic 1)
2. **`profiles`** (user-to-org mapping with roles, from Epic 1)
3. **`auth.users`** (Supabase Auth)
4. **`customers`** (from CRM module, referenced by `dispatch_jobs`)
5. **`customer_locations`** (from CRM module, referenced by `dispatch_jobs`)

### 2.2 Helper Functions

Epic 1 must be complete, providing:
- `get_user_org_id()` function
- `get_user_role()` function
- `is_user_technician()` function (optional)

---

## 3. Enums

All enums must be created before tables that reference them.

### 3.1 `employment_type_enum`

```sql
CREATE TYPE employment_type_enum AS ENUM (
  'employee',
  'contractor',
  'subcontractor'
);
```

**Usage**: `technician_profiles.employment_type`

### 3.2 `vehicle_type_enum`

```sql
CREATE TYPE vehicle_type_enum AS ENUM (
  'van',
  'truck',
  'car',
  'other'
);
```

**Usage**: `technician_profiles.vehicle_type`

### 3.3 `skill_proficiency_level_enum`

```sql
CREATE TYPE skill_proficiency_level_enum AS ENUM (
  'junior',
  'mid',
  'senior',
  'expert'
);
```

**Usage**: `technician_skills.proficiency_level`

### 3.4 `shift_type_enum`

```sql
CREATE TYPE shift_type_enum AS ENUM (
  'regular',
  'on_call',
  'overtime',
  'training'
);
```

**Usage**: `technician_shifts.shift_type`

### 3.5 `time_off_reason_enum`

```sql
CREATE TYPE time_off_reason_enum AS ENUM (
  'vacation',
  'sick',
  'personal',
  'other'
);
```

**Usage**: `technician_time_off.reason`

### 3.6 `job_priority_enum`

```sql
CREATE TYPE job_priority_enum AS ENUM (
  'low',
  'normal',
  'high',
  'emergency'
);
```

**Usage**: `dispatch_jobs.priority`

### 3.7 `job_status_enum`

```sql
CREATE TYPE job_status_enum AS ENUM (
  'unscheduled',
  'scheduled',
  'dispatched',
  'in_progress',
  'completed',
  'canceled'
);
```

**Usage**: `dispatch_jobs.status`

### 3.8 `time_window_source_enum`

```sql
CREATE TYPE time_window_source_enum AS ENUM (
  'system_suggested',
  'dispatcher_selected',
  'customer_selected'
);
```

**Usage**: `job_time_windows.source`

### 3.9 `assignment_status_enum`

```sql
CREATE TYPE assignment_status_enum AS ENUM (
  'assigned',
  'accepted',
  'declined',
  'en_route',
  'on_site',
  'completed',
  'no_show',
  'canceled'
);
```

**Usage**: `job_assignments.status`

### 3.10 `route_plan_status_enum`

```sql
CREATE TYPE route_plan_status_enum AS ENUM (
  'draft',
  'proposed',
  'finalized',
  'in_progress',
  'completed',
  'canceled'
);
```

**Usage**: `route_plans.status`

### 3.11 `optimization_strategy_enum`

```sql
CREATE TYPE optimization_strategy_enum AS ENUM (
  'time_minimization',
  'distance_minimization',
  'priority_first',
  'balanced'
);
```

**Usage**: `route_plans.optimization_strategy`

### 3.12 `route_generated_by_enum`

```sql
CREATE TYPE route_generated_by_enum AS ENUM (
  'manual',
  'rule_engine',
  'ai_optimizer'
);
```

**Usage**: `route_plans.generated_by`

### 3.13 `route_stop_type_enum`

```sql
CREATE TYPE route_stop_type_enum AS ENUM (
  'depot_start',
  'job',
  'break',
  'fuel',
  'depot_end',
  'other'
);
```

**Usage**: `route_stops.stop_type`

### 3.14 `calendar_provider_enum`

```sql
CREATE TYPE calendar_provider_enum AS ENUM (
  'google',
  'microsoft'
);
```

**Usage**: `calendar_integrations.provider`

### 3.15 `calendar_event_status_enum`

```sql
CREATE TYPE calendar_event_status_enum AS ENUM (
  'scheduled',
  'updated',
  'canceled',
  'deleted_by_user'
);
```

**Usage**: `calendar_events.status`

### 3.16 `calendar_sync_direction_enum`

```sql
CREATE TYPE calendar_sync_direction_enum AS ENUM (
  'internal_to_external',
  'external_to_internal',
  'bidirectional'
);
```

**Usage**: `calendar_events.sync_direction`

### 3.17 `notification_recipient_type_enum`

```sql
CREATE TYPE notification_recipient_type_enum AS ENUM (
  'customer',
  'technician',
  'dispatcher'
);
```

**Usage**: `job_notifications.recipient_type`

### 3.18 `notification_channel_enum`

```sql
CREATE TYPE notification_channel_enum AS ENUM (
  'sms',
  'email',
  'push',
  'in_app'
);
```

**Usage**: `job_notifications.channel`

### 3.19 `notification_type_enum`

```sql
CREATE TYPE notification_type_enum AS ENUM (
  'booking_confirmation',
  'pre_appointment_reminder',
  'same_day_reminder',
  'tech_on_the_way',
  'reschedule_notice',
  'cancellation_notice'
);
```

**Usage**: `job_notifications.notification_type`

### 3.20 `notification_status_enum`

```sql
CREATE TYPE notification_status_enum AS ENUM (
  'pending',
  'sent',
  'failed',
  'canceled'
);
```

**Usage**: `job_notifications.status`

---

## 4. Core Tables

### 4.1 `technician_profiles` Table

**Purpose**: Core technician entity for scheduling and dispatch.

**DDL**:

```sql
CREATE TABLE technician_profiles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  display_name TEXT NOT NULL,
  employment_type employment_type_enum,
  is_active BOOLEAN NOT NULL DEFAULT true,
  home_base_location_id UUID REFERENCES customer_locations(id) ON DELETE SET NULL,
  default_service_zone_id UUID, -- FK added after service_zones table exists
  max_daily_work_minutes INTEGER,
  max_concurrent_jobs INTEGER NOT NULL DEFAULT 1,
  vehicle_type vehicle_type_enum,
  metadata JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_technician_profiles_max_daily_work_minutes CHECK (
    max_daily_work_minutes IS NULL OR max_daily_work_minutes > 0
  ),
  CONSTRAINT chk_technician_profiles_max_concurrent_jobs CHECK (
    max_concurrent_jobs > 0
  )
);

-- Unique constraint: one profile per user per org
CREATE UNIQUE INDEX idx_technician_profiles_unique_user_org 
  ON technician_profiles(org_id, user_id);

-- Indexes
CREATE INDEX idx_technician_profiles_org_id ON technician_profiles(org_id);
CREATE INDEX idx_technician_profiles_user_id ON technician_profiles(user_id);
CREATE INDEX idx_technician_profiles_org_id_is_active ON technician_profiles(org_id, is_active) 
  WHERE is_active = true;
CREATE INDEX idx_technician_profiles_home_base_location_id ON technician_profiles(home_base_location_id) 
  WHERE home_base_location_id IS NOT NULL;

-- Trigger for updated_at
CREATE OR REPLACE FUNCTION update_technician_profiles_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_technician_profiles_updated_at
  BEFORE UPDATE ON technician_profiles
  FOR EACH ROW
  EXECUTE FUNCTION update_technician_profiles_updated_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `user_id` | UUID | NO | - | FK to `auth.users.id` (unique per org) |
| `display_name` | TEXT | NO | - | Technician display name |
| `employment_type` | employment_type_enum | YES | NULL | Employment classification |
| `is_active` | BOOLEAN | NO | `true` | Active status |
| `home_base_location_id` | UUID | YES | NULL | FK to `customer_locations.id` (home depot) |
| `default_service_zone_id` | UUID | YES | NULL | FK to `service_zones.id` (deferred) |
| `max_daily_work_minutes` | INTEGER | YES | NULL | Daily capacity limit |
| `max_concurrent_jobs` | INTEGER | NO | `1` | Max overlapping assignments |
| `vehicle_type` | vehicle_type_enum | YES | NULL | Vehicle type |
| `metadata` | JSONB | YES | NULL | Additional data (certifications, etc.) |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- One technician profile per user per org (enforced via unique index)
- `max_daily_work_minutes` must be positive if provided
- `max_concurrent_jobs` must be positive (default 1)

**Deferred FK**: `default_service_zone_id` FK added after `service_zones` table exists.

---

### 4.2 `technician_skills` Table

**Purpose**: Skills and certifications per technician for job matching.

**DDL**:

```sql
CREATE TABLE technician_skills (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  technician_id UUID NOT NULL REFERENCES technician_profiles(id) ON DELETE CASCADE,
  skill_code TEXT NOT NULL,
  proficiency_level skill_proficiency_level_enum NOT NULL DEFAULT 'mid',
  is_primary BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Unique constraint: one skill per technician per code per org
CREATE UNIQUE INDEX idx_technician_skills_unique 
  ON technician_skills(org_id, technician_id, skill_code);

-- Indexes
CREATE INDEX idx_technician_skills_org_id ON technician_skills(org_id);
CREATE INDEX idx_technician_skills_technician_id ON technician_skills(technician_id);
CREATE INDEX idx_technician_skills_org_id_skill_code ON technician_skills(org_id, skill_code);
CREATE INDEX idx_technician_skills_org_id_technician_id ON technician_skills(org_id, technician_id);
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `technician_id` | UUID | NO | - | FK to `technician_profiles.id` (CASCADE delete) |
| `skill_code` | TEXT | NO | - | Skill identifier (e.g., `hvac_install`) |
| `proficiency_level` | skill_proficiency_level_enum | NO | `'mid'` | Proficiency level |
| `is_primary` | BOOLEAN | NO | `false` | Primary skill flag |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |

**Business Rules**:
- One skill per technician per code per org (enforced via unique index)
- Skill codes are org-specific (same code can exist in different orgs)

---

### 4.3 `service_zones` Table

**Purpose**: Service areas or territories for dispatch optimization.

**DDL**:

```sql
CREATE TABLE service_zones (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  description TEXT,
  polygon JSONB, -- Geometry stored as GeoJSON or PostGIS geometry (decision documented below)
  postal_codes TEXT[],
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Unique constraint: zone name unique per org
CREATE UNIQUE INDEX idx_service_zones_unique_name 
  ON service_zones(org_id, name);

-- Indexes
CREATE INDEX idx_service_zones_org_id ON service_zones(org_id);
CREATE INDEX idx_service_zones_org_id_is_active ON service_zones(org_id, is_active) 
  WHERE is_active = true;
CREATE INDEX idx_service_zones_postal_codes ON service_zones USING gin(postal_codes) 
  WHERE postal_codes IS NOT NULL;

-- Trigger for updated_at
CREATE TRIGGER trigger_service_zones_updated_at
  BEFORE UPDATE ON service_zones
  FOR EACH ROW
  EXECUTE FUNCTION update_technician_profiles_updated_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `name` | TEXT | NO | - | Zone name (unique per org) |
| `description` | TEXT | YES | NULL | Zone description |
| `polygon` | JSONB | YES | NULL | Zone geometry (GeoJSON format) |
| `postal_codes` | TEXT[] | YES | NULL | Alternative: postal code array |
| `is_active` | BOOLEAN | NO | `true` | Active status |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Geometry Storage Decision**:

**Decision**: Use `JSONB` for `polygon` field storing GeoJSON format.

**Rationale**:
- No PostGIS extension required (simpler setup)
- GeoJSON is portable and widely supported
- Application layer handles geometry operations
- Can migrate to PostGIS later if needed

**GeoJSON Format Example**:
```json
{
  "type": "Polygon",
  "coordinates": [[
    [-122.5, 37.7],
    [-122.4, 37.7],
    [-122.4, 37.8],
    [-122.5, 37.8],
    [-122.5, 37.7]
  ]]
}
```

**Alternative**: If PostGIS is available, use `GEOGRAPHY(POLYGON, 4326)` type and GIST index.

---

### 4.4 `technician_service_zones` Table

**Purpose**: Many-to-many mapping between technicians and service zones.

**DDL**:

```sql
CREATE TABLE technician_service_zones (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  technician_id UUID NOT NULL REFERENCES technician_profiles(id) ON DELETE CASCADE,
  service_zone_id UUID NOT NULL REFERENCES service_zones(id) ON DELETE CASCADE,
  is_primary BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Unique constraint: one zone assignment per technician per zone per org
CREATE UNIQUE INDEX idx_technician_service_zones_unique 
  ON technician_service_zones(org_id, technician_id, service_zone_id);

-- Indexes
CREATE INDEX idx_technician_service_zones_org_id ON technician_service_zones(org_id);
CREATE INDEX idx_technician_service_zones_technician_id ON technician_service_zones(technician_id);
CREATE INDEX idx_technician_service_zones_service_zone_id ON technician_service_zones(service_zone_id);
CREATE INDEX idx_technician_service_zones_org_id_technician_id ON technician_service_zones(org_id, technician_id);
CREATE INDEX idx_technician_service_zones_org_id_service_zone_id ON technician_service_zones(org_id, service_zone_id);

-- Partial unique index: at most one primary zone per technician per org
CREATE UNIQUE INDEX idx_technician_service_zones_unique_primary 
  ON technician_service_zones(org_id, technician_id) 
  WHERE is_primary = true;
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `technician_id` | UUID | NO | - | FK to `technician_profiles.id` (CASCADE delete) |
| `service_zone_id` | UUID | NO | - | FK to `service_zones.id` (CASCADE delete) |
| `is_primary` | BOOLEAN | NO | `false` | Primary zone flag |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |

**Business Rules**:
- One zone assignment per technician per zone per org (enforced via unique index)
- At most one primary zone per technician per org (enforced via partial unique index)

---

### 4.5 `technician_shifts` Table

**Purpose**: Scheduled working time blocks for technicians.

**DDL**:

```sql
CREATE TABLE technician_shifts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  technician_id UUID NOT NULL REFERENCES technician_profiles(id) ON DELETE CASCADE,
  starts_at TIMESTAMPTZ NOT NULL,
  ends_at TIMESTAMPTZ NOT NULL,
  shift_type shift_type_enum NOT NULL DEFAULT 'regular',
  recurrence_rule TEXT, -- iCal RRULE string (e.g., "FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR")
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_by_user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_technician_shifts_ends_after_starts CHECK (
    ends_at > starts_at
  )
);

-- Indexes
CREATE INDEX idx_technician_shifts_org_id ON technician_shifts(org_id);
CREATE INDEX idx_technician_shifts_technician_id ON technician_shifts(technician_id);
CREATE INDEX idx_technician_shifts_org_id_technician_id_starts_at 
  ON technician_shifts(org_id, technician_id, starts_at);
CREATE INDEX idx_technician_shifts_org_id_starts_at_ends_at 
  ON technician_shifts(org_id, starts_at, ends_at);
CREATE INDEX idx_technician_shifts_org_id_is_active 
  ON technician_shifts(org_id, is_active) 
  WHERE is_active = true;

-- Trigger for updated_at
CREATE TRIGGER trigger_technician_shifts_updated_at
  BEFORE UPDATE ON technician_shifts
  FOR EACH ROW
  EXECUTE FUNCTION update_technician_profiles_updated_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `technician_id` | UUID | NO | - | FK to `technician_profiles.id` |
| `starts_at` | TIMESTAMPTZ | NO | - | Shift start time |
| `ends_at` | TIMESTAMPTZ | NO | - | Shift end time |
| `shift_type` | shift_type_enum | NO | `'regular'` | Shift type |
| `recurrence_rule` | TEXT | YES | NULL | iCal RRULE string for repeating shifts |
| `is_active` | BOOLEAN | NO | `true` | Active status |
| `created_by_user_id` | UUID | YES | NULL | FK to `auth.users.id` |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- `ends_at` must be after `starts_at` (enforced via CHECK constraint)
- `recurrence_rule` uses iCal RRULE format (expansion handled in application layer)

**Recurrence Rule Examples**:
- Daily: `FREQ=DAILY`
- Weekdays: `FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR`
- Every 2 weeks: `FREQ=WEEKLY;INTERVAL=2`

---

### 4.6 `technician_time_off` Table

**Purpose**: Blocks of time when technicians are unavailable.

**DDL**:

```sql
CREATE TABLE technician_time_off (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  technician_id UUID NOT NULL REFERENCES technician_profiles(id) ON DELETE CASCADE,
  starts_at TIMESTAMPTZ NOT NULL,
  ends_at TIMESTAMPTZ NOT NULL,
  reason time_off_reason_enum NOT NULL DEFAULT 'personal',
  notes TEXT,
  created_by_user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_technician_time_off_ends_after_starts CHECK (
    ends_at > starts_at
  )
);

-- Indexes
CREATE INDEX idx_technician_time_off_org_id ON technician_time_off(org_id);
CREATE INDEX idx_technician_time_off_technician_id ON technician_time_off(technician_id);
CREATE INDEX idx_technician_time_off_org_id_technician_id_starts_at 
  ON technician_time_off(org_id, technician_id, starts_at);
CREATE INDEX idx_technician_time_off_org_id_starts_at_ends_at 
  ON technician_time_off(org_id, starts_at, ends_at);

-- Trigger for updated_at
CREATE TRIGGER trigger_technician_time_off_updated_at
  BEFORE UPDATE ON technician_time_off
  FOR EACH ROW
  EXECUTE FUNCTION update_technician_profiles_updated_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `technician_id` | UUID | NO | - | FK to `technician_profiles.id` |
| `starts_at` | TIMESTAMPTZ | NO | - | Time-off start time |
| `ends_at` | TIMESTAMPTZ | NO | - | Time-off end time |
| `reason` | time_off_reason_enum | NO | `'personal'` | Reason for time off |
| `notes` | TEXT | YES | NULL | Additional notes |
| `created_by_user_id` | UUID | YES | NULL | FK to `auth.users.id` |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- `ends_at` must be after `starts_at` (enforced via CHECK constraint)
- Overlapping time-off entries are allowed (handled in application logic)

---

### 4.7 `dispatch_jobs` Table

**Purpose**: Schedulable job/visit records, typically linked to Work Orders.

**DDL**:

```sql
CREATE TABLE dispatch_jobs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  related_work_order_id UUID, -- FK to work_orders table (deferred, module not yet implemented)
  customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  location_id UUID NOT NULL REFERENCES customer_locations(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  description TEXT,
  job_type TEXT, -- e.g., 'maintenance', 'install', 'repair', 'inspection'
  priority job_priority_enum NOT NULL DEFAULT 'normal',
  status job_status_enum NOT NULL DEFAULT 'unscheduled',
  estimated_duration_minutes INTEGER NOT NULL DEFAULT 60,
  required_skills TEXT[], -- Array of skill codes
  required_crew_size INTEGER NOT NULL DEFAULT 1,
  service_zone_id UUID REFERENCES service_zones(id) ON DELETE SET NULL,
  sla_start_at TIMESTAMPTZ, -- Earliest allowed start (SLA)
  sla_end_at TIMESTAMPTZ, -- Latest allowed completion (SLA)
  is_customer_booked BOOLEAN NOT NULL DEFAULT false,
  notes_internal TEXT,
  created_by_user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_dispatch_jobs_estimated_duration CHECK (
    estimated_duration_minutes > 0
  ),
  CONSTRAINT chk_dispatch_jobs_required_crew_size CHECK (
    required_crew_size > 0
  ),
  CONSTRAINT chk_dispatch_jobs_sla_window CHECK (
    (sla_start_at IS NULL AND sla_end_at IS NULL) OR
    (sla_start_at IS NOT NULL AND sla_end_at IS NOT NULL AND sla_end_at > sla_start_at)
  )
);

-- Indexes
CREATE INDEX idx_dispatch_jobs_org_id ON dispatch_jobs(org_id);
CREATE INDEX idx_dispatch_jobs_customer_id ON dispatch_jobs(customer_id);
CREATE INDEX idx_dispatch_jobs_location_id ON dispatch_jobs(location_id);
CREATE INDEX idx_dispatch_jobs_org_id_status_priority 
  ON dispatch_jobs(org_id, status, priority);
CREATE INDEX idx_dispatch_jobs_org_id_sla_start_at 
  ON dispatch_jobs(org_id, sla_start_at) 
  WHERE sla_start_at IS NOT NULL;
CREATE INDEX idx_dispatch_jobs_org_id_status 
  ON dispatch_jobs(org_id, status);
CREATE INDEX idx_dispatch_jobs_service_zone_id ON dispatch_jobs(service_zone_id) 
  WHERE service_zone_id IS NOT NULL;
CREATE INDEX idx_dispatch_jobs_required_skills ON dispatch_jobs USING gin(required_skills) 
  WHERE required_skills IS NOT NULL;

-- Full-text search index
CREATE INDEX idx_dispatch_jobs_search ON dispatch_jobs USING gin(
  to_tsvector('english', coalesce(title, '') || ' ' || coalesce(description, ''))
);

-- Trigger for updated_at
CREATE TRIGGER trigger_dispatch_jobs_updated_at
  BEFORE UPDATE ON dispatch_jobs
  FOR EACH ROW
  EXECUTE FUNCTION update_technician_profiles_updated_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `related_work_order_id` | UUID | YES | NULL | FK to work orders (deferred) |
| `customer_id` | UUID | NO | - | FK to `customers.id` |
| `location_id` | UUID | NO | - | FK to `customer_locations.id` |
| `title` | TEXT | NO | - | Job title |
| `description` | TEXT | YES | NULL | Job description |
| `job_type` | TEXT | YES | NULL | Job type |
| `priority` | job_priority_enum | NO | `'normal'` | Priority level |
| `status` | job_status_enum | NO | `'unscheduled'` | Job status |
| `estimated_duration_minutes` | INTEGER | NO | `60` | Estimated duration |
| `required_skills` | TEXT[] | YES | NULL | Required skill codes |
| `required_crew_size` | INTEGER | NO | `1` | Required crew size |
| `service_zone_id` | UUID | YES | NULL | FK to `service_zones.id` |
| `sla_start_at` | TIMESTAMPTZ | YES | NULL | SLA earliest start |
| `sla_end_at` | TIMESTAMPTZ | YES | NULL | SLA latest completion |
| `is_customer_booked` | BOOLEAN | NO | `false` | Portal booking flag |
| `notes_internal` | TEXT | YES | NULL | Internal notes |
| `created_by_user_id` | UUID | YES | NULL | FK to `auth.users.id` |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- `estimated_duration_minutes` must be positive
- `required_crew_size` must be positive
- SLA window must be valid (`sla_end_at > sla_start_at` if both provided)

**Deferred FK**: `related_work_order_id` FK added when Work Order module is implemented.

---

### 4.8 `job_time_windows` Table

**Purpose**: Appointment windows for jobs, selectable by customers or dispatchers.

**DDL**:

```sql
CREATE TABLE job_time_windows (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  dispatch_job_id UUID NOT NULL REFERENCES dispatch_jobs(id) ON DELETE CASCADE,
  window_start TIMESTAMPTZ NOT NULL,
  window_end TIMESTAMPTZ NOT NULL,
  source time_window_source_enum NOT NULL DEFAULT 'system_suggested',
  is_selected BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_job_time_windows_end_after_start CHECK (
    window_end > window_start
  )
);

-- Indexes
CREATE INDEX idx_job_time_windows_org_id ON job_time_windows(org_id);
CREATE INDEX idx_job_time_windows_dispatch_job_id ON job_time_windows(dispatch_job_id);
CREATE INDEX idx_job_time_windows_org_id_dispatch_job_id 
  ON job_time_windows(org_id, dispatch_job_id);
CREATE INDEX idx_job_time_windows_org_id_is_selected 
  ON job_time_windows(org_id, is_selected) 
  WHERE is_selected = true;

-- Partial unique index: at most one selected window per job per org
CREATE UNIQUE INDEX idx_job_time_windows_unique_selected 
  ON job_time_windows(org_id, dispatch_job_id) 
  WHERE is_selected = true;
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `dispatch_job_id` | UUID | NO | - | FK to `dispatch_jobs.id` (CASCADE delete) |
| `window_start` | TIMESTAMPTZ | NO | - | Window start time |
| `window_end` | TIMESTAMPTZ | NO | - | Window end time |
| `source` | time_window_source_enum | NO | `'system_suggested'` | Window source |
| `is_selected` | BOOLEAN | NO | `false` | Selected window flag |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |

**Business Rules**:
- `window_end` must be after `window_start` (enforced via CHECK constraint)
- At most one selected window per job per org (enforced via partial unique index)
- Multiple windows can exist per job, but only one can be selected

**Selected Window Convention**:
- When a window is selected (`is_selected = true`), application logic should unselect other windows for the same job
- This can be enforced via trigger or application logic

---

### 4.9 `job_assignments` Table

**Purpose**: Mapping between jobs and technicians with scheduled times and status.

**DDL**:

```sql
CREATE TABLE job_assignments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  dispatch_job_id UUID NOT NULL REFERENCES dispatch_jobs(id) ON DELETE CASCADE,
  technician_id UUID NOT NULL REFERENCES technician_profiles(id) ON DELETE CASCADE,
  assigned_by_user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  scheduled_start_at TIMESTAMPTZ NOT NULL,
  scheduled_end_at TIMESTAMPTZ NOT NULL,
  arrival_window_start TIMESTAMPTZ, -- For customer communications
  arrival_window_end TIMESTAMPTZ,
  status assignment_status_enum NOT NULL DEFAULT 'assigned',
  tech_eta_at TIMESTAMPTZ, -- Dynamic ETA from technician
  sequence_in_route INTEGER, -- Relative order in technician's route
  is_primary_technician BOOLEAN NOT NULL DEFAULT true,
  notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_job_assignments_end_after_start CHECK (
    scheduled_end_at > scheduled_start_at
  ),
  CONSTRAINT chk_job_assignments_arrival_window CHECK (
    (arrival_window_start IS NULL AND arrival_window_end IS NULL) OR
    (arrival_window_start IS NOT NULL AND arrival_window_end IS NOT NULL AND 
     arrival_window_end > arrival_window_start)
  )
);

-- Indexes
CREATE INDEX idx_job_assignments_org_id ON job_assignments(org_id);
CREATE INDEX idx_job_assignments_dispatch_job_id ON job_assignments(dispatch_job_id);
CREATE INDEX idx_job_assignments_technician_id ON job_assignments(technician_id);
CREATE INDEX idx_job_assignments_org_id_technician_id_scheduled_start_at 
  ON job_assignments(org_id, technician_id, scheduled_start_at);
CREATE INDEX idx_job_assignments_org_id_dispatch_job_id 
  ON job_assignments(org_id, dispatch_job_id);
CREATE INDEX idx_job_assignments_org_id_status 
  ON job_assignments(org_id, status);
CREATE INDEX idx_job_assignments_org_id_technician_id_status 
  ON job_assignments(org_id, technician_id, status);

-- Trigger for updated_at
CREATE TRIGGER trigger_job_assignments_updated_at
  BEFORE UPDATE ON job_assignments
  FOR EACH ROW
  EXECUTE FUNCTION update_technician_profiles_updated_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `dispatch_job_id` | UUID | NO | - | FK to `dispatch_jobs.id` (CASCADE delete) |
| `technician_id` | UUID | NO | - | FK to `technician_profiles.id` |
| `assigned_by_user_id` | UUID | YES | NULL | FK to `auth.users.id` (who assigned) |
| `scheduled_start_at` | TIMESTAMPTZ | NO | - | Scheduled start time |
| `scheduled_end_at` | TIMESTAMPTZ | NO | - | Scheduled end time |
| `arrival_window_start` | TIMESTAMPTZ | YES | NULL | Arrival window start |
| `arrival_window_end` | TIMESTAMPTZ | YES | NULL | Arrival window end |
| `status` | assignment_status_enum | NO | `'assigned'` | Assignment status |
| `tech_eta_at` | TIMESTAMPTZ | YES | NULL | Dynamic ETA from technician |
| `sequence_in_route` | INTEGER | YES | NULL | Order in route |
| `is_primary_technician` | BOOLEAN | NO | `true` | Primary technician flag |
| `notes` | TEXT | YES | NULL | Assignment notes |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- `scheduled_end_at` must be after `scheduled_start_at`
- Arrival window must be valid if provided
- Multiple assignments per job allowed (for crew scenarios)

---

### 4.10 `route_plans` Table

**Purpose**: Daily route plans for technicians, typically optimized by AI.

**DDL**:

```sql
CREATE TABLE route_plans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  technician_id UUID NOT NULL REFERENCES technician_profiles(id) ON DELETE CASCADE,
  date DATE NOT NULL,
  status route_plan_status_enum NOT NULL DEFAULT 'draft',
  optimization_strategy optimization_strategy_enum,
  optimization_metadata JSONB, -- Optimizer results (distance, time, cost)
  generated_by route_generated_by_enum NOT NULL DEFAULT 'manual',
  generated_by_user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Unique constraint: one active route plan per tech per date per org
-- Using partial unique index to allow multiple drafts but only one finalized
CREATE UNIQUE INDEX idx_route_plans_unique_active 
  ON route_plans(org_id, technician_id, date) 
  WHERE status IN ('finalized', 'in_progress', 'completed');

-- Indexes
CREATE INDEX idx_route_plans_org_id ON route_plans(org_id);
CREATE INDEX idx_route_plans_technician_id ON route_plans(technician_id);
CREATE INDEX idx_route_plans_org_id_technician_id_date 
  ON route_plans(org_id, technician_id, date);
CREATE INDEX idx_route_plans_org_id_status 
  ON route_plans(org_id, status);
CREATE INDEX idx_route_plans_org_id_date 
  ON route_plans(org_id, date);

-- Trigger for updated_at
CREATE TRIGGER trigger_route_plans_updated_at
  BEFORE UPDATE ON route_plans
  FOR EACH ROW
  EXECUTE FUNCTION update_technician_profiles_updated_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `technician_id` | UUID | NO | - | FK to `technician_profiles.id` |
| `date` | DATE | NO | - | Route date |
| `status` | route_plan_status_enum | NO | `'draft'` | Route status |
| `optimization_strategy` | optimization_strategy_enum | YES | NULL | Optimization strategy |
| `optimization_metadata` | JSONB | YES | NULL | Optimizer results |
| `generated_by` | route_generated_by_enum | NO | `'manual'` | Generation method |
| `generated_by_user_id` | UUID | YES | NULL | FK to `auth.users.id` |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- One active route plan per tech per date per org (enforced via partial unique index)
- Multiple drafts allowed, but only one finalized/in_progress/completed per date

**Optimization Metadata JSONB Structure Example**:
```json
{
  "total_distance_km": 45.2,
  "total_travel_time_minutes": 65,
  "total_job_time_minutes": 240,
  "optimizer_version": "1.0",
  "optimized_at": "2024-01-15T10:30:00Z"
}
```

---

### 4.11 `route_stops` Table

**Purpose**: Individual stops in a route plan.

**DDL**:

```sql
CREATE TABLE route_stops (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  route_plan_id UUID NOT NULL REFERENCES route_plans(id) ON DELETE CASCADE,
  job_assignment_id UUID REFERENCES job_assignments(id) ON DELETE SET NULL,
  stop_type route_stop_type_enum NOT NULL DEFAULT 'job',
  sequence INTEGER NOT NULL,
  latitude NUMERIC(10, 8),
  longitude NUMERIC(11, 8),
  planned_arrival_at TIMESTAMPTZ,
  planned_departure_at TIMESTAMPTZ,
  actual_arrival_at TIMESTAMPTZ,
  actual_departure_at TIMESTAMPTZ,
  travel_time_minutes_from_prev INTEGER,
  distance_km_from_prev NUMERIC(10, 2),
  notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_route_stops_sequence_positive CHECK (
    sequence > 0
  ),
  CONSTRAINT chk_route_stops_coordinates CHECK (
    (latitude IS NULL AND longitude IS NULL) OR
    (latitude IS NOT NULL AND longitude IS NOT NULL AND
     latitude >= -90 AND latitude <= 90 AND
     longitude >= -180 AND longitude <= 180)
  ),
  CONSTRAINT chk_route_stops_departure_after_arrival CHECK (
    (planned_arrival_at IS NULL OR planned_departure_at IS NULL) OR
    (planned_arrival_at IS NOT NULL AND planned_departure_at IS NOT NULL AND
     planned_departure_at > planned_arrival_at)
  )
);

-- Unique constraint: sequence unique per route plan
CREATE UNIQUE INDEX idx_route_stops_unique_sequence 
  ON route_stops(route_plan_id, sequence);

-- Indexes
CREATE INDEX idx_route_stops_org_id ON route_stops(org_id);
CREATE INDEX idx_route_stops_route_plan_id ON route_stops(route_plan_id);
CREATE INDEX idx_route_stops_org_id_route_plan_id_sequence 
  ON route_stops(org_id, route_plan_id, sequence);
CREATE INDEX idx_route_stops_job_assignment_id ON route_stops(job_assignment_id) 
  WHERE job_assignment_id IS NOT NULL;

-- Trigger for updated_at
CREATE TRIGGER trigger_route_stops_updated_at
  BEFORE UPDATE ON route_stops
  FOR EACH ROW
  EXECUTE FUNCTION update_technician_profiles_updated_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `route_plan_id` | UUID | NO | - | FK to `route_plans.id` (CASCADE delete) |
| `job_assignment_id` | UUID | YES | NULL | FK to `job_assignments.id` (for job stops) |
| `stop_type` | route_stop_type_enum | NO | `'job'` | Stop type |
| `sequence` | INTEGER | NO | - | Stop order (unique per route) |
| `latitude` | NUMERIC(10,8) | YES | NULL | Stop latitude |
| `longitude` | NUMERIC(11,8) | YES | NULL | Stop longitude |
| `planned_arrival_at` | TIMESTAMPTZ | YES | NULL | Planned arrival time |
| `planned_departure_at` | TIMESTAMPTZ | YES | NULL | Planned departure time |
| `actual_arrival_at` | TIMESTAMPTZ | YES | NULL | Actual arrival time |
| `actual_departure_at` | TIMESTAMPTZ | YES | NULL | Actual departure time |
| `travel_time_minutes_from_prev` | INTEGER | YES | NULL | Travel time from previous stop |
| `distance_km_from_prev` | NUMERIC(10,2) | YES | NULL | Distance from previous stop |
| `notes` | TEXT | YES | NULL | Stop notes |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- `sequence` must be positive and unique per route plan
- Coordinates must be valid if provided
- `planned_departure_at` must be after `planned_arrival_at` if both provided

---

### 4.12 `calendar_integrations` Table

**Purpose**: OAuth/connection config for external calendars.

**DDL**:

```sql
CREATE TABLE calendar_integrations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  provider calendar_provider_enum NOT NULL,
  provider_account_id TEXT NOT NULL,
  access_token TEXT NOT NULL, -- Encrypted, never plaintext
  refresh_token TEXT, -- Encrypted, nullable
  expires_at TIMESTAMPTZ,
  calendar_id TEXT NOT NULL, -- Target calendar ID
  scope TEXT,
  is_active BOOLEAN NOT NULL DEFAULT true,
  metadata JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_calendar_integrations_expires_at CHECK (
    expires_at IS NULL OR expires_at > created_at
  )
);

-- Unique constraint: one integration per user per provider per org
CREATE UNIQUE INDEX idx_calendar_integrations_unique 
  ON calendar_integrations(org_id, user_id, provider);

-- Indexes
CREATE INDEX idx_calendar_integrations_org_id ON calendar_integrations(org_id);
CREATE INDEX idx_calendar_integrations_user_id ON calendar_integrations(user_id);
CREATE INDEX idx_calendar_integrations_org_id_user_id 
  ON calendar_integrations(org_id, user_id);
CREATE INDEX idx_calendar_integrations_org_id_is_active 
  ON calendar_integrations(org_id, is_active) 
  WHERE is_active = true;

-- Trigger for updated_at
CREATE TRIGGER trigger_calendar_integrations_updated_at
  BEFORE UPDATE ON calendar_integrations
  FOR EACH ROW
  EXECUTE FUNCTION update_technician_profiles_updated_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `user_id` | UUID | NO | - | FK to `auth.users.id` |
| `provider` | calendar_provider_enum | NO | - | Calendar provider |
| `provider_account_id` | TEXT | NO | - | Provider account identifier |
| `access_token` | TEXT | NO | - | **Encrypted** access token |
| `refresh_token` | TEXT | YES | NULL | **Encrypted** refresh token |
| `expires_at` | TIMESTAMPTZ | YES | NULL | Token expiration |
| `calendar_id` | TEXT | NO | - | Target calendar ID |
| `scope` | TEXT | YES | NULL | OAuth scope |
| `is_active` | BOOLEAN | NO | `true` | Active status |
| `metadata` | JSONB | YES | NULL | Additional metadata |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Security Requirements**:
- **CRITICAL**: `access_token` and `refresh_token` must be encrypted at rest
- Use Supabase Vault or application-level encryption
- Never expose tokens via RLS policies or API responses
- Document encryption strategy in security documentation

**Token Encryption Strategy**:
- Use Supabase Vault (recommended) or application-level encryption
- Encrypt before insert, decrypt after select (application layer)
- Document key management and rotation procedures

---

### 4.13 `calendar_events` Table

**Purpose**: Mapping between internal assignments and external calendar events.

**DDL**:

```sql
CREATE TABLE calendar_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  calendar_integration_id UUID NOT NULL REFERENCES calendar_integrations(id) ON DELETE CASCADE,
  job_assignment_id UUID REFERENCES job_assignments(id) ON DELETE SET NULL,
  dispatch_job_id UUID REFERENCES dispatch_jobs(id) ON DELETE SET NULL,
  provider_event_id TEXT NOT NULL, -- External calendar event ID
  status calendar_event_status_enum NOT NULL DEFAULT 'scheduled',
  last_synced_at TIMESTAMPTZ,
  sync_direction calendar_sync_direction_enum NOT NULL DEFAULT 'internal_to_external',
  metadata JSONB, -- Original payload snapshot
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_calendar_events_job_reference CHECK (
    job_assignment_id IS NOT NULL OR dispatch_job_id IS NOT NULL
  )
);

-- Unique constraint: one event per provider event ID per integration
CREATE UNIQUE INDEX idx_calendar_events_unique_provider_event 
  ON calendar_events(org_id, calendar_integration_id, provider_event_id);

-- Indexes
CREATE INDEX idx_calendar_events_org_id ON calendar_events(org_id);
CREATE INDEX idx_calendar_events_calendar_integration_id ON calendar_events(calendar_integration_id);
CREATE INDEX idx_calendar_events_org_id_provider_event_id 
  ON calendar_events(org_id, provider_event_id);
CREATE INDEX idx_calendar_events_job_assignment_id ON calendar_events(job_assignment_id) 
  WHERE job_assignment_id IS NOT NULL;
CREATE INDEX idx_calendar_events_dispatch_job_id ON calendar_events(dispatch_job_id) 
  WHERE dispatch_job_id IS NOT NULL;
CREATE INDEX idx_calendar_events_org_id_status 
  ON calendar_events(org_id, status);

-- Trigger for updated_at
CREATE TRIGGER trigger_calendar_events_updated_at
  BEFORE UPDATE ON calendar_events
  FOR EACH ROW
  EXECUTE FUNCTION update_technician_profiles_updated_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `calendar_integration_id` | UUID | NO | - | FK to `calendar_integrations.id` |
| `job_assignment_id` | UUID | YES | NULL | FK to `job_assignments.id` |
| `dispatch_job_id` | UUID | YES | NULL | FK to `dispatch_jobs.id` |
| `provider_event_id` | TEXT | NO | - | External calendar event ID |
| `status` | calendar_event_status_enum | NO | `'scheduled'` | Sync status |
| `last_synced_at` | TIMESTAMPTZ | YES | NULL | Last sync timestamp |
| `sync_direction` | calendar_sync_direction_enum | NO | `'internal_to_external'` | Sync direction |
| `metadata` | JSONB | YES | NULL | Original payload snapshot |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- At least one job reference required (`job_assignment_id` OR `dispatch_job_id`)
- One event per provider event ID per integration (enforced via unique index)

---

### 4.14 `job_notifications` Table

**Purpose**: Scheduling-related reminders and notifications.

**DDL**:

```sql
CREATE TABLE job_notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  dispatch_job_id UUID REFERENCES dispatch_jobs(id) ON DELETE CASCADE,
  job_assignment_id UUID REFERENCES job_assignments(id) ON DELETE CASCADE,
  recipient_type notification_recipient_type_enum NOT NULL,
  recipient_contact_id UUID, -- FK to customer_contacts.id or resolved via service
  channel notification_channel_enum NOT NULL,
  notification_type notification_type_enum NOT NULL,
  scheduled_send_at TIMESTAMPTZ NOT NULL,
  sent_at TIMESTAMPTZ,
  status notification_status_enum NOT NULL DEFAULT 'pending',
  error_message TEXT,
  metadata JSONB, -- Template variables, provider IDs
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_job_notifications_job_reference CHECK (
    dispatch_job_id IS NOT NULL OR job_assignment_id IS NOT NULL
  ),
  CONSTRAINT chk_job_notifications_sent_at CHECK (
    (status = 'sent' AND sent_at IS NOT NULL) OR
    (status != 'sent' AND sent_at IS NULL)
  )
);

-- Indexes
CREATE INDEX idx_job_notifications_org_id ON job_notifications(org_id);
CREATE INDEX idx_job_notifications_dispatch_job_id ON job_notifications(dispatch_job_id) 
  WHERE dispatch_job_id IS NOT NULL;
CREATE INDEX idx_job_notifications_job_assignment_id ON job_notifications(job_assignment_id) 
  WHERE job_assignment_id IS NOT NULL;
CREATE INDEX idx_job_notifications_org_id_scheduled_send_at_status 
  ON job_notifications(org_id, scheduled_send_at, status);
CREATE INDEX idx_job_notifications_org_id_status 
  ON job_notifications(org_id, status) 
  WHERE status = 'pending';
CREATE INDEX idx_job_notifications_recipient_contact_id ON job_notifications(recipient_contact_id) 
  WHERE recipient_contact_id IS NOT NULL;

-- Trigger for updated_at
CREATE TRIGGER trigger_job_notifications_updated_at
  BEFORE UPDATE ON job_notifications
  FOR EACH ROW
  EXECUTE FUNCTION update_technician_profiles_updated_at();

-- Trigger to auto-set sent_at when status changes to 'sent'
CREATE OR REPLACE FUNCTION set_job_notifications_sent_at()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.status = 'sent' AND OLD.status != 'sent' THEN
    NEW.sent_at = now();
  ELSIF NEW.status != 'sent' THEN
    NEW.sent_at = NULL;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_job_notifications_sent_at
  BEFORE UPDATE ON job_notifications
  FOR EACH ROW
  EXECUTE FUNCTION set_job_notifications_sent_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `dispatch_job_id` | UUID | YES | NULL | FK to `dispatch_jobs.id` |
| `job_assignment_id` | UUID | YES | NULL | FK to `job_assignments.id` |
| `recipient_type` | notification_recipient_type_enum | NO | - | Recipient type |
| `recipient_contact_id` | UUID | YES | NULL | Contact ID (resolved by service) |
| `channel` | notification_channel_enum | NO | - | Notification channel |
| `notification_type` | notification_type_enum | NO | - | Notification type |
| `scheduled_send_at` | TIMESTAMPTZ | NO | - | Scheduled send time |
| `sent_at` | TIMESTAMPTZ | YES | NULL | Actual send time |
| `status` | notification_status_enum | NO | `'pending'` | Notification status |
| `error_message` | TEXT | YES | NULL | Error message if failed |
| `metadata` | JSONB | YES | NULL | Template variables, provider IDs |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- At least one job reference required (`dispatch_job_id` OR `job_assignment_id`)
- `sent_at` automatically set when status changes to `'sent'` (via trigger)

**Recipient Contact ID Resolution Convention**:
- **Customer**: `recipient_contact_id` references `customer_contacts.id`
- **Technician**: `recipient_contact_id` references technician's contact info (resolved via `technician_profiles.user_id` -> `profiles` -> contact lookup)
- **Dispatcher**: `recipient_contact_id` references dispatcher's contact info (resolved via `profiles.user_id` -> contact lookup)
- Resolution handled in application/Edge Function layer, not enforced at DB level

---

## 5. Deferred Foreign Keys

### 5.1 `technician_profiles.default_service_zone_id`

**Action**: Add after `service_zones` table is created.

**SQL**:
```sql
ALTER TABLE technician_profiles 
  ADD CONSTRAINT fk_technician_profiles_default_service_zone 
  FOREIGN KEY (default_service_zone_id) 
  REFERENCES service_zones(id) 
  ON DELETE SET NULL;
```

### 5.2 `dispatch_jobs.related_work_order_id`

**Action**: Add when Work Order module is implemented.

**Note**: Placeholder column exists; FK will be added later.

---

## 6. Migration Strategy

### 6.1 Migration Order

Migrations must be executed in this order:

1. **Create all enums** (all 20 enum types)
2. **Create `technician_profiles` table**
3. **Create `technician_skills` table**
4. **Create `service_zones` table**
5. **Add deferred FK**: `technician_profiles.default_service_zone_id`
6. **Create `technician_service_zones` table**
7. **Create `technician_shifts` table**
8. **Create `technician_time_off` table**
9. **Create `dispatch_jobs` table**
10. **Create `job_time_windows` table**
11. **Create `job_assignments` table**
12. **Create `route_plans` table**
13. **Create `route_stops` table**
14. **Create `calendar_integrations` table**
15. **Create `calendar_events` table**
16. **Create `job_notifications` table**

### 6.2 Migration File Structure

```
supabase/migrations/
  20240101000000_create_dispatch_enums.sql
  20240101000001_create_technician_profiles_table.sql
  20240101000002_create_technician_skills_table.sql
  20240101000003_create_service_zones_table.sql
  20240101000004_add_technician_profiles_deferred_fk.sql
  20240101000005_create_technician_service_zones_table.sql
  20240101000006_create_technician_shifts_table.sql
  20240101000007_create_technician_time_off_table.sql
  20240101000008_create_dispatch_jobs_table.sql
  20240101000009_create_job_time_windows_table.sql
  20240101000010_create_job_assignments_table.sql
  20240101000011_create_route_plans_table.sql
  20240101000012_create_route_stops_table.sql
  20240101000013_create_calendar_integrations_table.sql
  20240101000014_create_calendar_events_table.sql
  20240101000015_create_job_notifications_table.sql
```

### 6.3 Rollback Strategy

Each migration should be idempotent (use `CREATE TABLE IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS`, etc.). Create corresponding `down` migrations for rollback.

---

## 7. Seed Data Requirements

### 7.1 Test Technicians

Create at least 2 technicians in Org A and 1 in Org B:

```sql
-- Org A technicians (assuming user IDs from auth)
INSERT INTO technician_profiles (
  org_id, user_id, display_name, employment_type, is_active, 
  max_daily_work_minutes, max_concurrent_jobs, vehicle_type
) VALUES
  (
    '00000000-0000-0000-0000-000000000001',
    '<tech1-user-id>',
    'John Smith',
    'employee',
    true,
    480,
    1,
    'van'
  ),
  (
    '00000000-0000-0000-0000-000000000001',
    '<tech2-user-id>',
    'Jane Doe',
    'employee',
    true,
    480,
    1,
    'truck'
  );

-- Org B technician
INSERT INTO technician_profiles (
  org_id, user_id, display_name, employment_type, is_active
) VALUES
  (
    '00000000-0000-0000-0000-000000000002',
    '<tech3-user-id>',
    'Bob Johnson',
    'contractor',
    true
  );
```

### 7.2 Test Skills

```sql
-- Skills for Org A technicians
INSERT INTO technician_skills (org_id, technician_id, skill_code, proficiency_level, is_primary)
SELECT 
  '00000000-0000-0000-0000-000000000001',
  id,
  'hvac_install',
  'senior',
  true
FROM technician_profiles 
WHERE display_name = 'John Smith' AND org_id = '00000000-0000-0000-0000-000000000001';

INSERT INTO technician_skills (org_id, technician_id, skill_code, proficiency_level, is_primary)
SELECT 
  '00000000-0000-0000-0000-000000000001',
  id,
  'hvac_service',
  'expert',
  false
FROM technician_profiles 
WHERE display_name = 'John Smith' AND org_id = '00000000-0000-0000-0000-000000000001';
```

### 7.3 Test Service Zones

```sql
-- Service zone for Org A
INSERT INTO service_zones (org_id, name, description, postal_codes, is_active)
VALUES (
  '00000000-0000-0000-0000-000000000001',
  'Downtown Zone',
  'Downtown service area',
  ARRAY['90210', '90211', '90212'],
  true
);
```

### 7.4 Test Shifts

```sql
-- Shift for Org A technician
INSERT INTO technician_shifts (
  org_id, technician_id, starts_at, ends_at, shift_type, recurrence_rule, is_active
)
SELECT 
  '00000000-0000-0000-0000-000000000001',
  id,
  '2024-01-15 08:00:00+00',
  '2024-01-15 17:00:00+00',
  'regular',
  'FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR',
  true
FROM technician_profiles 
WHERE display_name = 'John Smith' AND org_id = '00000000-0000-0000-0000-000000000001';
```

### 7.5 Test Dispatch Jobs

```sql
-- Unscheduled job
INSERT INTO dispatch_jobs (
  org_id, customer_id, location_id, title, description, priority, status,
  estimated_duration_minutes, required_crew_size
)
SELECT 
  '00000000-0000-0000-0000-000000000001',
  (SELECT id FROM customers LIMIT 1),
  (SELECT id FROM customer_locations LIMIT 1),
  'AC Maintenance',
  'Routine AC maintenance check',
  'normal',
  'unscheduled',
  60,
  1;

-- Scheduled job
INSERT INTO dispatch_jobs (
  org_id, customer_id, location_id, title, priority, status,
  estimated_duration_minutes, sla_start_at, sla_end_at
)
SELECT 
  '00000000-0000-0000-0000-000000000001',
  (SELECT id FROM customers LIMIT 1),
  (SELECT id FROM customer_locations LIMIT 1),
  'Emergency AC Repair',
  'emergency',
  'scheduled',
  120,
  '2024-01-15 10:00:00+00',
  '2024-01-15 14:00:00+00';
```

---

## 8. Performance Considerations

### 8.1 Index Strategy

- **Primary Keys**: All tables use UUID primary keys
- **Foreign Keys**: Indexed for join performance
- **Composite Indexes**: Created for common query patterns (`org_id` + filter)
- **Partial Indexes**: Used for filtered queries (e.g., `WHERE status = 'pending'`)
- **GIN Indexes**: Used for array and JSONB columns

### 8.2 Query Performance Targets

- **Schedule Board Queries**: < 500ms for typical orgs (dozens of technicians, hundreds of jobs)
- **Technician Availability**: < 300ms for date range queries
- **Route Plan Queries**: < 400ms for daily route retrieval

---

## 9. Integration Points

### 9.1 CRM Module

- `dispatch_jobs.customer_id` -> `customers.id`
- `dispatch_jobs.location_id` -> `customer_locations.id`
- `job_notifications.recipient_contact_id` -> `customer_contacts.id` (for customers)

### 9.2 Work Order Module (Future)

- `dispatch_jobs.related_work_order_id` -> work order table (deferred FK)

### 9.3 Auth Module

- All `user_id` and `created_by_user_id` columns reference `auth.users.id`
- `technician_profiles.user_id` links to `auth.users.id`

---

## 10. Implementation Checklist

### Story DISP-003: `technician_profiles` Table
- [ ] Table created with all specified columns
- [ ] Unique constraint on (`org_id`, `user_id`)
- [ ] Indexes created
- [ ] Test data created (2 techs in Org A, 1 in Org B)

### Story DISP-004: `technician_skills` Table
- [ ] Table created
- [ ] Unique constraint on (`org_id`, `technician_id`, `skill_code`)
- [ ] Seed data with 2+ skills per technician

### Story DISP-005: `service_zones` Table
- [ ] Table created
- [ ] Geometry storage approach documented (JSONB)
- [ ] Seed data with example zone

### Story DISP-006: `technician_service_zones` Table
- [ ] Table created
- [ ] Unique constraint on (`org_id`, `technician_id`, `service_zone_id`)
- [ ] Primary zone constraint enforced
- [ ] Seed data with multiple zones per tech

### Story DISP-007: `technician_shifts` Table
- [ ] Table created
- [ ] CHECK constraint prevents invalid intervals
- [ ] Index on (`org_id`, `technician_id`, `starts_at`)
- [ ] Seed data with recurring shift

### Story DISP-008: `technician_time_off` Table
- [ ] Table created
- [ ] CHECK constraint prevents invalid intervals
- [ ] Index on (`org_id`, `technician_id`, `starts_at`)
- [ ] Seed data with overlapping time-off

### Story DISP-009: `dispatch_jobs` Table
- [ ] Table created with all fields
- [ ] FKs to `customers` and `customer_locations`
- [ ] Indexes on status/priority and SLA
- [ ] Seed data with unscheduled and scheduled jobs

### Story DISP-010: `job_time_windows` Table
- [ ] Table created
- [ ] Cascade delete with `dispatch_job`
- [ ] Selected window convention documented
- [ ] Seed data with multiple windows, one selected

### Story DISP-011: `job_assignments` Table
- [ ] Table created
- [ ] Assignment status enum supports all states
- [ ] Indexes created
- [ ] Seed data with multi-assignment job (crew)

### Story DISP-012: `route_plans` Table
- [ ] Table created
- [ ] Uniqueness strategy documented (one active per tech/date)
- [ ] `optimization_metadata` JSONB structure documented
- [ ] Seed data with draft and finalized routes

### Story DISP-013: `route_stops` Table
- [ ] Table created
- [ ] Sequence uniqueness enforced per route
- [ ] Index on (`route_plan_id`, `sequence`)
- [ ] Seed data with depot start/end and job stops

### Story DISP-014: `calendar_integrations` Table
- [ ] Table created
- [ ] Token encryption strategy documented
- [ ] Provider enum includes google and microsoft
- [ ] Security approach reviewed

### Story DISP-015: `calendar_events` Table
- [ ] Table created
- [ ] `sync_direction` enum supports all directions
- [ ] Index for provider event ID lookup
- [ ] Seed data with mapping row

### Story DISP-016: `job_notifications` Table
- [ ] Table created with all enums
- [ ] Index for pending lookup by send time/status
- [ ] `recipient_contact_id` resolution convention documented
- [ ] Seed data with pending and sent notifications

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 2 – Core Scheduling Data Model. All specifications are designed to be directly consumable by LLM-based code generators, with exact data types, constraints, indexes, and relationships defined.

**Next Steps**: After completing Epic 2, proceed to Epic 3 (RLS Policies) which will secure all tables created here using the role conventions from Epic 1.

