# Technical Design Document – Epic 5: Dispatch Job Lifecycle APIs (Create, Assign, Reschedule, Cancel)

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 5 – Dispatch Job Lifecycle APIs (Create, Assign, Reschedule, Cancel)
- **Source**: Derived from `fdd_2_agile.md` Epic 5 (Stories DISP-026 through DISP-032)
- **Reference Documents**: 
  - `fdd_2.md` (Functional Design Document for Scheduling & Dispatch)
  - `fdd_2_agile.md` (Agile User Stories)
  - `tdd_2_epic_1.md` (Dispatch Epic 1 for tenancy/role conventions)
  - `tdd_2_epic_2.md` (Dispatch Epic 2 for table schemas)
  - `tdd_2_epic_3.md` (Dispatch Epic 3 for RLS policies)
  - `tdd_2_epic_4.md` (Dispatch Epic 4 for technician APIs)
- **Target Platform**: Supabase (PostgreSQL 15+, Edge Functions)
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing Dispatch Job Lifecycle APIs as Supabase Edge Functions. It covers:

- Job creation with scheduling constraints
- Job listing and filtering
- Job details retrieval
- Manual job assignment to technicians
- Assignment rescheduling and reassignment
- Assignment cancellation/removal
- Job status updates

All APIs are implemented as Supabase Edge Functions (Deno/TypeScript) that enforce authorization via RLS policies (from Epic 3) and provide comprehensive validation, error handling, and business logic for job lifecycle management.

This epic assumes Epic 1 (tenancy/roles), Epic 2 (tables), Epic 3 (RLS policies), and Epic 4 (technician APIs) are complete.

---

## 2. Prerequisites

### 2.1 Required Components

Before implementing Epic 5, ensure:

1. **Epic 1-4 Complete**: All previous epics are implemented
2. **Required Tables**: All dispatch tables exist:
   - `dispatch_jobs`
   - `job_time_windows`
   - `job_assignments`
   - `technician_profiles`
   - `technician_shifts`
   - `technician_time_off`
   - `service_zones`
   - `job_notifications`
   - `calendar_events`

3. **Required Tables from CRM**:
   - `customers`
   - `customer_locations`

### 2.2 Helper Functions

From Epic 1:
- `get_user_org_id()` - Returns authenticated user's org_id
- `get_user_role()` - Returns authenticated user's role

---

## 3. Common Patterns

### 3.1 Authorization Helper

Reuse authorization pattern from Epic 4:

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

### 3.2 Error and Success Response Helpers

Reuse from Epic 4:

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

### 3.3 Job Status Update Helper

Helper function to update job status based on assignments:

```typescript
async function updateJobStatusFromAssignments(
  supabase: SupabaseClient,
  jobId: string,
  orgId: string
): Promise<void> {
  // Get all active assignments for the job
  const { data: assignments } = await supabase
    .from('job_assignments')
    .select('status')
    .eq('dispatch_job_id', jobId)
    .eq('org_id', orgId)
    .in('status', ['assigned', 'accepted', 'en_route', 'on_site']);

  // Get job current status
  const { data: job } = await supabase
    .from('dispatch_jobs')
    .select('status')
    .eq('id', jobId)
    .eq('org_id', orgId)
    .single();

  if (!job) return;

  let newStatus: string;

  if (!assignments || assignments.length === 0) {
    // No active assignments -> unscheduled
    newStatus = 'unscheduled';
  } else {
    // Check if any assignment is in progress
    const inProgress = assignments.some(a => 
      ['en_route', 'on_site'].includes(a.status)
    );
    
    if (inProgress) {
      newStatus = 'in_progress';
    } else {
      // Has assignments but not in progress -> scheduled
      newStatus = 'scheduled';
    }
  }

  // Update if status changed
  if (newStatus !== job.status) {
    await supabase
      .from('dispatch_jobs')
      .update({ status: newStatus })
      .eq('id', jobId)
      .eq('org_id', orgId);
  }
}
```

---

## 4. API Endpoints

### 4.1 Story DISP-026: Create Dispatch Job API

#### 4.1.1 POST /dispatch/jobs

**Purpose**: Create a new dispatch job with scheduling constraints.

**Authorization**: `admin`, `dispatcher`, `csr`

**Request Schema**:

```typescript
interface CreateDispatchJobRequest {
  customer_id: string; // UUID, required, must exist in customers
  location_id: string; // UUID, required, must exist in customer_locations
  title: string; // required, min 1 char, max 255 chars
  description?: string; // optional
  job_type?: string; // optional, e.g., 'maintenance', 'install', 'repair', 'inspection'
  priority?: 'low' | 'normal' | 'high' | 'emergency'; // default: 'normal'
  estimated_duration_minutes?: number; // default: 60, must be > 0
  required_skills?: string[]; // optional array of skill codes
  required_crew_size?: number; // default: 1, must be > 0
  service_zone_id?: string; // UUID, optional, must exist in service_zones
  sla_start_at?: string; // ISO 8601 timestamp, optional
  sla_end_at?: string; // ISO 8601 timestamp, optional, must be after sla_start_at
  is_customer_booked?: boolean; // default: false
  notes_internal?: string; // optional
  related_work_order_id?: string; // UUID, optional (deferred FK)
  time_windows?: Array<{ // optional, creates job_time_windows
    window_start: string; // ISO 8601 timestamp
    window_end: string; // ISO 8601 timestamp
    source?: 'system_suggested' | 'dispatcher_selected' | 'customer_selected'; // default: 'dispatcher_selected'
    is_selected?: boolean; // default: false
  }>;
}
```

**Request Example (Dispatcher-Created Job)**:

```json
{
  "customer_id": "111e4567-e89b-12d3-a456-426614174000",
  "location_id": "222e4567-e89b-12d3-a456-426614174000",
  "title": "AC Maintenance",
  "description": "Routine AC maintenance check",
  "job_type": "maintenance",
  "priority": "normal",
  "estimated_duration_minutes": 60,
  "required_skills": ["hvac_service"],
  "required_crew_size": 1,
  "service_zone_id": "333e4567-e89b-12d3-a456-426614174000",
  "sla_start_at": "2024-01-20T08:00:00Z",
  "sla_end_at": "2024-01-20T17:00:00Z",
  "is_customer_booked": false,
  "time_windows": [
    {
      "window_start": "2024-01-20T09:00:00Z",
      "window_end": "2024-01-20T11:00:00Z",
      "source": "dispatcher_selected",
      "is_selected": true
    },
    {
      "window_start": "2024-01-20T13:00:00Z",
      "window_end": "2024-01-20T15:00:00Z",
      "source": "dispatcher_selected",
      "is_selected": false
    }
  ]
}
```

**Request Example (Customer-Booked Job)**:

```json
{
  "customer_id": "111e4567-e89b-12d3-a456-426614174000",
  "location_id": "222e4567-e89b-12d3-a456-426614174000",
  "title": "AC Repair",
  "description": "AC not cooling",
  "job_type": "repair",
  "priority": "normal",
  "estimated_duration_minutes": 120,
  "is_customer_booked": true,
  "time_windows": [
    {
      "window_start": "2024-01-22T10:00:00Z",
      "window_end": "2024-01-22T12:00:00Z",
      "source": "customer_selected",
      "is_selected": true
    }
  ]
}
```

**Response Schema**:

```typescript
interface DispatchJobResponse {
  id: string;
  org_id: string;
  customer_id: string;
  location_id: string;
  title: string;
  description: string | null;
  job_type: string | null;
  priority: string;
  status: string;
  estimated_duration_minutes: number;
  required_skills: string[] | null;
  required_crew_size: number;
  service_zone_id: string | null;
  sla_start_at: string | null;
  sla_end_at: string | null;
  is_customer_booked: boolean;
  notes_internal: string | null;
  related_work_order_id: string | null;
  created_by_user_id: string | null;
  created_at: string;
  updated_at: string;
  time_windows?: JobTimeWindowResponse[]; // included if created
}
```

**Response Example**:

```json
{
  "data": {
    "id": "444e4567-e89b-12d3-a456-426614174000",
    "org_id": "00000000-0000-0000-0000-000000000001",
    "customer_id": "111e4567-e89b-12d3-a456-426614174000",
    "location_id": "222e4567-e89b-12d3-a456-426614174000",
    "title": "AC Maintenance",
    "description": "Routine AC maintenance check",
    "job_type": "maintenance",
    "priority": "normal",
    "status": "unscheduled",
    "estimated_duration_minutes": 60,
    "required_skills": ["hvac_service"],
    "required_crew_size": 1,
    "service_zone_id": "333e4567-e89b-12d3-a456-426614174000",
    "sla_start_at": "2024-01-20T08:00:00Z",
    "sla_end_at": "2024-01-20T17:00:00Z",
    "is_customer_booked": false,
    "notes_internal": null,
    "related_work_order_id": null,
    "created_by_user_id": "555e4567-e89b-12d3-a456-426614174000",
    "created_at": "2024-01-15T10:30:00Z",
    "updated_at": "2024-01-15T10:30:00Z",
    "time_windows": [
      {
        "id": "666e4567-e89b-12d3-a456-426614174000",
        "window_start": "2024-01-20T09:00:00Z",
        "window_end": "2024-01-20T11:00:00Z",
        "source": "dispatcher_selected",
        "is_selected": true
      }
    ]
  }
}
```

**Validation Rules**:

1. `customer_id` must exist in `customers` and belong to the same org
2. `location_id` must exist in `customer_locations` and belong to the same org and customer
3. `title` must be 1-255 characters
4. `estimated_duration_minutes` must be > 0 if provided
5. `required_crew_size` must be > 0 if provided
6. `service_zone_id` must exist in `service_zones` and belong to the same org if provided
7. `sla_end_at` must be after `sla_start_at` if both provided
8. `time_windows` must have valid `window_end > window_start` for each window
9. Only one `time_window` can have `is_selected = true` (enforced in application logic)

**Business Logic**:

1. **Job Creation**:
   - Insert into `dispatch_jobs` with `status = 'unscheduled'`
   - Set `created_by_user_id` from authenticated user

2. **Time Windows Creation**:
   - If `time_windows` provided, create `job_time_windows` records
   - If multiple windows, ensure only one has `is_selected = true`
   - If no windows provided but `sla_start_at` and `sla_end_at` exist, optionally create a system-suggested window (deferred to Epic 7)

3. **Status Initialization**:
   - Job starts as `unscheduled` until assignments are created

**Error Responses**:

- `400 Bad Request`: Missing required fields, invalid data
- `403 Forbidden`: User not authorized
- `404 Not Found`: Customer or location not found
- `422 Unprocessable Entity`: Invalid foreign keys, invalid date ranges

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

  const auth = await authorizeUser(supabase, user.id, ['admin', 'dispatcher', 'csr']);
  if (!auth) {
    return errorResponse('Forbidden', 403, 'INSUFFICIENT_PERMISSIONS');
  }

  const body = await req.json();
  
  // Validate required fields
  const validation = validateRequiredFields(body, ['customer_id', 'location_id', 'title']);
  if (!validation.valid) {
    return errorResponse(
      `Missing required fields: ${validation.missing.join(', ')}`,
      400,
      'MISSING_FIELDS',
      { missing: validation.missing }
    );
  }

  // Validate customer exists and belongs to org
  const { data: customer, error: customerError } = await supabase
    .from('customers')
    .select('id, org_id')
    .eq('id', body.customer_id)
    .eq('org_id', auth.orgId)
    .single();
  
  if (customerError || !customer) {
    return errorResponse('Customer not found', 404, 'CUSTOMER_NOT_FOUND');
  }

  // Validate location exists, belongs to org, and belongs to customer
  const { data: location, error: locationError } = await supabase
    .from('customer_locations')
    .select('id, org_id, customer_id')
    .eq('id', body.location_id)
    .eq('org_id', auth.orgId)
    .eq('customer_id', body.customer_id)
    .single();
  
  if (locationError || !location) {
    return errorResponse('Location not found or does not belong to customer', 404, 'LOCATION_NOT_FOUND');
  }

  // Validate title length
  if (body.title.length < 1 || body.title.length > 255) {
    return errorResponse('title must be 1-255 characters', 400, 'INVALID_TITLE');
  }

  // Validate estimated_duration_minutes
  if (body.estimated_duration_minutes !== undefined && body.estimated_duration_minutes <= 0) {
    return errorResponse('estimated_duration_minutes must be > 0', 400, 'INVALID_DURATION');
  }

  // Validate required_crew_size
  if (body.required_crew_size !== undefined && body.required_crew_size <= 0) {
    return errorResponse('required_crew_size must be > 0', 400, 'INVALID_CREW_SIZE');
  }

  // Validate SLA window
  if (body.sla_start_at && body.sla_end_at) {
    const slaStart = new Date(body.sla_start_at);
    const slaEnd = new Date(body.sla_end_at);
    if (slaEnd <= slaStart) {
      return errorResponse('sla_end_at must be after sla_start_at', 400, 'INVALID_SLA_WINDOW');
    }
  }

  // Validate service_zone_id if provided
  if (body.service_zone_id) {
    const { data: zone, error: zoneError } = await supabase
      .from('service_zones')
      .select('id, org_id')
      .eq('id', body.service_zone_id)
      .eq('org_id', auth.orgId)
      .single();
    
    if (zoneError || !zone) {
      return errorResponse('Invalid service_zone_id', 422, 'INVALID_ZONE');
    }
  }

  // Validate time windows if provided
  if (body.time_windows && Array.isArray(body.time_windows)) {
    let selectedCount = 0;
    for (const window of body.time_windows) {
      const windowStart = new Date(window.window_start);
      const windowEnd = new Date(window.window_end);
      if (windowEnd <= windowStart) {
        return errorResponse('window_end must be after window_start', 400, 'INVALID_WINDOW');
      }
      if (window.is_selected) {
        selectedCount++;
      }
    }
    if (selectedCount > 1) {
      return errorResponse('Only one time window can be selected', 400, 'MULTIPLE_SELECTED_WINDOWS');
    }
  }

  // Create job
  const { data: job, error: createError } = await supabase
    .from('dispatch_jobs')
    .insert({
      org_id: auth.orgId,
      customer_id: body.customer_id,
      location_id: body.location_id,
      title: body.title,
      description: body.description || null,
      job_type: body.job_type || null,
      priority: body.priority || 'normal',
      status: 'unscheduled',
      estimated_duration_minutes: body.estimated_duration_minutes || 60,
      required_skills: body.required_skills || null,
      required_crew_size: body.required_crew_size || 1,
      service_zone_id: body.service_zone_id || null,
      sla_start_at: body.sla_start_at || null,
      sla_end_at: body.sla_end_at || null,
      is_customer_booked: body.is_customer_booked || false,
      notes_internal: body.notes_internal || null,
      related_work_order_id: body.related_work_order_id || null,
      created_by_user_id: user.id
    })
    .select()
    .single();

  if (createError) {
    return errorResponse('Failed to create job', 500, 'CREATE_ERROR', { error: createError.message });
  }

  // Create time windows if provided
  let timeWindows = [];
  if (body.time_windows && Array.isArray(body.time_windows)) {
    const windowsToInsert = body.time_windows.map((w: any) => ({
      org_id: auth.orgId,
      dispatch_job_id: job.id,
      window_start: w.window_start,
      window_end: w.window_end,
      source: w.source || 'dispatcher_selected',
      is_selected: w.is_selected || false
    }));

    const { data: createdWindows, error: windowsError } = await supabase
      .from('job_time_windows')
      .insert(windowsToInsert)
      .select();

    if (windowsError) {
      // Log error but don't fail job creation
      console.error('Failed to create time windows:', windowsError);
    } else {
      timeWindows = createdWindows || [];
    }
  }

  return successResponse({ ...job, time_windows: timeWindows }, 201);
});
```

---

### 4.2 Story DISP-027: List and Filter Dispatch Jobs API

#### 4.2.1 GET /dispatch/jobs

**Purpose**: List and filter dispatch jobs for schedule board.

**Authorization**: `admin`, `dispatcher`, `csr`, `technician` (technicians see assigned jobs only)

**Query Parameters**:

```typescript
interface ListDispatchJobsQuery {
  status?: 'unscheduled' | 'scheduled' | 'dispatched' | 'in_progress' | 'completed' | 'canceled';
  priority?: 'low' | 'normal' | 'high' | 'emergency';
  unscheduled_only?: boolean; // filter for unscheduled jobs
  sla_start_after?: string; // ISO 8601 date, filter jobs with SLA starting on or after
  sla_end_before?: string; // ISO 8601 date, filter jobs with SLA ending on or before
  customer_id?: string; // UUID, filter by customer
  service_zone_id?: string; // UUID, filter by service zone
  date?: string; // ISO 8601 date, filter jobs scheduled for this date (via assignments)
  limit?: number; // default: 100, max: 1000
  offset?: number; // default: 0
}
```

**Response Schema**:

```typescript
interface ListDispatchJobsResponse {
  data: Array<{
    id: string;
    title: string;
    customer_id: string;
    location_id: string;
    priority: string;
    status: string;
    estimated_duration_minutes: number;
    sla_start_at: string | null;
    sla_end_at: string | null;
    created_at: string;
    // Location details (joined)
    location?: {
      address_line1: string;
      city: string;
      state: string;
      postal_code: string;
    };
    // Customer details (joined)
    customer?: {
      name: string;
      phone: string;
    };
  }>;
  pagination: {
    total: number;
    limit: number;
    offset: number;
    has_more: boolean;
  };
}
```

**Response Example**:

```json
{
  "data": [
    {
      "id": "444e4567-e89b-12d3-a456-426614174000",
      "title": "AC Maintenance",
      "customer_id": "111e4567-e89b-12d3-a456-426614174000",
      "location_id": "222e4567-e89b-12d3-a456-426614174000",
      "priority": "normal",
      "status": "scheduled",
      "estimated_duration_minutes": 60,
      "sla_start_at": "2024-01-20T08:00:00Z",
      "sla_end_at": "2024-01-20T17:00:00Z",
      "created_at": "2024-01-15T10:30:00Z",
      "location": {
        "address_line1": "123 Main St",
        "city": "Springfield",
        "state": "IL",
        "postal_code": "62701"
      },
      "customer": {
        "name": "John Doe",
        "phone": "+15551234567"
      }
    }
  ],
  "pagination": {
    "total": 1,
    "limit": 100,
    "offset": 0,
    "has_more": false
  }
}
```

**Implementation**: Uses Supabase `.select()` with joins and filtering, RLS handles org_id filtering automatically.

**Performance Target**: < 500ms for typical org sizes (dozens of technicians, hundreds of daily jobs) per `fdd_2.md` §8.

---

### 4.3 Story DISP-028: View Dispatch Job Details API

#### 4.3.1 GET /dispatch/jobs/:id

**Purpose**: Get detailed job information including time windows and assignments.

**Authorization**: `admin`, `dispatcher`, `csr`, `technician` (technicians see assigned jobs only)

**Response Schema**:

```typescript
interface DispatchJobDetailsResponse {
  id: string;
  org_id: string;
  customer_id: string;
  location_id: string;
  title: string;
  description: string | null;
  job_type: string | null;
  priority: string;
  status: string;
  estimated_duration_minutes: number;
  required_skills: string[] | null;
  required_crew_size: number;
  service_zone_id: string | null;
  sla_start_at: string | null;
  sla_end_at: string | null;
  is_customer_booked: boolean;
  notes_internal: string | null;
  related_work_order_id: string | null;
  created_by_user_id: string | null;
  created_at: string;
  updated_at: string;
  // Related data
  time_windows: Array<{
    id: string;
    window_start: string;
    window_end: string;
    source: string;
    is_selected: boolean;
    created_at: string;
  }>;
  assignments: Array<{
    id: string;
    technician_id: string;
    technician_name: string; // from technician_profiles.display_name
    scheduled_start_at: string;
    scheduled_end_at: string;
    arrival_window_start: string | null;
    arrival_window_end: string | null;
    status: string;
    tech_eta_at: string | null;
    sequence_in_route: number | null;
    is_primary_technician: boolean;
    created_at: string;
  }>;
  customer: {
    id: string;
    name: string;
    phone: string;
    email: string | null;
  };
  location: {
    id: string;
    address_line1: string;
    address_line2: string | null;
    city: string;
    state: string;
    postal_code: string;
    latitude: number | null;
    longitude: number | null;
  };
}
```

**Implementation**: Uses Supabase `.select()` with joins to fetch related data in one query.

**Error Responses**:

- `403 Forbidden`: User not authorized or technician trying to access unassigned job
- `404 Not Found`: Job not found

---

### 4.4 Story DISP-029: Manual Assign Job to Technician API

#### 4.4.1 POST /dispatch/jobs/:id/assign

**Purpose**: Manually assign a job to a technician at a specific time.

**Authorization**: `admin`, `dispatcher`

**Request Schema**:

```typescript
interface AssignJobRequest {
  technician_id: string; // UUID, required, must exist in technician_profiles
  scheduled_start_at: string; // ISO 8601 timestamp, required
  scheduled_end_at: string; // ISO 8601 timestamp, required, must be after scheduled_start_at
  arrival_window_start?: string; // ISO 8601 timestamp, optional
  arrival_window_end?: string; // ISO 8601 timestamp, optional, must be after arrival_window_start
  is_primary_technician?: boolean; // default: true
  notes?: string; // optional
}
```

**Request Example**:

```json
{
  "technician_id": "777e4567-e89b-12d3-a456-426614174000",
  "scheduled_start_at": "2024-01-20T09:00:00Z",
  "scheduled_end_at": "2024-01-20T10:00:00Z",
  "arrival_window_start": "2024-01-20T09:00:00Z",
  "arrival_window_end": "2024-01-20T09:30:00Z",
  "is_primary_technician": true,
  "notes": "First visit"
}
```

**Response Schema**:

```typescript
interface AssignmentResponse {
  id: string;
  org_id: string;
  dispatch_job_id: string;
  technician_id: string;
  scheduled_start_at: string;
  scheduled_end_at: string;
  arrival_window_start: string | null;
  arrival_window_end: string | null;
  status: string; // 'assigned'
  tech_eta_at: string | null;
  sequence_in_route: number | null;
  is_primary_technician: boolean;
  notes: string | null;
  created_at: string;
  updated_at: string;
  warnings?: string[]; // optional warnings (overlaps, capacity, etc.)
}
```

**Validation Rules**:

1. **Availability Check**:
   - Technician must have an active shift covering `scheduled_start_at` to `scheduled_end_at`
   - No time-off conflicts during the assignment period

2. **Overlap Check**:
   - No overlapping assignments for the same technician (unless `max_concurrent_jobs > 1`)

3. **Capacity Check**:
   - Check `max_daily_work_minutes` if set
   - Check `max_concurrent_jobs` if set

4. **SLA/Time Window Check**:
   - `scheduled_start_at` should be within SLA window if SLA exists
   - `scheduled_start_at` should be within selected time window if one exists

5. **Job Status Update**:
   - Update `dispatch_jobs.status` to `'scheduled'` after assignment creation

**Validation Implementation**:

```typescript
async function validateAssignment(
  supabase: SupabaseClient,
  orgId: string,
  technicianId: string,
  scheduledStart: Date,
  scheduledEnd: Date,
  jobId: string
): Promise<{ valid: boolean; warnings: string[]; errors: string[] }> {
  const warnings: string[] = [];
  const errors: string[] = [];

  // Check technician exists
  const { data: technician } = await supabase
    .from('technician_profiles')
    .select('id, max_daily_work_minutes, max_concurrent_jobs')
    .eq('id', technicianId)
    .eq('org_id', orgId)
    .single();

  if (!technician) {
    errors.push('Technician not found');
    return { valid: false, warnings, errors };
  }

  // Check shift availability
  const { data: shifts } = await supabase
    .from('technician_shifts')
    .select('starts_at, ends_at')
    .eq('org_id', orgId)
    .eq('technician_id', technicianId)
    .eq('is_active', true)
    .lte('starts_at', scheduledEnd.toISOString())
    .gte('ends_at', scheduledStart.toISOString());

  if (!shifts || shifts.length === 0) {
    errors.push('Technician has no active shift covering the assignment time');
  }

  // Check time-off conflicts
  const { data: timeOff } = await supabase
    .from('technician_time_off')
    .select('id, starts_at, ends_at')
    .eq('org_id', orgId)
    .eq('technician_id', technicianId)
    .lte('starts_at', scheduledEnd.toISOString())
    .gte('ends_at', scheduledStart.toISOString());

  if (timeOff && timeOff.length > 0) {
    errors.push('Assignment conflicts with technician time-off');
  }

  // Check overlapping assignments
  const { data: overlappingAssignments } = await supabase
    .from('job_assignments')
    .select('id, scheduled_start_at, scheduled_end_at')
    .eq('org_id', orgId)
    .eq('technician_id', technicianId)
    .in('status', ['assigned', 'accepted', 'en_route', 'on_site'])
    .neq('dispatch_job_id', jobId)
    .or(`and(scheduled_start_at.lte.${scheduledEnd.toISOString()},scheduled_end_at.gte.${scheduledStart.toISOString()})`);

  if (overlappingAssignments && overlappingAssignments.length > 0) {
    if (technician.max_concurrent_jobs === 1) {
      errors.push('Assignment overlaps with existing assignments');
    } else {
      warnings.push(`Assignment overlaps with ${overlappingAssignments.length} existing assignment(s)`);
    }
  }

  // Check daily capacity
  if (technician.max_daily_work_minutes) {
    const assignmentDuration = (scheduledEnd.getTime() - scheduledStart.getTime()) / (1000 * 60);
    
    // Get all assignments for the day
    const dayStart = new Date(scheduledStart);
    dayStart.setHours(0, 0, 0, 0);
    const dayEnd = new Date(scheduledStart);
    dayEnd.setHours(23, 59, 59, 999);

    const { data: dayAssignments } = await supabase
      .from('job_assignments')
      .select('scheduled_start_at, scheduled_end_at')
      .eq('org_id', orgId)
      .eq('technician_id', technicianId)
      .in('status', ['assigned', 'accepted', 'en_route', 'on_site'])
      .gte('scheduled_start_at', dayStart.toISOString())
      .lte('scheduled_start_at', dayEnd.toISOString());

    let totalMinutes = assignmentDuration;
    if (dayAssignments) {
      for (const assignment of dayAssignments) {
        const start = new Date(assignment.scheduled_start_at);
        const end = new Date(assignment.scheduled_end_at);
        totalMinutes += (end.getTime() - start.getTime()) / (1000 * 60);
      }
    }

    if (totalMinutes > technician.max_daily_work_minutes) {
      warnings.push(`Assignment exceeds daily capacity (${totalMinutes} > ${technician.max_daily_work_minutes} minutes)`);
    }
  }

  // Check SLA window
  const { data: job } = await supabase
    .from('dispatch_jobs')
    .select('sla_start_at, sla_end_at')
    .eq('id', jobId)
    .eq('org_id', orgId)
    .single();

  if (job && job.sla_start_at && job.sla_end_at) {
    const slaStart = new Date(job.sla_start_at);
    const slaEnd = new Date(job.sla_end_at);
    
    if (scheduledStart < slaStart || scheduledEnd > slaEnd) {
      warnings.push('Assignment is outside SLA window');
    }
  }

  return {
    valid: errors.length === 0,
    warnings,
    errors
  };
}
```

**Business Logic**:

1. **Create Assignment**:
   - Insert into `job_assignments` with `status = 'assigned'`
   - Set `assigned_by_user_id` from authenticated user

2. **Update Job Status**:
   - Call `updateJobStatusFromAssignments()` to update job status to `'scheduled'`

3. **Calendar Sync** (deferred to Epic 9):
   - Flag assignment for calendar sync

4. **Notifications** (deferred to Epic 10):
   - Create notification records for assignment confirmation

**Error Responses**:

- `400 Bad Request`: Missing required fields, invalid data, validation failures
- `403 Forbidden`: User not authorized
- `404 Not Found`: Job or technician not found
- `409 Conflict`: Assignment conflicts (overlaps, availability)

**Implementation**: Create assignment with validation, return warnings if any.

---

### 4.5 Story DISP-030: Update Assignment (Reschedule/Reassign) API

#### 4.5.1 PATCH /dispatch/assignments/:id

**Purpose**: Reschedule or reassign an existing assignment.

**Authorization**: `admin`, `dispatcher`

**Request Schema**:

```typescript
interface UpdateAssignmentRequest {
  technician_id?: string; // UUID, reassign to different technician
  scheduled_start_at?: string; // ISO 8601 timestamp, reschedule start time
  scheduled_end_at?: string; // ISO 8601 timestamp, reschedule end time
  arrival_window_start?: string; // ISO 8601 timestamp
  arrival_window_end?: string; // ISO 8601 timestamp
  status?: 'assigned' | 'accepted' | 'declined' | 'en_route' | 'on_site' | 'completed' | 'no_show' | 'canceled';
  notes?: string;
  is_locked?: boolean; // prevent optimizer from changing this assignment
}
```

**Request Example (Reschedule)**:

```json
{
  "scheduled_start_at": "2024-01-20T10:00:00Z",
  "scheduled_end_at": "2024-01-20T11:00:00Z",
  "arrival_window_start": "2024-01-20T10:00:00Z",
  "arrival_window_end": "2024-01-20T10:30:00Z"
}
```

**Request Example (Reassign)**:

```json
{
  "technician_id": "888e4567-e89b-12d3-a456-426614174000"
}
```

**Response Schema**: Same as assignment response

**Validation Rules**:

1. Same validation as assignment creation (availability, overlaps, capacity)
2. If reassigning, validate new technician availability
3. If rescheduling, validate new time availability

**Business Logic**:

1. **Update Assignment**:
   - Update `job_assignments` record
   - If `is_locked` is set, store in metadata or separate field (implementation detail)

2. **Update Job Status**:
   - Call `updateJobStatusFromAssignments()` to update job status

3. **Calendar Sync** (deferred to Epic 9):
   - Update `calendar_events` status to `'updated'` for sync

4. **Notifications** (deferred to Epic 10):
   - Create reschedule notification if time changed significantly

**Error Responses**: Same as assignment creation

---

### 4.6 Story DISP-031: Cancel/Remove Assignment API

#### 4.6.1 DELETE /dispatch/assignments/:id

**Purpose**: Cancel or remove an assignment.

**Authorization**: `admin`, `dispatcher`

**Query Parameters**:

```typescript
interface DeleteAssignmentQuery {
  hard_delete?: boolean; // default: false (soft delete via status)
}
```

**Business Logic**:

**Soft Delete (Default)**:
- Update assignment `status` to `'canceled'`
- Retain assignment record for history
- Update job status via `updateJobStatusFromAssignments()`
- Update calendar events to `'canceled'` status
- Create cancellation notifications

**Hard Delete**:
- Delete assignment record (cascade handled by FK)
- Update job status
- Delete calendar events (or mark as canceled)
- Create cancellation notifications

**Response**: `204 No Content` on success

**Error Responses**:

- `403 Forbidden`: User not authorized
- `404 Not Found`: Assignment not found

---

### 4.7 Story DISP-032: Update Dispatch Job Status API

#### 4.7.1 PATCH /dispatch/jobs/:id

**Purpose**: Update job status and other non-assignment fields.

**Authorization**: `admin`, `dispatcher`, `csr` (limited fields)

**Request Schema**:

```typescript
interface UpdateDispatchJobRequest {
  status?: 'unscheduled' | 'scheduled' | 'dispatched' | 'in_progress' | 'completed' | 'canceled';
  priority?: 'low' | 'normal' | 'high' | 'emergency';
  title?: string;
  description?: string;
  notes_internal?: string;
  // CSR can only update: title, description, notes_internal
}
```

**Status Transition Rules**:

| From Status | To Status | Allowed | Notes |
|-------------|-----------|---------|-------|
| `unscheduled` | `scheduled` | ✅ | Via assignment creation (automatic) |
| `unscheduled` | `canceled` | ✅ | Manual cancellation |
| `scheduled` | `in_progress` | ✅ | Via assignment status update (automatic) |
| `scheduled` | `canceled` | ✅ | Manual cancellation |
| `scheduled` | `unscheduled` | ✅ | All assignments removed |
| `in_progress` | `completed` | ✅ | Via assignment completion (automatic) |
| `in_progress` | `canceled` | ✅ | Manual cancellation |
| `completed` | `canceled` | ❌ | Cannot cancel completed job |
| `canceled` | Any | ❌ | Cannot change canceled job |

**Validation**:

1. Status transitions must follow state machine rules
2. Cannot mark `completed` without completed assignments (unless explicitly allowed)
3. CSR can only update non-critical fields

**Error Responses**:

- `400 Bad Request`: Invalid status transition
- `403 Forbidden`: User not authorized or CSR trying to update restricted fields
- `404 Not Found`: Job not found

---

## 5. Error Handling

### 5.1 Standard Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid authentication token |
| `FORBIDDEN` | 403 | User lacks required permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `INVALID_STATUS_TRANSITION` | 400 | Invalid job status transition |
| `ASSIGNMENT_CONFLICT` | 409 | Assignment conflicts with existing assignments |
| `TECHNICIAN_UNAVAILABLE` | 409 | Technician not available for assignment time |
| `CAPACITY_EXCEEDED` | 409 | Assignment exceeds technician capacity |
| `INVALID_DATE_RANGE` | 400 | Invalid date range (end before start) |
| `MISSING_FIELDS` | 400 | Missing required fields |

---

## 6. Testing Requirements

### 6.1 Unit Tests

Test each endpoint with:

1. **Authorization Tests**: Valid/invalid roles
2. **Validation Tests**: Missing fields, invalid data, invalid transitions
3. **Business Logic Tests**: Status updates, assignment conflicts, capacity checks

### 6.2 Integration Tests

Test with real Supabase instance, multiple orgs, and complex scenarios.

---

## 7. Implementation Checklist

### Story DISP-026: Create Dispatch Job API
- [ ] POST /dispatch/jobs endpoint implemented
- [ ] Validation for all fields
- [ ] Time windows creation logic
- [ ] Error handling
- [ ] API documentation with examples

### Story DISP-027: List and Filter Dispatch Jobs API
- [ ] GET /dispatch/jobs endpoint implemented
- [ ] Query parameters (status, priority, date, etc.)
- [ ] Pagination support
- [ ] Performance optimization
- [ ] API documentation

### Story DISP-028: View Dispatch Job Details API
- [ ] GET /dispatch/jobs/:id endpoint implemented
- [ ] Related data joins (time windows, assignments, customer, location)
- [ ] Authorization enforced
- [ ] API documentation

### Story DISP-029: Manual Assign Job to Technician API
- [ ] POST /dispatch/jobs/:id/assign endpoint implemented
- [ ] Availability validation
- [ ] Overlap detection
- [ ] Capacity checks
- [ ] Job status update logic
- [ ] Warnings handling
- [ ] API documentation

### Story DISP-030: Update Assignment API
- [ ] PATCH /dispatch/assignments/:id endpoint implemented
- [ ] Reschedule validation
- [ ] Reassign validation
- [ ] Calendar sync flags
- [ ] Notification triggers
- [ ] Lock mechanism documented
- [ ] API documentation

### Story DISP-031: Cancel/Remove Assignment API
- [ ] DELETE /dispatch/assignments/:id endpoint implemented
- [ ] Soft delete vs hard delete logic
- [ ] Job status update
- [ ] Calendar event cancellation
- [ ] Notification creation
- [ ] API documentation

### Story DISP-032: Update Dispatch Job Status API
- [ ] PATCH /dispatch/jobs/:id endpoint implemented
- [ ] Status transition validation
- [ ] CSR field restrictions
- [ ] Error handling
- [ ] API documentation

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 5 – Dispatch Job Lifecycle APIs. All APIs are designed as Supabase Edge Functions with complete request/response schemas, validation rules, business logic, and error handling.

**Next Steps**: After completing Epic 5, proceed to Epic 6 (Technician Mobile Hooks) which will implement technician-facing status and ETA update endpoints.

