# Technical Design Document – Epic 6: Segmentation & Targeting

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 6 – Segmentation & Targeting
- **Source**: Derived from `fdd_1_agile.md` Epic 6 (Stories CRM-025 through CRM-028)
- **Reference Documents**: 
  - `fdd_1.md` (Functional Design Document for CRM, §4.5)
  - `fdd_1_agile.md` (Agile User Stories)
  - `tdd_1_epic_1.md` (CRM Core Data Model - prerequisite)
  - `tdd_1_epic_2.md` (Authentication, Authorization & RLS Policies - prerequisite)
  - `tdd_1_epic_3.md` (Customer Management APIs - prerequisite)
  - `functional.md` (Platform Capabilities)
  - `tooling.md` (Platform Choices: Supabase + Vercel)
- **Target Platform**: Supabase (PostgreSQL 15+ with Edge Functions)
- **Purpose**: Comprehensive technical specification for LLM-based code generation
- **Prerequisites**: Epic 1 (CRM Core Data Model), Epic 2 (RLS Policies), and Epic 3 (Customer Management APIs) must be completed first

---

## 1. Overview

This document provides complete technical specifications for implementing segmentation and targeting APIs in Supabase. It covers:

- Create & Update Segments API for managing segment definitions
- Rule-Based Segment Computation logic for evaluating customer criteria
- Get Segment Members API for retrieving segment membership
- Recompute Segment Members API for refreshing segment data
- Rule definition schema and validation
- AI-generated segment integration patterns (interface defined, implementation deferred to Epic 8)
- Request/response schemas with exact JSON structures
- Validation rules and error handling
- Performance optimizations and query strategies

All specifications are designed to be directly implementable as Supabase Edge Functions (Deno/TypeScript) or PostgreSQL RPC functions, with exact schemas, validation rules, and error codes defined.

---

## 2. Architecture Decisions

### 2.1 Implementation Approach

**Decision**: Support both Edge Functions and PostgreSQL RPC functions:

- **Edge Functions** (`/crm/segments`): Recommended for AI integration, complex workflows, and better error handling
- **RPC Functions** (`crm_create_segment()`, `crm_compute_segment_members()`, etc.): Alternative for direct database access, better performance for rule-based computation

**Rationale**: 
- Edge Functions provide better HTTP semantics, error handling, and integration with external AI services
- RPC functions provide lower latency and better performance for SQL-based rule evaluation
- Rule-based computation can be done efficiently in PostgreSQL
- AI-generated segments require Edge Functions for external API calls

### 2.2 Authentication & Authorization

- All endpoints require authenticated Supabase user (JWT token)
- `org_id` is automatically derived from user's profile (via RLS helper functions from Epic 2)
- RLS policies enforce org-scoping automatically
- Role-based access control enforced via RLS (from Epic 2):
  - Only `admin` and `manager` roles can create/update segments
  - All roles except `technician` can read segments
  - Only `admin` and `manager` can trigger recomputation

### 2.3 Segment Types

**Three segment types supported**:

1. **`static`**: Manually managed membership (members added/removed manually)
2. **`rule_based`**: Membership computed from SQL-like rules defined in JSONB
3. **`ai_generated`**: Membership computed via AI analysis (deferred to Epic 8, interface defined here)

### 2.4 Rule Definition Schema

**Decision**: Use a structured JSONB format for rule definitions that can be translated to SQL queries.

**Schema Structure**:
```json
{
  "operator": "AND" | "OR",
  "rules": [
    {
      "field": "<customer_field_name>",
      "operator": "<comparison_operator>",
      "value": "<comparison_value>"
    }
  ]
}
```

**Supported Fields**: Customer table fields, related entity fields (via joins), computed fields
**Supported Operators**: `equals`, `not_equals`, `in`, `not_in`, `greater_than`, `less_than`, `contains`, `starts_with`, `ends_with`

### 2.5 Idempotency

- Segment computation is idempotent: re-running replaces existing membership
- Use `DELETE ... INSERT` pattern or `UPSERT` to ensure no duplicates
- `last_computed_at` timestamp tracks when segment was last computed

---

## 3. Story CRM-025: Create & Update Segments API

### 3.1 Endpoint Specification

#### 3.1.1 Edge Function Endpoints

**Create**: `POST /crm/segments`
**Update**: `PATCH /crm/segments/:id`

**Method**: `POST` or `PATCH`

**Authentication**: Required (Supabase JWT)

**Content-Type**: `application/json`

#### 3.1.2 RPC Function Alternatives

**Create**: `crm_create_segment`
**Update**: `crm_update_segment`

### 3.2 Request Schema

#### 3.2.1 Create Segment Request

```typescript
interface CreateSegmentRequest {
  // Required fields
  name: string; // Max 100 characters, unique per org
  type: 'static' | 'rule_based' | 'ai_generated';
  
  // Optional fields
  description?: string; // Max 1000 characters
  is_active?: boolean; // Default: true
  
  // Required for rule_based segments
  definition?: {
    operator: 'AND' | 'OR';
    rules: Array<{
      field: string; // Customer field name or computed field
      operator: 'equals' | 'not_equals' | 'in' | 'not_in' | 
                'greater_than' | 'less_than' | 'greater_than_or_equal' | 
                'less_than_or_equal' | 'contains' | 'starts_with' | 
                'ends_with' | 'is_null' | 'is_not_null';
      value: any; // Value to compare against (type depends on field and operator)
    }>;
  };
  
  // Required for ai_generated segments
  ai_prompt?: string; // Max 5000 characters
  
  // Options
  compute_immediately?: boolean; // Default: true for rule_based, false for ai_generated
}
```

#### 3.2.2 Update Segment Request

```typescript
interface UpdateSegmentRequest {
  name?: string; // Max 100 characters
  description?: string; // Max 1000 characters
  is_active?: boolean;
  definition?: {
    operator: 'AND' | 'OR';
    rules: Array<{
      field: string;
      operator: string;
      value: any;
    }>;
  };
  ai_prompt?: string; // Max 5000 characters
  ai_explanation?: string; // Max 5000 characters
}
```

### 3.3 Rule Definition Schema

#### 3.3.1 Supported Customer Fields

**Direct Customer Fields**:
- `status` (enum: `active`, `prospect`, `inactive`, `blacklisted`)
- `lifecycle_stage` (enum: `lead`, `opportunity`, `customer`, `former_customer`)
- `type` (enum: `individual`, `company`)
- `name` (text)
- `email` (text)
- `phone` (text)
- `source` (text)
- `preferred_language` (text)
- `created_at` (timestamptz)
- `updated_at` (timestamptz)

**Related Entity Fields** (via joins):
- `tags.name` (array of tag names)
- `tags.id` (array of tag IDs)
- `locations.city` (array of city names)
- `locations.state` (array of state names)
- `contacts.type` (array of contact types)
- `contacts.value` (array of contact values)

**Computed Fields**:
- `interaction_count` (integer) - Number of interactions
- `last_interaction_at` (timestamptz) - Most recent interaction timestamp
- `followup_count` (integer) - Number of pending follow-ups
- `has_preferences` (boolean) - Whether preferences record exists

#### 3.3.2 Supported Operators

| Operator | Description | Value Type | Example |
|----------|-------------|------------|---------|
| `equals` | Exact match | Same as field type | `{"field": "status", "operator": "equals", "value": "active"}` |
| `not_equals` | Not equal | Same as field type | `{"field": "status", "operator": "not_equals", "value": "inactive"}` |
| `in` | Value in array | Array | `{"field": "status", "operator": "in", "value": ["active", "prospect"]}` |
| `not_in` | Value not in array | Array | `{"field": "lifecycle_stage", "operator": "not_in", "value": ["former_customer"]}` |
| `greater_than` | Greater than | Numeric/Date | `{"field": "created_at", "operator": "greater_than", "value": "2024-01-01"}` |
| `less_than` | Less than | Numeric/Date | `{"field": "created_at", "operator": "less_than", "value": "2024-12-31"}` |
| `greater_than_or_equal` | Greater than or equal | Numeric/Date | Same as above |
| `less_than_or_equal` | Less than or equal | Numeric/Date | Same as above |
| `contains` | String contains | String | `{"field": "name", "operator": "contains", "value": "Corp"}` |
| `starts_with` | String starts with | String | `{"field": "email", "operator": "starts_with", "value": "admin@"}` |
| `ends_with` | String ends with | String | `{"field": "email", "operator": "ends_with", "value": "@example.com"}` |
| `is_null` | Field is NULL | N/A (value ignored) | `{"field": "source", "operator": "is_null", "value": null}` |
| `is_not_null` | Field is not NULL | N/A (value ignored) | `{"field": "email", "operator": "is_not_null", "value": null}` |

#### 3.3.3 Rule Definition Examples

**Example 1: Simple Rule**
```json
{
  "operator": "AND",
  "rules": [
    {
      "field": "status",
      "operator": "equals",
      "value": "active"
    },
    {
      "field": "lifecycle_stage",
      "operator": "equals",
      "value": "customer"
    }
  ]
}
```

**Example 2: Complex Rule with Date Range**
```json
{
  "operator": "AND",
  "rules": [
    {
      "field": "status",
      "operator": "in",
      "value": ["active", "prospect"]
    },
    {
      "field": "created_at",
      "operator": "greater_than",
      "value": "2024-01-01T00:00:00Z"
    },
    {
      "field": "tags.name",
      "operator": "contains",
      "value": "VIP"
    }
  ]
}
```

**Example 3: Nested Rules (Future Enhancement)**
```json
{
  "operator": "OR",
  "rules": [
    {
      "operator": "AND",
      "rules": [
        {"field": "status", "operator": "equals", "value": "active"},
        {"field": "type", "operator": "equals", "value": "company"}
      ]
    },
    {
      "operator": "AND",
      "rules": [
        {"field": "status", "operator": "equals", "value": "active"},
        {"field": "lifecycle_stage", "operator": "equals", "value": "customer"}
      ]
    }
  ]
}
```

**Note**: Nested rules are not supported in initial implementation but schema allows for future extension.

### 3.4 Implementation: Edge Function

#### 3.4.1 Edge Function Structure

**File**: `supabase/functions/crm-segments/index.ts`

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
        JSON.stringify({ error: 'Only admins and managers can manage segments' }),
        { status: 403, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    }

    const orgId = profile.org_id;
    const body = await req.json();

    if (req.method === 'POST') {
      // Create segment
      const validationError = validateCreateSegmentRequest(body);
      if (validationError) {
        return new Response(
          JSON.stringify({ error: validationError }),
          { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
        );
      }

      const { data: segment, error: createError } = await supabaseClient.rpc(
        'crm_create_segment',
        {
          p_org_id: orgId,
          p_name: body.name,
          p_description: body.description || null,
          p_type: body.type,
          p_definition: body.definition || null,
          p_ai_prompt: body.ai_prompt || null,
          p_is_active: body.is_active !== false,
          p_created_by_user_id: user.id,
          p_compute_immediately: body.compute_immediately !== false,
        }
      );

      if (createError) {
        return new Response(
          JSON.stringify({ error: createError.message }),
          { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
        );
      }

      // Compute members if rule_based and compute_immediately is true
      if (body.type === 'rule_based' && body.compute_immediately !== false) {
        await supabaseClient.rpc('crm_compute_segment_members', {
          p_segment_id: segment.id,
          p_org_id: orgId,
        });
      }

      // Trigger AI computation if ai_generated and compute_immediately is true
      if (body.type === 'ai_generated' && body.compute_immediately === true) {
        // Deferred to Epic 8 - AI integration
        // await supabaseClient.functions.invoke('compute-ai-segment', {
        //   body: { segment_id: segment.id }
        // });
      }

      return new Response(
        JSON.stringify(segment),
        { status: 201, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    } else if (req.method === 'PATCH') {
      // Update segment
      const url = new URL(req.url);
      const segmentId = url.pathname.split('/').pop();

      const validationError = validateUpdateSegmentRequest(body);
      if (validationError) {
        return new Response(
          JSON.stringify({ error: validationError }),
          { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
        );
      }

      const { data: segment, error: updateError } = await supabaseClient.rpc(
        'crm_update_segment',
        {
          p_segment_id: segmentId,
          p_org_id: orgId,
          p_name: body.name,
          p_description: body.description,
          p_is_active: body.is_active,
          p_definition: body.definition,
          p_ai_prompt: body.ai_prompt,
          p_ai_explanation: body.ai_explanation,
        }
      );

      if (updateError) {
        return new Response(
          JSON.stringify({ error: updateError.message }),
          { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
        );
      }

      return new Response(
        JSON.stringify(segment),
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

function validateCreateSegmentRequest(body: any): string | null {
  if (!body.name || typeof body.name !== 'string' || body.name.trim().length === 0) {
    return 'name is required and must be a non-empty string';
  }
  
  if (body.name.length > 100) {
    return 'name must be 100 characters or less';
  }
  
  if (!body.type || !['static', 'rule_based', 'ai_generated'].includes(body.type)) {
    return 'type is required and must be one of: static, rule_based, ai_generated';
  }
  
  if (body.description && body.description.length > 1000) {
    return 'description must be 1000 characters or less';
  }
  
  if (body.type === 'rule_based' && !body.definition) {
    return 'definition is required for rule_based segments';
  }
  
  if (body.type === 'ai_generated' && !body.ai_prompt) {
    return 'ai_prompt is required for ai_generated segments';
  }
  
  if (body.definition) {
    const definitionError = validateRuleDefinition(body.definition);
    if (definitionError) {
      return definitionError;
    }
  }
  
  if (body.ai_prompt && body.ai_prompt.length > 5000) {
    return 'ai_prompt must be 5000 characters or less';
  }
  
  return null;
}

function validateUpdateSegmentRequest(body: any): string | null {
  if (body.name !== undefined) {
    if (typeof body.name !== 'string' || body.name.trim().length === 0) {
      return 'name must be a non-empty string';
    }
    if (body.name.length > 100) {
      return 'name must be 100 characters or less';
    }
  }
  
  if (body.description !== undefined && body.description !== null) {
    if (body.description.length > 1000) {
      return 'description must be 1000 characters or less';
    }
  }
  
  if (body.definition !== undefined && body.definition !== null) {
    const definitionError = validateRuleDefinition(body.definition);
    if (definitionError) {
      return definitionError;
    }
  }
  
  if (body.ai_prompt !== undefined && body.ai_prompt !== null) {
    if (body.ai_prompt.length > 5000) {
      return 'ai_prompt must be 5000 characters or less';
    }
  }
  
  if (body.ai_explanation !== undefined && body.ai_explanation !== null) {
    if (body.ai_explanation.length > 5000) {
      return 'ai_explanation must be 5000 characters or less';
    }
  }
  
  return null;
}

function validateRuleDefinition(definition: any): string | null {
  if (!definition || typeof definition !== 'object') {
    return 'definition must be an object';
  }
  
  if (!definition.operator || !['AND', 'OR'].includes(definition.operator)) {
    return 'definition.operator must be "AND" or "OR"';
  }
  
  if (!Array.isArray(definition.rules) || definition.rules.length === 0) {
    return 'definition.rules must be a non-empty array';
  }
  
  const supportedOperators = [
    'equals', 'not_equals', 'in', 'not_in',
    'greater_than', 'less_than', 'greater_than_or_equal', 'less_than_or_equal',
    'contains', 'starts_with', 'ends_with', 'is_null', 'is_not_null'
  ];
  
  const supportedFields = [
    'status', 'lifecycle_stage', 'type', 'name', 'email', 'phone',
    'source', 'preferred_language', 'created_at', 'updated_at',
    'tags.name', 'tags.id', 'locations.city', 'locations.state',
    'contacts.type', 'contacts.value',
    'interaction_count', 'last_interaction_at', 'followup_count', 'has_preferences'
  ];
  
  for (const rule of definition.rules) {
    if (!rule.field || typeof rule.field !== 'string') {
      return 'Each rule must have a field property';
    }
    
    if (!supportedFields.includes(rule.field)) {
      return `Unsupported field: ${rule.field}`;
    }
    
    if (!rule.operator || !supportedOperators.includes(rule.operator)) {
      return `Unsupported operator: ${rule.operator}`;
    }
    
    // Validate value based on operator
    if (['is_null', 'is_not_null'].includes(rule.operator)) {
      // Value can be null or omitted
    } else if (rule.value === undefined || rule.value === null) {
      return `Rule with operator ${rule.operator} requires a value`;
    }
    
    // Validate array operators
    if (['in', 'not_in'].includes(rule.operator)) {
      if (!Array.isArray(rule.value)) {
        return `Operator ${rule.operator} requires an array value`;
      }
    }
  }
  
  return null;
}
```

### 3.5 Implementation: PostgreSQL RPC Functions

#### 3.5.1 Create Segment Function

```sql
CREATE OR REPLACE FUNCTION crm_create_segment(
  p_org_id UUID,
  p_name TEXT,
  p_description TEXT DEFAULT NULL,
  p_type segment_type_enum,
  p_definition JSONB DEFAULT NULL,
  p_ai_prompt TEXT DEFAULT NULL,
  p_is_active BOOLEAN DEFAULT true,
  p_created_by_user_id UUID DEFAULT NULL,
  p_compute_immediately BOOLEAN DEFAULT true
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_user_id UUID;
  v_user_org_id UUID;
  v_segment_id UUID;
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
  
  -- Check role (admin or manager only)
  IF NOT EXISTS (
    SELECT 1 FROM profiles
    WHERE id = v_user_id
    AND role IN ('admin', 'manager')
    AND is_active = true
  ) THEN
    RAISE EXCEPTION 'Only admins and managers can create segments';
  END IF;
  
  -- Validate name
  IF p_name IS NULL OR trim(p_name) = '' THEN
    RAISE EXCEPTION 'name is required and must be non-empty';
  END IF;
  
  IF length(p_name) > 100 THEN
    RAISE EXCEPTION 'name must be 100 characters or less';
  END IF;
  
  -- Check name uniqueness
  IF EXISTS (
    SELECT 1 FROM crm_segments
    WHERE org_id = p_org_id AND name = p_name
  ) THEN
    RAISE EXCEPTION 'Segment name already exists';
  END IF;
  
  -- Validate type-specific requirements
  IF p_type = 'rule_based' AND p_definition IS NULL THEN
    RAISE EXCEPTION 'definition is required for rule_based segments';
  END IF;
  
  IF p_type = 'ai_generated' AND p_ai_prompt IS NULL THEN
    RAISE EXCEPTION 'ai_prompt is required for ai_generated segments';
  END IF;
  
  -- Validate description length
  IF p_description IS NOT NULL AND length(p_description) > 1000 THEN
    RAISE EXCEPTION 'description must be 1000 characters or less';
  END IF;
  
  -- Validate ai_prompt length
  IF p_ai_prompt IS NOT NULL AND length(p_ai_prompt) > 5000 THEN
    RAISE EXCEPTION 'ai_prompt must be 5000 characters or less';
  END IF;
  
  -- Insert segment
  INSERT INTO crm_segments (
    org_id,
    name,
    description,
    type,
    definition,
    ai_prompt,
    is_active,
    created_by_user_id
  ) VALUES (
    p_org_id,
    p_name,
    NULLIF(trim(p_description), ''),
    p_type,
    p_definition,
    NULLIF(trim(p_ai_prompt), ''),
    p_is_active,
    v_user_id
  )
  RETURNING id INTO v_segment_id;
  
  -- Return created segment
  SELECT jsonb_build_object(
    'id', cs.id,
    'org_id', cs.org_id,
    'name', cs.name,
    'description', cs.description,
    'type', cs.type,
    'definition', cs.definition,
    'ai_prompt', cs.ai_prompt,
    'ai_explanation', cs.ai_explanation,
    'is_active', cs.is_active,
    'last_computed_at', cs.last_computed_at,
    'created_by_user_id', cs.created_by_user_id,
    'created_at', cs.created_at,
    'updated_at', cs.updated_at
  )
  INTO v_result
  FROM crm_segments cs
  WHERE cs.id = v_segment_id;
  
  RETURN v_result;
END;
$$;
```

#### 3.5.2 Update Segment Function

```sql
CREATE OR REPLACE FUNCTION crm_update_segment(
  p_segment_id UUID,
  p_org_id UUID,
  p_name TEXT DEFAULT NULL,
  p_description TEXT DEFAULT NULL,
  p_is_active BOOLEAN DEFAULT NULL,
  p_definition JSONB DEFAULT NULL,
  p_ai_prompt TEXT DEFAULT NULL,
  p_ai_explanation TEXT DEFAULT NULL
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_user_id UUID;
  v_user_org_id UUID;
  v_current_segment RECORD;
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
    RAISE EXCEPTION 'Only admins and managers can update segments';
  END IF;
  
  -- Get current segment
  SELECT * INTO v_current_segment
  FROM crm_segments
  WHERE id = p_segment_id AND org_id = p_org_id;
  
  IF NOT FOUND THEN
    RAISE EXCEPTION 'Segment not found or access denied';
  END IF;
  
  -- Validate name if provided
  IF p_name IS NOT NULL THEN
    IF trim(p_name) = '' THEN
      RAISE EXCEPTION 'name must be non-empty';
    END IF;
    
    IF length(p_name) > 100 THEN
      RAISE EXCEPTION 'name must be 100 characters or less';
    END IF;
    
    -- Check name uniqueness (excluding current segment)
    IF EXISTS (
      SELECT 1 FROM crm_segments
      WHERE org_id = p_org_id
      AND name = p_name
      AND id != p_segment_id
    ) THEN
      RAISE EXCEPTION 'Segment name already exists';
    END IF;
  END IF;
  
  -- Validate description length
  IF p_description IS NOT NULL AND length(p_description) > 1000 THEN
    RAISE EXCEPTION 'description must be 1000 characters or less';
  END IF;
  
  -- Validate ai_prompt length
  IF p_ai_prompt IS NOT NULL AND length(p_ai_prompt) > 5000 THEN
    RAISE EXCEPTION 'ai_prompt must be 5000 characters or less';
  END IF;
  
  -- Validate ai_explanation length
  IF p_ai_explanation IS NOT NULL AND length(p_ai_explanation) > 5000 THEN
    RAISE EXCEPTION 'ai_explanation must be 5000 characters or less';
  END IF;
  
  -- Update segment
  UPDATE crm_segments
  SET
    name = COALESCE(p_name, name),
    description = CASE WHEN p_description IS NOT NULL THEN NULLIF(trim(p_description), '') ELSE description END,
    is_active = COALESCE(p_is_active, is_active),
    definition = COALESCE(p_definition, definition),
    ai_prompt = CASE WHEN p_ai_prompt IS NOT NULL THEN NULLIF(trim(p_ai_prompt), '') ELSE ai_prompt END,
    ai_explanation = CASE WHEN p_ai_explanation IS NOT NULL THEN NULLIF(trim(p_ai_explanation), '') ELSE ai_explanation END,
    updated_at = now()
  WHERE id = p_segment_id;
  
  -- Return updated segment
  SELECT jsonb_build_object(
    'id', cs.id,
    'org_id', cs.org_id,
    'name', cs.name,
    'description', cs.description,
    'type', cs.type,
    'definition', cs.definition,
    'ai_prompt', cs.ai_prompt,
    'ai_explanation', cs.ai_explanation,
    'is_active', cs.is_active,
    'last_computed_at', cs.last_computed_at,
    'created_by_user_id', cs.created_by_user_id,
    'created_at', cs.created_at,
    'updated_at', cs.updated_at
  )
  INTO v_result
  FROM crm_segments cs
  WHERE cs.id = p_segment_id;
  
  RETURN v_result;
END;
$$;
```

### 3.6 Response Schema

#### 3.6.1 Success Response (201 Created / 200 OK)

```typescript
interface SegmentResponse {
  id: string; // UUID
  org_id: string; // UUID
  name: string;
  description?: string;
  type: 'static' | 'rule_based' | 'ai_generated';
  definition?: {
    operator: 'AND' | 'OR';
    rules: Array<{
      field: string;
      operator: string;
      value: any;
    }>;
  };
  ai_prompt?: string;
  ai_explanation?: string;
  is_active: boolean;
  last_computed_at?: string; // ISO 8601 timestamp
  created_by_user_id?: string; // UUID
  created_at: string; // ISO 8601 timestamp
  updated_at: string; // ISO 8601 timestamp
}
```

---

## 4. Story CRM-026: Compute Members for Rule-Based Segments

### 4.1 Function Specification

**Function Name**: `crm_compute_segment_members`

**Purpose**: Evaluate rule-based segment definition and populate `crm_segment_members` table

**Parameters**:
- `p_segment_id UUID` - Segment ID
- `p_org_id UUID` - Organization ID (for security)

**Return Type**: `JSONB` - Computation result with member count

### 4.2 Rule Evaluation Logic

#### 4.2.1 Rule Translation to SQL

Rules are translated to SQL WHERE clauses:

**Single Rule Translation**:
- `{"field": "status", "operator": "equals", "value": "active"}` → `c.status = 'active'`
- `{"field": "status", "operator": "in", "value": ["active", "prospect"]}` → `c.status IN ('active', 'prospect')`
- `{"field": "created_at", "operator": "greater_than", "value": "2024-01-01"}` → `c.created_at > '2024-01-01'::timestamptz`
- `{"field": "name", "operator": "contains", "value": "Corp"}` → `c.name ILIKE '%Corp%'`

**Operator Translation**:
- `equals` → `=`
- `not_equals` → `!=`
- `in` → `IN (...)`
- `not_in` → `NOT IN (...)`
- `greater_than` → `>`
- `less_than` → `<`
- `greater_than_or_equal` → `>=`
- `less_than_or_equal` → `<=`
- `contains` → `ILIKE '%value%'`
- `starts_with` → `ILIKE 'value%'`
- `ends_with` → `ILIKE '%value'`
- `is_null` → `IS NULL`
- `is_not_null` → `IS NOT NULL`

**Related Entity Fields**:
- `tags.name` → Join with `crm_customer_tags` and `crm_tags`
- `tags.id` → Join with `crm_customer_tags`
- `locations.city` → Join with `customer_locations`
- `contacts.type` → Join with `customer_contacts`

**Computed Fields**:
- `interaction_count` → Subquery counting `crm_interactions`
- `last_interaction_at` → Subquery getting MAX `occurred_at` from `crm_interactions`
- `followup_count` → Subquery counting `crm_followups` WHERE `status = 'pending'`
- `has_preferences` → EXISTS subquery on `crm_preferences`

### 4.3 Implementation: PostgreSQL Function

```sql
CREATE OR REPLACE FUNCTION crm_compute_segment_members(
  p_segment_id UUID,
  p_org_id UUID
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_user_org_id UUID;
  v_segment RECORD;
  v_rule RECORD;
  v_sql TEXT;
  v_where_clause TEXT;
  v_join_clauses TEXT[];
  v_member_count INTEGER;
  v_result JSONB;
BEGIN
  -- Get user's org_id
  SELECT org_id INTO v_user_org_id
  FROM profiles
  WHERE id = auth.uid() AND is_active = true;
  
  IF v_user_org_id IS NULL THEN
    RAISE EXCEPTION 'User not authenticated or inactive';
  END IF;
  
  IF v_user_org_id != p_org_id THEN
    RAISE EXCEPTION 'Access denied';
  END IF;
  
  -- Get segment
  SELECT * INTO v_segment
  FROM crm_segments
  WHERE id = p_segment_id AND org_id = p_org_id;
  
  IF NOT FOUND THEN
    RAISE EXCEPTION 'Segment not found or access denied';
  END IF;
  
  IF v_segment.type != 'rule_based' THEN
    RAISE EXCEPTION 'Segment is not rule_based';
  END IF;
  
  IF v_segment.definition IS NULL THEN
    RAISE EXCEPTION 'Segment definition is missing';
  END IF;
  
  -- Build SQL query from definition
  v_where_clause := '';
  v_join_clauses := ARRAY[]::TEXT[];
  
  -- Process each rule
  FOR v_rule IN 
    SELECT * FROM jsonb_array_elements(v_segment.definition->'rules')
  LOOP
    v_where_clause := v_where_clause || build_rule_condition(
      v_rule.value,
      v_join_clauses
    );
    
    -- Add operator between rules
    IF v_where_clause != '' THEN
      v_where_clause := v_where_clause || ' ' || v_segment.definition->>'operator' || ' ';
    END IF;
  END LOOP;
  
  -- Remove trailing operator
  IF v_where_clause LIKE '% AND %' OR v_where_clause LIKE '% OR %' THEN
    v_where_clause := regexp_replace(v_where_clause, ' (AND|OR) $', '');
  END IF;
  
  -- Build final SQL
  v_sql := format('
    WITH matching_customers AS (
      SELECT DISTINCT c.id as customer_id
      FROM customers c
      %s
      WHERE c.org_id = %L
      AND (%s)
    )
    INSERT INTO crm_segment_members (org_id, segment_id, customer_id, created_at)
    SELECT %L, %L, customer_id, now()
    FROM matching_customers
    ON CONFLICT (org_id, segment_id, customer_id) DO NOTHING;
  ',
    array_to_string(v_join_clauses, ' '),
    p_org_id,
    v_where_clause,
    p_org_id,
    p_segment_id
  );
  
  -- Delete existing members (idempotent: replace membership)
  DELETE FROM crm_segment_members
  WHERE segment_id = p_segment_id AND org_id = p_org_id;
  
  -- Execute query to insert new members
  EXECUTE v_sql;
  
  -- Get member count
  SELECT COUNT(*) INTO v_member_count
  FROM crm_segment_members
  WHERE segment_id = p_segment_id AND org_id = p_org_id;
  
  -- Update segment last_computed_at
  UPDATE crm_segments
  SET last_computed_at = now()
  WHERE id = p_segment_id;
  
  -- Return result
  v_result := jsonb_build_object(
    'segment_id', p_segment_id,
    'member_count', v_member_count,
    'computed_at', now()
  );
  
  RETURN v_result;
  
EXCEPTION
  WHEN OTHERS THEN
    RAISE EXCEPTION 'Error computing segment members: %', SQLERRM;
END;
$$;

-- Helper function to build rule condition
CREATE OR REPLACE FUNCTION build_rule_condition(
  p_rule JSONB,
  INOUT p_join_clauses TEXT[]
)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
  v_field TEXT;
  v_operator TEXT;
  v_value TEXT;
  v_condition TEXT;
BEGIN
  v_field := p_rule->>'field';
  v_operator := p_rule->>'operator';
  v_value := p_rule->>'value';
  
  -- Handle different field types
  CASE v_field
    -- Direct customer fields
    WHEN 'status', 'lifecycle_stage', 'type', 'name', 'email', 'phone', 
         'source', 'preferred_language', 'created_at', 'updated_at' THEN
      v_condition := build_direct_field_condition('c.' || v_field, v_operator, v_value);
    
    -- Related entity fields
    WHEN 'tags.name' THEN
      v_condition := build_tags_name_condition(v_operator, v_value, p_join_clauses);
    WHEN 'tags.id' THEN
      v_condition := build_tags_id_condition(v_operator, v_value, p_join_clauses);
    WHEN 'locations.city' THEN
      v_condition := build_locations_city_condition(v_operator, v_value, p_join_clauses);
    WHEN 'locations.state' THEN
      v_condition := build_locations_state_condition(v_operator, v_value, p_join_clauses);
    WHEN 'contacts.type' THEN
      v_condition := build_contacts_type_condition(v_operator, v_value, p_join_clauses);
    WHEN 'contacts.value' THEN
      v_condition := build_contacts_value_condition(v_operator, v_value, p_join_clauses);
    
    -- Computed fields
    WHEN 'interaction_count' THEN
      v_condition := build_interaction_count_condition(v_operator, v_value);
    WHEN 'last_interaction_at' THEN
      v_condition := build_last_interaction_at_condition(v_operator, v_value);
    WHEN 'followup_count' THEN
      v_condition := build_followup_count_condition(v_operator, v_value);
    WHEN 'has_preferences' THEN
      v_condition := build_has_preferences_condition(v_operator, v_value);
    
    ELSE
      RAISE EXCEPTION 'Unsupported field: %', v_field;
  END CASE;
  
  RETURN v_condition;
END;
$$;

-- Helper function for direct field conditions
CREATE OR REPLACE FUNCTION build_direct_field_condition(
  p_field TEXT,
  p_operator TEXT,
  p_value JSONB
)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
  v_condition TEXT;
  v_sql_value TEXT;
BEGIN
  CASE p_operator
    WHEN 'equals' THEN
      v_sql_value := quote_literal(p_value::TEXT);
      v_condition := format('%s = %s', p_field, v_sql_value);
    
    WHEN 'not_equals' THEN
      v_sql_value := quote_literal(p_value::TEXT);
      v_condition := format('%s != %s', p_field, v_sql_value);
    
    WHEN 'in' THEN
      v_sql_value := array_to_string(
        ARRAY(SELECT quote_literal(elem::TEXT) FROM jsonb_array_elements(p_value)),
        ', '
      );
      v_condition := format('%s IN (%s)', p_field, v_sql_value);
    
    WHEN 'not_in' THEN
      v_sql_value := array_to_string(
        ARRAY(SELECT quote_literal(elem::TEXT) FROM jsonb_array_elements(p_value)),
        ', '
      );
      v_condition := format('%s NOT IN (%s)', p_field, v_sql_value);
    
    WHEN 'greater_than' THEN
      v_sql_value := quote_literal(p_value::TEXT);
      v_condition := format('%s > %s', p_field, v_sql_value);
    
    WHEN 'less_than' THEN
      v_sql_value := quote_literal(p_value::TEXT);
      v_condition := format('%s < %s', p_field, v_sql_value);
    
    WHEN 'greater_than_or_equal' THEN
      v_sql_value := quote_literal(p_value::TEXT);
      v_condition := format('%s >= %s', p_field, v_sql_value);
    
    WHEN 'less_than_or_equal' THEN
      v_sql_value := quote_literal(p_value::TEXT);
      v_condition := format('%s <= %s', p_field, v_sql_value);
    
    WHEN 'contains' THEN
      v_sql_value := quote_literal('%' || p_value::TEXT || '%');
      v_condition := format('%s ILIKE %s', p_field, v_sql_value);
    
    WHEN 'starts_with' THEN
      v_sql_value := quote_literal(p_value::TEXT || '%');
      v_condition := format('%s ILIKE %s', p_field, v_sql_value);
    
    WHEN 'ends_with' THEN
      v_sql_value := quote_literal('%' || p_value::TEXT);
      v_condition := format('%s ILIKE %s', p_field, v_sql_value);
    
    WHEN 'is_null' THEN
      v_condition := format('%s IS NULL', p_field);
    
    WHEN 'is_not_null' THEN
      v_condition := format('%s IS NOT NULL', p_field);
    
    ELSE
      RAISE EXCEPTION 'Unsupported operator: %', p_operator;
  END CASE;
  
  RETURN v_condition;
END;
$$;

-- Helper functions for related entity fields (simplified examples)
CREATE OR REPLACE FUNCTION build_tags_name_condition(
  p_operator TEXT,
  p_value JSONB,
  INOUT p_join_clauses TEXT[]
)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
  v_condition TEXT;
  v_join_alias TEXT := 'cct_tags';
BEGIN
  -- Add join if not already present
  IF NOT EXISTS (
    SELECT 1 FROM unnest(p_join_clauses) WHERE unnest LIKE '%' || v_join_alias || '%'
  ) THEN
    p_join_clauses := array_append(p_join_clauses, 
      format('LEFT JOIN crm_customer_tags %s ON %s.customer_id = c.id', v_join_alias, v_join_alias)
    );
    p_join_clauses := array_append(p_join_clauses,
      format('LEFT JOIN crm_tags t_tags ON t_tags.id = %s.tag_id', v_join_alias)
    );
  END IF;
  
  -- Build condition
  CASE p_operator
    WHEN 'contains' THEN
      v_condition := format('t_tags.name ILIKE %s', quote_literal('%' || p_value::TEXT || '%'));
    WHEN 'equals' THEN
      v_condition := format('t_tags.name = %s', quote_literal(p_value::TEXT));
    WHEN 'in' THEN
      v_condition := format('t_tags.name IN (%s)',
        array_to_string(
          ARRAY(SELECT quote_literal(elem::TEXT) FROM jsonb_array_elements(p_value)),
          ', '
        )
      );
    ELSE
      RAISE EXCEPTION 'Unsupported operator for tags.name: %', p_operator;
  END CASE;
  
  RETURN v_condition;
END;
$$;

-- Similar helper functions for other related fields...
-- (locations.city, locations.state, contacts.type, contacts.value, etc.)

-- Helper functions for computed fields
CREATE OR REPLACE FUNCTION build_interaction_count_condition(
  p_operator TEXT,
  p_value JSONB
)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
DECLARE
  v_condition TEXT;
  v_count_value INTEGER;
BEGIN
  v_count_value := (p_value::TEXT)::INTEGER;
  
  v_condition := format('(
    SELECT COUNT(*) FROM crm_interactions ci
    WHERE ci.customer_id = c.id
  ) %s %s',
    CASE p_operator
      WHEN 'equals' THEN '='
      WHEN 'greater_than' THEN '>'
      WHEN 'less_than' THEN '<'
      WHEN 'greater_than_or_equal' THEN '>='
      WHEN 'less_than_or_equal' THEN '<='
      ELSE RAISE EXCEPTION 'Unsupported operator for interaction_count: %', p_operator
    END,
    v_count_value
  );
  
  RETURN v_condition;
END;
$$;

-- Similar functions for other computed fields...
```

**Note**: The above implementation is simplified. A complete implementation would include all helper functions for related entity fields and computed fields.

### 4.4 Idempotency

- Delete existing members before inserting new ones
- Use `ON CONFLICT DO NOTHING` as safety net
- Update `last_computed_at` timestamp
- Return member count in result

---

## 5. Story CRM-027: Get Segment Members API

### 5.1 Endpoint Specification

**Path**: `GET /crm/segments/:id/members`

**Method**: `GET`

**Authentication**: Required (Supabase JWT)

**Query Parameters**:
- `limit` (optional, default: 20, max: 100) - Number of members per page
- `offset` (optional, default: 0) - Pagination offset
- `min_score` (optional) - Minimum score for AI segments (numeric)
- `sort_by` (optional, default: `name`) - Sort field: `name`, `created_at`, `score`
- `sort_order` (optional, default: `asc`) - Sort order: `asc`, `desc`

### 5.2 Response Schema

```typescript
interface GetSegmentMembersResponse {
  segment: {
    id: string; // UUID
    name: string;
    type: 'static' | 'rule_based' | 'ai_generated';
    last_computed_at?: string; // ISO 8601 timestamp
  };
  data: Array<{
    customer_id: string; // UUID
    customer: {
      id: string;
      name: string;
      type: 'individual' | 'company';
      email?: string;
      phone?: string;
      status: string;
      lifecycle_stage: string;
    };
    tags: Array<{
      id: string;
      name: string;
      color?: string;
    }>;
    score?: number; // For AI segments
    metadata?: {
      [key: string]: any;
    };
    created_at: string; // ISO 8601 timestamp
  }>;
  total: number;
  limit: number;
  offset: number;
  has_more: boolean;
}
```

### 5.3 Implementation: PostgreSQL RPC Function

```sql
CREATE OR REPLACE FUNCTION crm_get_segment_members(
  p_segment_id UUID,
  p_org_id UUID,
  p_limit INTEGER DEFAULT 20,
  p_offset INTEGER DEFAULT 0,
  p_min_score NUMERIC DEFAULT NULL,
  p_sort_by TEXT DEFAULT 'name',
  p_sort_order TEXT DEFAULT 'asc'
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_user_org_id UUID;
  v_segment RECORD;
  v_total INTEGER;
  v_result JSONB;
  v_sort_expr TEXT;
BEGIN
  -- Get user's org_id
  SELECT org_id INTO v_user_org_id
  FROM profiles
  WHERE id = auth.uid() AND is_active = true;
  
  IF v_user_org_id IS NULL THEN
    RAISE EXCEPTION 'User not authenticated or inactive';
  END IF;
  
  IF v_user_org_id != p_org_id THEN
    RAISE EXCEPTION 'Access denied';
  END IF;
  
  -- Get segment
  SELECT * INTO v_segment
  FROM crm_segments
  WHERE id = p_segment_id AND org_id = p_org_id;
  
  IF NOT FOUND THEN
    RAISE EXCEPTION 'Segment not found or access denied';
  END IF;
  
  -- Validate limits
  IF p_limit > 100 THEN
    p_limit := 100;
  END IF;
  IF p_limit < 1 THEN
    p_limit := 20;
  END IF;
  
  -- Validate sort_by
  IF p_sort_by NOT IN ('name', 'created_at', 'score') THEN
    p_sort_by := 'name';
  END IF;
  
  -- Validate sort_order
  IF p_sort_order NOT IN ('asc', 'desc') THEN
    p_sort_order := 'asc';
  END IF;
  
  v_sort_expr := p_sort_by || ' ' || UPPER(p_sort_order);
  
  -- Get total count
  SELECT COUNT(*) INTO v_total
  FROM crm_segment_members csm
  WHERE csm.segment_id = p_segment_id
  AND csm.org_id = p_org_id
  AND (p_min_score IS NULL OR csm.score IS NULL OR csm.score >= p_min_score);
  
  -- Build result
  SELECT jsonb_build_object(
    'segment', jsonb_build_object(
      'id', v_segment.id,
      'name', v_segment.name,
      'type', v_segment.type,
      'last_computed_at', v_segment.last_computed_at
    ),
    'data', COALESCE(
      (
        SELECT jsonb_agg(jsonb_build_object(
          'customer_id', csm.customer_id,
          'customer', jsonb_build_object(
            'id', c.id,
            'name', c.name,
            'type', c.type,
            'email', c.email,
            'phone', c.phone,
            'status', c.status,
            'lifecycle_stage', c.lifecycle_stage
          ),
          'tags', COALESCE(
            (
              SELECT jsonb_agg(jsonb_build_object(
                'id', t.id,
                'name', t.name,
                'color', t.color
              ))
              FROM crm_customer_tags cct
              JOIN crm_tags t ON t.id = cct.tag_id
              WHERE cct.customer_id = c.id
            ),
            '[]'::jsonb
          ),
          'score', csm.score,
          'metadata', csm.metadata,
          'created_at', csm.created_at
        ) ORDER BY
          CASE WHEN v_sort_expr LIKE 'name%' THEN c.name END,
          CASE WHEN v_sort_expr LIKE 'created_at DESC' THEN csm.created_at END DESC,
          CASE WHEN v_sort_expr LIKE 'created_at ASC' THEN csm.created_at END ASC,
          CASE WHEN v_sort_expr LIKE 'score DESC' THEN csm.score END DESC NULLS LAST,
          CASE WHEN v_sort_expr LIKE 'score ASC' THEN csm.score END ASC NULLS LAST
        )
        FROM crm_segment_members csm
        JOIN customers c ON c.id = csm.customer_id
        WHERE csm.segment_id = p_segment_id
        AND csm.org_id = p_org_id
        AND (p_min_score IS NULL OR csm.score IS NULL OR csm.score >= p_min_score)
        ORDER BY
          CASE WHEN v_sort_expr LIKE 'name DESC' THEN c.name END DESC,
          CASE WHEN v_sort_expr LIKE 'name ASC' THEN c.name END ASC,
          CASE WHEN v_sort_expr LIKE 'created_at DESC' THEN csm.created_at END DESC,
          CASE WHEN v_sort_expr LIKE 'created_at ASC' THEN csm.created_at END ASC,
          CASE WHEN v_sort_expr LIKE 'score DESC' THEN csm.score END DESC NULLS LAST,
          CASE WHEN v_sort_expr LIKE 'score ASC' THEN csm.score END ASC NULLS LAST
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

## 6. Story CRM-028: Recompute Segment Members API

### 6.1 Endpoint Specification

**Path**: `POST /crm/segments/:id/recompute`

**Method**: `POST`

**Authentication**: Required (Supabase JWT)

**Request Body**: Empty or optional parameters

### 6.2 Implementation: Edge Function

```typescript
// Add to existing Edge Function

if (req.method === 'POST' && url.pathname.includes('/recompute')) {
  const segmentId = url.pathname.split('/')[url.pathname.split('/').length - 2];
  
  const { data: profile } = await supabaseClient
    .from('profiles')
    .select('org_id, role')
    .eq('id', user.id)
    .single();
  
  if (!['admin', 'manager'].includes(profile.role)) {
    return new Response(
      JSON.stringify({ error: 'Only admins and managers can trigger recomputation' }),
      { status: 403, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
  
  // Get segment
  const { data: segment, error: segmentError } = await supabaseClient
    .from('crm_segments')
    .select('*')
    .eq('id', segmentId)
    .eq('org_id', profile.org_id)
    .single();
  
  if (segmentError || !segment) {
    return new Response(
      JSON.stringify({ error: 'Segment not found or access denied' }),
      { status: 404, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
  
  let result;
  
  if (segment.type === 'rule_based') {
    // Compute rule-based segment
    const { data: computeResult, error: computeError } = await supabaseClient.rpc(
      'crm_compute_segment_members',
      {
        p_segment_id: segmentId,
        p_org_id: profile.org_id,
      }
    );
    
    if (computeError) {
      return new Response(
        JSON.stringify({ error: computeError.message }),
        { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    }
    
    result = computeResult;
  } else if (segment.type === 'ai_generated') {
    // Trigger AI computation (deferred to Epic 8)
    // For now, return placeholder
    result = {
      segment_id: segmentId,
      status: 'queued',
      message: 'AI computation queued (implementation deferred to Epic 8)'
    };
    
    // Future: await supabaseClient.functions.invoke('compute-ai-segment', {
    //   body: { segment_id: segmentId }
    // });
  } else {
    return new Response(
      JSON.stringify({ error: 'Static segments cannot be recomputed' }),
      { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
  
  return new Response(
    JSON.stringify(result),
    { status: 200, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
  );
}
```

### 6.3 Response Schema

**Success Response (200 OK)**:
```typescript
interface RecomputeSegmentResponse {
  segment_id: string; // UUID
  member_count?: number; // For rule_based segments
  computed_at?: string; // ISO 8601 timestamp
  status?: string; // For ai_generated segments: 'queued', 'processing', 'completed'
  message?: string; // Status message
}
```

---

## 7. Error Handling

### 7.1 Standard Error Response Format

All APIs return errors in this format:

```typescript
interface ErrorResponse {
  error: string; // Human-readable error message
  code?: string; // Optional error code
  details?: any; // Optional additional error details
}
```

### 7.2 HTTP Status Codes

- **200 OK**: Successful GET/POST/PATCH request
- **201 Created**: Successful POST request
- **400 Bad Request**: Validation error or invalid input
- **401 Unauthorized**: Missing or invalid authentication token
- **403 Forbidden**: User lacks permission (role check failed)
- **404 Not Found**: Resource not found
- **500 Internal Server Error**: Server error

### 7.3 Error Codes

- `VALIDATION_ERROR`: Request validation failed
- `NOT_FOUND`: Resource not found
- `UNAUTHORIZED`: Authentication required
- `FORBIDDEN`: Insufficient permissions
- `INVALID_RULE_DEFINITION`: Rule definition syntax error
- `UNSUPPORTED_FIELD`: Field not supported in rule definition
- `UNSUPPORTED_OPERATOR`: Operator not supported for field type
- `DATABASE_ERROR`: Database operation failed
- `SEGMENT_NAME_EXISTS`: Segment name already exists

---

## 8. Performance Considerations

### 8.1 Query Optimization

- Use indexes defined in Epic 1 for all filter and sort operations
- Rule-based computation uses efficient SQL queries
- Consider limiting rule complexity for performance
- Batch insert segment members for better performance

### 8.2 Performance Targets

- Create Segment: < 200ms
- Compute Rule-Based Segment: < 5 seconds (for up to 50k customers)
- Get Segment Members: < 500ms (for 20 members with pagination)
- Recompute Segment: < 5 seconds (for rule_based segments)

### 8.3 Limitations

- Rule complexity: Maximum 50 rules per segment definition
- Member count: No hard limit, but performance degrades with very large segments (>100k members)
- Nested rules: Not supported in initial implementation

---

## 9. Testing Requirements

### 9.1 Unit Tests

- Rule definition validation
- Rule translation to SQL
- Field and operator validation
- Error handling paths

### 9.2 Integration Tests

- End-to-end segment creation and computation
- Rule-based segment computation with various rule types
- Member retrieval with pagination and filtering
- Recompute functionality
- RLS enforcement verification

### 9.3 Performance Tests

- Rule computation with large customer datasets
- Member retrieval performance
- Concurrent segment computations

---

## 10. Implementation Checklist

### Story CRM-025: Create & Update Segments
- [ ] Edge Functions implemented (`POST /crm/segments`, `PATCH /crm/segments/:id`)
- [ ] RPC functions implemented (`crm_create_segment`, `crm_update_segment`)
- [ ] Request validation implemented
- [ ] Rule definition schema validation
- [ ] Role-based authorization (admin/manager only)
- [ ] Name uniqueness validation
- [ ] Type-specific field validation
- [ ] Error handling
- [ ] Response schema matches specification
- [ ] Tests written
- [ ] API documentation with examples

### Story CRM-026: Compute Rule-Based Segment Members
- [ ] RPC function implemented (`crm_compute_segment_members`)
- [ ] Rule translation to SQL logic
- [ ] Support for all field types (direct, related, computed)
- [ ] Support for all operators
- [ ] Idempotent computation (delete then insert)
- [ ] `last_computed_at` timestamp update
- [ ] Performance validated
- [ ] Tests written with example rules
- [ ] Documentation with rule examples

### Story CRM-027: Get Segment Members
- [ ] Edge Function or RPC function implemented (`GET /crm/segments/:id/members`)
- [ ] Pagination support
- [ ] Filter support (min_score for AI segments)
- [ ] Sort support (name, created_at, score)
- [ ] Customer information included
- [ ] Tags included
- [ ] Score and metadata included (for AI segments)
- [ ] Performance validated
- [ ] Response schema matches specification
- [ ] Tests written
- [ ] API documentation with examples

### Story CRM-028: Recompute Segment Members
- [ ] Edge Function implemented (`POST /crm/segments/:id/recompute`)
- [ ] Rule-based recomputation logic
- [ ] AI-generated segment interface (deferred to Epic 8)
- [ ] Role-based authorization (admin/manager only)
- [ ] Status tracking
- [ ] Error handling
- [ ] Response schema matches specification
- [ ] Tests written
- [ ] API documentation with examples

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 6 – Segmentation & Targeting. All specifications are designed to be directly consumable by LLM-based code generators, with exact request/response schemas, validation rules, SQL functions, Edge Function implementations, rule definition schemas, and computation logic defined.

