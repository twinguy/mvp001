# Technical Design Document – Epic 6: Technician Mobile Hooks: Status & ETA Updates

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 6 – Technician Mobile Hooks: Status & ETA Updates
- **Source**: Derived from `fdd_2_agile.md` Epic 6 (Stories DISP-033 through DISP-035)
- **Reference Documents**: 
  - `fdd_2.md` (Functional Design Document for Scheduling & Dispatch)
  - `fdd_2_agile.md` (Agile User Stories)
  - `tdd_2_epic_1.md` (Dispatch Epic 1 for tenancy/role conventions)
  - `tdd_2_epic_2.md` (Dispatch Epic 2 for table schemas)
  - `tdd_2_epic_3.md` (Dispatch Epic 3 for RLS policies)
  - `tdd_2_epic_4.md` (Dispatch Epic 4 for technician APIs)
  - `tdd_2_epic_5.md` (Dispatch Epic 5 for job lifecycle APIs)
- **Target Platform**: Supabase (PostgreSQL 15+, Edge Functions, Realtime)
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing Technician Mobile Hooks APIs and real-time subscriptions. It covers:

- Assignment status update endpoint with state machine validation
- ETA update endpoint with notification triggers and deduplication
- Real-time subscription specifications for technician mobile apps

All APIs are implemented as Supabase Edge Functions (Deno/TypeScript) that enforce authorization via RLS policies (from Epic 3) and provide real-time capabilities via Supabase Realtime subscriptions.

This epic assumes Epic 1 (tenancy/roles), Epic 2 (tables), Epic 3 (RLS policies), Epic 4 (technician APIs), and Epic 5 (job lifecycle APIs) are complete.

---

## 2. Prerequisites

### 2.1 Required Components

Before implementing Epic 6, ensure:

1. **Epic 1-5 Complete**: All previous epics are implemented
2. **Required Tables**: All dispatch tables exist:
   - `job_assignments`
   - `dispatch_jobs`
   - `route_plans`
   - `route_stops`
   - `technician_profiles`
   - `job_notifications`

3. **Supabase Realtime**: Realtime is enabled on required tables (configured in Supabase dashboard)

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

### 3.2 Technician Self-Scope Validation

Helper to verify technician owns an assignment:

```typescript
async function verifyTechnicianOwnsAssignment(
  supabase: SupabaseClient,
  assignmentId: string,
  technicianUserId: string,
  orgId: string
): Promise<boolean> {
  // Get technician profile for user
  const { data: techProfile } = await supabase
    .from('technician_profiles')
    .select('id')
    .eq('user_id', technicianUserId)
    .eq('org_id', orgId)
    .single();

  if (!techProfile) {
    return false;
  }

  // Verify assignment belongs to technician
  const { data: assignment } = await supabase
    .from('job_assignments')
    .select('technician_id')
    .eq('id', assignmentId)
    .eq('org_id', orgId)
    .eq('technician_id', techProfile.id)
    .single();

  return assignment !== null;
}
```

### 3.3 Error and Success Response Helpers

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

### 3.4 Job Status Update Helper

Reuse from Epic 5:

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

### 4.1 Story DISP-033: Technician Assignment Status Update API

#### 4.1.1 PATCH /dispatch/assignments/:id/status

**Purpose**: Update assignment status (technician mobile app).

**Authorization**: `technician` (self-scoped only)

**Request Schema**:

```typescript
interface UpdateAssignmentStatusRequest {
  status: 'accepted' | 'declined' | 'en_route' | 'on_site' | 'completed' | 'no_show';
  notes?: string; // optional notes about status change
}
```

**Request Example**:

```json
{
  "status": "en_route",
  "notes": "Left depot, ETA 15 minutes"
}
```

**Response Schema**:

```typescript
interface AssignmentStatusResponse {
  id: string;
  status: string;
  updated_at: string;
  job_status_updated?: boolean; // indicates if parent job status was updated
}
```

**Response Example**:

```json
{
  "data": {
    "id": "999e4567-e89b-12d3-a456-426614174000",
    "status": "en_route",
    "updated_at": "2024-01-20T09:15:00Z",
    "job_status_updated": true
  }
}
```

#### 4.1.2 Assignment Status State Machine

**Valid Status Transitions**:

| From Status | To Status | Allowed | Notes |
|-------------|-----------|---------|-------|
| `assigned` | `accepted` | ✅ | Technician accepts assignment |
| `assigned` | `declined` | ✅ | Technician declines assignment |
| `assigned` | `canceled` | ❌ | Only dispatcher can cancel |
| `accepted` | `en_route` | ✅ | Technician starts traveling |
| `accepted` | `declined` | ❌ | Cannot decline after accepting |
| `en_route` | `on_site` | ✅ | Technician arrives at location |
| `en_route` | `completed` | ❌ | Must mark on_site first |
| `on_site` | `completed` | ✅ | Work completed |
| `on_site` | `no_show` | ❌ | Only dispatcher can mark no_show |
| `completed` | Any | ❌ | Cannot change completed status |
| `declined` | Any | ❌ | Cannot change declined status |
| `canceled` | Any | ❌ | Cannot change canceled status |
| `no_show` | Any | ❌ | Cannot change no_show status |

**State Machine Implementation**:

```typescript
const VALID_TRANSITIONS: Record<string, string[]> = {
  'assigned': ['accepted', 'declined'],
  'accepted': ['en_route'],
  'en_route': ['on_site'],
  'on_site': ['completed'],
  'completed': [],
  'declined': [],
  'canceled': [],
  'no_show': []
};

function isValidStatusTransition(
  fromStatus: string,
  toStatus: string
): boolean {
  const allowedTransitions = VALID_TRANSITIONS[fromStatus] || [];
  return allowedTransitions.includes(toStatus);
}
```

**Validation Rules**:

1. Technician must own the assignment (verified via RLS and helper function)
2. Status transition must be valid according to state machine
3. Cannot update assignments with status `completed`, `declined`, `canceled`, or `no_show`

**Business Logic**:

1. **Status Update**:
   - Update `job_assignments.status`
   - Update `job_assignments.updated_at` (automatic via trigger)

2. **Job Status Update**:
   - Call `updateJobStatusFromAssignments()` to update parent job status
   - Job status may change to `in_progress` when status becomes `en_route` or `on_site`

3. **Notifications** (deferred to Epic 10):
   - Status changes may trigger notifications (e.g., "tech on the way" when status becomes `en_route`)

**Error Responses**:

- `400 Bad Request`: Invalid status transition, missing required fields
- `403 Forbidden`: User not authorized or assignment doesn't belong to technician
- `404 Not Found`: Assignment not found
- `409 Conflict`: Invalid status transition (e.g., trying to complete without being on_site)

**Implementation** (Edge Function):

```typescript
Deno.serve(async (req) => {
  if (req.method !== 'PATCH') {
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

  const auth = await authorizeUser(supabase, user.id, ['technician']);
  if (!auth) {
    return errorResponse('Forbidden', 403, 'INSUFFICIENT_PERMISSIONS');
  }

  // Extract assignment ID from URL
  const url = new URL(req.url);
  const assignmentId = url.pathname.split('/').pop();

  if (!assignmentId) {
    return errorResponse('Assignment ID required', 400, 'MISSING_ASSIGNMENT_ID');
  }

  // Verify technician owns assignment
  const ownsAssignment = await verifyTechnicianOwnsAssignment(
    supabase,
    assignmentId,
    user.id,
    auth.orgId
  );

  if (!ownsAssignment) {
    return errorResponse('Assignment not found or access denied', 403, 'FORBIDDEN');
  }

  // Get current assignment status
  const { data: assignment, error: assignmentError } = await supabase
    .from('job_assignments')
    .select('status, dispatch_job_id')
    .eq('id', assignmentId)
    .eq('org_id', auth.orgId)
    .single();

  if (assignmentError || !assignment) {
    return errorResponse('Assignment not found', 404, 'ASSIGNMENT_NOT_FOUND');
  }

  const body = await req.json();

  if (!body.status) {
    return errorResponse('status is required', 400, 'MISSING_STATUS');
  }

  // Validate status transition
  if (!isValidStatusTransition(assignment.status, body.status)) {
    return errorResponse(
      `Invalid status transition from ${assignment.status} to ${body.status}`,
      409,
      'INVALID_STATUS_TRANSITION',
      {
        from_status: assignment.status,
        to_status: body.status,
        allowed_transitions: VALID_TRANSITIONS[assignment.status] || []
      }
    );
  }

  // Update assignment status
  const updateData: any = {
    status: body.status
  };

  if (body.notes) {
    updateData.notes = body.notes;
  }

  const { data: updatedAssignment, error: updateError } = await supabase
    .from('job_assignments')
    .update(updateData)
    .eq('id', assignmentId)
    .eq('org_id', auth.orgId)
    .select()
    .single();

  if (updateError) {
    return errorResponse('Failed to update assignment', 500, 'UPDATE_ERROR', { error: updateError.message });
  }

  // Update parent job status
  let jobStatusUpdated = false;
  if (assignment.dispatch_job_id) {
    await updateJobStatusFromAssignments(
      supabase,
      assignment.dispatch_job_id,
      auth.orgId
    );
    jobStatusUpdated = true;
  }

  return successResponse({
    id: updatedAssignment.id,
    status: updatedAssignment.status,
    updated_at: updatedAssignment.updated_at,
    job_status_updated: jobStatusUpdated
  });
});
```

---

### 4.2 Story DISP-034: Technician ETA Update API

#### 4.2.1 PATCH /dispatch/assignments/:id/eta

**Purpose**: Update ETA for an assignment (technician mobile app).

**Authorization**: `technician` (self-scoped only)

**Request Schema**:

```typescript
interface UpdateAssignmentETARequest {
  tech_eta_at: string; // ISO 8601 timestamp, required, must be in the future
}
```

**Request Example**:

```json
{
  "tech_eta_at": "2024-01-20T09:30:00Z"
}
```

**Response Schema**:

```typescript
interface AssignmentETAResponse {
  id: string;
  tech_eta_at: string;
  updated_at: string;
  notification_scheduled?: boolean; // indicates if notification was scheduled
}
```

**Response Example**:

```json
{
  "data": {
    "id": "999e4567-e89b-12d3-a456-426614174000",
    "tech_eta_at": "2024-01-20T09:30:00Z",
    "updated_at": "2024-01-20T09:15:00Z",
    "notification_scheduled": true
  }
}
```

#### 4.2.2 ETA Update Deduplication Strategy

**Problem**: Rapid repeated ETA updates can trigger notification spam.

**Solution**: Implement deduplication logic:

1. **Time-Based Deduplication**: Only trigger notification if ETA changed by more than threshold (e.g., 5 minutes)
2. **Rate Limiting**: Limit notification creation to once per N minutes (e.g., 10 minutes)
3. **Last ETA Tracking**: Store last notification ETA in assignment metadata or separate tracking table

**Deduplication Implementation**:

```typescript
interface ETADeduplicationConfig {
  minChangeMinutes: number; // Minimum change to trigger notification (default: 5)
  minIntervalMinutes: number; // Minimum interval between notifications (default: 10)
}

async function shouldTriggerETANotification(
  supabase: SupabaseClient,
  assignmentId: string,
  newETA: Date,
  config: ETADeduplicationConfig = { minChangeMinutes: 5, minIntervalMinutes: 10 }
): Promise<boolean> {
  // Get current assignment
  const { data: assignment } = await supabase
    .from('job_assignments')
    .select('tech_eta_at, metadata')
    .eq('id', assignmentId)
    .single();

  if (!assignment) {
    return false;
  }

  // Check if ETA changed significantly
  if (assignment.tech_eta_at) {
    const oldETA = new Date(assignment.tech_eta_at);
    const changeMinutes = Math.abs((newETA.getTime() - oldETA.getTime()) / (1000 * 60));
    
    if (changeMinutes < config.minChangeMinutes) {
      return false; // Change too small
    }
  }

  // Check last notification time from metadata
  const metadata = assignment.metadata || {};
  const lastNotificationETA = metadata.last_notification_eta_at 
    ? new Date(metadata.last_notification_eta_at)
    : null;

  if (lastNotificationETA) {
    const intervalMinutes = (newETA.getTime() - lastNotificationETA.getTime()) / (1000 * 60);
    if (intervalMinutes < config.minIntervalMinutes) {
      return false; // Too soon since last notification
    }
  }

  return true;
}
```

#### 4.2.3 Notification Trigger Rules

**When to Trigger "Tech on the Way" Notification**:

1. ETA is updated AND
2. Assignment status is `en_route` AND
3. Deduplication check passes AND
4. ETA is within reasonable window (e.g., next 2 hours)

**Notification Creation** (deferred to Epic 10):

```typescript
async function createETANotification(
  supabase: SupabaseClient,
  assignmentId: string,
  orgId: string,
  eta: Date
): Promise<void> {
  // Get assignment details
  const { data: assignment } = await supabase
    .from('job_assignments')
    .select(`
      id,
      dispatch_job_id,
      dispatch_jobs!inner(
        id,
        customer_id,
        location_id
      )
    `)
    .eq('id', assignmentId)
    .eq('org_id', orgId)
    .single();

  if (!assignment || !assignment.dispatch_jobs) {
    return;
  }

  // Get customer contact info
  const { data: customerContact } = await supabase
    .from('customer_contacts')
    .select('id')
    .eq('customer_id', assignment.dispatch_jobs.customer_id)
    .eq('type', 'mobile')
    .eq('is_primary', true)
    .single();

  if (!customerContact) {
    return;
  }

  // Create notification (deferred to Epic 10)
  // For now, just log
  console.log('Would create ETA notification:', {
    assignment_id: assignmentId,
    customer_contact_id: customerContact.id,
    eta: eta.toISOString()
  });

  // Update assignment metadata with last notification ETA
  await supabase
    .from('job_assignments')
    .update({
      metadata: {
        last_notification_eta_at: eta.toISOString()
      }
    })
    .eq('id', assignmentId);
}
```

**Validation Rules**:

1. Technician must own the assignment
2. `tech_eta_at` must be a valid ISO 8601 timestamp
3. `tech_eta_at` should be in the future (warning, not blocking)
4. Assignment status should be `en_route` or `on_site` for meaningful ETA (warning, not blocking)

**Error Responses**:

- `400 Bad Request`: Invalid timestamp format, ETA in the past
- `403 Forbidden`: User not authorized or assignment doesn't belong to technician
- `404 Not Found`: Assignment not found

**Implementation** (Edge Function):

```typescript
Deno.serve(async (req) => {
  if (req.method !== 'PATCH') {
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

  const auth = await authorizeUser(supabase, user.id, ['technician']);
  if (!auth) {
    return errorResponse('Forbidden', 403, 'INSUFFICIENT_PERMISSIONS');
  }

  // Extract assignment ID from URL
  const url = new URL(req.url);
  const assignmentId = url.pathname.split('/').pop();

  if (!assignmentId) {
    return errorResponse('Assignment ID required', 400, 'MISSING_ASSIGNMENT_ID');
  }

  // Verify technician owns assignment
  const ownsAssignment = await verifyTechnicianOwnsAssignment(
    supabase,
    assignmentId,
    user.id,
    auth.orgId
  );

  if (!ownsAssignment) {
    return errorResponse('Assignment not found or access denied', 403, 'FORBIDDEN');
  }

  const body = await req.json();

  if (!body.tech_eta_at) {
    return errorResponse('tech_eta_at is required', 400, 'MISSING_ETA');
  }

  // Validate timestamp format
  const etaDate = new Date(body.tech_eta_at);
  if (isNaN(etaDate.getTime())) {
    return errorResponse('Invalid timestamp format', 400, 'INVALID_TIMESTAMP');
  }

  // Warn if ETA is in the past (but allow it)
  const now = new Date();
  if (etaDate < now) {
    console.warn('ETA is in the past:', body.tech_eta_at);
  }

  // Get current assignment status
  const { data: assignment } = await supabase
    .from('job_assignments')
    .select('status')
    .eq('id', assignmentId)
    .eq('org_id', auth.orgId)
    .single();

  if (!assignment) {
    return errorResponse('Assignment not found', 404, 'ASSIGNMENT_NOT_FOUND');
  }

  // Check if notification should be triggered
  const shouldNotify = await shouldTriggerETANotification(
    supabase,
    assignmentId,
    etaDate
  );

  // Update ETA
  const { data: updatedAssignment, error: updateError } = await supabase
    .from('job_assignments')
    .update({
      tech_eta_at: body.tech_eta_at
    })
    .eq('id', assignmentId)
    .eq('org_id', auth.orgId)
    .select()
    .single();

  if (updateError) {
    return errorResponse('Failed to update ETA', 500, 'UPDATE_ERROR', { error: updateError.message });
  }

  // Trigger notification if conditions met
  let notificationScheduled = false;
  if (shouldNotify && ['en_route', 'on_site'].includes(assignment.status)) {
    await createETANotification(
      supabase,
      assignmentId,
      auth.orgId,
      etaDate
    );
    notificationScheduled = true;
  }

  return successResponse({
    id: updatedAssignment.id,
    tech_eta_at: updatedAssignment.tech_eta_at,
    updated_at: updatedAssignment.updated_at,
    notification_scheduled: notificationScheduled
  });
});
```

---

### 4.3 Story DISP-035: Real-Time Subscriptions for Technician Views

#### 4.3.1 Overview

Technicians need real-time updates to their assignments, jobs, and route plans when dispatchers make changes. Supabase Realtime provides this capability via PostgreSQL replication and WebSocket connections.

#### 4.3.2 Realtime Channel Specifications

**Channel Naming Convention**:
- Format: `technician:{technician_profile_id}:{resource_type}`
- Example: `technician:777e4567-e89b-12d3-a456-426614174000:assignments`

**Required Channels**:

1. **Assignments Channel**: `technician:{technician_profile_id}:assignments`
2. **Jobs Channel**: `technician:{technician_profile_id}:jobs`
3. **Route Plans Channel**: `technician:{technician_profile_id}:routes`
4. **Route Stops Channel**: `technician:{technician_profile_id}:route_stops`

#### 4.3.3 Channel 1: Assignments

**Channel Name**: `technician:{technician_profile_id}:assignments`

**Table**: `job_assignments`

**Filter**: `technician_id = {technician_profile_id} AND org_id = {org_id}`

**Events**: `INSERT`, `UPDATE`, `DELETE`

**Payload Fields**:
- `id`
- `dispatch_job_id`
- `scheduled_start_at`
- `scheduled_end_at`
- `arrival_window_start`
- `arrival_window_end`
- `status`
- `tech_eta_at`
- `sequence_in_route`
- `updated_at`

**Client Implementation** (TypeScript/JavaScript):

```typescript
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

// Get technician profile ID
const { data: techProfile } = await supabase
  .from('technician_profiles')
  .select('id')
  .eq('user_id', userId)
  .single();

const technicianId = techProfile.id;

// Subscribe to assignments
const assignmentsChannel = supabase
  .channel(`technician:${technicianId}:assignments`)
  .on(
    'postgres_changes',
    {
      event: '*', // INSERT, UPDATE, DELETE
      schema: 'public',
      table: 'job_assignments',
      filter: `technician_id=eq.${technicianId}`
    },
    (payload) => {
      console.log('Assignment change:', payload);
      // Handle assignment update in mobile app
      handleAssignmentChange(payload);
    }
  )
  .subscribe();
```

**Security**: RLS policies from Epic 3 ensure technicians can only see their own assignments.

#### 4.3.4 Channel 2: Jobs

**Channel Name**: `technician:{technician_profile_id}:jobs`

**Table**: `dispatch_jobs`

**Filter**: `id IN (SELECT dispatch_job_id FROM job_assignments WHERE technician_id = {technician_profile_id})`

**Events**: `UPDATE` (technicians don't create jobs, only see assigned ones)

**Payload Fields**:
- `id`
- `title`
- `description`
- `priority`
- `status`
- `estimated_duration_minutes`
- `sla_start_at`
- `sla_end_at`
- `updated_at`

**Client Implementation**:

```typescript
// Subscribe to jobs (only updates, not inserts)
const jobsChannel = supabase
  .channel(`technician:${technicianId}:jobs`)
  .on(
    'postgres_changes',
    {
      event: 'UPDATE',
      schema: 'public',
      table: 'dispatch_jobs',
      filter: `id=in.(${assignedJobIds.join(',')})` // Dynamic filter based on assignments
    },
    (payload) => {
      console.log('Job update:', payload);
      handleJobUpdate(payload);
    }
  )
  .subscribe();
```

**Note**: Filter must be dynamic based on current assignments. Consider using a database function or view for better performance.

#### 4.3.5 Channel 3: Route Plans

**Channel Name**: `technician:{technician_profile_id}:routes`

**Table**: `route_plans`

**Filter**: `technician_id = {technician_profile_id} AND org_id = {org_id}`

**Events**: `INSERT`, `UPDATE`, `DELETE`

**Payload Fields**:
- `id`
- `date`
- `status`
- `optimization_strategy`
- `optimization_metadata`
- `updated_at`

**Client Implementation**:

```typescript
const routesChannel = supabase
  .channel(`technician:${technicianId}:routes`)
  .on(
    'postgres_changes',
    {
      event: '*',
      schema: 'public',
      table: 'route_plans',
      filter: `technician_id=eq.${technicianId}`
    },
    (payload) => {
      console.log('Route plan change:', payload);
      handleRoutePlanChange(payload);
    }
  )
  .subscribe();
```

#### 4.3.6 Channel 4: Route Stops

**Channel Name**: `technician:{technician_profile_id}:route_stops`

**Table**: `route_stops`

**Filter**: `route_plan_id IN (SELECT id FROM route_plans WHERE technician_id = {technician_profile_id})`

**Events**: `INSERT`, `UPDATE`, `DELETE`

**Payload Fields**:
- `id`
- `route_plan_id`
- `job_assignment_id`
- `stop_type`
- `sequence`
- `planned_arrival_at`
- `planned_departure_at`
- `actual_arrival_at`
- `actual_departure_at`
- `updated_at`

**Client Implementation**:

```typescript
// Get route plan IDs for technician
const { data: routePlans } = await supabase
  .from('route_plans')
  .select('id')
  .eq('technician_id', technicianId);

const routePlanIds = routePlans?.map(rp => rp.id) || [];

const routeStopsChannel = supabase
  .channel(`technician:${technicianId}:route_stops`)
  .on(
    'postgres_changes',
    {
      event: '*',
      schema: 'public',
      table: 'route_stops',
      filter: `route_plan_id=in.(${routePlanIds.join(',')})`
    },
    (payload) => {
      console.log('Route stop change:', payload);
      handleRouteStopChange(payload);
    }
  )
  .subscribe();
```

#### 4.3.7 Minimizing Payload Size

**Strategies**:

1. **Selective Fields**: Only subscribe to fields needed by mobile app
2. **Filter Aggressively**: Use RLS and channel filters to minimize events
3. **Debouncing**: Client-side debouncing for rapid updates
4. **Pagination**: Use pagination for initial data load, realtime for updates only

**Example**: Subscribe only to status and ETA fields:

```typescript
// Not directly supported by Supabase Realtime
// Workaround: Filter on client side or use database views
```

**Database View Approach** (Optional):

```sql
-- Create lightweight view for realtime subscriptions
CREATE VIEW technician_assignments_realtime AS
SELECT 
  id,
  dispatch_job_id,
  scheduled_start_at,
  scheduled_end_at,
  status,
  tech_eta_at,
  updated_at
FROM job_assignments;

-- Subscribe to view instead of table
```

#### 4.3.8 Reconnect and Backfill Strategy

**Reconnection Handling**:

```typescript
let assignmentsChannel: RealtimeChannel;

function setupRealtimeSubscriptions() {
  assignmentsChannel = supabase
    .channel(`technician:${technicianId}:assignments`)
    .on('postgres_changes', { ... }, handleAssignmentChange)
    .on('system', {}, (payload) => {
      if (payload.status === 'SUBSCRIBED') {
        console.log('Realtime connected');
        // Backfill any missed updates
        backfillMissedUpdates();
      } else if (payload.status === 'CHANNEL_ERROR') {
        console.error('Realtime error:', payload);
        // Reconnect after delay
        setTimeout(setupRealtimeSubscriptions, 5000);
      }
    })
    .subscribe();
}

async function backfillMissedUpdates() {
  // Get last known update timestamp from local storage
  const lastUpdate = localStorage.getItem('last_assignment_update');
  
  // Fetch updates since last known timestamp
  const { data: updates } = await supabase
    .from('job_assignments')
    .select('*')
    .eq('technician_id', technicianId)
    .gt('updated_at', lastUpdate || '1970-01-01')
    .order('updated_at', { ascending: true });

  // Process updates
  updates?.forEach(update => {
    handleAssignmentChange({ eventType: 'UPDATE', new: update });
  });

  // Update last known timestamp
  if (updates && updates.length > 0) {
    localStorage.setItem('last_assignment_update', updates[updates.length - 1].updated_at);
  }
}
```

**Backfill Strategy**:

1. **On Reconnect**: Fetch updates since last known timestamp
2. **On App Start**: Fetch all current assignments, then subscribe to realtime
3. **Timestamp Tracking**: Store last update timestamp in local storage

#### 4.3.9 Security Considerations

**RLS Enforcement**:

- RLS policies from Epic 3 ensure technicians can only see their own data
- Realtime subscriptions respect RLS policies automatically
- Channel filters provide additional security layer

**Verification**:

1. Technician A cannot see Technician B's assignments via realtime
2. Cross-org data is blocked by RLS
3. Channel filters match RLS policy filters

**Testing Checklist**:

- [ ] Technician can only subscribe to own assignments
- [ ] Cross-technician access is blocked
- [ ] Cross-org access is blocked
- [ ] Realtime events match RLS permissions

#### 4.3.10 Realtime Configuration

**Enable Realtime on Tables** (Supabase Dashboard):

1. Navigate to Database > Replication
2. Enable replication for:
   - `job_assignments`
   - `dispatch_jobs`
   - `route_plans`
   - `route_stops`

**Or via SQL**:

```sql
-- Enable replication (requires superuser or Supabase dashboard)
ALTER PUBLICATION supabase_realtime ADD TABLE job_assignments;
ALTER PUBLICATION supabase_realtime ADD TABLE dispatch_jobs;
ALTER PUBLICATION supabase_realtime ADD TABLE route_plans;
ALTER PUBLICATION supabase_realtime ADD TABLE route_stops;
```

**RLS and Realtime**:

- RLS policies automatically apply to realtime subscriptions
- No additional configuration needed
- Users only receive events for rows they can access via RLS

---

## 5. Error Handling

### 5.1 Standard Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid authentication token |
| `FORBIDDEN` | 403 | User not authorized or assignment doesn't belong to technician |
| `NOT_FOUND` | 404 | Assignment not found |
| `INVALID_STATUS_TRANSITION` | 409 | Invalid status transition according to state machine |
| `INVALID_TIMESTAMP` | 400 | Invalid timestamp format |
| `MISSING_ASSIGNMENT_ID` | 400 | Assignment ID missing from URL |
| `MISSING_STATUS` | 400 | Status missing from request body |
| `MISSING_ETA` | 400 | tech_eta_at missing from request body |

---

## 6. Testing Requirements

### 6.1 Unit Tests

Test each endpoint with:

1. **Authorization Tests**:
   - Valid technician access
   - Invalid/expired token
   - Technician trying to access another technician's assignment
   - Non-technician role access

2. **Validation Tests**:
   - Invalid status transitions
   - Invalid timestamp formats
   - Missing required fields

3. **Business Logic Tests**:
   - Status state machine transitions
   - Job status updates triggered by assignment status changes
   - ETA notification deduplication
   - ETA notification triggers

### 6.2 Integration Tests

Test with:
- Real Supabase instance
- Multiple technicians in same org
- Real-time subscription functionality
- Reconnection scenarios

### 6.3 Realtime Tests

1. **Subscription Tests**:
   - Technician receives own assignment updates
   - Technician does not receive other technicians' updates
   - Cross-org data is blocked

2. **Reconnection Tests**:
   - Backfill works correctly after reconnect
   - No duplicate events on reconnect
   - Timestamp tracking works correctly

---

## 7. Implementation Checklist

### Story DISP-033: Technician Assignment Status Update API

- [ ] **PATCH /dispatch/assignments/:id/status**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced (technician self-scope only)
  - [ ] State machine validation implemented
  - [ ] Status transition rules documented
  - [ ] Job status update logic integrated
  - [ ] Error handling implemented
  - [ ] API documentation with state machine diagram

- [ ] **State Machine Documentation**:
  - [ ] Valid transitions documented in Markdown
  - [ ] State machine diagram created
  - [ ] Invalid transition error messages include allowed transitions

- [ ] **Testing**:
  - [ ] Tests confirm technicians cannot update others' assignments
  - [ ] Tests cover all valid status transitions
  - [ ] Tests cover invalid status transitions
  - [ ] Tests verify job status updates

### Story DISP-034: Technician ETA Update API

- [ ] **PATCH /dispatch/assignments/:id/eta**:
  - [ ] Endpoint implemented
  - [ ] Authorization enforced (technician self-scope only)
  - [ ] Timestamp validation implemented
  - [ ] ETA deduplication logic implemented
  - [ ] Notification trigger logic implemented
  - [ ] Error handling implemented
  - [ ] API documentation with examples

- [ ] **Deduplication Strategy**:
  - [ ] Time-based deduplication implemented
  - [ ] Rate limiting implemented
  - [ ] Last ETA tracking implemented
  - [ ] Configuration documented (minChangeMinutes, minIntervalMinutes)

- [ ] **Notification Triggers**:
  - [ ] Trigger rules documented
  - [ ] Notification creation logic (deferred to Epic 10, but interface defined)
  - [ ] Metadata tracking for last notification ETA

- [ ] **Testing**:
  - [ ] Tests cover rapid repeated ETA updates
  - [ ] Tests verify deduplication works correctly
  - [ ] Tests verify notification triggers

### Story DISP-035: Real-Time Subscriptions for Technician Views

- [ ] **Realtime Channel Specifications**:
  - [ ] Assignments channel documented
  - [ ] Jobs channel documented
  - [ ] Route plans channel documented
  - [ ] Route stops channel documented
  - [ ] Channel naming convention documented

- [ ] **Client Implementation**:
  - [ ] TypeScript/JavaScript examples provided
  - [ ] Reconnection handling documented
  - [ ] Backfill strategy documented
  - [ ] Payload minimization strategies documented

- [ ] **Security Review**:
  - [ ] RLS policies verified for realtime
  - [ ] Channel filters verified
  - [ ] Cross-technician access blocked
  - [ ] Cross-org access blocked

- [ ] **Realtime Configuration**:
  - [ ] Tables enabled for replication
  - [ ] Configuration documented
  - [ ] Testing checklist completed

---

## 8. Deployment

### 8.1 Edge Function Structure

```
supabase/functions/
  dispatch-assignment-status/
    index.ts
    _shared/
      auth.ts
      validation.ts
      state_machine.ts
      errors.ts
  dispatch-assignment-eta/
    index.ts
    _shared/
      auth.ts
      deduplication.ts
      notifications.ts
      errors.ts
```

### 8.2 Deployment Commands

```bash
# Deploy Edge Functions
supabase functions deploy dispatch-assignment-status
supabase functions deploy dispatch-assignment-eta

# Test locally
supabase functions serve dispatch-assignment-status
supabase functions serve dispatch-assignment-eta
```

### 8.3 Realtime Configuration

Enable replication via Supabase Dashboard or SQL (see Section 4.3.10).

---

## 9. Appendix: State Machine Diagram

### 9.1 Assignment Status State Machine

```
[assigned] --accept--> [accepted] --en_route--> [en_route] --on_site--> [on_site] --completed--> [completed]
     |                      |                                                                          |
     |                      |                                                                          |
     |                      |                                                                          |
     +--decline--> [declined]                                                                    [no_show]
                                                                                                    (dispatcher only)

Terminal States: [completed], [declined], [canceled], [no_show]
```

**Legend**:
- `[state]` - Status state
- `--action-->` - Valid transition
- Terminal states cannot transition to other states

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 6 – Technician Mobile Hooks: Status & ETA Updates. All APIs are designed as Supabase Edge Functions with complete request/response schemas, state machine validation, deduplication logic, and real-time subscription specifications.

**Next Steps**: After completing Epic 6, proceed to Epic 7 (Auto-Scheduling and Route Optimization) which will implement AI-driven scheduling and route optimization Edge Functions.

