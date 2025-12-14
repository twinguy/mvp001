# Technical Design Document – Epic 7: Auto-Scheduling and Route Optimization (Edge Functions)

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 7 – Auto-Scheduling and Route Optimization (Edge Functions)
- **Source**: Derived from `fdd_2_agile.md` Epic 7 (Stories DISP-036 through DISP-041)
- **Reference Documents**: 
  - `fdd_2.md` (Functional Design Document for Scheduling & Dispatch)
  - `fdd_2_agile.md` (Agile User Stories)
  - `tdd_2_epic_1.md` (Dispatch Epic 1 for tenancy/role conventions)
  - `tdd_2_epic_2.md` (Dispatch Epic 2 for table schemas)
  - `tdd_2_epic_3.md` (Dispatch Epic 3 for RLS policies)
  - `tdd_2_epic_4.md` (Dispatch Epic 4 for technician APIs)
  - `tdd_2_epic_5.md` (Dispatch Epic 5 for job lifecycle APIs)
  - `tdd_2_epic_6.md` (Dispatch Epic 6 for technician mobile hooks)
- **Target Platform**: Supabase (PostgreSQL 15+, Edge Functions)
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing Auto-Scheduling and Route Optimization Edge Functions. It covers:

- Routing/mapping provider strategy and abstraction
- Auto-scheduling single job (propose/commit modes)
- Technician candidate suggestion with scoring
- Route optimization for individual technicians
- Bulk daily optimization for all technicians
- Asynchronous optimization job tracking and progress reporting

All APIs are implemented as Supabase Edge Functions (Deno/TypeScript) that integrate with routing providers, implement optimization algorithms, and provide comprehensive scheduling automation.

This epic assumes Epic 1 (tenancy/roles), Epic 2 (tables), Epic 3 (RLS policies), Epic 4 (technician APIs), Epic 5 (job lifecycle APIs), and Epic 6 (technician mobile hooks) are complete.

---

## 2. Prerequisites

### 2.1 Required Components

Before implementing Epic 7, ensure:

1. **Epic 1-6 Complete**: All previous epics are implemented
2. **Required Tables**: All dispatch tables exist:
   - `dispatch_jobs`
   - `job_assignments`
   - `job_time_windows`
   - `technician_profiles`
   - `technician_skills`
   - `technician_service_zones`
   - `technician_shifts`
   - `technician_time_off`
   - `service_zones`
   - `route_plans`
   - `route_stops`
   - `customer_locations` (from CRM, for coordinates)

3. **Location Data**: `customer_locations` table has `latitude` and `longitude` populated for routing

### 2.2 Helper Functions

From Epic 1:
- `get_user_org_id()` - Returns authenticated user's org_id
- `get_user_role()` - Returns authenticated user's role

From Epic 5:
- `updateJobStatusFromAssignments()` - Updates job status based on assignments

---

## 3. Story DISP-036: Routing/Mapping Provider Strategy

### 3.1 Provider Evaluation

**Candidate Providers**:

1. **Google Maps Platform**:
   - **API**: Directions API, Distance Matrix API
   - **Pros**: Accurate, comprehensive, widely used
   - **Cons**: Cost at scale, requires billing account
   - **Rate Limits**: 40,000 requests/day free tier, then pay-as-you-go

2. **Mapbox**:
   - **API**: Directions API, Matrix API
   - **Pros**: Good pricing, developer-friendly, open-source components
   - **Cons**: Less comprehensive than Google
   - **Rate Limits**: 100,000 requests/month free tier

3. **OpenRouteService**:
   - **API**: Directions API, Matrix API
   - **Pros**: Free tier, open-source
   - **Cons**: Less accurate, rate limits on free tier

### 3.2 Provider Selection Decision

**Decision**: Use **Mapbox** as primary provider with abstraction layer for future swapping.

**Rationale**:
- Good balance of accuracy and cost
- Developer-friendly API
- Generous free tier for MVP
- Can swap to Google Maps later if needed

**Alternative**: If budget allows, use Google Maps for better accuracy.

### 3.3 Provider Abstraction Layer

**Design**: Create abstraction interface to allow swapping providers without changing optimization logic.

**Interface Definition**:

```typescript
interface RoutingProvider {
  /**
   * Get travel time and distance between two points
   */
  getRoute(
    origin: { lat: number; lng: number },
    destination: { lat: number; lng: number },
    options?: RoutingOptions
  ): Promise<RouteResult>;

  /**
   * Get distance matrix for multiple origins and destinations
   */
  getDistanceMatrix(
    origins: Array<{ lat: number; lng: number }>,
    destinations: Array<{ lat: number; lng: number }>,
    options?: RoutingOptions
  ): Promise<DistanceMatrixResult>;
}

interface RoutingOptions {
  departure_time?: Date; // For traffic-aware routing
  mode?: 'driving' | 'walking' | 'cycling'; // Default: 'driving'
  avoid?: string[]; // e.g., ['tolls', 'highways']
}

interface RouteResult {
  distance_meters: number;
  duration_seconds: number;
  polyline?: string; // Encoded polyline for map display
}

interface DistanceMatrixResult {
  rows: Array<{
    elements: Array<{
      distance_meters: number;
      duration_seconds: number;
      status: 'OK' | 'NOT_FOUND' | 'ZERO_RESULTS';
    }>;
  }>;
}
```

### 3.4 Mapbox Implementation

**Mapbox API Details**:

- **Base URL**: `https://api.mapbox.com`
- **Directions API**: `/directions/v5/mapbox/driving/{coordinates}`
- **Matrix API**: `/directions-matrix/v1/mapbox/driving/{coordinates}`

**API Key Storage**:

- Store Mapbox access token in Supabase Secrets: `MAPBOX_ACCESS_TOKEN`
- Access via `Deno.env.get('MAPBOX_ACCESS_TOKEN')`
- Never expose to frontend

**Implementation**:

```typescript
class MapboxRoutingProvider implements RoutingProvider {
  private accessToken: string;
  private baseUrl = 'https://api.mapbox.com';

  constructor() {
    this.accessToken = Deno.env.get('MAPBOX_ACCESS_TOKEN') || '';
    if (!this.accessToken) {
      throw new Error('MAPBOX_ACCESS_TOKEN not configured');
    }
  }

  async getRoute(
    origin: { lat: number; lng: number },
    destination: { lat: number; lng: number },
    options?: RoutingOptions
  ): Promise<RouteResult> {
    const coords = `${origin.lng},${origin.lat};${destination.lng},${destination.lat}`;
    const url = `${this.baseUrl}/directions/v5/mapbox/driving/${coords}?access_token=${this.accessToken}`;

    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`Mapbox API error: ${response.statusText}`);
    }

    const data = await response.json();
    const route = data.routes[0];

    return {
      distance_meters: route.distance,
      duration_seconds: route.duration,
      polyline: route.geometry // Encoded polyline
    };
  }

  async getDistanceMatrix(
    origins: Array<{ lat: number; lng: number }>,
    destinations: Array<{ lat: number; lng: number }>,
    options?: RoutingOptions
  ): Promise<DistanceMatrixResult> {
    // Mapbox Matrix API supports up to 25 coordinates total
    if (origins.length * destinations.length > 25) {
      throw new Error('Mapbox Matrix API supports max 25 coordinates total');
    }

    const coords = [
      ...origins.map(o => `${o.lng},${o.lat}`),
      ...destinations.map(d => `${d.lng},${d.lat}`)
    ].join(';');

    const sources = Array.from({ length: origins.length }, (_, i) => i).join(';');
    const destinationsIndices = Array.from(
      { length: destinations.length },
      (_, i) => origins.length + i
    ).join(';');

    const url = `${this.baseUrl}/directions-matrix/v1/mapbox/driving/${coords}?sources=${sources}&destinations=${destinationsIndices}&access_token=${this.accessToken}`;

    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`Mapbox API error: ${response.statusText}`);
    }

    const data = await response.json();

    return {
      rows: data.durations.map((row: number[], rowIndex: number) => ({
        elements: row.map((duration: number, colIndex: number) => ({
          distance_meters: data.distances[rowIndex][colIndex],
          duration_seconds: duration,
          status: duration >= 0 ? 'OK' : 'NOT_FOUND'
        }))
      }))
    };
  }
}
```

### 3.5 Caching Strategy

**Problem**: Routing API calls are expensive and rate-limited.

**Solution**: Cache route results in database or Redis.

**Caching Approach**:

1. **Database Table**: `route_cache`
2. **Cache Key**: Hash of origin + destination coordinates (rounded to ~100m precision)
3. **TTL**: 24 hours for routes, 1 hour for distance matrix
4. **Invalidation**: Manual invalidation on location updates

**Cache Table Schema**:

```sql
CREATE TABLE route_cache (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cache_key TEXT NOT NULL UNIQUE,
  origin_lat NUMERIC(10, 8) NOT NULL,
  origin_lng NUMERIC(11, 8) NOT NULL,
  destination_lat NUMERIC(10, 8) NOT NULL,
  destination_lng NUMERIC(11, 8) NOT NULL,
  distance_meters INTEGER NOT NULL,
  duration_seconds INTEGER NOT NULL,
  polyline TEXT,
  cached_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at TIMESTAMPTZ NOT NULL,
  
  CONSTRAINT chk_route_cache_expires_after_cached CHECK (
    expires_at > cached_at
  )
);

CREATE INDEX idx_route_cache_key ON route_cache(cache_key);
CREATE INDEX idx_route_cache_expires_at ON route_cache(expires_at);
```

**Cache Key Generation**:

```typescript
function generateCacheKey(
  origin: { lat: number; lng: number },
  destination: { lat: number; lng: number }
): string {
  // Round to ~100m precision (4 decimal places ≈ 11m precision)
  const roundedOrigin = {
    lat: Math.round(origin.lat * 10000) / 10000,
    lng: Math.round(origin.lng * 10000) / 10000
  };
  const roundedDest = {
    lat: Math.round(destination.lat * 10000) / 10000,
    lng: Math.round(destination.lng * 10000) / 10000
  };
  
  return `${roundedOrigin.lat},${roundedOrigin.lng}:${roundedDest.lat},${roundedDest.lng}`;
}
```

**Cache Implementation** (deferred to later epic, but interface defined):

```typescript
async function getCachedRoute(
  supabase: SupabaseClient,
  origin: { lat: number; lng: number },
  destination: { lat: number; lng: number }
): Promise<RouteResult | null> {
  const cacheKey = generateCacheKey(origin, destination);
  
  const { data: cached } = await supabase
    .from('route_cache')
    .select('distance_meters, duration_seconds, polyline')
    .eq('cache_key', cacheKey)
    .gt('expires_at', new Date().toISOString())
    .single();

  if (cached) {
    return {
      distance_meters: cached.distance_meters,
      duration_seconds: cached.duration_seconds,
      polyline: cached.polyline || undefined
    };
  }

  return null;
}

async function cacheRoute(
  supabase: SupabaseClient,
  origin: { lat: number; lng: number },
  destination: { lat: number; lng: number },
  result: RouteResult,
  ttlHours: number = 24
): Promise<void> {
  const cacheKey = generateCacheKey(origin, destination);
  const expiresAt = new Date();
  expiresAt.setHours(expiresAt.getHours() + ttlHours);

  await supabase
    .from('route_cache')
    .upsert({
      cache_key: cacheKey,
      origin_lat: origin.lat,
      origin_lng: origin.lng,
      destination_lat: destination.lat,
      destination_lng: destination.lng,
      distance_meters: result.distance_meters,
      duration_seconds: result.duration_seconds,
      polyline: result.polyline,
      expires_at: expiresAt.toISOString()
    }, {
      onConflict: 'cache_key'
    });
}
```

### 3.6 Rate Limits and Cost Considerations

**Mapbox Rate Limits**:
- Free tier: 100,000 requests/month
- Matrix API: Up to 25 coordinates per request
- Directions API: Unlimited requests (within monthly limit)

**Cost Optimization**:
- Use caching aggressively
- Batch requests when possible
- Use Matrix API for multiple routes
- Monitor usage via Mapbox dashboard

**Cost Monitoring**:
- Track API calls in `optimization_runs` metadata
- Alert when approaching limits
- Consider upgrading plan if needed

---

## 4. Story DISP-037: Auto-Schedule Single Job Edge Function

### 4.1 POST /dispatch/jobs/:id/auto_schedule

**Purpose**: Automatically schedule a job by finding the best technician and time slot.

**Authorization**: `admin`, `dispatcher`

**Request Schema**:

```typescript
interface AutoScheduleJobRequest {
  mode: 'propose' | 'commit'; // default: 'propose'
  preferred_time?: string; // ISO 8601 timestamp, optional hint
  exclude_technician_ids?: string[]; // UUIDs, optional exclusion list
  max_candidates?: number; // default: 5, max candidates to return
}
```

**Request Example**:

```json
{
  "mode": "propose",
  "preferred_time": "2024-01-20T10:00:00Z",
  "max_candidates": 3
}
```

**Response Schema** (Propose Mode):

```typescript
interface AutoScheduleProposalResponse {
  job_id: string;
  recommendations: Array<{
    technician_id: string;
    technician_name: string;
    scheduled_start_at: string;
    scheduled_end_at: string;
    score: number; // 0-100, higher is better
    rationale: string; // Human-readable explanation
    warnings?: string[]; // Optional warnings
  }>;
  best_recommendation: {
    technician_id: string;
    scheduled_start_at: string;
    scheduled_end_at: string;
  };
}
```

**Response Example** (Propose Mode):

```json
{
  "data": {
    "job_id": "444e4567-e89b-12d3-a456-426614174000",
    "recommendations": [
      {
        "technician_id": "777e4567-e89b-12d3-a456-426614174000",
        "technician_name": "John Smith",
        "scheduled_start_at": "2024-01-20T09:00:00Z",
        "scheduled_end_at": "2024-01-20T10:00:00Z",
        "score": 85,
        "rationale": "Best match: Skills match (100%), Zone match (100%), Available, Low current load (2 jobs today)",
        "warnings": []
      },
      {
        "technician_id": "888e4567-e89b-12d3-a456-426614174000",
        "technician_name": "Jane Doe",
        "scheduled_start_at": "2024-01-20T10:00:00Z",
        "scheduled_end_at": "2024-01-20T11:00:00Z",
        "score": 72,
        "rationale": "Good match: Skills match (80%), Zone match (100%), Available, Medium current load (4 jobs today)",
        "warnings": ["Slightly outside preferred time window"]
      }
    ],
    "best_recommendation": {
      "technician_id": "777e4567-e89b-12d3-a456-426614174000",
      "scheduled_start_at": "2024-01-20T09:00:00Z",
      "scheduled_end_at": "2024-01-20T10:00:00Z"
    }
  }
}
```

**Response Schema** (Commit Mode):

```typescript
interface AutoScheduleCommitResponse {
  job_id: string;
  assignment_id: string;
  technician_id: string;
  scheduled_start_at: string;
  scheduled_end_at: string;
  score: number;
  rationale: string;
}
```

### 4.2 Auto-Scheduling Algorithm

**Algorithm Overview**:

1. **Get Job Constraints**: Skills, zone, SLA, time windows, priority
2. **Find Candidate Technicians**: Filter by skills, zone, availability
3. **Score Each Candidate**: Calculate score based on multiple factors
4. **Select Best Time Slot**: For each candidate, find best available time
5. **Rank Recommendations**: Sort by score, return top N
6. **Commit or Propose**: Create assignment if commit mode, return proposals otherwise

**Scoring Factors**:

1. **Skill Match** (0-40 points):
   - Perfect match: 40 points
   - Partial match: 20-30 points
   - No match: 0 points (excluded)

2. **Zone Match** (0-20 points):
   - Primary zone: 20 points
   - Secondary zone: 15 points
   - No zone match: 0 points (excluded if zone required)

3. **Proximity** (0-15 points):
   - Based on travel time from last job or home base
   - Closer = higher score

4. **Current Load** (0-15 points):
   - Lower load = higher score
   - Based on scheduled minutes today vs capacity

5. **Time Window Match** (0-10 points):
   - Within selected time window: 10 points
   - Within SLA window: 5 points
   - Outside SLA: 0 points (excluded)

**Scoring Implementation**:

```typescript
interface CandidateTechnician {
  technician_id: string;
  technician_name: string;
  skills: string[];
  zones: string[];
  home_base: { lat: number; lng: number } | null;
  current_load_minutes: number;
  max_daily_work_minutes: number | null;
  available_slots: Array<{
    starts_at: Date;
    ends_at: Date;
  }>;
}

interface ScoringResult {
  technician_id: string;
  score: number;
  breakdown: {
    skill_match: number;
    zone_match: number;
    proximity: number;
    current_load: number;
    time_window_match: number;
  };
  rationale: string;
}

function scoreTechnician(
  candidate: CandidateTechnician,
  job: {
    required_skills: string[];
    service_zone_id: string | null;
    location: { lat: number; lng: number };
    selected_time_window: { start: Date; end: Date } | null;
    sla_start_at: Date | null;
    sla_end_at: Date | null;
    priority: string;
  },
  travelTimeFromLastJob: number | null
): ScoringResult {
  let skillMatch = 0;
  let zoneMatch = 0;
  let proximity = 0;
  let currentLoad = 0;
  let timeWindowMatch = 0;

  // Skill match (0-40 points)
  if (job.required_skills && job.required_skills.length > 0) {
    const matchingSkills = job.required_skills.filter(skill =>
      candidate.skills.includes(skill)
    );
    const matchRatio = matchingSkills.length / job.required_skills.length;
    skillMatch = Math.round(matchRatio * 40);
    
    // Exclude if no skills match
    if (skillMatch === 0) {
      return {
        technician_id: candidate.technician_id,
        score: 0,
        breakdown: { skill_match: 0, zone_match: 0, proximity: 0, current_load: 0, time_window_match: 0 },
        rationale: 'No matching skills'
      };
    }
  } else {
    skillMatch = 40; // No skills required, full points
  }

  // Zone match (0-20 points)
  if (job.service_zone_id) {
    if (candidate.zones.includes(job.service_zone_id)) {
      // Check if primary zone
      zoneMatch = 20; // Simplified, would check is_primary flag
    } else {
      return {
        technician_id: candidate.technician_id,
        score: 0,
        breakdown: { skill_match: skillMatch, zone_match: 0, proximity: 0, current_load: 0, time_window_match: 0 },
        rationale: 'Not in required service zone'
      };
    }
  } else {
    zoneMatch = 20; // No zone required, full points
  }

  // Proximity (0-15 points)
  if (travelTimeFromLastJob !== null) {
    // Closer = higher score
    // Assume max travel time of 60 minutes = 0 points, 0 minutes = 15 points
    proximity = Math.max(0, Math.round(15 * (1 - travelTimeFromLastJob / 60)));
  } else if (candidate.home_base) {
    // Use home base distance
    // Simplified: would calculate actual distance
    proximity = 10; // Default middle score
  } else {
    proximity = 7; // No location data, lower score
  }

  // Current load (0-15 points)
  if (candidate.max_daily_work_minutes) {
    const utilizationRatio = candidate.current_load_minutes / candidate.max_daily_work_minutes;
    currentLoad = Math.round(15 * (1 - utilizationRatio)); // Lower utilization = higher score
  } else {
    currentLoad = 10; // No capacity limit, middle score
  }

  // Time window match (0-10 points)
  if (job.selected_time_window) {
    // Would check if available slot overlaps with selected window
    timeWindowMatch = 10; // Simplified
  } else if (job.sla_start_at && job.sla_end_at) {
    // Check if available slot is within SLA
    timeWindowMatch = 5; // Simplified
  } else {
    timeWindowMatch = 5; // No time constraints, middle score
  }

  const totalScore = skillMatch + zoneMatch + proximity + currentLoad + timeWindowMatch;

  return {
    technician_id: candidate.technician_id,
    score: totalScore,
    breakdown: {
      skill_match: skillMatch,
      zone_match: zoneMatch,
      proximity: proximity,
      current_load: currentLoad,
      time_window_match: timeWindowMatch
    },
    rationale: `Skills: ${skillMatch}/40, Zone: ${zoneMatch}/20, Proximity: ${proximity}/15, Load: ${currentLoad}/15, Time: ${timeWindowMatch}/10`
  };
}
```

### 4.3 Finding Available Time Slots

**Algorithm**:

1. Get technician shifts for the day
2. Get existing assignments for the day
3. Get time-off for the day
4. Calculate available slots (shifts minus assignments minus time-off)
5. Filter slots by job constraints (duration, SLA, time windows)
6. Score each slot based on preferences

**Implementation**:

```typescript
interface TimeSlot {
  starts_at: Date;
  ends_at: Date;
  available_minutes: number;
}

async function findAvailableSlots(
  supabase: SupabaseClient,
  technicianId: string,
  orgId: string,
  date: Date,
  jobDurationMinutes: number,
  slaStart: Date | null,
  slaEnd: Date | null,
  selectedWindow: { start: Date; end: Date } | null
): Promise<TimeSlot[]> {
  const dayStart = new Date(date);
  dayStart.setHours(0, 0, 0, 0);
  const dayEnd = new Date(date);
  dayEnd.setHours(23, 59, 59, 999);

  // Get shifts
  const { data: shifts } = await supabase
    .from('technician_shifts')
    .select('starts_at, ends_at')
    .eq('org_id', orgId)
    .eq('technician_id', technicianId)
    .eq('is_active', true)
    .gte('starts_at', dayStart.toISOString())
    .lte('starts_at', dayEnd.toISOString());

  // Get existing assignments
  const { data: assignments } = await supabase
    .from('job_assignments')
    .select('scheduled_start_at, scheduled_end_at')
    .eq('org_id', orgId)
    .eq('technician_id', technicianId)
    .in('status', ['assigned', 'accepted', 'en_route', 'on_site'])
    .gte('scheduled_start_at', dayStart.toISOString())
    .lte('scheduled_start_at', dayEnd.toISOString());

  // Get time-off
  const { data: timeOff } = await supabase
    .from('technician_time_off')
    .select('starts_at, ends_at')
    .eq('org_id', orgId)
    .eq('technician_id', technicianId)
    .lte('starts_at', dayEnd.toISOString())
    .gte('ends_at', dayStart.toISOString());

  // Calculate available slots
  const slots: TimeSlot[] = [];

  if (!shifts || shifts.length === 0) {
    return slots; // No shifts = no availability
  }

  for (const shift of shifts) {
    const shiftStart = new Date(shift.starts_at);
    const shiftEnd = new Date(shift.ends_at);

    // Start with full shift
    let currentStart = shiftStart;

    // Sort assignments and time-off by start time
    const blocks = [
      ...(assignments || []).map(a => ({
        start: new Date(a.scheduled_start_at),
        end: new Date(a.scheduled_end_at)
      })),
      ...(timeOff || []).map(t => ({
        start: new Date(t.starts_at),
        end: new Date(t.ends_at)
      }))
    ].sort((a, b) => a.start.getTime() - b.start.getTime());

    // Find gaps
    for (const block of blocks) {
      if (block.start > currentStart) {
        // Gap found
        const gapEnd = block.start < shiftEnd ? block.start : shiftEnd;
        const gapMinutes = (gapEnd.getTime() - currentStart.getTime()) / (1000 * 60);

        if (gapMinutes >= jobDurationMinutes) {
          // Check SLA constraints
          const slotStart = currentStart;
          const slotEnd = new Date(slotStart.getTime() + jobDurationMinutes * 60000);

          let valid = true;

          if (slaStart && slotStart < slaStart) valid = false;
          if (slaEnd && slotEnd > slaEnd) valid = false;
          if (selectedWindow) {
            if (slotStart < selectedWindow.start || slotEnd > selectedWindow.end) {
              valid = false;
            }
          }

          if (valid) {
            slots.push({
              starts_at: slotStart,
              ends_at: slotEnd,
              available_minutes: gapMinutes
            });
          }
        }
      }

      // Move current start to end of block
      if (block.end > currentStart) {
        currentStart = block.end;
      }
    }

    // Check gap after last block
    if (currentStart < shiftEnd) {
      const gapMinutes = (shiftEnd.getTime() - currentStart.getTime()) / (1000 * 60);
      if (gapMinutes >= jobDurationMinutes) {
        const slotStart = currentStart;
        const slotEnd = new Date(slotStart.getTime() + jobDurationMinutes * 60000);

        let valid = true;
        if (slaStart && slotStart < slaStart) valid = false;
        if (slaEnd && slotEnd > slaEnd) valid = false;
        if (selectedWindow) {
          if (slotStart < selectedWindow.start || slotEnd > selectedWindow.end) {
            valid = false;
          }
        }

        if (valid) {
          slots.push({
            starts_at: slotStart,
            ends_at: slotEnd,
            available_minutes: gapMinutes
          });
        }
      }
    }
  }

  return slots;
}
```

### 4.4 Implementation (Edge Function)

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
  const maxCandidates = Math.min(body.max_candidates || 5, 10);

  // Get job details
  const { data: job, error: jobError } = await supabase
    .from('dispatch_jobs')
    .select(`
      *,
      job_time_windows(*),
      customer_locations!inner(latitude, longitude)
    `)
    .eq('id', jobId)
    .eq('org_id', auth.orgId)
    .single();

  if (jobError || !job) {
    return errorResponse('Job not found', 404, 'JOB_NOT_FOUND');
  }

  if (job.status === 'completed' || job.status === 'canceled') {
    return errorResponse('Cannot schedule completed or canceled job', 400, 'INVALID_JOB_STATUS');
  }

  // Get job location
  const jobLocation = {
    lat: parseFloat(job.customer_locations.latitude),
    lng: parseFloat(job.customer_locations.longitude)
  };

  if (isNaN(jobLocation.lat) || isNaN(jobLocation.lng)) {
    return errorResponse('Job location missing coordinates', 400, 'MISSING_COORDINATES');
  }

  // Get selected time window
  const selectedWindow = job.job_time_windows?.find((w: any) => w.is_selected);
  const windowStart = selectedWindow ? new Date(selectedWindow.window_start) : null;
  const windowEnd = selectedWindow ? new Date(selectedWindow.window_end) : null;

  // Get SLA window
  const slaStart = job.sla_start_at ? new Date(job.sla_start_at) : null;
  const slaEnd = job.sla_end_at ? new Date(job.sla_end_at) : null;

  // Find candidate technicians
  let techQuery = supabase
    .from('technician_profiles')
    .select(`
      id,
      display_name,
      max_daily_work_minutes,
      home_base_location_id,
      customer_locations(latitude, longitude),
      technician_skills(skill_code),
      technician_service_zones(service_zone_id, is_primary)
    `)
    .eq('org_id', auth.orgId)
    .eq('is_active', true);

  // Filter by excluded technicians
  if (body.exclude_technician_ids && body.exclude_technician_ids.length > 0) {
    techQuery = techQuery.not('id', 'in', `(${body.exclude_technician_ids.join(',')})`);
  }

  const { data: technicians, error: techError } = await techQuery;

  if (techError) {
    return errorResponse('Failed to fetch technicians', 500, 'FETCH_ERROR');
  }

  if (!technicians || technicians.length === 0) {
    return errorResponse('No available technicians', 404, 'NO_TECHNICIANS');
  }

  // Initialize routing provider
  const routingProvider = new MapboxRoutingProvider();

  // Score each technician
  const candidates: Array<{
    technician: any;
    scores: ScoringResult[];
    bestSlot: TimeSlot | null;
  }> = [];

  for (const tech of technicians) {
    // Get technician skills
    const techSkills = tech.technician_skills?.map((s: any) => s.skill_code) || [];
    
    // Get technician zones
    const techZones = tech.technician_service_zones?.map((z: any) => z.service_zone_id) || [];

    // Check skill match
    if (job.required_skills && job.required_skills.length > 0) {
      const hasRequiredSkills = job.required_skills.some(skill => techSkills.includes(skill));
      if (!hasRequiredSkills) {
        continue; // Skip technicians without required skills
      }
    }

    // Check zone match
    if (job.service_zone_id && !techZones.includes(job.service_zone_id)) {
      continue; // Skip technicians not in required zone
    }

    // Get current load
    const today = new Date();
    today.setHours(0, 0, 0, 0);
    const { data: todayAssignments } = await supabase
      .from('job_assignments')
      .select('scheduled_start_at, scheduled_end_at')
      .eq('org_id', auth.orgId)
      .eq('technician_id', tech.id)
      .in('status', ['assigned', 'accepted', 'en_route', 'on_site'])
      .gte('scheduled_start_at', today.toISOString())
      .lt('scheduled_start_at', new Date(today.getTime() + 86400000).toISOString());

    let currentLoadMinutes = 0;
    if (todayAssignments) {
      for (const assignment of todayAssignments) {
        const start = new Date(assignment.scheduled_start_at);
        const end = new Date(assignment.scheduled_end_at);
        currentLoadMinutes += (end.getTime() - start.getTime()) / (1000 * 60);
      }
    }

    // Get technician home base or last job location
    let techLocation: { lat: number; lng: number } | null = null;
    if (tech.customer_locations?.latitude && tech.customer_locations?.longitude) {
      techLocation = {
        lat: parseFloat(tech.customer_locations.latitude),
        lng: parseFloat(tech.customer_locations.longitude)
      };
    }

    // Calculate travel time (simplified, would use routing provider)
    let travelTimeMinutes: number | null = null;
    if (techLocation) {
      try {
        const route = await routingProvider.getRoute(techLocation, jobLocation);
        travelTimeMinutes = route.duration_seconds / 60;
      } catch (error) {
        console.warn('Failed to get route:', error);
      }
    }

    // Find available slots
    const targetDate = body.preferred_time ? new Date(body.preferred_time) : new Date();
    const availableSlots = await findAvailableSlots(
      supabase,
      tech.id,
      auth.orgId,
      targetDate,
      job.estimated_duration_minutes,
      slaStart,
      slaEnd,
      windowStart && windowEnd ? { start: windowStart, end: windowEnd } : null
    );

    if (availableSlots.length === 0) {
      continue; // No available slots
    }

    // Score technician for each slot
    const slotScores = availableSlots.map(slot => {
      const candidate: CandidateTechnician = {
        technician_id: tech.id,
        technician_name: tech.display_name,
        skills: techSkills,
        zones: techZones,
        home_base: techLocation,
        current_load_minutes: currentLoadMinutes,
        max_daily_work_minutes: tech.max_daily_work_minutes,
        available_slots: [slot]
      };

      return scoreTechnician(
        candidate,
        {
          required_skills: job.required_skills || [],
          service_zone_id: job.service_zone_id,
          location: jobLocation,
          selected_time_window: windowStart && windowEnd ? { start: windowStart, end: windowEnd } : null,
          sla_start_at: slaStart,
          sla_end_at: slaEnd,
          priority: job.priority
        },
        travelTimeMinutes
      );
    });

    // Find best slot
    const bestSlotScore = slotScores.reduce((best, current) =>
      current.score > best.score ? current : best
    );
    const bestSlot = availableSlots[slotScores.indexOf(bestSlotScore)];

    candidates.push({
      technician: tech,
      scores: slotScores,
      bestSlot: bestSlot || null
    });
  }

  // Sort by best score
  candidates.sort((a, b) => {
    const scoreA = a.scores.length > 0 ? Math.max(...a.scores.map(s => s.score)) : 0;
    const scoreB = b.scores.length > 0 ? Math.max(...b.scores.map(s => s.score)) : 0;
    return scoreB - scoreA;
  });

  // Take top N candidates
  const topCandidates = candidates.slice(0, maxCandidates);

  // Format recommendations
  const recommendations = topCandidates.map(candidate => {
    const bestScore = candidate.scores.length > 0
      ? candidate.scores.reduce((best, current) => current.score > best.score ? current : best)
      : null;

    if (!bestScore || !candidate.bestSlot) {
      return null;
    }

    return {
      technician_id: candidate.technician.id,
      technician_name: candidate.technician.display_name,
      scheduled_start_at: candidate.bestSlot.starts_at.toISOString(),
      scheduled_end_at: candidate.bestSlot.ends_at.toISOString(),
      score: bestScore.score,
      rationale: bestScore.rationale,
      warnings: [] // Would add warnings for capacity, overlaps, etc.
    };
  }).filter(r => r !== null);

  if (recommendations.length === 0) {
    return errorResponse('No suitable technicians found', 404, 'NO_CANDIDATES');
  }

  // If commit mode, create assignment
  if (mode === 'commit') {
    const bestRecommendation = recommendations[0];
    
    const { data: assignment, error: assignError } = await supabase
      .from('job_assignments')
      .insert({
        org_id: auth.orgId,
        dispatch_job_id: jobId,
        technician_id: bestRecommendation.technician_id,
        scheduled_start_at: bestRecommendation.scheduled_start_at,
        scheduled_end_at: bestRecommendation.scheduled_end_at,
        status: 'assigned',
        assigned_by_user_id: user.id,
        is_primary_technician: true
      })
      .select()
      .single();

    if (assignError) {
      return errorResponse('Failed to create assignment', 500, 'CREATE_ERROR', { error: assignError.message });
    }

    // Update job status
    await updateJobStatusFromAssignments(supabase, jobId, auth.orgId);

    return successResponse({
      job_id: jobId,
      assignment_id: assignment.id,
      technician_id: assignment.technician_id,
      scheduled_start_at: assignment.scheduled_start_at,
      scheduled_end_at: assignment.scheduled_end_at,
      score: bestRecommendation.score,
      rationale: bestRecommendation.rationale
    });
  }

  // Propose mode: return recommendations
  return successResponse({
    job_id: jobId,
    recommendations: recommendations,
    best_recommendation: {
      technician_id: recommendations[0].technician_id,
      scheduled_start_at: recommendations[0].scheduled_start_at,
      scheduled_end_at: recommendations[0].scheduled_end_at
    }
  });
});
```

### 4.5 Idempotency

**Idempotency Key**: Use job ID + timestamp (or hash of job constraints) as idempotency key.

**Strategy**: Check for existing assignment before creating new one in commit mode.

```typescript
// Check for existing assignment
const { data: existingAssignment } = await supabase
  .from('job_assignments')
  .select('id')
  .eq('dispatch_job_id', jobId)
  .eq('org_id', auth.orgId)
  .in('status', ['assigned', 'accepted', 'en_route', 'on_site'])
  .single();

if (existingAssignment) {
  return errorResponse('Job already has active assignment', 409, 'DUPLICATE_ASSIGNMENT');
}
```

---

## 5. Story DISP-038: Suggest Technician Candidates for a Job

### 5.1 POST /dispatch/jobs/:id/suggest_assignments

**Purpose**: Get ranked list of technician candidates with scores and rationales.

**Authorization**: `admin`, `dispatcher`

**Request Schema**:

```typescript
interface SuggestAssignmentsRequest {
  preferred_time?: string; // ISO 8601 timestamp, optional
  max_candidates?: number; // default: 10, max: 20
}
```

**Response Schema**:

```typescript
interface SuggestAssignmentsResponse {
  job_id: string;
  candidates: Array<{
    technician_id: string;
    technician_name: string;
    score: number; // 0-100
    score_breakdown: {
      skill_match: number;
      zone_match: number;
      proximity: number;
      current_load: number;
      time_window_match: number;
    };
    rationale: string;
    suggested_slots: Array<{
      starts_at: string;
      ends_at: string;
      score: number;
    }>;
    warnings?: string[];
  }>;
}
```

**Implementation**: Reuse scoring logic from DISP-037, but return all candidates with detailed breakdowns.

---

## 6. Story DISP-039: Optimize Route for One Technician

### 6.1 POST /dispatch/technicians/:id/optimize_route

**Purpose**: Optimize a technician's route for a given day.

**Authorization**: `admin`, `dispatcher`

**Request Schema**:

```typescript
interface OptimizeRouteRequest {
  date: string; // ISO 8601 date (YYYY-MM-DD), required
  mode?: 'propose' | 'commit'; // default: 'propose'
  optimization_strategy?: 'time_minimization' | 'distance_minimization' | 'priority_first' | 'balanced'; // default: 'balanced'
  assignment_ids?: string[]; // Optional: specific assignments to optimize (default: all for date)
  respect_locks?: boolean; // default: true, respect manual locks
}
```

**Request Example**:

```json
{
  "date": "2024-01-20",
  "mode": "commit",
  "optimization_strategy": "time_minimization",
  "respect_locks": true
}
```

**Response Schema** (Propose Mode):

```typescript
interface OptimizeRouteProposalResponse {
  technician_id: string;
  date: string;
  current_route: {
    total_distance_km: number;
    total_travel_time_minutes: number;
    total_job_time_minutes: number;
    stops: Array<{
      assignment_id: string | null;
      sequence: number;
      planned_arrival_at: string;
      planned_departure_at: string;
    }>;
  };
  optimized_route: {
    total_distance_km: number;
    total_travel_time_minutes: number;
    total_job_time_minutes: number;
    estimated_savings_minutes: number;
    estimated_savings_km: number;
    stops: Array<{
      assignment_id: string | null;
      sequence: number;
      planned_arrival_at: string;
      planned_departure_at: string;
    }>;
  };
}
```

**Response Schema** (Commit Mode):

```typescript
interface OptimizeRouteCommitResponse {
  route_plan_id: string;
  technician_id: string;
  date: string;
  total_distance_km: number;
  total_travel_time_minutes: number;
  total_job_time_minutes: number;
  estimated_savings_minutes: number;
  estimated_savings_km: number;
  stops_updated: number;
}
```

### 6.2 Route Optimization Algorithm

**Algorithm Overview**:

1. **Get Assignments**: Fetch all assignments for technician on date
2. **Build Cost Matrix**: Calculate travel time/distance between all locations
3. **Solve TSP**: Use optimization algorithm (heuristic or exact) to find best sequence
4. **Update Sequence**: Update `sequence_in_route` and times for assignments
5. **Create Route Plan**: Create/update `route_plans` and `route_stops`

**TSP Solver Options**:

1. **Nearest Neighbor** (Simple heuristic):
   - Start from depot/home base
   - Always go to nearest unvisited job
   - Fast but not optimal

2. **2-Opt** (Local search):
   - Start with initial route
   - Try swapping edges to improve
   - Better than nearest neighbor

3. **OR-Tools** (Google's optimization library):
   - Can be called via external API or embedded
   - Provides optimal or near-optimal solutions
   - More complex setup

**MVP Approach**: Use Nearest Neighbor with 2-Opt improvement (simple, fast, good enough for MVP).

**Implementation**:

```typescript
interface RouteStop {
  assignment_id: string | null;
  location: { lat: number; lng: number };
  job_duration_minutes: number;
  priority: string;
  sla_end_at: Date | null;
}

interface OptimizedSequence {
  stops: Array<{
    assignment_id: string | null;
    sequence: number;
    planned_arrival_at: Date;
    planned_departure_at: Date;
    travel_time_from_prev_minutes: number;
    distance_from_prev_km: number;
  }>;
  total_distance_km: number;
  total_travel_time_minutes: number;
}

async function optimizeRouteSequence(
  routingProvider: RoutingProvider,
  depot: { lat: number; lng: number },
  stops: RouteStop[],
  strategy: 'time_minimization' | 'distance_minimization' | 'priority_first' | 'balanced',
  startTime: Date
): Promise<OptimizedSequence> {
  if (stops.length === 0) {
    return {
      stops: [],
      total_distance_km: 0,
      total_travel_time_minutes: 0
    };
  }

  // Build cost matrix
  const locations = [depot, ...stops.map(s => s.location)];
  const costMatrix: number[][] = [];

  for (let i = 0; i < locations.length; i++) {
    costMatrix[i] = [];
    for (let j = 0; j < locations.length; j++) {
      if (i === j) {
        costMatrix[i][j] = 0;
      } else {
        try {
          const route = await routingProvider.getRoute(locations[i], locations[j]);
          // Use time or distance based on strategy
          if (strategy === 'distance_minimization') {
            costMatrix[i][j] = route.distance_meters / 1000; // km
          } else {
            costMatrix[i][j] = route.duration_seconds / 60; // minutes
          }
        } catch (error) {
          // Fallback: use straight-line distance
          const dist = haversineDistance(locations[i], locations[j]);
          costMatrix[i][j] = strategy === 'distance_minimization' ? dist : dist * 0.02; // Estimate 2 min/km
        }
      }
    }
  }

  // Nearest Neighbor algorithm
  const visited = new Set<number>();
  const sequence: number[] = [0]; // Start at depot (index 0)
  visited.add(0);

  let current = 0;
  let totalCost = 0;

  while (visited.size < locations.length) {
    let nearest = -1;
    let nearestCost = Infinity;

    for (let i = 1; i < locations.length; i++) {
      if (!visited.has(i)) {
        let cost = costMatrix[current][i];
        
        // Adjust cost based on strategy
        if (strategy === 'priority_first') {
          const stop = stops[i - 1];
          if (stop.priority === 'emergency') cost *= 0.1;
          else if (stop.priority === 'high') cost *= 0.5;
        }

        if (cost < nearestCost) {
          nearestCost = cost;
          nearest = i;
        }
      }
    }

    if (nearest === -1) break;

    sequence.push(nearest);
    visited.add(nearest);
    totalCost += nearestCost;
    current = nearest;
  }

  // Return to depot
  sequence.push(0);
  totalCost += costMatrix[current][0];

  // Build optimized sequence with times
  const optimizedStops: OptimizedSequence['stops'] = [];
  let currentTime = new Date(startTime);

  for (let i = 1; i < sequence.length - 1; i++) {
    const stopIndex = sequence[i] - 1; // -1 because depot is index 0
    const stop = stops[stopIndex];
    const prevIndex = sequence[i - 1];
    
    const travelTime = costMatrix[prevIndex][sequence[i]];
    const travelDistance = costMatrix[prevIndex][sequence[i]]; // Simplified, would use actual distance

    const arrivalTime = new Date(currentTime.getTime() + travelTime * 60000);
    const departureTime = new Date(arrivalTime.getTime() + stop.job_duration_minutes * 60000);

    optimizedStops.push({
      assignment_id: stop.assignment_id || null,
      sequence: i - 1, // 0-indexed, excluding depot
      planned_arrival_at: arrivalTime,
      planned_departure_at: departureTime,
      travel_time_from_prev_minutes: travelTime,
      distance_from_prev_km: travelDistance
    });

    currentTime = departureTime;
  }

  return {
    stops: optimizedStops,
    total_distance_km: totalCost, // Simplified
    total_travel_time_minutes: totalCost // Simplified, would separate distance and time
  };
}

function haversineDistance(
  point1: { lat: number; lng: number },
  point2: { lat: number; lng: number }
): number {
  const R = 6371; // Earth radius in km
  const dLat = (point2.lat - point1.lat) * Math.PI / 180;
  const dLng = (point2.lng - point1.lng) * Math.PI / 180;
  const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos(point1.lat * Math.PI / 180) * Math.cos(point2.lat * Math.PI / 180) *
    Math.sin(dLng / 2) * Math.sin(dLng / 2);
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  return R * c;
}
```

### 6.3 Implementation (Edge Function)

**Complete Implementation** (simplified, full version would include all error handling):

```typescript
Deno.serve(async (req) => {
  // ... authorization code ...

  const body = await req.json();
  const date = body.date;
  const mode = body.mode || 'propose';
  const strategy = body.optimization_strategy || 'balanced';
  const respectLocks = body.respect_locks !== false;

  // Get technician
  const { data: technician } = await supabase
    .from('technician_profiles')
    .select('id, display_name, home_base_location_id, customer_locations(latitude, longitude)')
    .eq('id', technicianId)
    .eq('org_id', auth.orgId)
    .single();

  // Get assignments for date
  const dateStart = new Date(date);
  dateStart.setHours(0, 0, 0, 0);
  const dateEnd = new Date(date);
  dateEnd.setHours(23, 59, 59, 999);

  let assignmentsQuery = supabase
    .from('job_assignments')
    .select(`
      id,
      dispatch_job_id,
      scheduled_start_at,
      scheduled_end_at,
      sequence_in_route,
      dispatch_jobs!inner(
        id,
        priority,
        estimated_duration_minutes,
        sla_end_at,
        customer_locations!inner(latitude, longitude)
      )
    `)
    .eq('org_id', auth.orgId)
    .eq('technician_id', technicianId)
    .in('status', ['assigned', 'accepted'])
    .gte('scheduled_start_at', dateStart.toISOString())
    .lte('scheduled_start_at', dateEnd.toISOString());

  if (body.assignment_ids && body.assignment_ids.length > 0) {
    assignmentsQuery = assignmentsQuery.in('id', body.assignment_ids);
  }

  const { data: assignments } = await assignmentsQuery;

  if (!assignments || assignments.length === 0) {
    return errorResponse('No assignments found for date', 404, 'NO_ASSIGNMENTS');
  }

  // Get depot location
  const depot = technician.customer_locations
    ? { lat: parseFloat(technician.customer_locations.latitude), lng: parseFloat(technician.customer_locations.longitude) }
    : { lat: 0, lng: 0 }; // Fallback

  // Build route stops
  const routeStops: RouteStop[] = assignments.map((assignment: any) => ({
    assignment_id: assignment.id,
    location: {
      lat: parseFloat(assignment.dispatch_jobs.customer_locations.latitude),
      lng: parseFloat(assignment.dispatch_jobs.customer_locations.longitude)
    },
    job_duration_minutes: assignment.dispatch_jobs.estimated_duration_minutes,
    priority: assignment.dispatch_jobs.priority,
    sla_end_at: assignment.dispatch_jobs.sla_end_at ? new Date(assignment.dispatch_jobs.sla_end_at) : null
  }));

  // Optimize route
  const routingProvider = new MapboxRoutingProvider();
  const optimized = await optimizeRouteSequence(
    routingProvider,
    depot,
    routeStops,
    strategy,
    dateStart
  );

  // Calculate current route metrics (for comparison)
  let currentDistance = 0;
  let currentTravelTime = 0;
  
  // Simplified: would calculate actual current route metrics

  if (mode === 'commit') {
    // Create/update route plan
    const { data: routePlan, error: routePlanError } = await supabase
      .from('route_plans')
      .upsert({
        org_id: auth.orgId,
        technician_id: technicianId,
        date: date,
        status: 'finalized',
        optimization_strategy: strategy,
        optimization_metadata: {
          total_distance_km: optimized.total_distance_km,
          total_travel_time_minutes: optimized.total_travel_time_minutes,
          optimized_at: new Date().toISOString()
        },
        generated_by: 'ai_optimizer',
        generated_by_user_id: user.id
      }, {
        onConflict: 'org_id,technician_id,date'
      })
      .select()
      .single();

    // Delete existing route stops
    await supabase
      .from('route_stops')
      .delete()
      .eq('route_plan_id', routePlan.id);

    // Create route stops
    const stopsToInsert = optimized.stops.map((stop, index) => ({
      org_id: auth.orgId,
      route_plan_id: routePlan.id,
      job_assignment_id: stop.assignment_id,
      stop_type: 'job',
      sequence: index + 1,
      planned_arrival_at: stop.planned_arrival_at.toISOString(),
      planned_departure_at: stop.planned_departure_at.toISOString(),
      travel_time_minutes_from_prev: stop.travel_time_from_prev_minutes,
      distance_km_from_prev: stop.distance_from_prev_km
    }));

    await supabase
      .from('route_stops')
      .insert(stopsToInsert);

    // Update assignments
    for (const stop of optimized.stops) {
      if (stop.assignment_id) {
        await supabase
          .from('job_assignments')
          .update({
            scheduled_start_at: stop.planned_arrival_at.toISOString(),
            scheduled_end_at: stop.planned_departure_at.toISOString(),
            sequence_in_route: stop.sequence
          })
          .eq('id', stop.assignment_id);
      }
    }

    return successResponse({
      route_plan_id: routePlan.id,
      technician_id: technicianId,
      date: date,
      total_distance_km: optimized.total_distance_km,
      total_travel_time_minutes: optimized.total_travel_time_minutes,
      total_job_time_minutes: assignments.reduce((sum: number, a: any) => sum + a.dispatch_jobs.estimated_duration_minutes, 0),
      estimated_savings_minutes: currentTravelTime - optimized.total_travel_time_minutes,
      estimated_savings_km: currentDistance - optimized.total_distance_km,
      stops_updated: optimized.stops.length
    });
  }

  // Propose mode
  return successResponse({
    technician_id: technicianId,
    date: date,
    current_route: {
      total_distance_km: currentDistance,
      total_travel_time_minutes: currentTravelTime,
      total_job_time_minutes: assignments.reduce((sum: number, a: any) => sum + a.dispatch_jobs.estimated_duration_minutes, 0),
      stops: assignments.map((a: any, index: number) => ({
        assignment_id: a.id,
        sequence: index,
        planned_arrival_at: a.scheduled_start_at,
        planned_departure_at: a.scheduled_end_at
      }))
    },
    optimized_route: {
      total_distance_km: optimized.total_distance_km,
      total_travel_time_minutes: optimized.total_travel_time_minutes,
      total_job_time_minutes: assignments.reduce((sum: number, a: any) => sum + a.dispatch_jobs.estimated_duration_minutes, 0),
      estimated_savings_minutes: currentTravelTime - optimized.total_travel_time_minutes,
      estimated_savings_km: currentDistance - optimized.total_distance_km,
      stops: optimized.stops.map(s => ({
        assignment_id: s.assignment_id,
        sequence: s.sequence,
        planned_arrival_at: s.planned_arrival_at.toISOString(),
        planned_departure_at: s.planned_departure_at.toISOString()
      }))
    }
  });
});
```

---

## 7. Story DISP-040: Bulk Daily Optimization

### 7.1 POST /dispatch/days/:date/optimize_all

**Purpose**: Optimize routes for all technicians for a day.

**Authorization**: `admin`, `dispatcher`

**Request Schema**:

```typescript
interface BulkOptimizeRequest {
  respect_locks?: boolean; // default: true
  optimization_strategy?: 'time_minimization' | 'distance_minimization' | 'priority_first' | 'balanced';
  mode?: 'propose' | 'commit'; // default: 'propose'
}
```

**Response Schema**:

```typescript
interface BulkOptimizeResponse {
  date: string;
  run_id: string; // For async tracking
  summary: {
    technicians_processed: number;
    technicians_optimized: number;
    total_distance_saved_km: number;
    total_time_saved_minutes: number;
  };
  technician_results: Array<{
    technician_id: string;
    optimized: boolean;
    distance_saved_km: number;
    time_saved_minutes: number;
  }>;
}
```

**Implementation**: Iterate over all active technicians, call optimize_route for each, aggregate results.

---

## 8. Story DISP-041: Asynchronous Optimization Jobs

### 8.1 Optimization Run Tracking Table

**Schema**:

```sql
CREATE TABLE optimization_runs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  run_type TEXT NOT NULL, -- 'single_job', 'single_technician', 'bulk_daily'
  target_id UUID, -- job_id, technician_id, or date (as text)
  status TEXT NOT NULL DEFAULT 'queued', -- 'queued', 'running', 'succeeded', 'failed'
  optimization_strategy TEXT,
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  error_message TEXT,
  result_metadata JSONB, -- Results summary
  created_by_user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_optimization_runs_org_id_status ON optimization_runs(org_id, status);
CREATE INDEX idx_optimization_runs_org_id_created_at ON optimization_runs(org_id, created_at DESC);
```

### 8.2 Async Processing Pattern

**Approach**: Return run ID immediately, process in background.

**Implementation**:

```typescript
// Endpoint returns immediately with run ID
const runId = crypto.randomUUID();

// Insert run record
await supabase
  .from('optimization_runs')
  .insert({
    id: runId,
    org_id: auth.orgId,
    run_type: 'bulk_daily',
    target_id: date,
    status: 'queued',
    optimization_strategy: strategy,
    created_by_user_id: user.id
  });

// Process asynchronously (would use Supabase Edge Function background job or queue)
processOptimizationRun(runId, date, strategy);

return successResponse({
  run_id: runId,
  status: 'queued',
  message: 'Optimization job queued'
});
```

### 8.3 Progress Reporting

**GET /dispatch/optimization-runs/:run_id**

**Response**:

```typescript
interface OptimizationRunResponse {
  id: string;
  status: 'queued' | 'running' | 'succeeded' | 'failed';
  progress?: {
    completed: number;
    total: number;
    percentage: number;
  };
  result_metadata?: any;
  error_message?: string;
  started_at: string | null;
  completed_at: string | null;
}
```

---

## 9. Error Handling

### 9.1 Standard Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `ROUTING_API_ERROR` | 502 | Routing provider API error |
| `NO_CANDIDATES` | 404 | No suitable technicians found |
| `MISSING_COORDINATES` | 400 | Job location missing coordinates |
| `OPTIMIZATION_FAILED` | 500 | Route optimization algorithm failed |
| `INVALID_STRATEGY` | 400 | Invalid optimization strategy |

---

## 10. Implementation Checklist

### Story DISP-036: Define Routing/Mapping Provider Strategy
- [ ] Provider evaluation documented
- [ ] Provider selected (Mapbox recommended)
- [ ] Abstraction interface defined
- [ ] Mapbox implementation created
- [ ] API key storage documented
- [ ] Caching strategy documented
- [ ] Rate limits and cost considerations documented

### Story DISP-037: Auto-Schedule Single Job
- [ ] POST /dispatch/jobs/:id/auto_schedule endpoint implemented
- [ ] Propose mode implemented
- [ ] Commit mode implemented
- [ ] Scoring algorithm implemented
- [ ] Available slot finding implemented
- [ ] Idempotency handled
- [ ] API documentation with examples

### Story DISP-038: Suggest Technician Candidates
- [ ] POST /dispatch/jobs/:id/suggest_assignments endpoint implemented
- [ ] Scoring with breakdown implemented
- [ ] Rationale generation implemented
- [ ] API documentation with examples

### Story DISP-039: Optimize Route for One Technician
- [ ] POST /dispatch/technicians/:id/optimize_route endpoint implemented
- [ ] Propose mode implemented
- [ ] Commit mode implemented
- [ ] TSP solver implemented (Nearest Neighbor + 2-Opt)
- [ ] Cost matrix building implemented
- [ ] Route plan creation implemented
- [ ] Route stops creation implemented
- [ ] Assignment updates implemented
- [ ] API documentation with examples

### Story DISP-040: Bulk Daily Optimization
- [ ] POST /dispatch/days/:date/optimize_all endpoint implemented
- [ ] Lock respect logic implemented
- [ ] Summary calculation implemented
- [ ] API documentation with examples

### Story DISP-041: Asynchronous Optimization Jobs
- [ ] optimization_runs table created
- [ ] Run tracking implemented
- [ ] Progress reporting endpoint implemented
- [ ] Async processing pattern documented
- [ ] API documentation with examples

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 7 – Auto-Scheduling and Route Optimization. All APIs are designed as Supabase Edge Functions with complete algorithms, scoring logic, and optimization strategies.

**Next Steps**: After completing Epic 7, proceed to Epic 8 (Emergency / Priority Job Handling) which will implement emergency job insertion workflows.

