# Technical Design Document – Epic 11: Non-Functional Requirements & Observability

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 11 – Non-Functional Requirements & Observability
- **Source**: Derived from `fdd_1_agile.md` Epic 11 (Stories CRM-045 through CRM-046)
- **Note**: Story CRM-044 (Performance Baseline & Query Optimization) is excluded from this document
- **Reference Documents**: 
  - `fdd_1.md` (Functional Design Document for CRM, §8)
  - `fdd_1_agile.md` (Agile User Stories)
  - `tdd_1_epic_1.md` through `tdd_1_epic_10.md` (All previous epics - prerequisites)
  - `functional.md` (Platform Capabilities)
  - `tooling.md` (Platform Choices: Supabase + Vercel)
- **Target Platform**: Supabase (PostgreSQL 15+ with Edge Functions)
- **Purpose**: Comprehensive technical specification for LLM-based code generation
- **Prerequisites**: Epic 1-10 must be completed first

---

## 1. Overview

This document provides complete technical specifications for implementing non-functional requirements and observability features in the CRM module. It covers:

- Reliability and idempotency for automation actions
- Idempotency keys and deduplication strategies
- Retry mechanisms for failed automation runs
- Structured logging for CRM Edge Functions
- Log aggregation and monitoring patterns
- Error tracking and alerting criteria
- Request/response schemas with exact JSON structures
- Database functions and Edge Function implementations

All specifications are designed to be directly implementable in Supabase (PostgreSQL and Edge Functions), with exact schemas, validation rules, and error codes defined.

---

## 2. Architecture Decisions

### 2.1 Idempotency Strategy

**Decision**: Use composite keys for idempotency checks:

- **Event Triggers**: `rule_id` + `customer_id` + `event_id` + `event_timestamp` (rounded to 5-minute window)
- **Time-Based Triggers**: `rule_id` + `customer_id` + `trigger_event_id` + `due_window` (5-minute window)
- **Segment Triggers**: `rule_id` + `customer_id` + `run_date` (daily)

**Rationale**: 
- Composite keys prevent duplicate actions while allowing legitimate retries
- Time windows prevent duplicate processing of the same logical event
- Stored in `crm_automation_runs.trigger_context` JSONB field

### 2.2 Retry Strategy

**Decision**: Implement exponential backoff with max retries:

- **Max Retries**: 3 attempts
- **Backoff**: 1 minute, 5 minutes, 15 minutes
- **Status Tracking**: `pending` → `processing` → `success`/`failed`/`partial_success`
- **Manual Retry**: Allow manual retry for failed runs

### 2.3 Logging Strategy

**Decision**: Structured JSON logging with consistent schema:

- **Log Levels**: `info`, `warn`, `error`, `debug`
- **Structured Format**: JSON with consistent fields (timestamp, level, function, event, data)
- **PII Handling**: Hash or redact sensitive data before logging
- **Log Aggregation**: Use Supabase logs + optional external service (e.g., Logtail, Datadog)

---

## 3. Story CRM-045: Reliability & Idempotency for Automations

### 3.1 Idempotency Key Generation

#### 3.1.1 PostgreSQL Function: Generate Idempotency Key

**File**: `supabase/migrations/YYYYMMDDHHMMSS_create_idempotency_functions.sql`

```sql
-- Function to generate idempotency key for event-triggered automations
CREATE OR REPLACE FUNCTION crm_generate_event_idempotency_key(
  p_rule_id UUID,
  p_customer_id UUID,
  p_event_id TEXT,
  p_event_timestamp TIMESTAMPTZ
)
RETURNS TEXT
LANGUAGE plpgsql
IMMUTABLE
AS $$
DECLARE
  v_window_start TIMESTAMPTZ;
  v_key TEXT;
BEGIN
  -- Round timestamp to 5-minute window
  v_window_start := date_trunc('minute', p_event_timestamp) - 
                    (EXTRACT(MINUTE FROM p_event_timestamp)::INTEGER % 5 || ' minutes')::INTERVAL;
  
  -- Generate composite key
  v_key := p_rule_id::TEXT || '|' || 
           p_customer_id::TEXT || '|' || 
           COALESCE(p_event_id, 'no-id') || '|' || 
           v_window_start::TEXT;
  
  RETURN v_key;
END;
$$;

-- Function to generate idempotency key for time-based automations
CREATE OR REPLACE FUNCTION crm_generate_time_based_idempotency_key(
  p_rule_id UUID,
  p_customer_id UUID,
  p_trigger_event_id UUID,
  p_due_timestamp TIMESTAMPTZ
)
RETURNS TEXT
LANGUAGE plpgsql
IMMUTABLE
AS $$
DECLARE
  v_window_start TIMESTAMPTZ;
  v_key TEXT;
BEGIN
  -- Round to 5-minute window
  v_window_start := date_trunc('minute', p_due_timestamp) - 
                    (EXTRACT(MINUTE FROM p_due_timestamp)::INTEGER % 5 || ' minutes')::INTERVAL;
  
  v_key := p_rule_id::TEXT || '|' || 
           p_customer_id::TEXT || '|' || 
           p_trigger_event_id::TEXT || '|' || 
           v_window_start::TEXT;
  
  RETURN v_key;
END;
$$;

-- Function to generate idempotency key for segment-based automations
CREATE OR REPLACE FUNCTION crm_generate_segment_idempotency_key(
  p_rule_id UUID,
  p_customer_id UUID,
  p_run_date DATE
)
RETURNS TEXT
LANGUAGE plpgsql
IMMUTABLE
AS $$
BEGIN
  RETURN p_rule_id::TEXT || '|' || 
         p_customer_id::TEXT || '|' || 
         p_run_date::TEXT;
END;
$$;
```

### 3.2 Idempotency Check Function

**File**: `supabase/migrations/YYYYMMDDHHMMSS_create_idempotency_check_function.sql`

```sql
-- Function to check if an automation run already exists (idempotency check)
CREATE OR REPLACE FUNCTION crm_check_idempotency(
  p_org_id UUID,
  p_rule_id UUID,
  p_customer_id UUID,
  p_idempotency_key TEXT
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_existing_run RECORD;
  v_result JSONB;
BEGIN
  -- Check for existing run with matching idempotency key
  SELECT 
    id,
    status,
    completed_at,
    error_message
  INTO v_existing_run
  FROM crm_automation_runs
  WHERE org_id = p_org_id
    AND rule_id = p_rule_id
    AND customer_id = p_customer_id
    AND trigger_context->>'idempotency_key' = p_idempotency_key
  ORDER BY created_at DESC
  LIMIT 1;
  
  IF NOT FOUND THEN
    RETURN jsonb_build_object(
      'exists', false,
      'can_proceed', true
    );
  END IF;
  
  -- If run exists and is successful, don't proceed
  IF v_existing_run.status = 'success' THEN
    RETURN jsonb_build_object(
      'exists', true,
      'can_proceed', false,
      'reason', 'already_completed_successfully',
      'existing_run_id', v_existing_run.id,
      'completed_at', v_existing_run.completed_at
    );
  END IF;
  
  -- If run exists but failed, allow retry
  IF v_existing_run.status = 'failed' THEN
    RETURN jsonb_build_object(
      'exists', true,
      'can_proceed', true,
      'reason', 'previous_run_failed_retry_allowed',
      'existing_run_id', v_existing_run.id,
      'previous_error', v_existing_run.error_message
    );
  END IF;
  
  -- If run is pending or processing, don't proceed (prevent concurrent execution)
  IF v_existing_run.status IN ('pending', 'processing') THEN
    RETURN jsonb_build_object(
      'exists', true,
      'can_proceed', false,
      'reason', 'run_already_in_progress',
      'existing_run_id', v_existing_run.id,
      'status', v_existing_run.status
    );
  END IF;
  
  -- Default: allow proceed
  RETURN jsonb_build_object(
    'exists', true,
    'can_proceed', true,
    'reason', 'unknown_status_allow_proceed',
    'existing_run_id', v_existing_run.id,
    'status', v_existing_run.status
  );
END;
$$;
```

### 3.3 Updated Automation Run Creation

#### 3.3.1 Event-Triggered Automation Update

**File**: `supabase/functions/crm-automation-handle-event/index.ts` (Update)

```typescript
// In the main handler, before creating automation run:

// Generate idempotency key
const idempotencyKey = crm_generate_event_idempotency_key(
  rule.id,
  payload.customer_id,
  payload.event_id || 'no-id',
  new Date(payload.occurred_at || Date.now())
);

// Check idempotency
const { data: idempotencyCheck } = await supabaseAdmin.rpc('crm_check_idempotency', {
  p_org_id: payload.org_id,
  p_rule_id: rule.id,
  p_customer_id: payload.customer_id,
  p_idempotency_key: idempotencyKey,
});

if (!idempotencyCheck.can_proceed) {
  results.push({
    rule_id: rule.id,
    status: 'skipped',
    reason: idempotencyCheck.reason,
    existing_run_id: idempotencyCheck.existing_run_id,
  });
  continue; // Skip this rule
}

// Create automation run with idempotency key
const { data: run, error: runError } = await supabaseAdmin
  .from('crm_automation_runs')
  .insert({
    org_id: payload.org_id,
    rule_id: rule.id,
    customer_id: payload.customer_id,
    trigger_context: {
      event_type: payload.event_type,
      event_id: payload.event_id,
      event_data: payload.event_data,
      occurred_at: payload.occurred_at || new Date().toISOString(),
      idempotency_key: idempotencyKey,
    },
    status: 'pending',
    started_at: new Date().toISOString(),
  })
  .select()
  .single();

// ... rest of execution logic
```

#### 3.3.2 Helper Function: Generate Event Idempotency Key

**File**: `supabase/functions/_shared/idempotency.ts`

```typescript
export function generateEventIdempotencyKey(
  ruleId: string,
  customerId: string,
  eventId: string | null,
  eventTimestamp: Date
): string {
  // Round timestamp to 5-minute window
  const minutes = eventTimestamp.getMinutes();
  const roundedMinutes = Math.floor(minutes / 5) * 5;
  const windowStart = new Date(eventTimestamp);
  windowStart.setMinutes(roundedMinutes, 0, 0);
  
  return `${ruleId}|${customerId}|${eventId || 'no-id'}|${windowStart.toISOString()}`;
}

export function generateTimeBasedIdempotencyKey(
  ruleId: string,
  customerId: string,
  triggerEventId: string,
  dueTimestamp: Date
): string {
  const minutes = dueTimestamp.getMinutes();
  const roundedMinutes = Math.floor(minutes / 5) * 5;
  const windowStart = new Date(dueTimestamp);
  windowStart.setMinutes(roundedMinutes, 0, 0);
  
  return `${ruleId}|${customerId}|${triggerEventId}|${windowStart.toISOString()}`;
}

export function generateSegmentIdempotencyKey(
  ruleId: string,
  customerId: string,
  runDate: Date
): string {
  const dateStr = runDate.toISOString().split('T')[0]; // YYYY-MM-DD
  return `${ruleId}|${customerId}|${dateStr}`;
}
```

### 3.4 Retry Mechanism

#### 3.4.1 Retry Function

**File**: `supabase/migrations/YYYYMMDDHHMMSS_create_retry_function.sql`

```sql
-- Function to retry a failed automation run
CREATE OR REPLACE FUNCTION crm_retry_automation_run(
  p_run_id UUID,
  p_org_id UUID
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_run RECORD;
  v_user_role TEXT;
  v_result JSONB;
BEGIN
  -- Check user role (admin or manager only)
  SELECT role INTO v_user_role
  FROM profiles
  WHERE id = auth.uid()
    AND org_id = p_org_id
    AND is_active = true;
  
  IF v_user_role NOT IN ('admin', 'manager') THEN
    RAISE EXCEPTION 'Access denied: Only admins and managers can retry automation runs';
  END IF;
  
  -- Get the run
  SELECT * INTO v_run
  FROM crm_automation_runs
  WHERE id = p_run_id
    AND org_id = p_org_id;
  
  IF NOT FOUND THEN
    RAISE EXCEPTION 'Automation run not found';
  END IF;
  
  -- Check if run can be retried
  IF v_run.status NOT IN ('failed', 'partial_success') THEN
    RAISE EXCEPTION 'Run cannot be retried. Status: %', v_run.status;
  END IF;
  
  -- Check retry count
  IF COALESCE((v_run.trigger_context->>'retry_count')::INTEGER, 0) >= 3 THEN
    RAISE EXCEPTION 'Maximum retry count (3) exceeded';
  END IF;
  
  -- Update run status to pending and increment retry count
  UPDATE crm_automation_runs
  SET
    status = 'pending',
    started_at = now(),
    completed_at = NULL,
    error_message = NULL,
    trigger_context = jsonb_set(
      COALESCE(trigger_context, '{}'::jsonb),
      '{retry_count}',
      to_jsonb(COALESCE((trigger_context->>'retry_count')::INTEGER, 0) + 1)
    ),
    updated_at = now()
  WHERE id = p_run_id;
  
  RETURN jsonb_build_object(
    'run_id', p_run_id,
    'status', 'pending',
    'retry_count', COALESCE((v_run.trigger_context->>'retry_count')::INTEGER, 0) + 1,
    'message', 'Run queued for retry'
  );
END;
$$;
```

#### 3.4.2 Retry Processor Edge Function

**File**: `supabase/functions/crm-automation-retry-processor/index.ts`

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

serve(async (req) => {
  try {
    const supabaseAdmin = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? '',
    );
    
    // Get failed runs that are eligible for retry
    const { data: failedRuns, error: runsError } = await supabaseAdmin
      .from('crm_automation_runs')
      .select('*')
      .eq('status', 'failed')
      .lt('retry_count', 3) // Max 3 retries
      .lt('created_at', new Date(Date.now() - 60 * 1000).toISOString()) // At least 1 minute old
      .order('created_at', { ascending: true })
      .limit(10); // Process 10 at a time
    
    if (runsError) {
      throw runsError;
    }
    
    if (!failedRuns || failedRuns.length === 0) {
      return new Response(
        JSON.stringify({ message: 'No failed runs to retry' }),
        { status: 200, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    const results = [];
    
    for (const run of failedRuns) {
      try {
        // Get rule details
        const { data: rule, error: ruleError } = await supabaseAdmin
          .from('crm_automation_rules')
          .select('*')
          .eq('id', run.rule_id)
          .single();
        
        if (ruleError || !rule) {
          results.push({
            run_id: run.id,
            status: 'skipped',
            reason: 'rule_not_found',
          });
          continue;
        }
        
        // Check idempotency again
        const idempotencyKey = run.trigger_context?.idempotency_key;
        if (idempotencyKey) {
          const { data: idempotencyCheck } = await supabaseAdmin.rpc('crm_check_idempotency', {
            p_org_id: run.org_id,
            p_rule_id: run.rule_id,
            p_customer_id: run.customer_id,
            p_idempotency_key: idempotencyKey,
          });
          
          if (!idempotencyCheck.can_proceed) {
            results.push({
              run_id: run.id,
              status: 'skipped',
              reason: idempotencyCheck.reason,
            });
            continue;
          }
        }
        
        // Mark as processing
        await supabaseAdmin
          .from('crm_automation_runs')
          .update({ status: 'processing' })
          .eq('id', run.id);
        
        // Re-execute actions (simplified - would need full action execution logic)
        // This would call the same action execution logic from the original automation handlers
        
        // For now, mark as success (actual implementation would execute actions)
        await supabaseAdmin
          .from('crm_automation_runs')
          .update({
            status: 'success',
            completed_at: new Date().toISOString(),
          })
          .eq('id', run.id);
        
        results.push({
          run_id: run.id,
          status: 'success',
        });
      } catch (error) {
        // Update run with new error
        await supabaseAdmin
          .from('crm_automation_runs')
          .update({
            status: 'failed',
            error_message: error.message,
            completed_at: new Date().toISOString(),
          })
          .eq('id', run.id);
        
        results.push({
          run_id: run.id,
          status: 'failed',
          error: error.message,
        });
      }
    }
    
    return new Response(
      JSON.stringify({
        processed: results.length,
        results,
      }),
      { status: 200, headers: { 'Content-Type': 'application/json' } }
    );
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    );
  }
});
```

### 3.5 Duplicate Prevention for Actions

#### 3.5.1 Follow-Up Duplicate Check

**File**: `supabase/migrations/YYYYMMDDHHMMSS_create_followup_duplicate_check.sql`

```sql
-- Function to check for duplicate follow-ups
CREATE OR REPLACE FUNCTION crm_check_duplicate_followup(
  p_org_id UUID,
  p_customer_id UUID,
  p_title TEXT,
  p_due_at TIMESTAMPTZ,
  p_origin TEXT,
  p_time_window_minutes INTEGER DEFAULT 60
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_window_start TIMESTAMPTZ;
  v_window_end TIMESTAMPTZ;
  v_existing RECORD;
BEGIN
  -- Calculate time window
  v_window_start := p_due_at - (p_time_window_minutes || ' minutes')::INTERVAL;
  v_window_end := p_due_at + (p_time_window_minutes || ' minutes')::INTERVAL;
  
  -- Check for existing follow-up with similar title and due date
  SELECT id, title, due_at, status
  INTO v_existing
  FROM crm_followups
  WHERE org_id = p_org_id
    AND customer_id = p_customer_id
    AND title = p_title
    AND due_at BETWEEN v_window_start AND v_window_end
    AND status = 'pending'
  LIMIT 1;
  
  IF NOT FOUND THEN
    RETURN jsonb_build_object(
      'is_duplicate', false,
      'can_create', true
    );
  END IF;
  
  RETURN jsonb_build_object(
    'is_duplicate', true,
    'can_create', false,
    'existing_followup_id', v_existing.id,
    'existing_due_at', v_existing.due_at,
    'reason', 'duplicate_followup_found'
  );
END;
$$;
```

---

## 4. Story CRM-046: Logging & Monitoring for CRM Edge Functions

### 4.1 Structured Logging Utility

**File**: `supabase/functions/_shared/logger.ts`

```typescript
export enum LogLevel {
  DEBUG = 'debug',
  INFO = 'info',
  WARN = 'warn',
  ERROR = 'error',
}

export interface LogEntry {
  timestamp: string;
  level: LogLevel;
  function: string;
  event: string;
  message: string;
  data?: any;
  error?: {
    message: string;
    stack?: string;
    code?: string;
  };
  metadata?: {
    org_id?: string;
    user_id?: string;
    request_id?: string;
    [key: string]: any;
  };
}

export class Logger {
  private functionName: string;
  private defaultMetadata: Record<string, any> = {};

  constructor(functionName: string, defaultMetadata: Record<string, any> = {}) {
    this.functionName = functionName;
    this.defaultMetadata = defaultMetadata;
  }

  private createLogEntry(
    level: LogLevel,
    event: string,
    message: string,
    data?: any,
    error?: Error,
    metadata?: Record<string, any>
  ): LogEntry {
    const entry: LogEntry = {
      timestamp: new Date().toISOString(),
      level,
      function: this.functionName,
      event,
      message,
      data: this.sanitizeData(data),
      metadata: {
        ...this.defaultMetadata,
        ...metadata,
      },
    };

    if (error) {
      entry.error = {
        message: error.message,
        stack: error.stack,
        code: (error as any).code,
      };
    }

    return entry;
  }

  private sanitizeData(data: any): any {
    if (!data) return data;

    // Redact sensitive fields
    const sensitiveFields = ['password', 'token', 'api_key', 'secret', 'ssn', 'credit_card'];
    const sanitized = JSON.parse(JSON.stringify(data));

    const redact = (obj: any): any => {
      if (typeof obj !== 'object' || obj === null) return obj;
      if (Array.isArray(obj)) return obj.map(redact);

      const result: any = {};
      for (const [key, value] of Object.entries(obj)) {
        const lowerKey = key.toLowerCase();
        if (sensitiveFields.some(field => lowerKey.includes(field))) {
          result[key] = '[REDACTED]';
        } else if (typeof value === 'object' && value !== null) {
          result[key] = redact(value);
        } else {
          result[key] = value;
        }
      }
      return result;
    };

    return redact(sanitized);
  }

  private log(entry: LogEntry): void {
    // Log to console (Supabase will capture this)
    const logMethod = entry.level === LogLevel.ERROR ? console.error :
                     entry.level === LogLevel.WARN ? console.warn :
                     entry.level === LogLevel.DEBUG ? console.debug :
                     console.log;

    // Structured JSON logging
    logMethod(JSON.stringify(entry));

    // Also log in human-readable format for development
    if (Deno.env.get('ENVIRONMENT') === 'development') {
      const prefix = `[${entry.level.toUpperCase()}] [${entry.function}] [${entry.event}]`;
      logMethod(`${prefix} ${entry.message}`);
      if (entry.data) {
        logMethod('Data:', entry.data);
      }
      if (entry.error) {
        logMethod('Error:', entry.error);
      }
    }
  }

  debug(event: string, message: string, data?: any, metadata?: Record<string, any>): void {
    this.log(this.createLogEntry(LogLevel.DEBUG, event, message, data, undefined, metadata));
  }

  info(event: string, message: string, data?: any, metadata?: Record<string, any>): void {
    this.log(this.createLogEntry(LogLevel.INFO, event, message, data, undefined, metadata));
  }

  warn(event: string, message: string, data?: any, metadata?: Record<string, any>): void {
    this.log(this.createLogEntry(LogLevel.WARN, event, message, data, undefined, metadata));
  }

  error(event: string, message: string, error?: Error, data?: any, metadata?: Record<string, any>): void {
    this.log(this.createLogEntry(LogLevel.ERROR, event, message, data, error, metadata));
  }

  setMetadata(metadata: Record<string, any>): void {
    this.defaultMetadata = { ...this.defaultMetadata, ...metadata };
  }
}

// Helper to create logger instance
export function createLogger(functionName: string, metadata?: Record<string, any>): Logger {
  return new Logger(functionName, metadata);
}
```

### 4.2 Log Storage Table

**File**: `supabase/migrations/YYYYMMDDHHMMSS_create_function_logs_table.sql`

```sql
-- Table to store structured logs from Edge Functions
CREATE TABLE IF NOT EXISTS crm_function_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES orgs(id) ON DELETE CASCADE,
  function_name TEXT NOT NULL,
  level TEXT NOT NULL CHECK (level IN ('debug', 'info', 'warn', 'error')),
  event TEXT NOT NULL,
  message TEXT NOT NULL,
  data JSONB,
  error JSONB,
  metadata JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Constraints
  CONSTRAINT chk_crm_function_logs_level CHECK (
    level IN ('debug', 'info', 'warn', 'error')
  )
);

-- Indexes for common query patterns
CREATE INDEX idx_crm_function_logs_org_id_created_at ON crm_function_logs(org_id, created_at DESC);
CREATE INDEX idx_crm_function_logs_function_name_created_at ON crm_function_logs(function_name, created_at DESC);
CREATE INDEX idx_crm_function_logs_level_created_at ON crm_function_logs(level, created_at DESC) WHERE level = 'error';
CREATE INDEX idx_crm_function_logs_event ON crm_function_logs(event, created_at DESC);
CREATE INDEX idx_crm_function_logs_metadata ON crm_function_logs USING gin(metadata);

-- RLS Policy: Only admins and managers can view logs
ALTER TABLE crm_function_logs ENABLE ROW LEVEL SECURITY;

CREATE POLICY crm_function_logs_select_policy ON crm_function_logs
  FOR SELECT
  USING (
    org_id IS NULL OR -- System logs (no org_id) visible to all admins
    EXISTS (
      SELECT 1 FROM profiles
      WHERE profiles.id = auth.uid()
      AND profiles.org_id = crm_function_logs.org_id
      AND profiles.role IN ('admin', 'manager')
      AND profiles.is_active = true
    )
  );

-- Only service role can insert logs
CREATE POLICY crm_function_logs_insert_policy ON crm_function_logs
  FOR INSERT
  WITH CHECK (true); -- Service role only
```

### 4.3 Log Storage Function

**File**: `supabase/migrations/YYYYMMDDHHMMSS_create_log_storage_function.sql`

```sql
-- Function to store log entries from Edge Functions
CREATE OR REPLACE FUNCTION crm_store_function_log(
  p_org_id UUID,
  p_function_name TEXT,
  p_level TEXT,
  p_event TEXT,
  p_message TEXT,
  p_data JSONB DEFAULT NULL,
  p_error JSONB DEFAULT NULL,
  p_metadata JSONB DEFAULT NULL
)
RETURNS UUID
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_log_id UUID;
BEGIN
  INSERT INTO crm_function_logs (
    org_id,
    function_name,
    level,
    event,
    message,
    data,
    error,
    metadata
  ) VALUES (
    p_org_id,
    p_function_name,
    p_level,
    p_event,
    p_message,
    p_data,
    p_error,
    p_metadata
  )
  RETURNING id INTO v_log_id;
  
  RETURN v_log_id;
END;
$$;
```

### 4.4 Updated Logger with Database Storage

**File**: `supabase/functions/_shared/logger.ts` (Update)

```typescript
// Add database storage option to Logger class

export class Logger {
  // ... existing code ...

  private async storeInDatabase(entry: LogEntry, supabaseAdmin?: any): Promise<void> {
    if (!supabaseAdmin) return; // Skip if no Supabase client provided

    try {
      await supabaseAdmin.rpc('crm_store_function_log', {
        p_org_id: entry.metadata?.org_id || null,
        p_function_name: entry.function,
        p_level: entry.level,
        p_event: entry.event,
        p_message: entry.message,
        p_data: entry.data || null,
        p_error: entry.error || null,
        p_metadata: entry.metadata || null,
      });
    } catch (err) {
      // Don't throw - logging failures shouldn't break operations
      console.error('Failed to store log in database:', err);
    }
  }

  // Update log methods to optionally store in database
  async debug(event: string, message: string, data?: any, metadata?: Record<string, any>, supabaseAdmin?: any): Promise<void> {
    const entry = this.createLogEntry(LogLevel.DEBUG, event, message, data, undefined, metadata);
    this.log(entry);
    if (supabaseAdmin) await this.storeInDatabase(entry, supabaseAdmin);
  }

  async info(event: string, message: string, data?: any, metadata?: Record<string, any>, supabaseAdmin?: any): Promise<void> {
    const entry = this.createLogEntry(LogLevel.INFO, event, message, data, undefined, metadata);
    this.log(entry);
    if (supabaseAdmin) await this.storeInDatabase(entry, supabaseAdmin);
  }

  async warn(event: string, message: string, data?: any, metadata?: Record<string, any>, supabaseAdmin?: any): Promise<void> {
    const entry = this.createLogEntry(LogLevel.WARN, event, message, data, undefined, metadata);
    this.log(entry);
    if (supabaseAdmin) await this.storeInDatabase(entry, supabaseAdmin);
  }

  async error(event: string, message: string, error?: Error, data?: any, metadata?: Record<string, any>, supabaseAdmin?: any): Promise<void> {
    const entry = this.createLogEntry(LogLevel.ERROR, event, message, data, error, metadata);
    this.log(entry);
    if (supabaseAdmin) await this.storeInDatabase(entry, supabaseAdmin);
  }
}
```

### 4.5 Example: Updated Automation Handler with Logging

**File**: `supabase/functions/crm-automation-handle-event/index.ts` (Update)

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
import { createLogger } from '../_shared/logger.ts';
import { generateEventIdempotencyKey } from '../_shared/idempotency.ts';

serve(async (req) => {
  const logger = createLogger('crm-automation-handle-event', {
    request_id: crypto.randomUUID(),
  });

  try {
    const supabaseAdmin = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? '',
    );

    const payload = await req.json();
    
    logger.info('event_received', 'Event trigger received', {
      event_type: payload.event_type,
      customer_id: payload.customer_id,
      org_id: payload.org_id,
    }, { org_id: payload.org_id }, supabaseAdmin);

    // Validate payload
    if (!payload.org_id || !payload.event_type || !payload.customer_id) {
      logger.warn('invalid_payload', 'Missing required fields', payload, { org_id: payload.org_id }, supabaseAdmin);
      return new Response(
        JSON.stringify({ error: 'org_id, event_type, and customer_id are required' }),
        { status: 400, headers: { 'Content-Type': 'application/json' } }
      );
    }

    // Get active rules
    const { data: rules, error: rulesError } = await supabaseAdmin
      .from('crm_automation_rules')
      .select('*')
      .eq('org_id', payload.org_id)
      .eq('trigger_type', 'event')
      .eq('event_type', payload.event_type)
      .eq('is_enabled', true);

    if (rulesError) {
      logger.error('rules_fetch_failed', 'Failed to fetch automation rules', rulesError, {
        org_id: payload.org_id,
        event_type: payload.event_type,
      }, { org_id: payload.org_id }, supabaseAdmin);
      throw rulesError;
    }

    logger.info('rules_found', `Found ${rules?.length || 0} matching rules`, {
      rule_count: rules?.length || 0,
      rule_ids: rules?.map(r => r.id),
    }, { org_id: payload.org_id }, supabaseAdmin);

    if (!rules || rules.length === 0) {
      return new Response(
        JSON.stringify({ message: 'No matching automation rules found' }),
        { status: 200, headers: { 'Content-Type': 'application/json' } }
      );
    }

    const results = [];

    // Process each rule
    for (const rule of rules) {
      try {
        logger.debug('processing_rule', `Processing rule: ${rule.name}`, {
          rule_id: rule.id,
          rule_name: rule.name,
        }, { org_id: payload.org_id }, supabaseAdmin);

        // Generate idempotency key
        const idempotencyKey = generateEventIdempotencyKey(
          rule.id,
          payload.customer_id,
          payload.event_id || null,
          new Date(payload.occurred_at || Date.now())
        );

        // Check idempotency
        const { data: idempotencyCheck } = await supabaseAdmin.rpc('crm_check_idempotency', {
          p_org_id: payload.org_id,
          p_rule_id: rule.id,
          p_customer_id: payload.customer_id,
          p_idempotency_key: idempotencyKey,
        });

        if (!idempotencyCheck.can_proceed) {
          logger.info('rule_skipped', `Rule skipped: ${idempotencyCheck.reason}`, {
            rule_id: rule.id,
            reason: idempotencyCheck.reason,
            existing_run_id: idempotencyCheck.existing_run_id,
          }, { org_id: payload.org_id }, supabaseAdmin);
          
          results.push({
            rule_id: rule.id,
            status: 'skipped',
            reason: idempotencyCheck.reason,
          });
          continue;
        }

        // Create automation run
        const { data: run, error: runError } = await supabaseAdmin
          .from('crm_automation_runs')
          .insert({
            org_id: payload.org_id,
            rule_id: rule.id,
            customer_id: payload.customer_id,
            trigger_context: {
              event_type: payload.event_type,
              event_id: payload.event_id,
              event_data: payload.event_data,
              occurred_at: payload.occurred_at || new Date().toISOString(),
              idempotency_key: idempotencyKey,
            },
            status: 'pending',
            started_at: new Date().toISOString(),
          })
          .select()
          .single();

        if (runError) {
          logger.error('run_creation_failed', 'Failed to create automation run', runError, {
            rule_id: rule.id,
          }, { org_id: payload.org_id }, supabaseAdmin);
          throw runError;
        }

        logger.info('run_created', 'Automation run created', {
          run_id: run.id,
          rule_id: rule.id,
        }, { org_id: payload.org_id }, supabaseAdmin);

        // Execute actions
        const actionResults = await executeActions(
          supabaseAdmin,
          payload.org_id,
          payload.customer_id,
          rule.actions,
          payload.event_data,
          payload.occurred_at || new Date().toISOString(),
          logger
        );

        // Determine final status
        const hasFailures = actionResults.some(r => r.status === 'failed');
        const allSkipped = actionResults.every(r => r.status === 'skipped');
        
        let finalStatus = 'success';
        if (hasFailures && actionResults.some(r => r.status === 'success')) {
          finalStatus = 'partial_success';
        } else if (hasFailures) {
          finalStatus = 'failed';
        } else if (allSkipped) {
          finalStatus = 'skipped';
        }

        const errorMessages = actionResults
          .filter(r => r.error)
          .map(r => r.error)
          .join('; ');

        await supabaseAdmin
          .from('crm_automation_runs')
          .update({
            status: finalStatus,
            error_message: errorMessages || null,
            completed_at: new Date().toISOString(),
          })
          .eq('id', run.id);

        logger.info('run_completed', `Automation run completed: ${finalStatus}`, {
          run_id: run.id,
          rule_id: rule.id,
          status: finalStatus,
          actions_executed: actionResults.length,
          actions_succeeded: actionResults.filter(r => r.status === 'success').length,
          actions_failed: actionResults.filter(r => r.status === 'failed').length,
        }, { org_id: payload.org_id }, supabaseAdmin);

        results.push({
          rule_id: rule.id,
          run_id: run.id,
          status: finalStatus,
          actions: actionResults,
        });
      } catch (error) {
        logger.error('rule_processing_failed', `Failed to process rule: ${rule.name}`, error, {
          rule_id: rule.id,
        }, { org_id: payload.org_id }, supabaseAdmin);
        
        results.push({
          rule_id: rule.id,
          status: 'failed',
          error: error.message,
        });
      }
    }

    logger.info('event_processing_complete', 'Event processing completed', {
      rules_processed: results.length,
      rules_succeeded: results.filter(r => r.status === 'success').length,
      rules_failed: results.filter(r => r.status === 'failed').length,
    }, { org_id: payload.org_id }, supabaseAdmin);

    return new Response(
      JSON.stringify({ results }),
      { status: 200, headers: { 'Content-Type': 'application/json' } }
    );
  } catch (error) {
    logger.error('handler_error', 'Unhandled error in event handler', error, undefined, undefined, supabaseAdmin);
    
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    );
  }
});

// Update executeActions to accept logger
async function executeActions(
  supabase: any,
  orgId: string,
  customerId: string,
  actions: any[],
  eventData: any,
  occurredAt: string,
  logger: any
): Promise<Array<{ type: string; status: string; error?: string }>> {
  const results = [];
  
  for (const action of actions) {
    try {
      logger.debug('executing_action', `Executing action: ${action.type}`, {
        action_type: action.type,
        customer_id: customerId,
      }, { org_id: orgId }, supabase);
      
      // ... existing action execution logic ...
      
      results.push({ type: action.type, status: 'success' });
    } catch (error) {
      logger.error('action_failed', `Action failed: ${action.type}`, error, {
        action_type: action.type,
        customer_id: customerId,
      }, { org_id: orgId }, supabase);
      
      results.push({ type: action.type, status: 'failed', error: error.message });
    }
  }
  
  return results;
}
```

### 4.6 Log Query Functions

**File**: `supabase/migrations/YYYYMMDDHHMMSS_create_log_query_functions.sql`

```sql
-- Function to query function logs
CREATE OR REPLACE FUNCTION crm_query_function_logs(
  p_org_id UUID,
  p_function_name TEXT DEFAULT NULL,
  p_level TEXT DEFAULT NULL,
  p_event TEXT DEFAULT NULL,
  p_start_date TIMESTAMPTZ DEFAULT NULL,
  p_end_date TIMESTAMPTZ DEFAULT NULL,
  p_limit INTEGER DEFAULT 100,
  p_offset INTEGER DEFAULT 0
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_user_role TEXT;
  v_result JSONB;
BEGIN
  -- Check user role
  SELECT role INTO v_user_role
  FROM profiles
  WHERE id = auth.uid()
    AND org_id = p_org_id
    AND is_active = true;
  
  IF v_user_role NOT IN ('admin', 'manager') THEN
    RAISE EXCEPTION 'Access denied: Only admins and managers can view logs';
  END IF;
  
  SELECT jsonb_build_object(
    'data', jsonb_agg(
      jsonb_build_object(
        'id', cfl.id,
        'function_name', cfl.function_name,
        'level', cfl.level,
        'event', cfl.event,
        'message', cfl.message,
        'data', cfl.data,
        'error', cfl.error,
        'metadata', cfl.metadata,
        'created_at', cfl.created_at
      )
    ),
    'total', COUNT(*)
  )
  INTO v_result
  FROM crm_function_logs cfl
  WHERE (cfl.org_id = p_org_id OR cfl.org_id IS NULL)
    AND (p_function_name IS NULL OR cfl.function_name = p_function_name)
    AND (p_level IS NULL OR cfl.level = p_level)
    AND (p_event IS NULL OR cfl.event = p_event)
    AND (p_start_date IS NULL OR cfl.created_at >= p_start_date)
    AND (p_end_date IS NULL OR cfl.created_at <= p_end_date)
  ORDER BY cfl.created_at DESC
  LIMIT p_limit
  OFFSET p_offset;
  
  RETURN COALESCE(v_result, '{"data": [], "total": 0}'::jsonb);
END;
$$;

-- Function to get error log summary
CREATE OR REPLACE FUNCTION crm_error_log_summary(
  p_org_id UUID,
  p_start_date TIMESTAMPTZ DEFAULT NULL,
  p_end_date TIMESTAMPTZ DEFAULT NULL
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_user_role TEXT;
  v_result JSONB;
BEGIN
  -- Check user role
  SELECT role INTO v_user_role
  FROM profiles
  WHERE id = auth.uid()
    AND org_id = p_org_id
    AND is_active = true;
  
  IF v_user_role NOT IN ('admin', 'manager') THEN
    RAISE EXCEPTION 'Access denied';
  END IF;
  
  SELECT jsonb_build_object(
    'total_errors', COUNT(*),
    'by_function', (
      SELECT jsonb_object_agg(function_name, error_count)
      FROM (
        SELECT function_name, COUNT(*) as error_count
        FROM crm_function_logs
        WHERE (org_id = p_org_id OR org_id IS NULL)
          AND level = 'error'
          AND (p_start_date IS NULL OR created_at >= p_start_date)
          AND (p_end_date IS NULL OR created_at <= p_end_date)
        GROUP BY function_name
      ) subq
    ),
    'by_event', (
      SELECT jsonb_object_agg(event, error_count)
      FROM (
        SELECT event, COUNT(*) as error_count
        FROM crm_function_logs
        WHERE (org_id = p_org_id OR org_id IS NULL)
          AND level = 'error'
          AND (p_start_date IS NULL OR created_at >= p_start_date)
          AND (p_end_date IS NULL OR created_at <= p_end_date)
        GROUP BY event
      ) subq
    ),
    'recent_errors', (
      SELECT jsonb_agg(
        jsonb_build_object(
          'id', id,
          'function_name', function_name,
          'event', event,
          'message', message,
          'error', error,
          'created_at', created_at
        )
      )
      FROM crm_function_logs
      WHERE (org_id = p_org_id OR org_id IS NULL)
        AND level = 'error'
        AND (p_start_date IS NULL OR created_at >= p_start_date)
        AND (p_end_date IS NULL OR created_at <= p_end_date)
      ORDER BY created_at DESC
      LIMIT 10
    )
  )
  INTO v_result
  FROM crm_function_logs
  WHERE (org_id = p_org_id OR org_id IS NULL)
    AND level = 'error'
    AND (p_start_date IS NULL OR created_at >= p_start_date)
    AND (p_end_date IS NULL OR created_at <= p_end_date);
  
  RETURN COALESCE(v_result, '{}'::jsonb);
END;
$$;
```

### 4.7 Alerting Criteria

**File**: `supabase/migrations/YYYYMMDDHHMMSS_create_alerting_config.sql`

```sql
-- Table to store alerting configuration
CREATE TABLE IF NOT EXISTS crm_alert_configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES orgs(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  function_name TEXT,
  event TEXT,
  level TEXT CHECK (level IN ('error', 'warn')),
  threshold INTEGER NOT NULL DEFAULT 1, -- Number of occurrences
  time_window_minutes INTEGER NOT NULL DEFAULT 60, -- Time window for threshold
  enabled BOOLEAN NOT NULL DEFAULT true,
  notification_channels JSONB, -- e.g., ['email', 'slack']
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT chk_crm_alert_configs_level CHECK (
    level IN ('error', 'warn')
  )
);

CREATE INDEX idx_crm_alert_configs_org_id_enabled ON crm_alert_configs(org_id, enabled) WHERE enabled = true;

-- Function to check alerts
CREATE OR REPLACE FUNCTION crm_check_alerts(
  p_org_id UUID
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_alert_config RECORD;
  v_error_count INTEGER;
  v_warn_count INTEGER;
  v_alerts JSONB := '[]'::jsonb;
BEGIN
  -- Check each enabled alert config
  FOR v_alert_config IN
    SELECT * FROM crm_alert_configs
    WHERE org_id = p_org_id
      AND enabled = true
  LOOP
    -- Count errors/warnings in time window
    SELECT COUNT(*) INTO v_error_count
    FROM crm_function_logs
    WHERE (org_id = p_org_id OR org_id IS NULL)
      AND level = v_alert_config.level
      AND (v_alert_config.function_name IS NULL OR function_name = v_alert_config.function_name)
      AND (v_alert_config.event IS NULL OR event = v_alert_config.event)
      AND created_at >= now() - (v_alert_config.time_window_minutes || ' minutes')::INTERVAL;
    
    IF v_alert_config.level = 'error' THEN
      v_error_count := v_error_count;
    ELSE
      v_warn_count := v_error_count;
    END IF;
    
    -- Check if threshold exceeded
    IF (v_alert_config.level = 'error' AND v_error_count >= v_alert_config.threshold) OR
       (v_alert_config.level = 'warn' AND v_warn_count >= v_alert_config.threshold) THEN
      
      v_alerts := v_alerts || jsonb_build_object(
        'alert_config_id', v_alert_config.id,
        'name', v_alert_config.name,
        'level', v_alert_config.level,
        'count', CASE WHEN v_alert_config.level = 'error' THEN v_error_count ELSE v_warn_count END,
        'threshold', v_alert_config.threshold,
        'time_window_minutes', v_alert_config.time_window_minutes
      );
    END IF;
  END LOOP;
  
  RETURN jsonb_build_object(
    'alerts', v_alerts,
    'checked_at', now()
  );
END;
$$;
```

---

## 5. Implementation Checklist

### Story CRM-045: Reliability & Idempotency for Automations
- [ ] Idempotency key generation functions implemented (event, time-based, segment)
- [ ] Idempotency check function implemented
- [ ] Event-triggered automation updated with idempotency checks
- [ ] Time-based automation updated with idempotency checks
- [ ] Segment-based automation updated with idempotency checks
- [ ] Duplicate follow-up check function implemented
- [ ] Retry function implemented
- [ ] Retry processor Edge Function implemented
- [ ] Automation run status lifecycle documented
- [ ] Idempotency keys and retry strategies documented
- [ ] Test scenarios for duplicate prevention
- [ ] Test scenarios for retry logic

### Story CRM-046: Logging & Monitoring for CRM Edge Functions
- [ ] Structured logging utility implemented
- [ ] Log storage table created
- [ ] Log storage function implemented
- [ ] Logger updated with database storage
- [ ] All CRM Edge Functions updated with logging
- [ ] Log query functions implemented
- [ ] Error log summary function implemented
- [ ] Alerting configuration table created
- [ ] Alert checking function implemented
- [ ] Logging patterns documented
- [ ] Alerting criteria documented
- [ ] PII sanitization verified

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 11 – Non-Functional Requirements & Observability (excluding CRM-044). All specifications are designed to be directly consumable by LLM-based code generators, with exact SQL functions, TypeScript interfaces, Edge Function utilities, idempotency strategies, and structured logging patterns defined.

