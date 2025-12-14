# Technical Design Document – Epic 16: Integration Hooks, Change Triggers, and Real-Time Location (Dispatch-Adjacent)

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 16 – Integration Hooks, Change Triggers, and Real-Time Location (Dispatch-Adjacent)
- **Source**: Derived from `fdd_2_agile.md` Epic 16 (Stories DISP-065 through DISP-071)
- **Reference Documents**: 
  - `fdd_2.md` (Functional Design Document for Scheduling & Dispatch)
  - `fdd_2_agile.md` (Agile User Stories)
  - `tdd_2_epic_1.md` (Dispatch Epic 1 for tenancy/role conventions)
  - `tdd_2_epic_2.md` (Dispatch Epic 2 for table schemas)
  - `tdd_2_epic_3.md` (Dispatch Epic 3 for RLS policies)
  - `tdd_2_epic_5.md` (Dispatch Epic 5 for job lifecycle APIs)
  - `tdd_2_epic_7.md` (Dispatch Epic 7 for auto-scheduling and route optimization)
  - `tdd_2_epic_8.md` (Dispatch Epic 8 for emergency job handling)
  - `tdd_2_epic_9.md` (Dispatch Epic 9 for calendar integration)
  - `tdd_2_epic_11.md` (Dispatch Epic 11 for Next.js UI patterns)
- **Target Platform**: PostgreSQL 15+, Supabase Edge Functions (Deno/TypeScript), Next.js 14+ (App Router), shadcn/ui
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing Integration Hooks, Change Triggers, and Real-Time Location features for the Scheduling & Dispatch module. It covers:

- Work Order to Dispatch Job synchronization strategy and event handlers
- System-suggested time window generation rules
- Re-optimization triggers on schedule/availability changes
- SLA and time-window conflict detection (warnings/blocks)
- Technician location ingestion and storage for map-based dispatch
- Calendar token refresh and expiration handling

All integration hooks are implemented as Supabase Edge Functions with event-driven patterns. Real-time location tracking enables map-based dispatch views, and constraint validation ensures schedule integrity.

This epic assumes Epic 1-15 are complete and all dispatch functionality is operational.

---

## 2. Prerequisites

### 2.1 Required Components

Before implementing Epic 16, ensure:

1. **Epic 1-15 Complete**: All previous epics are implemented
2. **Required Tables**: All dispatch tables exist:
   - `dispatch_jobs`
   - `job_assignments`
   - `job_time_windows`
   - `route_plans`
   - `route_stops`
   - `technician_profiles`
   - `technician_shifts`
   - `technician_time_off`
   - `calendar_integrations`
   - `calendar_events`

3. **Work Order Module**: Work Order tables exist (assumed separate module):
   - `work_orders` (or unified with `dispatch_jobs`)
   - Work order event system (webhooks or database triggers)

4. **Mobile App**: Technician mobile app exists and can send location updates

---

## 3. Story DISP-065: Work Order → Dispatch Job Synchronization Strategy

### 3.1 Architecture Decision: Separate vs Unified Model

**Decision**: Use **separate model** with `related_work_order_id` foreign key.

**Rationale**:
- Work Orders and Dispatch Jobs serve different purposes
- Work Orders: Operational work tracking, materials, billing
- Dispatch Jobs: Scheduling, routing, technician assignment
- Separation allows independent evolution of each domain
- Clear separation of concerns

**Schema Addition**:

```sql
-- Add foreign key to dispatch_jobs (if not already present)
ALTER TABLE dispatch_jobs
ADD COLUMN IF NOT EXISTS related_work_order_id UUID REFERENCES work_orders(id) ON DELETE SET NULL;

CREATE INDEX IF NOT EXISTS idx_dispatch_jobs_related_work_order_id 
  ON dispatch_jobs(related_work_order_id) 
  WHERE related_work_order_id IS NOT NULL;

-- Add index for reverse lookup
CREATE INDEX IF NOT EXISTS idx_dispatch_jobs_org_work_order 
  ON dispatch_jobs(org_id, related_work_order_id) 
  WHERE related_work_order_id IS NOT NULL;
```

**Alternative (Unified Model)**: If work orders and dispatch jobs are unified, skip `related_work_order_id` and add dispatch fields directly to `work_orders` table.

### 3.2 Event Contract

**Work Order Events**:

```typescript
interface WorkOrderEvent {
  event_type: 'work_order.created' | 'work_order.updated' | 'work_order.canceled' | 'work_order.completed';
  work_order_id: string;
  org_id: string;
  timestamp: string; // ISO 8601
  payload: {
    work_order: {
      id: string;
      org_id: string;
      customer_id: string;
      location_id: string;
      status: string;
      priority?: 'low' | 'normal' | 'high' | 'emergency';
      title: string;
      description?: string;
      estimated_duration_minutes?: number;
      scheduled_start_at?: string; // ISO 8601
      scheduled_end_at?: string; // ISO 8601
      required_skills?: string[];
      service_zone_id?: string;
      metadata?: any;
    };
    changes?: {
      // Fields that changed (for updated events)
      field_name: string;
      old_value: any;
      new_value: any;
    }[];
  };
  event_id: string; // Unique event ID for idempotency
  source: 'work_order_module'; // Source system identifier
}
```

**Event Types**:

1. **`work_order.created`**: New work order created → Create dispatch job
2. **`work_order.updated`**: Work order updated → Update dispatch job (if exists)
3. **`work_order.canceled`**: Work order canceled → Cancel dispatch job and assignments
4. **`work_order.completed`**: Work order completed → Complete dispatch job and assignments

### 3.3 Synchronization Flow

**Happy Path: Work Order Created → Dispatch Job Created**

```
1. Work Order Module creates work_order record
2. Work Order Module emits work_order.created event
3. Dispatch Event Handler receives event
4. Handler validates event (idempotency, org_id, permissions)
5. Handler creates dispatch_job with:
   - related_work_order_id = work_order.id
   - priority = work_order.priority (mapped)
   - location_id = work_order.location_id
   - estimated_duration_minutes = work_order.estimated_duration_minutes
   - required_skills = work_order.required_skills
   - service_zone_id = work_order.service_zone_id
   - status = 'pending'
6. Handler creates job_time_windows if work_order.scheduled_start_at exists
7. Handler returns success response
```

**Happy Path: Work Order Updated → Dispatch Job Updated**

```
1. Work Order Module updates work_order record
2. Work Order Module emits work_order.updated event
3. Dispatch Event Handler receives event
4. Handler finds existing dispatch_job by related_work_order_id
5. Handler updates dispatch_job fields:
   - priority (if changed)
   - location_id (if changed)
   - estimated_duration_minutes (if changed)
   - required_skills (if changed)
   - service_zone_id (if changed)
6. Handler updates job_time_windows if scheduled times changed
7. Handler triggers re-optimization if assignments exist (via Epic 16 Story DISP-068)
8. Handler returns success response
```

**Happy Path: Work Order Canceled → Dispatch Job Canceled**

```
1. Work Order Module cancels work_order record
2. Work Order Module emits work_order.canceled event
3. Dispatch Event Handler receives event
4. Handler finds existing dispatch_job by related_work_order_id
5. Handler cancels all job_assignments:
   - Set status = 'canceled'
   - Set canceled_at = now()
   - Set canceled_reason = 'Work order canceled'
6. Handler updates dispatch_job:
   - Set status = 'canceled'
   - Set canceled_at = now()
7. Handler triggers re-optimization for affected technicians (via Epic 16 Story DISP-068)
8. Handler returns success response
```

**Happy Path: Work Order Completed → Dispatch Job Completed**

```
1. Work Order Module completes work_order record
2. Work Order Module emits work_order.completed event
3. Dispatch Event Handler receives event
4. Handler finds existing dispatch_job by related_work_order_id
5. Handler completes all job_assignments:
   - Set status = 'completed'
   - Set completed_at = now()
6. Handler updates dispatch_job:
   - Set status = 'completed'
   - Set completed_at = now()
7. Handler returns success response
```

### 3.4 Field Mapping

**Work Order → Dispatch Job Field Mapping**:

| Work Order Field | Dispatch Job Field | Notes |
|-----------------|-------------------|-------|
| `id` | `related_work_order_id` | Foreign key reference |
| `org_id` | `org_id` | Same |
| `customer_id` | `customer_id` | Same |
| `location_id` | `location_id` | Same |
| `priority` | `priority` | Direct mapping |
| `title` | `title` | Same |
| `description` | `description` | Same |
| `estimated_duration_minutes` | `estimated_duration_minutes` | Same |
| `scheduled_start_at` | `job_time_windows.start_at` | Creates time window |
| `scheduled_end_at` | `job_time_windows.end_at` | Creates time window |
| `required_skills` | `required_skills` (JSONB array) | Same |
| `service_zone_id` | `service_zone_id` | Same |
| `metadata` | `metadata` (JSONB) | Merged with dispatch-specific metadata |

**Priority Mapping**:

```typescript
const PRIORITY_MAP: Record<string, 'low' | 'normal' | 'high' | 'emergency'> = {
  'low': 'low',
  'normal': 'normal',
  'standard': 'normal',
  'high': 'high',
  'urgent': 'high',
  'emergency': 'emergency',
  'critical': 'emergency'
};
```

### 3.5 Documentation Requirements

**Integration Contract Document**: `docs/technical/work-order-dispatch-integration.md`

**Contents**:
- Architecture decision (separate vs unified)
- Event contract schema
- Synchronization flows (happy paths)
- Field mapping table
- Error handling and retry logic
- Idempotency strategy
- Security model (authentication, authorization)
- Testing scenarios

---

## 4. Story DISP-066: Implement Work Order Event Handler Hook

### 4.1 Event Handler Architecture

**Approach**: Edge Function webhook endpoint

**Endpoint**: `POST /webhooks/work-orders`

**Authentication**: Service role or signed webhook secret

### 4.2 Event Handler Implementation

**File**: `supabase/functions/work-order-event-handler/index.ts`

```typescript
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
import { corsHeaders } from '../_shared/cors.ts';
import { checkIdempotency, completeIdempotency } from '../_shared/idempotency.ts';
import { createLogger, generateTraceId } from '../_shared/logging.ts';

interface WorkOrderEvent {
  event_type: 'work_order.created' | 'work_order.updated' | 'work_order.canceled' | 'work_order.completed';
  work_order_id: string;
  org_id: string;
  timestamp: string;
  payload: {
    work_order: {
      id: string;
      org_id: string;
      customer_id: string;
      location_id: string;
      status: string;
      priority?: string;
      title: string;
      description?: string;
      estimated_duration_minutes?: number;
      scheduled_start_at?: string;
      scheduled_end_at?: string;
      required_skills?: string[];
      service_zone_id?: string;
      metadata?: any;
    };
    changes?: Array<{
      field_name: string;
      old_value: any;
      new_value: any;
    }>;
  };
  event_id: string;
  source: string;
}

Deno.serve(async (req) => {
  // Handle CORS
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders });
  }

  // Verify webhook signature (if using signed webhooks)
  const signature = req.headers.get('X-Webhook-Signature');
  const webhookSecret = Deno.env.get('WORK_ORDER_WEBHOOK_SECRET');
  
  if (webhookSecret && signature) {
    const isValid = await verifyWebhookSignature(req, signature, webhookSecret);
    if (!isValid) {
      return new Response(
        JSON.stringify({ error: 'Invalid webhook signature' }),
        { status: 401, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    }
  }

  // Parse event
  const event: WorkOrderEvent = await req.json();

  // Initialize Supabase client with service role
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
  );

  // Generate trace ID
  const traceId = generateTraceId(event.org_id, 'work_order_event');
  const logger = createLogger({
    traceId,
    orgId: event.org_id,
    operationType: 'work_order_event_handler'
  });

  logger.info('Work order event received', {
    event_type: event.event_type,
    work_order_id: event.work_order_id,
    event_id: event.event_id
  });

  // Check idempotency
  const idempotencyResult = await checkIdempotency(supabase, {
    orgId: event.org_id,
    operationType: 'work_order_event',
    targetId: event.work_order_id,
    requestPayload: {
      event_type: event.event_type,
      event_id: event.event_id,
      timestamp: event.timestamp
    },
    ttlHours: 24
  });

  if (idempotencyResult.isDuplicate) {
    logger.info('Duplicate event detected, returning cached response', {
      event_id: event.event_id
    });
    return new Response(
      JSON.stringify({ success: true, message: 'Event already processed', data: idempotencyResult.existingResponse }),
      { headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }

  try {
    let result: any;

    // Route to appropriate handler
    switch (event.event_type) {
      case 'work_order.created':
        result = await handleWorkOrderCreated(supabase, event, logger);
        break;
      case 'work_order.updated':
        result = await handleWorkOrderUpdated(supabase, event, logger);
        break;
      case 'work_order.canceled':
        result = await handleWorkOrderCanceled(supabase, event, logger);
        break;
      case 'work_order.completed':
        result = await handleWorkOrderCompleted(supabase, event, logger);
        break;
      default:
        throw new Error(`Unknown event type: ${event.event_type}`);
    }

    // Mark idempotency as complete
    await completeIdempotency(supabase, event.org_id, idempotencyResult.key, result);

    logger.info('Work order event processed successfully', {
      event_type: event.event_type,
      work_order_id: event.work_order_id
    });

    return new Response(
      JSON.stringify({ success: true, data: result }),
      { headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  } catch (error) {
    logger.error('Work order event processing failed', error as Error, {
      event_type: event.event_type,
      work_order_id: event.work_order_id
    });

    await completeIdempotency(supabase, event.org_id, idempotencyResult.key, null, error as Error);

    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
});

async function handleWorkOrderCreated(
  supabase: SupabaseClient,
  event: WorkOrderEvent,
  logger: Logger
): Promise<any> {
  const { work_order } = event.payload;

  logger.info('Creating dispatch job from work order', {
    work_order_id: work_order.id
  });

  // Map priority
  const priorityMap: Record<string, 'low' | 'normal' | 'high' | 'emergency'> = {
    'low': 'low',
    'normal': 'normal',
    'standard': 'normal',
    'high': 'high',
    'urgent': 'high',
    'emergency': 'emergency',
    'critical': 'emergency'
  };

  const dispatchPriority = priorityMap[work_order.priority?.toLowerCase() || 'normal'] || 'normal';

  // Create dispatch job
  const { data: dispatchJob, error: jobError } = await supabase
    .from('dispatch_jobs')
    .insert({
      org_id: work_order.org_id,
      customer_id: work_order.customer_id,
      location_id: work_order.location_id,
      related_work_order_id: work_order.id,
      title: work_order.title,
      description: work_order.description,
      priority: dispatchPriority,
      status: 'pending',
      estimated_duration_minutes: work_order.estimated_duration_minutes,
      required_skills: work_order.required_skills || [],
      service_zone_id: work_order.service_zone_id,
      metadata: {
        ...work_order.metadata,
        source: 'work_order',
        work_order_id: work_order.id,
        created_from_event: event.event_id
      }
    })
    .select()
    .single();

  if (jobError) {
    throw new Error(`Failed to create dispatch job: ${jobError.message}`);
  }

  logger.info('Dispatch job created', {
    dispatch_job_id: dispatchJob.id,
    work_order_id: work_order.id
  });

  // Create time windows if scheduled times provided
  if (work_order.scheduled_start_at && work_order.scheduled_end_at) {
    const { error: windowError } = await supabase
      .from('job_time_windows')
      .insert({
        org_id: work_order.org_id,
        dispatch_job_id: dispatchJob.id,
        start_at: work_order.scheduled_start_at,
        end_at: work_order.scheduled_end_at,
        source: 'system_suggested',
        is_selected: false
      });

    if (windowError) {
      logger.warn('Failed to create time window', {
        dispatch_job_id: dispatchJob.id,
        error: windowError.message
      });
    }
  }

  return {
    dispatch_job_id: dispatchJob.id,
    created: true
  };
}

async function handleWorkOrderUpdated(
  supabase: SupabaseClient,
  event: WorkOrderEvent,
  logger: Logger
): Promise<any> {
  const { work_order, changes } = event.payload;

  logger.info('Updating dispatch job from work order', {
    work_order_id: work_order.id
  });

  // Find existing dispatch job
  const { data: existingJob, error: findError } = await supabase
    .from('dispatch_jobs')
    .select('id, status')
    .eq('org_id', work_order.org_id)
    .eq('related_work_order_id', work_order.id)
    .single();

  if (findError || !existingJob) {
    // No existing job, create one
    logger.info('No existing dispatch job found, creating new one', {
      work_order_id: work_order.id
    });
    return await handleWorkOrderCreated(supabase, event, logger);
  }

  // Map priority
  const priorityMap: Record<string, 'low' | 'normal' | 'high' | 'emergency'> = {
    'low': 'low',
    'normal': 'normal',
    'standard': 'normal',
    'high': 'high',
    'urgent': 'high',
    'emergency': 'emergency',
    'critical': 'emergency'
  };

  const dispatchPriority = priorityMap[work_order.priority?.toLowerCase() || 'normal'] || 'normal';

  // Build update object
  const updates: any = {
    title: work_order.title,
    description: work_order.description,
    priority: dispatchPriority,
    estimated_duration_minutes: work_order.estimated_duration_minutes,
    required_skills: work_order.required_skills || [],
    service_zone_id: work_order.service_zone_id,
    metadata: {
      ...work_order.metadata,
      source: 'work_order',
      work_order_id: work_order.id,
      last_updated_from_event: event.event_id
    }
  };

  // Update location if changed
  if (changes?.some(c => c.field_name === 'location_id')) {
    updates.location_id = work_order.location_id;
  }

  // Update dispatch job
  const { error: updateError } = await supabase
    .from('dispatch_jobs')
    .update(updates)
    .eq('id', existingJob.id);

  if (updateError) {
    throw new Error(`Failed to update dispatch job: ${updateError.message}`);
  }

  // Update time windows if scheduled times changed
  if (changes?.some(c => c.field_name === 'scheduled_start_at' || c.field_name === 'scheduled_end_at')) {
    if (work_order.scheduled_start_at && work_order.scheduled_end_at) {
      // Delete existing windows
      await supabase
        .from('job_time_windows')
        .delete()
        .eq('dispatch_job_id', existingJob.id);

      // Create new window
      await supabase
        .from('job_time_windows')
        .insert({
          org_id: work_order.org_id,
          dispatch_job_id: existingJob.id,
          start_at: work_order.scheduled_start_at,
          end_at: work_order.scheduled_end_at,
          source: 'system_suggested',
          is_selected: false
        });
    }
  }

  // Trigger re-optimization if assignments exist (via Epic 16 Story DISP-068)
  const { data: assignments } = await supabase
    .from('job_assignments')
    .select('technician_id')
    .eq('dispatch_job_id', existingJob.id)
    .in('status', ['assigned', 'accepted', 'en_route', 'on_site']);

  if (assignments && assignments.length > 0) {
    // Queue re-optimization (implementation in Story DISP-068)
    logger.info('Queueing re-optimization for affected technicians', {
      dispatch_job_id: existingJob.id,
      technician_count: assignments.length
    });
  }

  return {
    dispatch_job_id: existingJob.id,
    updated: true
  };
}

async function handleWorkOrderCanceled(
  supabase: SupabaseClient,
  event: WorkOrderEvent,
  logger: Logger
): Promise<any> {
  const { work_order } = event.payload;

  logger.info('Canceling dispatch job from work order', {
    work_order_id: work_order.id
  });

  // Find existing dispatch job
  const { data: existingJob, error: findError } = await supabase
    .from('dispatch_jobs')
    .select('id')
    .eq('org_id', work_order.org_id)
    .eq('related_work_order_id', work_order.id)
    .single();

  if (findError || !existingJob) {
    logger.warn('No existing dispatch job found', {
      work_order_id: work_order.id
    });
    return { canceled: false, message: 'No dispatch job found' };
  }

  // Cancel all assignments
  const { data: assignments } = await supabase
    .from('job_assignments')
    .select('id, technician_id')
    .eq('dispatch_job_id', existingJob.id)
    .in('status', ['assigned', 'accepted', 'en_route', 'on_site']);

  if (assignments && assignments.length > 0) {
    const assignmentIds = assignments.map(a => a.id);
    const technicianIds = [...new Set(assignments.map(a => a.technician_id))];

    await supabase
      .from('job_assignments')
      .update({
        status: 'canceled',
        canceled_at: new Date().toISOString(),
        canceled_reason: 'Work order canceled'
      })
      .in('id', assignmentIds);

    logger.info('Canceled assignments', {
      dispatch_job_id: existingJob.id,
      assignment_count: assignments.length
    });

    // Trigger re-optimization for affected technicians (via Epic 16 Story DISP-068)
    logger.info('Queueing re-optimization for affected technicians', {
      technician_ids: technicianIds
    });
  }

  // Update dispatch job
  await supabase
    .from('dispatch_jobs')
    .update({
      status: 'canceled',
      canceled_at: new Date().toISOString()
    })
    .eq('id', existingJob.id);

  return {
    dispatch_job_id: existingJob.id,
    canceled: true,
    assignments_canceled: assignments?.length || 0
  };
}

async function handleWorkOrderCompleted(
  supabase: SupabaseClient,
  event: WorkOrderEvent,
  logger: Logger
): Promise<any> {
  const { work_order } = event.payload;

  logger.info('Completing dispatch job from work order', {
    work_order_id: work_order.id
  });

  // Find existing dispatch job
  const { data: existingJob, error: findError } = await supabase
    .from('dispatch_jobs')
    .select('id')
    .eq('org_id', work_order.org_id)
    .eq('related_work_order_id', work_order.id)
    .single();

  if (findError || !existingJob) {
    logger.warn('No existing dispatch job found', {
      work_order_id: work_order.id
    });
    return { completed: false, message: 'No dispatch job found' };
  }

  // Complete all assignments
  const { data: assignments } = await supabase
    .from('job_assignments')
    .select('id')
    .eq('dispatch_job_id', existingJob.id)
    .in('status', ['assigned', 'accepted', 'en_route', 'on_site', 'completed']);

  if (assignments && assignments.length > 0) {
    const assignmentIds = assignments.map(a => a.id);

    await supabase
      .from('job_assignments')
      .update({
        status: 'completed',
        completed_at: new Date().toISOString()
      })
      .in('id', assignmentIds)
      .in('status', ['assigned', 'accepted', 'en_route', 'on_site']); // Only update non-completed

    logger.info('Completed assignments', {
      dispatch_job_id: existingJob.id,
      assignment_count: assignments.length
    });
  }

  // Update dispatch job
  await supabase
    .from('dispatch_jobs')
    .update({
      status: 'completed',
      completed_at: new Date().toISOString()
    })
    .eq('id', existingJob.id);

  return {
    dispatch_job_id: existingJob.id,
    completed: true,
    assignments_completed: assignments?.length || 0
  };
}

async function verifyWebhookSignature(
  req: Request,
  signature: string,
  secret: string
): Promise<boolean> {
  // Implement HMAC-SHA256 signature verification
  const body = await req.clone().text();
  const expectedSignature = await crypto.subtle.digest(
    'SHA-256',
    new TextEncoder().encode(secret + body)
  );
  const expectedHex = Array.from(new Uint8Array(expectedSignature))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
  
  return signature === expectedHex;
}
```

### 4.3 Security Model

**Authentication Options**:

1. **Service Role Key**: Use Supabase service role key for internal calls
2. **Webhook Secret**: Use HMAC-SHA256 signed webhooks for external calls
3. **IP Whitelist**: Restrict to known IP addresses (if applicable)

**Authorization**:
- Handler validates `org_id` from event payload
- Handler checks that work order exists and belongs to org
- Handler uses service role for database operations (bypasses RLS)

**Idempotency**:
- Uses `event_id` as deduplication key
- Stores idempotency keys in `idempotency_keys` table (from Epic 14)
- TTL: 24 hours

### 4.4 Error Handling and Retries

**Error Scenarios**:
1. **Invalid Event**: Return 400 Bad Request
2. **Work Order Not Found**: Log warning, return success (idempotent)
3. **Dispatch Job Creation Failed**: Return 500, allow retry
4. **Database Error**: Return 500, allow retry

**Retry Strategy**:
- Webhook caller should implement exponential backoff
- Handler is idempotent, safe to retry
- Failed events can be manually reprocessed via admin UI

---

## 5. Story DISP-067: System-Suggested Time Windows Generation Rules

### 5.1 Algorithm Overview

**Purpose**: Generate sensible default appointment windows when none are provided

**Inputs**:
- `dispatch_job` (with SLA, priority, location, required skills)
- Available technicians (with shifts, time-off, existing assignments)
- Service zones
- Business hours (org-level configuration)

**Outputs**:
- Array of `job_time_windows` with `source = 'system_suggested'`
- Exactly one can be selected at a time per job

### 5.2 Window Generation Algorithm

**Step 1: Determine Time Constraints**

```typescript
interface TimeConstraints {
  earliestStart: Date; // From SLA or now()
  latestCompletion: Date; // From SLA or default (e.g., +7 days)
  businessHoursStart: number; // e.g., 8 (8 AM)
  businessHoursEnd: number; // e.g., 17 (5 PM)
  excludeWeekends: boolean;
  excludeHolidays: boolean;
}

function determineTimeConstraints(job: DispatchJob, orgConfig: OrgConfig): TimeConstraints {
  const now = new Date();
  const earliestStart = job.sla_earliest_start_at 
    ? new Date(job.sla_earliest_start_at)
    : now;
  
  const latestCompletion = job.sla_latest_completion_at
    ? new Date(job.sla_latest_completion_at)
    : addDays(now, 7); // Default: 7 days from now

  return {
    earliestStart,
    latestCompletion,
    businessHoursStart: orgConfig.business_hours_start || 8,
    businessHoursEnd: orgConfig.business_hours_end || 17,
    excludeWeekends: orgConfig.exclude_weekends !== false,
    excludeHolidays: orgConfig.exclude_holidays !== false
  };
}
```

**Step 2: Find Available Slots**

```typescript
interface AvailableSlot {
  start: Date;
  end: Date;
  technicianId?: string; // If slot is tied to specific technician
  score: number; // Higher = better
}

async function findAvailableSlots(
  supabase: SupabaseClient,
  job: DispatchJob,
  constraints: TimeConstraints
): Promise<AvailableSlot[]> {
  const slots: AvailableSlot[] = [];
  const currentDate = new Date(constraints.earliestStart);
  const endDate = new Date(constraints.latestCompletion);

  // Iterate through each day
  while (currentDate <= endDate) {
    // Skip weekends if configured
    if (constraints.excludeWeekends && isWeekend(currentDate)) {
      currentDate = addDays(currentDate, 1);
      continue;
    }

    // Skip holidays if configured
    if (constraints.excludeHolidays && await isHoliday(supabase, job.org_id, currentDate)) {
      currentDate = addDays(currentDate, 1);
      continue;
    }

    // Find available technicians for this day
    const availableTechnicians = await findAvailableTechniciansForDay(
      supabase,
      job,
      currentDate
    );

    if (availableTechnicians.length === 0) {
      currentDate = addDays(currentDate, 1);
      continue;
    }

    // Generate slots for this day
    const daySlots = generateDaySlots(
      currentDate,
      constraints,
      job.estimated_duration_minutes || 60,
      availableTechnicians
    );

    slots.push(...daySlots);

    currentDate = addDays(currentDate, 1);
  }

  // Score and sort slots
  return scoreSlots(slots, job, constraints);
}
```

**Step 3: Generate Day Slots**

```typescript
function generateDaySlots(
  date: Date,
  constraints: TimeConstraints,
  durationMinutes: number,
  technicians: Technician[]
): AvailableSlot[] {
  const slots: AvailableSlot[] = [];
  const dayStart = setHours(setMinutes(date, 0), constraints.businessHoursStart);
  const dayEnd = setHours(setMinutes(date, 0), constraints.businessHoursEnd);
  const slotDuration = durationMinutes + 30; // Add buffer for travel

  let currentSlotStart = dayStart;

  while (currentSlotStart < dayEnd) {
    const currentSlotEnd = addMinutes(currentSlotStart, slotDuration);

    if (currentSlotEnd > dayEnd) {
      break;
    }

    // Check if any technician is available for this slot
    const availableTechs = technicians.filter(tech => 
      isTechnicianAvailable(tech, currentSlotStart, currentSlotEnd)
    );

    if (availableTechs.length > 0) {
      slots.push({
        start: currentSlotStart,
        end: currentSlotEnd,
        technicianId: availableTechs[0].id, // Prefer first available
        score: 0 // Will be scored later
      });
    }

    // Move to next slot (30-minute increments)
    currentSlotStart = addMinutes(currentSlotStart, 30);
  }

  return slots;
}
```

**Step 4: Score Slots**

```typescript
function scoreSlots(
  slots: AvailableSlot[],
  job: DispatchJob,
  constraints: TimeConstraints
): AvailableSlot[] {
  const now = new Date();

  return slots.map(slot => {
    let score = 100;

    // Prefer earlier slots (within reason)
    const hoursUntilSlot = differenceInHours(slot.start, now);
    if (hoursUntilSlot < 24) {
      score += 20; // Same day or next day
    } else if (hoursUntilSlot < 48) {
      score += 10; // Next 2 days
    }

    // Prefer slots closer to SLA deadline (if tight)
    if (job.sla_latest_completion_at) {
      const slaDeadline = new Date(job.sla_latest_completion_at);
      const hoursUntilDeadline = differenceInHours(slaDeadline, slot.start);
      if (hoursUntilDeadline < 24) {
        score += 15; // Urgent
      }
    }

    // Prefer morning slots (8-12)
    const hour = slot.start.getHours();
    if (hour >= 8 && hour < 12) {
      score += 10;
    }

    // Penalize late slots (after 4 PM)
    if (hour >= 16) {
      score -= 5;
    }

    // Prefer slots with specific technician (if assigned)
    if (slot.technicianId) {
      score += 5;
    }

    return { ...slot, score };
  }).sort((a, b) => b.score - a.score);
}
```

**Step 5: Generate Windows**

```typescript
async function generateTimeWindows(
  supabase: SupabaseClient,
  job: DispatchJob
): Promise<JobTimeWindow[]> {
  // Get org configuration
  const orgConfig = await getOrgConfig(supabase, job.org_id);

  // Determine constraints
  const constraints = determineTimeConstraints(job, orgConfig);

  // Find available slots
  const slots = await findAvailableSlots(supabase, job, constraints);

  if (slots.length === 0) {
    // No slots available
    return [];
  }

  // Take top N slots (e.g., 5)
  const topSlots = slots.slice(0, 5);

  // Convert to job_time_windows
  const windows: JobTimeWindow[] = topSlots.map((slot, index) => ({
    org_id: job.org_id,
    dispatch_job_id: job.id,
    start_at: slot.start.toISOString(),
    end_at: slot.end.toISOString(),
    source: 'system_suggested',
    is_selected: false,
    metadata: {
      score: slot.score,
      rank: index + 1,
      technician_id: slot.technicianId
    }
  }));

  return windows;
}
```

### 5.3 Edge Cases

**Case 1: SLA Too Tight**

```typescript
if (constraints.latestCompletion < addHours(now, 2)) {
  // SLA requires completion within 2 hours
  // Check for emergency availability
  const emergencySlots = await findEmergencySlots(supabase, job, constraints);
  if (emergencySlots.length === 0) {
    return {
      windows: [],
      reason: 'SLA too tight - no available technicians within required timeframe'
    };
  }
  return { windows: emergencySlots };
}
```

**Case 2: No Technicians Available**

```typescript
const availableTechnicians = await findAvailableTechniciansForDay(...);
if (availableTechnicians.length === 0) {
  return {
    windows: [],
    reason: 'No technicians available in service zone with required skills'
  };
}
```

**Case 3: Time-Off Conflicts**

```typescript
// Already handled in isTechnicianAvailable() check
// Technicians with time-off are excluded from available list
```

**Case 4: Zone Mismatch**

```typescript
// Already handled in findAvailableTechniciansForDay()
// Only technicians in job's service zone are considered
```

**Case 5: Emergency Priority**

```typescript
if (job.priority === 'emergency') {
  // Generate immediate slots (next 2 hours)
  const emergencyConstraints = {
    ...constraints,
    earliestStart: now,
    latestCompletion: addHours(now, 2)
  };
  return await findAvailableSlots(supabase, job, emergencyConstraints);
}
```

### 5.4 Implementation

**Edge Function**: `generate-time-windows`

**Endpoint**: `POST /dispatch/jobs/:id/generate-time-windows`

**Request**:
```typescript
interface GenerateTimeWindowsRequest {
  max_windows?: number; // Default: 5
  include_technician_suggestions?: boolean; // Default: false
}
```

**Response**:
```typescript
interface GenerateTimeWindowsResponse {
  windows: Array<{
    id: string;
    start_at: string;
    end_at: string;
    source: 'system_suggested';
    score: number;
    technician_suggestion?: {
      technician_id: string;
      technician_name: string;
    };
  }>;
  reason?: string; // If no windows generated
}
```

### 5.5 Integration Points

**Job Creation**: Auto-generate windows if none provided

**Customer Portal**: Use generated windows for booking UI

**Dispatcher UI**: Show suggested windows with "Accept" button

---

## 6. Story DISP-068: Re-Optimization Triggers on Schedule/Availability Change

### 6.1 Trigger Events

**Events That Invalidate Routes**:

1. **Assignment Moved**: `job_assignments.scheduled_start_at` or `scheduled_end_at` changed
2. **Assignment Canceled**: `job_assignments.status` changed to 'canceled'
3. **Technician Time-Off Added**: New `technician_time_off` record created
4. **Technician Shift Changed**: `technician_shifts` created, updated, or deleted
5. **Emergency Job Inserted**: New `dispatch_job` with `priority = 'emergency'` created and assigned
6. **Job Canceled**: `dispatch_job.status` changed to 'canceled'
7. **Job Priority Changed**: `dispatch_job.priority` changed (especially to 'emergency')

### 6.2 Trigger Strategy

**Approach**: Database triggers + Edge Function queue

**Two Modes**:
1. **Automatic Re-optimization**: Queue optimization run immediately
2. **Flag for Review**: Mark route as "needs review", dispatcher triggers optimization

**Decision Logic**:

```typescript
interface ReOptimizationDecision {
  action: 'auto_optimize' | 'flag_for_review';
  affectedTechnicians: string[];
  affectedDate: string; // ISO date
  reason: string;
}

function decideReOptimizationAction(
  eventType: string,
  eventData: any
): ReOptimizationDecision {
  switch (eventType) {
    case 'assignment_moved':
      // Auto-optimize if moved > 30 minutes
      const timeDiff = Math.abs(
        differenceInMinutes(
          new Date(eventData.new_scheduled_start_at),
          new Date(eventData.old_scheduled_start_at)
        )
      );
      if (timeDiff > 30) {
        return {
          action: 'auto_optimize',
          affectedTechnicians: [eventData.technician_id],
          affectedDate: format(new Date(eventData.new_scheduled_start_at), 'yyyy-MM-dd'),
          reason: `Assignment moved by ${timeDiff} minutes`
        };
      }
      return {
        action: 'flag_for_review',
        affectedTechnicians: [eventData.technician_id],
        affectedDate: format(new Date(eventData.new_scheduled_start_at), 'yyyy-MM-dd'),
        reason: 'Assignment moved'
      };

    case 'assignment_canceled':
      return {
        action: 'auto_optimize',
        affectedTechnicians: [eventData.technician_id],
        affectedDate: format(new Date(eventData.scheduled_start_at), 'yyyy-MM-dd'),
        reason: 'Assignment canceled'
      };

    case 'technician_time_off_added':
      return {
        action: 'auto_optimize',
        affectedTechnicians: [eventData.technician_id],
        affectedDate: format(new Date(eventData.starts_at), 'yyyy-MM-dd'),
        reason: 'Technician time-off added'
      };

    case 'emergency_job_inserted':
      return {
        action: 'auto_optimize',
        affectedTechnicians: eventData.affected_technician_ids,
        affectedDate: format(new Date(eventData.scheduled_start_at), 'yyyy-MM-dd'),
        reason: 'Emergency job inserted'
      };

    case 'job_priority_changed':
      if (eventData.new_priority === 'emergency') {
        return {
          action: 'auto_optimize',
          affectedTechnicians: eventData.assigned_technician_ids,
          affectedDate: format(new Date(eventData.scheduled_start_at), 'yyyy-MM-dd'),
          reason: 'Job priority changed to emergency'
        };
      }
      return {
        action: 'flag_for_review',
        affectedTechnicians: eventData.assigned_technician_ids,
        affectedDate: format(new Date(eventData.scheduled_start_at), 'yyyy-MM-dd'),
        reason: 'Job priority changed'
      };

    default:
      return {
        action: 'flag_for_review',
        affectedTechnicians: [],
        affectedDate: '',
        reason: 'Unknown event'
      };
  }
}
```

### 6.3 Debounce and Coalesce Strategy

**Problem**: Multiple rapid changes can trigger multiple optimizations

**Solution**: Debounce optimization requests by technician + date

**Implementation**:

```sql
-- Table to track pending optimizations
CREATE TABLE IF NOT EXISTS re_optimization_queue (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  technician_id UUID REFERENCES technician_profiles(id) ON DELETE CASCADE,
  date DATE NOT NULL,
  reason TEXT NOT NULL,
  priority INTEGER NOT NULL DEFAULT 0, -- Higher = more urgent
  status TEXT NOT NULL DEFAULT 'queued', -- 'queued', 'processing', 'completed', 'failed'
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  processed_at TIMESTAMPTZ,
  UNIQUE(org_id, technician_id, date, status) -- Prevent duplicates
);

CREATE INDEX idx_re_optimization_queue_org_status ON re_optimization_queue(org_id, status, created_at);
CREATE INDEX idx_re_optimization_queue_tech_date ON re_optimization_queue(technician_id, date);
```

**Debounce Logic**:

```typescript
async function queueReOptimization(
  supabase: SupabaseClient,
  decision: ReOptimizationDecision
): Promise<void> {
  // Check for existing queued optimization
  const { data: existing } = await supabase
    .from('re_optimization_queue')
    .select('id, created_at')
    .eq('org_id', decision.orgId)
    .eq('technician_id', decision.affectedTechnicians[0])
    .eq('date', decision.affectedDate)
    .eq('status', 'queued')
    .single();

  if (existing) {
    // Update existing queue entry (coalesce)
    const ageMinutes = differenceInMinutes(new Date(), new Date(existing.created_at));
    if (ageMinutes < 5) {
      // Within debounce window, update reason
      await supabase
        .from('re_optimization_queue')
        .update({
          reason: `${existing.reason}; ${decision.reason}`,
          priority: Math.max(decision.priority || 0, existing.priority || 0)
        })
        .eq('id', existing.id);
      return;
    }
  }

  // Create new queue entry
  for (const technicianId of decision.affectedTechnicians) {
    await supabase
      .from('re_optimization_queue')
      .upsert({
        org_id: decision.orgId,
        technician_id: technicianId,
        date: decision.affectedDate,
        reason: decision.reason,
        priority: decision.priority || 0,
        status: 'queued'
      }, {
        onConflict: 'org_id,technician_id,date,status'
      });
  }

  // Trigger processing (with delay for debouncing)
  setTimeout(() => {
    processReOptimizationQueue(supabase, decision.orgId);
  }, 5000); // 5 second debounce
}
```

### 6.4 Database Triggers

**Assignment Update Trigger**:

```sql
CREATE OR REPLACE FUNCTION trigger_re_optimization_on_assignment_change()
RETURNS TRIGGER AS $$
DECLARE
  v_org_id UUID;
  v_technician_id UUID;
  v_date DATE;
  v_time_diff_minutes INTEGER;
BEGIN
  -- Only trigger on significant changes
  IF (OLD.scheduled_start_at IS DISTINCT FROM NEW.scheduled_start_at) OR
     (OLD.scheduled_end_at IS DISTINCT FROM NEW.scheduled_end_at) OR
     (OLD.status IS DISTINCT FROM NEW.status) THEN
    
    v_org_id := NEW.org_id;
    v_technician_id := NEW.technician_id;
    v_date := DATE(NEW.scheduled_start_at);

    -- Calculate time difference
    IF OLD.scheduled_start_at IS NOT NULL AND NEW.scheduled_start_at IS NOT NULL THEN
      v_time_diff_minutes := EXTRACT(EPOCH FROM (NEW.scheduled_start_at - OLD.scheduled_start_at)) / 60;
    ELSE
      v_time_diff_minutes := 999; -- New assignment or canceled
    END IF;

    -- Queue re-optimization
    INSERT INTO re_optimization_queue (
      org_id,
      technician_id,
      date,
      reason,
      priority
    ) VALUES (
      v_org_id,
      v_technician_id,
      v_date,
      CASE 
        WHEN NEW.status = 'canceled' THEN 'Assignment canceled'
        WHEN ABS(v_time_diff_minutes) > 30 THEN format('Assignment moved by %s minutes', ABS(v_time_diff_minutes))
        ELSE 'Assignment updated'
      END,
      CASE 
        WHEN NEW.status = 'canceled' THEN 10
        WHEN ABS(v_time_diff_minutes) > 60 THEN 8
        ELSE 5
      END
    )
    ON CONFLICT (org_id, technician_id, date, status) 
    DO UPDATE SET
      reason = re_optimization_queue.reason || '; ' || EXCLUDED.reason,
      priority = GREATEST(re_optimization_queue.priority, EXCLUDED.priority),
      created_at = now();
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_re_optimization_assignment_change
  AFTER INSERT OR UPDATE ON job_assignments
  FOR EACH ROW
  EXECUTE FUNCTION trigger_re_optimization_on_assignment_change();
```

**Time-Off Trigger**:

```sql
CREATE OR REPLACE FUNCTION trigger_re_optimization_on_time_off()
RETURNS TRIGGER AS $$
DECLARE
  v_date DATE;
BEGIN
  v_date := DATE(NEW.starts_at);

  -- Queue re-optimization for each day in time-off range
  WHILE v_date <= DATE(NEW.ends_at) LOOP
    INSERT INTO re_optimization_queue (
      org_id,
      technician_id,
      date,
      reason,
      priority
    ) VALUES (
      NEW.org_id,
      NEW.technician_id,
      v_date,
      'Technician time-off added',
      8
    )
    ON CONFLICT (org_id, technician_id, date, status) 
    DO UPDATE SET
      reason = re_optimization_queue.reason || '; ' || EXCLUDED.reason,
      priority = GREATEST(re_optimization_queue.priority, EXCLUDED.priority);

    v_date := v_date + INTERVAL '1 day';
  END LOOP;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_re_optimization_time_off
  AFTER INSERT ON technician_time_off
  FOR EACH ROW
  EXECUTE FUNCTION trigger_re_optimization_on_time_off();
```

### 6.5 Processing Queue

**Edge Function**: `process-re-optimization-queue`

**Cron Schedule**: Every 5 minutes

**Implementation**:

```typescript
Deno.serve(async (req) => {
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
  );

  // Fetch queued optimizations (grouped by technician + date)
  const { data: queueItems } = await supabase
    .from('re_optimization_queue')
    .select('*')
    .eq('status', 'queued')
    .order('priority', { ascending: false })
    .order('created_at', { ascending: true })
    .limit(100);

  if (!queueItems || queueItems.length === 0) {
    return new Response(JSON.stringify({ message: 'No items to process' }));
  }

  // Group by technician + date
  const grouped = queueItems.reduce((acc, item) => {
    const key = `${item.technician_id}:${item.date}`;
    if (!acc[key]) {
      acc[key] = [];
    }
    acc[key].push(item);
    return acc;
  }, {} as Record<string, typeof queueItems>);

  // Process each group
  for (const [key, items] of Object.entries(grouped)) {
    const [technicianId, date] = key.split(':');
    const highestPriority = Math.max(...items.map(i => i.priority || 0));

    // Mark as processing
    await supabase
      .from('re_optimization_queue')
      .update({ status: 'processing' })
      .in('id', items.map(i => i.id));

    try {
      // Call optimize route function
      const { data: result, error } = await supabase.functions.invoke('optimize-route', {
        body: {
          technician_id: technicianId,
          date: date,
          strategy: 'time_minimization',
          respect_locks: false // Allow re-optimization to override locks
        }
      });

      if (error) {
        throw error;
      }

      // Mark as completed
      await supabase
        .from('re_optimization_queue')
        .update({
          status: 'completed',
          processed_at: new Date().toISOString()
        })
        .in('id', items.map(i => i.id));

    } catch (error) {
      // Mark as failed
      await supabase
        .from('re_optimization_queue')
        .update({
          status: 'failed',
          processed_at: new Date().toISOString()
        })
        .in('id', items.map(i => i.id));
    }
  }

  return new Response(JSON.stringify({ processed: Object.keys(grouped).length }));
});
```

### 6.6 Flag for Review Mode

**Route Status Table**:

```sql
CREATE TABLE IF NOT EXISTS route_review_flags (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  route_plan_id UUID REFERENCES route_plans(id) ON DELETE CASCADE,
  technician_id UUID REFERENCES technician_profiles(id) ON DELETE CASCADE,
  date DATE NOT NULL,
  reason TEXT NOT NULL,
  severity TEXT NOT NULL DEFAULT 'medium', -- 'low', 'medium', 'high'
  acknowledged_at TIMESTAMPTZ,
  acknowledged_by_user_id UUID REFERENCES auth.users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(org_id, route_plan_id, date)
);

CREATE INDEX idx_route_review_flags_org_unacknowledged 
  ON route_review_flags(org_id, acknowledged_at) 
  WHERE acknowledged_at IS NULL;
```

**UI Indicator**: Show badge/alert in schedule board for routes needing review

---

## 7. Story DISP-069: SLA and Time-Window Conflict Detection

### 7.1 Constraint Policy

**Hard Constraints** (Block):
- Assignment outside selected time window
- Assignment before SLA earliest start
- Assignment after SLA latest completion
- Assignment conflicts with technician time-off
- Assignment conflicts with technician shift boundaries

**Soft Constraints** (Warn):
- Assignment near SLA deadline (< 2 hours buffer)
- Assignment creates tight travel time (< 15 minutes between jobs)
- Assignment exceeds technician daily capacity
- Assignment outside business hours (if configured)

### 7.2 Validation Function

**Database Function**:

```sql
CREATE OR REPLACE FUNCTION validate_assignment_constraints(
  p_assignment_id UUID,
  p_scheduled_start_at TIMESTAMPTZ,
  p_scheduled_end_at TIMESTAMPTZ,
  p_technician_id UUID,
  p_dispatch_job_id UUID
)
RETURNS TABLE (
  constraint_type TEXT,
  severity TEXT,
  message TEXT,
  blocks_assignment BOOLEAN
) AS $$
DECLARE
  v_job RECORD;
  v_technician RECORD;
  v_selected_window RECORD;
  v_sla_earliest_start TIMESTAMPTZ;
  v_sla_latest_completion TIMESTAMPTZ;
  v_time_window_start TIMESTAMPTZ;
  v_time_window_end TIMESTAMPTZ;
  v_prev_assignment RECORD;
  v_next_assignment RECORD;
BEGIN
  -- Fetch job details
  SELECT * INTO v_job
  FROM dispatch_jobs
  WHERE id = p_dispatch_job_id;

  -- Fetch technician details
  SELECT * INTO v_technician
  FROM technician_profiles
  WHERE id = p_technician_id;

  -- Fetch selected time window
  SELECT * INTO v_selected_window
  FROM job_time_windows
  WHERE dispatch_job_id = p_dispatch_job_id
    AND is_selected = true
  LIMIT 1;

  -- Set SLA constraints
  v_sla_earliest_start := v_job.sla_earliest_start_at;
  v_sla_latest_completion := v_job.sla_latest_completion_at;

  -- Set time window constraints
  IF v_selected_window IS NOT NULL THEN
    v_time_window_start := v_selected_window.start_at;
    v_time_window_end := v_selected_window.end_at;
  END IF;

  -- Check 1: Assignment before SLA earliest start (HARD)
  IF v_sla_earliest_start IS NOT NULL AND p_scheduled_start_at < v_sla_earliest_start THEN
    RETURN QUERY SELECT 
      'sla_earliest_start'::TEXT,
      'error'::TEXT,
      format('Assignment starts before SLA earliest start (%s)', v_sla_earliest_start)::TEXT,
      true::BOOLEAN;
  END IF;

  -- Check 2: Assignment after SLA latest completion (HARD)
  IF v_sla_latest_completion IS NOT NULL AND p_scheduled_end_at > v_sla_latest_completion THEN
    RETURN QUERY SELECT 
      'sla_latest_completion'::TEXT,
      'error'::TEXT,
      format('Assignment ends after SLA latest completion (%s)', v_sla_latest_completion)::TEXT,
      true::BOOLEAN;
  END IF;

  -- Check 3: Assignment outside selected time window (HARD)
  IF v_time_window_start IS NOT NULL AND v_time_window_end IS NOT NULL THEN
    IF p_scheduled_start_at < v_time_window_start OR p_scheduled_end_at > v_time_window_end THEN
      RETURN QUERY SELECT 
        'time_window'::TEXT,
        'error'::TEXT,
        format('Assignment outside selected time window (%s - %s)', v_time_window_start, v_time_window_end)::TEXT,
        true::BOOLEAN;
    END IF;
  END IF;

  -- Check 4: Assignment conflicts with time-off (HARD)
  IF EXISTS (
    SELECT 1 FROM technician_time_off
    WHERE technician_id = p_technician_id
      AND starts_at < p_scheduled_end_at
      AND ends_at > p_scheduled_start_at
  ) THEN
    RETURN QUERY SELECT 
      'technician_time_off'::TEXT,
      'error'::TEXT,
      'Assignment conflicts with technician time-off'::TEXT,
      true::BOOLEAN;
  END IF;

  -- Check 5: Assignment conflicts with shift (HARD)
  IF NOT EXISTS (
    SELECT 1 FROM technician_shifts
    WHERE technician_id = p_technician_id
      AND is_active = true
      AND starts_at <= p_scheduled_start_at
      AND ends_at >= p_scheduled_end_at
  ) THEN
    RETURN QUERY SELECT 
      'technician_shift'::TEXT,
      'error'::TEXT,
      'Assignment outside technician shift hours'::TEXT,
      true::BOOLEAN;
  END IF;

  -- Check 6: Assignment near SLA deadline (SOFT)
  IF v_sla_latest_completion IS NOT NULL THEN
    IF p_scheduled_end_at > (v_sla_latest_completion - INTERVAL '2 hours') THEN
      RETURN QUERY SELECT 
        'sla_deadline_buffer'::TEXT,
        'warning'::TEXT,
        format('Assignment ends within 2 hours of SLA deadline (%s)', v_sla_latest_completion)::TEXT,
        false::BOOLEAN;
    END IF;
  END IF;

  -- Check 7: Tight travel time (SOFT)
  SELECT * INTO v_prev_assignment
  FROM job_assignments
  WHERE technician_id = p_technician_id
    AND scheduled_end_at <= p_scheduled_start_at
    AND status NOT IN ('canceled', 'completed')
  ORDER BY scheduled_end_at DESC
  LIMIT 1;

  IF v_prev_assignment IS NOT NULL THEN
    IF EXTRACT(EPOCH FROM (p_scheduled_start_at - v_prev_assignment.scheduled_end_at)) / 60 < 15 THEN
      RETURN QUERY SELECT 
        'travel_time'::TEXT,
        'warning'::TEXT,
        format('Less than 15 minutes travel time from previous assignment')::TEXT,
        false::BOOLEAN;
    END IF;
  END IF;

  -- Check 8: Exceeds daily capacity (SOFT)
  -- (Implementation would calculate total scheduled minutes for the day)

  RETURN;
END;
$$ LANGUAGE plpgsql;
```

### 7.3 API Integration

**Assignment Create/Update Endpoint**: Call validation before saving

```typescript
// In assignment create/update Edge Function
const validationResults = await supabase.rpc('validate_assignment_constraints', {
  p_assignment_id: assignmentId || null,
  p_scheduled_start_at: requestBody.scheduled_start_at,
  p_scheduled_end_at: requestBody.scheduled_end_at,
  p_technician_id: requestBody.technician_id,
  p_dispatch_job_id: requestBody.dispatch_job_id
});

const blockingErrors = validationResults.data?.filter((r: any) => r.blocks_assignment) || [];
const warnings = validationResults.data?.filter((r: any) => !r.blocks_assignment) || [];

if (blockingErrors.length > 0) {
  return errorResponse('Validation failed', 400, {
    errors: blockingErrors,
    warnings: warnings
  });
}

// Proceed with assignment creation/update
// Return warnings in response
```

### 7.4 UI Integration

**Schedule Board**: Show warning badges on assignments

**Assignment Form**: Show validation errors/warnings in real-time

**Capacity View**: Highlight SLA risk warnings

---

## 8. Story DISP-070: Technician Location Ingestion and Storage

### 8.1 Storage Model

**Table**: `technician_locations`

```sql
CREATE TABLE IF NOT EXISTS technician_locations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  technician_id UUID NOT NULL REFERENCES technician_profiles(id) ON DELETE CASCADE,
  latitude NUMERIC(10, 8) NOT NULL,
  longitude NUMERIC(11, 8) NOT NULL,
  accuracy_meters NUMERIC(6, 2), -- GPS accuracy
  heading_degrees NUMERIC(5, 2), -- Direction of travel (0-360)
  speed_meters_per_second NUMERIC(6, 2), -- Speed
  altitude_meters NUMERIC(8, 2), -- Altitude (if available)
  source TEXT NOT NULL DEFAULT 'mobile_app', -- 'mobile_app', 'manual', 'estimated'
  recorded_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(org_id, technician_id) -- Only latest location per technician
);

CREATE INDEX idx_technician_locations_org_tech ON technician_locations(org_id, technician_id);
CREATE INDEX idx_technician_locations_org_recorded ON technician_locations(org_id, recorded_at DESC);

-- Optional: Location history table
CREATE TABLE IF NOT EXISTS technician_location_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  technician_id UUID NOT NULL REFERENCES technician_profiles(id) ON DELETE CASCADE,
  latitude NUMERIC(10, 8) NOT NULL,
  longitude NUMERIC(11, 8) NOT NULL,
  accuracy_meters NUMERIC(6, 2),
  heading_degrees NUMERIC(5, 2),
  speed_meters_per_second NUMERIC(6, 2),
  altitude_meters NUMERIC(8, 2),
  source TEXT NOT NULL DEFAULT 'mobile_app',
  recorded_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_technician_location_history_org_tech_recorded 
  ON technician_location_history(org_id, technician_id, recorded_at DESC);

-- Partition by month (optional, for large datasets)
-- CREATE TABLE technician_location_history_2024_01 PARTITION OF technician_location_history
--   FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

**Trigger to Maintain Latest Location**:

```sql
CREATE OR REPLACE FUNCTION update_technician_latest_location()
RETURNS TRIGGER AS $$
BEGIN
  -- Upsert into latest location table
  INSERT INTO technician_locations (
    org_id,
    technician_id,
    latitude,
    longitude,
    accuracy_meters,
    heading_degrees,
    speed_meters_per_second,
    altitude_meters,
    source,
    recorded_at
  ) VALUES (
    NEW.org_id,
    NEW.technician_id,
    NEW.latitude,
    NEW.longitude,
    NEW.accuracy_meters,
    NEW.heading_degrees,
    NEW.speed_meters_per_second,
    NEW.altitude_meters,
    NEW.source,
    NEW.recorded_at
  )
  ON CONFLICT (org_id, technician_id) 
  DO UPDATE SET
    latitude = EXCLUDED.latitude,
    longitude = EXCLUDED.longitude,
    accuracy_meters = EXCLUDED.accuracy_meters,
    heading_degrees = EXCLUDED.heading_degrees,
    speed_meters_per_second = EXCLUDED.speed_meters_per_second,
    altitude_meters = EXCLUDED.altitude_meters,
    source = EXCLUDED.source,
    recorded_at = EXCLUDED.recorded_at,
    created_at = now();

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_latest_location
  AFTER INSERT ON technician_location_history
  FOR EACH ROW
  EXECUTE FUNCTION update_technician_latest_location();
```

### 8.2 Location Ingestion API

**Endpoint**: `POST /dispatch/technicians/:id/location`

**Authentication**: Technician (self-scope) or service role

**Request**:

```typescript
interface UpdateLocationRequest {
  latitude: number;
  longitude: number;
  accuracy_meters?: number;
  heading_degrees?: number;
  speed_meters_per_second?: number;
  altitude_meters?: number;
  source?: 'mobile_app' | 'manual' | 'estimated';
  recorded_at?: string; // ISO 8601, defaults to now()
}
```

**Implementation**:

```typescript
// File: supabase/functions/update-technician-location/index.ts

Deno.serve(async (req) => {
  const { technicianId } = await req.json();
  const authHeader = req.headers.get('Authorization');
  
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_ANON_KEY')!,
    { global: { headers: { Authorization: authHeader } } }
  );

  const { data: { user }, error: userError } = await supabase.auth.getUser();
  if (userError || !user) {
    return errorResponse('Unauthorized', 401);
  }

  // Verify technician owns this location update
  const { data: technician } = await supabase
    .from('technician_profiles')
    .select('id, org_id, user_id')
    .eq('id', technicianId)
    .eq('user_id', user.id)
    .single();

  if (!technician) {
    return errorResponse('Forbidden', 403);
  }

  const requestBody: UpdateLocationRequest = await req.json();

  // Validate coordinates
  if (requestBody.latitude < -90 || requestBody.latitude > 90) {
    return errorResponse('Invalid latitude', 400);
  }
  if (requestBody.longitude < -180 || requestBody.longitude > 180) {
    return errorResponse('Invalid longitude', 400);
  }

  // Insert into history
  const { data: location, error } = await supabase
    .from('technician_location_history')
    .insert({
      org_id: technician.org_id,
      technician_id: technicianId,
      latitude: requestBody.latitude,
      longitude: requestBody.longitude,
      accuracy_meters: requestBody.accuracy_meters,
      heading_degrees: requestBody.heading_degrees,
      speed_meters_per_second: requestBody.speed_meters_per_second,
      altitude_meters: requestBody.altitude_meters,
      source: requestBody.source || 'mobile_app',
      recorded_at: requestBody.recorded_at || new Date().toISOString()
    })
    .select()
    .single();

  if (error) {
    return errorResponse('Failed to update location', 500);
  }

  return successResponse({
    location_id: location.id,
    updated_at: location.recorded_at
  });
});
```

### 8.3 Update Frequency Guidelines

**Recommended Frequencies**:

- **Active Assignment**: Every 30-60 seconds
- **En Route**: Every 60-120 seconds
- **On Site**: Every 5-10 minutes (or on status change)
- **Idle**: Every 15-30 minutes

**Battery Optimization**:

- Reduce frequency when battery < 20%
- Pause updates when app in background (configurable)
- Use significant location changes API when available

**Network Optimization**:

- Batch updates when offline, send when online
- Compress location data
- Use delta encoding (only send if moved > 50 meters)

### 8.4 Privacy and Retention

**Privacy Constraints**:

- Technicians can opt-out of location tracking (with org approval)
- Location data only accessible to dispatchers/admins
- Technicians can view their own location history

**Retention Policy**:

- Latest location: Indefinite (until updated)
- Location history: 90 days (configurable per org)
- Auto-delete old history via cron job

**RLS Policies**:

```sql
-- Technicians can view their own location
CREATE POLICY "technician_locations_self_read"
ON technician_locations FOR SELECT
USING (
  org_id = get_user_org_id() AND
  technician_id IN (
    SELECT id FROM technician_profiles 
    WHERE user_id = auth.uid() AND org_id = get_user_org_id()
  )
);

-- Dispatchers/admins can view all locations
CREATE POLICY "technician_locations_dispatcher_read"
ON technician_locations FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
);

-- Technicians can update their own location
CREATE POLICY "technician_locations_self_update"
ON technician_locations FOR INSERT
WITH CHECK (
  org_id = get_user_org_id() AND
  technician_id IN (
    SELECT id FROM technician_profiles 
    WHERE user_id = auth.uid() AND org_id = get_user_org_id()
  )
);
```

### 8.5 Map Integration

**Real-time Subscription**: Subscribe to `technician_locations` table

**Display**: Show technician markers on map with:
- Current position
- Heading (arrow direction)
- Speed (if available)
- Last update time

---

## 9. Story DISP-071: Calendar Token Refresh and Expiration Handling

### 9.1 Token Storage Strategy

**Secure Storage**: Use Supabase Vault or application-level encryption

**Table**: `calendar_integrations` (from Epic 9)

**Fields**:
- `access_token` (encrypted)
- `refresh_token` (encrypted)
- `token_expires_at` (TIMESTAMPTZ)
- `token_refreshed_at` (TIMESTAMPTZ)

### 9.2 Token Refresh Function

**Database Function**:

```sql
CREATE OR REPLACE FUNCTION refresh_calendar_token(
  p_integration_id UUID
)
RETURNS TABLE (
  success BOOLEAN,
  error_message TEXT
) AS $$
DECLARE
  v_integration RECORD;
  v_new_access_token TEXT;
  v_new_refresh_token TEXT;
  v_expires_in INTEGER;
BEGIN
  -- Fetch integration
  SELECT * INTO v_integration
  FROM calendar_integrations
  WHERE id = p_integration_id;

  IF NOT FOUND THEN
    RETURN QUERY SELECT false, 'Integration not found';
    RETURN;
  END IF;

  -- Call provider-specific refresh (would be done in Edge Function)
  -- This is a placeholder - actual implementation in Edge Function

  RETURN QUERY SELECT true, NULL;
END;
$$ LANGUAGE plpgsql;
```

**Edge Function**: `refresh-calendar-tokens`

**Cron Schedule**: Every hour

**Implementation**:

```typescript
// File: supabase/functions/refresh-calendar-tokens/index.ts

Deno.serve(async (req) => {
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
  );

  // Find integrations needing refresh (expires within 1 hour)
  const oneHourFromNow = new Date();
  oneHourFromNow.setHours(oneHourFromNow.getHours() + 1);

  const { data: integrations, error: fetchError } = await supabase
    .from('calendar_integrations')
    .select('*')
    .eq('is_active', true)
    .lte('token_expires_at', oneHourFromNow.toISOString());

  if (fetchError) {
    return errorResponse('Failed to fetch integrations', 500);
  }

  const results = [];

  for (const integration of integrations || []) {
    try {
      let newAccessToken: string;
      let newRefreshToken: string | undefined;
      let expiresIn: number;

      if (integration.provider === 'google') {
        const result = await refreshGoogleToken(integration.refresh_token);
        newAccessToken = result.access_token;
        newRefreshToken = result.refresh_token;
        expiresIn = result.expires_in;
      } else if (integration.provider === 'microsoft') {
        const result = await refreshMicrosoftToken(integration.refresh_token);
        newAccessToken = result.access_token;
        newRefreshToken = result.refresh_token;
        expiresIn = result.expires_in;
      } else {
        throw new Error(`Unknown provider: ${integration.provider}`);
      }

      // Calculate expiration time
      const expiresAt = new Date();
      expiresAt.setSeconds(expiresAt.getSeconds() + expiresIn);

      // Update integration
      const { error: updateError } = await supabase
        .from('calendar_integrations')
        .update({
          access_token: await encryptToken(newAccessToken),
          refresh_token: newRefreshToken ? await encryptToken(newRefreshToken) : integration.refresh_token,
          token_expires_at: expiresAt.toISOString(),
          token_refreshed_at: new Date().toISOString()
        })
        .eq('id', integration.id);

      if (updateError) {
        throw updateError;
      }

      results.push({ integration_id: integration.id, status: 'refreshed' });
    } catch (error) {
      // Mark integration as inactive if refresh fails
      await supabase
        .from('calendar_integrations')
        .update({
          is_active: false,
          last_error: error.message
        })
        .eq('id', integration.id);

      results.push({ 
        integration_id: integration.id, 
        status: 'failed', 
        error: error.message 
      });

      // Notify user (via notification system)
      await notifyUserOfTokenFailure(integration.org_id, integration.created_by_user_id);
    }
  }

  return successResponse({ results });
});

async function refreshGoogleToken(refreshToken: string): Promise<{
  access_token: string;
  refresh_token?: string;
  expires_in: number;
}> {
  const response = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      client_id: Deno.env.get('GOOGLE_CLIENT_ID')!,
      client_secret: Deno.env.get('GOOGLE_CLIENT_SECRET')!,
      refresh_token: refreshToken,
      grant_type: 'refresh_token'
    })
  });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(`Google token refresh failed: ${error.error_description || error.error}`);
  }

  return await response.json();
}

async function refreshMicrosoftToken(refreshToken: string): Promise<{
  access_token: string;
  refresh_token?: string;
  expires_in: number;
}> {
  const response = await fetch('https://login.microsoftonline.com/common/oauth2/v2.0/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      client_id: Deno.env.get('MICROSOFT_CLIENT_ID')!,
      client_secret: Deno.env.get('MICROSOFT_CLIENT_SECRET')!,
      refresh_token: refreshToken,
      grant_type: 'refresh_token',
      scope: 'https://graph.microsoft.com/Calendars.ReadWrite'
    })
  });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(`Microsoft token refresh failed: ${error.error_description || error.error}`);
  }

  return await response.json();
}

async function encryptToken(token: string): Promise<string> {
  // Use Supabase Vault or application-level encryption
  // Placeholder - implement based on encryption strategy
  return token; // In production, encrypt here
}

async function notifyUserOfTokenFailure(orgId: string, userId: string): Promise<void> {
  // Create notification for user
  // Implementation depends on notification system
}
```

### 9.3 Sync Function Integration

**Update Sync Functions**: Check token expiration before API calls

```typescript
async function ensureValidToken(
  supabase: SupabaseClient,
  integrationId: string
): Promise<string> {
  const { data: integration } = await supabase
    .from('calendar_integrations')
    .select('access_token, token_expires_at, refresh_token')
    .eq('id', integrationId)
    .single();

  if (!integration) {
    throw new Error('Integration not found');
  }

  // Check if token expires within 5 minutes
  const expiresAt = new Date(integration.token_expires_at);
  const fiveMinutesFromNow = new Date();
  fiveMinutesFromNow.setMinutes(fiveMinutesFromNow.getMinutes() + 5);

  if (expiresAt < fiveMinutesFromNow) {
    // Refresh token
    await supabase.functions.invoke('refresh-calendar-tokens', {
      body: { integration_id: integrationId }
    });

    // Fetch updated token
    const { data: updated } = await supabase
      .from('calendar_integrations')
      .select('access_token')
      .eq('id', integrationId)
      .single();

    return decryptToken(updated.access_token);
  }

  return decryptToken(integration.access_token);
}
```

### 9.4 Error Handling

**Token Refresh Failures**:

1. **Invalid Refresh Token**: Mark integration inactive, notify user
2. **Provider Error**: Retry with exponential backoff, mark inactive after 3 failures
3. **Network Error**: Retry, don't mark inactive

**User Notification**:

- Email notification when token refresh fails
- In-app notification in dispatch console
- Link to re-connect calendar integration

### 9.5 Troubleshooting Guide

**Common Issues**:

1. **Token Expired**: Check `token_expires_at`, verify refresh function runs
2. **Refresh Token Invalid**: User needs to re-authorize
3. **Provider Rate Limits**: Implement backoff, reduce sync frequency
4. **Encryption Errors**: Verify Vault configuration

**Documentation**: `docs/technical/calendar-integration-troubleshooting.md`

---

## 10. Next.js UI Components

### 10.1 Re-Optimization Queue Viewer

**File**: `app/dispatch/optimization/queue/page.tsx`

**Purpose**: View and manage re-optimization queue

**Components**:
- Queue table with status, reason, priority
- Manual trigger button
- Acknowledge flags

### 10.2 Constraint Validation UI

**File**: `app/dispatch/schedule-board/components/ConstraintWarnings.tsx`

**Purpose**: Display validation warnings/errors on schedule board

**Components**:
- Warning badges on assignments
- Error tooltips
- Constraint violation details

### 10.3 Technician Location Map

**File**: `app/dispatch/map-view/components/TechnicianLocationLayer.tsx`

**Purpose**: Display real-time technician locations on map

**Components**:
- Technician markers with heading
- Last update indicators
- Location history trail (optional)

### 10.4 Calendar Integration Status

**File**: `app/dispatch/settings/calendar/page.tsx`

**Purpose**: View and manage calendar integrations

**Components**:
- Integration status cards
- Token expiration warnings
- Re-connect buttons
- Sync status indicators

---

## 11. Implementation Checklist

### Story DISP-065: Work Order Synchronization Strategy

- [ ] **Architecture Decision**:
  - [ ] Decision documented (separate vs unified)
  - [ ] Schema changes implemented (if needed)
  - [ ] Foreign key added to `dispatch_jobs`

- [ ] **Event Contract**:
  - [ ] Event schema documented
  - [ ] Event types defined
  - [ ] Field mapping table created

- [ ] **Synchronization Flows**:
  - [ ] Happy paths documented
  - [ ] Error scenarios documented
  - [ ] Integration contract document created

### Story DISP-066: Work Order Event Handler

- [ ] **Event Handler**:
  - [ ] Edge Function created
  - [ ] Webhook endpoint implemented
  - [ ] Authentication/authorization implemented

- [ ] **Event Handlers**:
  - [ ] `handleWorkOrderCreated()` implemented
  - [ ] `handleWorkOrderUpdated()` implemented
  - [ ] `handleWorkOrderCanceled()` implemented
  - [ ] `handleWorkOrderCompleted()` implemented

- [ ] **Idempotency**:
  - [ ] Idempotency check implemented
  - [ ] Deduplication key documented

- [ ] **Security**:
  - [ ] Webhook signature verification implemented
  - [ ] Service role usage documented
  - [ ] Security model documented

### Story DISP-067: System-Suggested Time Windows

- [ ] **Algorithm**:
  - [ ] Window generation algorithm implemented
  - [ ] Slot finding logic implemented
  - [ ] Scoring logic implemented

- [ ] **Edge Cases**:
  - [ ] SLA too tight handled
  - [ ] No technicians available handled
  - [ ] Time-off conflicts handled
  - [ ] Zone mismatch handled
  - [ ] Emergency priority handled

- [ ] **Edge Function**:
  - [ ] `generate-time-windows` function created
  - [ ] API endpoint implemented
  - [ ] Integration with job creation

- [ ] **Documentation**:
  - [ ] Algorithm documented
  - [ ] Test scenarios documented

### Story DISP-068: Re-Optimization Triggers

- [ ] **Trigger Events**:
  - [ ] Event list documented
  - [ ] Decision logic implemented

- [ ] **Database Triggers**:
  - [ ] Assignment change trigger created
  - [ ] Time-off trigger created
  - [ ] Shift change trigger created

- [ ] **Queue System**:
  - [ ] `re_optimization_queue` table created
  - [ ] Debounce logic implemented
  - [ ] Coalesce logic implemented

- [ ] **Processing**:
  - [ ] Queue processor Edge Function created
  - [ ] Cron job scheduled
  - [ ] Flag for review mode implemented

### Story DISP-069: SLA and Time-Window Conflict Detection

- [ ] **Validation Function**:
  - [ ] `validate_assignment_constraints()` function created
  - [ ] Hard constraints implemented
  - [ ] Soft constraints implemented

- [ ] **API Integration**:
  - [ ] Validation called in assignment endpoints
  - [ ] Error responses include validation results
  - [ ] Warnings returned in responses

- [ ] **UI Integration**:
  - [ ] Warning badges displayed
  - [ ] Error messages shown
  - [ ] Constraint details displayed

- [ ] **Documentation**:
  - [ ] Constraint policy documented
  - [ ] Example scenarios documented

### Story DISP-070: Technician Location Ingestion

- [ ] **Storage Model**:
  - [ ] `technician_locations` table created
  - [ ] `technician_location_history` table created
  - [ ] Triggers created

- [ ] **API**:
  - [ ] Location update endpoint created
  - [ ] Authentication implemented
  - [ ] Validation implemented

- [ ] **Privacy**:
  - [ ] RLS policies created
  - [ ] Retention policy implemented
  - [ ] Privacy constraints documented

- [ ] **Map Integration**:
  - [ ] Real-time subscription implemented
  - [ ] Map markers displayed
  - [ ] Location history displayed (optional)

### Story DISP-071: Calendar Token Refresh

- [ ] **Token Storage**:
  - [ ] Encryption implemented
  - [ ] Token expiration tracking

- [ ] **Refresh Function**:
  - [ ] `refresh-calendar-tokens` Edge Function created
  - [ ] Google token refresh implemented
  - [ ] Microsoft token refresh implemented
  - [ ] Cron job scheduled

- [ ] **Error Handling**:
  - [ ] Failure detection implemented
  - [ ] User notification implemented
  - [ ] Integration deactivation implemented

- [ ] **Documentation**:
  - [ ] Troubleshooting guide created
  - [ ] Common issues documented

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 16 – Integration Hooks, Change Triggers, and Real-Time Location. All integration hooks are event-driven, constraint validation ensures schedule integrity, and real-time location enables map-based dispatch.

**Next Steps**: After completing Epic 16, the Scheduling & Dispatch module is complete. Proceed to Epic 15 (Documentation & API Reference) to finalize module documentation.

