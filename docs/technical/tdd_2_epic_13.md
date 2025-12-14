# Technical Design Document – Epic 13: Analytics & KPI Foundations (Scheduling Data Outputs)

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 13 – Analytics & KPI Foundations (Scheduling Data Outputs)
- **Source**: Derived from `fdd_2_agile.md` Epic 13 (Stories DISP-058 through DISP-059)
- **Reference Documents**: 
  - `fdd_2.md` (Functional Design Document for Scheduling & Dispatch)
  - `fdd_2_agile.md` (Agile User Stories)
  - `tdd_2_epic_1.md` (Dispatch Epic 1 for tenancy/role conventions)
  - `tdd_2_epic_2.md` (Dispatch Epic 2 for table schemas)
  - `tdd_2_epic_3.md` (Dispatch Epic 3 for RLS policies)
  - `tdd_2_epic_5.md` (Dispatch Epic 5 for job lifecycle APIs)
  - `tdd_2_epic_7.md` (Dispatch Epic 7 for auto-scheduling and route optimization)
  - `tdd_2_epic_8.md` (Dispatch Epic 8 for emergency job handling)
  - `tdd_2_epic_11.md` (Dispatch Epic 11 for Next.js UI patterns)
- **Target Platform**: PostgreSQL 15+, Supabase, Next.js 14+ (App Router), shadcn/ui
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing Analytics & KPI Foundations for scheduling data. It covers:

- KPI view specifications with formulas and source fields
- Database views and materialized views for KPI calculations
- Audit trail table and triggers for schedule changes
- Next.js UI components for viewing KPIs and audit logs
- API endpoints for KPI data

All KPIs are calculated from dispatch tables using PostgreSQL views and materialized views, with Next.js components providing visualization using shadcn/ui components.

This epic assumes Epic 1-12 are complete and all dispatch tables are populated with historical data.

---

## 2. Prerequisites

### 2.1 Required Components

Before implementing Epic 13, ensure:

1. **Epic 1-12 Complete**: All previous epics are implemented
2. **Required Tables**: All dispatch tables exist and are populated:
   - `dispatch_jobs`
   - `job_assignments`
   - `route_plans`
   - `route_stops`
   - `technician_profiles`
   - `technician_shifts`
   - `technician_time_off`

3. **Data Quality**: Historical data exists for KPI calculations:
   - Completed jobs with actual times
   - Assignment status transitions
   - Route plan data with travel times

4. **Next.js Project Setup**:
   - Next.js 14+ with App Router
   - React 18+
   - TypeScript
   - Tailwind CSS
   - shadcn/ui installed and configured

---

## 3. Story DISP-058: Define Scheduling KPI View Specifications

### 3.1 KPI Overview

**KPIs to Implement**:

1. **On-Time Arrival Rate**: Percentage of jobs where technician arrived within scheduled window
2. **Utilization Percentage**: Percentage of available time technicians are scheduled
3. **Average Travel Time**: Average travel time per job or per technician
4. **Emergency Response Time**: Time from emergency job creation to assignment/arrival

### 3.2 KPI 1: On-Time Arrival Rate

**Formula**:
```
On-Time Arrival Rate = (Jobs Arrived On-Time / Total Completed Jobs) × 100

Where:
- Jobs Arrived On-Time = Jobs where actual_arrival_at is within arrival_window_start and arrival_window_end
- Total Completed Jobs = Jobs with status = 'completed' and actual_arrival_at is not null
```

**Source Fields**:
- `job_assignments.arrival_window_start`
- `job_assignments.arrival_window_end`
- `job_assignments.actual_arrival_at` (from `route_stops.actual_arrival_at` or assignment metadata)
- `job_assignments.status`
- `dispatch_jobs.status`

**Data Quality Requirements**:
- `actual_arrival_at` must be populated for completed assignments
- `arrival_window_start` and `arrival_window_end` must be set
- Assignment must have `status = 'completed'` or job must have `status = 'completed'`

**View Definition**:

```sql
-- Materialized view for on-time arrival rate
CREATE MATERIALIZED VIEW IF NOT EXISTS kpi_on_time_arrival_rate AS
SELECT 
  org_id,
  DATE_TRUNC('day', scheduled_start_at) AS date,
  COUNT(*) FILTER (WHERE 
    actual_arrival_at IS NOT NULL 
    AND arrival_window_start IS NOT NULL 
    AND arrival_window_end IS NOT NULL
    AND actual_arrival_at >= arrival_window_start 
    AND actual_arrival_at <= arrival_window_end
  ) AS on_time_count,
  COUNT(*) FILTER (WHERE 
    actual_arrival_at IS NOT NULL 
    AND status IN ('completed')
  ) AS total_completed_count,
  CASE 
    WHEN COUNT(*) FILTER (WHERE actual_arrival_at IS NOT NULL AND status IN ('completed')) > 0
    THEN ROUND(
      (COUNT(*) FILTER (WHERE 
        actual_arrival_at IS NOT NULL 
        AND arrival_window_start IS NOT NULL 
        AND arrival_window_end IS NOT NULL
        AND actual_arrival_at >= arrival_window_start 
        AND actual_arrival_at <= arrival_window_end
      )::NUMERIC / 
      COUNT(*) FILTER (WHERE actual_arrival_at IS NOT NULL AND status IN ('completed'))::NUMERIC) * 100,
      2
    )
    ELSE NULL
  END AS on_time_rate_percent,
  last_calculated_at TIMESTAMPTZ DEFAULT now()
FROM job_assignments
WHERE status IN ('completed')
GROUP BY org_id, DATE_TRUNC('day', scheduled_start_at);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_kpi_on_time_arrival_rate_org_date 
  ON kpi_on_time_arrival_rate(org_id, date DESC);

-- Refresh function
CREATE OR REPLACE FUNCTION refresh_kpi_on_time_arrival_rate()
RETURNS void AS $$
BEGIN
  REFRESH MATERIALIZED VIEW CONCURRENTLY kpi_on_time_arrival_rate;
END;
$$ LANGUAGE plpgsql;
```

**Per-Technician View**:

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS kpi_on_time_arrival_rate_by_technician AS
SELECT 
  org_id,
  technician_id,
  DATE_TRUNC('day', scheduled_start_at) AS date,
  COUNT(*) FILTER (WHERE 
    actual_arrival_at IS NOT NULL 
    AND arrival_window_start IS NOT NULL 
    AND arrival_window_end IS NOT NULL
    AND actual_arrival_at >= arrival_window_start 
    AND actual_arrival_at <= arrival_window_end
  ) AS on_time_count,
  COUNT(*) FILTER (WHERE 
    actual_arrival_at IS NOT NULL 
    AND status IN ('completed')
  ) AS total_completed_count,
  CASE 
    WHEN COUNT(*) FILTER (WHERE actual_arrival_at IS NOT NULL AND status IN ('completed')) > 0
    THEN ROUND(
      (COUNT(*) FILTER (WHERE 
        actual_arrival_at IS NOT NULL 
        AND arrival_window_start IS NOT NULL 
        AND arrival_window_end IS NOT NULL
        AND actual_arrival_at >= arrival_window_start 
        AND actual_arrival_at <= arrival_window_end
      )::NUMERIC / 
      COUNT(*) FILTER (WHERE actual_arrival_at IS NOT NULL AND status IN ('completed'))::NUMERIC) * 100,
      2
    )
    ELSE NULL
  END AS on_time_rate_percent
FROM job_assignments
WHERE status IN ('completed')
GROUP BY org_id, technician_id, DATE_TRUNC('day', scheduled_start_at);

CREATE INDEX IF NOT EXISTS idx_kpi_on_time_arrival_rate_by_tech_org_date 
  ON kpi_on_time_arrival_rate_by_technician(org_id, technician_id, date DESC);
```

### 3.3 KPI 2: Utilization Percentage

**Formula**:
```
Utilization Percentage = (Scheduled Minutes / Available Minutes) × 100

Where:
- Scheduled Minutes = Sum of (scheduled_end_at - scheduled_start_at) for assignments in date range
- Available Minutes = Sum of (shift ends_at - shift starts_at) minus time-off minutes
```

**Source Fields**:
- `job_assignments.scheduled_start_at`
- `job_assignments.scheduled_end_at`
- `technician_shifts.starts_at`
- `technician_shifts.ends_at`
- `technician_time_off.starts_at`
- `technician_time_off.ends_at`
- `technician_profiles.max_daily_work_minutes`

**Data Quality Requirements**:
- Shifts must be defined for technicians
- Assignment times must be within shift times
- Time-off must be properly recorded

**View Definition**:

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS kpi_technician_utilization AS
WITH scheduled_minutes AS (
  SELECT 
    org_id,
    technician_id,
    DATE_TRUNC('day', scheduled_start_at) AS date,
    SUM(EXTRACT(EPOCH FROM (scheduled_end_at - scheduled_start_at)) / 60)::INTEGER AS scheduled_minutes
  FROM job_assignments
  WHERE status IN ('assigned', 'accepted', 'en_route', 'on_site', 'completed')
  GROUP BY org_id, technician_id, DATE_TRUNC('day', scheduled_start_at)
),
available_minutes AS (
  SELECT 
    ts.org_id,
    ts.technician_id,
    DATE_TRUNC('day', ts.starts_at) AS date,
    SUM(EXTRACT(EPOCH FROM (ts.ends_at - ts.starts_at)) / 60)::INTEGER AS shift_minutes,
    COALESCE(SUM(EXTRACT(EPOCH FROM (
      LEAST(tt.ends_at, ts.ends_at) - GREATEST(tt.starts_at, ts.starts_at)
    )) / 60)::INTEGER, 0) AS time_off_minutes
  FROM technician_shifts ts
  LEFT JOIN technician_time_off tt ON 
    tt.technician_id = ts.technician_id
    AND tt.org_id = ts.org_id
    AND ts.starts_at < tt.ends_at
    AND ts.ends_at > tt.starts_at
  WHERE ts.is_active = true
  GROUP BY ts.org_id, ts.technician_id, DATE_TRUNC('day', ts.starts_at)
)
SELECT 
  COALESCE(sm.org_id, am.org_id) AS org_id,
  COALESCE(sm.technician_id, am.technician_id) AS technician_id,
  COALESCE(sm.date, am.date) AS date,
  COALESCE(sm.scheduled_minutes, 0) AS scheduled_minutes,
  COALESCE(am.shift_minutes, 0) - COALESCE(am.time_off_minutes, 0) AS available_minutes,
  CASE 
    WHEN (COALESCE(am.shift_minutes, 0) - COALESCE(am.time_off_minutes, 0)) > 0
    THEN ROUND(
      (COALESCE(sm.scheduled_minutes, 0)::NUMERIC / 
       (COALESCE(am.shift_minutes, 0) - COALESCE(am.time_off_minutes, 0))::NUMERIC) * 100,
      2
    )
    ELSE NULL
  END AS utilization_percent,
  tp.max_daily_work_minutes AS capacity_minutes
FROM scheduled_minutes sm
FULL OUTER JOIN available_minutes am ON 
  sm.org_id = am.org_id 
  AND sm.technician_id = am.technician_id 
  AND sm.date = am.date
LEFT JOIN technician_profiles tp ON 
  tp.id = COALESCE(sm.technician_id, am.technician_id)
  AND tp.org_id = COALESCE(sm.org_id, am.org_id);

CREATE INDEX IF NOT EXISTS idx_kpi_technician_utilization_org_tech_date 
  ON kpi_technician_utilization(org_id, technician_id, date DESC);
```

### 3.4 KPI 3: Average Travel Time

**Formula**:
```
Average Travel Time = Sum of Travel Times / Number of Jobs

Where:
- Travel Time = route_stops.travel_time_minutes_from_prev (for jobs in routes)
- Or calculated from assignment sequence (for jobs not in routes)
```

**Source Fields**:
- `route_stops.travel_time_minutes_from_prev`
- `route_stops.job_assignment_id`
- `job_assignments.scheduled_start_at`
- `job_assignments.sequence_in_route`

**Data Quality Requirements**:
- Route stops must have travel time populated
- Jobs must be part of route plans for accurate travel time

**View Definition**:

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS kpi_average_travel_time AS
WITH travel_times AS (
  SELECT 
    ja.org_id,
    ja.technician_id,
    DATE_TRUNC('day', ja.scheduled_start_at) AS date,
    rs.travel_time_minutes_from_prev AS travel_minutes,
    ja.id AS assignment_id
  FROM job_assignments ja
  INNER JOIN route_stops rs ON rs.job_assignment_id = ja.id
  WHERE rs.travel_time_minutes_from_prev IS NOT NULL
    AND rs.travel_time_minutes_from_prev > 0
)
SELECT 
  org_id,
  technician_id,
  date,
  COUNT(*) AS job_count,
  ROUND(AVG(travel_minutes), 2) AS avg_travel_minutes,
  ROUND(MIN(travel_minutes), 2) AS min_travel_minutes,
  ROUND(MAX(travel_minutes), 2) AS max_travel_minutes,
  ROUND(SUM(travel_minutes), 2) AS total_travel_minutes
FROM travel_times
GROUP BY org_id, technician_id, date;

CREATE INDEX IF NOT EXISTS idx_kpi_average_travel_time_org_tech_date 
  ON kpi_average_travel_time(org_id, technician_id, date DESC);

-- Overall average (not per technician)
CREATE MATERIALIZED VIEW IF NOT EXISTS kpi_average_travel_time_overall AS
SELECT 
  org_id,
  DATE_TRUNC('day', scheduled_start_at) AS date,
  COUNT(*) AS job_count,
  ROUND(AVG(travel_minutes), 2) AS avg_travel_minutes
FROM (
  SELECT 
    ja.org_id,
    DATE_TRUNC('day', ja.scheduled_start_at) AS scheduled_start_at,
    rs.travel_time_minutes_from_prev AS travel_minutes
  FROM job_assignments ja
  INNER JOIN route_stops rs ON rs.job_assignment_id = ja.id
  WHERE rs.travel_time_minutes_from_prev IS NOT NULL
    AND rs.travel_time_minutes_from_prev > 0
) travel_times
GROUP BY org_id, DATE_TRUNC('day', scheduled_start_at);

CREATE INDEX IF NOT EXISTS idx_kpi_average_travel_time_overall_org_date 
  ON kpi_average_travel_time_overall(org_id, date DESC);
```

### 3.5 KPI 4: Emergency Response Time

**Formula**:
```
Emergency Response Time = Time from Job Creation to Assignment Creation (or Arrival)

Where:
- Response Time = job_assignments.created_at - dispatch_jobs.created_at (for assignment)
- Or actual_arrival_at - dispatch_jobs.created_at (for arrival)
```

**Source Fields**:
- `dispatch_jobs.created_at`
- `dispatch_jobs.priority`
- `job_assignments.created_at`
- `job_assignments.actual_arrival_at`
- `route_stops.actual_arrival_at`

**Data Quality Requirements**:
- Jobs must have `priority = 'emergency'`
- Assignments must exist for emergency jobs
- Actual arrival times preferred but assignment creation time acceptable

**View Definition**:

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS kpi_emergency_response_time AS
WITH emergency_jobs AS (
  SELECT 
    dj.id AS job_id,
    dj.org_id,
    dj.created_at AS job_created_at,
    dj.priority,
    MIN(ja.created_at) AS first_assignment_created_at,
    MIN(rs.actual_arrival_at) AS first_arrival_at
  FROM dispatch_jobs dj
  LEFT JOIN job_assignments ja ON ja.dispatch_job_id = dj.id
  LEFT JOIN route_stops rs ON rs.job_assignment_id = ja.id AND rs.stop_type = 'job'
  WHERE dj.priority = 'emergency'
  GROUP BY dj.id, dj.org_id, dj.created_at, dj.priority
)
SELECT 
  org_id,
  DATE_TRUNC('day', job_created_at) AS date,
  COUNT(*) AS emergency_job_count,
  ROUND(AVG(
    EXTRACT(EPOCH FROM (
      COALESCE(first_arrival_at, first_assignment_created_at, job_created_at) - job_created_at
    )) / 60
  ), 2) AS avg_response_minutes,
  ROUND(MIN(
    EXTRACT(EPOCH FROM (
      COALESCE(first_arrival_at, first_assignment_created_at, job_created_at) - job_created_at
    )) / 60
  ), 2) AS min_response_minutes,
  ROUND(MAX(
    EXTRACT(EPOCH FROM (
      COALESCE(first_arrival_at, first_assignment_created_at, job_created_at) - job_created_at
    )) / 60
  ), 2) AS max_response_minutes,
  COUNT(*) FILTER (WHERE 
    EXTRACT(EPOCH FROM (
      COALESCE(first_arrival_at, first_assignment_created_at, job_created_at) - job_created_at
    )) / 60 <= 30
  ) AS responded_within_30min_count,
  COUNT(*) FILTER (WHERE 
    EXTRACT(EPOCH FROM (
      COALESCE(first_arrival_at, first_assignment_created_at, job_created_at) - job_created_at
    )) / 60 <= 60
  ) AS responded_within_60min_count
FROM emergency_jobs
GROUP BY org_id, DATE_TRUNC('day', job_created_at);

CREATE INDEX IF NOT EXISTS idx_kpi_emergency_response_time_org_date 
  ON kpi_emergency_response_time(org_id, date DESC);
```

### 3.6 Combined KPI View

**Purpose**: Single view for dashboard display

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS kpi_dispatch_summary AS
SELECT 
  org_id,
  date,
  -- On-time arrival
  COALESCE(otar.on_time_rate_percent, 0) AS on_time_arrival_rate,
  COALESCE(otar.total_completed_count, 0) AS completed_jobs_count,
  -- Utilization (average across technicians)
  ROUND(AVG(util.utilization_percent), 2) AS avg_utilization_percent,
  COUNT(DISTINCT util.technician_id) AS active_technicians_count,
  -- Travel time
  COALESCE(att.avg_travel_minutes, 0) AS avg_travel_minutes,
  COALESCE(att.job_count, 0) AS jobs_with_travel_data,
  -- Emergency response
  COALESCE(ert.avg_response_minutes, 0) AS avg_emergency_response_minutes,
  COALESCE(ert.emergency_job_count, 0) AS emergency_jobs_count
FROM (
  SELECT DISTINCT org_id, date 
  FROM kpi_on_time_arrival_rate
  UNION
  SELECT DISTINCT org_id, date 
  FROM kpi_technician_utilization
  UNION
  SELECT DISTINCT org_id, date 
  FROM kpi_average_travel_time_overall
  UNION
  SELECT DISTINCT org_id, date 
  FROM kpi_emergency_response_time
) dates
LEFT JOIN kpi_on_time_arrival_rate otar ON 
  otar.org_id = dates.org_id AND otar.date = dates.date
LEFT JOIN kpi_technician_utilization util ON 
  util.org_id = dates.org_id AND util.date = dates.date
LEFT JOIN kpi_average_travel_time_overall att ON 
  att.org_id = dates.org_id AND att.date = dates.date
LEFT JOIN kpi_emergency_response_time ert ON 
  ert.org_id = dates.org_id AND ert.date = dates.date
GROUP BY 
  dates.org_id, 
  dates.date,
  otar.on_time_rate_percent,
  otar.total_completed_count,
  att.avg_travel_minutes,
  att.job_count,
  ert.avg_response_minutes,
  ert.emergency_job_count;

CREATE INDEX IF NOT EXISTS idx_kpi_dispatch_summary_org_date 
  ON kpi_dispatch_summary(org_id, date DESC);
```

### 3.7 Materialized View Refresh Strategy

**Refresh Schedule**:

```sql
-- Refresh function for all KPIs
CREATE OR REPLACE FUNCTION refresh_all_kpi_views()
RETURNS void AS $$
BEGIN
  REFRESH MATERIALIZED VIEW CONCURRENTLY kpi_on_time_arrival_rate;
  REFRESH MATERIALIZED VIEW CONCURRENTLY kpi_on_time_arrival_rate_by_technician;
  REFRESH MATERIALIZED VIEW CONCURRENTLY kpi_technician_utilization;
  REFRESH MATERIALIZED VIEW CONCURRENTLY kpi_average_travel_time;
  REFRESH MATERIALIZED VIEW CONCURRENTLY kpi_average_travel_time_overall;
  REFRESH MATERIALIZED VIEW CONCURRENTLY kpi_emergency_response_time;
  REFRESH MATERIALIZED VIEW CONCURRENTLY kpi_dispatch_summary;
END;
$$ LANGUAGE plpgsql;

-- Schedule via pg_cron (runs daily at 2 AM)
SELECT cron.schedule(
  'refresh-kpi-views',
  '0 2 * * *', -- Daily at 2 AM
  $$SELECT refresh_all_kpi_views();$$
);
```

**Manual Refresh**: Can be triggered via Edge Function or admin UI

---

## 4. Story DISP-059: Add Auditability Hooks for Schedule Changes

### 4.1 Audit Trail Table

**Purpose**: Track all schedule changes for compliance and debugging

**DDL**:

```sql
CREATE TABLE IF NOT EXISTS dispatch_audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  action_type TEXT NOT NULL, -- 'assignment_created', 'assignment_updated', 'assignment_canceled', 'job_created', 'route_optimized', etc.
  entity_type TEXT NOT NULL, -- 'job_assignment', 'dispatch_job', 'route_plan'
  entity_id UUID NOT NULL,
  change_source TEXT NOT NULL, -- 'manual', 'auto_schedule', 'optimizer', 'emergency_insert', 'calendar_sync'
  run_id UUID, -- Correlation ID for automated changes (from optimization_runs, etc.)
  before_state JSONB, -- Snapshot of entity before change
  after_state JSONB, -- Snapshot of entity after change
  changed_fields TEXT[], -- Array of field names that changed
  reason TEXT, -- Optional reason/notes for change
  metadata JSONB, -- Additional context (e.g., optimization strategy, affected assignments)
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Indexes
CREATE INDEX idx_dispatch_audit_log_org_id ON dispatch_audit_log(org_id);
CREATE INDEX idx_dispatch_audit_log_org_id_created_at ON dispatch_audit_log(org_id, created_at DESC);
CREATE INDEX idx_dispatch_audit_log_entity_type_id ON dispatch_audit_log(entity_type, entity_id);
CREATE INDEX idx_dispatch_audit_log_user_id ON dispatch_audit_log(user_id) WHERE user_id IS NOT NULL;
CREATE INDEX idx_dispatch_audit_log_run_id ON dispatch_audit_log(run_id) WHERE run_id IS NOT NULL;
CREATE INDEX idx_dispatch_audit_log_action_type ON dispatch_audit_log(action_type);

-- RLS Policy: Only admins and dispatchers can read audit logs
ALTER TABLE dispatch_audit_log ENABLE ROW LEVEL SECURITY;

CREATE POLICY "dispatch_audit_log_admin_dispatcher_read"
ON dispatch_audit_log FOR SELECT
USING (
  org_id = get_user_org_id() AND
  get_user_role() IN ('admin', 'dispatcher')
);
```

### 4.2 Audit Trigger Functions

**Assignment Update Trigger**:

```sql
CREATE OR REPLACE FUNCTION audit_job_assignment_changes()
RETURNS TRIGGER AS $$
DECLARE
  v_user_id UUID;
  v_change_source TEXT;
  v_run_id UUID;
  v_reason TEXT;
BEGIN
  -- Get current user ID from JWT (if available)
  v_user_id := auth.uid();
  
  -- Determine change source from context (would be set via session variable or function parameter)
  v_change_source := COALESCE(
    current_setting('app.change_source', true),
    'manual'
  );
  
  v_run_id := NULLIF(current_setting('app.run_id', true), '')::UUID;
  v_reason := NULLIF(current_setting('app.change_reason', true), '');

  -- Only audit if significant fields changed
  IF (OLD.scheduled_start_at IS DISTINCT FROM NEW.scheduled_start_at) OR
     (OLD.scheduled_end_at IS DISTINCT FROM NEW.scheduled_end_at) OR
     (OLD.technician_id IS DISTINCT FROM NEW.technician_id) OR
     (OLD.status IS DISTINCT FROM NEW.status) THEN
    
    INSERT INTO dispatch_audit_log (
      org_id,
      user_id,
      action_type,
      entity_type,
      entity_id,
      change_source,
      run_id,
      before_state,
      after_state,
      changed_fields,
      reason,
      metadata
    ) VALUES (
      NEW.org_id,
      v_user_id,
      CASE 
        WHEN OLD.id IS NULL THEN 'assignment_created'
        WHEN NEW.status = 'canceled' THEN 'assignment_canceled'
        ELSE 'assignment_updated'
      END,
      'job_assignment',
      NEW.id,
      v_change_source,
      v_run_id,
      jsonb_build_object(
        'scheduled_start_at', OLD.scheduled_start_at,
        'scheduled_end_at', OLD.scheduled_end_at,
        'technician_id', OLD.technician_id,
        'status', OLD.status,
        'arrival_window_start', OLD.arrival_window_start,
        'arrival_window_end', OLD.arrival_window_end
      ),
      jsonb_build_object(
        'scheduled_start_at', NEW.scheduled_start_at,
        'scheduled_end_at', NEW.scheduled_end_at,
        'technician_id', NEW.technician_id,
        'status', NEW.status,
        'arrival_window_start', NEW.arrival_window_start,
        'arrival_window_end', NEW.arrival_window_end
      ),
      ARRAY[]::TEXT[] || 
        CASE WHEN OLD.scheduled_start_at IS DISTINCT FROM NEW.scheduled_start_at THEN ARRAY['scheduled_start_at'] ELSE ARRAY[] END ||
        CASE WHEN OLD.scheduled_end_at IS DISTINCT FROM NEW.scheduled_end_at THEN ARRAY['scheduled_end_at'] ELSE ARRAY[] END ||
        CASE WHEN OLD.technician_id IS DISTINCT FROM NEW.technician_id THEN ARRAY['technician_id'] ELSE ARRAY[] END ||
        CASE WHEN OLD.status IS DISTINCT FROM NEW.status THEN ARRAY['status'] ELSE ARRAY[] END,
      v_reason,
      jsonb_build_object(
        'dispatch_job_id', NEW.dispatch_job_id,
        'sequence_in_route', NEW.sequence_in_route
      )
    );
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Trigger
CREATE TRIGGER trigger_audit_job_assignment_changes
  AFTER INSERT OR UPDATE ON job_assignments
  FOR EACH ROW
  EXECUTE FUNCTION audit_job_assignment_changes();
```

**Job Update Trigger**:

```sql
CREATE OR REPLACE FUNCTION audit_dispatch_job_changes()
RETURNS TRIGGER AS $$
DECLARE
  v_user_id UUID;
  v_change_source TEXT;
BEGIN
  v_user_id := auth.uid();
  v_change_source := COALESCE(
    current_setting('app.change_source', true),
    'manual'
  );

  -- Audit status changes and priority changes
  IF (OLD.status IS DISTINCT FROM NEW.status) OR
     (OLD.priority IS DISTINCT FROM NEW.priority) THEN
    
    INSERT INTO dispatch_audit_log (
      org_id,
      user_id,
      action_type,
      entity_type,
      entity_id,
      change_source,
      before_state,
      after_state,
      changed_fields,
      metadata
    ) VALUES (
      NEW.org_id,
      v_user_id,
      CASE 
        WHEN OLD.id IS NULL THEN 'job_created'
        WHEN OLD.status IS DISTINCT FROM NEW.status THEN 'job_status_changed'
        WHEN OLD.priority IS DISTINCT FROM NEW.priority THEN 'job_priority_changed'
        ELSE 'job_updated'
      END,
      'dispatch_job',
      NEW.id,
      v_change_source,
      jsonb_build_object(
        'status', OLD.status,
        'priority', OLD.priority,
        'title', OLD.title
      ),
      jsonb_build_object(
        'status', NEW.status,
        'priority', NEW.priority,
        'title', NEW.title
      ),
      ARRAY[]::TEXT[] ||
        CASE WHEN OLD.status IS DISTINCT FROM NEW.status THEN ARRAY['status'] ELSE ARRAY[] END ||
        CASE WHEN OLD.priority IS DISTINCT FROM NEW.priority THEN ARRAY['priority'] ELSE ARRAY[] END,
      jsonb_build_object(
        'customer_id', NEW.customer_id,
        'location_id', NEW.location_id
      )
    );
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER trigger_audit_dispatch_job_changes
  AFTER INSERT OR UPDATE ON dispatch_jobs
  FOR EACH ROW
  EXECUTE FUNCTION audit_dispatch_job_changes();
```

### 4.3 Setting Change Context in Edge Functions

**Helper Function for Edge Functions**:

```typescript
// File: supabase/functions/_shared/audit-context.ts

export async function setAuditContext(
  supabase: SupabaseClient,
  changeSource: 'manual' | 'auto_schedule' | 'optimizer' | 'emergency_insert' | 'calendar_sync',
  runId?: string,
  reason?: string
): Promise<void> {
  // Set session variables for triggers to read
  await supabase.rpc('set_audit_context', {
    p_change_source: changeSource,
    p_run_id: runId || null,
    p_reason: reason || null
  });
}

// Database function to set context
CREATE OR REPLACE FUNCTION set_audit_context(
  p_change_source TEXT,
  p_run_id UUID DEFAULT NULL,
  p_reason TEXT DEFAULT NULL
)
RETURNS void AS $$
BEGIN
  PERFORM set_config('app.change_source', p_change_source, true);
  IF p_run_id IS NOT NULL THEN
    PERFORM set_config('app.run_id', p_run_id::TEXT, true);
  END IF;
  IF p_reason IS NOT NULL THEN
    PERFORM set_config('app.change_reason', p_reason, true);
  END IF;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

**Usage in Edge Functions**:

```typescript
// Example: In auto-schedule Edge Function
import { setAuditContext } from '../_shared/audit-context.ts';

// Before creating assignment
await setAuditContext(supabase, 'auto_schedule', runId, 'Auto-scheduled via optimization');

// Create assignment (trigger will use context)
await supabase.from('job_assignments').insert({...});
```

### 4.4 Audit Log Query API

**Edge Function**: `get-audit-logs`

**Purpose**: Retrieve audit logs with filtering

**Authorization**: `admin`, `dispatcher`

**Query Parameters**:

```typescript
interface AuditLogQuery {
  entity_type?: 'job_assignment' | 'dispatch_job' | 'route_plan';
  entity_id?: string;
  action_type?: string;
  change_source?: string;
  user_id?: string;
  run_id?: string;
  start_date?: string; // ISO 8601 date
  end_date?: string; // ISO 8601 date
  limit?: number; // default: 100, max: 1000
  offset?: number; // default: 0
}
```

**Response Schema**:

```typescript
interface AuditLogResponse {
  id: string;
  user_id: string | null;
  user_name: string | null; // From profiles
  action_type: string;
  entity_type: string;
  entity_id: string;
  change_source: string;
  run_id: string | null;
  before_state: any;
  after_state: any;
  changed_fields: string[];
  reason: string | null;
  metadata: any;
  created_at: string;
}
```

**Implementation**:

```typescript
Deno.serve(async (req) => {
  if (req.method !== 'GET') {
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

  const url = new URL(req.url);
  const entityType = url.searchParams.get('entity_type');
  const entityId = url.searchParams.get('entity_id');
  const actionType = url.searchParams.get('action_type');
  const changeSource = url.searchParams.get('change_source');
  const userId = url.searchParams.get('user_id');
  const runId = url.searchParams.get('run_id');
  const startDate = url.searchParams.get('start_date');
  const endDate = url.searchParams.get('end_date');
  const limit = Math.min(parseInt(url.searchParams.get('limit') || '100'), 1000);
  const offset = parseInt(url.searchParams.get('offset') || '0');

  let query = supabase
    .from('dispatch_audit_log')
    .select(`
      *,
      profiles!dispatch_audit_log_user_id_fkey(
        user_id,
        role
      )
    `)
    .eq('org_id', auth.orgId)
    .order('created_at', { ascending: false })
    .range(offset, offset + limit - 1);

  if (entityType) {
    query = query.eq('entity_type', entityType);
  }
  if (entityId) {
    query = query.eq('entity_id', entityId);
  }
  if (actionType) {
    query = query.eq('action_type', actionType);
  }
  if (changeSource) {
    query = query.eq('change_source', changeSource);
  }
  if (userId) {
    query = query.eq('user_id', userId);
  }
  if (runId) {
    query = query.eq('run_id', runId);
  }
  if (startDate) {
    query = query.gte('created_at', `${startDate}T00:00:00Z`);
  }
  if (endDate) {
    query = query.lte('created_at', `${endDate}T23:59:59Z`);
  }

  const { data: logs, error: logsError } = await query;

  if (logsError) {
    return errorResponse('Failed to fetch audit logs', 500, 'FETCH_ERROR');
  }

  // Get total count
  let countQuery = supabase
    .from('dispatch_audit_log')
    .select('id', { count: 'exact', head: true })
    .eq('org_id', auth.orgId);

  if (entityType) countQuery = countQuery.eq('entity_type', entityType);
  if (entityId) countQuery = countQuery.eq('entity_id', entityId);
  if (actionType) countQuery = countQuery.eq('action_type', actionType);
  if (changeSource) countQuery = countQuery.eq('change_source', changeSource);
  if (userId) countQuery = countQuery.eq('user_id', userId);
  if (runId) countQuery = countQuery.eq('run_id', runId);
  if (startDate) countQuery = countQuery.gte('created_at', `${startDate}T00:00:00Z`);
  if (endDate) countQuery = countQuery.lte('created_at', `${endDate}T23:59:59Z`);

  const { count } = await countQuery;

  // Transform response
  const transformedLogs = (logs || []).map((log: any) => ({
    id: log.id,
    user_id: log.user_id,
    user_name: log.profiles?.user_id ? 'User' : null, // Would join with profiles for name
    action_type: log.action_type,
    entity_type: log.entity_type,
    entity_id: log.entity_id,
    change_source: log.change_source,
    run_id: log.run_id,
    before_state: log.before_state,
    after_state: log.after_state,
    changed_fields: log.changed_fields,
    reason: log.reason,
    metadata: log.metadata,
    created_at: log.created_at
  }));

  return successResponse({
    logs: transformedLogs,
    pagination: {
      total: count || 0,
      limit,
      offset,
      has_more: (count || 0) > offset + limit
    }
  });
});
```

---

## 5. Next.js UI Components

### 5.1 KPI Dashboard Page

**File**: `app/dispatch/analytics/page.tsx`

```typescript
'use client';

import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { createSupabaseClient } from '@/lib/supabase/client';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { Calendar } from '@/components/ui/calendar';
import { Popover, PopoverContent, PopoverTrigger } from '@/components/ui/popover';
import { Button } from '@/components/ui/button';
import { Skeleton } from '@/components/ui/skeleton';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { format, subDays } from 'date-fns';
import { CalendarIcon, TrendingUp, Clock, Users, AlertTriangle } from 'lucide-react';
import { cn } from '@/lib/utils';
import { KPISummaryCards } from './components/KPISummaryCards';
import { OnTimeArrivalChart } from './components/OnTimeArrivalChart';
import { UtilizationChart } from './components/UtilizationChart';
import { TravelTimeChart } from './components/TravelTimeChart';
import { EmergencyResponseChart } from './components/EmergencyResponseChart';

export default function AnalyticsPage() {
  const [dateRange, setDateRange] = useState({
    start: subDays(new Date(), 30),
    end: new Date()
  });
  const [selectedTechnician, setSelectedTechnician] = useState<string>('all');

  const supabase = createSupabaseClient();

  const { data: kpiData, isLoading, error } = useQuery({
    queryKey: ['kpi-summary', dateRange.start, dateRange.end, selectedTechnician],
    queryFn: async () => {
      const { data: { session } } = await supabase.auth.getSession();
      
      const params = new URLSearchParams({
        start_date: format(dateRange.start, 'yyyy-MM-dd'),
        end_date: format(dateRange.end, 'yyyy-MM-dd')
      });
      if (selectedTechnician !== 'all') {
        params.append('technician_id', selectedTechnician);
      }

      const response = await fetch(`/api/dispatch/analytics/kpi-summary?${params.toString()}`, {
        headers: {
          'Authorization': `Bearer ${session?.access_token}`
        }
      });

      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.error?.message || 'Failed to load KPI data');
      }

      return response.json();
    }
  });

  if (isLoading) {
    return (
      <div className="container mx-auto py-8 space-y-4">
        <Skeleton className="h-12 w-64" />
        <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
          {[1, 2, 3, 4].map((i) => (
            <Skeleton key={i} className="h-32" />
          ))}
        </div>
      </div>
    );
  }

  if (error) {
    return (
      <div className="container mx-auto py-8">
        <Alert variant="destructive">
          <AlertTriangle className="h-4 w-4" />
          <AlertDescription>
            Failed to load analytics data. Please try again.
          </AlertDescription>
        </Alert>
      </div>
    );
  }

  return (
    <div className="container mx-auto py-8 space-y-6">
      <div className="flex items-center justify-between">
        <div>
          <h1 className="text-2xl font-bold mb-2">Dispatch Analytics</h1>
          <p className="text-muted-foreground">
            Key performance indicators and metrics
          </p>
        </div>
        <div className="flex items-center gap-4">
          <Popover>
            <PopoverTrigger asChild>
              <Button variant="outline" className="w-[280px] justify-start text-left font-normal">
                <CalendarIcon className="mr-2 h-4 w-4" />
                {format(dateRange.start, 'MMM d')} - {format(dateRange.end, 'MMM d')}
              </Button>
            </PopoverTrigger>
            <PopoverContent className="w-auto p-0">
              <Calendar
                mode="range"
                selected={{ from: dateRange.start, to: dateRange.end }}
                onSelect={(range) => {
                  if (range?.from && range?.to) {
                    setDateRange({ start: range.from, end: range.to });
                  }
                }}
                initialFocus
              />
            </PopoverContent>
          </Popover>
          <Select value={selectedTechnician} onValueChange={setSelectedTechnician}>
            <SelectTrigger className="w-[200px]">
              <SelectValue placeholder="All Technicians" />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="all">All Technicians</SelectItem>
              {/* Technicians loaded from API */}
            </SelectContent>
          </Select>
        </div>
      </div>

      <KPISummaryCards data={kpiData?.data} />

      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <OnTimeArrivalChart 
          startDate={dateRange.start} 
          endDate={dateRange.end}
          technicianId={selectedTechnician !== 'all' ? selectedTechnician : undefined}
        />
        <UtilizationChart 
          startDate={dateRange.start} 
          endDate={dateRange.end}
          technicianId={selectedTechnician !== 'all' ? selectedTechnician : undefined}
        />
        <TravelTimeChart 
          startDate={dateRange.start} 
          endDate={dateRange.end}
          technicianId={selectedTechnician !== 'all' ? selectedTechnician : undefined}
        />
        <EmergencyResponseChart 
          startDate={dateRange.start} 
          endDate={dateRange.end}
        />
      </div>
    </div>
  );
}
```

### 5.2 KPI Summary Cards Component

**File**: `app/dispatch/analytics/components/KPISummaryCards.tsx`

```typescript
'use client';

import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { TrendingUp, Clock, Users, AlertTriangle } from 'lucide-react';
import { cn } from '@/lib/utils';

interface KPISummaryData {
  on_time_arrival_rate: number;
  avg_utilization_percent: number;
  avg_travel_minutes: number;
  avg_emergency_response_minutes: number;
}

interface KPISummaryCardsProps {
  data: KPISummaryData | undefined;
}

export function KPISummaryCards({ data }: KPISummaryCardsProps) {
  if (!data) return null;

  const cards = [
    {
      title: 'On-Time Arrival Rate',
      value: `${data.on_time_arrival_rate.toFixed(1)}%`,
      description: 'Jobs arrived within scheduled window',
      icon: Clock,
      color: data.on_time_arrival_rate >= 90 ? 'text-green-500' : data.on_time_arrival_rate >= 75 ? 'text-yellow-500' : 'text-red-500'
    },
    {
      title: 'Average Utilization',
      value: `${data.avg_utilization_percent.toFixed(1)}%`,
      description: 'Technician capacity utilization',
      icon: Users,
      color: 'text-blue-500'
    },
    {
      title: 'Avg Travel Time',
      value: `${data.avg_travel_minutes.toFixed(1)} min`,
      description: 'Average travel time per job',
      icon: TrendingUp,
      color: 'text-purple-500'
    },
    {
      title: 'Emergency Response',
      value: `${data.avg_emergency_response_minutes.toFixed(1)} min`,
      description: 'Average emergency response time',
      icon: AlertTriangle,
      color: 'text-red-500'
    }
  ];

  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
      {cards.map((card) => {
        const Icon = card.icon;
        return (
          <Card key={card.title}>
            <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
              <CardTitle className="text-sm font-medium">{card.title}</CardTitle>
              <Icon className={cn('h-4 w-4', card.color)} />
            </CardHeader>
            <CardContent>
              <div className="text-2xl font-bold">{card.value}</div>
              <p className="text-xs text-muted-foreground mt-1">{card.description}</p>
            </CardContent>
          </Card>
        );
      })}
    </div>
  );
}
```

### 5.3 Chart Components

**File**: `app/dispatch/analytics/components/OnTimeArrivalChart.tsx`

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';
import { createSupabaseClient } from '@/lib/supabase/client';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Skeleton } from '@/components/ui/skeleton';
import { format } from 'date-fns';
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer } from 'recharts';

interface OnTimeArrivalChartProps {
  startDate: Date;
  endDate: Date;
  technicianId?: string;
}

export function OnTimeArrivalChart({ startDate, endDate, technicianId }: OnTimeArrivalChartProps) {
  const supabase = createSupabaseClient();

  const { data, isLoading } = useQuery({
    queryKey: ['kpi-on-time-arrival', startDate, endDate, technicianId],
    queryFn: async () => {
      const { data: { session } } = await supabase.auth.getSession();
      
      const params = new URLSearchParams({
        start_date: format(startDate, 'yyyy-MM-dd'),
        end_date: format(endDate, 'yyyy-MM-dd')
      });
      if (technicianId) {
        params.append('technician_id', technicianId);
      }

      const response = await fetch(`/api/dispatch/analytics/on-time-arrival?${params.toString()}`, {
        headers: {
          'Authorization': `Bearer ${session?.access_token}`
        }
      });

      if (!response.ok) {
        throw new Error('Failed to load data');
      }

      return response.json();
    }
  });

  if (isLoading) {
    return (
      <Card>
        <CardHeader>
          <CardTitle>On-Time Arrival Rate</CardTitle>
        </CardHeader>
        <CardContent>
          <Skeleton className="h-64 w-full" />
        </CardContent>
      </Card>
    );
  }

  const chartData = (data?.data || []).map((item: any) => ({
    date: format(new Date(item.date), 'MMM d'),
    rate: parseFloat(item.on_time_rate_percent || 0),
    completed: parseInt(item.total_completed_count || 0)
  }));

  return (
    <Card>
      <CardHeader>
        <CardTitle>On-Time Arrival Rate</CardTitle>
        <CardDescription>Percentage of jobs arrived within scheduled window</CardDescription>
      </CardHeader>
      <CardContent>
        <ResponsiveContainer width="100%" height={300}>
          <LineChart data={chartData}>
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis dataKey="date" />
            <YAxis domain={[0, 100]} />
            <Tooltip />
            <Legend />
            <Line type="monotone" dataKey="rate" stroke="#8884d8" name="On-Time Rate %" />
          </LineChart>
        </ResponsiveContainer>
      </CardContent>
    </Card>
  );
}
```

**Note**: Similar chart components would be created for Utilization, Travel Time, and Emergency Response using the same pattern with `recharts` library.

### 5.4 Audit Log View Page

**File**: `app/dispatch/audit/page.tsx`

```typescript
'use client';

import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { createSupabaseClient } from '@/lib/supabase/client';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { Input } from '@/components/ui/input';
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
import { Skeleton } from '@/components/ui/skeleton';
import { format, parseISO } from 'date-fns';
import { Search, Filter } from 'lucide-react';

export default function AuditLogPage() {
  const [filters, setFilters] = useState({
    entity_type: 'all',
    action_type: 'all',
    change_source: 'all',
    search: ''
  });

  const supabase = createSupabaseClient();

  const { data, isLoading } = useQuery({
    queryKey: ['audit-logs', filters],
    queryFn: async () => {
      const { data: { session } } = await supabase.auth.getSession();
      
      const params = new URLSearchParams();
      if (filters.entity_type !== 'all') {
        params.append('entity_type', filters.entity_type);
      }
      if (filters.action_type !== 'all') {
        params.append('action_type', filters.action_type);
      }
      if (filters.change_source !== 'all') {
        params.append('change_source', filters.change_source);
      }

      const response = await fetch(`/api/dispatch/audit-logs?${params.toString()}`, {
        headers: {
          'Authorization': `Bearer ${session?.access_token}`
        }
      });

      if (!response.ok) {
        throw new Error('Failed to load audit logs');
      }

      return response.json();
    }
  });

  const logs = data?.data?.logs || [];

  return (
    <div className="container mx-auto py-8 space-y-6">
      <div>
        <h1 className="text-2xl font-bold mb-2">Audit Log</h1>
        <p className="text-muted-foreground">
          Track all schedule changes and modifications
        </p>
      </div>

      <Card>
        <CardHeader>
          <CardTitle>Filters</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
            <Select
              value={filters.entity_type}
              onValueChange={(value) => setFilters({ ...filters, entity_type: value })}
            >
              <SelectTrigger>
                <SelectValue placeholder="Entity Type" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="all">All Entities</SelectItem>
                <SelectItem value="job_assignment">Job Assignment</SelectItem>
                <SelectItem value="dispatch_job">Dispatch Job</SelectItem>
                <SelectItem value="route_plan">Route Plan</SelectItem>
              </SelectContent>
            </Select>

            <Select
              value={filters.action_type}
              onValueChange={(value) => setFilters({ ...filters, action_type: value })}
            >
              <SelectTrigger>
                <SelectValue placeholder="Action Type" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="all">All Actions</SelectItem>
                <SelectItem value="assignment_created">Assignment Created</SelectItem>
                <SelectItem value="assignment_updated">Assignment Updated</SelectItem>
                <SelectItem value="assignment_canceled">Assignment Canceled</SelectItem>
                <SelectItem value="job_created">Job Created</SelectItem>
                <SelectItem value="route_optimized">Route Optimized</SelectItem>
              </SelectContent>
            </Select>

            <Select
              value={filters.change_source}
              onValueChange={(value) => setFilters({ ...filters, change_source: value })}
            >
              <SelectTrigger>
                <SelectValue placeholder="Change Source" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="all">All Sources</SelectItem>
                <SelectItem value="manual">Manual</SelectItem>
                <SelectItem value="auto_schedule">Auto-Schedule</SelectItem>
                <SelectItem value="optimizer">Optimizer</SelectItem>
                <SelectItem value="emergency_insert">Emergency Insert</SelectItem>
                <SelectItem value="calendar_sync">Calendar Sync</SelectItem>
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
          <CardTitle>Audit Log Entries</CardTitle>
          <CardDescription>
            {data?.data?.pagination?.total || 0} total entries
          </CardDescription>
        </CardHeader>
        <CardContent>
          {isLoading ? (
            <div className="space-y-2">
              {[1, 2, 3].map((i) => (
                <Skeleton key={i} className="h-16 w-full" />
              ))}
            </div>
          ) : logs.length === 0 ? (
            <div className="text-center py-8 text-muted-foreground">
              No audit log entries found
            </div>
          ) : (
            <Table>
              <TableHeader>
                <TableRow>
                  <TableHead>Timestamp</TableHead>
                  <TableHead>User</TableHead>
                  <TableHead>Action</TableHead>
                  <TableHead>Entity</TableHead>
                  <TableHead>Source</TableHead>
                  <TableHead>Changes</TableHead>
                  <TableHead>Reason</TableHead>
                </TableRow>
              </TableHeader>
              <TableBody>
                {logs.map((log: any) => (
                  <TableRow key={log.id}>
                    <TableCell className="text-sm">
                      {format(parseISO(log.created_at), 'MMM d, yyyy h:mm a')}
                    </TableCell>
                    <TableCell className="text-sm">
                      {log.user_name || 'System'}
                    </TableCell>
                    <TableCell>
                      <Badge variant="outline">{log.action_type}</Badge>
                    </TableCell>
                    <TableCell className="text-sm">
                      {log.entity_type} ({log.entity_id.slice(0, 8)}...)
                    </TableCell>
                    <TableCell>
                      <Badge variant="secondary">{log.change_source}</Badge>
                    </TableCell>
                    <TableCell className="text-sm">
                      {log.changed_fields.length > 0 ? (
                        <div className="flex flex-wrap gap-1">
                          {log.changed_fields.map((field: string) => (
                            <Badge key={field} variant="outline" className="text-xs">
                              {field}
                            </Badge>
                          ))}
                        </div>
                      ) : (
                        <span className="text-muted-foreground">-</span>
                      )}
                    </TableCell>
                    <TableCell className="text-sm text-muted-foreground">
                      {log.reason || '-'}
                    </TableCell>
                  </TableRow>
                ))}
              </TableBody>
            </Table>
          )}
        </CardContent>
      </Card>
    </div>
  );
}
```

### 5.5 API Routes

**File**: `app/api/dispatch/analytics/kpi-summary/route.ts`

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
  const startDate = searchParams.get('start_date');
  const endDate = searchParams.get('end_date');
  const technicianId = searchParams.get('technician_id');

  // Query materialized view
  let query = supabase
    .from('kpi_dispatch_summary')
    .select('*')
    .gte('date', startDate || '1970-01-01')
    .lte('date', endDate || '9999-12-31')
    .order('date', { ascending: true });

  const { data, error } = await query;

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  // Aggregate data
  const summary = {
    on_time_arrival_rate: data?.length > 0 
      ? data.reduce((sum, item) => sum + (item.on_time_arrival_rate || 0), 0) / data.length 
      : 0,
    avg_utilization_percent: data?.length > 0
      ? data.reduce((sum, item) => sum + (item.avg_utilization_percent || 0), 0) / data.length
      : 0,
    avg_travel_minutes: data?.length > 0
      ? data.reduce((sum, item) => sum + (item.avg_travel_minutes || 0), 0) / data.length
      : 0,
    avg_emergency_response_minutes: data?.length > 0
      ? data.reduce((sum, item) => sum + (item.avg_emergency_response_minutes || 0), 0) / data.length
      : 0
  };

  return NextResponse.json({ data: summary });
}
```

**File**: `app/api/dispatch/audit-logs/route.ts`

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
  const entityType = searchParams.get('entity_type');
  const actionType = searchParams.get('action_type');
  const changeSource = searchParams.get('change_source');
  const limit = parseInt(searchParams.get('limit') || '100');
  const offset = parseInt(searchParams.get('offset') || '0');

  // Call Edge Function
  const { data, error } = await supabase.functions.invoke('get-audit-logs', {
    body: {
      entity_type: entityType || undefined,
      action_type: actionType || undefined,
      change_source: changeSource || undefined,
      limit,
      offset
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

## 6. Data Quality Requirements

### 6.1 Required Fields for KPIs

**On-Time Arrival Rate**:
- `job_assignments.arrival_window_start` (NOT NULL)
- `job_assignments.arrival_window_end` (NOT NULL)
- `job_assignments.actual_arrival_at` (NOT NULL for completed jobs)
- `route_stops.actual_arrival_at` (alternative source)

**Utilization Percentage**:
- `technician_shifts.starts_at` (NOT NULL)
- `technician_shifts.ends_at` (NOT NULL)
- `job_assignments.scheduled_start_at` (NOT NULL)
- `job_assignments.scheduled_end_at` (NOT NULL)

**Average Travel Time**:
- `route_stops.travel_time_minutes_from_prev` (NOT NULL, > 0)
- `route_stops.job_assignment_id` (NOT NULL)

**Emergency Response Time**:
- `dispatch_jobs.created_at` (NOT NULL)
- `dispatch_jobs.priority` (must be 'emergency')
- `job_assignments.created_at` (NOT NULL for assigned jobs)
- `route_stops.actual_arrival_at` (preferred but optional)

### 6.2 Data Validation

**Validation Queries**:

```sql
-- Check for missing arrival data
SELECT COUNT(*) 
FROM job_assignments 
WHERE status = 'completed' 
  AND actual_arrival_at IS NULL;

-- Check for missing travel times
SELECT COUNT(*) 
FROM route_stops 
WHERE job_assignment_id IS NOT NULL 
  AND travel_time_minutes_from_prev IS NULL;

-- Check for emergency jobs without assignments
SELECT COUNT(*) 
FROM dispatch_jobs 
WHERE priority = 'emergency' 
  AND id NOT IN (SELECT DISTINCT dispatch_job_id FROM job_assignments);
```

---

## 7. Implementation Checklist

### Story DISP-058: Define Scheduling KPI View Specifications

- [ ] **KPI Specifications**:
  - [ ] On-time arrival rate formula documented
  - [ ] Utilization percentage formula documented
  - [ ] Average travel time formula documented
  - [ ] Emergency response time formula documented
  - [ ] Source fields documented for each KPI
  - [ ] Data quality requirements documented

- [ ] **Database Views**:
  - [ ] `kpi_on_time_arrival_rate` materialized view created
  - [ ] `kpi_on_time_arrival_rate_by_technician` materialized view created
  - [ ] `kpi_technician_utilization` materialized view created
  - [ ] `kpi_average_travel_time` materialized view created
  - [ ] `kpi_average_travel_time_overall` materialized view created
  - [ ] `kpi_emergency_response_time` materialized view created
  - [ ] `kpi_dispatch_summary` materialized view created
  - [ ] Indexes created on all views
  - [ ] Refresh functions created
  - [ ] Cron job scheduled for refresh

- [ ] **Next.js UI Components**:
  - [ ] Analytics dashboard page created
  - [ ] KPI summary cards component created
  - [ ] On-time arrival chart component created
  - [ ] Utilization chart component created
  - [ ] Travel time chart component created
  - [ ] Emergency response chart component created
  - [ ] Date range picker implemented
  - [ ] Technician filter implemented
  - [ ] shadcn/ui components used throughout

- [ ] **API Routes**:
  - [ ] GET /api/dispatch/analytics/kpi-summary route implemented
  - [ ] GET /api/dispatch/analytics/on-time-arrival route implemented
  - [ ] GET /api/dispatch/analytics/utilization route implemented
  - [ ] GET /api/dispatch/analytics/travel-time route implemented
  - [ ] GET /api/dispatch/analytics/emergency-response route implemented

### Story DISP-059: Add Auditability Hooks

- [ ] **Audit Trail Table**:
  - [ ] `dispatch_audit_log` table created
  - [ ] Indexes created
  - [ ] RLS policies created
  - [ ] Security access rules documented

- [ ] **Audit Triggers**:
  - [ ] `audit_job_assignment_changes()` function created
  - [ ] Trigger on `job_assignments` created
  - [ ] `audit_dispatch_job_changes()` function created
  - [ ] Trigger on `dispatch_jobs` created
  - [ ] Change context helper functions created

- [ ] **Edge Function Integration**:
  - [ ] `set_audit_context()` database function created
  - [ ] Audit context helper in Edge Functions
  - [ ] Context set in auto-schedule function
  - [ ] Context set in optimize functions
  - [ ] Context set in emergency insert function
  - [ ] Context set in calendar sync functions

- [ ] **Audit Log API**:
  - [ ] `get-audit-logs` Edge Function implemented
  - [ ] Filtering implemented
  - [ ] Pagination implemented
  - [ ] Authorization enforced

- [ ] **Next.js UI Components**:
  - [ ] Audit log page created
  - [ ] Filters implemented
  - [ ] Table display with shadcn/ui
  - [ ] Change diff visualization (optional)
  - [ ] Export functionality (optional)

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 13 – Analytics & KPI Foundations. All KPIs are calculated using PostgreSQL materialized views, and audit trails are captured via database triggers with Next.js UI components providing visualization and access.

**Next Steps**: After completing Epic 13, proceed to Epic 14 (Reliability, Performance, and Observability) which will address non-functional requirements and observability needs.

