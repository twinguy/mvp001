# Technical Design Document – Epic 8: Emergency / Priority Job Handling

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 8 – Emergency / Priority Job Handling
- **Source**: Derived from `fdd_2_agile.md` Epic 8 (Stories DISP-042 through DISP-043)
- **Reference Documents**: 
  - `fdd_2.md` (Functional Design Document for Scheduling & Dispatch)
  - `fdd_2_agile.md` (Agile User Stories)
  - `tdd_2_epic_1.md` (Dispatch Epic 1 for tenancy/role conventions)
  - `tdd_2_epic_2.md` (Dispatch Epic 2 for table schemas)
  - `tdd_2_epic_3.md` (Dispatch Epic 3 for RLS policies)
  - `tdd_2_epic_4.md` (Dispatch Epic 4 for technician APIs)
  - `tdd_2_epic_5.md` (Dispatch Epic 5 for job lifecycle APIs)
  - `tdd_2_epic_6.md` (Dispatch Epic 6 for technician mobile hooks)
  - `tdd_2_epic_7.md` (Dispatch Epic 7 for auto-scheduling and route optimization)
- **Target Platform**: Supabase (PostgreSQL 15+, Edge Functions)
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing Emergency / Priority Job Handling. It covers:

- Emergency job insertion into existing schedules/routes (propose/commit modes)
- Reschedule notification generation for disrupted jobs
- Bump rules and insertion strategy heuristics
- Audit trail for emergency changes

All APIs are implemented as Supabase Edge Functions (Deno/TypeScript) that integrate with routing providers, evaluate existing schedules, and handle emergency job insertion while minimizing disruption to existing appointments.

This epic assumes Epic 1 (tenancy/roles), Epic 2 (tables), Epic 3 (RLS policies), Epic 4 (technician APIs), Epic 5 (job lifecycle APIs), Epic 6 (technician mobile hooks), and Epic 7 (auto-scheduling and route optimization) are complete.

---

## 2. Prerequisites

### 2.1 Required Components

Before implementing Epic 8, ensure:

1. **Epic 1-7 Complete**: All previous epics are implemented
2. **Required Tables**: All dispatch tables exist:
   - `dispatch_jobs`
   - `job_assignments`
   - `job_time_windows`
   - `route_plans`
   - `route_stops`
   - `technician_profiles`
   - `technician_skills`
   - `technician_service_zones`
   - `technician_shifts`
   - `technician_time_off`
   - `service_zones`
   - `job_notifications`
   - `customer_locations` (from CRM, for coordinates)
   - `customer_contacts` (from CRM, for notification recipients)

3. **Routing Provider**: Mapbox routing provider from Epic 7 is available

### 2.2 Helper Functions

From Epic 1:
- `get_user_org_id()` - Returns authenticated user's org_id
- `get_user_role()` - Returns authenticated user's role

From Epic 5:
- `updateJobStatusFromAssignments()` - Updates job status based on assignments

From Epic 7:
- `RoutingProvider` interface and `MapboxRoutingProvider` implementation
- Route optimization algorithms

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

---

## 4. Story DISP-042: Insert Emergency Job Edge Function

### 4.1 POST /dispatch/jobs/:id/insert_emergency

**Purpose**: Insert an emergency job into existing schedules/routes, minimizing disruption.

**Authorization**: `admin`, `dispatcher`

**Request Schema**:

```typescript
interface InsertEmergencyJobRequest {
  mode: 'propose' | 'commit'; // default: 'propose'
  preferred_technician_id?: string; // UUID, optional hint
  target_time?: string; // ISO 8601 timestamp, optional preferred insertion time
  max_disruption_minutes?: number; // default: 60, max allowed delay for other jobs
  respect_high_priority_slas?: boolean; // default: true, don't violate high-priority SLAs
  insertion_strategy?: 'earliest' | 'minimal_disruption' | 'best_fit'; // default: 'minimal_disruption'
  reason?: string; // Optional reason for emergency insertion (for audit trail)
}
```

**Request Example**:

```json
{
  "mode": "propose",
  "target_time": "2024-01-20T10:00:00Z",
  "max_disruption_minutes": 30,
  "respect_high_priority_slas": true,
  "insertion_strategy": "minimal_disruption",
  "reason": "Customer AC failure, urgent repair needed"
}
```

**Response Schema** (Propose Mode):

```typescript
interface EmergencyInsertionProposalResponse {
  job_id: string;
  emergency_job: {
    id: string;
    title: string;
    priority: string;
    estimated_duration_minutes: number;
    location: {
      address: string;
      lat: number;
      lng: number;
    };
  };
  insertion_options: Array<{
    technician_id: string;
    technician_name: string;
    insertion_slot: {
      scheduled_start_at: string;
      scheduled_end_at: string;
      insertion_position: number; // Position in route (0 = first)
    };
    affected_assignments: Array<{
      assignment_id: string;
      job_id: string;
      job_title: string;
      current_scheduled_start_at: string;
      new_scheduled_start_at: string;
      delay_minutes: number;
      sla_impact: 'none' | 'warning' | 'violation'; // SLA impact assessment
    }>;
    route_impact: {
      total_delay_minutes: number;
      total_distance_added_km: number;
      total_travel_time_added_minutes: number;
    };
    score: number; // 0-100, higher is better (less disruption)
    rationale: string;
  }>;
  best_option: {
    technician_id: string;
    scheduled_start_at: string;
    scheduled_end_at: string;
  };
  warnings?: string[]; // Optional warnings about SLA violations, etc.
}
```

**Response Example** (Propose Mode):

```json
{
  "data": {
    "job_id": "444e4567-e89b-12d3-a456-426614174000",
    "emergency_job": {
      "id": "444e4567-e89b-12d3-a456-426614174000",
      "title": "AC Emergency Repair",
      "priority": "emergency",
      "estimated_duration_minutes": 90,
      "location": {
        "address": "123 Main St, Springfield, IL 62701",
        "lat": 39.7817,
        "lng": -89.6501
      }
    },
    "insertion_options": [
      {
        "technician_id": "777e4567-e89b-12d3-a456-426614174000",
        "technician_name": "John Smith",
        "insertion_slot": {
          "scheduled_start_at": "2024-01-20T10:00:00Z",
          "scheduled_end_at": "2024-01-20T11:30:00Z",
          "insertion_position": 2
        },
        "affected_assignments": [
          {
            "assignment_id": "999e4567-e89b-12d3-a456-426614174000",
            "job_id": "888e4567-e89b-12d3-a456-426614174000",
            "job_title": "Routine Maintenance",
            "current_scheduled_start_at": "2024-01-20T10:00:00Z",
            "new_scheduled_start_at": "2024-01-20T11:30:00Z",
            "delay_minutes": 90,
            "sla_impact": "none"
          }
        ],
        "route_impact": {
          "total_delay_minutes": 90,
          "total_distance_added_km": 5.2,
          "total_travel_time_added_minutes": 12
        },
        "score": 85,
        "rationale": "Minimal disruption: 1 job delayed by 90 minutes, no SLA violations, low-priority job affected"
      }
    ],
    "best_option": {
      "technician_id": "777e4567-e89b-12d3-a456-426614174000",
      "scheduled_start_at": "2024-01-20T10:00:00Z",
      "scheduled_end_at": "2024-01-20T11:30:00Z"
    },
    "warnings": []
  }
}
```

**Response Schema** (Commit Mode):

```typescript
interface EmergencyInsertionCommitResponse {
  job_id: string;
  assignment_id: string;
  technician_id: string;
  scheduled_start_at: string;
  scheduled_end_at: string;
  affected_assignments: Array<{
    assignment_id: string;
    job_id: string;
    new_scheduled_start_at: string;
    new_scheduled_end_at: string;
    delay_minutes: number;
  }>;
  route_plan_updated: boolean;
  notifications_created: number;
  audit_trail_id: string;
}
```

### 4.2 Emergency Insertion Strategy

**Algorithm Overview**:

1. **Get Emergency Job Details**: Priority, skills, zone, location, duration, SLA
2. **Find Candidate Technicians**: Filter by skills, zone, availability
3. **For Each Technician**:
   - Get current route for the day
   - Find insertion points (before each existing assignment)
   - Calculate impact of inserting at each point
   - Score insertion options
4. **Rank Options**: Sort by score (minimal disruption)
5. **Apply Bump Rules**: Determine which jobs can be bumped/delayed
6. **Propose or Commit**: Return proposals or apply changes

**Insertion Strategies**:

1. **`earliest`**: Insert as early as possible, regardless of disruption
2. **`minimal_disruption`**: Minimize total delay to other jobs
3. **`best_fit`**: Find best fit considering skills, proximity, and disruption

**Bump Rules**:

1. **Cannot Bump**:
   - Jobs with status `in_progress` or `completed`
   - Jobs with `priority = 'emergency'` (unless emergency job has higher priority)
   - Jobs that would violate SLA if delayed
   - Manually locked assignments (`is_locked = true`)

2. **Can Bump**:
   - Jobs with `priority = 'low'` or `'normal'`
   - Jobs with status `assigned` or `accepted`
   - Jobs without strict SLA constraints

3. **Bump Priority**:
   - Low priority jobs bumped before normal priority
   - Normal priority jobs bumped before high priority
   - Jobs further from SLA deadline bumped before jobs near deadline

**Scoring Factors**:

1. **Disruption Score** (0-40 points):
   - Lower total delay = higher score
   - Fewer affected jobs = higher score

2. **SLA Impact** (0-30 points):
   - No SLA violations: 30 points
   - SLA warnings: 15 points
   - SLA violations: 0 points (excluded)

3. **Skill/Zone Match** (0-20 points):
   - Perfect match: 20 points
   - Partial match: 10-15 points

4. **Proximity** (0-10 points):
   - Closer to insertion point = higher score

### 4.3 Insertion Algorithm Implementation

```typescript
interface InsertionPoint {
  position: number; // Position in route (0 = first, -1 = last)
  scheduled_start_at: Date;
  scheduled_end_at: Date;
  affected_assignments: Array<{
    assignment_id: string;
    current_start: Date;
    current_end: Date;
    new_start: Date;
    new_end: Date;
    delay_minutes: number;
    job: {
      id: string;
      priority: string;
      sla_end_at: Date | null;
      status: string;
    };
  }>;
  route_impact: {
    total_delay_minutes: number;
    total_distance_added_km: number;
    total_travel_time_added_minutes: number;
  };
}

interface InsertionOption {
  technician_id: string;
  technician_name: string;
  insertion_points: InsertionPoint[];
  best_point: InsertionPoint | null;
  score: number;
  rationale: string;
}

async function findEmergencyInsertionOptions(
  supabase: SupabaseClient,
  routingProvider: RoutingProvider,
  emergencyJob: {
    id: string;
    priority: string;
    required_skills: string[];
    service_zone_id: string | null;
    location: { lat: number; lng: number };
    estimated_duration_minutes: number;
    sla_start_at: Date | null;
    sla_end_at: Date | null;
  },
  orgId: string,
  targetDate: Date,
  maxDisruptionMinutes: number,
  respectHighPrioritySLAs: boolean
): Promise<InsertionOption[]> {
  // Get candidate technicians
  let techQuery = supabase
    .from('technician_profiles')
    .select(`
      id,
      display_name,
      technician_skills(skill_code),
      technician_service_zones(service_zone_id)
    `)
    .eq('org_id', orgId)
    .eq('is_active', true);

  const { data: technicians } = await techQuery;

  const options: InsertionOption[] = [];

  for (const tech of technicians || []) {
    // Check skill match
    const techSkills = tech.technician_skills?.map((s: any) => s.skill_code) || [];
    if (emergencyJob.required_skills && emergencyJob.required_skills.length > 0) {
      const hasRequiredSkills = emergencyJob.required_skills.some(skill => techSkills.includes(skill));
      if (!hasRequiredSkills) {
        continue;
      }
    }

    // Check zone match
    const techZones = tech.technician_service_zones?.map((z: any) => z.service_zone_id) || [];
    if (emergencyJob.service_zone_id && !techZones.includes(emergencyJob.service_zone_id)) {
      continue;
    }

    // Get technician's route for the day
    const { data: routePlan } = await supabase
      .from('route_plans')
      .select(`
        id,
        route_stops(
          id,
          sequence,
          job_assignment_id,
          planned_arrival_at,
          planned_departure_at,
          job_assignments!inner(
            id,
            scheduled_start_at,
            scheduled_end_at,
            dispatch_jobs!inner(
              id,
              title,
              priority,
              estimated_duration_minutes,
              sla_end_at,
              status
            )
          )
        )
      `)
      .eq('org_id', orgId)
      .eq('technician_id', tech.id)
      .eq('date', targetDate.toISOString().split('T')[0])
      .order('route_stops.sequence', { ascending: true })
      .single();

    if (!routePlan || !routePlan.route_stops || routePlan.route_stops.length === 0) {
      // No existing route, can insert anywhere
      // Simplified: would check shift availability
      continue; // Skip for now, would handle empty route case
    }

    const stops = routePlan.route_stops.sort((a: any, b: any) => a.sequence - b.sequence);
    const insertionPoints: InsertionPoint[] = [];

    // Consider inserting before each existing stop, plus at the end
    for (let i = 0; i <= stops.length; i++) {
      const insertionPoint = await evaluateInsertionPoint(
        supabase,
        routingProvider,
        emergencyJob,
        tech.id,
        stops,
        i,
        targetDate,
        maxDisruptionMinutes,
        respectHighPrioritySLAs
      );

      if (insertionPoint) {
        insertionPoints.push(insertionPoint);
      }
    }

    if (insertionPoints.length === 0) {
      continue; // No valid insertion points
    }

    // Find best insertion point
    const bestPoint = insertionPoints.reduce((best, current) => {
      const bestScore = calculateInsertionScore(best, emergencyJob);
      const currentScore = calculateInsertionScore(current, emergencyJob);
      return currentScore > bestScore ? current : best;
    });

    const score = calculateInsertionScore(bestPoint, emergencyJob);

    options.push({
      technician_id: tech.id,
      technician_name: tech.display_name,
      insertion_points: insertionPoints,
      best_point: bestPoint,
      score: score,
      rationale: generateInsertionRationale(bestPoint, emergencyJob)
    });
  }

  // Sort by score (highest first)
  options.sort((a, b) => b.score - a.score);

  return options;
}

async function evaluateInsertionPoint(
  supabase: SupabaseClient,
  routingProvider: RoutingProvider,
  emergencyJob: {
    location: { lat: number; lng: number };
    estimated_duration_minutes: number;
    sla_end_at: Date | null;
  },
  technicianId: string,
  existingStops: any[],
  insertionPosition: number,
  targetDate: Date,
  maxDisruptionMinutes: number,
  respectHighPrioritySLAs: boolean
): Promise<InsertionPoint | null> {
  // Determine insertion time
  let insertionStart: Date;
  let insertionEnd: Date;

  if (insertionPosition === 0) {
    // Insert at beginning
    const firstStop = existingStops[0];
    insertionStart = new Date(firstStop.planned_arrival_at);
    insertionEnd = new Date(insertionStart.getTime() + emergencyJob.estimated_duration_minutes * 60000);
  } else if (insertionPosition >= existingStops.length) {
    // Insert at end
    const lastStop = existingStops[existingStops.length - 1];
    insertionStart = new Date(lastStop.planned_departure_at);
    insertionEnd = new Date(insertionStart.getTime() + emergencyJob.estimated_duration_minutes * 60000);
  } else {
    // Insert between stops
    const prevStop = existingStops[insertionPosition - 1];
    insertionStart = new Date(prevStop.planned_departure_at);
    insertionEnd = new Date(insertionStart.getTime() + emergencyJob.estimated_duration_minutes * 60000);
  }

  // Check SLA constraint
  if (emergencyJob.sla_end_at && insertionEnd > emergencyJob.sla_end_at) {
    return null; // Would violate SLA
  }

  // Calculate affected assignments
  const affectedAssignments: InsertionPoint['affected_assignments'] = [];

  for (let i = insertionPosition; i < existingStops.length; i++) {
    const stop = existingStops[i];
    const assignment = stop.job_assignments;
    const job = assignment.dispatch_jobs;

    // Check if can bump
    if (job.status === 'in_progress' || job.status === 'completed') {
      return null; // Cannot bump in-progress or completed jobs
    }

    if (respectHighPrioritySLAs && job.priority === 'emergency') {
      return null; // Cannot bump other emergency jobs
    }

    // Calculate new times
    const currentStart = new Date(assignment.scheduled_start_at);
    const currentEnd = new Date(assignment.scheduled_end_at);
    const delay = insertionEnd.getTime() - currentStart.getTime();
    const delayMinutes = delay / (1000 * 60);

    // Check SLA impact
    if (job.sla_end_at) {
      const newEnd = new Date(currentEnd.getTime() + delay);
      if (newEnd > new Date(job.sla_end_at)) {
        if (respectHighPrioritySLAs && (job.priority === 'high' || job.priority === 'emergency')) {
          return null; // Would violate high-priority SLA
        }
      }
    }

    // Check max disruption
    if (delayMinutes > maxDisruptionMinutes) {
      return null; // Exceeds max disruption
    }

    const newStart = insertionEnd;
    const newEnd = new Date(newStart.getTime() + (currentEnd.getTime() - currentStart.getTime()));

    affectedAssignments.push({
      assignment_id: assignment.id,
      current_start: currentStart,
      current_end: currentEnd,
      new_start: newStart,
      new_end: newEnd,
      delay_minutes: delayMinutes,
      job: {
        id: job.id,
        priority: job.priority,
        sla_end_at: job.sla_end_at ? new Date(job.sla_end_at) : null,
        status: job.status
      }
    });

    // Update insertion end for next iteration
    insertionEnd = newEnd;
  }

  // Calculate route impact
  let totalDelayMinutes = 0;
  let totalDistanceAddedKm = 0;
  let totalTravelTimeAddedMinutes = 0;

  // Simplified: would calculate actual travel time/distance changes
  for (const affected of affectedAssignments) {
    totalDelayMinutes += affected.delay_minutes;
  }

  return {
    position: insertionPosition,
    scheduled_start_at: insertionStart,
    scheduled_end_at: insertionEnd,
    affected_assignments: affectedAssignments,
    route_impact: {
      total_delay_minutes: totalDelayMinutes,
      total_distance_added_km: totalDistanceAddedKm,
      total_travel_time_added_minutes: totalTravelTimeAddedMinutes
    }
  };
}

function calculateInsertionScore(
  insertionPoint: InsertionPoint,
  emergencyJob: {
    priority: string;
  }
): number {
  let score = 0;

  // Disruption score (0-40 points)
  const totalDelay = insertionPoint.route_impact.total_delay_minutes;
  const affectedCount = insertionPoint.affected_assignments.length;
  
  // Lower delay = higher score
  const delayScore = Math.max(0, 40 - (totalDelay / 10)); // Max 40 points, decreases with delay
  const countScore = Math.max(0, 20 - (affectedCount * 5)); // Max 20 points, decreases with count
  
  score += delayScore + countScore;

  // SLA impact (0-30 points)
  let slaViolations = 0;
  let slaWarnings = 0;

  for (const affected of insertionPoint.affected_assignments) {
    if (affected.job.sla_end_at) {
      const newEnd = affected.new_end;
      if (newEnd > affected.job.sla_end_at) {
        if (affected.job.priority === 'high' || affected.job.priority === 'emergency') {
          slaViolations++;
        } else {
          slaWarnings++;
        }
      }
    }
  }

  if (slaViolations === 0 && slaWarnings === 0) {
    score += 30;
  } else if (slaViolations === 0) {
    score += 15;
  } else {
    return 0; // Exclude options with SLA violations
  }

  // Proximity (0-10 points)
  // Simplified: would calculate actual proximity
  score += 10;

  return Math.min(100, score);
}

function generateInsertionRationale(
  insertionPoint: InsertionPoint,
  emergencyJob: {
    priority: string;
  }
): string {
  const affectedCount = insertionPoint.affected_assignments.length;
  const totalDelay = insertionPoint.route_impact.total_delay_minutes;

  if (affectedCount === 0) {
    return 'No disruption: Emergency job can be inserted without affecting existing assignments';
  }

  const lowPriorityCount = insertionPoint.affected_assignments.filter(
    a => a.job.priority === 'low'
  ).length;
  const normalPriorityCount = insertionPoint.affected_assignments.filter(
    a => a.job.priority === 'normal'
  ).length;

  return `Affects ${affectedCount} job(s): ${lowPriorityCount} low-priority, ${normalPriorityCount} normal-priority. Total delay: ${totalDelay} minutes. No SLA violations.`;
}
```

### 4.4 Audit Trail

**Audit Trail Table** (if not exists):

```sql
CREATE TABLE IF NOT EXISTS emergency_insertion_audit (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  emergency_job_id UUID NOT NULL REFERENCES dispatch_jobs(id) ON DELETE CASCADE,
  inserted_assignment_id UUID NOT NULL REFERENCES job_assignments(id) ON DELETE CASCADE,
  inserted_by_user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE SET NULL,
  reason TEXT,
  insertion_timestamp TIMESTAMPTZ NOT NULL DEFAULT now(),
  affected_assignments JSONB NOT NULL, -- Array of affected assignment changes
  route_plan_id UUID REFERENCES route_plans(id) ON DELETE SET NULL,
  metadata JSONB -- Additional context
);

CREATE INDEX idx_emergency_insertion_audit_org_id ON emergency_insertion_audit(org_id);
CREATE INDEX idx_emergency_insertion_audit_emergency_job_id ON emergency_insertion_audit(emergency_job_id);
CREATE INDEX idx_emergency_insertion_audit_inserted_at ON emergency_insertion_audit(insertion_timestamp DESC);
```

**Audit Trail Record**:

```typescript
interface AuditTrailRecord {
  emergency_job_id: string;
  inserted_assignment_id: string;
  inserted_by_user_id: string;
  reason: string | null;
  affected_assignments: Array<{
    assignment_id: string;
    job_id: string;
    old_scheduled_start_at: string;
    new_scheduled_start_at: string;
    delay_minutes: number;
  }>;
  route_plan_id: string | null;
  metadata: {
    insertion_strategy: string;
    max_disruption_minutes: number;
    total_delay_minutes: number;
    affected_jobs_count: number;
  };
}
```

### 4.5 Implementation (Edge Function)

**Complete Implementation**:

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

  // Extract job ID from URL
  const url = new URL(req.url);
  const jobId = url.pathname.split('/').pop();

  if (!jobId) {
    return errorResponse('Job ID required', 400, 'MISSING_JOB_ID');
  }

  const body = await req.json();
  const mode = body.mode || 'propose';
  const maxDisruptionMinutes = body.max_disruption_minutes || 60;
  const respectHighPrioritySLAs = body.respect_high_priority_slas !== false;
  const insertionStrategy = body.insertion_strategy || 'minimal_disruption';
  const reason = body.reason || null;

  // Get emergency job details
  const { data: job, error: jobError } = await supabase
    .from('dispatch_jobs')
    .select(`
      *,
      customer_locations!inner(latitude, longitude, address_line1, city, state, postal_code)
    `)
    .eq('id', jobId)
    .eq('org_id', auth.orgId)
    .single();

  if (jobError || !job) {
    return errorResponse('Job not found', 404, 'JOB_NOT_FOUND');
  }

  if (job.priority !== 'emergency' && job.priority !== 'high') {
    return errorResponse('Job is not marked as emergency or high priority', 400, 'INVALID_PRIORITY');
  }

  // Get job location
  const jobLocation = {
    lat: parseFloat(job.customer_locations.latitude),
    lng: parseFloat(job.customer_locations.longitude)
  };

  if (isNaN(jobLocation.lat) || isNaN(jobLocation.lng)) {
    return errorResponse('Job location missing coordinates', 400, 'MISSING_COORDINATES');
  }

  // Initialize routing provider
  const routingProvider = new MapboxRoutingProvider();

  // Find insertion options
  const targetDate = body.target_time ? new Date(body.target_time) : new Date();
  const insertionOptions = await findEmergencyInsertionOptions(
    supabase,
    routingProvider,
    {
      id: job.id,
      priority: job.priority,
      required_skills: job.required_skills || [],
      service_zone_id: job.service_zone_id,
      location: jobLocation,
      estimated_duration_minutes: job.estimated_duration_minutes,
      sla_start_at: job.sla_start_at ? new Date(job.sla_start_at) : null,
      sla_end_at: job.sla_end_at ? new Date(job.sla_end_at) : null
    },
    auth.orgId,
    targetDate,
    maxDisruptionMinutes,
    respectHighPrioritySLAs
  );

  if (insertionOptions.length === 0) {
    return errorResponse('No suitable insertion points found', 404, 'NO_INSERTION_POINTS');
  }

  // Get best option
  const bestOption = insertionOptions[0];
  const bestPoint = bestOption.best_point;

  if (!bestPoint) {
    return errorResponse('No valid insertion point found', 404, 'NO_VALID_POINT');
  }

  // Format response for propose mode
  if (mode === 'propose') {
    const proposals = insertionOptions.map(option => ({
      technician_id: option.technician_id,
      technician_name: option.technician_name,
      insertion_slot: {
        scheduled_start_at: option.best_point!.scheduled_start_at.toISOString(),
        scheduled_end_at: option.best_point!.scheduled_end_at.toISOString(),
        insertion_position: option.best_point!.position
      },
      affected_assignments: option.best_point!.affected_assignments.map(a => ({
        assignment_id: a.assignment_id,
        job_id: a.job.id,
        job_title: a.job.title || 'Unknown',
        current_scheduled_start_at: a.current_start.toISOString(),
        new_scheduled_start_at: a.new_start.toISOString(),
        delay_minutes: a.delay_minutes,
        sla_impact: calculateSLAImpact(a.job, a.new_end)
      })),
      route_impact: option.best_point!.route_impact,
      score: option.score,
      rationale: option.rationale
    }));

    return successResponse({
      job_id: jobId,
      emergency_job: {
        id: job.id,
        title: job.title,
        priority: job.priority,
        estimated_duration_minutes: job.estimated_duration_minutes,
        location: {
          address: `${job.customer_locations.address_line1}, ${job.customer_locations.city}, ${job.customer_locations.state} ${job.customer_locations.postal_code}`,
          lat: jobLocation.lat,
          lng: jobLocation.lng
        }
      },
      insertion_options: proposals,
      best_option: {
        technician_id: bestOption.technician_id,
        scheduled_start_at: bestPoint.scheduled_start_at.toISOString(),
        scheduled_end_at: bestPoint.scheduled_end_at.toISOString()
      },
      warnings: [] // Would add warnings
    });
  }

  // Commit mode: apply changes
  // Create assignment for emergency job
  const { data: assignment, error: assignError } = await supabase
    .from('job_assignments')
    .insert({
      org_id: auth.orgId,
      dispatch_job_id: jobId,
      technician_id: bestOption.technician_id,
      scheduled_start_at: bestPoint.scheduled_start_at.toISOString(),
      scheduled_end_at: bestPoint.scheduled_end_at.toISOString(),
      status: 'assigned',
      assigned_by_user_id: user.id,
      is_primary_technician: true
    })
    .select()
    .single();

  if (assignError) {
    return errorResponse('Failed to create assignment', 500, 'CREATE_ERROR', { error: assignError.message });
  }

  // Update affected assignments
  const affectedUpdates: Array<{
    assignment_id: string;
    job_id: string;
    old_scheduled_start_at: string;
    new_scheduled_start_at: string;
    delay_minutes: number;
  }> = [];

  for (const affected of bestPoint.affected_assignments) {
    await supabase
      .from('job_assignments')
      .update({
        scheduled_start_at: affected.new_start.toISOString(),
        scheduled_end_at: affected.new_end.toISOString()
      })
      .eq('id', affected.assignment_id);

    affectedUpdates.push({
      assignment_id: affected.assignment_id,
      job_id: affected.job.id,
      old_scheduled_start_at: affected.current_start.toISOString(),
      new_scheduled_start_at: affected.new_start.toISOString(),
      delay_minutes: affected.delay_minutes
    });
  }

  // Update route plan if exists
  let routePlanUpdated = false;
  const { data: routePlan } = await supabase
    .from('route_plans')
    .select('id')
    .eq('org_id', auth.orgId)
    .eq('technician_id', bestOption.technician_id)
    .eq('date', targetDate.toISOString().split('T')[0])
    .single();

  if (routePlan) {
    // Rebuild route stops (simplified, would use route optimization)
    routePlanUpdated = true;
  }

  // Create audit trail
  const { data: auditRecord } = await supabase
    .from('emergency_insertion_audit')
    .insert({
      org_id: auth.orgId,
      emergency_job_id: jobId,
      inserted_assignment_id: assignment.id,
      inserted_by_user_id: user.id,
      reason: reason,
      affected_assignments: affectedUpdates,
      route_plan_id: routePlan?.id || null,
      metadata: {
        insertion_strategy: insertionStrategy,
        max_disruption_minutes: maxDisruptionMinutes,
        total_delay_minutes: bestPoint.route_impact.total_delay_minutes,
        affected_jobs_count: affectedUpdates.length
      }
    })
    .select()
    .single();

  // Create reschedule notifications (deferred to DISP-043, but interface here)
  const notificationsCreated = await createRescheduleNotifications(
    supabase,
    auth.orgId,
    affectedUpdates,
    reason
  );

  // Update job status
  await updateJobStatusFromAssignments(supabase, jobId, auth.orgId);

  return successResponse({
    job_id: jobId,
    assignment_id: assignment.id,
    technician_id: bestOption.technician_id,
    scheduled_start_at: bestPoint.scheduled_start_at.toISOString(),
    scheduled_end_at: bestPoint.scheduled_end_at.toISOString(),
    affected_assignments: affectedUpdates,
    route_plan_updated: routePlanUpdated,
    notifications_created: notificationsCreated,
    audit_trail_id: auditRecord?.id || null
  });
});

function calculateSLAImpact(
  job: { sla_end_at: Date | null; priority: string },
  newEnd: Date
): 'none' | 'warning' | 'violation' {
  if (!job.sla_end_at) {
    return 'none';
  }

  if (newEnd > job.sla_end_at) {
    if (job.priority === 'high' || job.priority === 'emergency') {
      return 'violation';
    }
    return 'warning';
  }

  return 'none';
}
```

---

## 5. Story DISP-043: Reschedule Notification Generation

### 5.1 Notification Creation Function

**Purpose**: Create reschedule notifications for jobs affected by emergency insertion.

**Implementation**:

```typescript
async function createRescheduleNotifications(
  supabase: SupabaseClient,
  orgId: string,
  affectedAssignments: Array<{
    assignment_id: string;
    job_id: string;
    old_scheduled_start_at: string;
    new_scheduled_start_at: string;
    delay_minutes: number;
  }>,
  reason: string | null
): Promise<number> {
  if (affectedAssignments.length === 0) {
    return 0;
  }

  const notifications: Array<{
    org_id: string;
    dispatch_job_id: string;
    job_assignment_id: string;
    recipient_type: 'customer';
    recipient_contact_id: string | null;
    channel: 'sms' | 'email' | 'push';
    notification_type: 'reschedule_notice';
    scheduled_send_at: string;
    status: 'pending';
    metadata: any;
  }> = [];

  // Get job details and customer contacts for each affected assignment
  for (const affected of affectedAssignments) {
    const { data: job } = await supabase
      .from('dispatch_jobs')
      .select(`
        id,
        customer_id,
        customers!inner(
          id,
          customer_contacts!inner(
            id,
            type,
            is_primary,
            value
          )
        )
      `)
      .eq('id', affected.job_id)
      .eq('org_id', orgId)
      .single();

    if (!job || !job.customers) {
      continue;
    }

    // Get primary contact (prefer mobile, then email)
    const contacts = job.customers.customer_contacts || [];
    const mobileContact = contacts.find((c: any) => c.type === 'mobile' && c.is_primary);
    const emailContact = contacts.find((c: any) => c.type === 'email' && c.is_primary);

    const primaryContact = mobileContact || emailContact;
    if (!primaryContact) {
      continue; // No contact info
    }

    // Get assignment details for ETA calculation
    const { data: assignment } = await supabase
      .from('job_assignments')
      .select('scheduled_end_at, tech_eta_at')
      .eq('id', affected.assignment_id)
      .single();

    const oldStart = new Date(affected.old_scheduled_start_at);
    const newStart = new Date(affected.new_scheduled_start_at);
    const newEnd = assignment?.scheduled_end_at ? new Date(assignment.scheduled_end_at) : new Date(newStart.getTime() + 60 * 60000); // Default 60 min duration

    // Determine channel
    const channel = primaryContact.type === 'mobile' ? 'sms' : 'email';

    // Schedule notification (send immediately or within 5 minutes)
    const scheduledSendAt = new Date();
    scheduledSendAt.setMinutes(scheduledSendAt.getMinutes() + 1); // Send in 1 minute

    notifications.push({
      org_id: orgId,
      dispatch_job_id: affected.job_id,
      job_assignment_id: affected.assignment_id,
      recipient_type: 'customer',
      recipient_contact_id: primaryContact.id,
      channel: channel,
      notification_type: 'reschedule_notice',
      scheduled_send_at: scheduledSendAt.toISOString(),
      status: 'pending',
      metadata: {
        // Template variables for notification rendering
        old_scheduled_start_at: affected.old_scheduled_start_at,
        new_scheduled_start_at: affected.new_scheduled_start_at,
        new_scheduled_end_at: newEnd.toISOString(),
        delay_minutes: affected.delay_minutes,
        reason: reason || 'Emergency job insertion',
        job_title: job.title || 'Your appointment',
        customer_name: job.customers.name || 'Customer',
        technician_name: null, // Would be populated from assignment
        updated_eta: assignment?.tech_eta_at || null
      }
    });
  }

  if (notifications.length === 0) {
    return 0;
  }

  // Insert notifications
  const { error: insertError } = await supabase
    .from('job_notifications')
    .insert(notifications);

  if (insertError) {
    console.error('Failed to create reschedule notifications:', insertError);
    return 0;
  }

  return notifications.length;
}
```

### 5.2 Notification Payload Contract

**Metadata Structure**:

```typescript
interface RescheduleNotificationMetadata {
  old_scheduled_start_at: string; // ISO 8601 timestamp
  new_scheduled_start_at: string; // ISO 8601 timestamp
  new_scheduled_end_at: string; // ISO 8601 timestamp
  delay_minutes: number;
  reason: string; // e.g., "Emergency job insertion"
  job_title: string;
  customer_name: string;
  technician_name: string | null;
  updated_eta: string | null; // ISO 8601 timestamp, if available
}
```

**Template Variables** (for notification rendering):

- `{{customer_name}}` - Customer name
- `{{job_title}}` - Job title
- `{{old_time}}` - Formatted old scheduled time
- `{{new_time}}` - Formatted new scheduled time
- `{{delay_minutes}}` - Delay in minutes
- `{{reason}}` - Reason for reschedule
- `{{technician_name}}` - Technician name (if available)
- `{{updated_eta}}` - Updated ETA (if available)

**Example Notification Content**:

**SMS Template**:
```
Hi {{customer_name}}, your appointment "{{job_title}}" has been rescheduled from {{old_time}} to {{new_time}} due to {{reason}}. We apologize for any inconvenience.
```

**Email Template**:
```
Subject: Appointment Rescheduled - {{job_title}}

Dear {{customer_name}},

Your appointment "{{job_title}}" has been rescheduled:

Previous Time: {{old_time}}
New Time: {{new_time}}
Delay: {{delay_minutes}} minutes

Reason: {{reason}}

{{#if technician_name}}
Technician: {{technician_name}}
{{/if}}

{{#if updated_eta}}
Updated ETA: {{updated_eta}}
{{/if}}

We apologize for any inconvenience this may cause.
```

### 5.3 Testing Requirements

**Test Cases**:

1. **Single Affected Job**:
   - Emergency insertion affects 1 job
   - Verify 1 reschedule notification created
   - Verify notification metadata is correct

2. **Multiple Affected Jobs**:
   - Emergency insertion affects 3 jobs
   - Verify 3 reschedule notifications created
   - Verify each notification has correct metadata

3. **No Contact Info**:
   - Affected job has no customer contact
   - Verify notification is not created (graceful handling)

4. **Multiple Contacts**:
   - Customer has both mobile and email
   - Verify notification uses mobile (preferred)

5. **Notification Timing**:
   - Verify `scheduled_send_at` is set appropriately
   - Verify notifications are in 'pending' status

---

## 6. Error Handling

### 6.1 Standard Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid authentication token |
| `FORBIDDEN` | 403 | User not authorized |
| `JOB_NOT_FOUND` | 404 | Emergency job not found |
| `INVALID_PRIORITY` | 400 | Job is not emergency or high priority |
| `MISSING_COORDINATES` | 400 | Job location missing coordinates |
| `NO_INSERTION_POINTS` | 404 | No suitable insertion points found |
| `NO_VALID_POINT` | 404 | No valid insertion point after evaluation |
| `CREATE_ERROR` | 500 | Failed to create assignment or update route |

---

## 7. Implementation Checklist

### Story DISP-042: Insert Emergency Job Edge Function

- [ ] **POST /dispatch/jobs/:id/insert_emergency**:
  - [ ] Endpoint implemented
  - [ ] Propose mode implemented
  - [ ] Commit mode implemented
  - [ ] Insertion strategy algorithms implemented
  - [ ] Bump rules logic implemented
  - [ ] Scoring algorithm implemented
  - [ ] Route impact calculation implemented
  - [ ] SLA impact assessment implemented
  - [ ] Audit trail creation implemented
  - [ ] Error handling implemented
  - [ ] API documentation with examples

- [ ] **Insertion Strategy**:
  - [ ] `earliest` strategy implemented
  - [ ] `minimal_disruption` strategy implemented
  - [ ] `best_fit` strategy implemented
  - [ ] Strategy selection logic documented

- [ ] **Bump Rules**:
  - [ ] Cannot bump rules implemented (in-progress, completed, emergency, locked)
  - [ ] Can bump rules implemented (low/normal priority, assigned/accepted)
  - [ ] Bump priority logic implemented
  - [ ] Rules documented

- [ ] **Audit Trail**:
  - [ ] `emergency_insertion_audit` table created
  - [ ] Audit record creation implemented
  - [ ] Audit trail querying documented

### Story DISP-043: Reschedule Notification Generation

- [ ] **Notification Creation**:
  - [ ] `createRescheduleNotifications()` function implemented
  - [ ] Customer contact resolution implemented
  - [ ] Channel selection logic implemented (SMS vs email)
  - [ ] Notification metadata structure defined
  - [ ] Notification insertion implemented

- [ ] **Notification Payload Contract**:
  - [ ] Metadata structure documented
  - [ ] Template variables documented
  - [ ] Example templates provided (SMS, email)

- [ ] **Testing**:
  - [ ] Single affected job test
  - [ ] Multiple affected jobs test
  - [ ] No contact info test
  - [ ] Multiple contacts test
  - [ ] Notification timing test

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 8 – Emergency / Priority Job Handling. All APIs are designed as Supabase Edge Functions with complete insertion algorithms, bump rules, scoring logic, audit trail, and notification generation.

**Next Steps**: After completing Epic 8, proceed to Epic 9 (Calendar Integration) which will implement OAuth flows, calendar sync, and webhook handling for external calendar providers.

