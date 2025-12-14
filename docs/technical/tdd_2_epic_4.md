# Technical Design Document – Epic 4: Technician Configuration & Availability APIs

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 4 – Technician Configuration & Availability APIs
- **Source**: Derived from `fdd_2_agile.md` Epic 4 (Stories DISP-021 through DISP-025)
- **Reference Documents**: 
  - `fdd_2.md` (Functional Design Document for Scheduling & Dispatch)
  - `fdd_2_agile.md` (Agile User Stories)
  - `tdd_2_epic_1.md` (Dispatch Epic 1 for tenancy/role conventions)
  - `tdd_2_epic_2.md` (Dispatch Epic 2 for table schemas)
  - `tdd_2_epic_3.md` (Dispatch Epic 3 for RLS policies)
- **Target Platform**: Supabase (PostgreSQL 15+, Edge Functions)
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing Technician Configuration & Availability APIs as Supabase Edge Functions. It covers:

- Technician profile CRUD endpoints
- Technician skills management endpoints
- Technician service zones management endpoints
- Shifts creation and listing endpoints
- Time-off creation and listing endpoints

All APIs are implemented as Supabase Edge Functions (Deno/TypeScript) that enforce authorization via RLS policies (from Epic 3) and provide comprehensive validation, error handling, and response formatting.

This epic assumes Epic 1 (tenancy/roles), Epic 2 (tables), and Epic 3 (RLS policies) are complete.

---

## 2. Prerequisites

### 2.1 Required Components

Before implementing Epic 4, ensure:

1. **Epic 1 Complete**: Helper functions exist:
   - `get_user_org_id()` - Returns authenticated user's org_id
   - `get_user_role()` - Returns authenticated user's role

2. **Epic 2 Complete**: All dispatch tables exist:
   - `technician_profiles`
   - `technician_skills`
   - `service_zones`
   - `technician_service_zones`
   - `technician_shifts`
   - `technician_time_off`

3. **Epic 3 Complete**: RLS policies are enabled and configured

4. **Required Tables**:
   - `orgs` table exists
   - `profiles` table exists
   - `auth.users` table exists (Supabase Auth)
   - `customers` table exists (from CRM, for home_base_location_id validation)
   - `customer_locations` table exists (from CRM)

### 2.2 Edge Function Setup

Edge Functions are deployed to Supabase and accessible via:
- Base URL: `https://<project-ref>.supabase.co/functions/v1`
- Authentication: Bearer token from Supabase Auth JWT

---

## 3. Common Patterns

### 3.1 Authorization Helper

All Edge Functions use this authorization pattern:

```typescript
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

interface AuthResult {
  orgId: string;
  role: string;
  userId: string;
}

async function authorizeUser(
  supabase: SupabaseClient,
  userId: string,
  requiredRoles: string[]
): Promise<AuthResult | null> {
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
  
  return {
    orgId: profile.org_id,
    role: profile.role,
    userId: userId
  };
}
```

### 3.2 Error Response Helper

Standardized error responses:

```typescript
function errorResponse(
  message: string,
  status: number = 400,
  code?: string,
  details?: any
): Response {
  return new Response(
    JSON.stringify({
      error: {
        message,
        code: code || 'ERROR',
        details
      }
    }),
    {
      status,
      headers: { 'Content-Type': 'application/json' }
    }
  );
}
```

### 3.3 Success Response Helper

Standardized success responses:

```typescript
function successResponse(data: any, status: number = 200): Response {
  return new Response(
    JSON.stringify({ data }),
    {
      status,
      headers: { 'Content-Type': 'application/json' }
    }
  );
}
```

### 3.4 Request Validation Helper

```typescript
function validateRequiredFields(
  body: any,
  requiredFields: string[]
): { valid: boolean; missing: string[] } {
  const missing = requiredFields.filter(field => !(field in body));
  return {
    valid: missing.length === 0,
    missing
  };
}
```

---

## 4. API Endpoints

### 4.1 Story DISP-021: Technician Profiles API

#### 4.1.1 POST /dispatch/technicians

**Purpose**: Create a new technician profile.

**Authorization**: `admin`, `dispatcher`

**Request Schema**:

```typescript
interface CreateTechnicianRequest {
  user_id: string; // UUID, required, must exist in auth.users
  display_name: string; // required, min 1 char, max 255 chars
  employment_type?: 'employee' | 'contractor' | 'subcontractor';
  is_active?: boolean; // default: true
  home_base_location_id?: string; // UUID, optional, must exist in customer_locations
  default_service_zone_id?: string; // UUID, optional, must exist in service_zones
  max_daily_work_minutes?: number; // optional, must be > 0 if provided
  max_concurrent_jobs?: number; // optional, default: 1, must be > 0
  vehicle_type?: 'van' | 'truck' | 'car' | 'other';
  metadata?: Record<string, any>; // optional JSONB
}
```

**Request Example**:

```json
{
  "user_id": "123e4567-e89b-12d3-a456-426614174000",
  "display_name": "John Smith",
  "employment_type": "employee",
  "is_active": true,
  "max_daily_work_minutes": 480,
  "max_concurrent_jobs": 1,
  "vehicle_type": "van"
}
```

**Response Schema**:

```typescript
interface TechnicianProfileResponse {
  id: string;
  org_id: string;
  user_id: string;
  display_name: string;
  employment_type: string | null;
  is_active: boolean;
  home_base_location_id: string | null;
  default_service_zone_id: string | null;
  max_daily_work_minutes: number | null;
  max_concurrent_jobs: number;
  vehicle_type: string | null;
  metadata: Record<string, any> | null;
  created_at: string;
  updated_at: string;
}
```

**Response Example**:

```json
{
  "data": {
    "id": "789e4567-e89b-12d3-a456-426614174001",
    "org_id": "00000000-0000-0000-0000-000000000001",
    "user_id": "123e4567-e89b-12d3-a456-426614174000",
    "display_name": "John Smith",
    "employment_type": "employee",
    "is_active": true,
    "home_base_location_id": null,
    "default_service_zone_id": null,
    "max_daily_work_minutes": 480,
    "max_concurrent_jobs": 1,
    "vehicle_type": "van",
    "metadata": null,
    "created_at": "2024-01-15T10:30:00Z",
    "updated_at": "2024-01-15T10:30:00Z"
  }
}
```

**Validation Rules**:

1. `user_id` must exist in `auth.users`
2. `user_id` must not already have a technician profile in the same org (unique constraint)
3. `display_name` must be 1-255 characters
4. `home_base_location_id` must exist in `customer_locations` and belong to the same org
5. `default_service_zone_id` must exist in `service_zones` and belong to the same org
6. `max_daily_work_minutes` must be > 0 if provided
7. `max_concurrent_jobs` must be > 0

**Error Responses**:

- `400 Bad Request`: Missing required fields, invalid data
- `403 Forbidden`: User not authorized (not admin/dispatcher)
- `404 Not Found`: `user_id` doesn't exist in auth.users
- `409 Conflict`: Technician profile already exists for user in org
- `422 Unprocessable Entity`: Foreign key validation failed (location_id, service_zone_id)

**Implementation** (Edge Function):

```typescript
Deno.serve(async (req) => {
  if (req.method !== 'POST') {
    return errorResponse('Method not allowed', 405);
  }

  const authHeader = req.headers.get('Authorization');
  if (!authHeader) {
    return errorResponse('Unauthorized', 401);
  }

  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_ANON_KEY')!,
    { global: { headers: { Authorization: authHeader } } }
  );

  const { data: { user }, error: userError } = await supabase.auth.getUser();
  if (userError || !user) {
    return errorResponse('Unauthorized', 401);
  }

  const auth = await authorizeUser(supabase, user.id, ['admin', 'dispatcher']);
  if (!auth) {
    return errorResponse('Forbidden', 403, 'INSUFFICIENT_PERMISSIONS');
  }

  const body = await req.json();
  
  // Validate required fields
  const validation = validateRequiredFields(body, ['user_id', 'display_name']);
  if (!validation.valid) {
    return errorResponse(
      `Missing required fields: ${validation.missing.join(', ')}`,
      400,
      'MISSING_FIELDS',
      { missing: validation.missing }
    );
  }

  // Validate user_id exists
  const { data: authUser, error: authUserError } = await supabase.auth.admin.getUserById(body.user_id);
  if (authUserError || !authUser) {
    return errorResponse('User not found', 404, 'USER_NOT_FOUND');
  }

  // Validate display_name length
  if (body.display_name.length < 1 || body.display_name.length > 255) {
    return errorResponse('display_name must be 1-255 characters', 400, 'INVALID_DISPLAY_NAME');
  }

  // Validate max_daily_work_minutes
  if (body.max_daily_work_minutes !== undefined && body.max_daily_work_minutes <= 0) {
    return errorResponse('max_daily_work_minutes must be > 0', 400, 'INVALID_MAX_DAILY_WORK');
  }

  // Validate max_concurrent_jobs
  if (body.max_concurrent_jobs !== undefined && body.max_concurrent_jobs <= 0) {
    return errorResponse('max_concurrent_jobs must be > 0', 400, 'INVALID_MAX_CONCURRENT');
  }

  // Validate home_base_location_id if provided
  if (body.home_base_location_id) {
    const { data: location, error: locationError } = await supabase
      .from('customer_locations')
      .select('id, org_id')
      .eq('id', body.home_base_location_id)
      .eq('org_id', auth.orgId)
      .single();
    
    if (locationError || !location) {
      return errorResponse('Invalid home_base_location_id', 422, 'INVALID_LOCATION');
    }
  }

  // Validate default_service_zone_id if provided
  if (body.default_service_zone_id) {
    const { data: zone, error: zoneError } = await supabase
      .from('service_zones')
      .select('id, org_id')
      .eq('id', body.default_service_zone_id)
      .eq('org_id', auth.orgId)
      .single();
    
    if (zoneError || !zone) {
      return errorResponse('Invalid default_service_zone_id', 422, 'INVALID_ZONE');
    }
  }

  // Create technician profile
  const { data: technician, error: createError } = await supabase
    .from('technician_profiles')
    .insert({
      org_id: auth.orgId,
      user_id: body.user_id,
      display_name: body.display_name,
      employment_type: body.employment_type || null,
      is_active: body.is_active !== undefined ? body.is_active : true,
      home_base_location_id: body.home_base_location_id || null,
      default_service_zone_id: body.default_service_zone_id || null,
      max_daily_work_minutes: body.max_daily_work_minutes || null,
      max_concurrent_jobs: body.max_concurrent_jobs || 1,
      vehicle_type: body.vehicle_type || null,
      metadata: body.metadata || null
    })
    .select()
    .single();

  if (createError) {
    // Check for unique constraint violation
    if (createError.code === '23505') {
      return errorResponse(
        'Technician profile already exists for this user',
        409,
        'DUPLICATE_TECHNICIAN'
      );
    }
    return errorResponse('Failed to create technician', 500, 'CREATE_ERROR', { error: createError.message });
  }

  return successResponse(technician, 201);
});
```

#### 4.1.2 PATCH /dispatch/technicians/:id

**Purpose**: Update an existing technician profile.

**Authorization**: `admin`, `dispatcher`

**Request Schema**:

```typescript
interface UpdateTechnicianRequest {
  display_name?: string; // min 1 char, max 255 chars
  employment_type?: 'employee' | 'contractor' | 'subcontractor' | null;
  is_active?: boolean;
  home_base_location_id?: string | null; // UUID, must exist in customer_locations
  default_service_zone_id?: string | null; // UUID, must exist in service_zones
  max_daily_work_minutes?: number | null; // must be > 0 if provided
  max_concurrent_jobs?: number; // must be > 0
  vehicle_type?: 'van' | 'truck' | 'car' | 'other' | null;
  metadata?: Record<string, any> | null;
}
```

**Request Example**:

```json
{
  "display_name": "John Smith Jr.",
  "max_daily_work_minutes": 500,
  "is_active": false
}
```

**Response Schema**: Same as POST response

**Validation Rules**: Same as POST, but all fields optional

**Error Responses**:

- `400 Bad Request`: Invalid data
- `403 Forbidden`: User not authorized
- `404 Not Found`: Technician profile not found
- `422 Unprocessable Entity`: Foreign key validation failed

**Implementation**: Similar to POST, but uses `.update()` instead of `.insert()`

#### 4.1.3 GET /dispatch/technicians

**Purpose**: List all technicians in the organization.

**Authorization**: `admin`, `dispatcher`, `csr`, `technician`

**Query Parameters**:

```typescript
interface ListTechniciansQuery {
  is_active?: boolean; // filter by active status
  employment_type?: 'employee' | 'contractor' | 'subcontractor'; // filter by employment type
  limit?: number; // default: 100, max: 1000
  offset?: number; // default: 0
}
```

**Response Schema**:

```typescript
interface ListTechniciansResponse {
  data: TechnicianProfileResponse[];
  pagination: {
    total: number;
    limit: number;
    offset: number;
    has_more: boolean;
  };
}
```

**Implementation**: Uses Supabase `.select()` with RLS filtering

#### 4.1.4 GET /dispatch/technicians/:id

**Purpose**: Get a specific technician profile.

**Authorization**: `admin`, `dispatcher`, `csr`, `technician` (technicians can only read own profile)

**Response Schema**: Same as POST response

**Error Responses**:

- `403 Forbidden`: User not authorized or technician trying to read another technician's profile
- `404 Not Found`: Technician profile not found

---

### 4.2 Story DISP-022: Technician Skills API

#### 4.2.1 POST /dispatch/technicians/:id/skills

**Purpose**: Add a skill to a technician.

**Authorization**: `admin`, `dispatcher`

**Request Schema**:

```typescript
interface AddTechnicianSkillRequest {
  skill_code: string; // required, e.g., 'hvac_install'
  proficiency_level?: 'junior' | 'mid' | 'senior' | 'expert'; // default: 'mid'
  is_primary?: boolean; // default: false
}
```

**Request Example**:

```json
{
  "skill_code": "hvac_install",
  "proficiency_level": "senior",
  "is_primary": true
}
```

**Response Schema**:

```typescript
interface TechnicianSkillResponse {
  id: string;
  org_id: string;
  technician_id: string;
  skill_code: string;
  proficiency_level: string;
  is_primary: boolean;
  created_at: string;
}
```

**Validation Rules**:

1. `technician_id` must exist and belong to the same org
2. `skill_code` must not be empty
3. Unique constraint: (`org_id`, `technician_id`, `skill_code`) must be unique

**Error Responses**:

- `400 Bad Request`: Missing required fields, invalid data
- `403 Forbidden`: User not authorized
- `404 Not Found`: Technician not found
- `409 Conflict`: Skill already exists for technician

**Implementation**: Similar pattern to technician profile creation

#### 4.2.2 PATCH /dispatch/technicians/:id/skills/:skill_id

**Purpose**: Update a technician skill.

**Authorization**: `admin`, `dispatcher`

**Request Schema**:

```typescript
interface UpdateTechnicianSkillRequest {
  proficiency_level?: 'junior' | 'mid' | 'senior' | 'expert';
  is_primary?: boolean;
}
```

**Implementation**: Update skill record

#### 4.2.3 DELETE /dispatch/technicians/:id/skills/:skill_id

**Purpose**: Remove a skill from a technician.

**Authorization**: `admin`, `dispatcher`

**Response**: `204 No Content` on success

**Error Responses**:

- `403 Forbidden`: User not authorized
- `404 Not Found`: Skill not found

#### 4.2.4 GET /dispatch/technicians/:id/skills

**Purpose**: List all skills for a technician.

**Authorization**: `admin`, `dispatcher`, `csr`, `technician` (technicians can only read own skills)

**Response Schema**:

```typescript
interface ListTechnicianSkillsResponse {
  data: TechnicianSkillResponse[];
}
```

---

### 4.3 Story DISP-023: Technician Service Zones API

#### 4.3.1 POST /dispatch/technicians/:id/service-zones

**Purpose**: Add a service zone to a technician.

**Authorization**: `admin`, `dispatcher`

**Request Schema**:

```typescript
interface AddTechnicianServiceZoneRequest {
  service_zone_id: string; // UUID, required, must exist in service_zones
  is_primary?: boolean; // default: false
}
```

**Request Example**:

```json
{
  "service_zone_id": "456e4567-e89b-12d3-a456-426614174002",
  "is_primary": true
}
```

**Response Schema**:

```typescript
interface TechnicianServiceZoneResponse {
  id: string;
  org_id: string;
  technician_id: string;
  service_zone_id: string;
  is_primary: boolean;
  created_at: string;
}
```

**Validation Rules**:

1. `technician_id` must exist and belong to the same org
2. `service_zone_id` must exist in `service_zones` and belong to the same org
3. Unique constraint: (`org_id`, `technician_id`, `service_zone_id`) must be unique
4. Primary zone logic: If `is_primary = true`, unset primary flag on other zones for this technician

**Primary Zone Logic Implementation**:

```typescript
// If setting as primary, unset other primary zones
if (body.is_primary) {
  await supabase
    .from('technician_service_zones')
    .update({ is_primary: false })
    .eq('org_id', auth.orgId)
    .eq('technician_id', technicianId)
    .neq('service_zone_id', body.service_zone_id);
}

// Then insert/update the zone assignment
const { data, error } = await supabase
  .from('technician_service_zones')
  .upsert({
    org_id: auth.orgId,
    technician_id: technicianId,
    service_zone_id: body.service_zone_id,
    is_primary: body.is_primary || false
  }, {
    onConflict: 'org_id,technician_id,service_zone_id'
  })
  .select()
  .single();
```

**Error Responses**:

- `400 Bad Request`: Missing required fields
- `403 Forbidden`: User not authorized
- `404 Not Found`: Technician or service zone not found
- `409 Conflict`: Zone assignment already exists

#### 4.3.2 DELETE /dispatch/technicians/:id/service-zones/:zone_id

**Purpose**: Remove a service zone from a technician.

**Authorization**: `admin`, `dispatcher`

**Response**: `204 No Content` on success

#### 4.3.3 GET /dispatch/technicians/:id/service-zones

**Purpose**: List all service zones for a technician.

**Authorization**: `admin`, `dispatcher`, `csr`, `technician` (technicians can only read own zones)

**Response Schema**:

```typescript
interface ListTechnicianServiceZonesResponse {
  data: TechnicianServiceZoneResponse[];
}
```

---

### 4.4 Story DISP-024: Shifts API

#### 4.4.1 POST /dispatch/technicians/:id/shifts

**Purpose**: Create a shift for a technician.

**Authorization**: `admin`, `dispatcher`

**Request Schema**:

```typescript
interface CreateShiftRequest {
  starts_at: string; // ISO 8601 timestamp, required
  ends_at: string; // ISO 8601 timestamp, required, must be after starts_at
  shift_type?: 'regular' | 'on_call' | 'overtime' | 'training'; // default: 'regular'
  recurrence_rule?: string; // iCal RRULE string, optional
  is_active?: boolean; // default: true
}
```

**Request Example**:

```json
{
  "starts_at": "2024-01-15T08:00:00Z",
  "ends_at": "2024-01-15T17:00:00Z",
  "shift_type": "regular",
  "recurrence_rule": "FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR",
  "is_active": true
}
```

**Response Schema**:

```typescript
interface ShiftResponse {
  id: string;
  org_id: string;
  technician_id: string;
  starts_at: string;
  ends_at: string;
  shift_type: string;
  recurrence_rule: string | null;
  is_active: boolean;
  created_by_user_id: string | null;
  created_at: string;
  updated_at: string;
}
```

**Validation Rules**:

1. `technician_id` must exist and belong to the same org
2. `ends_at` must be after `starts_at`
3. `starts_at` and `ends_at` must be valid ISO 8601 timestamps
4. `recurrence_rule` must be valid iCal RRULE format if provided (validation can be deferred to application layer)

**Overlap Detection** (Optional Warning):

```typescript
// Check for overlapping shifts (warning, not blocking)
const { data: overlappingShifts } = await supabase
  .from('technician_shifts')
  .select('id, starts_at, ends_at')
  .eq('org_id', auth.orgId)
  .eq('technician_id', technicianId)
  .eq('is_active', true)
  .or(`and(starts_at.lte.${body.ends_at},ends_at.gte.${body.starts_at})`);

if (overlappingShifts && overlappingShifts.length > 0) {
  // Log warning but don't block creation
  console.warn('Overlapping shifts detected:', overlappingShifts);
}
```

**Error Responses**:

- `400 Bad Request`: Missing required fields, invalid timestamps, ends_at before starts_at
- `403 Forbidden`: User not authorized
- `404 Not Found`: Technician not found

**Implementation**: Create shift record with validation

#### 4.4.2 GET /dispatch/technicians/:id/shifts

**Purpose**: List shifts for a technician.

**Authorization**: `admin`, `dispatcher`, `csr`, `technician` (technicians can only read own shifts)

**Query Parameters**:

```typescript
interface ListShiftsQuery {
  start_date?: string; // ISO 8601 date, filter shifts starting on or after
  end_date?: string; // ISO 8601 date, filter shifts ending on or before
  is_active?: boolean; // filter by active status
  shift_type?: 'regular' | 'on_call' | 'overtime' | 'training'; // filter by type
  limit?: number; // default: 100, max: 1000
  offset?: number; // default: 0
}
```

**Response Schema**:

```typescript
interface ListShiftsResponse {
  data: ShiftResponse[];
  pagination: {
    total: number;
    limit: number;
    offset: number;
    has_more: boolean;
  };
}
```

**Date Range Filtering Implementation**:

```typescript
let query = supabase
  .from('technician_shifts')
  .select('*', { count: 'exact' })
  .eq('org_id', auth.orgId)
  .eq('technician_id', technicianId);

if (queryParams.start_date) {
  query = query.gte('starts_at', queryParams.start_date);
}

if (queryParams.end_date) {
  query = query.lte('ends_at', queryParams.end_date);
}

if (queryParams.is_active !== undefined) {
  query = query.eq('is_active', queryParams.is_active);
}

if (queryParams.shift_type) {
  query = query.eq('shift_type', queryParams.shift_type);
}

const limit = Math.min(queryParams.limit || 100, 1000);
const offset = queryParams.offset || 0;

const { data, error, count } = await query
  .order('starts_at', { ascending: true })
  .range(offset, offset + limit - 1);
```

---

### 4.5 Story DISP-025: Time Off API

#### 4.5.1 POST /dispatch/technicians/:id/time-off

**Purpose**: Create a time-off record for a technician.

**Authorization**: `admin`, `dispatcher`, `technician` (technicians can create own time-off)

**Request Schema**:

```typescript
interface CreateTimeOffRequest {
  starts_at: string; // ISO 8601 timestamp, required
  ends_at: string; // ISO 8601 timestamp, required, must be after starts_at
  reason?: 'vacation' | 'sick' | 'personal' | 'other'; // default: 'personal'
  notes?: string; // optional
}
```

**Request Example**:

```json
{
  "starts_at": "2024-02-01T00:00:00Z",
  "ends_at": "2024-02-05T23:59:59Z",
  "reason": "vacation",
  "notes": "Family vacation"
}
```

**Response Schema**:

```typescript
interface TimeOffResponse {
  id: string;
  org_id: string;
  technician_id: string;
  starts_at: string;
  ends_at: string;
  reason: string;
  notes: string | null;
  created_by_user_id: string | null;
  created_at: string;
  updated_at: string;
}
```

**Validation Rules**:

1. `technician_id` must exist and belong to the same org
2. If technician role: must be creating time-off for themselves (`technician_id` matches their profile)
3. `ends_at` must be after `starts_at`
4. `starts_at` and `ends_at` must be valid ISO 8601 timestamps

**Overlap Detection** (Optional Warning):

```typescript
// Check for overlapping time-off (warning, not blocking)
const { data: overlappingTimeOff } = await supabase
  .from('technician_time_off')
  .select('id, starts_at, ends_at')
  .eq('org_id', auth.orgId)
  .eq('technician_id', technicianId)
  .or(`and(starts_at.lte.${body.ends_at},ends_at.gte.${body.starts_at})`);

if (overlappingTimeOff && overlappingTimeOff.length > 0) {
  console.warn('Overlapping time-off detected:', overlappingTimeOff);
}

// Check for overlapping shifts (warning)
const { data: overlappingShifts } = await supabase
  .from('technician_shifts')
  .select('id, starts_at, ends_at')
  .eq('org_id', auth.orgId)
  .eq('technician_id', technicianId)
  .eq('is_active', true)
  .or(`and(starts_at.lte.${body.ends_at},ends_at.gte.${body.starts_at})`);

if (overlappingShifts && overlappingShifts.length > 0) {
  console.warn('Time-off overlaps with active shifts:', overlappingShifts);
}
```

**Error Responses**:

- `400 Bad Request`: Missing required fields, invalid timestamps, ends_at before starts_at
- `403 Forbidden`: User not authorized or technician trying to create time-off for another technician
- `404 Not Found`: Technician not found

**Implementation**: Create time-off record with validation

#### 4.5.2 GET /dispatch/technicians/:id/time-off

**Purpose**: List time-off records for a technician.

**Authorization**: `admin`, `dispatcher`, `csr`, `technician` (technicians can only read own time-off)

**Query Parameters**:

```typescript
interface ListTimeOffQuery {
  start_date?: string; // ISO 8601 date, filter time-off starting on or after
  end_date?: string; // ISO 8601 date, filter time-off ending on or before
  reason?: 'vacation' | 'sick' | 'personal' | 'other'; // filter by reason
  limit?: number; // default: 100, max: 1000
  offset?: number; // default: 0
}
```

**Response Schema**:

```typescript
interface ListTimeOffResponse {
  data: TimeOffResponse[];
  pagination: {
    total: number;
    limit: number;
    offset: number;
    has_more: boolean;
  };
}
```

**Implementation**: Similar to shifts listing with date range filtering

---

## 5. Error Handling

### 5.1 Standard Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid authentication token |
| `FORBIDDEN` | 403 | User lacks required permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `DUPLICATE_TECHNICIAN` | 409 | Technician profile already exists |
| `DUPLICATE_SKILL` | 409 | Skill already exists for technician |
| `DUPLICATE_ZONE` | 409 | Service zone already assigned to technician |
| `INVALID_LOCATION` | 422 | Invalid home_base_location_id |
| `INVALID_ZONE` | 422 | Invalid service_zone_id |
| `INVALID_TIMESTAMP` | 400 | Invalid timestamp format |
| `INVALID_DATE_RANGE` | 400 | ends_at before starts_at |
| `MISSING_FIELDS` | 400 | Missing required fields |
| `INVALID_DISPLAY_NAME` | 400 | Invalid display_name length |
| `CREATE_ERROR` | 500 | Database error during creation |
| `UPDATE_ERROR` | 500 | Database error during update |
| `DELETE_ERROR` | 500 | Database error during deletion |

### 5.2 Error Response Format

All errors follow this format:

```json
{
  "error": {
    "message": "Human-readable error message",
    "code": "ERROR_CODE",
    "details": {
      // Optional additional details
    }
  }
}
```

---

## 6. Testing Requirements

### 6.1 Unit Tests

Test each endpoint with:

1. **Authorization Tests**:
   - Valid admin/dispatcher access
   - Invalid/expired token
   - Insufficient permissions (CSR trying to create technician)
   - Technician trying to access other technician's data

2. **Validation Tests**:
   - Missing required fields
   - Invalid data types
   - Invalid enum values
   - Invalid foreign keys
   - Invalid date ranges

3. **Business Logic Tests**:
   - Unique constraint violations
   - Primary zone logic
   - Overlap detection (warnings)

### 6.2 Integration Tests

Test with:
- Real Supabase instance
- Multiple organizations
- Multiple users per role
- Cross-org isolation

### 6.3 Test Data Requirements

Create test data for:
- At least 2 organizations
- At least 2 users per role per org
- At least 2 technicians per org
- At least 2 service zones per org
- Sample shifts and time-off records

---

## 7. Implementation Checklist

### Story DISP-021: Create and Update Technician Profiles API

- [ ] **POST /dispatch/technicians**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced (admin/dispatcher only)
  - [ ] Validation implemented (user_id, display_name, foreign keys)
  - [ ] Error handling implemented
  - [ ] API documentation with examples

- [ ] **PATCH /dispatch/technicians/:id**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced
  - [ ] Validation implemented (all fields optional)
  - [ ] Error handling implemented
  - [ ] API documentation with examples

- [ ] **GET /dispatch/technicians**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced (admin/dispatcher/csr/technician)
  - [ ] Query parameters implemented (filtering, pagination)
  - [ ] API documentation with examples

- [ ] **GET /dispatch/technicians/:id**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced (technicians can only read own)
  - [ ] Error handling implemented
  - [ ] API documentation with examples

### Story DISP-022: Manage Technician Skills API

- [ ] **POST /dispatch/technicians/:id/skills**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced
  - [ ] Unique constraint handling
  - [ ] Error handling implemented
  - [ ] API documentation with examples

- [ ] **PATCH /dispatch/technicians/:id/skills/:skill_id**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced
  - [ ] Error handling implemented

- [ ] **DELETE /dispatch/technicians/:id/skills/:skill_id**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced
  - [ ] Error handling implemented

- [ ] **GET /dispatch/technicians/:id/skills**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced
  - [ ] API documentation with examples

### Story DISP-023: Manage Technician Service Zones API

- [ ] **POST /dispatch/technicians/:id/service-zones**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced
  - [ ] Primary zone logic implemented
  - [ ] Unique constraint handling
  - [ ] Error handling implemented
  - [ ] API documentation with examples

- [ ] **DELETE /dispatch/technicians/:id/service-zones/:zone_id**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced
  - [ ] Error handling implemented

- [ ] **GET /dispatch/technicians/:id/service-zones**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced
  - [ ] API documentation with examples

### Story DISP-024: Create and List Shifts API

- [ ] **POST /dispatch/technicians/:id/shifts**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced (admin/dispatcher only)
  - [ ] Validation implemented (timestamps, date range)
  - [ ] Overlap detection (optional warning)
  - [ ] Error handling implemented
  - [ ] API documentation with examples

- [ ] **GET /dispatch/technicians/:id/shifts**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced
  - [ ] Query parameters implemented (date range, filtering, pagination)
  - [ ] API documentation with examples

### Story DISP-025: Create and List Time Off API

- [ ] **POST /dispatch/technicians/:id/time-off**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced (admin/dispatcher/technician)
  - [ ] Technician self-scope validation
  - [ ] Validation implemented (timestamps, date range)
  - [ ] Overlap detection (optional warning)
  - [ ] Error handling implemented
  - [ ] API documentation with examples

- [ ] **GET /dispatch/technicians/:id/time-off**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced
  - [ ] Query parameters implemented (date range, filtering, pagination)
  - [ ] API documentation with examples

---

## 8. Deployment

### 8.1 Edge Function Structure

```
supabase/functions/
  dispatch-technicians/
    index.ts
    _shared/
      auth.ts
      validation.ts
      errors.ts
```

### 8.2 Deployment Commands

```bash
# Deploy Edge Function
supabase functions deploy dispatch-technicians

# Test locally
supabase functions serve dispatch-technicians
```

### 8.3 Environment Variables

No additional environment variables required (uses Supabase defaults).

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 4 – Technician Configuration & Availability APIs. All APIs are designed as Supabase Edge Functions with complete request/response schemas, validation rules, error handling, and authorization patterns.

**Next Steps**: After completing Epic 4, proceed to Epic 5 (Dispatch Job Lifecycle APIs) which will implement job creation, assignment, and rescheduling endpoints.

