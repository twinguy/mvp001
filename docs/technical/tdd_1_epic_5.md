# Technical Design Document – Epic 5: Follow-Ups & Reminders

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 5 – Follow-Ups & Reminders
- **Source**: Derived from `fdd_1_agile.md` Epic 5 (Stories CRM-022 through CRM-024)
- **Reference Documents**: 
  - `fdd_1.md` (Functional Design Document for CRM, §4.4)
  - `fdd_1_agile.md` (Agile User Stories)
  - `tdd_1_epic_1.md` (CRM Core Data Model - prerequisite)
  - `tdd_1_epic_2.md` (Authentication, Authorization & RLS Policies - prerequisite)
  - `tdd_1_epic_3.md` (Customer Management APIs - prerequisite)
  - `tdd_1_epic_4.md` (Interaction & Communication Logging - prerequisite)
  - `functional.md` (Platform Capabilities)
  - `tooling.md` (Platform Choices: Supabase + Vercel)
- **Target Platform**: Supabase (PostgreSQL 15+ with Edge Functions)
- **Purpose**: Comprehensive technical specification for LLM-based code generation
- **Prerequisites**: Epic 1 (CRM Core Data Model), Epic 2 (RLS Policies), Epic 3 (Customer Management APIs), and Epic 4 (Interaction Logging) must be completed first

---

## 1. Overview

This document provides complete technical specifications for implementing follow-ups and reminders APIs in Supabase. It covers:

- Create Manual Follow-Up API for scheduling customer follow-ups
- List & Filter Follow-Ups API for dashboard views and workload management
- Complete/Cancel Follow-Up API for status transitions and completion tracking
- Request/response schemas with exact JSON structures
- Validation rules and error handling
- Status transition validation and business rules
- Authorization rules (who can modify which follow-ups)
- Performance optimizations and query strategies

All specifications are designed to be directly implementable as Supabase Edge Functions (Deno/TypeScript) or PostgreSQL RPC functions, with exact schemas, validation rules, and error codes defined.

---

## 2. Architecture Decisions

### 2.1 Implementation Approach

**Decision**: Support both Edge Functions and PostgreSQL RPC functions:

- **Edge Functions** (`/crm/followups`): Recommended for complex workflows, reminder scheduling, and better error handling
- **RPC Functions** (`crm_create_followup()`, `crm_list_followups()`, `crm_update_followup()`): Alternative for direct database access, better performance for simple operations

**Rationale**: 
- Edge Functions provide better HTTP semantics, error handling, and integration with external services (reminders)
- RPC functions provide lower latency and simpler deployment for database-only operations
- Frontend can choose based on use case

### 2.2 Authentication & Authorization

- All endpoints require authenticated Supabase user (JWT token)
- `org_id` is automatically derived from user's profile (via RLS helper functions from Epic 2)
- RLS policies enforce org-scoping automatically
- Role-based access control enforced via RLS (from Epic 2)
- Authorization rules for modifying follow-ups:
  - Assignee can modify their own follow-ups
  - Managers/admins can modify any follow-up in their org
  - Technicians can only read their own assigned follow-ups (from Epic 2)

### 2.3 Status Transition Rules

**Valid Status Transitions**:

| From Status | To Status | Allowed By | Notes |
|-------------|----------|------------|-------|
| `pending` | `completed` | Assignee, Manager, Admin | Normal completion |
| `pending` | `canceled` | Assignee, Manager, Admin | Cancellation |
| `pending` | `expired` | System only | Auto-expired by scheduled job |
| `completed` | `pending` | Admin only | Reopening (rare) |
| `completed` | `canceled` | Admin only | Marking completed as canceled (rare) |
| `canceled` | `pending` | Admin only | Reactivation (rare) |
| `expired` | `pending` | Manager, Admin | Reactivation after expiration |
| `expired` | `completed` | Manager, Admin | Mark expired as completed |

**Business Rules**:
- Once `completed` or `canceled`, follow-up cannot be modified by assignee (only admin can reopen)
- `expired` status is set automatically by scheduled job when `due_at` passes
- Status transitions are validated in application logic and database constraints

### 2.4 Reminder Scheduling

- Reminder scheduling is optional and deferred to future implementation
- Edge Functions can integrate with external reminder services (email, SMS, push notifications)
- Reminder logic can be implemented as separate Edge Function or scheduled job

---

## 3. Story CRM-022: Create Manual Follow-Up API

### 3.1 Endpoint Specification

#### 3.1.1 Edge Function Endpoint

**Path**: `POST /crm/followups`

**Method**: `POST`

**Authentication**: Required (Supabase JWT)

**Content-Type**: `application/json`

#### 3.1.2 RPC Function Alternative

**Function Name**: `crm_create_followup`

**Schema**: `public`

**Parameters**: JSONB input parameter

### 3.2 Request Schema

#### 3.2.1 Request Body Structure

```typescript
interface CreateFollowUpRequest {
  // Required fields
  customer_id: string; // UUID
  title: string; // Max 255 characters
  due_at: string; // ISO 8601 timestamp
  
  // Optional fields
  description?: string; // Max 5000 characters
  priority?: 'low' | 'medium' | 'high'; // Default: 'medium'
  assigned_to_user_id?: string; // UUID, defaults to creator if not provided
  related_interaction_id?: string; // UUID, optional link to interaction
  related_work_order_id?: string; // UUID, optional link to work order (future module)
  origin?: 'manual' | 'system_rule' | 'ai_recommendation'; // Default: 'manual'
  
  // Options
  send_reminder?: boolean; // Default: false, future feature
  reminder_minutes_before?: number; // Default: 15, future feature
}
```

#### 3.2.2 Request Validation Rules

**Required Fields**:
- `customer_id` must be valid UUID and exist in `customers` table for user's org
- `title` must be non-empty string (max 255 characters)
- `due_at` must be valid ISO 8601 timestamp

**Format Validation**:
- `title` max length: 255 characters
- `description` max length: 5000 characters
- `due_at` must be valid ISO 8601 timestamp
- `due_at` cannot be more than 10 years in the future (prevents unrealistic dates)
- `due_at` cannot be more than 1 year in the past (prevents backdating)
- `assigned_to_user_id` must be valid UUID and exist in `profiles` table for user's org
- `related_interaction_id` must be valid UUID and exist in `crm_interactions` table for user's org
- `priority` must be one of: `'low'`, `'medium'`, `'high'`
- `origin` must be one of: `'manual'`, `'system_rule'`, `'ai_recommendation'`

**Business Rules**:
- `customer_id` must belong to the authenticated user's organization (enforced via RLS)
- `assigned_to_user_id` must belong to the same org as the creator
- If `assigned_to_user_id` is not provided, defaults to `created_by_user_id`
- `origin` defaults to `'manual'` for manual creation
- `status` is always set to `'pending'` on creation
- `completed_at` is always `NULL` on creation

### 3.3 Implementation: Edge Function

#### 3.3.1 Edge Function Structure

**File**: `supabase/functions/crm-followups/index.ts`

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
  // Handle CORS preflight
  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: corsHeaders });
  }

  try {
    // Initialize Supabase client
    const supabaseClient = createClient(
      Deno.env.get('SUPABASE_URL') ?? '',
      Deno.env.get('SUPABASE_ANON_KEY') ?? '',
      {
        global: {
          headers: { Authorization: req.headers.get('Authorization')! },
        },
      }
    );

    // Get authenticated user
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

    // Get user's org_id from profile
    const { data: profile, error: profileError } = await supabaseClient
      .from('profiles')
      .select('org_id, role')
      .eq('id', user.id)
      .single();

    if (profileError || !profile) {
      return new Response(
        JSON.stringify({ error: 'User profile not found' }),
        { status: 403, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    }

    const orgId = profile.org_id;

    // Parse request body
    const body = await req.json();

    // Validate request
    const validationError = validateCreateFollowUpRequest(body, orgId);
    if (validationError) {
      return new Response(
        JSON.stringify({ error: validationError }),
        { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    }

    // Set defaults
    const assignedToUserId = body.assigned_to_user_id || user.id;
    const priority = body.priority || 'medium';
    const origin = body.origin || 'manual';

    // Verify assigned user belongs to same org
    if (body.assigned_to_user_id) {
      const { data: assignedUser, error: assignedUserError } = await supabaseClient
        .from('profiles')
        .select('org_id')
        .eq('id', assignedToUserId)
        .single();

      if (assignedUserError || !assignedUser || assignedUser.org_id !== orgId) {
        return new Response(
          JSON.stringify({ error: 'Assigned user not found or does not belong to your organization' }),
          { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
        );
      }
    }

    // Call RPC function
    const { data: followup, error: createError } = await supabaseClient.rpc(
      'crm_create_followup',
      {
        p_org_id: orgId,
        p_customer_id: body.customer_id,
        p_title: body.title,
        p_description: body.description || null,
        p_due_at: body.due_at,
        p_priority: priority,
        p_assigned_to_user_id: assignedToUserId,
        p_origin: origin,
        p_related_interaction_id: body.related_interaction_id || null,
        p_related_work_order_id: body.related_work_order_id || null,
        p_created_by_user_id: user.id,
      }
    );

    if (createError) {
      return new Response(
        JSON.stringify({ error: createError.message }),
        { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    }

    // TODO: Schedule reminder if requested (future feature)
    // if (body.send_reminder) {
    //   await scheduleReminder(followup.id, body.reminder_minutes_before || 15);
    // }

    return new Response(
      JSON.stringify(followup),
      { status: 201, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
});

function validateCreateFollowUpRequest(body: any, orgId: string): string | null {
  // Required fields
  if (!body.customer_id || typeof body.customer_id !== 'string') {
    return 'customer_id is required and must be a valid UUID';
  }
  
  const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
  if (!uuidRegex.test(body.customer_id)) {
    return 'customer_id must be a valid UUID';
  }
  
  if (!body.title || typeof body.title !== 'string' || body.title.trim().length === 0) {
    return 'title is required and must be a non-empty string';
  }
  
  if (body.title.length > 255) {
    return 'title must be 255 characters or less';
  }
  
  if (!body.due_at || typeof body.due_at !== 'string') {
    return 'due_at is required and must be a valid ISO 8601 timestamp';
  }
  
  // Validate due_at timestamp
  const dueAt = new Date(body.due_at);
  if (isNaN(dueAt.getTime())) {
    return 'due_at must be a valid ISO 8601 timestamp';
  }
  
  const now = new Date();
  const tenYearsFromNow = new Date(now.getTime() + 10 * 365 * 24 * 60 * 60 * 1000);
  const oneYearAgo = new Date(now.getTime() - 365 * 24 * 60 * 60 * 1000);
  
  if (dueAt > tenYearsFromNow) {
    return 'due_at cannot be more than 10 years in the future';
  }
  
  if (dueAt < oneYearAgo) {
    return 'due_at cannot be more than 1 year in the past';
  }
  
  // Validate description length
  if (body.description && body.description.length > 5000) {
    return 'description must be 5000 characters or less';
  }
  
  // Validate priority
  if (body.priority && !['low', 'medium', 'high'].includes(body.priority)) {
    return 'priority must be one of: low, medium, high';
  }
  
  // Validate origin
  if (body.origin && !['manual', 'system_rule', 'ai_recommendation'].includes(body.origin)) {
    return 'origin must be one of: manual, system_rule, ai_recommendation';
  }
  
  // Validate UUIDs if provided
  if (body.assigned_to_user_id && !uuidRegex.test(body.assigned_to_user_id)) {
    return 'assigned_to_user_id must be a valid UUID';
  }
  
  if (body.related_interaction_id && !uuidRegex.test(body.related_interaction_id)) {
    return 'related_interaction_id must be a valid UUID';
  }
  
  if (body.related_work_order_id && !uuidRegex.test(body.related_work_order_id)) {
    return 'related_work_order_id must be a valid UUID';
  }
  
  return null; // Validation passed
}
```

### 3.4 Implementation: PostgreSQL RPC Function

#### 3.4.1 RPC Function Definition

**Function Name**: `crm_create_followup`

**Parameters**:
- `p_org_id UUID` - Organization ID
- `p_customer_id UUID` - Customer ID
- `p_title TEXT` - Follow-up title
- `p_description TEXT` - Follow-up description (nullable)
- `p_due_at TIMESTAMPTZ` - Due date/time
- `p_priority followup_priority_enum` - Priority level
- `p_assigned_to_user_id UUID` - Assigned user ID
- `p_origin followup_origin_enum` - Origin type
- `p_related_interaction_id UUID` - Related interaction ID (nullable)
- `p_related_work_order_id UUID` - Related work order ID (nullable)
- `p_created_by_user_id UUID` - Creator user ID

**Return Type**: `JSONB` - Created follow-up record

**DDL**:

```sql
CREATE OR REPLACE FUNCTION crm_create_followup(
  p_org_id UUID,
  p_customer_id UUID,
  p_title TEXT,
  p_description TEXT DEFAULT NULL,
  p_due_at TIMESTAMPTZ,
  p_priority followup_priority_enum DEFAULT 'medium',
  p_assigned_to_user_id UUID DEFAULT NULL,
  p_origin followup_origin_enum DEFAULT 'manual',
  p_related_interaction_id UUID DEFAULT NULL,
  p_related_work_order_id UUID DEFAULT NULL,
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
  v_assigned_user_id UUID;
  v_followup_id UUID;
  v_result JSONB;
BEGIN
  -- Get authenticated user ID
  v_user_id := COALESCE(p_created_by_user_id, auth.uid());
  
  -- Get user's org_id
  SELECT org_id INTO v_user_org_id
  FROM profiles
  WHERE id = v_user_id AND is_active = true;
  
  IF v_user_org_id IS NULL THEN
    RAISE EXCEPTION 'User not authenticated or inactive';
  END IF;
  
  -- Verify org_id matches
  IF v_user_org_id != p_org_id THEN
    RAISE EXCEPTION 'Access denied: org_id mismatch';
  END IF;
  
  -- Verify customer exists and belongs to org
  IF NOT EXISTS (
    SELECT 1 FROM customers
    WHERE id = p_customer_id AND org_id = p_org_id
  ) THEN
    RAISE EXCEPTION 'Customer not found or access denied';
  END IF;
  
  -- Validate title
  IF p_title IS NULL OR trim(p_title) = '' THEN
    RAISE EXCEPTION 'title is required and must be non-empty';
  END IF;
  
  IF length(p_title) > 255 THEN
    RAISE EXCEPTION 'title must be 255 characters or less';
  END IF;
  
  -- Validate description length
  IF p_description IS NOT NULL AND length(p_description) > 5000 THEN
    RAISE EXCEPTION 'description must be 5000 characters or less';
  END IF;
  
  -- Validate due_at
  IF p_due_at IS NULL THEN
    RAISE EXCEPTION 'due_at is required';
  END IF;
  
  IF p_due_at > now() + interval '10 years' THEN
    RAISE EXCEPTION 'due_at cannot be more than 10 years in the future';
  END IF;
  
  IF p_due_at < now() - interval '1 year' THEN
    RAISE EXCEPTION 'due_at cannot be more than 1 year in the past';
  END IF;
  
  -- Set assigned user (default to creator if not provided)
  v_assigned_user_id := COALESCE(p_assigned_to_user_id, v_user_id);
  
  -- Verify assigned user belongs to same org
  IF NOT EXISTS (
    SELECT 1 FROM profiles
    WHERE id = v_assigned_user_id
    AND org_id = p_org_id
    AND is_active = true
  ) THEN
    RAISE EXCEPTION 'Assigned user not found or does not belong to organization';
  END IF;
  
  -- Validate related_interaction_id if provided
  IF p_related_interaction_id IS NOT NULL THEN
    IF NOT EXISTS (
      SELECT 1 FROM crm_interactions
      WHERE id = p_related_interaction_id
      AND org_id = p_org_id
      AND customer_id = p_customer_id
    ) THEN
      RAISE EXCEPTION 'Related interaction not found or does not belong to customer';
    END IF;
  END IF;
  
  -- Insert follow-up
  INSERT INTO crm_followups (
    org_id,
    customer_id,
    assigned_to_user_id,
    title,
    description,
    due_at,
    status,
    priority,
    origin,
    related_interaction_id,
    related_work_order_id,
    created_by_user_id
  ) VALUES (
    p_org_id,
    p_customer_id,
    v_assigned_user_id,
    p_title,
    NULLIF(trim(p_description), ''),
    p_due_at,
    'pending',
    p_priority,
    p_origin,
    p_related_interaction_id,
    p_related_work_order_id,
    v_user_id
  )
  RETURNING id INTO v_followup_id;
  
  -- Return created follow-up
  SELECT jsonb_build_object(
    'id', cf.id,
    'org_id', cf.org_id,
    'customer_id', cf.customer_id,
    'assigned_to_user_id', cf.assigned_to_user_id,
    'title', cf.title,
    'description', cf.description,
    'due_at', cf.due_at,
    'status', cf.status,
    'priority', cf.priority,
    'origin', cf.origin,
    'related_interaction_id', cf.related_interaction_id,
    'related_work_order_id', cf.related_work_order_id,
    'created_by_user_id', cf.created_by_user_id,
    'completed_at', cf.completed_at,
    'completion_notes', cf.completion_notes,
    'created_at', cf.created_at,
    'updated_at', cf.updated_at
  )
  INTO v_result
  FROM crm_followups cf
  WHERE cf.id = v_followup_id;
  
  RETURN v_result;
  
EXCEPTION
  WHEN OTHERS THEN
    RAISE EXCEPTION 'Error creating follow-up: %', SQLERRM;
END;
$$;
```

### 3.5 Response Schema

#### 3.5.1 Success Response (201 Created)

```typescript
interface CreateFollowUpResponse {
  id: string; // UUID
  org_id: string; // UUID
  customer_id: string; // UUID
  assigned_to_user_id: string; // UUID
  title: string;
  description?: string;
  due_at: string; // ISO 8601 timestamp
  status: 'pending';
  priority: 'low' | 'medium' | 'high';
  origin: 'manual' | 'system_rule' | 'ai_recommendation';
  related_interaction_id?: string; // UUID
  related_work_order_id?: string; // UUID
  created_by_user_id: string; // UUID
  completed_at: null;
  completion_notes: null;
  created_at: string; // ISO 8601 timestamp
  updated_at: string; // ISO 8601 timestamp
}
```

#### 3.5.2 Error Responses

**400 Bad Request** (Validation Error):
```json
{
  "error": "title is required and must be a non-empty string"
}
```

**401 Unauthorized**:
```json
{
  "error": "Unauthorized"
}
```

**403 Forbidden**:
```json
{
  "error": "User profile not found"
}
```

**500 Internal Server Error**:
```json
{
  "error": "Error creating follow-up: <database error message>"
}
```

---

## 4. Story CRM-023: List & Filter Follow-Ups API

### 4.1 Endpoint Specification

**Path**: `GET /crm/followups`

**Method**: `GET`

**Authentication**: Required (Supabase JWT)

**Query Parameters**:
- `assigned_to_user_id` (optional) - Filter by assigned user (UUID)
- `customer_id` (optional) - Filter by customer (UUID)
- `status` (optional) - Filter by status: `pending`, `completed`, `canceled`, `expired` (can be repeated for multiple)
- `priority` (optional) - Filter by priority: `low`, `medium`, `high` (can be repeated)
- `origin` (optional) - Filter by origin: `manual`, `system_rule`, `ai_recommendation` (can be repeated)
- `start_date` (optional) - ISO 8601 timestamp, filter follow-ups due from this date onwards
- `end_date` (optional) - ISO 8601 timestamp, filter follow-ups due up to this date
- `overdue_only` (optional, default: `false`) - Boolean, show only overdue follow-ups (`due_at < now()` and `status = 'pending'`)
- `limit` (optional, default: 20, max: 100) - Number of results per page
- `offset` (optional, default: 0) - Pagination offset
- `sort_by` (optional, default: `due_at`) - Sort field: `due_at`, `created_at`, `priority`
- `sort_order` (optional, default: `asc`) - Sort order: `asc`, `desc`

### 4.2 Response Schema

```typescript
interface ListFollowUpsResponse {
  data: Array<{
    id: string; // UUID
    customer_id: string; // UUID
    customer: {
      id: string;
      name: string;
      type: 'individual' | 'company';
      email?: string;
      phone?: string;
    };
    assigned_to_user_id?: string; // UUID
    assigned_to_user?: {
      id: string;
      first_name?: string;
      last_name?: string;
      email?: string;
    };
    title: string;
    description?: string;
    due_at: string; // ISO 8601 timestamp
    status: 'pending' | 'completed' | 'canceled' | 'expired';
    priority: 'low' | 'medium' | 'high';
    origin: 'manual' | 'system_rule' | 'ai_recommendation';
    related_interaction_id?: string; // UUID
    created_by_user_id?: string; // UUID
    completed_at?: string; // ISO 8601 timestamp
    completion_notes?: string;
    created_at: string; // ISO 8601 timestamp
    updated_at: string; // ISO 8601 timestamp
    is_overdue: boolean; // Computed: due_at < now() AND status = 'pending'
  }>;
  total: number;
  limit: number;
  offset: number;
  has_more: boolean;
}
```

### 4.3 Implementation: Edge Function

**File**: `supabase/functions/crm-followups/index.ts` (add GET handler)

```typescript
// Add to existing Edge Function

if (req.method === 'GET') {
  const url = new URL(req.url);
  const searchParams = url.searchParams;
  
  // Parse query parameters
  const assignedToUserId = searchParams.get('assigned_to_user_id');
  const customerId = searchParams.get('customer_id');
  const statuses = searchParams.getAll('status');
  const priorities = searchParams.getAll('priority');
  const origins = searchParams.getAll('origin');
  const startDate = searchParams.get('start_date');
  const endDate = searchParams.get('end_date');
  const overdueOnly = searchParams.get('overdue_only') === 'true';
  const limit = Math.min(parseInt(searchParams.get('limit') || '20'), 100);
  const offset = parseInt(searchParams.get('offset') || '0');
  const sortBy = searchParams.get('sort_by') || 'due_at';
  const sortOrder = searchParams.get('sort_order') || 'asc';
  
  // Get user's org_id
  const { data: profile } = await supabaseClient
    .from('profiles')
    .select('org_id, role')
    .eq('id', user.id)
    .single();
  
  // Role-based filtering: technicians only see their own follow-ups
  let effectiveAssignedToUserId = assignedToUserId;
  if (profile.role === 'technician') {
    effectiveAssignedToUserId = user.id; // Force filter to own follow-ups
  }
  
  // Call RPC function
  const { data: result, error: fetchError } = await supabaseClient.rpc(
    'crm_list_followups',
    {
      p_org_id: profile.org_id,
      p_assigned_to_user_id: effectiveAssignedToUserId || null,
      p_customer_id: customerId || null,
      p_statuses: statuses.length > 0 ? statuses : null,
      p_priorities: priorities.length > 0 ? priorities : null,
      p_origins: origins.length > 0 ? origins : null,
      p_start_date: startDate || null,
      p_end_date: endDate || null,
      p_overdue_only: overdueOnly,
      p_limit: limit,
      p_offset: offset,
      p_sort_by: sortBy,
      p_sort_order: sortOrder,
    }
  );
  
  if (fetchError) {
    return new Response(
      JSON.stringify({ error: fetchError.message }),
      { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
  
  return new Response(
    JSON.stringify(result),
    { status: 200, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
  );
}
```

### 4.4 Implementation: PostgreSQL RPC Function

```sql
CREATE OR REPLACE FUNCTION crm_list_followups(
  p_org_id UUID,
  p_assigned_to_user_id UUID DEFAULT NULL,
  p_customer_id UUID DEFAULT NULL,
  p_statuses TEXT[] DEFAULT NULL,
  p_priorities TEXT[] DEFAULT NULL,
  p_origins TEXT[] DEFAULT NULL,
  p_start_date TIMESTAMPTZ DEFAULT NULL,
  p_end_date TIMESTAMPTZ DEFAULT NULL,
  p_overdue_only BOOLEAN DEFAULT false,
  p_limit INTEGER DEFAULT 20,
  p_offset INTEGER DEFAULT 0,
  p_sort_by TEXT DEFAULT 'due_at',
  p_sort_order TEXT DEFAULT 'asc'
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_user_org_id UUID;
  v_result JSONB;
  v_total INTEGER;
  v_sort_expr TEXT;
BEGIN
  -- Get user's org_id
  SELECT org_id INTO v_user_org_id
  FROM profiles
  WHERE id = auth.uid() AND is_active = true;
  
  IF v_user_org_id IS NULL THEN
    RAISE EXCEPTION 'User not authenticated or inactive';
  END IF;
  
  -- Verify org_id matches
  IF v_user_org_id != p_org_id THEN
    RAISE EXCEPTION 'Access denied';
  END IF;
  
  -- Validate and set limits
  IF p_limit > 100 THEN
    p_limit := 100;
  END IF;
  IF p_limit < 1 THEN
    p_limit := 20;
  END IF;
  
  -- Validate sort_by
  IF p_sort_by NOT IN ('due_at', 'created_at', 'priority') THEN
    p_sort_by := 'due_at';
  END IF;
  
  -- Validate sort_order
  IF p_sort_order NOT IN ('asc', 'desc') THEN
    p_sort_order := 'asc';
  END IF;
  
  -- Build sort expression
  v_sort_expr := p_sort_by || ' ' || UPPER(p_sort_order);
  
  -- Get total count
  SELECT COUNT(*) INTO v_total
  FROM crm_followups cf
  WHERE cf.org_id = p_org_id
  AND (p_assigned_to_user_id IS NULL OR cf.assigned_to_user_id = p_assigned_to_user_id)
  AND (p_customer_id IS NULL OR cf.customer_id = p_customer_id)
  AND (p_statuses IS NULL OR cf.status::TEXT = ANY(p_statuses))
  AND (p_priorities IS NULL OR cf.priority::TEXT = ANY(p_priorities))
  AND (p_origins IS NULL OR cf.origin::TEXT = ANY(p_origins))
  AND (p_start_date IS NULL OR cf.due_at >= p_start_date)
  AND (p_end_date IS NULL OR cf.due_at <= p_end_date)
  AND (
    p_overdue_only = false OR
    (cf.due_at < now() AND cf.status = 'pending')
  );
  
  -- Build result
  SELECT jsonb_build_object(
    'data', COALESCE(
      (
        SELECT jsonb_agg(jsonb_build_object(
          'id', cf.id,
          'customer_id', cf.customer_id,
          'customer', jsonb_build_object(
            'id', c.id,
            'name', c.name,
            'type', c.type,
            'email', c.email,
            'phone', c.phone
          ),
          'assigned_to_user_id', cf.assigned_to_user_id,
          'assigned_to_user', CASE
            WHEN cf.assigned_to_user_id IS NOT NULL THEN jsonb_build_object(
              'id', p.id,
              'first_name', p.first_name,
              'last_name', p.last_name,
              'email', p.email
            )
            ELSE NULL
          END,
          'title', cf.title,
          'description', cf.description,
          'due_at', cf.due_at,
          'status', cf.status,
          'priority', cf.priority,
          'origin', cf.origin,
          'related_interaction_id', cf.related_interaction_id,
          'created_by_user_id', cf.created_by_user_id,
          'completed_at', cf.completed_at,
          'completion_notes', cf.completion_notes,
          'created_at', cf.created_at,
          'updated_at', cf.updated_at,
          'is_overdue', (cf.due_at < now() AND cf.status = 'pending')
        ) ORDER BY
          CASE WHEN v_sort_expr LIKE 'due_at%' THEN cf.due_at END,
          CASE WHEN v_sort_expr LIKE 'created_at%' THEN cf.created_at END,
          CASE WHEN v_sort_expr LIKE 'priority%' THEN 
            CASE cf.priority
              WHEN 'high' THEN 1
              WHEN 'medium' THEN 2
              WHEN 'low' THEN 3
            END
          END
        )
        FROM crm_followups cf
        JOIN customers c ON c.id = cf.customer_id
        LEFT JOIN profiles p ON p.id = cf.assigned_to_user_id
        WHERE cf.org_id = p_org_id
        AND (p_assigned_to_user_id IS NULL OR cf.assigned_to_user_id = p_assigned_to_user_id)
        AND (p_customer_id IS NULL OR cf.customer_id = p_customer_id)
        AND (p_statuses IS NULL OR cf.status::TEXT = ANY(p_statuses))
        AND (p_priorities IS NULL OR cf.priority::TEXT = ANY(p_priorities))
        AND (p_origins IS NULL OR cf.origin::TEXT = ANY(p_origins))
        AND (p_start_date IS NULL OR cf.due_at >= p_start_date)
        AND (p_end_date IS NULL OR cf.due_at <= p_end_date)
        AND (
          p_overdue_only = false OR
          (cf.due_at < now() AND cf.status = 'pending')
        )
        ORDER BY
          CASE WHEN v_sort_expr LIKE 'due_at DESC' THEN cf.due_at END DESC,
          CASE WHEN v_sort_expr LIKE 'due_at ASC' THEN cf.due_at END ASC,
          CASE WHEN v_sort_expr LIKE 'created_at DESC' THEN cf.created_at END DESC,
          CASE WHEN v_sort_expr LIKE 'created_at ASC' THEN cf.created_at END ASC,
          CASE WHEN v_sort_expr LIKE 'priority DESC' THEN 
            CASE cf.priority
              WHEN 'high' THEN 1
              WHEN 'medium' THEN 2
              WHEN 'low' THEN 3
            END
          END DESC,
          CASE WHEN v_sort_expr LIKE 'priority ASC' THEN 
            CASE cf.priority
              WHEN 'high' THEN 1
              WHEN 'medium' THEN 2
              WHEN 'low' THEN 3
            END
          END ASC
        LIMIT p_limit
        OFFSET p_offset
      ),
      '[]'::jsonb
    ),
    'total', v_total,
    'limit', p_limit,
    'offset', p_offset,
    'has_more', (p_offset + p_limit) < v_total
  )
  INTO v_result;
  
  RETURN v_result;
END;
$$;
```

---

## 5. Story CRM-024: Complete/Cancel Follow-Up API

### 5.1 Endpoint Specification

**Path**: `PATCH /crm/followups/:id`

**Method**: `PATCH`

**Authentication**: Required (Supabase JWT)

**Content-Type**: `application/json`

### 5.2 Request Schema

```typescript
interface UpdateFollowUpRequest {
  status?: 'pending' | 'completed' | 'canceled' | 'expired';
  completion_notes?: string; // Max 2000 characters, required when status = 'completed'
  description?: string; // Max 5000 characters
  priority?: 'low' | 'medium' | 'high';
  due_at?: string; // ISO 8601 timestamp
  assigned_to_user_id?: string; // UUID
}
```

### 5.3 Status Transition Validation

**Business Rules**:
- Only assignee, manager, or admin can modify follow-ups
- Status transitions must follow valid transition rules (see §2.3)
- `completion_notes` is required when transitioning to `completed`
- `completed_at` is automatically set when status changes to `completed` (via database trigger)
- `completed_at` is cleared when status changes away from `completed`

### 5.4 Implementation: Edge Function

**File**: `supabase/functions/crm-followups/index.ts` (add PATCH handler)

```typescript
// Add to existing Edge Function

if (req.method === 'PATCH') {
  const url = new URL(req.url);
  const pathParts = url.pathname.split('/');
  const followupId = pathParts[pathParts.length - 1];
  
  if (!followupId) {
    return new Response(
      JSON.stringify({ error: 'Follow-up ID is required' }),
      { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
  
  const body = await req.json();
  
  // Validate request
  const validationError = validateUpdateFollowUpRequest(body);
  if (validationError) {
    return new Response(
      JSON.stringify({ error: validationError }),
      { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
  
  // Get user's org_id and role
  const { data: profile } = await supabaseClient
    .from('profiles')
    .select('org_id, role')
    .eq('id', user.id)
    .single();
  
  // Get current follow-up to check authorization and current status
  const { data: currentFollowup, error: fetchError } = await supabaseClient
    .from('crm_followups')
    .select('*')
    .eq('id', followupId)
    .eq('org_id', profile.org_id)
    .single();
  
  if (fetchError || !currentFollowup) {
    return new Response(
      JSON.stringify({ error: 'Follow-up not found or access denied' }),
      { status: 404, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
  
  // Check authorization
  const isAssignee = currentFollowup.assigned_to_user_id === user.id;
  const isManagerOrAdmin = ['admin', 'manager'].includes(profile.role);
  
  if (!isAssignee && !isManagerOrAdmin) {
    return new Response(
      JSON.stringify({ error: 'You do not have permission to modify this follow-up' }),
      { status: 403, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
  
  // Validate status transition
  if (body.status && body.status !== currentFollowup.status) {
    const transitionError = validateStatusTransition(
      currentFollowup.status,
      body.status,
      profile.role
    );
    
    if (transitionError) {
      return new Response(
        JSON.stringify({ error: transitionError }),
        { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    }
  }
  
  // Validate completion_notes if completing
  if (body.status === 'completed' && !body.completion_notes) {
    return new Response(
      JSON.stringify({ error: 'completion_notes is required when status is completed' }),
      { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
  
  // Call RPC function
  const { data: followup, error: updateError } = await supabaseClient.rpc(
    'crm_update_followup',
    {
      p_followup_id: followupId,
      p_org_id: profile.org_id,
      p_status: body.status || null,
      p_completion_notes: body.completion_notes || null,
      p_description: body.description || null,
      p_priority: body.priority || null,
      p_due_at: body.due_at || null,
      p_assigned_to_user_id: body.assigned_to_user_id || null,
    }
  );
  
  if (updateError) {
    return new Response(
      JSON.stringify({ error: updateError.message }),
      { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
  
  return new Response(
    JSON.stringify(followup),
    { status: 200, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
  );
}

function validateUpdateFollowUpRequest(body: any): string | null {
  // Validate completion_notes length
  if (body.completion_notes && body.completion_notes.length > 2000) {
    return 'completion_notes must be 2000 characters or less';
  }
  
  // Validate description length
  if (body.description && body.description.length > 5000) {
    return 'description must be 5000 characters or less';
  }
  
  // Validate priority
  if (body.priority && !['low', 'medium', 'high'].includes(body.priority)) {
    return 'priority must be one of: low, medium, high';
  }
  
  // Validate status
  if (body.status && !['pending', 'completed', 'canceled', 'expired'].includes(body.status)) {
    return 'status must be one of: pending, completed, canceled, expired';
  }
  
  // Validate due_at if provided
  if (body.due_at) {
    const dueAt = new Date(body.due_at);
    if (isNaN(dueAt.getTime())) {
      return 'due_at must be a valid ISO 8601 timestamp';
    }
  }
  
  return null;
}

function validateStatusTransition(
  fromStatus: string,
  toStatus: string,
  userRole: string
): string | null {
  // Define valid transitions
  const validTransitions: Record<string, string[]> = {
    'pending': ['completed', 'canceled'],
    'completed': ['pending'], // Admin only
    'canceled': ['pending'], // Admin only
    'expired': ['pending', 'completed'], // Manager/Admin only
  };
  
  // Check if transition is valid
  const allowedStatuses = validTransitions[fromStatus] || [];
  if (!allowedStatuses.includes(toStatus)) {
    return `Invalid status transition from ${fromStatus} to ${toStatus}`;
  }
  
  // Check role-based restrictions
  if (fromStatus === 'completed' && toStatus === 'pending' && userRole !== 'admin') {
    return 'Only admins can reopen completed follow-ups';
  }
  
  if (fromStatus === 'canceled' && toStatus === 'pending' && userRole !== 'admin') {
    return 'Only admins can reactivate canceled follow-ups';
  }
  
  if (fromStatus === 'expired' && !['admin', 'manager'].includes(userRole)) {
    return 'Only managers and admins can modify expired follow-ups';
  }
  
  return null; // Transition is valid
}
```

### 5.5 Implementation: PostgreSQL RPC Function

```sql
CREATE OR REPLACE FUNCTION crm_update_followup(
  p_followup_id UUID,
  p_org_id UUID,
  p_status followup_status_enum DEFAULT NULL,
  p_completion_notes TEXT DEFAULT NULL,
  p_description TEXT DEFAULT NULL,
  p_priority followup_priority_enum DEFAULT NULL,
  p_due_at TIMESTAMPTZ DEFAULT NULL,
  p_assigned_to_user_id UUID DEFAULT NULL
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_user_id UUID;
  v_user_org_id UUID;
  v_user_role TEXT;
  v_current_followup RECORD;
  v_result JSONB;
BEGIN
  -- Get authenticated user ID
  v_user_id := auth.uid();
  
  -- Get user's org_id and role
  SELECT org_id, role INTO v_user_org_id, v_user_role
  FROM profiles
  WHERE id = v_user_id AND is_active = true;
  
  IF v_user_org_id IS NULL THEN
    RAISE EXCEPTION 'User not authenticated or inactive';
  END IF;
  
  -- Verify org_id matches
  IF v_user_org_id != p_org_id THEN
    RAISE EXCEPTION 'Access denied';
  END IF;
  
  -- Get current follow-up
  SELECT * INTO v_current_followup
  FROM crm_followups
  WHERE id = p_followup_id AND org_id = p_org_id;
  
  IF NOT FOUND THEN
    RAISE EXCEPTION 'Follow-up not found or access denied';
  END IF;
  
  -- Check authorization
  IF v_current_followup.assigned_to_user_id != v_user_id 
     AND v_user_role NOT IN ('admin', 'manager') THEN
    RAISE EXCEPTION 'You do not have permission to modify this follow-up';
  END IF;
  
  -- Validate status transition if status is being changed
  IF p_status IS NOT NULL AND p_status != v_current_followup.status THEN
    -- Validate transition (simplified, full validation in application layer)
    IF v_current_followup.status = 'completed' AND p_status != 'pending' AND v_user_role != 'admin' THEN
      RAISE EXCEPTION 'Only admins can modify completed follow-ups';
    END IF;
    
    IF v_current_followup.status = 'canceled' AND p_status != 'pending' AND v_user_role != 'admin' THEN
      RAISE EXCEPTION 'Only admins can reactivate canceled follow-ups';
    END IF;
    
    IF v_current_followup.status = 'expired' AND v_user_role NOT IN ('admin', 'manager') THEN
      RAISE EXCEPTION 'Only managers and admins can modify expired follow-ups';
    END IF;
  END IF;
  
  -- Validate completion_notes if completing
  IF p_status = 'completed' AND p_completion_notes IS NULL THEN
    RAISE EXCEPTION 'completion_notes is required when status is completed';
  END IF;
  
  -- Validate description length
  IF p_description IS NOT NULL AND length(p_description) > 5000 THEN
    RAISE EXCEPTION 'description must be 5000 characters or less';
  END IF;
  
  -- Validate completion_notes length
  IF p_completion_notes IS NOT NULL AND length(p_completion_notes) > 2000 THEN
    RAISE EXCEPTION 'completion_notes must be 2000 characters or less';
  END IF;
  
  -- Validate assigned_to_user_id if provided
  IF p_assigned_to_user_id IS NOT NULL THEN
    IF NOT EXISTS (
      SELECT 1 FROM profiles
      WHERE id = p_assigned_to_user_id
      AND org_id = p_org_id
      AND is_active = true
    ) THEN
      RAISE EXCEPTION 'Assigned user not found or does not belong to organization';
    END IF;
  END IF;
  
  -- Update follow-up
  UPDATE crm_followups
  SET
    status = COALESCE(p_status, status),
    completion_notes = CASE 
      WHEN p_status = 'completed' THEN COALESCE(p_completion_notes, completion_notes)
      WHEN p_status IS NOT NULL AND p_status != 'completed' THEN NULL
      ELSE completion_notes
    END,
    description = COALESCE(p_description, description),
    priority = COALESCE(p_priority, priority),
    due_at = COALESCE(p_due_at, due_at),
    assigned_to_user_id = COALESCE(p_assigned_to_user_id, assigned_to_user_id),
    updated_at = now()
  WHERE id = p_followup_id;
  
  -- Note: completed_at is set automatically by trigger when status changes to 'completed'
  
  -- Return updated follow-up
  SELECT jsonb_build_object(
    'id', cf.id,
    'org_id', cf.org_id,
    'customer_id', cf.customer_id,
    'assigned_to_user_id', cf.assigned_to_user_id,
    'title', cf.title,
    'description', cf.description,
    'due_at', cf.due_at,
    'status', cf.status,
    'priority', cf.priority,
    'origin', cf.origin,
    'related_interaction_id', cf.related_interaction_id,
    'created_by_user_id', cf.created_by_user_id,
    'completed_at', cf.completed_at,
    'completion_notes', cf.completion_notes,
    'created_at', cf.created_at,
    'updated_at', cf.updated_at
  )
  INTO v_result
  FROM crm_followups cf
  WHERE cf.id = p_followup_id;
  
  RETURN v_result;
  
EXCEPTION
  WHEN OTHERS THEN
    RAISE EXCEPTION 'Error updating follow-up: %', SQLERRM;
END;
$$;
```

### 5.6 Response Schema

**Success Response (200 OK)**: Same structure as CreateFollowUpResponse

**Error Responses**: Same as Create Follow-Up API, plus:

**403 Forbidden** (Authorization Error):
```json
{
  "error": "You do not have permission to modify this follow-up"
}
```

**400 Bad Request** (Status Transition Error):
```json
{
  "error": "Invalid status transition from completed to canceled"
}
```

---

## 6. Error Handling

### 6.1 Standard Error Response Format

All APIs return errors in this format:

```typescript
interface ErrorResponse {
  error: string; // Human-readable error message
  code?: string; // Optional error code (e.g., 'VALIDATION_ERROR', 'NOT_FOUND')
  details?: any; // Optional additional error details
}
```

### 6.2 HTTP Status Codes

- **200 OK**: Successful GET/PATCH request
- **201 Created**: Successful POST request
- **400 Bad Request**: Validation error or invalid input
- **401 Unauthorized**: Missing or invalid authentication token
- **403 Forbidden**: User lacks permission (RLS blocked or authorization failed)
- **404 Not Found**: Resource not found
- **500 Internal Server Error**: Server error

### 6.3 Error Codes

- `VALIDATION_ERROR`: Request validation failed
- `NOT_FOUND`: Resource not found
- `UNAUTHORIZED`: Authentication required
- `FORBIDDEN`: Insufficient permissions
- `INVALID_STATUS_TRANSITION`: Status transition not allowed
- `DATABASE_ERROR`: Database operation failed
- `CUSTOMER_NOT_FOUND`: Customer does not exist or access denied
- `USER_NOT_FOUND`: Assigned user does not exist or access denied

---

## 7. Performance Considerations

### 7.1 Query Optimization

- Use indexes defined in Epic 1 for all filter and sort operations
- Index on `(org_id, due_at)` optimizes dashboard queries
- Index on `(assigned_to_user_id, due_at)` optimizes user-specific queries
- Partial index on `(org_id, status, due_at) WHERE status = 'pending'` optimizes active follow-ups
- Limit result sets (default 20, max 100)
- Use pagination for large datasets

### 7.2 Caching Strategy

- Follow-up lists should not be cached (real-time data)
- Use Supabase real-time subscriptions for live updates
- Consider materialized views for aggregated follow-up statistics (future)

### 7.3 Performance Targets

- Create Follow-Up: < 200ms
- List Follow-Ups: < 300ms (for 20 follow-ups with filters)
- Update Follow-Up: < 200ms

---

## 8. Testing Requirements

### 8.1 Unit Tests

- Request validation functions
- Status transition validation logic
- Authorization checks
- Error handling paths
- Edge cases (null values, empty strings, etc.)

### 8.2 Integration Tests

- End-to-end API calls
- RLS enforcement verification
- Status transition flows
- Authorization scenarios (assignee, manager, admin, technician)
- Pagination and filtering

### 8.3 Performance Tests

- Load testing with realistic follow-up volumes
- Query performance validation
- Concurrent request handling

---

## 9. Implementation Checklist

### Story CRM-022: Create Manual Follow-Up
- [ ] Edge Function implemented (`POST /crm/followups`)
- [ ] RPC function implemented (`crm_create_followup`)
- [ ] Request validation implemented
- [ ] Default assignment to creator if not specified
- [ ] Assigned user validation (belongs to same org)
- [ ] Customer validation (belongs to user's org)
- [ ] Related interaction validation
- [ ] Timestamp validation (not too far in future/past)
- [ ] Error handling and rollback
- [ ] Response schema matches specification
- [ ] Tests written (happy path and error cases)
- [ ] API documentation with examples

### Story CRM-023: List & Filter Follow-Ups
- [ ] Edge Function or RPC function implemented (`GET /crm/followups`)
- [ ] Pagination support (limit/offset)
- [ ] Filter support (assigned_to_user_id, customer_id, status, priority, origin, date range, overdue_only)
- [ ] Sort support (due_at, created_at, priority)
- [ ] Customer information included in response
- [ ] Assigned user information included in response
- [ ] Role-based filtering (technicians only see own)
- [ ] Performance validated
- [ ] Response schema matches specification
- [ ] Tests written
- [ ] API documentation with examples

### Story CRM-024: Complete/Cancel Follow-Up
- [ ] Edge Function implemented (`PATCH /crm/followups/:id`)
- [ ] RPC function implemented (`crm_update_followup`)
- [ ] Status transition validation
- [ ] Authorization checks (assignee, manager, admin)
- [ ] Completion notes validation (required when completing)
- [ ] Automatic `completed_at` setting (via trigger)
- [ ] Error handling for invalid transitions
- [ ] Response schema matches specification
- [ ] Tests written (all status transitions)
- [ ] API documentation with examples

---

## 10. Future Enhancements

### 10.1 Reminder Scheduling

- Email reminders before due date
- SMS reminders for high-priority follow-ups
- Push notifications for mobile apps
- Configurable reminder timing

### 10.2 Follow-Up Templates

- Pre-defined follow-up templates
- Quick creation from templates
- Template management UI

### 10.3 Follow-Up Analytics

- Completion rate metrics
- Average time to completion
- Overdue follow-up trends
- Priority distribution
- User workload distribution

### 10.4 Bulk Operations

- Bulk status updates
- Bulk assignment changes
- Bulk deletion (admin only)

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 5 – Follow-Ups & Reminders. All specifications are designed to be directly consumable by LLM-based code generators, with exact request/response schemas, validation rules, SQL functions, Edge Function implementations, status transition logic, and authorization patterns defined.

