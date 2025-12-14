# Technical Design Document – Epic 14: Reliability, Performance, and Observability

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 14 – Reliability, Performance, and Observability
- **Source**: Derived from `fdd_2_agile.md` Epic 14 (Stories DISP-060 through DISP-062)
- **Reference Documents**: 
  - `fdd_2.md` (Functional Design Document for Scheduling & Dispatch)
  - `fdd_2_agile.md` (Agile User Stories)
  - `tdd_2_epic_1.md` (Dispatch Epic 1 for tenancy/role conventions)
  - `tdd_2_epic_2.md` (Dispatch Epic 2 for table schemas)
  - `tdd_2_epic_3.md` (Dispatch Epic 3 for RLS policies)
  - `tdd_2_epic_5.md` (Dispatch Epic 5 for job lifecycle APIs)
  - `tdd_2_epic_7.md` (Dispatch Epic 7 for auto-scheduling and route optimization)
  - `tdd_2_epic_10.md` (Dispatch Epic 10 for notifications)
  - `tdd_2_epic_11.md` (Dispatch Epic 11 for Next.js UI patterns)
- **Target Platform**: PostgreSQL 15+, Supabase Edge Functions (Deno/TypeScript), Next.js 14+ (App Router), shadcn/ui
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing Reliability, Performance, and Observability features for the Scheduling & Dispatch module. It covers:

- Performance baseline and optimization strategies for schedule board queries
- Idempotency patterns for automation operations
- Structured logging and trace ID implementation
- Observability UI components for monitoring and debugging

All performance optimizations target sub-500ms response times for typical org sizes (dozens of technicians, hundreds of daily jobs). Idempotency ensures reliability during retries and duplicate triggers. Structured logging enables production debugging and correlation with persisted artifacts.

This epic assumes Epic 1-13 are complete and all dispatch functionality is operational.

---

## 2. Prerequisites

### 2.1 Required Components

Before implementing Epic 14, ensure:

1. **Epic 1-13 Complete**: All previous epics are implemented
2. **Required Tables**: All dispatch tables exist and are populated:
   - `dispatch_jobs`
   - `job_assignments`
   - `route_plans`
   - `route_stops`
   - `technician_profiles`
   - `technician_shifts`
   - `technician_time_off`
   - `optimization_runs`
   - `job_notifications`

3. **Performance Testing Data**: Realistic test data volumes:
   - 50-100 technicians per org
   - 500-1000 jobs per day per org
   - Historical data spanning 30+ days

4. **Logging Infrastructure**: Supabase Edge Functions logging enabled

---

## 3. Story DISP-060: Performance Baseline for Schedule Board Queries

### 3.1 Performance Targets

**Target Response Times** (from `fdd_2.md` §8):
- Schedule board queries: **< 500ms** for typical org sizes
- Route optimization: Async (long-running, with progress indicators)
- Real-time subscriptions: < 100ms latency for updates

**Typical Org Size Definition**:
- 50-100 active technicians
- 500-1000 jobs per day
- 2000-5000 assignments per day

### 3.2 Key Query Patterns

#### 3.2.1 Assignments by Day (Schedule Board)

**Query Pattern**: Load all assignments for a date range with technician and job details

**Base Query**:
```sql
SELECT 
  ja.id,
  ja.org_id,
  ja.technician_id,
  ja.dispatch_job_id,
  ja.scheduled_start_at,
  ja.scheduled_end_at,
  ja.status,
  ja.arrival_window_start,
  ja.arrival_window_end,
  tp.first_name || ' ' || tp.last_name AS technician_name,
  dj.title AS job_title,
  dj.priority,
  dj.status AS job_status,
  cl.address_line1,
  cl.city,
  cl.state,
  cl.postal_code
FROM job_assignments ja
INNER JOIN technician_profiles tp ON tp.id = ja.technician_id AND tp.org_id = ja.org_id
INNER JOIN dispatch_jobs dj ON dj.id = ja.dispatch_job_id AND dj.org_id = ja.org_id
LEFT JOIN customer_locations cl ON cl.id = dj.location_id AND cl.org_id = dj.org_id
WHERE ja.org_id = $1
  AND ja.scheduled_start_at >= $2::date
  AND ja.scheduled_start_at < ($2::date + INTERVAL '1 day')
  AND ja.status NOT IN ('canceled')
ORDER BY ja.technician_id, ja.scheduled_start_at;
```

**Performance Requirements**:
- Must execute in < 500ms for 2000 assignments per day
- Must support date range queries (1-7 days)

**Required Indexes**:
```sql
-- Primary index for date-based queries
CREATE INDEX IF NOT EXISTS idx_job_assignments_org_date_status 
  ON job_assignments(org_id, scheduled_start_at, status)
  WHERE status NOT IN ('canceled');

-- Composite index for technician schedule queries
CREATE INDEX IF NOT EXISTS idx_job_assignments_org_tech_date 
  ON job_assignments(org_id, technician_id, scheduled_start_at)
  WHERE status NOT IN ('canceled');

-- Covering index for common fields (reduces table lookups)
CREATE INDEX IF NOT EXISTS idx_job_assignments_covering 
  ON job_assignments(org_id, scheduled_start_at, status)
  INCLUDE (technician_id, dispatch_job_id, scheduled_end_at, arrival_window_start, arrival_window_end)
  WHERE status NOT IN ('canceled');
```

**Query Optimization Strategy**:
1. Use date range filtering with indexed `scheduled_start_at`
2. Filter canceled assignments at index level (partial index)
3. Use `INNER JOIN` for required relationships
4. Limit result set to visible date range (1-7 days)
5. Consider materialized view for frequently accessed date ranges

#### 3.2.2 Jobs by Status

**Query Pattern**: Load jobs filtered by status with assignment counts

**Base Query**:
```sql
SELECT 
  dj.id,
  dj.org_id,
  dj.title,
  dj.priority,
  dj.status,
  dj.created_at,
  dj.scheduled_start_at,
  dj.scheduled_end_at,
  COUNT(ja.id) FILTER (WHERE ja.status NOT IN ('canceled')) AS active_assignment_count,
  COUNT(ja.id) FILTER (WHERE ja.status = 'completed') AS completed_assignment_count
FROM dispatch_jobs dj
LEFT JOIN job_assignments ja ON ja.dispatch_job_id = dj.id AND ja.org_id = dj.org_id
WHERE dj.org_id = $1
  AND dj.status = ANY($2::text[]) -- e.g., ['pending', 'assigned', 'in_progress']
GROUP BY dj.id, dj.org_id, dj.title, dj.priority, dj.status, dj.created_at, dj.scheduled_start_at, dj.scheduled_end_at
ORDER BY dj.priority DESC, dj.created_at DESC
LIMIT 100;
```

**Performance Requirements**:
- Must execute in < 300ms for 1000 jobs
- Must support pagination

**Required Indexes**:
```sql
-- Status-based queries
CREATE INDEX IF NOT EXISTS idx_dispatch_jobs_org_status_priority 
  ON dispatch_jobs(org_id, status, priority DESC, created_at DESC);

-- Covering index for common fields
CREATE INDEX IF NOT EXISTS idx_dispatch_jobs_covering 
  ON dispatch_jobs(org_id, status)
  INCLUDE (title, priority, created_at, scheduled_start_at, scheduled_end_at);
```

#### 3.2.3 Technician Availability

**Query Pattern**: Check technician availability for a time window

**Base Query**:
```sql
WITH technician_shifts_cte AS (
  SELECT 
    ts.technician_id,
    ts.starts_at,
    ts.ends_at
  FROM technician_shifts ts
  WHERE ts.org_id = $1
    AND ts.technician_id = $2
    AND ts.is_active = true
    AND ts.starts_at < $4::timestamptz
    AND ts.ends_at > $3::timestamptz
),
technician_time_off_cte AS (
  SELECT 
    tto.technician_id,
    tto.starts_at,
    tto.ends_at
  FROM technician_time_off tto
  WHERE tto.org_id = $1
    AND tto.technician_id = $2
    AND tto.starts_at < $4::timestamptz
    AND tto.ends_at > $3::timestamptz
),
technician_assignments_cte AS (
  SELECT 
    ja.technician_id,
    ja.scheduled_start_at,
    ja.scheduled_end_at
  FROM job_assignments ja
  WHERE ja.org_id = $1
    AND ja.technician_id = $2
    AND ja.status NOT IN ('canceled', 'completed')
    AND ja.scheduled_start_at < $4::timestamptz
    AND ja.scheduled_end_at > $3::timestamptz
)
SELECT 
  CASE 
    WHEN EXISTS (SELECT 1 FROM technician_shifts_cte) THEN true
    ELSE false
  END AS has_shift,
  CASE 
    WHEN EXISTS (SELECT 1 FROM technician_time_off_cte) THEN true
    ELSE false
  END AS has_time_off,
  CASE 
    WHEN EXISTS (
      SELECT 1 FROM technician_assignments_cte 
      WHERE scheduled_start_at < $4::timestamptz 
        AND scheduled_end_at > $3::timestamptz
    ) THEN true
    ELSE false
  END AS has_conflict
```

**Performance Requirements**:
- Must execute in < 100ms for single technician check
- Must support batch checks (multiple technicians)

**Required Indexes**:
```sql
-- Technician shifts
CREATE INDEX IF NOT EXISTS idx_technician_shifts_org_tech_active 
  ON technician_shifts(org_id, technician_id, is_active, starts_at, ends_at)
  WHERE is_active = true;

-- Technician time off
CREATE INDEX IF NOT EXISTS idx_technician_time_off_org_tech_dates 
  ON technician_time_off(org_id, technician_id, starts_at, ends_at);

-- Technician assignments (for conflict detection)
CREATE INDEX IF NOT EXISTS idx_job_assignments_org_tech_time 
  ON job_assignments(org_id, technician_id, scheduled_start_at, scheduled_end_at)
  WHERE status NOT IN ('canceled', 'completed');
```

### 3.3 Index Strategy Summary

**Critical Indexes** (must exist):
1. `idx_job_assignments_org_date_status` - Primary schedule board query
2. `idx_job_assignments_org_tech_date` - Technician-specific queries
3. `idx_dispatch_jobs_org_status_priority` - Job filtering
4. `idx_technician_shifts_org_tech_active` - Availability checks
5. `idx_job_assignments_org_tech_time` - Conflict detection

**Optimization Indexes** (performance improvements):
1. `idx_job_assignments_covering` - Reduces table lookups
2. `idx_dispatch_jobs_covering` - Reduces table lookups
3. `idx_technician_time_off_org_tech_dates` - Time-off queries

**Index Maintenance**:
- Monitor index bloat with `pg_stat_user_indexes`
- Reindex quarterly or when bloat exceeds 30%
- Use `REINDEX CONCURRENTLY` for production

### 3.4 Query Performance Validation

**Test Data Setup**:
```sql
-- Generate test data for performance testing
-- 100 technicians, 1000 jobs per day, 2000 assignments per day
-- Run EXPLAIN ANALYZE on all key queries
```

**Performance Test Queries**:
```sql
-- Test 1: Schedule board query (single day)
EXPLAIN ANALYZE
SELECT ... FROM job_assignments ...
WHERE org_id = 'test-org-id'
  AND scheduled_start_at >= '2024-01-15'::date
  AND scheduled_start_at < '2024-01-16'::date;

-- Test 2: Schedule board query (7 days)
EXPLAIN ANALYZE
SELECT ... FROM job_assignments ...
WHERE org_id = 'test-org-id'
  AND scheduled_start_at >= '2024-01-15'::date
  AND scheduled_start_at < '2024-01-22'::date;

-- Test 3: Jobs by status
EXPLAIN ANALYZE
SELECT ... FROM dispatch_jobs ...
WHERE org_id = 'test-org-id'
  AND status = ANY(ARRAY['pending', 'assigned', 'in_progress']);

-- Test 4: Technician availability
EXPLAIN ANALYZE
-- Availability query for single technician
```

**Performance Targets**:
- Schedule board (1 day): < 200ms
- Schedule board (7 days): < 500ms
- Jobs by status: < 300ms
- Technician availability: < 100ms

**Monitoring Queries**:
```sql
-- Check index usage
SELECT 
  schemaname,
  tablename,
  indexname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
  AND tablename IN ('job_assignments', 'dispatch_jobs', 'technician_shifts')
ORDER BY idx_scan DESC;

-- Check slow queries (requires pg_stat_statements extension)
SELECT 
  query,
  calls,
  total_exec_time,
  mean_exec_time,
  max_exec_time
FROM pg_stat_statements
WHERE query LIKE '%job_assignments%'
  OR query LIKE '%dispatch_jobs%'
ORDER BY mean_exec_time DESC
LIMIT 20;
```

### 3.5 Known Bottlenecks and Future Optimizations

**Current Limitations**:
1. **JOIN Overhead**: Multiple JOINs in schedule board query can be slow with large datasets
   - **Mitigation**: Use covering indexes to reduce table lookups
   - **Future**: Consider denormalized view or materialized view

2. **Real-time Subscription Performance**: Supabase Realtime can be slow with many concurrent subscriptions
   - **Mitigation**: Limit subscription scope to visible date range
   - **Future**: Use WebSocket connection pooling

3. **Route Plan Queries**: Loading full route plans with stops can be slow
   - **Mitigation**: Paginate route stops, load on demand
   - **Future**: Cache route plans in Redis

**Future Optimizations**:
1. **Materialized Views**: Pre-compute schedule board data for common date ranges
2. **Read Replicas**: Use read replicas for analytics and reporting queries
3. **Caching Layer**: Redis cache for frequently accessed data (technician availability, job counts)
4. **Query Result Caching**: Cache query results in Next.js with React Query
5. **Pagination**: Implement cursor-based pagination for large result sets

### 3.6 Performance Documentation

**Documentation Requirements**:
- Performance targets documented in code comments
- Index creation scripts version-controlled
- Performance test results stored in `docs/technical/performance-baseline.md`
- Query patterns documented with examples

---

## 4. Story DISP-061: Idempotency Strategy for Optimization and Notification Processing

### 4.1 Idempotency Principles

**Definition**: An operation is idempotent if performing it multiple times produces the same result as performing it once.

**Key Requirements**:
- Same input → Same output (deterministic)
- Duplicate requests → No side effects
- Partial failures → Retry-safe

### 4.2 Idempotency Key Strategy

**Idempotency Key Format**:
```
{operation_type}:{org_id}:{target_id}:{input_hash}
```

**Examples**:
- `auto_schedule:org-123:job-456:hash-789`
- `optimize_route:org-123:tech-789:date-2024-01-15:hash-abc`
- `emergency_insert:org-123:job-456:hash-def`
- `process_notification:org-123:notification-789:hash-ghi`

**Storage**: Use `idempotency_keys` table

**DDL**:
```sql
CREATE TABLE IF NOT EXISTS idempotency_keys (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  key_hash TEXT NOT NULL, -- SHA-256 hash of idempotency key
  operation_type TEXT NOT NULL, -- 'auto_schedule', 'optimize_route', 'emergency_insert', 'process_notification'
  target_id UUID, -- job_id, technician_id, notification_id, etc.
  status TEXT NOT NULL DEFAULT 'processing', -- 'processing', 'completed', 'failed'
  request_payload JSONB, -- Stored for validation
  response_payload JSONB, -- Stored for duplicate request responses
  error_message TEXT,
  expires_at TIMESTAMPTZ NOT NULL DEFAULT (now() + INTERVAL '24 hours'),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at TIMESTAMPTZ,
  UNIQUE(org_id, key_hash)
);

CREATE INDEX idx_idempotency_keys_org_key ON idempotency_keys(org_id, key_hash);
CREATE INDEX idx_idempotency_keys_expires ON idempotency_keys(expires_at) WHERE status = 'completed';

-- Cleanup expired keys (run daily)
CREATE OR REPLACE FUNCTION cleanup_expired_idempotency_keys()
RETURNS void AS $$
BEGIN
  DELETE FROM idempotency_keys
  WHERE expires_at < now()
    AND status IN ('completed', 'failed');
END;
$$ LANGUAGE plpgsql;

-- Schedule cleanup
SELECT cron.schedule(
  'cleanup-idempotency-keys',
  '0 2 * * *', -- Daily at 2 AM
  $$SELECT cleanup_expired_idempotency_keys();$$
);
```

### 4.3 Idempotency Helper Function

**Edge Function Helper**:

```typescript
// File: supabase/functions/_shared/idempotency.ts

interface IdempotencyOptions {
  orgId: string;
  operationType: 'auto_schedule' | 'optimize_route' | 'emergency_insert' | 'process_notification';
  targetId: string;
  requestPayload: any;
  ttlHours?: number; // Default 24 hours
}

interface IdempotencyResult<T> {
  isDuplicate: boolean;
  existingResponse?: T;
  key: string;
}

export async function checkIdempotency<T>(
  supabase: SupabaseClient,
  options: IdempotencyOptions
): Promise<IdempotencyResult<T>> {
  const { orgId, operationType, targetId, requestPayload, ttlHours = 24 } = options;

  // Generate idempotency key
  const inputHash = await crypto.subtle.digest(
    'SHA-256',
    new TextEncoder().encode(JSON.stringify(requestPayload))
  );
  const inputHashHex = Array.from(new Uint8Array(inputHash))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
  
  const key = `${operationType}:${orgId}:${targetId}:${inputHashHex}`;
  const keyHash = await crypto.subtle.digest(
    'SHA-256',
    new TextEncoder().encode(key)
  );
  const keyHashHex = Array.from(new Uint8Array(keyHash))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');

  // Check for existing key
  const { data: existing, error: checkError } = await supabase
    .from('idempotency_keys')
    .select('*')
    .eq('org_id', orgId)
    .eq('key_hash', keyHashHex)
    .single();

  if (checkError && checkError.code !== 'PGRST116') { // PGRST116 = not found
    throw new Error(`Idempotency check failed: ${checkError.message}`);
  }

  if (existing) {
    // Check if expired
    if (new Date(existing.expires_at) < new Date()) {
      // Delete expired key and proceed
      await supabase
        .from('idempotency_keys')
        .delete()
        .eq('id', existing.id);
    } else {
      // Return existing response
      return {
        isDuplicate: true,
        existingResponse: existing.response_payload as T,
        key
      };
    }
  }

  // Create new idempotency key record
  const expiresAt = new Date();
  expiresAt.setHours(expiresAt.getHours() + ttlHours);

  const { error: insertError } = await supabase
    .from('idempotency_keys')
    .insert({
      org_id: orgId,
      key_hash: keyHashHex,
      operation_type: operationType,
      target_id: targetId,
      status: 'processing',
      request_payload: requestPayload,
      expires_at: expiresAt.toISOString()
    });

  if (insertError) {
    // Race condition: another request created the key
    // Check again
    const { data: raceExisting } = await supabase
      .from('idempotency_keys')
      .select('*')
      .eq('org_id', orgId)
      .eq('key_hash', keyHashHex)
      .single();

    if (raceExisting && new Date(raceExisting.expires_at) >= new Date()) {
      return {
        isDuplicate: true,
        existingResponse: raceExisting.response_payload as T,
        key
      };
    }
  }

  return {
    isDuplicate: false,
    key
  };
}

export async function completeIdempotency<T>(
  supabase: SupabaseClient,
  orgId: string,
  key: string,
  response: T,
  error?: Error
): Promise<void> {
  const keyHash = await crypto.subtle.digest(
    'SHA-256',
    new TextEncoder().encode(key)
  );
  const keyHashHex = Array.from(new Uint8Array(keyHash))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');

  const updateData: any = {
    status: error ? 'failed' : 'completed',
    completed_at: new Date().toISOString(),
    response_payload: response
  };

  if (error) {
    updateData.error_message = error.message;
  }

  await supabase
    .from('idempotency_keys')
    .update(updateData)
    .eq('org_id', orgId)
    .eq('key_hash', keyHashHex);
}
```

### 4.4 Auto-Schedule Job Idempotency

**Operation**: `POST /dispatch/jobs/:id/auto_schedule`

**Idempotency Key**: `auto_schedule:{org_id}:{job_id}:{input_hash}`

**Input Hash Includes**:
- Job ID
- Requested time window (if specified)
- Technician preferences (if specified)
- Mode (propose/commit)

**Implementation**:

```typescript
// In auto-schedule Edge Function
import { checkIdempotency, completeIdempotency } from '../_shared/idempotency.ts';

Deno.serve(async (req) => {
  // ... auth and validation ...

  const idempotencyKey = req.headers.get('Idempotency-Key') || 
    `auto_schedule:${auth.orgId}:${jobId}:${Date.now()}`;

  const idempotencyResult = await checkIdempotency(supabase, {
    orgId: auth.orgId,
    operationType: 'auto_schedule',
    targetId: jobId,
    requestPayload: {
      job_id: jobId,
      time_window: requestBody.time_window,
      technician_preferences: requestBody.technician_preferences,
      mode: requestBody.mode
    }
  });

  if (idempotencyResult.isDuplicate) {
    return successResponse(idempotencyResult.existingResponse!);
  }

  try {
    // Perform auto-schedule operation
    const result = await performAutoSchedule(...);

    // Mark idempotency as complete
    await completeIdempotency(supabase, auth.orgId, idempotencyResult.key, result);

    return successResponse(result);
  } catch (error) {
    await completeIdempotency(supabase, auth.orgId, idempotencyResult.key, null, error);
    throw error;
  }
});
```

**Idempotency Behavior**:
- Same job + same inputs → Returns existing assignment
- Same job + different inputs → Creates new assignment (different key)
- Retry after failure → Retries operation (key expired or status = 'failed')

### 4.5 Optimize Routes Idempotency

**Operation**: `POST /dispatch/technicians/:id/optimize_route`

**Idempotency Key**: `optimize_route:{org_id}:{technician_id}:{date}:{input_hash}`

**Input Hash Includes**:
- Technician ID
- Date
- Optimization strategy
- Lock respect flag

**Implementation**:

```typescript
const idempotencyResult = await checkIdempotency(supabase, {
  orgId: auth.orgId,
  operationType: 'optimize_route',
  targetId: technicianId,
  requestPayload: {
    technician_id: technicianId,
    date: requestBody.date,
    strategy: requestBody.strategy,
    respect_locks: requestBody.respect_locks
  }
});

if (idempotencyResult.isDuplicate) {
  return successResponse(idempotencyResult.existingResponse!);
}
```

**Idempotency Behavior**:
- Same technician + same date + same strategy → Returns existing route plan
- Different strategy → Creates new optimization (different key)
- Retry → Returns existing result if within TTL

### 4.6 Emergency Insert Idempotency

**Operation**: `POST /dispatch/jobs/:id/insert_emergency`

**Idempotency Key**: `emergency_insert:{org_id}:{job_id}:{input_hash}`

**Input Hash Includes**:
- Job ID
- Insertion strategy
- Bump rules
- Mode (propose/commit)

**Implementation**:

```typescript
const idempotencyResult = await checkIdempotency(supabase, {
  orgId: auth.orgId,
  operationType: 'emergency_insert',
  targetId: jobId,
  requestPayload: {
    job_id: jobId,
    strategy: requestBody.strategy,
    bump_rules: requestBody.bump_rules,
    mode: requestBody.mode
  }
});

if (idempotencyResult.isDuplicate) {
  return successResponse(idempotencyResult.existingResponse!);
}
```

**Idempotency Behavior**:
- Same job + same strategy → Returns existing insertion plan
- Different strategy → Creates new insertion (different key)
- Retry → Returns existing result if within TTL

### 4.7 Notification Processing Idempotency

**Operation**: `process_job_notifications` (Cron Edge Function)

**Idempotency Key**: `process_notification:{org_id}:{notification_id}:{input_hash}`

**Input Hash Includes**:
- Notification ID
- Processing timestamp (rounded to minute for batch processing)

**Implementation**:

```typescript
// In notification processor cron function
const notifications = await fetchPendingNotifications(...);

for (const notification of notifications) {
  const idempotencyResult = await checkIdempotency(supabase, {
    orgId: notification.org_id,
    operationType: 'process_notification',
    targetId: notification.id,
    requestPayload: {
      notification_id: notification.id,
      processing_minute: Math.floor(Date.now() / 60000) // Round to minute
    },
    ttlHours: 1 // Shorter TTL for notifications
  });

  if (idempotencyResult.isDuplicate) {
    console.log(`Skipping duplicate notification ${notification.id}`);
    continue;
  }

  try {
    await sendNotification(notification);
    await completeIdempotency(supabase, notification.org_id, idempotencyResult.key, {
      sent_at: new Date().toISOString()
    });
  } catch (error) {
    await completeIdempotency(supabase, notification.org_id, idempotencyResult.key, null, error);
    // Retry logic handled by cron
  }
}
```

**Idempotency Behavior**:
- Same notification + same minute → Skips sending (already sent)
- Retry after failure → Retries sending (key expired or status = 'failed')
- Prevents duplicate notifications within same processing window

### 4.8 Retry Behaviors for External Providers

#### 4.8.1 Routing Provider (Mapbox)

**Retry Strategy**:
- Exponential backoff: 1s, 2s, 4s, 8s, 16s
- Max retries: 3
- Retry on: Network errors, 5xx errors, rate limit (429)
- Don't retry on: 4xx errors (except 429), invalid requests

**Implementation**:

```typescript
async function callMapboxWithRetry(
  url: string,
  options: RequestInit,
  maxRetries = 3
): Promise<Response> {
  let lastError: Error | null = null;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, options);

      if (response.ok) {
        return response;
      }

      // Don't retry on 4xx (except 429)
      if (response.status >= 400 && response.status < 500 && response.status !== 429) {
        throw new Error(`Mapbox API error: ${response.status} ${response.statusText}`);
      }

      // Retry on 429 or 5xx
      if (response.status === 429 || response.status >= 500) {
        if (attempt < maxRetries) {
          const delay = Math.pow(2, attempt) * 1000; // Exponential backoff
          await new Promise(resolve => setTimeout(resolve, delay));
          continue;
        }
        throw new Error(`Mapbox API error after ${maxRetries} retries: ${response.status}`);
      }

      return response;
    } catch (error) {
      lastError = error as Error;
      if (attempt < maxRetries) {
        const delay = Math.pow(2, attempt) * 1000;
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
    }
  }

  throw lastError || new Error('Mapbox API call failed after retries');
}
```

#### 4.8.2 Calendar Provider (Google/Microsoft)

**Retry Strategy**:
- Exponential backoff: 2s, 4s, 8s, 16s
- Max retries: 3
- Retry on: Network errors, 5xx errors, rate limit (429)
- Token refresh: Retry once after refreshing token

**Implementation**:

```typescript
async function callCalendarAPIWithRetry(
  accessToken: string,
  url: string,
  options: RequestInit,
  refreshTokenFn?: () => Promise<string>
): Promise<Response> {
  let currentToken = accessToken;
  let lastError: Error | null = null;

  for (let attempt = 0; attempt <= 3; attempt++) {
    try {
      const response = await fetch(url, {
        ...options,
        headers: {
          ...options.headers,
          'Authorization': `Bearer ${currentToken}`
        }
      });

      if (response.ok) {
        return response;
      }

      // Token expired - refresh and retry once
      if (response.status === 401 && refreshTokenFn && attempt === 0) {
        currentToken = await refreshTokenFn();
        continue;
      }

      // Don't retry on 4xx (except 429)
      if (response.status >= 400 && response.status < 500 && response.status !== 429) {
        throw new Error(`Calendar API error: ${response.status}`);
      }

      // Retry on 429 or 5xx
      if (response.status === 429 || response.status >= 500) {
        if (attempt < 3) {
          const delay = Math.pow(2, attempt) * 2000;
          await new Promise(resolve => setTimeout(resolve, delay));
          continue;
        }
        throw new Error(`Calendar API error after retries: ${response.status}`);
      }

      return response;
    } catch (error) {
      lastError = error as Error;
      if (attempt < 3) {
        const delay = Math.pow(2, attempt) * 2000;
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
    }
  }

  throw lastError || new Error('Calendar API call failed after retries');
}
```

#### 4.8.3 Messaging Provider (SMS/Email)

**Retry Strategy**:
- Exponential backoff: 5s, 10s, 20s, 40s
- Max retries: 3
- Retry on: Network errors, 5xx errors, rate limit (429)
- Dead letter queue: Store failed notifications after max retries

**Implementation**:

```typescript
async function sendNotificationWithRetry(
  notification: JobNotification,
  provider: 'sms' | 'email' | 'push',
  maxRetries = 3
): Promise<void> {
  let lastError: Error | null = null;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      await sendViaProvider(notification, provider);
      return; // Success
    } catch (error) {
      lastError = error as Error;
      
      // Don't retry on invalid recipient errors
      if (error.message.includes('invalid') || error.message.includes('not found')) {
        throw error;
      }

      if (attempt < maxRetries) {
        const delay = Math.pow(2, attempt) * 5000;
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
    }
  }

  // Store in dead letter queue after max retries
  await storeFailedNotification(notification, lastError!);
  throw lastError!;
}
```

### 4.9 Idempotency Testing

**Test Scenarios**:

1. **Duplicate Request**: Send same request twice → Second returns cached response
2. **Retry After Failure**: Send request, fail, retry → Retries operation
3. **Different Inputs**: Send request with different inputs → Creates new operation
4. **Expired Key**: Send request after TTL expires → Creates new operation
5. **Race Condition**: Send two requests simultaneously → One succeeds, one returns cached response

**Test Implementation**:

```typescript
// Test: Duplicate request
const request1 = await autoScheduleJob(jobId, { mode: 'commit' });
const request2 = await autoScheduleJob(jobId, { mode: 'commit' }); // Same inputs
assert(request1.assignment_id === request2.assignment_id); // Same result

// Test: Retry after failure
try {
  await optimizeRoute(techId, date, { strategy: 'invalid' });
} catch (error) {
  // Expected failure
}
const retry = await optimizeRoute(techId, date, { strategy: 'invalid' });
// Should retry (not return cached error)

// Test: Different inputs
const result1 = await autoScheduleJob(jobId, { mode: 'propose' });
const result2 = await autoScheduleJob(jobId, { mode: 'commit' });
assert(result1.assignment_id !== result2.assignment_id); // Different results
```

---

## 5. Story DISP-062: Structured Logging and Trace IDs for Dispatch Edge Functions

### 5.1 Logging Principles

**Structured Logging**: All logs are JSON-formatted with consistent fields

**Trace IDs**: Unique identifier for each operation, propagated across all related operations

**PII Redaction**: Sensitive data (customer addresses, phone numbers, tokens) must be redacted

**Log Levels**: `DEBUG`, `INFO`, `WARN`, `ERROR`

### 5.2 Trace ID Generation

**Format**: `{org_id}:{operation_type}:{timestamp}:{random}`

**Example**: `org-123:auto_schedule:20240115T143022Z:abc123`

**Implementation**:

```typescript
// File: supabase/functions/_shared/logging.ts

export function generateTraceId(
  orgId: string,
  operationType: string
): string {
  const timestamp = new Date().toISOString().replace(/[-:]/g, '').split('.')[0] + 'Z';
  const random = crypto.randomUUID().split('-')[0];
  return `${orgId}:${operationType}:${timestamp}:${random}`;
}

export function extractTraceIdFromContext(traceId: string): {
  orgId: string;
  operationType: string;
  timestamp: string;
  random: string;
} {
  const parts = traceId.split(':');
  return {
    orgId: parts[0],
    operationType: parts[1],
    timestamp: parts[2],
    random: parts[3]
  };
}
```

### 5.3 Structured Logging Helper

**Logger Interface**:

```typescript
// File: supabase/functions/_shared/logging.ts

interface LogContext {
  traceId: string;
  orgId: string;
  userId?: string;
  operationType: string;
  [key: string]: any;
}

interface LogEntry {
  level: 'DEBUG' | 'INFO' | 'WARN' | 'ERROR';
  timestamp: string;
  traceId: string;
  orgId: string;
  userId?: string;
  operationType: string;
  message: string;
  data?: any;
  error?: {
    name: string;
    message: string;
    stack?: string;
  };
  correlationIds?: {
    jobId?: string;
    assignmentId?: string;
    technicianId?: string;
    routePlanId?: string;
    notificationId?: string;
    optimizationRunId?: string;
  };
}

class Logger {
  private context: LogContext;

  constructor(context: LogContext) {
    this.context = context;
  }

  private log(level: LogEntry['level'], message: string, data?: any, error?: Error): void {
    const entry: LogEntry = {
      level,
      timestamp: new Date().toISOString(),
      traceId: this.context.traceId,
      orgId: this.context.orgId,
      userId: this.context.userId,
      operationType: this.context.operationType,
      message: this.redactPII(message),
      data: this.redactPII(data),
      correlationIds: this.extractCorrelationIds(data)
    };

    if (error) {
      entry.error = {
        name: error.name,
        message: error.message,
        stack: process.env.NODE_ENV === 'development' ? error.stack : undefined
      };
    }

    // Output as JSON
    console.log(JSON.stringify(entry));
  }

  debug(message: string, data?: any): void {
    this.log('DEBUG', message, data);
  }

  info(message: string, data?: any): void {
    this.log('INFO', message, data);
  }

  warn(message: string, data?: any): void {
    this.log('WARN', message, data);
  }

  error(message: string, error?: Error, data?: any): void {
    this.log('ERROR', message, data, error);
  }

  private redactPII(obj: any): any {
    if (!obj || typeof obj !== 'object') {
      return obj;
    }

    const redacted = { ...obj };
    const piiFields = [
      'phone', 'email', 'address', 'address_line1', 'address_line2',
      'city', 'state', 'postal_code', 'latitude', 'longitude',
      'token', 'access_token', 'refresh_token', 'api_key', 'secret'
    ];

    for (const key in redacted) {
      if (piiFields.some(field => key.toLowerCase().includes(field))) {
        redacted[key] = '[REDACTED]';
      } else if (typeof redacted[key] === 'object' && redacted[key] !== null) {
        redacted[key] = this.redactPII(redacted[key]);
      }
    }

    return redacted;
  }

  private extractCorrelationIds(data: any): LogEntry['correlationIds'] {
    if (!data || typeof data !== 'object') {
      return undefined;
    }

    const ids: LogEntry['correlationIds'] = {};
    
    if (data.job_id || data.dispatch_job_id) {
      ids.jobId = data.job_id || data.dispatch_job_id;
    }
    if (data.assignment_id || data.job_assignment_id) {
      ids.assignmentId = data.assignment_id || data.job_assignment_id;
    }
    if (data.technician_id) {
      ids.technicianId = data.technician_id;
    }
    if (data.route_plan_id) {
      ids.routePlanId = data.route_plan_id;
    }
    if (data.notification_id || data.job_notification_id) {
      ids.notificationId = data.notification_id || data.job_notification_id;
    }
    if (data.run_id || data.optimization_run_id) {
      ids.optimizationRunId = data.run_id || data.optimization_run_id;
    }

    return Object.keys(ids).length > 0 ? ids : undefined;
  }
}

export function createLogger(context: LogContext): Logger {
  return new Logger(context);
}
```

### 5.4 Logging in Edge Functions

#### 5.4.1 Auto-Schedule Function

**Example**:

```typescript
// File: supabase/functions/auto-schedule-job/index.ts

import { createLogger, generateTraceId } from '../_shared/logging.ts';

Deno.serve(async (req) => {
  const traceId = generateTraceId(auth.orgId, 'auto_schedule');
  const logger = createLogger({
    traceId,
    orgId: auth.orgId,
    userId: user.id,
    operationType: 'auto_schedule'
  });

  logger.info('Auto-schedule job request received', {
    job_id: jobId,
    mode: requestBody.mode,
    time_window: requestBody.time_window
  });

  try {
    // Fetch job
    const { data: job, error: jobError } = await supabase
      .from('dispatch_jobs')
      .select('*')
      .eq('id', jobId)
      .single();

    if (jobError) {
      logger.error('Failed to fetch job', jobError, { job_id: jobId });
      throw jobError;
    }

    logger.debug('Job fetched', {
      job_id: jobId,
      priority: job.priority,
      status: job.status
    });

    // Find available technicians
    const technicians = await findAvailableTechnicians(job, logger);
    
    logger.info('Technicians found', {
      job_id: jobId,
      technician_count: technicians.length,
      technician_ids: technicians.map(t => t.id)
    });

    // Score technicians
    const scored = await scoreTechnicians(job, technicians, logger);
    
    logger.info('Technicians scored', {
      job_id: jobId,
      top_candidate: scored[0]?.technician_id,
      top_score: scored[0]?.score,
      candidate_count: scored.length
    });

    // Create assignment
    const assignment = await createAssignment(job, scored[0], logger);
    
    logger.info('Assignment created', {
      job_id: jobId,
      assignment_id: assignment.id,
      technician_id: assignment.technician_id,
      scheduled_start_at: assignment.scheduled_start_at
    });

    // Update job status
    await updateJobStatus(job.id, 'assigned', logger);

    logger.info('Auto-schedule completed successfully', {
      job_id: jobId,
      assignment_id: assignment.id
    });

    return successResponse({
      assignment_id: assignment.id,
      trace_id: traceId
    });
  } catch (error) {
    logger.error('Auto-schedule failed', error as Error, {
      job_id: jobId
    });
    throw error;
  }
});
```

#### 5.4.2 Route Optimization Function

**Example**:

```typescript
// File: supabase/functions/optimize-route/index.ts

const traceId = generateTraceId(auth.orgId, 'optimize_route');
const logger = createLogger({
  traceId,
  orgId: auth.orgId,
  userId: user.id,
  operationType: 'optimize_route'
});

logger.info('Route optimization request received', {
  technician_id: technicianId,
  date: requestBody.date,
  strategy: requestBody.strategy
});

// Fetch assignments
const assignments = await fetchAssignments(technicianId, date, logger);

logger.debug('Assignments fetched', {
  technician_id: technicianId,
  assignment_count: assignments.length,
  assignment_ids: assignments.map(a => a.id)
});

// Calculate cost matrix
const costMatrix = await calculateCostMatrix(assignments, logger);

logger.info('Cost matrix calculated', {
  technician_id: technicianId,
  matrix_size: costMatrix.length,
  total_distance_km: costMatrix.totalDistance
});

// Solve TSP
const solution = await solveTSP(costMatrix, logger);

logger.info('TSP solution found', {
  technician_id: technicianId,
  optimized_sequence: solution.sequence,
  distance_saved_km: solution.distanceSaved,
  time_saved_minutes: solution.timeSaved
});

// Create route plan
const routePlan = await createRoutePlan(technicianId, solution, logger);

logger.info('Route plan created', {
  technician_id: technicianId,
  route_plan_id: routePlan.id,
  stop_count: routePlan.stops.length
});
```

#### 5.4.3 Notification Processor Function

**Example**:

```typescript
// File: supabase/functions/process-job-notifications/index.ts

const traceId = generateTraceId('system', 'process_notifications');
const logger = createLogger({
  traceId,
  orgId: 'system',
  operationType: 'process_notifications'
});

logger.info('Notification processing started', {
  processing_minute: Math.floor(Date.now() / 60000)
});

const notifications = await fetchPendingNotifications();

logger.info('Pending notifications fetched', {
  notification_count: notifications.length
});

for (const notification of notifications) {
  const notificationTraceId = `${traceId}:notification:${notification.id}`;
  const notificationLogger = createLogger({
    traceId: notificationTraceId,
    orgId: notification.org_id,
    operationType: 'process_notification',
    correlationIds: {
      notificationId: notification.id,
      jobId: notification.dispatch_job_id
    }
  });

  notificationLogger.info('Processing notification', {
    notification_id: notification.id,
    type: notification.notification_type,
    recipient_type: notification.recipient_type
  });

  try {
    await sendNotification(notification, notificationLogger);
    
    notificationLogger.info('Notification sent successfully', {
      notification_id: notification.id,
      sent_at: new Date().toISOString()
    });
  } catch (error) {
    notificationLogger.error('Notification send failed', error as Error, {
      notification_id: notification.id
    });
  }
}

logger.info('Notification processing completed', {
  processed_count: notifications.length
});
```

### 5.5 Log Correlation with Persisted Artifacts

**Correlation Strategy**: Include IDs in log entries that map to database records

**Correlation IDs**:
- `jobId` → `dispatch_jobs.id`
- `assignmentId` → `job_assignments.id`
- `technicianId` → `technician_profiles.id`
- `routePlanId` → `route_plans.id`
- `notificationId` → `job_notifications.id`
- `optimizationRunId` → `optimization_runs.id`

**Querying Logs**:

```sql
-- Find all logs for a specific job
SELECT * FROM edge_function_logs
WHERE data->>'jobId' = 'job-123'
ORDER BY timestamp DESC;

-- Find all logs for an optimization run
SELECT * FROM edge_function_logs
WHERE data->>'optimizationRunId' = 'run-456'
ORDER BY timestamp DESC;

-- Find all logs for a trace ID
SELECT * FROM edge_function_logs
WHERE traceId = 'org-123:auto_schedule:20240115T143022Z:abc123'
ORDER BY timestamp DESC;
```

**Note**: Supabase Edge Functions logs are stored in Supabase Dashboard. For production, consider exporting to external logging service (Datadog, LogRocket, etc.).

### 5.6 PII Redaction Rules

**Redacted Fields**:
- Phone numbers: `phone`, `mobile_phone`, `work_phone`
- Email addresses: `email`, `contact_email`
- Addresses: `address`, `address_line1`, `address_line2`, `city`, `state`, `postal_code`
- Coordinates: `latitude`, `longitude` (redact to city-level precision)
- Tokens: `token`, `access_token`, `refresh_token`, `api_key`, `secret`

**Redaction Examples**:

```typescript
// Before redaction
{
  phone: '+1-555-123-4567',
  email: 'customer@example.com',
  address_line1: '123 Main St',
  latitude: 40.7128,
  longitude: -74.0060
}

// After redaction
{
  phone: '[REDACTED]',
  email: '[REDACTED]',
  address_line1: '[REDACTED]',
  latitude: '[REDACTED]',
  longitude: '[REDACTED]'
}
```

**Coordinate Redaction**: Round to 2 decimal places (city-level precision)

```typescript
if (key === 'latitude' || key === 'longitude') {
  redacted[key] = Math.round(obj[key] * 100) / 100; // Round to 2 decimals
}
```

### 5.7 Logging Conventions Documentation

**Documentation Requirements**:
- Logging conventions documented in `docs/technical/logging-conventions.md`
- PII redaction rules documented
- Trace ID format documented
- Correlation ID mapping documented
- Examples for each operation type

---

## 6. Observability UI Components

### 6.1 Performance Monitoring Dashboard

**File**: `app/dispatch/observability/performance/page.tsx`

**Purpose**: Display query performance metrics and slow query analysis

**Components**:
- Query performance cards (average response time, p95, p99)
- Slow query table with EXPLAIN plans
- Index usage statistics
- Performance trends chart

**Implementation**:

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';
import { createSupabaseClient } from '@/lib/supabase/client';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table';
import { Badge } from '@/components/ui/badge';
import { Skeleton } from '@/components/ui/skeleton';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { format } from 'date-fns';
import { Activity, Database, TrendingUp } from 'lucide-react';

export default function PerformanceMonitoringPage() {
  const supabase = createSupabaseClient();

  const { data: performanceData, isLoading } = useQuery({
    queryKey: ['performance-metrics'],
    queryFn: async () => {
      const { data: { session } } = await supabase.auth.getSession();
      
      const response = await fetch('/api/dispatch/observability/performance', {
        headers: {
          'Authorization': `Bearer ${session?.access_token}`
        }
      });

      if (!response.ok) {
        throw new Error('Failed to load performance data');
      }

      return response.json();
    },
    refetchInterval: 30000 // Refresh every 30 seconds
  });

  if (isLoading) {
    return (
      <div className="container mx-auto py-8 space-y-4">
        <Skeleton className="h-12 w-64" />
        <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
          {[1, 2, 3].map((i) => (
            <Skeleton key={i} className="h-32" />
          ))}
        </div>
      </div>
    );
  }

  return (
    <div className="container mx-auto py-8 space-y-6">
      <div>
        <h1 className="text-2xl font-bold mb-2">Performance Monitoring</h1>
        <p className="text-muted-foreground">
          Query performance metrics and slow query analysis
        </p>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
        <Card>
          <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
            <CardTitle className="text-sm font-medium">Avg Response Time</CardTitle>
            <Activity className="h-4 w-4 text-muted-foreground" />
          </CardHeader>
          <CardContent>
            <div className="text-2xl font-bold">
              {performanceData?.data?.avg_response_time_ms?.toFixed(0) || 0}ms
            </div>
            <p className="text-xs text-muted-foreground mt-1">
              Target: &lt; 500ms
            </p>
          </CardContent>
        </Card>

        <Card>
          <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
            <CardTitle className="text-sm font-medium">P95 Response Time</CardTitle>
            <TrendingUp className="h-4 w-4 text-muted-foreground" />
          </CardHeader>
          <CardContent>
            <div className="text-2xl font-bold">
              {performanceData?.data?.p95_response_time_ms?.toFixed(0) || 0}ms
            </div>
            <p className="text-xs text-muted-foreground mt-1">
              95th percentile
            </p>
          </CardContent>
        </Card>

        <Card>
          <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
            <CardTitle className="text-sm font-medium">Slow Queries</CardTitle>
            <Database className="h-4 w-4 text-muted-foreground" />
          </CardHeader>
          <CardContent>
            <div className="text-2xl font-bold">
              {performanceData?.data?.slow_query_count || 0}
            </div>
            <p className="text-xs text-muted-foreground mt-1">
              Queries &gt; 500ms
            </p>
          </CardContent>
        </Card>
      </div>

      <Tabs defaultValue="slow-queries">
        <TabsList>
          <TabsTrigger value="slow-queries">Slow Queries</TabsTrigger>
          <TabsTrigger value="index-usage">Index Usage</TabsTrigger>
          <TabsTrigger value="query-trends">Query Trends</TabsTrigger>
        </TabsList>

        <TabsContent value="slow-queries">
          <Card>
            <CardHeader>
              <CardTitle>Slow Queries</CardTitle>
              <CardDescription>
                Queries exceeding 500ms response time
              </CardDescription>
            </CardHeader>
            <CardContent>
              <Table>
                <TableHeader>
                  <TableRow>
                    <TableHead>Query</TableHead>
                    <TableHead>Avg Time</TableHead>
                    <TableHead>Calls</TableHead>
                    <TableHead>Status</TableHead>
                  </TableRow>
                </TableHeader>
                <TableBody>
                  {(performanceData?.data?.slow_queries || []).map((query: any) => (
                    <TableRow key={query.query}>
                      <TableCell className="font-mono text-xs">
                        {query.query.substring(0, 100)}...
                      </TableCell>
                      <TableCell>
                        <Badge variant={query.mean_exec_time > 1000 ? 'destructive' : 'warning'}>
                          {query.mean_exec_time.toFixed(0)}ms
                        </Badge>
                      </TableCell>
                      <TableCell>{query.calls}</TableCell>
                      <TableCell>
                        <Badge variant="outline">Needs Optimization</Badge>
                      </TableCell>
                    </TableRow>
                  ))}
                </TableBody>
              </Table>
            </CardContent>
          </Card>
        </TabsContent>

        <TabsContent value="index-usage">
          <Card>
            <CardHeader>
              <CardTitle>Index Usage Statistics</CardTitle>
            </CardHeader>
            <CardContent>
              <Table>
                <TableHeader>
                  <TableRow>
                    <TableHead>Index</TableHead>
                    <TableHead>Scans</TableHead>
                    <TableHead>Tuples Read</TableHead>
                    <TableHead>Usage</TableHead>
                  </TableRow>
                </TableHeader>
                <TableBody>
                  {(performanceData?.data?.index_usage || []).map((index: any) => (
                    <TableRow key={index.indexname}>
                      <TableCell className="font-mono text-xs">{index.indexname}</TableCell>
                      <TableCell>{index.idx_scan}</TableCell>
                      <TableCell>{index.idx_tup_read}</TableCell>
                      <TableCell>
                        <Badge variant={index.idx_scan > 1000 ? 'default' : 'secondary'}>
                          {index.idx_scan > 1000 ? 'Active' : 'Low Usage'}
                        </Badge>
                      </TableCell>
                    </TableRow>
                  ))}
                </TableBody>
              </Table>
            </CardContent>
          </Card>
        </TabsContent>
      </Tabs>
    </div>
  );
}
```

### 6.2 Log Viewer Component

**File**: `app/dispatch/observability/logs/page.tsx`

**Purpose**: View and search structured logs from Edge Functions

**Components**:
- Log search filters (trace ID, operation type, date range)
- Log table with JSON viewer
- Log detail drawer
- Export functionality

**Implementation**:

```typescript
'use client';

import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { createSupabaseClient } from '@/lib/supabase/client';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Input } from '@/components/ui/input';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table';
import {
  Sheet,
  SheetContent,
  SheetDescription,
  SheetHeader,
  SheetTitle,
} from '@/components/ui/sheet';
import { format, parseISO } from 'date-fns';
import { Search, Filter, Download } from 'lucide-react';

export default function LogViewerPage() {
  const [filters, setFilters] = useState({
    trace_id: '',
    operation_type: 'all',
    level: 'all',
    search: ''
  });
  const [selectedLog, setSelectedLog] = useState<any>(null);

  const supabase = createSupabaseClient();

  const { data: logs, isLoading } = useQuery({
    queryKey: ['edge-function-logs', filters],
    queryFn: async () => {
      const { data: { session } } = await supabase.auth.getSession();
      
      const params = new URLSearchParams();
      if (filters.trace_id) params.append('trace_id', filters.trace_id);
      if (filters.operation_type !== 'all') params.append('operation_type', filters.operation_type);
      if (filters.level !== 'all') params.append('level', filters.level);
      if (filters.search) params.append('search', filters.search);

      const response = await fetch(`/api/dispatch/observability/logs?${params.toString()}`, {
        headers: {
          'Authorization': `Bearer ${session?.access_token}`
        }
      });

      if (!response.ok) {
        throw new Error('Failed to load logs');
      }

      return response.json();
    }
  });

  return (
    <div className="container mx-auto py-8 space-y-6">
      <div className="flex items-center justify-between">
        <div>
          <h1 className="text-2xl font-bold mb-2">Log Viewer</h1>
          <p className="text-muted-foreground">
            View and search structured logs from Edge Functions
          </p>
        </div>
        <Button variant="outline">
          <Download className="mr-2 h-4 w-4" />
          Export Logs
        </Button>
      </div>

      <Card>
        <CardHeader>
          <CardTitle>Filters</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
            <Input
              placeholder="Trace ID"
              value={filters.trace_id}
              onChange={(e) => setFilters({ ...filters, trace_id: e.target.value })}
            />
            <Select
              value={filters.operation_type}
              onValueChange={(value) => setFilters({ ...filters, operation_type: value })}
            >
              <SelectTrigger>
                <SelectValue placeholder="Operation Type" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="all">All Operations</SelectItem>
                <SelectItem value="auto_schedule">Auto-Schedule</SelectItem>
                <SelectItem value="optimize_route">Optimize Route</SelectItem>
                <SelectItem value="emergency_insert">Emergency Insert</SelectItem>
                <SelectItem value="process_notification">Process Notification</SelectItem>
              </SelectContent>
            </Select>
            <Select
              value={filters.level}
              onValueChange={(value) => setFilters({ ...filters, level: value })}
            >
              <SelectTrigger>
                <SelectValue placeholder="Log Level" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="all">All Levels</SelectItem>
                <SelectItem value="DEBUG">DEBUG</SelectItem>
                <SelectItem value="INFO">INFO</SelectItem>
                <SelectItem value="WARN">WARN</SelectItem>
                <SelectItem value="ERROR">ERROR</SelectItem>
              </SelectContent>
            </Select>
            <Input
              placeholder="Search..."
              value={filters.search}
              onChange={(e) => setFilters({ ...filters, search: e.target.value })}
            />
          </div>
        </CardContent>
      </Card>

      <Card>
        <CardHeader>
          <CardTitle>Log Entries</CardTitle>
          <CardDescription>
            {logs?.data?.pagination?.total || 0} total entries
          </CardDescription>
        </CardHeader>
        <CardContent>
          <Table>
            <TableHeader>
              <TableRow>
                <TableHead>Timestamp</TableHead>
                <TableHead>Level</TableHead>
                <TableHead>Operation</TableHead>
                <TableHead>Trace ID</TableHead>
                <TableHead>Message</TableHead>
                <TableHead>Actions</TableHead>
              </TableRow>
            </TableHeader>
            <TableBody>
              {(logs?.data?.logs || []).map((log: any) => (
                <TableRow key={log.id}>
                  <TableCell className="text-sm">
                    {format(parseISO(log.timestamp), 'MMM d, yyyy h:mm:ss.SSS a')}
                  </TableCell>
                  <TableCell>
                    <Badge
                      variant={
                        log.level === 'ERROR' ? 'destructive' :
                        log.level === 'WARN' ? 'warning' :
                        'secondary'
                      }
                    >
                      {log.level}
                    </Badge>
                  </TableCell>
                  <TableCell className="text-sm">{log.operationType}</TableCell>
                  <TableCell className="font-mono text-xs">{log.traceId}</TableCell>
                  <TableCell className="text-sm">{log.message}</TableCell>
                  <TableCell>
                    <Button
                      variant="ghost"
                      size="sm"
                      onClick={() => setSelectedLog(log)}
                    >
                      View Details
                    </Button>
                  </TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </CardContent>
      </Card>

      <Sheet open={!!selectedLog} onOpenChange={(open) => !open && setSelectedLog(null)}>
        <SheetContent className="w-full sm:max-w-2xl overflow-y-auto">
          <SheetHeader>
            <SheetTitle>Log Entry Details</SheetTitle>
            <SheetDescription>
              Trace ID: {selectedLog?.traceId}
            </SheetDescription>
          </SheetHeader>
          <div className="mt-4 space-y-4">
            <div>
              <h3 className="font-semibold mb-2">Message</h3>
              <p className="text-sm">{selectedLog?.message}</p>
            </div>
            {selectedLog?.data && (
              <div>
                <h3 className="font-semibold mb-2">Data</h3>
                <pre className="text-xs bg-muted p-4 rounded overflow-auto">
                  {JSON.stringify(selectedLog.data, null, 2)}
                </pre>
              </div>
            )}
            {selectedLog?.error && (
              <div>
                <h3 className="font-semibold mb-2">Error</h3>
                <pre className="text-xs bg-destructive/10 p-4 rounded overflow-auto">
                  {JSON.stringify(selectedLog.error, null, 2)}
                </pre>
              </div>
            )}
            {selectedLog?.correlationIds && (
              <div>
                <h3 className="font-semibold mb-2">Correlation IDs</h3>
                <div className="space-y-1">
                  {Object.entries(selectedLog.correlationIds).map(([key, value]) => (
                    <div key={key} className="text-sm">
                      <span className="font-medium">{key}:</span> {value as string}
                    </div>
                  ))}
                </div>
              </div>
            )}
          </div>
        </SheetContent>
      </Sheet>
    </div>
  );
}
```

### 6.3 Idempotency Key Viewer

**File**: `app/dispatch/observability/idempotency/page.tsx`

**Purpose**: View idempotency key usage and duplicate request detection

**Components**:
- Idempotency key table
- Duplicate request statistics
- Key expiration monitoring

**Implementation**:

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';
import { createSupabaseClient } from '@/lib/supabase/client';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table';
import { Badge } from '@/components/ui/badge';
import { format, parseISO } from 'date-fns';
import { CheckCircle2, XCircle, Clock } from 'lucide-react';

export default function IdempotencyViewerPage() {
  const supabase = createSupabaseClient();

  const { data: idempotencyData, isLoading } = useQuery({
    queryKey: ['idempotency-keys'],
    queryFn: async () => {
      const { data: { session } } = await supabase.auth.getSession();
      
      const response = await fetch('/api/dispatch/observability/idempotency', {
        headers: {
          'Authorization': `Bearer ${session?.access_token}`
        }
      });

      if (!response.ok) {
        throw new Error('Failed to load idempotency data');
      }

      return response.json();
    },
    refetchInterval: 10000 // Refresh every 10 seconds
  });

  return (
    <div className="container mx-auto py-8 space-y-6">
      <div>
        <h1 className="text-2xl font-bold mb-2">Idempotency Keys</h1>
        <p className="text-muted-foreground">
          Monitor idempotency key usage and duplicate request detection
        </p>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
        <Card>
          <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
            <CardTitle className="text-sm font-medium">Total Keys</CardTitle>
            <CheckCircle2 className="h-4 w-4 text-muted-foreground" />
          </CardHeader>
          <CardContent>
            <div className="text-2xl font-bold">
              {idempotencyData?.data?.total_keys || 0}
            </div>
          </CardContent>
        </Card>

        <Card>
          <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
            <CardTitle className="text-sm font-medium">Duplicate Requests</CardTitle>
            <XCircle className="h-4 w-4 text-muted-foreground" />
          </CardHeader>
          <CardContent>
            <div className="text-2xl font-bold">
              {idempotencyData?.data?.duplicate_count || 0}
            </div>
            <p className="text-xs text-muted-foreground mt-1">
              Prevented duplicate operations
            </p>
          </CardContent>
        </Card>

        <Card>
          <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
            <CardTitle className="text-sm font-medium">Processing</CardTitle>
            <Clock className="h-4 w-4 text-muted-foreground" />
          </CardHeader>
          <CardContent>
            <div className="text-2xl font-bold">
              {idempotencyData?.data?.processing_count || 0}
            </div>
            <p className="text-xs text-muted-foreground mt-1">
              Currently processing
            </p>
          </CardContent>
        </Card>
      </div>

      <Card>
        <CardHeader>
          <CardTitle>Recent Idempotency Keys</CardTitle>
        </CardHeader>
        <CardContent>
          <Table>
            <TableHeader>
              <TableRow>
                <TableHead>Operation Type</TableHead>
                <TableHead>Target ID</TableHead>
                <TableHead>Status</TableHead>
                <TableHead>Created At</TableHead>
                <TableHead>Completed At</TableHead>
              </TableRow>
            </TableHeader>
            <TableBody>
              {(idempotencyData?.data?.keys || []).map((key: any) => (
                <TableRow key={key.id}>
                  <TableCell>
                    <Badge variant="outline">{key.operation_type}</Badge>
                  </TableCell>
                  <TableCell className="font-mono text-xs">{key.target_id}</TableCell>
                  <TableCell>
                    <Badge
                      variant={
                        key.status === 'completed' ? 'default' :
                        key.status === 'failed' ? 'destructive' :
                        'secondary'
                      }
                    >
                      {key.status}
                    </Badge>
                  </TableCell>
                  <TableCell className="text-sm">
                    {format(parseISO(key.created_at), 'MMM d, yyyy h:mm a')}
                  </TableCell>
                  <TableCell className="text-sm">
                    {key.completed_at
                      ? format(parseISO(key.completed_at), 'MMM d, yyyy h:mm a')
                      : '-'}
                  </TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </CardContent>
      </Card>
    </div>
  );
}
```

### 6.4 API Routes

**File**: `app/api/dispatch/observability/performance/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';

export async function GET(request: NextRequest) {
  const supabase = createRouteHandlerClient({ cookies });
  
  const {
    data: { session },
  } = await supabase.auth.getSession();

  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Call Edge Function to get performance metrics
  const { data, error } = await supabase.functions.invoke('get-performance-metrics', {
    headers: {
      Authorization: `Bearer ${session.access_token}`,
    },
  });

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json({ data });
}
```

**File**: `app/api/dispatch/observability/logs/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';

export async function GET(request: NextRequest) {
  const supabase = createRouteHandlerClient({ cookies });
  
  const {
    data: { session },
  } = await supabase.auth.getSession();

  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { searchParams } = new URL(request.url);
  const traceId = searchParams.get('trace_id');
  const operationType = searchParams.get('operation_type');
  const level = searchParams.get('level');
  const search = searchParams.get('search');

  // Call Edge Function to get logs
  const { data, error } = await supabase.functions.invoke('get-edge-function-logs', {
    body: {
      trace_id: traceId || undefined,
      operation_type: operationType || undefined,
      level: level || undefined,
      search: search || undefined
    },
    headers: {
      Authorization: `Bearer ${session.access_token}`,
    },
  });

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json({ data });
}
```

---

## 7. Implementation Checklist

### Story DISP-060: Performance Baseline

- [ ] **Query Patterns Documented**:
  - [ ] Assignments by day query documented
  - [ ] Jobs by status query documented
  - [ ] Technician availability query documented
  - [ ] Performance targets defined (< 500ms)

- [ ] **Indexes Created**:
  - [ ] `idx_job_assignments_org_date_status` created
  - [ ] `idx_job_assignments_org_tech_date` created
  - [ ] `idx_job_assignments_covering` created
  - [ ] `idx_dispatch_jobs_org_status_priority` created
  - [ ] `idx_dispatch_jobs_covering` created
  - [ ] `idx_technician_shifts_org_tech_active` created
  - [ ] `idx_technician_time_off_org_tech_dates` created
  - [ ] `idx_job_assignments_org_tech_time` created

- [ ] **Performance Testing**:
  - [ ] Test data generated (100 technicians, 1000 jobs/day)
  - [ ] EXPLAIN ANALYZE run on all key queries
  - [ ] Performance targets validated
  - [ ] Slow queries identified and optimized

- [ ] **Documentation**:
  - [ ] Performance notes captured in `docs/technical/performance-baseline.md`
  - [ ] Known bottlenecks documented
  - [ ] Future optimizations documented

### Story DISP-061: Idempotency Strategy

- [ ] **Idempotency Table**:
  - [ ] `idempotency_keys` table created
  - [ ] Indexes created
  - [ ] Cleanup function created
  - [ ] Cron job scheduled

- [ ] **Idempotency Helpers**:
  - [ ] `checkIdempotency()` function implemented
  - [ ] `completeIdempotency()` function implemented
  - [ ] Key generation logic implemented

- [ ] **Operation Idempotency**:
  - [ ] Auto-schedule idempotency implemented
  - [ ] Optimize route idempotency implemented
  - [ ] Emergency insert idempotency implemented
  - [ ] Notification processing idempotency implemented

- [ ] **Retry Behaviors**:
  - [ ] Mapbox retry logic implemented
  - [ ] Calendar API retry logic implemented
  - [ ] Messaging provider retry logic implemented
  - [ ] Dead letter queue for failed notifications

- [ ] **Testing**:
  - [ ] Duplicate request test implemented
  - [ ] Retry after failure test implemented
  - [ ] Different inputs test implemented
  - [ ] Race condition test implemented

- [ ] **Documentation**:
  - [ ] Idempotency strategy documented
  - [ ] Retry behaviors documented
  - [ ] Test scenarios documented

### Story DISP-062: Structured Logging

- [ ] **Logging Infrastructure**:
  - [ ] Trace ID generation implemented
  - [ ] Logger class implemented
  - [ ] PII redaction implemented
  - [ ] Correlation ID extraction implemented

- [ ] **Edge Function Logging**:
  - [ ] Auto-schedule function logging implemented
  - [ ] Optimize route function logging implemented
  - [ ] Emergency insert function logging implemented
  - [ ] Notification processor logging implemented

- [ ] **Log Correlation**:
  - [ ] Correlation IDs included in all logs
  - [ ] Log-to-database mapping documented
  - [ ] Query examples documented

- [ ] **PII Redaction**:
  - [ ] Phone number redaction implemented
  - [ ] Email redaction implemented
  - [ ] Address redaction implemented
  - [ ] Token redaction implemented
  - [ ] Coordinate redaction implemented

- [ ] **Documentation**:
  - [ ] Logging conventions documented in `docs/technical/logging-conventions.md`
  - [ ] PII redaction rules documented
  - [ ] Trace ID format documented
  - [ ] Examples for each operation type

- [ ] **Observability UI**:
  - [ ] Performance monitoring dashboard created
  - [ ] Log viewer component created
  - [ ] Idempotency key viewer created
  - [ ] API routes implemented

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 14 – Reliability, Performance, and Observability. All performance optimizations target sub-500ms response times, idempotency ensures reliability during retries, and structured logging enables production debugging.

**Next Steps**: After completing Epic 14, proceed to Epic 15 (Documentation & API Reference) and Epic 16 (Integration Hooks) to complete the Scheduling & Dispatch module.

