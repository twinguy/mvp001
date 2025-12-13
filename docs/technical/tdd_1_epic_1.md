# Technical Design Document – Epic 1: CRM Core Data Model & Supabase Schema

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 1 – CRM Core Data Model & Supabase Schema
- **Source**: Derived from `fdd_1_agile.md` Epic 1 (Stories CRM-001 through CRM-012)
- **Reference Documents**: 
  - `fdd_1.md` (Functional Design Document for CRM)
  - `fdd_1_agile.md` (Agile User Stories)
  - `functional.md` (Platform Capabilities)
  - `tooling.md` (Platform Choices: Supabase + Vercel)
- **Target Platform**: Supabase (PostgreSQL 15+)
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing the CRM core data model in Supabase PostgreSQL. It covers all database schema elements required for Epic 1, including:

- Multi-tenancy foundation (`orgs` table and `org_id` conventions)
- Core CRM entities (customers, locations, contacts, preferences)
- Interaction and follow-up tracking
- Tagging and segmentation infrastructure
- Message templates and automation rules
- Indexes, constraints, triggers, and performance optimizations

All specifications are designed to be directly implementable via SQL migrations in Supabase, with exact data types, constraints, and relationships defined.

---

## 2. Multi-Tenancy Foundation

### 2.1 Organizational Model

**Decision**: The CRM module assumes an `orgs` table exists or will be created to represent tenant organizations. If a global `orgs` or `accounts` table exists elsewhere in the platform, it should be reused. Otherwise, create a dedicated `orgs` table for CRM tenancy.

### 2.2 `org_id` Convention

- **Column Name**: `org_id` (consistent across all CRM tables)
- **Data Type**: `UUID` (references `orgs.id`)
- **Nullability**: `NOT NULL` on all CRM tables requiring tenancy
- **RLS Integration**: All RLS policies will filter by `org_id` matching the authenticated user's organization
- **Derivation**: `org_id` is derived from the authenticated user's profile via:
  - `profiles.org_id` (if profiles table exists)
  - Or JWT claims (`app.current_org_id`) set by middleware/Edge Functions
  - Documented in CRM README with exact implementation pattern

### 2.3 Cross-Org Data Isolation

- All queries must include `org_id` filtering
- Foreign key relationships enforce `org_id` consistency where applicable
- Example query pattern: `WHERE org_id = current_setting('app.current_org_id')::uuid`

---

## 3. Database Schema Specifications

### 3.1 Enums

All enums must be created before tables that reference them. Order matters for dependencies.

#### 3.1.1 `customer_type_enum`

```sql
CREATE TYPE customer_type_enum AS ENUM (
  'individual',
  'company'
);
```

**Usage**: `customers.type`

#### 3.1.2 `customer_status_enum`

```sql
CREATE TYPE customer_status_enum AS ENUM (
  'active',
  'prospect',
  'inactive',
  'blacklisted'
);
```

**Usage**: `customers.status`

#### 3.1.3 `customer_lifecycle_stage_enum`

```sql
CREATE TYPE customer_lifecycle_stage_enum AS ENUM (
  'lead',
  'opportunity',
  'customer',
  'former_customer'
);
```

**Usage**: `customers.lifecycle_stage`

#### 3.1.4 `location_type_enum`

```sql
CREATE TYPE location_type_enum AS ENUM (
  'billing',
  'service',
  'both'
);
```

**Usage**: `customer_locations.type`

#### 3.1.5 `contact_type_enum`

```sql
CREATE TYPE contact_type_enum AS ENUM (
  'email',
  'mobile',
  'phone',
  'fax',
  'whatsapp',
  'telegram',
  'portal'
);
```

**Usage**: `customer_contacts.type`

#### 3.1.6 `interaction_channel_enum`

```sql
CREATE TYPE interaction_channel_enum AS ENUM (
  'phone_inbound',
  'phone_outbound',
  'email_inbound',
  'email_outbound',
  'sms_inbound',
  'sms_outbound',
  'portal_message',
  'note',
  'in_person'
);
```

**Usage**: `crm_interactions.channel`

#### 3.1.7 `interaction_direction_enum`

```sql
CREATE TYPE interaction_direction_enum AS ENUM (
  'inbound',
  'outbound',
  'system_generated'
);
```

**Usage**: `crm_interactions.direction`

#### 3.1.8 `interaction_sentiment_enum`

```sql
CREATE TYPE interaction_sentiment_enum AS ENUM (
  'positive',
  'neutral',
  'negative'
);
```

**Usage**: `crm_interactions.sentiment`

#### 3.1.9 `followup_status_enum`

```sql
CREATE TYPE followup_status_enum AS ENUM (
  'pending',
  'completed',
  'canceled',
  'expired'
);
```

**Usage**: `crm_followups.status`

#### 3.1.10 `followup_priority_enum`

```sql
CREATE TYPE followup_priority_enum AS ENUM (
  'low',
  'medium',
  'high'
);
```

**Usage**: `crm_followups.priority`

#### 3.1.11 `followup_origin_enum`

```sql
CREATE TYPE followup_origin_enum AS ENUM (
  'manual',
  'system_rule',
  'ai_recommendation'
);
```

**Usage**: `crm_followups.origin`

#### 3.1.12 `segment_type_enum`

```sql
CREATE TYPE segment_type_enum AS ENUM (
  'static',
  'rule_based',
  'ai_generated'
);
```

**Usage**: `crm_segments.type`

#### 3.1.13 `message_template_channel_enum`

```sql
CREATE TYPE message_template_channel_enum AS ENUM (
  'email',
  'sms',
  'phone_script',
  'portal_message'
);
```

**Usage**: `crm_message_templates.channel`

#### 3.1.14 `automation_trigger_type_enum`

```sql
CREATE TYPE automation_trigger_type_enum AS ENUM (
  'event',
  'time_based',
  'segment_membership'
);
```

**Usage**: `crm_automation_rules.trigger_type`

#### 3.1.15 `automation_run_status_enum`

```sql
CREATE TYPE automation_run_status_enum AS ENUM (
  'pending',
  'success',
  'failed',
  'skipped'
);
```

**Usage**: `crm_automation_runs.status`

---

### 3.2 Core Tables

#### 3.2.1 `orgs` Table

**Purpose**: Represents tenant organizations. If this table exists elsewhere, reuse it; otherwise create it here.

**DDL**:

```sql
CREATE TABLE IF NOT EXISTS orgs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Index for lookups (though PK is sufficient, this may be useful for name searches)
CREATE INDEX IF NOT EXISTS idx_orgs_name ON orgs(name);
```

**Notes**:
- This table may be extended in other modules (e.g., billing, settings)
- If a global `accounts` or `organizations` table exists, use that instead and document the decision

---

#### 3.2.2 `customers` Table

**Purpose**: Core customer entity representing individuals or companies.

**DDL**:

```sql
CREATE TABLE customers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  external_ref TEXT,
  type customer_type_enum NOT NULL,
  name TEXT NOT NULL,
  first_name TEXT,
  last_name TEXT,
  company_name TEXT,
  primary_location_id UUID, -- FK added after customer_locations table exists
  primary_contact_id UUID,  -- FK added after customer_contacts table exists
  email TEXT,
  phone TEXT,
  status customer_status_enum NOT NULL DEFAULT 'prospect',
  lifecycle_stage customer_lifecycle_stage_enum NOT NULL DEFAULT 'lead',
  source TEXT,
  preferred_language TEXT,
  notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Constraints
  CONSTRAINT chk_customers_individual_has_names CHECK (
    (type = 'individual' AND first_name IS NOT NULL AND last_name IS NOT NULL) OR
    (type = 'company' AND company_name IS NOT NULL) OR
    (type = 'individual' AND first_name IS NULL AND last_name IS NULL AND company_name IS NULL)
  ),
  CONSTRAINT chk_customers_email_format CHECK (
    email IS NULL OR email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
  )
);

-- Foreign keys (deferred until referenced tables exist)
-- ALTER TABLE customers ADD CONSTRAINT fk_customers_primary_location 
--   FOREIGN KEY (primary_location_id) REFERENCES customer_locations(id) ON DELETE SET NULL;
-- ALTER TABLE customers ADD CONSTRAINT fk_customers_primary_contact 
--   FOREIGN KEY (primary_contact_id) REFERENCES customer_contacts(id) ON DELETE SET NULL;

-- Indexes
CREATE INDEX idx_customers_org_id ON customers(org_id);
CREATE INDEX idx_customers_org_id_status ON customers(org_id, status);
CREATE INDEX idx_customers_org_id_lifecycle_stage ON customers(org_id, lifecycle_stage);
CREATE INDEX idx_customers_org_id_name ON customers(org_id, name);
CREATE INDEX idx_customers_org_id_email ON customers(org_id, email) WHERE email IS NOT NULL;
CREATE INDEX idx_customers_org_id_phone ON customers(org_id, phone) WHERE phone IS NOT NULL;
CREATE INDEX idx_customers_external_ref ON customers(org_id, external_ref) WHERE external_ref IS NOT NULL;

-- Full-text search index (for name/email/phone search)
CREATE INDEX idx_customers_search ON customers USING gin(
  to_tsvector('english', coalesce(name, '') || ' ' || coalesce(email, '') || ' ' || coalesce(phone, ''))
);

-- Trigger for updated_at
CREATE OR REPLACE FUNCTION update_customers_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_customers_updated_at
  BEFORE UPDATE ON customers
  FOR EACH ROW
  EXECUTE FUNCTION update_customers_updated_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | Foreign key to `orgs.id` |
| `external_ref` | TEXT | YES | NULL | Legacy system reference |
| `type` | customer_type_enum | NO | - | `individual` or `company` |
| `name` | TEXT | NO | - | Full name or company name |
| `first_name` | TEXT | YES | NULL | For individuals |
| `last_name` | TEXT | YES | NULL | For individuals |
| `company_name` | TEXT | YES | NULL | For companies |
| `primary_location_id` | UUID | YES | NULL | FK to `customer_locations.id` (deferred) |
| `primary_contact_id` | UUID | YES | NULL | FK to `customer_contacts.id` (deferred) |
| `email` | TEXT | YES | NULL | Convenience field (may mirror primary contact) |
| `phone` | TEXT | YES | NULL | Convenience field (may mirror primary contact) |
| `status` | customer_status_enum | NO | `'prospect'` | Customer status |
| `lifecycle_stage` | customer_lifecycle_stage_enum | NO | `'lead'` | Sales lifecycle stage |
| `source` | TEXT | YES | NULL | Lead source (e.g., `web`, `phone`, `referral`) |
| `preferred_language` | TEXT | YES | NULL | ISO language code (e.g., `en`, `es`) |
| `notes` | TEXT | YES | NULL | Internal notes |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- If `type = 'individual'`, `first_name` and `last_name` should be provided (enforced via CHECK constraint)
- If `type = 'company'`, `company_name` should be provided (enforced via CHECK constraint)
- `email` must match basic email format if provided (enforced via CHECK constraint)
- `primary_location_id` and `primary_contact_id` are set after locations/contacts are created (deferred FKs)

**Performance Notes**:
- Composite indexes on `(org_id, status)` and `(org_id, lifecycle_stage)` support common filtering
- Partial indexes on `email` and `phone` optimize search queries
- Full-text search index enables fast name/email/phone searches

---

#### 3.2.3 `customer_locations` Table

**Purpose**: Stores billing and service locations for customers.

**DDL**:

```sql
CREATE TABLE customer_locations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  label TEXT,
  type location_type_enum NOT NULL,
  address_line1 TEXT NOT NULL,
  address_line2 TEXT,
  city TEXT NOT NULL,
  state TEXT,
  postal_code TEXT,
  country TEXT DEFAULT 'US',
  latitude NUMERIC(10, 8),
  longitude NUMERIC(11, 8),
  is_primary BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Constraints
  CONSTRAINT chk_customer_locations_coordinates CHECK (
    (latitude IS NULL AND longitude IS NULL) OR
    (latitude IS NOT NULL AND longitude IS NOT NULL AND
     latitude >= -90 AND latitude <= 90 AND
     longitude >= -180 AND longitude <= 180)
  )
);

-- Indexes
CREATE INDEX idx_customer_locations_org_id ON customer_locations(org_id);
CREATE INDEX idx_customer_locations_customer_id ON customer_locations(customer_id);
CREATE INDEX idx_customer_locations_org_id_customer_id ON customer_locations(org_id, customer_id);
CREATE INDEX idx_customer_locations_org_id_type ON customer_locations(org_id, type);
CREATE INDEX idx_customer_locations_geo ON customer_locations USING gist(
  ll_to_earth(latitude, longitude)
) WHERE latitude IS NOT NULL AND longitude IS NOT NULL;

-- Partial unique index: at most one primary location per customer per type per org
CREATE UNIQUE INDEX idx_customer_locations_unique_primary 
  ON customer_locations(org_id, customer_id, type) 
  WHERE is_primary = true;

-- Trigger for updated_at
CREATE TRIGGER trigger_customer_locations_updated_at
  BEFORE UPDATE ON customer_locations
  FOR EACH ROW
  EXECUTE FUNCTION update_customers_updated_at(); -- Reuse same function
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `customer_id` | UUID | NO | - | FK to `customers.id` (CASCADE delete) |
| `label` | TEXT | YES | NULL | Human-readable label (e.g., "Home", "Office") |
| `type` | location_type_enum | NO | - | `billing`, `service`, or `both` |
| `address_line1` | TEXT | NO | - | Street address line 1 |
| `address_line2` | TEXT | YES | NULL | Street address line 2 |
| `city` | TEXT | NO | - | City |
| `state` | TEXT | YES | NULL | State/province |
| `postal_code` | TEXT | YES | NULL | ZIP/postal code |
| `country` | TEXT | YES | `'US'` | Country code |
| `latitude` | NUMERIC(10,8) | YES | NULL | Latitude (decimal degrees) |
| `longitude` | NUMERIC(11,8) | YES | NULL | Longitude (decimal degrees) |
| `is_primary` | BOOLEAN | NO | `false` | Primary location flag for this type |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- Coordinates must be valid (latitude -90 to 90, longitude -180 to 180) if provided
- At most one primary location per customer per `type` per org (enforced via unique partial index)
- Deleting a customer cascades to all locations

**Performance Notes**:
- GIST index on coordinates supports geo-queries (requires PostGIS extension or custom function)
- If PostGIS is not available, use standard B-tree indexes on `(latitude, longitude)` for range queries

**PostGIS Extension (Optional)**:

If PostGIS is enabled in Supabase:

```sql
-- Enable PostGIS extension (run as superuser or via Supabase dashboard)
-- CREATE EXTENSION IF NOT EXISTS postgis;

-- Use PostGIS geography type for better geo queries
-- ALTER TABLE customer_locations ADD COLUMN location_point GEOGRAPHY(POINT, 4326);
-- CREATE INDEX idx_customer_locations_geo_postgis ON customer_locations USING gist(location_point);
```

---

#### 3.2.4 `customer_contacts` Table

**Purpose**: Stores multiple communication channels per customer.

**DDL**:

```sql
CREATE TABLE customer_contacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  type contact_type_enum NOT NULL,
  value TEXT NOT NULL,
  is_primary BOOLEAN NOT NULL DEFAULT false,
  is_verified BOOLEAN NOT NULL DEFAULT false,
  opt_in_marketing BOOLEAN NOT NULL DEFAULT true,
  opt_in_transactional BOOLEAN NOT NULL DEFAULT true,
  preferred_channel BOOLEAN NOT NULL DEFAULT false,
  notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Constraints
  CONSTRAINT chk_customer_contacts_email_format CHECK (
    (type = 'email' AND value ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$') OR
    type != 'email'
  ),
  CONSTRAINT chk_customer_contacts_phone_format CHECK (
    (type IN ('mobile', 'phone', 'fax') AND value ~ '^\+?[1-9]\d{1,14}$') OR
    type NOT IN ('mobile', 'phone', 'fax')
  )
);

-- Indexes
CREATE INDEX idx_customer_contacts_org_id ON customer_contacts(org_id);
CREATE INDEX idx_customer_contacts_customer_id ON customer_contacts(customer_id);
CREATE INDEX idx_customer_contacts_org_id_customer_id ON customer_contacts(org_id, customer_id);
CREATE INDEX idx_customer_contacts_org_id_type ON customer_contacts(org_id, type);
CREATE INDEX idx_customer_contacts_org_id_value ON customer_contacts(org_id, value);

-- Partial unique index: one primary contact per type per customer per org
CREATE UNIQUE INDEX idx_customer_contacts_unique_primary 
  ON customer_contacts(org_id, customer_id, type) 
  WHERE is_primary = true;

-- Optional: unique email per org (if business rule requires)
-- CREATE UNIQUE INDEX idx_customer_contacts_unique_email 
--   ON customer_contacts(org_id, value) 
--   WHERE type = 'email';

-- Trigger for updated_at
CREATE TRIGGER trigger_customer_contacts_updated_at
  BEFORE UPDATE ON customer_contacts
  FOR EACH ROW
  EXECUTE FUNCTION update_customers_updated_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `customer_id` | UUID | NO | - | FK to `customers.id` (CASCADE delete) |
| `type` | contact_type_enum | NO | - | Contact channel type |
| `value` | TEXT | NO | - | Contact value (email, phone number, etc.) |
| `is_primary` | BOOLEAN | NO | `false` | Primary contact flag for this type |
| `is_verified` | BOOLEAN | NO | `false` | Verification status |
| `opt_in_marketing` | BOOLEAN | NO | `true` | Marketing opt-in |
| `opt_in_transactional` | BOOLEAN | NO | `true` | Transactional opt-in |
| `preferred_channel` | BOOLEAN | NO | `false` | Preferred channel flag |
| `notes` | TEXT | YES | NULL | Notes about this contact |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- Email format validation for `type = 'email'`
- Phone format validation for `type IN ('mobile', 'phone', 'fax')` (E.164 format)
- At most one primary contact per `type` per customer per org (enforced via unique partial index)
- Optional: unique email per org (commented out; enable if business rule requires)

**Performance Notes**:
- Index on `(org_id, value)` supports contact lookups by value
- Partial unique index ensures data integrity for primary contacts

---

#### 3.2.5 `crm_preferences` Table

**Purpose**: Customer-level communication preferences and privacy flags.

**DDL**:

```sql
CREATE TABLE crm_preferences (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  do_not_contact BOOLEAN NOT NULL DEFAULT false,
  do_not_email BOOLEAN NOT NULL DEFAULT false,
  do_not_sms BOOLEAN NOT NULL DEFAULT false,
  do_not_call BOOLEAN NOT NULL DEFAULT false,
  preferred_contact_window_start TIME,
  preferred_contact_window_end TIME,
  notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Constraints
  CONSTRAINT chk_crm_preferences_contact_window CHECK (
    (preferred_contact_window_start IS NULL AND preferred_contact_window_end IS NULL) OR
    (preferred_contact_window_start IS NOT NULL AND preferred_contact_window_end IS NOT NULL AND
     preferred_contact_window_start < preferred_contact_window_end)
  )
);

-- Unique constraint: one preference record per customer per org
CREATE UNIQUE INDEX idx_crm_preferences_unique_customer 
  ON crm_preferences(org_id, customer_id);

-- Indexes
CREATE INDEX idx_crm_preferences_org_id ON crm_preferences(org_id);
CREATE INDEX idx_crm_preferences_customer_id ON crm_preferences(customer_id);
CREATE INDEX idx_crm_preferences_org_id_do_not_contact ON crm_preferences(org_id, do_not_contact) 
  WHERE do_not_contact = true;

-- Trigger for updated_at
CREATE TRIGGER trigger_crm_preferences_updated_at
  BEFORE UPDATE ON crm_preferences
  FOR EACH ROW
  EXECUTE FUNCTION update_customers_updated_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `customer_id` | UUID | NO | - | FK to `customers.id` (CASCADE delete) |
| `do_not_contact` | BOOLEAN | NO | `false` | Global do-not-contact flag |
| `do_not_email` | BOOLEAN | NO | `false` | Do-not-email flag |
| `do_not_sms` | BOOLEAN | NO | `false` | Do-not-SMS flag |
| `do_not_call` | BOOLEAN | NO | `false` | Do-not-call flag |
| `preferred_contact_window_start` | TIME | YES | NULL | Preferred contact start time |
| `preferred_contact_window_end` | TIME | YES | NULL | Preferred contact end time |
| `notes` | TEXT | YES | NULL | Notes about preferences |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- One preference record per customer per org (enforced via unique index)
- Contact window must have both start and end times if specified, and start < end
- `do_not_contact` is a global override (interpretation: if true, all channels are blocked regardless of channel-specific flags)

**Performance Notes**:
- Partial index on `do_not_contact = true` optimizes queries filtering opted-out customers

---

#### 3.2.6 `crm_interactions` Table

**Purpose**: Tracks communication and contact history.

**DDL**:

```sql
CREATE TABLE crm_interactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  location_id UUID REFERENCES customer_locations(id) ON DELETE SET NULL,
  related_work_order_id UUID, -- FK to work_orders table (deferred, module not yet implemented)
  related_quote_id UUID, -- FK to quotes table (deferred, module not yet implemented)
  channel interaction_channel_enum NOT NULL,
  direction interaction_direction_enum,
  subject TEXT,
  summary TEXT,
  body TEXT,
  metadata JSONB,
  sentiment interaction_sentiment_enum,
  created_by_user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Constraints
  CONSTRAINT chk_crm_interactions_occurred_at CHECK (
    occurred_at <= now() + interval '1 hour' -- Allow slight future timestamp for scheduled logs
  )
);

-- Indexes
CREATE INDEX idx_crm_interactions_org_id ON crm_interactions(org_id);
CREATE INDEX idx_crm_interactions_customer_id ON crm_interactions(customer_id);
CREATE INDEX idx_crm_interactions_org_id_customer_id ON crm_interactions(org_id, customer_id);
CREATE INDEX idx_crm_interactions_org_id_occurred_at ON crm_interactions(org_id, occurred_at DESC);
CREATE INDEX idx_crm_interactions_customer_id_occurred_at ON crm_interactions(customer_id, occurred_at DESC);
CREATE INDEX idx_crm_interactions_org_id_channel ON crm_interactions(org_id, channel);
CREATE INDEX idx_crm_interactions_org_id_sentiment ON crm_interactions(org_id, sentiment) 
  WHERE sentiment IS NOT NULL;
CREATE INDEX idx_crm_interactions_location_id ON crm_interactions(location_id) 
  WHERE location_id IS NOT NULL;
CREATE INDEX idx_crm_interactions_related_work_order_id ON crm_interactions(related_work_order_id) 
  WHERE related_work_order_id IS NOT NULL;
CREATE INDEX idx_crm_interactions_created_by_user_id ON crm_interactions(created_by_user_id) 
  WHERE created_by_user_id IS NOT NULL;

-- GIN index for JSONB metadata searches
CREATE INDEX idx_crm_interactions_metadata ON crm_interactions USING gin(metadata) 
  WHERE metadata IS NOT NULL;

-- Full-text search index for subject/summary/body
CREATE INDEX idx_crm_interactions_search ON crm_interactions USING gin(
  to_tsvector('english', coalesce(subject, '') || ' ' || coalesce(summary, '') || ' ' || coalesce(body, ''))
);
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `customer_id` | UUID | NO | - | FK to `customers.id` |
| `location_id` | UUID | YES | NULL | FK to `customer_locations.id` (SET NULL on delete) |
| `related_work_order_id` | UUID | YES | NULL | FK to work orders (deferred) |
| `related_quote_id` | UUID | YES | NULL | FK to quotes (deferred) |
| `channel` | interaction_channel_enum | NO | - | Communication channel |
| `direction` | interaction_direction_enum | YES | NULL | Direction of interaction |
| `subject` | TEXT | YES | NULL | Subject line (email, etc.) |
| `summary` | TEXT | YES | NULL | Brief summary |
| `body` | TEXT | YES | NULL | Full message body |
| `metadata` | JSONB | YES | NULL | Provider-specific data (message IDs, call duration, etc.) |
| `sentiment` | interaction_sentiment_enum | YES | NULL | AI-analyzed sentiment |
| `created_by_user_id` | UUID | YES | NULL | FK to `auth.users.id` |
| `occurred_at` | TIMESTAMPTZ | NO | `now()` | When interaction occurred |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Record creation timestamp |

**Business Rules**:
- `occurred_at` cannot be more than 1 hour in the future (allows slight scheduling flexibility)
- `metadata` JSONB structure examples:
  - Email: `{"message_id": "...", "thread_id": "...", "provider": "sendgrid"}`
  - Phone: `{"call_duration_seconds": 120, "call_sid": "...", "provider": "twilio"}`
  - SMS: `{"message_sid": "...", "provider": "twilio"}`

**Performance Notes**:
- Composite index on `(org_id, occurred_at DESC)` optimizes timeline queries
- GIN index on `metadata` enables efficient JSONB queries
- Full-text search index enables content search

---

#### 3.2.7 `crm_followups` Table

**Purpose**: Scheduled follow-ups and reminders.

**DDL**:

```sql
CREATE TABLE crm_followups (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  assigned_to_user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  title TEXT NOT NULL,
  description TEXT,
  due_at TIMESTAMPTZ NOT NULL,
  status followup_status_enum NOT NULL DEFAULT 'pending',
  priority followup_priority_enum NOT NULL DEFAULT 'medium',
  origin followup_origin_enum NOT NULL DEFAULT 'manual',
  related_interaction_id UUID REFERENCES crm_interactions(id) ON DELETE SET NULL,
  related_work_order_id UUID, -- FK to work_orders table (deferred)
  created_by_user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  completed_at TIMESTAMPTZ,
  completion_notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Constraints
  CONSTRAINT chk_crm_followups_completed_at CHECK (
    (status = 'completed' AND completed_at IS NOT NULL) OR
    (status != 'completed' AND completed_at IS NULL)
  )
);

-- Indexes
CREATE INDEX idx_crm_followups_org_id ON crm_followups(org_id);
CREATE INDEX idx_crm_followups_customer_id ON crm_followups(customer_id);
CREATE INDEX idx_crm_followups_org_id_due_at ON crm_followups(org_id, due_at);
CREATE INDEX idx_crm_followups_assigned_to_user_id_due_at ON crm_followups(assigned_to_user_id, due_at) 
  WHERE assigned_to_user_id IS NOT NULL;
CREATE INDEX idx_crm_followups_org_id_status ON crm_followups(org_id, status);
CREATE INDEX idx_crm_followups_org_id_status_due_at ON crm_followups(org_id, status, due_at) 
  WHERE status = 'pending';
CREATE INDEX idx_crm_followups_customer_id_status ON crm_followups(customer_id, status);
CREATE INDEX idx_crm_followups_related_interaction_id ON crm_followups(related_interaction_id) 
  WHERE related_interaction_id IS NOT NULL;

-- Trigger for updated_at
CREATE TRIGGER trigger_crm_followups_updated_at
  BEFORE UPDATE ON crm_followups
  FOR EACH ROW
  EXECUTE FUNCTION update_customers_updated_at();

-- Trigger to auto-set completed_at when status changes to 'completed'
CREATE OR REPLACE FUNCTION set_crm_followups_completed_at()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.status = 'completed' AND OLD.status != 'completed' THEN
    NEW.completed_at = now();
  ELSIF NEW.status != 'completed' THEN
    NEW.completed_at = NULL;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_crm_followups_completed_at
  BEFORE UPDATE ON crm_followups
  FOR EACH ROW
  EXECUTE FUNCTION set_crm_followups_completed_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `customer_id` | UUID | NO | - | FK to `customers.id` |
| `assigned_to_user_id` | UUID | YES | NULL | FK to `auth.users.id` (assignee) |
| `title` | TEXT | NO | - | Follow-up title |
| `description` | TEXT | YES | NULL | Detailed description |
| `due_at` | TIMESTAMPTZ | NO | - | Due date/time |
| `status` | followup_status_enum | NO | `'pending'` | Current status |
| `priority` | followup_priority_enum | NO | `'medium'` | Priority level |
| `origin` | followup_origin_enum | NO | `'manual'` | How follow-up was created |
| `related_interaction_id` | UUID | YES | NULL | FK to `crm_interactions.id` |
| `related_work_order_id` | UUID | YES | NULL | FK to work orders (deferred) |
| `created_by_user_id` | UUID | YES | NULL | FK to `auth.users.id` (creator) |
| `completed_at` | TIMESTAMPTZ | YES | NULL | Completion timestamp |
| `completion_notes` | TEXT | YES | NULL | Notes on completion |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- `completed_at` is automatically set when `status` changes to `'completed'` (via trigger)
- `completed_at` is cleared when status changes away from `'completed'`
- `due_at` is required and must be a valid timestamp

**Performance Notes**:
- Composite indexes on `(org_id, due_at)` and `(assigned_to_user_id, due_at)` optimize dashboard queries
- Partial index on `status = 'pending'` optimizes active follow-up queries

---

#### 3.2.8 `crm_tags` Table

**Purpose**: Tag definitions (reusable labels).

**DDL**:

```sql
CREATE TABLE crm_tags (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  description TEXT,
  color TEXT, -- Hex color code (e.g., '#FF5733')
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Constraints
  CONSTRAINT chk_crm_tags_color_format CHECK (
    color IS NULL OR color ~ '^#[0-9A-Fa-f]{6}$'
  )
);

-- Unique constraint: tag name unique per org
CREATE UNIQUE INDEX idx_crm_tags_unique_name ON crm_tags(org_id, name);

-- Indexes
CREATE INDEX idx_crm_tags_org_id ON crm_tags(org_id);
CREATE INDEX idx_crm_tags_org_id_name ON crm_tags(org_id, name);
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `name` | TEXT | NO | - | Tag name (unique per org) |
| `description` | TEXT | YES | NULL | Tag description |
| `color` | TEXT | YES | NULL | Hex color code (e.g., `#FF5733`) |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |

**Business Rules**:
- Tag name must be unique per org (enforced via unique index)
- Color must be valid hex format if provided (enforced via CHECK constraint)

---

#### 3.2.9 `crm_customer_tags` Table

**Purpose**: Many-to-many relationship between customers and tags.

**DDL**:

```sql
CREATE TABLE crm_customer_tags (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  tag_id UUID NOT NULL REFERENCES crm_tags(id) ON DELETE CASCADE,
  assigned_by_user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Ensure tag belongs to same org as customer
  CONSTRAINT chk_crm_customer_tags_org_consistency CHECK (
    EXISTS (
      SELECT 1 FROM customers c 
      WHERE c.id = customer_id AND c.org_id = org_id
    ) AND
    EXISTS (
      SELECT 1 FROM crm_tags t 
      WHERE t.id = tag_id AND t.org_id = org_id
    )
  )
);

-- Unique constraint: one tag assignment per customer per tag per org
CREATE UNIQUE INDEX idx_crm_customer_tags_unique ON crm_customer_tags(org_id, customer_id, tag_id);

-- Indexes
CREATE INDEX idx_crm_customer_tags_org_id ON crm_customer_tags(org_id);
CREATE INDEX idx_crm_customer_tags_customer_id ON crm_customer_tags(customer_id);
CREATE INDEX idx_crm_customer_tags_tag_id ON crm_customer_tags(tag_id);
CREATE INDEX idx_crm_customer_tags_org_id_customer_id ON crm_customer_tags(org_id, customer_id);
CREATE INDEX idx_crm_customer_tags_org_id_tag_id ON crm_customer_tags(org_id, tag_id);
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `customer_id` | UUID | NO | - | FK to `customers.id` (CASCADE delete) |
| `tag_id` | UUID | NO | - | FK to `crm_tags.id` (CASCADE delete) |
| `assigned_by_user_id` | UUID | YES | NULL | FK to `auth.users.id` (who assigned) |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |

**Business Rules**:
- One tag assignment per customer per tag per org (enforced via unique index)
- Tag and customer must belong to the same org (enforced via CHECK constraint)

---

#### 3.2.10 `crm_segments` Table

**Purpose**: Customer segment definitions.

**DDL**:

```sql
CREATE TABLE crm_segments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  description TEXT,
  type segment_type_enum NOT NULL,
  definition JSONB, -- Rule definition for rule_based segments
  ai_prompt TEXT, -- Prompt for ai_generated segments
  ai_explanation TEXT, -- Human-readable explanation from AI
  is_active BOOLEAN NOT NULL DEFAULT true,
  last_computed_at TIMESTAMPTZ,
  created_by_user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Constraints
  CONSTRAINT chk_crm_segments_rule_based_has_definition CHECK (
    (type = 'rule_based' AND definition IS NOT NULL) OR
    type != 'rule_based'
  ),
  CONSTRAINT chk_crm_segments_ai_generated_has_prompt CHECK (
    (type = 'ai_generated' AND ai_prompt IS NOT NULL) OR
    type != 'ai_generated'
  )
);

-- Unique constraint: segment name unique per org
CREATE UNIQUE INDEX idx_crm_segments_unique_name ON crm_segments(org_id, name);

-- Indexes
CREATE INDEX idx_crm_segments_org_id ON crm_segments(org_id);
CREATE INDEX idx_crm_segments_org_id_type ON crm_segments(org_id, type);
CREATE INDEX idx_crm_segments_org_id_is_active ON crm_segments(org_id, is_active) 
  WHERE is_active = true;
CREATE INDEX idx_crm_segments_definition ON crm_segments USING gin(definition) 
  WHERE definition IS NOT NULL;

-- Trigger for updated_at
CREATE TRIGGER trigger_crm_segments_updated_at
  BEFORE UPDATE ON crm_segments
  FOR EACH ROW
  EXECUTE FUNCTION update_customers_updated_at();
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `name` | TEXT | NO | - | Segment name (unique per org) |
| `description` | TEXT | YES | NULL | Segment description |
| `type` | segment_type_enum | NO | - | Segment type |
| `definition` | JSONB | YES | NULL | Rule definition (required for `rule_based`) |
| `ai_prompt` | TEXT | YES | NULL | AI prompt (required for `ai_generated`) |
| `ai_explanation` | TEXT | YES | NULL | AI-generated explanation |
| `is_active` | BOOLEAN | NO | `true` | Active status |
| `last_computed_at` | TIMESTAMPTZ | YES | NULL | Last computation timestamp |
| `created_by_user_id` | UUID | YES | NULL | FK to `auth.users.id` |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- `rule_based` segments must have `definition` JSONB (enforced via CHECK constraint)
- `ai_generated` segments must have `ai_prompt` (enforced via CHECK constraint)
- `definition` JSONB structure example (for `rule_based`):
  ```json
  {
    "operator": "AND",
    "rules": [
      {"field": "status", "operator": "equals", "value": "active"},
      {"field": "lifecycle_stage", "operator": "equals", "value": "customer"},
      {"field": "created_at", "operator": "greater_than", "value": "2024-01-01"}
    ]
  }
  ```

**Performance Notes**:
- GIN index on `definition` enables efficient JSONB queries for rule-based segments

---

#### 3.2.11 `crm_segment_members` Table

**Purpose**: Tracks which customers belong to which segments.

**DDL**:

```sql
CREATE TABLE crm_segment_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  segment_id UUID NOT NULL REFERENCES crm_segments(id) ON DELETE CASCADE,
  customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  score NUMERIC(5, 2), -- Optional ranking score (0.00 to 999.99)
  metadata JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Constraints
  CONSTRAINT chk_crm_segment_members_score_range CHECK (
    score IS NULL OR (score >= 0 AND score <= 999.99)
  ),
  -- Ensure segment and customer belong to same org
  CONSTRAINT chk_crm_segment_members_org_consistency CHECK (
    EXISTS (
      SELECT 1 FROM crm_segments s 
      WHERE s.id = segment_id AND s.org_id = org_id
    ) AND
    EXISTS (
      SELECT 1 FROM customers c 
      WHERE c.id = customer_id AND c.org_id = org_id
    )
  )
);

-- Unique constraint: one membership per customer per segment per org
CREATE UNIQUE INDEX idx_crm_segment_members_unique ON crm_segment_members(org_id, segment_id, customer_id);

-- Indexes
CREATE INDEX idx_crm_segment_members_org_id ON crm_segment_members(org_id);
CREATE INDEX idx_crm_segment_members_segment_id ON crm_segment_members(segment_id);
CREATE INDEX idx_crm_segment_members_customer_id ON crm_segment_members(customer_id);
CREATE INDEX idx_crm_segment_members_org_id_segment_id ON crm_segment_members(org_id, segment_id);
CREATE INDEX idx_crm_segment_members_org_id_customer_id ON crm_segment_members(org_id, customer_id);
CREATE INDEX idx_crm_segment_members_segment_id_score ON crm_segment_members(segment_id, score DESC NULLS LAST) 
  WHERE score IS NOT NULL;
CREATE INDEX idx_crm_segment_members_metadata ON crm_segment_members USING gin(metadata) 
  WHERE metadata IS NOT NULL;
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `segment_id` | UUID | NO | - | FK to `crm_segments.id` (CASCADE delete) |
| `customer_id` | UUID | NO | - | FK to `customers.id` (CASCADE delete) |
| `score` | NUMERIC(5,2) | YES | NULL | Optional ranking score (0.00-999.99) |
| `metadata` | JSONB | YES | NULL | Additional metadata (AI explanations, etc.) |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |

**Business Rules**:
- One membership per customer per segment per org (enforced via unique index)
- Score must be between 0.00 and 999.99 if provided
- Segment and customer must belong to the same org (enforced via CHECK constraint)

**Performance Notes**:
- Index on `(segment_id, score DESC NULLS LAST)` optimizes ranked member queries

---

#### 3.2.12 `crm_message_templates` Table

**Purpose**: Reusable message templates for communications.

**DDL**:

```sql
CREATE TABLE crm_message_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  channel message_template_channel_enum NOT NULL,
  subject TEXT,
  body TEXT NOT NULL,
  variables JSONB, -- List of supported variables (e.g., ["customer.first_name", "customer.email"])
  is_system BOOLEAN NOT NULL DEFAULT false,
  created_by_user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Unique constraint: template name unique per org per channel (optional, adjust if needed)
-- CREATE UNIQUE INDEX idx_crm_message_templates_unique_name 
--   ON crm_message_templates(org_id, channel, name);

-- Indexes
CREATE INDEX idx_crm_message_templates_org_id ON crm_message_templates(org_id);
CREATE INDEX idx_crm_message_templates_org_id_channel ON crm_message_templates(org_id, channel);
CREATE INDEX idx_crm_message_templates_org_id_is_system ON crm_message_templates(org_id, is_system) 
  WHERE is_system = true;
CREATE INDEX idx_crm_message_templates_variables ON crm_message_templates USING gin(variables) 
  WHERE variables IS NOT NULL;
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `name` | TEXT | NO | - | Template name |
| `channel` | message_template_channel_enum | NO | - | Communication channel |
| `subject` | TEXT | YES | NULL | Subject line (email, etc.) |
| `body` | TEXT | NO | - | Template body with variables |
| `variables` | JSONB | YES | NULL | Supported variables list |
| `is_system` | BOOLEAN | NO | `false` | System template flag (locked) |
| `created_by_user_id` | UUID | YES | NULL | FK to `auth.users.id` |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- `body` supports placeholder variables (e.g., `{{customer.first_name}}`, `{{customer.email}}`)
- `variables` JSONB structure example:
  ```json
  ["customer.first_name", "customer.last_name", "customer.email", "customer.phone"]
  ```
- System templates (`is_system = true`) are read-only for non-admin users (enforced in application logic, not DB)

**Performance Notes**:
- GIN index on `variables` enables efficient variable lookup queries

---

#### 3.2.13 `crm_automation_rules` Table

**Purpose**: Defines automation logic for CRM workflows.

**DDL**:

```sql
CREATE TABLE crm_automation_rules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  description TEXT,
  is_enabled BOOLEAN NOT NULL DEFAULT true,
  trigger_type automation_trigger_type_enum NOT NULL,
  event_type TEXT, -- e.g., 'work_order_completed', 'quote_sent'
  time_offset_minutes INTEGER, -- Minutes after event for time_based triggers
  segment_id UUID REFERENCES crm_segments(id) ON DELETE SET NULL,
  conditions JSONB, -- Additional filter conditions
  actions JSONB NOT NULL, -- Structured actions (create follow-up, send message, tag customer)
  created_by_user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Constraints
  CONSTRAINT chk_crm_automation_rules_event_type CHECK (
    (trigger_type = 'event' AND event_type IS NOT NULL) OR
    trigger_type != 'event'
  ),
  CONSTRAINT chk_crm_automation_rules_time_offset CHECK (
    (trigger_type = 'time_based' AND time_offset_minutes IS NOT NULL) OR
    trigger_type != 'time_based'
  ),
  CONSTRAINT chk_crm_automation_rules_segment_id CHECK (
    (trigger_type = 'segment_membership' AND segment_id IS NOT NULL) OR
    trigger_type != 'segment_membership'
  )
);

-- Indexes
CREATE INDEX idx_crm_automation_rules_org_id ON crm_automation_rules(org_id);
CREATE INDEX idx_crm_automation_rules_org_id_is_enabled ON crm_automation_rules(org_id, is_enabled) 
  WHERE is_enabled = true;
CREATE INDEX idx_crm_automation_rules_org_id_trigger_type ON crm_automation_rules(org_id, trigger_type);
CREATE INDEX idx_crm_automation_rules_event_type ON crm_automation_rules(event_type) 
  WHERE event_type IS NOT NULL;
CREATE INDEX idx_crm_automation_rules_segment_id ON crm_automation_rules(segment_id) 
  WHERE segment_id IS NOT NULL;
CREATE INDEX idx_crm_automation_rules_conditions ON crm_automation_rules USING gin(conditions) 
  WHERE conditions IS NOT NULL;
CREATE INDEX idx_crm_automation_rules_actions ON crm_automation_rules USING gin(actions);
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `name` | TEXT | NO | - | Rule name |
| `description` | TEXT | YES | NULL | Rule description |
| `is_enabled` | BOOLEAN | NO | `true` | Enabled status |
| `trigger_type` | automation_trigger_type_enum | NO | - | Trigger type |
| `event_type` | TEXT | YES | NULL | Event type (required for `event` trigger) |
| `time_offset_minutes` | INTEGER | YES | NULL | Time offset (required for `time_based`) |
| `segment_id` | UUID | YES | NULL | FK to `crm_segments.id` (required for `segment_membership`) |
| `conditions` | JSONB | YES | NULL | Additional filter conditions |
| `actions` | JSONB | NO | - | Actions to execute |
| `created_by_user_id` | UUID | YES | NULL | FK to `auth.users.id` |
| `created_at` | TIMESTAMPTZ | NO | `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | `now()` | Last update timestamp |

**Business Rules**:
- `event_type` required when `trigger_type = 'event'` (enforced via CHECK constraint)
- `time_offset_minutes` required when `trigger_type = 'time_based'` (enforced via CHECK constraint)
- `segment_id` required when `trigger_type = 'segment_membership'` (enforced via CHECK constraint)
- `actions` JSONB structure example:
  ```json
  [
    {
      "type": "create_followup",
      "title": "Follow up on quote",
      "description": "Check in on quote sent {{days_ago}} days ago",
      "due_at_offset_minutes": 1440,
      "priority": "high"
    },
    {
      "type": "send_message",
      "template_id": "uuid-here",
      "channel": "email"
    },
    {
      "type": "tag_customer",
      "tag_id": "uuid-here"
    }
  ]
  ```
- `conditions` JSONB structure example:
  ```json
  {
    "customer_status": "active",
    "lifecycle_stage": "customer"
  }
  ```

**Performance Notes**:
- GIN indexes on `conditions` and `actions` enable efficient JSONB queries

---

#### 3.2.14 `crm_automation_runs` Table

**Purpose**: Tracks executions of automation rules.

**DDL**:

```sql
CREATE TABLE crm_automation_runs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  rule_id UUID NOT NULL REFERENCES crm_automation_rules(id) ON DELETE CASCADE,
  customer_id UUID REFERENCES customers(id) ON DELETE SET NULL,
  trigger_context JSONB, -- Payload that caused the run
  status automation_run_status_enum NOT NULL DEFAULT 'pending',
  error_message TEXT,
  started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at TIMESTAMPTZ
);

-- Indexes
CREATE INDEX idx_crm_automation_runs_org_id ON crm_automation_runs(org_id);
CREATE INDEX idx_crm_automation_runs_rule_id ON crm_automation_runs(rule_id);
CREATE INDEX idx_crm_automation_runs_rule_id_started_at ON crm_automation_runs(rule_id, started_at DESC);
CREATE INDEX idx_crm_automation_runs_org_id_rule_id ON crm_automation_runs(org_id, rule_id);
CREATE INDEX idx_crm_automation_runs_customer_id ON crm_automation_runs(customer_id) 
  WHERE customer_id IS NOT NULL;
CREATE INDEX idx_crm_automation_runs_org_id_status ON crm_automation_runs(org_id, status);
CREATE INDEX idx_crm_automation_runs_trigger_context ON crm_automation_runs USING gin(trigger_context) 
  WHERE trigger_context IS NOT NULL;
```

**Field Specifications**:

| Column | Type | Nullable | Default | Description |
|--------|------|----------|---------|-------------|
| `id` | UUID | NO | `gen_random_uuid()` | Primary key |
| `org_id` | UUID | NO | - | FK to `orgs.id` |
| `rule_id` | UUID | NO | - | FK to `crm_automation_rules.id` (CASCADE delete) |
| `customer_id` | UUID | YES | NULL | FK to `customers.id` (if applicable) |
| `trigger_context` | JSONB | YES | NULL | Event payload or context |
| `status` | automation_run_status_enum | NO | `'pending'` | Run status |
| `error_message` | TEXT | YES | NULL | Error message if failed |
| `started_at` | TIMESTAMPTZ | NO | `now()` | Start timestamp |
| `completed_at` | TIMESTAMPTZ | YES | NULL | Completion timestamp |

**Business Rules**:
- `trigger_context` JSONB structure example:
  ```json
  {
    "event_type": "work_order_completed",
    "work_order_id": "uuid-here",
    "customer_id": "uuid-here",
    "completed_at": "2024-01-15T10:30:00Z"
  }
  ```

**Performance Notes**:
- Composite index on `(rule_id, started_at DESC)` optimizes recent run queries
- GIN index on `trigger_context` enables efficient JSONB queries

---

### 3.3 Deferred Foreign Keys

The following foreign keys are deferred until referenced tables exist (from other modules):

1. **`customers.primary_location_id`** → `customer_locations.id`
   - **Action**: Add after `customer_locations` table is created
   - **SQL**:
     ```sql
     ALTER TABLE customers 
       ADD CONSTRAINT fk_customers_primary_location 
       FOREIGN KEY (primary_location_id) 
       REFERENCES customer_locations(id) 
       ON DELETE SET NULL;
     ```

2. **`customers.primary_contact_id`** → `customer_contacts.id`
   - **Action**: Add after `customer_contacts` table is created
   - **SQL**:
     ```sql
     ALTER TABLE customers 
       ADD CONSTRAINT fk_customers_primary_contact 
       FOREIGN KEY (primary_contact_id) 
       REFERENCES customer_contacts(id) 
       ON DELETE SET NULL;
     ```

3. **`crm_interactions.related_work_order_id`** → `work_orders.id` (future module)
   - **Action**: Add when Work Order module is implemented
   - **Note**: Placeholder column exists; FK will be added later

4. **`crm_interactions.related_quote_id`** → `quotes.id` (future module)
   - **Action**: Add when Quoting module is implemented
   - **Note**: Placeholder column exists; FK will be added later

5. **`crm_followups.related_work_order_id`** → `work_orders.id` (future module)
   - **Action**: Add when Work Order module is implemented
   - **Note**: Placeholder column exists; FK will be added later

---

## 4. Migration Strategy

### 4.1 Migration Order

Migrations must be executed in the following order to satisfy dependencies:

1. **Create enums** (all enum types)
2. **Create `orgs` table** (if not exists)
3. **Create `customers` table**
4. **Create `customer_locations` table**
5. **Create `customer_contacts` table**
6. **Add deferred FKs** (`customers.primary_location_id`, `customers.primary_contact_id`)
7. **Create `crm_preferences` table**
8. **Create `crm_interactions` table**
9. **Create `crm_followups` table**
10. **Create `crm_tags` table**
11. **Create `crm_customer_tags` table**
12. **Create `crm_segments` table**
13. **Create `crm_segment_members` table**
14. **Create `crm_message_templates` table**
15. **Create `crm_automation_rules` table**
16. **Create `crm_automation_runs` table**

### 4.2 Migration File Structure

Recommended migration file naming convention:

```
supabase/migrations/
  20240101000000_create_crm_enums.sql
  20240101000001_create_orgs_table.sql
  20240101000002_create_customers_table.sql
  20240101000003_create_customer_locations_table.sql
  20240101000004_create_customer_contacts_table.sql
  20240101000005_add_customers_deferred_fks.sql
  20240101000006_create_crm_preferences_table.sql
  20240101000007_create_crm_interactions_table.sql
  20240101000008_create_crm_followups_table.sql
  20240101000009_create_crm_tags_tables.sql
  20240101000010_create_crm_segments_tables.sql
  20240101000011_create_crm_message_templates_table.sql
  20240101000012_create_crm_automation_tables.sql
```

### 4.3 Rollback Strategy

Each migration should be idempotent (use `CREATE TABLE IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS`, etc.). For rollback:

- Create corresponding `down` migrations that drop objects in reverse order
- Test rollback on non-production environments before applying to production

---

## 5. Seed Data Requirements

### 5.1 Test Organization

Create at least one test organization:

```sql
INSERT INTO orgs (id, name) 
VALUES 
  ('00000000-0000-0000-0000-000000000001', 'Test HVAC Company')
ON CONFLICT (id) DO NOTHING;
```

### 5.2 Sample Customers

Create sample customers for testing:

```sql
-- Individual customer
INSERT INTO customers (
  org_id, type, name, first_name, last_name, email, phone, status, lifecycle_stage, source
) VALUES (
  '00000000-0000-0000-0000-000000000001',
  'individual',
  'John Doe',
  'John',
  'Doe',
  'john.doe@example.com',
  '+15551234567',
  'active',
  'customer',
  'web'
);

-- Company customer
INSERT INTO customers (
  org_id, type, name, company_name, email, phone, status, lifecycle_stage, source
) VALUES (
  '00000000-0000-0000-0000-000000000001',
  'company',
  'ACME Corp',
  'ACME Corporation',
  'contact@acme.com',
  '+15559876543',
  'active',
  'customer',
  'referral'
);
```

### 5.3 Sample Locations

```sql
-- Location for individual customer (assuming customer ID from above)
INSERT INTO customer_locations (
  org_id, customer_id, label, type, address_line1, city, state, postal_code, country, is_primary
) VALUES (
  '00000000-0000-0000-0000-000000000001',
  (SELECT id FROM customers WHERE email = 'john.doe@example.com' LIMIT 1),
  'Home',
  'both',
  '123 Main St',
  'Springfield',
  'IL',
  '62701',
  'US',
  true
);
```

### 5.4 Sample Contacts

```sql
-- Contact for individual customer
INSERT INTO customer_contacts (
  org_id, customer_id, type, value, is_primary, is_verified, opt_in_marketing, opt_in_transactional
) VALUES (
  '00000000-0000-0000-0000-000000000001',
  (SELECT id FROM customers WHERE email = 'john.doe@example.com' LIMIT 1),
  'email',
  'john.doe@example.com',
  true,
  true,
  true,
  true
);
```

### 5.5 Sample Tags

```sql
INSERT INTO crm_tags (org_id, name, description, color) VALUES
  ('00000000-0000-0000-0000-000000000001', 'VIP', 'VIP customers', '#FFD700'),
  ('00000000-0000-0000-0000-000000000001', 'Warranty', 'Under warranty', '#00FF00'),
  ('00000000-0000-0000-0000-000000000001', 'Commercial', 'Commercial customers', '#0066CC');
```

### 5.6 Sample Segments

```sql
-- Static segment
INSERT INTO crm_segments (org_id, name, description, type, is_active) VALUES
  ('00000000-0000-0000-0000-000000000001', 'All Active Customers', 'All customers with active status', 'static', true);

-- Rule-based segment
INSERT INTO crm_segments (org_id, name, description, type, definition, is_active) VALUES
  (
    '00000000-0000-0000-0000-000000000001',
    'High-Value Customers',
    'Customers with active status and customer lifecycle stage',
    'rule_based',
    '{"operator": "AND", "rules": [{"field": "status", "operator": "equals", "value": "active"}, {"field": "lifecycle_stage", "operator": "equals", "value": "customer"}]}'::jsonb,
    true
  );
```

---

## 6. Performance Considerations

### 6.1 Index Strategy

- **Primary Keys**: All tables use UUID primary keys with default `gen_random_uuid()`
- **Foreign Keys**: Indexed for join performance
- **Composite Indexes**: Created for common query patterns (`org_id` + filter column)
- **Partial Indexes**: Used for filtered queries (e.g., `WHERE status = 'pending'`)
- **Full-Text Search**: GIN indexes on text fields for search functionality
- **JSONB**: GIN indexes on JSONB columns for efficient JSON queries

### 6.2 Query Performance Targets

- **Customer Search**: < 500ms for up to 50k customers per org
- **Interaction Timeline**: < 300ms for last 100 interactions per customer
- **Follow-ups Dashboard**: < 400ms for upcoming follow-ups (paginated)
- **Segment Member Queries**: < 600ms for segments with up to 10k members

### 6.3 Optimization Notes

- Use `EXPLAIN ANALYZE` to validate query plans
- Monitor index usage via `pg_stat_user_indexes`
- Consider partitioning `crm_interactions` by `occurred_at` if volume exceeds 1M rows per org
- Consider archiving old interactions/follow-ups to separate tables

---

## 7. Validation & Constraints

### 7.1 Data Validation Rules

All validation rules are enforced at the database level via CHECK constraints:

- **Email Format**: Basic regex validation (`^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$`)
- **Phone Format**: E.164 format validation (`^\+?[1-9]\d{1,14}$`)
- **Coordinates**: Latitude (-90 to 90), Longitude (-180 to 180)
- **Color Codes**: Hex format (`^#[0-9A-Fa-f]{6}$`)
- **Score Ranges**: Numeric ranges (e.g., 0.00 to 999.99)

### 7.2 Business Logic Constraints

- **Primary Location/Contact**: At most one per type per customer per org (enforced via unique partial indexes)
- **Preferences**: One record per customer per org (enforced via unique index)
- **Tag Assignments**: One assignment per customer per tag per org (enforced via unique index)
- **Segment Memberships**: One membership per customer per segment per org (enforced via unique index)

---

## 8. Integration Points

### 8.1 Supabase Auth Integration

- **User References**: All `created_by_user_id` and `assigned_to_user_id` columns reference `auth.users.id`
- **RLS Policies**: Will be implemented in Epic 2 (Story CRM-014, CRM-015)
- **Profile Table**: Assumes `profiles` table exists with `org_id` column (or equivalent)

### 8.2 Future Module Integrations

- **Work Orders**: Placeholder columns (`related_work_order_id`) exist; FKs will be added when module is implemented
- **Quotes**: Placeholder columns (`related_quote_id`) exist; FKs will be added when module is implemented
- **Scheduling/Dispatch**: `customer_locations` will be consumed for routing

---

## 9. Documentation Requirements

### 9.1 Schema Documentation

- Document all tables, columns, constraints, and indexes
- Include JSONB schema examples for `definition`, `actions`, `conditions`, `metadata`
- Document enum values and their meanings

### 9.2 API Documentation

- Document how `org_id` is derived from authenticated user
- Document deferred FK strategy and when they will be added
- Document JSONB structure expectations for all JSONB columns

### 9.3 Developer Onboarding

- Provide migration execution instructions
- Provide seed data scripts
- Document how to verify schema in Supabase dashboard

---

## 10. Testing Requirements

### 10.1 Schema Validation Tests

- Verify all tables exist with correct columns
- Verify all indexes are created
- Verify all constraints are enforced
- Verify all triggers function correctly

### 10.2 Data Integrity Tests

- Verify `org_id` isolation (queries from one org cannot see another org's data)
- Verify cascade deletes work correctly
- Verify unique constraints prevent duplicates
- Verify CHECK constraints reject invalid data

### 10.3 Performance Tests

- Run `EXPLAIN ANALYZE` on representative queries
- Verify indexes are used in query plans
- Measure query performance against targets

---

## 11. Implementation Checklist

### Story CRM-001: Multi-Tenancy & Org Model
- [ ] `orgs` table created (or decision documented to reuse existing)
- [ ] All CRM tables include `org_id` column
- [ ] `org_id` derivation pattern documented
- [ ] Example queries demonstrate org isolation

### Story CRM-002: `customers` Table
- [ ] `customers` table created with all specified columns
- [ ] Enums defined (`customer_type_enum`, `customer_status_enum`, `customer_lifecycle_stage_enum`)
- [ ] Indexes created
- [ ] Constraints enforced
- [ ] Test data created

### Story CRM-003: `customer_locations` Table
- [ ] `customer_locations` table created
- [ ] `location_type_enum` defined
- [ ] Primary location constraint enforced
- [ ] Geo indexes created (if applicable)
- [ ] Test data with multiple locations created

### Story CRM-004: `customer_contacts` Table
- [ ] `customer_contacts` table created
- [ ] `contact_type_enum` defined
- [ ] Primary contact constraint enforced
- [ ] Email/phone format validation implemented
- [ ] Test data with multiple contact types created

### Story CRM-005: `crm_preferences` Table
- [ ] `crm_preferences` table created
- [ ] Unique constraint on `customer_id` enforced
- [ ] Contact window validation implemented
- [ ] Test data with various preference combinations created

### Story CRM-006: `crm_interactions` Table
- [ ] `crm_interactions` table created
- [ ] Enums defined (`interaction_channel_enum`, `interaction_direction_enum`, `interaction_sentiment_enum`)
- [ ] Indexes created (including JSONB and full-text)
- [ ] Test data with multiple channels and directions created

### Story CRM-007: `crm_followups` Table
- [ ] `crm_followups` table created
- [ ] Enums defined (`followup_status_enum`, `followup_priority_enum`, `followup_origin_enum`)
- [ ] Indexes created
- [ ] `completed_at` trigger implemented
- [ ] Test data with various statuses and priorities created

### Story CRM-008: `crm_tags` and `crm_customer_tags` Tables
- [ ] Both tables created
- [ ] Unique constraints enforced
- [ ] Color format validation implemented
- [ ] Test data with tags and assignments created

### Story CRM-009: `crm_segments` and `crm_segment_members` Tables
- [ ] Both tables created
- [ ] `segment_type_enum` defined
- [ ] Unique constraints enforced
- [ ] JSONB indexes created
- [ ] Test data with static and rule-based segments created

### Story CRM-010: `crm_message_templates` Table
- [ ] `crm_message_templates` table created
- [ ] `message_template_channel_enum` defined
- [ ] JSONB indexes created
- [ ] Test data with system and org templates created

### Story CRM-011: `crm_automation_rules` and `crm_automation_runs` Tables
- [ ] Both tables created
- [ ] `automation_trigger_type_enum` and `automation_run_status_enum` defined
- [ ] Validation constraints enforced
- [ ] JSONB indexes created
- [ ] Test data with sample rules and runs created

### Story CRM-012: Indexing & Performance Strategy
- [ ] All required indexes implemented
- [ ] Performance targets documented
- [ ] Example `EXPLAIN ANALYZE` output captured
- [ ] Deferred indexes documented

---

## 12. Appendix: Complete SQL Migration Script

A complete, executable migration script combining all DDL statements in correct order is provided in a separate file: `supabase/migrations/YYYYMMDDHHMMSS_create_crm_epic1_schema.sql`

**Note**: This TDD provides the specification; the actual migration script should be generated from these specifications and tested in a development environment before applying to production.

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 1 – CRM Core Data Model & Supabase Schema. All specifications are designed to be directly consumable by LLM-based code generators, with exact data types, constraints, indexes, and relationships defined.

