# Technical Design Document – Epic 7: Automation Engine

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 7 – Automation Engine
- **Source**: Derived from `fdd_1_agile.md` Epic 7 (Stories CRM-029 through CRM-032)
- **Reference Documents**: 
  - `fdd_1.md` (Functional Design Document for CRM, §4.6)
  - `fdd_1_agile.md` (Agile User Stories)
  - `tdd_1_epic_1.md` (CRM Core Data Model - prerequisite)
  - `tdd_1_epic_2.md` (Authentication, Authorization & RLS Policies - prerequisite)
  - `tdd_1_epic_3.md` (Customer Management APIs - prerequisite)
  - `tdd_1_epic_5.md` (Follow-Ups & Reminders - prerequisite)
  - `tdd_1_epic_6.md` (Segmentation & Targeting - prerequisite)
  - `functional.md` (Platform Capabilities)
  - `tooling.md` (Platform Choices: Supabase + Vercel)
- **Target Platform**: Supabase (PostgreSQL 15+ with Edge Functions and Cron Jobs)
- **Purpose**: Comprehensive technical specification for LLM-based code generation
- **Prerequisites**: Epic 1 (CRM Core Data Model), Epic 2 (RLS Policies), Epic 3 (Customer Management APIs), Epic 5 (Follow-Ups), and Epic 6 (Segmentation) must be completed first

---

## 1. Overview

This document provides complete technical specifications for implementing the automation engine in Supabase. It covers:

- Manage Automation Rules API (CRUD operations)
- Event-Triggered Automation Handler for real-time event processing
- Time-Based Automation Processor for scheduled follow-ups
- Segment Membership Automation Processor for segment-based actions
- Action execution logic (create follow-up, send message, tag customer)
- Conditions evaluation and filtering
- Error handling and logging via `crm_automation_runs`
- Idempotency strategies to prevent duplicate actions
- Request/response schemas with exact JSON structures
- Validation rules and error handling

All specifications are designed to be directly implementable as Supabase Edge Functions (Deno/TypeScript) and PostgreSQL RPC functions, with exact schemas, validation rules, and error codes defined.

---

## 2. Architecture Decisions

### 2.1 Implementation Approach

**Decision**: Use Edge Functions for all automation processors:

- **Edge Functions**: Recommended for all automation logic due to:
  - External service integration (email/SMS providers)
  - Scheduled cron job support
  - Better error handling and logging
  - Isolation from database load
- **PostgreSQL RPC Functions**: Used for CRUD operations and rule queries

**Rationale**: 
- Edge Functions provide better HTTP semantics, external service integration, and scheduled execution
- RPC functions provide efficient database queries for rule lookup and action execution
- Separation of concerns: Edge Functions handle orchestration, RPC functions handle data operations

### 2.2 Authentication & Authorization

- CRUD APIs require authenticated Supabase user (JWT token)
- Event trigger handler uses service role key (called by other modules)
- Scheduled processors use service role key (internal system operations)
- Role-based access control enforced via RLS (from Epic 2):
  - Only `admin` and `manager` roles can create/update automation rules
  - All roles except `technician` can read automation rules

### 2.3 Trigger Types

**Three trigger types supported**:

1. **`event`**: Triggered by external events (e.g., work order completed, quote sent)
2. **`time_based`**: Triggered after a time offset from an event
3. **`segment_membership`**: Triggered when customers are in specific segments

### 2.4 Action Types

**Supported action types**:

1. **`create_followup`**: Creates a follow-up task
2. **`send_message`**: Sends email/SMS using message template
3. **`tag_customer`**: Adds a tag to a customer

**Future action types** (deferred):
- `update_customer_field`
- `create_interaction`
- `add_to_segment`
- `remove_from_segment`

### 2.5 Idempotency Strategy

- **Event triggers**: Use unique key based on `rule_id` + `customer_id` + `event_id` + `event_timestamp`
- **Time-based triggers**: Use unique key based on `rule_id` + `customer_id` + `trigger_event_id` + `due_window`
- **Segment triggers**: Use unique key based on `rule_id` + `customer_id` + `run_date`
- Store idempotency keys in `crm_automation_runs.trigger_context` JSONB

### 2.6 Error Handling

- All automation runs are logged in `crm_automation_runs`
- Failed runs include error messages in `error_message` field
- Partial failures (some actions succeed, others fail) are logged with status `partial_success`
- Retry logic can be implemented based on run status

---

## 3. Story CRM-029: Manage Automation Rules API

### 3.1 Endpoint Specification

#### 3.1.1 Edge Function Endpoints

**Create**: `POST /crm/automation/rules`
**Update**: `PATCH /crm/automation/rules/:id`
**List**: `GET /crm/automation/rules`

**Method**: `POST`, `PATCH`, `GET`

**Authentication**: Required (Supabase JWT)

**Content-Type**: `application/json`

### 3.2 Request Schema

#### 3.2.1 Create Automation Rule Request

```typescript
interface CreateAutomationRuleRequest {
  // Required fields
  name: string; // Max 100 characters
  trigger_type: 'event' | 'time_based' | 'segment_membership';
  
  // Optional fields
  description?: string; // Max 1000 characters
  is_enabled?: boolean; // Default: true
  
  // Required for event triggers
  event_type?: string; // e.g., 'work_order_completed', 'quote_sent', 'customer_created'
  
  // Required for time_based triggers
  time_offset_minutes?: number; // Minutes after event (positive integer)
  
  // Required for segment_membership triggers
  segment_id?: string; // UUID
  
  // Optional conditions (filters)
  conditions?: {
    operator: 'AND' | 'OR';
    rules: Array<{
      field: string; // Customer field name
      operator: 'equals' | 'not_equals' | 'in' | 'not_in' | 
                'greater_than' | 'less_than' | 'is_null' | 'is_not_null';
      value: any;
    }>;
  };
  
  // Required: actions to execute
  actions: Array<{
    type: 'create_followup' | 'send_message' | 'tag_customer';
    // For create_followup
    followup_title?: string;
    followup_description?: string;
    followup_priority?: 'low' | 'medium' | 'high';
    followup_due_at_offset_minutes?: number; // Offset from trigger time
    // For send_message
    template_id?: string; // UUID of message template
    channel?: 'email' | 'sms';
    // For tag_customer
    tag_id?: string; // UUID of tag
    tag_name?: string; // Name of tag (will be created if doesn't exist)
  }>;
}
```

#### 3.2.2 Update Automation Rule Request

```typescript
interface UpdateAutomationRuleRequest {
  name?: string;
  description?: string;
  is_enabled?: boolean;
  event_type?: string;
  time_offset_minutes?: number;
  segment_id?: string;
  conditions?: {
    operator: 'AND' | 'OR';
    rules: Array<{
      field: string;
      operator: string;
      value: any;
    }>;
  };
  actions?: Array<{
    type: string;
    [key: string]: any;
  }>;
}
```

#### 3.2.3 Actions Schema Examples

**Create Follow-Up Action**:
```json
{
  "type": "create_followup",
  "followup_title": "Follow up on {{event.work_order_type}}",
  "followup_description": "Work order {{event.work_order_id}} was completed",
  "followup_priority": "high",
  "followup_due_at_offset_minutes": 1440
}
```

**Send Message Action**:
```json
{
  "type": "send_message",
  "template_id": "uuid-of-template",
  "channel": "email"
}
```

**Tag Customer Action**:
```json
{
  "type": "tag_customer",
  "tag_id": "uuid-of-tag"
}
```

or

```json
{
  "type": "tag_customer",
  "tag_name": "Follow-up Required"
}
```

### 3.3 Implementation: Edge Function

#### 3.3.1 Edge Function Structure

**File**: `supabase/functions/crm-automation-rules/index.ts`

**Dependencies**: 
- `@supabase/supabase-js` (Supabase client)
- Deno runtime (Supabase Edge Functions)

**Code Structure**:

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'authorization, x-client-info, apikey, content-type',
};

serve(async (req) => {
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders });
  }

  try {
    const supabaseClient = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_ANON_KEY') ?? '',
      {
        global: {
          headers: { Authorization: req.headers.get('Authorization')! },
        },
      }
    );

    const {
      data: { user },
      error: authError,
    } = await supabaseClient.auth.getUser();

    if (authError || !user) {
      return new Response(
        JSON.stringify({ error: 'Unauthorized' }),
        { status: 401, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    }

    const { data: profile } = await supabaseClient
      .from('profiles')
      .select('org_id, role')
      .eq('id', user.id)
      .single();

    if (!profile || !['admin', 'manager'].includes(profile.role)) {
      return new Response(
        JSON.stringify({ error: 'Only admins and managers can manage automation rules' }),
        { status: 403, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    }

    const orgId = profile.org_id;
    const url = new URL(req.url);

    if (req.method === 'POST') {
      // Create rule
      const body = await req.json();
      
      const validationError = validateCreateRuleRequest(body);
      if (validationError) {
        return new Response(
          JSON.stringify({ error: validationError }),
          { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
        );
      }

      const { data: rule, error: createError } = await supabaseClient.rpc(
        'crm_create_automation_rule',
        {
          p_org_id: orgId,
          p_name: body.name,
          p_description: body.description || null,
          p_trigger_type: body.trigger_type,
          p_event_type: body.event_type || null,
          p_time_offset_minutes: body.time_offset_minutes || null,
          p_segment_id: body.segment_id || null,
          p_conditions: body.conditions || null,
          p_actions: body.actions,
          p_is_enabled: body.is_enabled !== false,
          p_created_by_user_id: user.id,
        }
      );

      if (createError) {
        return new Response(
          JSON.stringify({ error: createError.message }),
          { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
        );
      }

      return new Response(
        JSON.stringify(rule),
        { status: 201, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    } else if (req.method === 'PATCH') {
      // Update rule
      const ruleId = url.pathname.split('/').pop();
      const body = await req.json();
      
      const validationError = validateUpdateRuleRequest(body);
      if (validationError) {
        return new Response(
          JSON.stringify({ error: validationError }),
          { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
        );
      }

      const { data: rule, error: updateError } = await supabaseClient.rpc(
        'crm_update_automation_rule',
        {
          p_rule_id: ruleId,
          p_org_id: orgId,
          p_name: body.name,
          p_description: body.description,
          p_is_enabled: body.is_enabled,
          p_event_type: body.event_type,
          p_time_offset_minutes: body.time_offset_minutes,
          p_segment_id: body.segment_id,
          p_conditions: body.conditions,
          p_actions: body.actions,
        }
      );

      if (updateError) {
        return new Response(
          JSON.stringify({ error: updateError.message }),
          { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
        );
      }

      return new Response(
        JSON.stringify(rule),
        { status: 200, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    } else if (req.method === 'GET') {
      // List rules
      const searchParams = url.searchParams;
      const isEnabled = searchParams.get('is_enabled');
      const triggerType = searchParams.get('trigger_type');
      
      const { data: rules, error: listError } = await supabaseClient.rpc(
        'crm_list_automation_rules',
        {
          p_org_id: orgId,
          p_is_enabled: isEnabled ? isEnabled === 'true' : null,
          p_trigger_type: triggerType || null,
        }
      );

      if (listError) {
        return new Response(
          JSON.stringify({ error: listError.message }),
          { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
        );
      }

      return new Response(
        JSON.stringify({ data: rules }),
        { status: 200, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    }
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
});

function validateCreateRuleRequest(body: any): string | null {
  if (!body.name || typeof body.name !== 'string' || body.name.trim().length === 0) {
    return 'name is required and must be a non-empty string';
  }
  
  if (body.name.length > 100) {
    return 'name must be 100 characters or less';
  }
  
  if (!body.trigger_type || !['event', 'time_based', 'segment_membership'].includes(body.trigger_type)) {
    return 'trigger_type is required and must be one of: event, time_based, segment_membership';
  }
  
  // Validate trigger-specific requirements
  if (body.trigger_type === 'event' && !body.event_type) {
    return 'event_type is required for event triggers';
  }
  
  if (body.trigger_type === 'time_based' && body.time_offset_minutes === undefined) {
    return 'time_offset_minutes is required for time_based triggers';
  }
  
  if (body.trigger_type === 'time_based' && (typeof body.time_offset_minutes !== 'number' || body.time_offset_minutes < 0)) {
    return 'time_offset_minutes must be a non-negative integer';
  }
  
  if (body.trigger_type === 'segment_membership' && !body.segment_id) {
    return 'segment_id is required for segment_membership triggers';
  }
  
  // Validate actions
  if (!body.actions || !Array.isArray(body.actions) || body.actions.length === 0) {
    return 'actions is required and must be a non-empty array';
  }
  
  for (const action of body.actions) {
    if (!action.type || !['create_followup', 'send_message', 'tag_customer'].includes(action.type)) {
      return `Invalid action type: ${action.type}`;
    }
    
    // Validate action-specific fields
    if (action.type === 'create_followup') {
      if (!action.followup_title) {
        return 'followup_title is required for create_followup actions';
      }
    }
    
    if (action.type === 'send_message') {
      if (!action.template_id && !action.channel) {
        return 'template_id or channel is required for send_message actions';
      }
    }
    
    if (action.type === 'tag_customer') {
      if (!action.tag_id && !action.tag_name) {
        return 'tag_id or tag_name is required for tag_customer actions';
      }
    }
  }
  
  return null;
}

function validateUpdateRuleRequest(body: any): string | null {
  // Similar validation but all fields optional
  if (body.name !== undefined) {
    if (typeof body.name !== 'string' || body.name.trim().length === 0) {
      return 'name must be a non-empty string';
    }
    if (body.name.length > 100) {
      return 'name must be 100 characters or less';
    }
  }
  
  if (body.time_offset_minutes !== undefined && body.time_offset_minutes < 0) {
    return 'time_offset_minutes must be a non-negative integer';
  }
  
  if (body.actions !== undefined) {
    if (!Array.isArray(body.actions) || body.actions.length === 0) {
      return 'actions must be a non-empty array';
    }
    
    for (const action of body.actions) {
      if (!action.type || !['create_followup', 'send_message', 'tag_customer'].includes(action.type)) {
        return `Invalid action type: ${action.type}`;
      }
    }
  }
  
  return null;
}
```

### 3.4 Implementation: PostgreSQL RPC Functions

#### 3.4.1 Create Automation Rule Function

```sql
CREATE OR REPLACE FUNCTION crm_create_automation_rule(
  p_org_id UUID,
  p_name TEXT,
  p_description TEXT DEFAULT NULL,
  p_trigger_type automation_trigger_type_enum,
  p_event_type TEXT DEFAULT NULL,
  p_time_offset_minutes INTEGER DEFAULT NULL,
  p_segment_id UUID DEFAULT NULL,
  p_conditions JSONB DEFAULT NULL,
  p_actions JSONB,
  p_is_enabled BOOLEAN DEFAULT true,
  p_created_by_user_id UUID DEFAULT NULL
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_user_id UUID;
  v_user_org_id UUID;
  v_rule_id UUID;
  v_result JSONB;
BEGIN
  v_user_id := COALESCE(p_created_by_user_id, auth.uid());
  
  SELECT org_id INTO v_user_org_id
  FROM profiles
  WHERE id = v_user_id AND is_active = true;
  
  IF v_user_org_id IS NULL THEN
    RAISE EXCEPTION 'User not authenticated or inactive';
  END IF;
  
  IF v_user_org_id != p_org_id THEN
    RAISE EXCEPTION 'Access denied: org_id mismatch';
  END IF;
  
  -- Check role
  IF NOT EXISTS (
    SELECT 1 FROM profiles
    WHERE id = v_user_id
    AND role IN ('admin', 'manager')
    AND is_active = true
  ) THEN
    RAISE EXCEPTION 'Only admins and managers can create automation rules';
  END IF;
  
  -- Validate name
  IF p_name IS NULL OR trim(p_name) = '' THEN
    RAISE EXCEPTION 'name is required and must be non-empty';
  END IF;
  
  IF length(p_name) > 100 THEN
    RAISE EXCEPTION 'name must be 100 characters or less';
  END IF;
  
  -- Validate trigger-specific requirements
  IF p_trigger_type = 'event' AND p_event_type IS NULL THEN
    RAISE EXCEPTION 'event_type is required for event triggers';
  END IF;
  
  IF p_trigger_type = 'time_based' AND p_time_offset_minutes IS NULL THEN
    RAISE EXCEPTION 'time_offset_minutes is required for time_based triggers';
  END IF;
  
  IF p_trigger_type = 'segment_membership' AND p_segment_id IS NULL THEN
    RAISE EXCEPTION 'segment_id is required for segment_membership triggers';
  END IF;
  
  -- Validate segment_id if provided
  IF p_segment_id IS NOT NULL THEN
    IF NOT EXISTS (
      SELECT 1 FROM crm_segments
      WHERE id = p_segment_id AND org_id = p_org_id
    ) THEN
      RAISE EXCEPTION 'Segment not found or does not belong to organization';
    END IF;
  END IF;
  
  -- Validate actions
  IF p_actions IS NULL OR jsonb_array_length(p_actions) = 0 THEN
    RAISE EXCEPTION 'actions is required and must be a non-empty array';
  END IF;
  
  -- Insert rule
  INSERT INTO crm_automation_rules (
    org_id,
    name,
    description,
    trigger_type,
    event_type,
    time_offset_minutes,
    segment_id,
    conditions,
    actions,
    is_enabled,
    created_by_user_id
  ) VALUES (
    p_org_id,
    p_name,
    NULLIF(trim(p_description), ''),
    p_trigger_type,
    p_event_type,
    p_time_offset_minutes,
    p_segment_id,
    p_conditions,
    p_actions,
    p_is_enabled,
    v_user_id
  )
  RETURNING id INTO v_rule_id;
  
  -- Return created rule
  SELECT jsonb_build_object(
    'id', car.id,
    'org_id', car.org_id,
    'name', car.name,
    'description', car.description,
    'trigger_type', car.trigger_type,
    'event_type', car.event_type,
    'time_offset_minutes', car.time_offset_minutes,
    'segment_id', car.segment_id,
    'conditions', car.conditions,
    'actions', car.actions,
    'is_enabled', car.is_enabled,
    'created_by_user_id', car.created_by_user_id,
    'created_at', car.created_at,
    'updated_at', car.updated_at
  )
  INTO v_result
  FROM crm_automation_rules car
  WHERE car.id = v_rule_id;
  
  RETURN v_result;
END;
$$;
```

#### 3.4.2 Update Automation Rule Function

```sql
CREATE OR REPLACE FUNCTION crm_update_automation_rule(
  p_rule_id UUID,
  p_org_id UUID,
  p_name TEXT DEFAULT NULL,
  p_description TEXT DEFAULT NULL,
  p_is_enabled BOOLEAN DEFAULT NULL,
  p_event_type TEXT DEFAULT NULL,
  p_time_offset_minutes INTEGER DEFAULT NULL,
  p_segment_id UUID DEFAULT NULL,
  p_conditions JSONB DEFAULT NULL,
  p_actions JSONB DEFAULT NULL
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_user_id UUID;
  v_user_org_id UUID;
  v_current_rule RECORD;
  v_result JSONB;
BEGIN
  v_user_id := auth.uid();
  
  SELECT org_id INTO v_user_org_id
  FROM profiles
  WHERE id = v_user_id AND is_active = true;
  
  IF v_user_org_id IS NULL THEN
    RAISE EXCEPTION 'User not authenticated or inactive';
  END IF;
  
  IF v_user_org_id != p_org_id THEN
    RAISE EXCEPTION 'Access denied';
  END IF;
  
  -- Check role
  IF NOT EXISTS (
    SELECT 1 FROM profiles
    WHERE id = v_user_id
    AND role IN ('admin', 'manager')
    AND is_active = true
  ) THEN
    RAISE EXCEPTION 'Only admins and managers can update automation rules';
  END IF;
  
  -- Get current rule
  SELECT * INTO v_current_rule
  FROM crm_automation_rules
  WHERE id = p_rule_id AND org_id = p_org_id;
  
  IF NOT FOUND THEN
    RAISE EXCEPTION 'Rule not found or access denied';
  END IF;
  
  -- Validate name if provided
  IF p_name IS NOT NULL THEN
    IF trim(p_name) = '' THEN
      RAISE EXCEPTION 'name must be non-empty';
    END IF;
    
    IF length(p_name) > 100 THEN
      RAISE EXCEPTION 'name must be 100 characters or less';
    END IF;
  END IF;
  
  -- Validate segment_id if provided
  IF p_segment_id IS NOT NULL THEN
    IF NOT EXISTS (
      SELECT 1 FROM crm_segments
      WHERE id = p_segment_id AND org_id = p_org_id
    ) THEN
      RAISE EXCEPTION 'Segment not found or does not belong to organization';
    END IF;
  END IF;
  
  -- Validate actions if provided
  IF p_actions IS NOT NULL THEN
    IF jsonb_array_length(p_actions) = 0 THEN
      RAISE EXCEPTION 'actions must be a non-empty array';
    END IF;
  END IF;
  
  -- Update rule
  UPDATE crm_automation_rules
  SET
    name = COALESCE(p_name, name),
    description = CASE WHEN p_description IS NOT NULL THEN NULLIF(trim(p_description), '') ELSE description END,
    is_enabled = COALESCE(p_is_enabled, is_enabled),
    event_type = COALESCE(p_event_type, event_type),
    time_offset_minutes = COALESCE(p_time_offset_minutes, time_offset_minutes),
    segment_id = COALESCE(p_segment_id, segment_id),
    conditions = COALESCE(p_conditions, conditions),
    actions = COALESCE(p_actions, actions),
    updated_at = now()
  WHERE id = p_rule_id;
  
  -- Return updated rule
  SELECT jsonb_build_object(
    'id', car.id,
    'org_id', car.org_id,
    'name', car.name,
    'description', car.description,
    'trigger_type', car.trigger_type,
    'event_type', car.event_type,
    'time_offset_minutes', car.time_offset_minutes,
    'segment_id', car.segment_id,
    'conditions', car.conditions,
    'actions', car.actions,
    'is_enabled', car.is_enabled,
    'created_by_user_id', car.created_by_user_id,
    'created_at', car.created_at,
    'updated_at', car.updated_at
  )
  INTO v_result
  FROM crm_automation_rules car
  WHERE car.id = p_rule_id;
  
  RETURN v_result;
END;
$$;
```

#### 3.4.3 List Automation Rules Function

```sql
CREATE OR REPLACE FUNCTION crm_list_automation_rules(
  p_org_id UUID,
  p_is_enabled BOOLEAN DEFAULT NULL,
  p_trigger_type automation_trigger_type_enum DEFAULT NULL
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_user_org_id UUID;
  v_result JSONB;
BEGIN
  SELECT org_id INTO v_user_org_id
  FROM profiles
  WHERE id = auth.uid() AND is_active = true;
  
  IF v_user_org_id IS NULL THEN
    RAISE EXCEPTION 'User not authenticated or inactive';
  END IF;
  
  IF v_user_org_id != p_org_id THEN
    RAISE EXCEPTION 'Access denied';
  END IF;
  
  -- Build result
  SELECT jsonb_agg(jsonb_build_object(
    'id', car.id,
    'org_id', car.org_id,
    'name', car.name,
    'description', car.description,
    'trigger_type', car.trigger_type,
    'event_type', car.event_type,
    'time_offset_minutes', car.time_offset_minutes,
    'segment_id', car.segment_id,
    'conditions', car.conditions,
    'actions', car.actions,
    'is_enabled', car.is_enabled,
    'created_by_user_id', car.created_by_user_id,
    'created_at', car.created_at,
    'updated_at', car.updated_at
  ) ORDER BY car.created_at DESC)
  INTO v_result
  FROM crm_automation_rules car
  WHERE car.org_id = p_org_id
  AND (p_is_enabled IS NULL OR car.is_enabled = p_is_enabled)
  AND (p_trigger_type IS NULL OR car.trigger_type = p_trigger_type);
  
  RETURN COALESCE(v_result, '[]'::jsonb);
END;
$$;
```

### 3.5 Response Schema

#### 3.5.1 Success Response (201 Created / 200 OK)

```typescript
interface AutomationRuleResponse {
  id: string; // UUID
  org_id: string; // UUID
  name: string;
  description?: string;
  trigger_type: 'event' | 'time_based' | 'segment_membership';
  event_type?: string;
  time_offset_minutes?: number;
  segment_id?: string; // UUID
  conditions?: {
    operator: 'AND' | 'OR';
    rules: Array<{
      field: string;
      operator: string;
      value: any;
    }>;
  };
  actions: Array<{
    type: string;
    [key: string]: any;
  }>;
  is_enabled: boolean;
  created_by_user_id?: string; // UUID
  created_at: string; // ISO 8601 timestamp
  updated_at: string; // ISO 8601 timestamp
}
```

---

## 4. Story CRM-030: Event-Triggered Automation Handler

### 4.1 Endpoint Specification

**Path**: `POST /crm/automation/handle-event`

**Method**: `POST`

**Authentication**: Service Role Key (called by other modules)

**Content-Type**: `application/json`

### 4.2 Event Payload Schema

```typescript
interface EventTriggerPayload {
  org_id: string; // UUID
  event_type: string; // e.g., 'work_order_completed', 'quote_sent', 'customer_created'
  event_id?: string; // UUID, optional unique event identifier
  customer_id: string; // UUID
  event_data: {
    [key: string]: any; // Event-specific data
    // Examples:
    // work_order_id?: string;
    // work_order_type?: string;
    // quote_id?: string;
    // quote_amount?: number;
  };
  occurred_at?: string; // ISO 8601 timestamp, default: now()
}
```

### 4.3 Implementation: Edge Function

**File**: `supabase/functions/crm-automation-handle-event/index.ts`

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

serve(async (req) => {
  try {
    // Use service role for internal calls
    const supabaseAdmin = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? '',
      {
        auth: {
          autoRefreshToken: false,
          persistSession: false
        }
      }
    );
    
    const payload: EventTriggerPayload = await req.json();
    
    // Validate payload
    if (!payload.org_id || !payload.event_type || !payload.customer_id) {
      return new Response(
        JSON.stringify({ error: 'org_id, event_type, and customer_id are required' }),
        { status: 400, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Get active rules matching event type
    const { data: rules, error: rulesError } = await supabaseAdmin
      .from('crm_automation_rules')
      .select('*')
      .eq('org_id', payload.org_id)
      .eq('trigger_type', 'event')
      .eq('event_type', payload.event_type)
      .eq('is_enabled', true);
    
    if (rulesError) {
      return new Response(
        JSON.stringify({ error: rulesError.message }),
        { status: 500, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    if (!rules || rules.length === 0) {
      return new Response(
        JSON.stringify({ message: 'No matching automation rules found' }),
        { status: 200, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    const results = [];
    
    // Process each matching rule
    for (const rule of rules) {
      try {
        // Evaluate conditions if present
        if (rule.conditions) {
          const conditionsMet = await evaluateConditions(
            supabaseAdmin,
            payload.org_id,
            payload.customer_id,
            rule.conditions
        );
          
          if (!conditionsMet) {
            continue; // Skip this rule
          }
        }
        
        // Generate idempotency key
        const idempotencyKey = `${rule.id}-${payload.customer_id}-${payload.event_id || 'no-id'}-${payload.occurred_at || new Date().toISOString()}`;
        
        // Check if already processed (idempotency check)
        const { data: existingRun } = await supabaseAdmin
          .from('crm_automation_runs')
          .select('id')
          .eq('org_id', payload.org_id)
          .eq('rule_id', rule.id)
          .eq('customer_id', payload.customer_id)
          .contains('trigger_context', { idempotency_key: idempotencyKey })
          .single();
        
        if (existingRun) {
          results.push({
            rule_id: rule.id,
            status: 'skipped',
            reason: 'already_processed'
          });
          continue;
        }
        
        // Create automation run record
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
              idempotency_key: idempotencyKey
            },
            status: 'pending',
            started_at: new Date().toISOString()
          })
          .select()
          .single();
        
        if (runError) {
          results.push({
            rule_id: rule.id,
            status: 'failed',
            error: runError.message
          });
          continue;
        }
        
        // Execute actions
        const actionResults = await executeActions(
          supabaseAdmin,
          payload.org_id,
          payload.customer_id,
          rule.actions,
          payload.event_data,
          payload.occurred_at || new Date().toISOString()
        );
        
        // Update run status
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
            completed_at: new Date().toISOString()
          })
          .eq('id', run.id);
        
        results.push({
          rule_id: rule.id,
          run_id: run.id,
          status: finalStatus,
          actions: actionResults
        });
      } catch (error) {
        results.push({
          rule_id: rule.id,
          status: 'failed',
          error: error.message
        });
      }
    }
    
    return new Response(
      JSON.stringify({ results }),
      { status: 200, headers: { 'Content-Type': 'application/json' } }
    );
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { 'Content-Type': 'application/json' } }
    );
  }
});

async function evaluateConditions(
  supabase: any,
  orgId: string,
  customerId: string,
  conditions: any
): Promise<boolean> {
  // Get customer data
  const { data: customer } = await supabase
    .from('customers')
    .select('*')
    .eq('id', customerId)
    .eq('org_id', orgId)
    .single();
  
  if (!customer) {
    return false;
  }
  
  // Evaluate each rule
  const ruleResults = [];
  for (const rule of conditions.rules) {
    const result = evaluateRule(customer, rule);
    ruleResults.push(result);
  }
  
  // Combine results based on operator
  if (conditions.operator === 'AND') {
    return ruleResults.every(r => r === true);
  } else {
    return ruleResults.some(r => r === true);
  }
}

function evaluateRule(customer: any, rule: any): boolean {
  const fieldValue = customer[rule.field];
  
  switch (rule.operator) {
    case 'equals':
      return fieldValue === rule.value;
    case 'not_equals':
      return fieldValue !== rule.value;
    case 'in':
      return Array.isArray(rule.value) && rule.value.includes(fieldValue);
    case 'not_in':
      return Array.isArray(rule.value) && !rule.value.includes(fieldValue);
    case 'greater_than':
      return fieldValue > rule.value;
    case 'less_than':
      return fieldValue < rule.value;
    case 'is_null':
      return fieldValue === null || fieldValue === undefined;
    case 'is_not_null':
      return fieldValue !== null && fieldValue !== undefined;
    default:
      return false;
  }
}

async function executeActions(
  supabase: any,
  orgId: string,
  customerId: string,
  actions: any[],
  eventData: any,
  occurredAt: string
): Promise<Array<{ type: string; status: string; error?: string }>> {
  const results = [];
  
  for (const action of actions) {
    try {
      if (action.type === 'create_followup') {
        const dueAt = new Date(occurredAt);
        dueAt.setMinutes(dueAt.getMinutes() + (action.followup_due_at_offset_minutes || 0));
        
        // Replace template variables in title/description
        const title = replaceTemplateVariables(action.followup_title, customerId, eventData);
        const description = replaceTemplateVariables(action.followup_description || '', customerId, eventData);
        
        const { error } = await supabase.rpc('crm_create_followup', {
          p_org_id: orgId,
          p_customer_id: customerId,
          p_title: title,
          p_description: description,
          p_due_at: dueAt.toISOString(),
          p_priority: action.followup_priority || 'medium',
          p_origin: 'system_rule',
        });
        
        if (error) {
          results.push({ type: action.type, status: 'failed', error: error.message });
        } else {
          results.push({ type: action.type, status: 'success' });
        }
      } else if (action.type === 'tag_customer') {
        // Find or create tag
        let tagId = action.tag_id;
        
        if (!tagId && action.tag_name) {
          const { data: existingTag } = await supabase
            .from('crm_tags')
            .select('id')
            .eq('org_id', orgId)
            .eq('name', action.tag_name)
            .single();
          
          if (existingTag) {
            tagId = existingTag.id;
          } else {
            const { data: newTag } = await supabase
              .from('crm_tags')
              .insert({
                org_id: orgId,
                name: action.tag_name
              })
              .select()
              .single();
            
            tagId = newTag.id;
          }
        }
        
        if (tagId) {
          const { error } = await supabase
            .from('crm_customer_tags')
            .insert({
              org_id: orgId,
              customer_id: customerId,
              tag_id: tagId
            });
          
          if (error && !error.message.includes('duplicate')) {
            results.push({ type: action.type, status: 'failed', error: error.message });
          } else {
            results.push({ type: action.type, status: 'success' });
          }
        } else {
          results.push({ type: action.type, status: 'failed', error: 'Tag ID or name required' });
        }
      } else if (action.type === 'send_message') {
        // Deferred to messaging integration
        results.push({ type: action.type, status: 'skipped', error: 'Message sending not yet implemented' });
      } else {
        results.push({ type: action.type, status: 'failed', error: `Unknown action type: ${action.type}` });
      }
    } catch (error) {
      results.push({ type: action.type, status: 'failed', error: error.message });
    }
  }
  
  return results;
}

function replaceTemplateVariables(template: string, customerId: string, eventData: any): string {
  // Simple template variable replacement
  // {{customer.field}} and {{event.field}} patterns
  let result = template;
  
  // Replace event variables
  for (const [key, value] of Object.entries(eventData)) {
    result = result.replace(new RegExp(`\\{\\{event\\.${key}\\}\\}`, 'g'), String(value));
  }
  
  // Customer variables would need customer data fetched separately
  // For now, return as-is if customer variables are present
  return result;
}
```

---

## 5. Story CRM-031: Time-Based Automation Processor

### 5.1 Function Specification

**Path**: `POST /crm/automation/process-time-based` (called by Supabase Cron)

**Method**: `POST`

**Authentication**: Service Role Key (internal cron job)

**Schedule**: Every 5 minutes (configurable)

### 5.2 Implementation: Edge Function

**File**: `supabase/functions/crm-automation-process-time-based/index.ts`

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

serve(async (req) => {
  try {
    const supabaseAdmin = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? '',
      {
        auth: {
          autoRefreshToken: false,
          persistSession: false
        }
      }
    );
    
    const now = new Date();
    const results = [];
    
    // Get all active time_based rules
    const { data: rules, error: rulesError } = await supabaseAdmin
      .from('crm_automation_rules')
      .select('*')
      .eq('trigger_type', 'time_based')
      .eq('is_enabled', true);
    
    if (rulesError) {
      return new Response(
        JSON.stringify({ error: rulesError.message }),
        { status: 500, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    if (!rules || rules.length === 0) {
      return new Response(
        JSON.stringify({ message: 'No time-based rules found' }),
        { status: 200, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Process each rule
    for (const rule of rules) {
      try {
        // Find events that are due for this rule
        // This requires a tracking mechanism - for now, we'll use a simplified approach
        // In production, you'd track events in a separate table or use interaction/work_order timestamps
        
        // Example: Find interactions that occurred time_offset_minutes ago (within a 5-minute window)
        const windowStart = new Date(now.getTime() - (rule.time_offset_minutes + 5) * 60 * 1000);
        const windowEnd = new Date(now.getTime() - (rule.time_offset_minutes - 5) * 60 * 1000);
        
        // Query for events (this is simplified - actual implementation depends on event tracking)
        const { data: events, error: eventsError } = await supabaseAdmin
          .from('crm_interactions')
          .select('customer_id, occurred_at, id')
          .gte('occurred_at', windowStart.toISOString())
          .lte('occurred_at', windowEnd.toISOString())
          .eq('org_id', rule.org_id);
        
        if (eventsError) {
          results.push({
            rule_id: rule.id,
            status: 'failed',
            error: eventsError.message
          });
          continue;
        }
        
        if (!events || events.length === 0) {
          continue; // No events due
        }
        
        // Process each event
        for (const event of events) {
          // Generate idempotency key
          const idempotencyKey = `${rule.id}-${event.customer_id}-${event.id}-${Math.floor(event.occurred_at.getTime() / (5 * 60 * 1000))}`;
          
          // Check if already processed
          const { data: existingRun } = await supabaseAdmin
            .from('crm_automation_runs')
            .select('id')
            .eq('org_id', rule.org_id)
            .eq('rule_id', rule.id)
            .eq('customer_id', event.customer_id)
            .contains('trigger_context', { idempotency_key: idempotencyKey })
            .single();
          
          if (existingRun) {
            continue; // Already processed
          }
          
          // Evaluate conditions if present
          if (rule.conditions) {
            const conditionsMet = await evaluateConditions(
              supabaseAdmin,
              rule.org_id,
              event.customer_id,
              rule.conditions
            );
            
            if (!conditionsMet) {
              continue; // Skip this customer
            }
          }
          
          // Create automation run
          const { data: run } = await supabaseAdmin
            .from('crm_automation_runs')
            .insert({
              org_id: rule.org_id,
              rule_id: rule.id,
              customer_id: event.customer_id,
              trigger_context: {
                event_id: event.id,
                event_occurred_at: event.occurred_at,
                time_offset_minutes: rule.time_offset_minutes,
                idempotency_key: idempotencyKey
              },
              status: 'pending',
              started_at: now.toISOString()
            })
            .select()
            .single();
          
          // Execute actions
          const actionResults = await executeActions(
            supabaseAdmin,
            rule.org_id,
            event.customer_id,
            rule.actions,
            { interaction_id: event.id },
            event.occurred_at
          );
          
          // Update run status
          const hasFailures = actionResults.some(r => r.status === 'failed');
          const finalStatus = hasFailures ? 'partial_success' : 'success';
          
          await supabaseAdmin
            .from('crm_automation_runs')
            .update({
              status: finalStatus,
              completed_at: now.toISOString()
            })
            .eq('id', run.id);
          
          results.push({
            rule_id: rule.id,
            customer_id: event.customer_id,
            status: finalStatus
          });
        }
      } catch (error) {
        results.push({
          rule_id: rule.id,
          status: 'failed',
          error: error.message
        });
      }
    }
    
    return new Response(
      JSON.stringify({ 
        processed_at: now.toISOString(),
        results 
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

### 5.3 Supabase Cron Configuration

**File**: `supabase/migrations/YYYYMMDDHHMMSS_create_cron_job_time_based_automations.sql`

```sql
-- Create cron job for time-based automations
-- Note: Supabase cron syntax may vary - this is a placeholder

-- Example cron job (runs every 5 minutes)
SELECT cron.schedule(
  'process-time-based-automations',
  '*/5 * * * *', -- Every 5 minutes
  $$
  SELECT net.http_post(
    url := 'https://<project-ref>.supabase.co/functions/v1/crm-automation-process-time-based',
    headers := jsonb_build_object(
      'Content-Type', 'application/json',
      'Authorization', 'Bearer ' || current_setting('app.settings.service_role_key')
    ),
    body := '{}'::jsonb
  );
  $$
);
```

**Note**: Actual Supabase cron implementation may use pg_cron extension or Supabase's scheduled Edge Functions feature.

---

## 6. Story CRM-032: Segment Membership Automation Processor

### 6.1 Function Specification

**Path**: `POST /crm/automation/process-segment-membership` (called by Supabase Cron)

**Method**: `POST`

**Authentication**: Service Role Key (internal cron job)

**Schedule**: Every 15 minutes (configurable)

### 6.2 Implementation: Edge Function

**File**: `supabase/functions/crm-automation-process-segment-membership/index.ts`

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

serve(async (req) => {
  try {
    const supabaseAdmin = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? '',
      {
        auth: {
          autoRefreshToken: false,
          persistSession: false
        }
      }
    );
    
    const now = new Date();
    const today = now.toISOString().split('T')[0]; // YYYY-MM-DD
    const results = [];
    
    // Get all active segment_membership rules
    const { data: rules, error: rulesError } = await supabaseAdmin
      .from('crm_automation_rules')
      .select('*')
      .eq('trigger_type', 'segment_membership')
      .eq('is_enabled', true);
    
    if (rulesError) {
      return new Response(
        JSON.stringify({ error: rulesError.message }),
        { status: 500, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    if (!rules || rules.length === 0) {
      return new Response(
        JSON.stringify({ message: 'No segment_membership rules found' }),
        { status: 200, headers: { 'Content-Type': 'application/json' } }
      );
    }
    
    // Process each rule
    for (const rule of rules) {
      try {
        if (!rule.segment_id) {
          continue; // Skip invalid rules
        }
        
        // Get segment members
        const { data: members, error: membersError } = await supabaseAdmin
          .from('crm_segment_members')
          .select('customer_id')
          .eq('segment_id', rule.segment_id)
          .eq('org_id', rule.org_id);
        
        if (membersError) {
          results.push({
            rule_id: rule.id,
            status: 'failed',
            error: membersError.message
          });
          continue;
        }
        
        if (!members || members.length === 0) {
          continue; // No members in segment
        }
        
        // Limit processing per run (safeguard)
        const maxCustomersPerRun = 100;
        const customersToProcess = members.slice(0, maxCustomersPerRun).map(m => m.customer_id);
        
        // Process each customer
        for (const customerId of customersToProcess) {
          // Generate idempotency key (one run per day per customer per rule)
          const idempotencyKey = `${rule.id}-${customerId}-${today}`;
          
          // Check if already processed today
          const { data: existingRun } = await supabaseAdmin
            .from('crm_automation_runs')
            .select('id')
            .eq('org_id', rule.org_id)
            .eq('rule_id', rule.id)
            .eq('customer_id', customerId)
            .contains('trigger_context', { idempotency_key: idempotencyKey })
            .single();
          
          if (existingRun) {
            continue; // Already processed today
          }
          
          // Evaluate conditions if present
          if (rule.conditions) {
            const conditionsMet = await evaluateConditions(
              supabaseAdmin,
              rule.org_id,
              customerId,
              rule.conditions
            );
            
            if (!conditionsMet) {
              continue; // Skip this customer
            }
          }
          
          // Create automation run
          const { data: run } = await supabaseAdmin
            .from('crm_automation_runs')
            .insert({
              org_id: rule.org_id,
              rule_id: rule.id,
              customer_id: customerId,
              trigger_context: {
                segment_id: rule.segment_id,
                run_date: today,
                idempotency_key: idempotencyKey
              },
              status: 'pending',
              started_at: now.toISOString()
            })
            .select()
            .single();
          
          // Execute actions
          const actionResults = await executeActions(
            supabaseAdmin,
            rule.org_id,
            customerId,
            rule.actions,
            { segment_id: rule.segment_id },
            now.toISOString()
          );
          
          // Update run status
          const hasFailures = actionResults.some(r => r.status === 'failed');
          const finalStatus = hasFailures ? 'partial_success' : 'success';
          
          await supabaseAdmin
            .from('crm_automation_runs')
            .update({
              status: finalStatus,
              completed_at: now.toISOString()
            })
            .eq('id', run.id);
          
          results.push({
            rule_id: rule.id,
            customer_id: customerId,
            status: finalStatus
          });
        }
      } catch (error) {
        results.push({
          rule_id: rule.id,
          status: 'failed',
          error: error.message
        });
      }
    }
    
    return new Response(
      JSON.stringify({ 
        processed_at: now.toISOString(),
        results 
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

---

## 7. Action Execution Logic

### 7.1 Create Follow-Up Action

**Implementation Details**:
- Uses `crm_create_followup` RPC function from Epic 5
- Supports template variables in title/description
- Calculates `due_at` based on `followup_due_at_offset_minutes`
- Sets `origin = 'system_rule'`

**Template Variables**:
- `{{event.field}}` - Event data fields
- `{{customer.field}}` - Customer fields (requires customer data fetch)

### 7.2 Tag Customer Action

**Implementation Details**:
- Finds existing tag by ID or name
- Creates tag if doesn't exist (when using `tag_name`)
- Links tag to customer via `crm_customer_tags`
- Handles duplicate tag assignments gracefully

### 7.3 Send Message Action

**Implementation Details**:
- Deferred to messaging integration module
- Interface defined for future implementation
- Requires message template and channel selection

---

## 8. Conditions Evaluation

### 8.1 Condition Schema

Same as rule definition schema from Epic 6, but simpler:
- Supports customer fields only (not related entities or computed fields)
- Operators: `equals`, `not_equals`, `in`, `not_in`, `greater_than`, `less_than`, `is_null`, `is_not_null`

### 8.2 Evaluation Logic

- Fetches customer data
- Evaluates each rule condition
- Combines results using `AND` or `OR` operator
- Returns boolean result

---

## 9. Error Handling

### 9.1 Automation Run Status

- `pending`: Run created but not yet executed
- `success`: All actions executed successfully
- `partial_success`: Some actions succeeded, some failed
- `failed`: All actions failed or error occurred
- `skipped`: Run skipped due to conditions or idempotency

### 9.2 Error Logging

- All errors stored in `crm_automation_runs.error_message`
- Action-level errors included in run context
- Failed runs can be retried manually

---

## 10. Performance Considerations

### 10.1 Processing Limits

- Segment membership: Max 100 customers per run (configurable)
- Time-based: Process events in 5-minute windows
- Event-based: Process immediately (no batching)

### 10.2 Performance Targets

- Event trigger: < 2 seconds per rule
- Time-based processor: < 30 seconds per run
- Segment processor: < 60 seconds per run (for 100 customers)

---

## 11. Testing Requirements

### 11.1 Unit Tests

- Rule validation logic
- Condition evaluation
- Action execution
- Idempotency checks

### 11.2 Integration Tests

- End-to-end event trigger flow
- Time-based automation scheduling
- Segment membership automation
- Error handling and retry logic

---

## 12. Implementation Checklist

### Story CRM-029: Manage Automation Rules
- [ ] Edge Functions implemented (POST, PATCH, GET)
- [ ] RPC functions implemented (create, update, list)
- [ ] Request validation implemented
- [ ] Trigger-specific field validation
- [ ] Actions validation
- [ ] Role-based authorization
- [ ] Error handling
- [ ] Tests written
- [ ] API documentation with examples

### Story CRM-030: Event-Triggered Automation Handler
- [ ] Edge Function implemented
- [ ] Event payload validation
- [ ] Rule lookup by event_type
- [ ] Conditions evaluation
- [ ] Action execution
- [ ] Idempotency checks
- [ ] Error logging
- [ ] Tests written
- [ ] Integration documentation

### Story CRM-031: Time-Based Automation Processor
- [ ] Edge Function implemented
- [ ] Cron job configured
- [ ] Event window calculation
- [ ] Idempotency strategy
- [ ] Action execution
- [ ] Error handling
- [ ] Tests written
- [ ] Scheduling documentation

### Story CRM-032: Segment Membership Automation Processor
- [ ] Edge Function implemented
- [ ] Cron job configured
- [ ] Segment member lookup
- [ ] Processing limits
- [ ] Daily idempotency
- [ ] Action execution
- [ ] Error handling
- [ ] Tests written
- [ ] Scheduling documentation

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 7 – Automation Engine. All specifications are designed to be directly consumable by LLM-based code generators, with exact request/response schemas, validation rules, SQL functions, Edge Function implementations, action execution logic, conditions evaluation, and idempotency strategies defined.

