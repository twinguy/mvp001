# Technical Design Document – Epic 4: Interaction & Communication Logging

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 4 – Interaction & Communication Logging
- **Source**: Derived from `fdd_1_agile.md` Epic 4 (Stories CRM-020 through CRM-021)
- **Reference Documents**: 
  - `fdd_1.md` (Functional Design Document for CRM, §4.3)
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

This document provides complete technical specifications for implementing interaction and communication logging APIs in Supabase. It covers:

- Log Interaction API for recording customer communications
- Fetch Interaction History API for retrieving communication timelines
- Request/response schemas with exact JSON structures
- Validation rules and error handling
- Support for manual logging and system-generated interactions
- Integration patterns for external systems (email/SMS webhooks)
- Sentiment analysis integration (async processing)
- Performance optimizations and query strategies

All specifications are designed to be directly implementable as Supabase Edge Functions (Deno/TypeScript) or PostgreSQL RPC functions, with exact schemas, validation rules, and error codes defined.

---

## 2. Architecture Decisions

### 2.1 Implementation Approach

**Decision**: Support both Edge Functions and PostgreSQL RPC functions:

- **Edge Functions** (`/crm/interactions`): Recommended for webhook integrations, external system calls, and async processing (sentiment analysis)
- **RPC Functions** (`crm_log_interaction()`, `crm_get_interaction_history()`): Alternative for direct database access, better performance for simple operations

**Rationale**: 
- Edge Functions provide better HTTP semantics, webhook handling, and integration with external services
- RPC functions provide lower latency and simpler deployment for database-only operations
- Frontend can choose based on use case

### 2.2 Authentication & Authorization

- All endpoints require authenticated Supabase user (JWT token) except for system-generated interactions
- System-generated interactions use service role key (never exposed to client)
- `org_id` is automatically derived from user's profile (via RLS helper functions from Epic 2)
- RLS policies enforce org-scoping automatically
- Role-based access control enforced via RLS (from Epic 2)

### 2.3 System-Generated Interactions

- System-generated interactions (e.g., from email/SMS webhooks) can be created without `created_by_user_id`
- Use service role key for system-generated interactions
- `direction` set to `'system_generated'` for automated interactions
- Documented pattern for webhook integrations

### 2.4 Sentiment Analysis

- Sentiment analysis is performed asynchronously after interaction creation
- Edge Function enqueues sentiment analysis job (or calls AI API directly)
- Sentiment is updated via separate endpoint or database trigger
- Implementation details deferred to Epic 8 (AI & Analytics Features)

---

## 3. Story CRM-020: Log Interaction API

### 3.1 Endpoint Specification

#### 3.1.1 Edge Function Endpoint

**Path**: `POST /crm/interactions`

**Method**: `POST`

**Authentication**: Required (Supabase JWT) OR Service Role Key (for system-generated)

**Content-Type**: `application/json`

#### 3.1.2 RPC Function Alternative

**Function Name**: `crm_log_interaction`

**Schema**: `public`

**Parameters**: JSONB input parameter

### 3.2 Request Schema

#### 3.2.1 Request Body Structure

```typescript
interface LogInteractionRequest {
  // Required fields
  customer_id: string; // UUID
  channel: 'phone_inbound' | 'phone_outbound' | 'email_inbound' | 'email_outbound' | 
           'sms_inbound' | 'sms_outbound' | 'portal_message' | 'note' | 'in_person';
  
  // Optional fields
  direction?: 'inbound' | 'outbound' | 'system_generated'; // Default: inferred from channel if possible
  subject?: string; // Max 500 characters
  summary?: string; // Max 2000 characters
  body?: string; // Full message body, no length limit
  metadata?: {
    // Provider-specific data
    message_id?: string; // Email message ID, SMS message SID, etc.
    thread_id?: string; // Email thread ID
    call_duration_seconds?: number; // For phone calls
    call_sid?: string; // Twilio call SID
    provider?: string; // e.g., 'sendgrid', 'twilio', 'system'
    [key: string]: any; // Additional provider-specific fields
  };
  occurred_at?: string; // ISO 8601 timestamp, default: now()
  location_id?: string; // UUID, optional location reference
  related_work_order_id?: string; // UUID, optional work order reference (future module)
  related_quote_id?: string; // UUID, optional quote reference (future module)
  
  // Options
  trigger_sentiment_analysis?: boolean; // Default: true for email/sms/portal_message channels
  created_by_user_id?: string; // UUID, optional (for system-generated interactions)
}
```

#### 3.2.2 Request Validation Rules

**Required Fields**:
- `customer_id` must be valid UUID and exist in `customers` table for user's org
- `channel` must be valid enum value

**Conditional Validation**:
- `direction` can be omitted; will be inferred from `channel` if possible:
  - `phone_inbound`, `email_inbound`, `sms_inbound` → `direction = 'inbound'`
  - `phone_outbound`, `email_outbound`, `sms_outbound` → `direction = 'outbound'`
  - `portal_message`, `note`, `in_person` → `direction` must be explicitly provided or defaults to `'system_generated'`
- `occurred_at` must be valid ISO 8601 timestamp
- `occurred_at` cannot be more than 1 hour in the future (allows slight scheduling flexibility)
- `occurred_at` cannot be more than 1 year in the past (prevents historical data errors)

**Format Validation**:
- `subject` max length: 500 characters
- `summary` max length: 2000 characters
- `body` no length limit (stored as TEXT)
- `location_id` must be valid UUID and belong to the customer
- `related_work_order_id` and `related_quote_id` are validated if provided (future modules)

**Business Rules**:
- `customer_id` must belong to the authenticated user's organization (enforced via RLS)
- `location_id` must belong to the specified customer (enforced via validation)
- System-generated interactions (`direction = 'system_generated'`) can omit `created_by_user_id`
- Manual interactions require authenticated user (implicit `created_by_user_id`)

### 3.3 Implementation: Edge Function

#### 3.3.1 Edge Function Structure

**File**: `supabase/functions/crm-interactions/index.ts`

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
    // Determine if using service role (for system-generated interactions)
    const authHeader = req.headers.get('Authorization');
    const isServiceRole = authHeader?.includes('Bearer') && 
                         authHeader.includes(Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') || '');
    
    let supabaseClient;
    let userId: string | null = null;
    let orgId: string | null = null;
    
    if (isServiceRole) {
      // Service role client (bypasses RLS)
      supabaseClient = createClient(
        Deno.env.get('SUPABASE_URL') ?? '',
        Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? '',
        {
          auth: {
            autoRefreshToken: false,
            persistSession: false
          }
        }
      );
      
      // For service role, org_id and user_id must be provided in request
      const body = await req.json();
      orgId = body.org_id;
      userId = body.created_by_user_id || null;
    } else {
      // Authenticated user client
      supabaseClient = createClient(
        Deno.env.get('SUPABASE_URL') ?? '',
        Deno.env.get('SUPABASE_ANON_KEY') ?? '',
        {
          global: {
            headers: { Authorization: authHeader! },
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
      
      userId = user.id;
      
      // Get user's org_id from profile
      const { data: profile, error: profileError } = await supabaseClient
        .from('profiles')
        .select('org_id')
        .eq('id', user.id)
        .single();

      if (profileError || !profile) {
        return new Response(
          JSON.stringify({ error: 'User profile not found' }),
          { status: 403, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
        );
      }
      
      orgId = profile.org_id;
    }

    // Parse request body
    const body = await req.json();

    // Validate request
    const validationError = validateLogInteractionRequest(body, orgId!);
    if (validationError) {
      return new Response(
        JSON.stringify({ error: validationError }),
        { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    }

    // Infer direction from channel if not provided
    const direction = body.direction || inferDirectionFromChannel(body.channel);

    // Prepare interaction data
    const interactionData = {
      org_id: orgId,
      customer_id: body.customer_id,
      location_id: body.location_id || null,
      related_work_order_id: body.related_work_order_id || null,
      related_quote_id: body.related_quote_id || null,
      channel: body.channel,
      direction: direction,
      subject: body.subject || null,
      summary: body.summary || null,
      body: body.body || null,
      metadata: body.metadata || null,
      created_by_user_id: userId,
      occurred_at: body.occurred_at || new Date().toISOString(),
    };

    // Insert interaction
    const { data: interaction, error: insertError } = await supabaseClient
      .from('crm_interactions')
      .insert(interactionData)
      .select()
      .single();

    if (insertError) {
      return new Response(
        JSON.stringify({ error: insertError.message }),
        { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    }

    // Trigger sentiment analysis if requested and applicable
    const shouldAnalyzeSentiment = body.trigger_sentiment_analysis !== false && 
                                   ['email_inbound', 'email_outbound', 'sms_inbound', 'sms_outbound', 'portal_message'].includes(body.channel) &&
                                   (body.body || body.summary);

    if (shouldAnalyzeSentiment) {
      // Enqueue sentiment analysis (async, non-blocking)
      // This will be implemented in Epic 8 (AI & Analytics)
      // For now, just log that it should be triggered
      console.log(`Sentiment analysis should be triggered for interaction ${interaction.id}`);
      
      // Optionally call Edge Function for sentiment analysis
      // await supabaseClient.functions.invoke('analyze-interaction-sentiment', {
      //   body: { interaction_id: interaction.id }
      // });
    }

    return new Response(
      JSON.stringify(interaction),
      { status: 201, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
});

function inferDirectionFromChannel(channel: string): string {
  const directionMap: Record<string, string> = {
    'phone_inbound': 'inbound',
    'phone_outbound': 'outbound',
    'email_inbound': 'inbound',
    'email_outbound': 'outbound',
    'sms_inbound': 'inbound',
    'sms_outbound': 'outbound',
    'portal_message': 'system_generated',
    'note': 'system_generated',
    'in_person': 'system_generated',
  };
  
  return directionMap[channel] || 'system_generated';
}

function validateLogInteractionRequest(body: any, orgId: string): string | null {
  // Required fields
  if (!body.customer_id || typeof body.customer_id !== 'string') {
    return 'customer_id is required and must be a valid UUID';
  }
  
  // Validate UUID format
  const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
  if (!uuidRegex.test(body.customer_id)) {
    return 'customer_id must be a valid UUID';
  }
  
  if (!body.channel || ![
    'phone_inbound', 'phone_outbound', 'email_inbound', 'email_outbound',
    'sms_inbound', 'sms_outbound', 'portal_message', 'note', 'in_person'
  ].includes(body.channel)) {
    return 'channel is required and must be a valid interaction channel';
  }
  
  // Validate occurred_at if provided
  if (body.occurred_at) {
    const occurredAt = new Date(body.occurred_at);
    if (isNaN(occurredAt.getTime())) {
      return 'occurred_at must be a valid ISO 8601 timestamp';
    }
    
    const now = new Date();
    const oneHourFromNow = new Date(now.getTime() + 60 * 60 * 1000);
    const oneYearAgo = new Date(now.getTime() - 365 * 24 * 60 * 60 * 1000);
    
    if (occurredAt > oneHourFromNow) {
      return 'occurred_at cannot be more than 1 hour in the future';
    }
    
    if (occurredAt < oneYearAgo) {
      return 'occurred_at cannot be more than 1 year in the past';
    }
  }
  
  // Validate subject length
  if (body.subject && body.subject.length > 500) {
    return 'subject must be 500 characters or less';
  }
  
  // Validate summary length
  if (body.summary && body.summary.length > 2000) {
    return 'summary must be 2000 characters or less';
  }
  
  // Validate location_id if provided
  if (body.location_id && !uuidRegex.test(body.location_id)) {
    return 'location_id must be a valid UUID';
  }
  
  // Validate related IDs if provided
  if (body.related_work_order_id && !uuidRegex.test(body.related_work_order_id)) {
    return 'related_work_order_id must be a valid UUID';
  }
  
  if (body.related_quote_id && !uuidRegex.test(body.related_quote_id)) {
    return 'related_quote_id must be a valid UUID';
  }
  
  return null; // Validation passed
}
```

### 3.4 Implementation: PostgreSQL RPC Function

#### 3.4.1 RPC Function Definition

**Function Name**: `crm_log_interaction`

**Parameters**:
- `p_interaction_data JSONB` - Interaction data (validated request body)

**Return Type**: `JSONB` - Created interaction record

**DDL**:

```sql
CREATE OR REPLACE FUNCTION crm_log_interaction(
  p_interaction_data JSONB
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_user_id UUID;
  v_org_id UUID;
  v_customer_id UUID;
  v_location_id UUID;
  v_direction interaction_direction_enum;
  v_occurred_at TIMESTAMPTZ;
  v_interaction_id UUID;
  v_result JSONB;
BEGIN
  -- Get authenticated user ID
  v_user_id := auth.uid();
  
  -- Get user's org_id from profile
  SELECT org_id INTO v_org_id
  FROM profiles
  WHERE id = v_user_id AND is_active = true;
  
  IF v_org_id IS NULL THEN
    RAISE EXCEPTION 'User not authenticated or inactive';
  END IF;
  
  -- Extract and validate customer_id
  v_customer_id := (p_interaction_data->>'customer_id')::UUID;
  
  IF v_customer_id IS NULL THEN
    RAISE EXCEPTION 'customer_id is required';
  END IF;
  
  -- Verify customer exists and belongs to org
  IF NOT EXISTS (
    SELECT 1 FROM customers
    WHERE id = v_customer_id AND org_id = v_org_id
  ) THEN
    RAISE EXCEPTION 'Customer not found or access denied';
  END IF;
  
  -- Validate channel
  IF p_interaction_data->>'channel' IS NULL THEN
    RAISE EXCEPTION 'channel is required';
  END IF;
  
  -- Infer direction from channel if not provided
  IF p_interaction_data->>'direction' IS NOT NULL THEN
    v_direction := (p_interaction_data->>'direction')::interaction_direction_enum;
  ELSE
    CASE p_interaction_data->>'channel'
      WHEN 'phone_inbound', 'email_inbound', 'sms_inbound' THEN
        v_direction := 'inbound';
      WHEN 'phone_outbound', 'email_outbound', 'sms_outbound' THEN
        v_direction := 'outbound';
      ELSE
        v_direction := 'system_generated';
    END CASE;
  END IF;
  
  -- Validate and set occurred_at
  IF p_interaction_data->>'occurred_at' IS NOT NULL THEN
    v_occurred_at := (p_interaction_data->>'occurred_at')::TIMESTAMPTZ;
    
    -- Validate occurred_at is not too far in future or past
    IF v_occurred_at > now() + interval '1 hour' THEN
      RAISE EXCEPTION 'occurred_at cannot be more than 1 hour in the future';
    END IF;
    
    IF v_occurred_at < now() - interval '1 year' THEN
      RAISE EXCEPTION 'occurred_at cannot be more than 1 year in the past';
    END IF;
  ELSE
    v_occurred_at := now();
  END IF;
  
  -- Validate location_id if provided
  IF p_interaction_data->>'location_id' IS NOT NULL THEN
    v_location_id := (p_interaction_data->>'location_id')::UUID;
    
    -- Verify location belongs to customer
    IF NOT EXISTS (
      SELECT 1 FROM customer_locations
      WHERE id = v_location_id
      AND customer_id = v_customer_id
      AND org_id = v_org_id
    ) THEN
      RAISE EXCEPTION 'Location not found or does not belong to customer';
    END IF;
  END IF;
  
  -- Validate subject and summary lengths
  IF p_interaction_data->>'subject' IS NOT NULL AND 
     length(p_interaction_data->>'subject') > 500 THEN
    RAISE EXCEPTION 'subject must be 500 characters or less';
  END IF;
  
  IF p_interaction_data->>'summary' IS NOT NULL AND 
     length(p_interaction_data->>'summary') > 2000 THEN
    RAISE EXCEPTION 'summary must be 2000 characters or less';
  END IF;
  
  -- Insert interaction
  INSERT INTO crm_interactions (
    org_id,
    customer_id,
    location_id,
    related_work_order_id,
    related_quote_id,
    channel,
    direction,
    subject,
    summary,
    body,
    metadata,
    created_by_user_id,
    occurred_at
  ) VALUES (
    v_org_id,
    v_customer_id,
    v_location_id,
    CASE WHEN p_interaction_data->>'related_work_order_id' IS NOT NULL 
         THEN (p_interaction_data->>'related_work_order_id')::UUID 
         ELSE NULL END,
    CASE WHEN p_interaction_data->>'related_quote_id' IS NOT NULL 
         THEN (p_interaction_data->>'related_quote_id')::UUID 
         ELSE NULL END,
    (p_interaction_data->>'channel')::interaction_channel_enum,
    v_direction,
    NULLIF(p_interaction_data->>'subject', ''),
    NULLIF(p_interaction_data->>'summary', ''),
    NULLIF(p_interaction_data->>'body', ''),
    p_interaction_data->'metadata',
    v_user_id,
    v_occurred_at
  )
  RETURNING id INTO v_interaction_id;
  
  -- Return created interaction
  SELECT jsonb_build_object(
    'id', ci.id,
    'org_id', ci.org_id,
    'customer_id', ci.customer_id,
    'location_id', ci.location_id,
    'related_work_order_id', ci.related_work_order_id,
    'related_quote_id', ci.related_quote_id,
    'channel', ci.channel,
    'direction', ci.direction,
    'subject', ci.subject,
    'summary', ci.summary,
    'body', ci.body,
    'metadata', ci.metadata,
    'sentiment', ci.sentiment,
    'created_by_user_id', ci.created_by_user_id,
    'occurred_at', ci.occurred_at,
    'created_at', ci.created_at
  )
  INTO v_result
  FROM crm_interactions ci
  WHERE ci.id = v_interaction_id;
  
  RETURN v_result;
  
EXCEPTION
  WHEN OTHERS THEN
    RAISE EXCEPTION 'Error logging interaction: %', SQLERRM;
END;
$$;
```

### 3.5 System-Generated Interaction Pattern

For system-generated interactions (e.g., webhooks), use service role key:

**Edge Function Pattern**:

```typescript
// Webhook handler for email/SMS providers
const supabaseAdmin = createClient(
  Deno.env.get('SUPABASE_URL')!,
  Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!,
  {
    auth: {
      autoRefreshToken: false,
      persistSession: false
    }
  }
);

// Determine org_id from customer_id
const { data: customer } = await supabaseAdmin
  .from('customers')
  .select('org_id')
  .eq('id', customerId)
  .single();

// Create interaction with service role
const { data: interaction } = await supabaseAdmin
  .from('crm_interactions')
  .insert({
    org_id: customer.org_id,
    customer_id: customerId,
    channel: 'email_inbound',
    direction: 'inbound',
    subject: emailSubject,
    body: emailBody,
    metadata: {
      message_id: emailMessageId,
      provider: 'sendgrid'
    },
    occurred_at: emailReceivedAt,
    // created_by_user_id is NULL for system-generated
  })
  .select()
  .single();
```

### 3.6 Response Schema

#### 3.6.1 Success Response (201 Created)

```typescript
interface LogInteractionResponse {
  id: string; // UUID
  org_id: string; // UUID
  customer_id: string; // UUID
  location_id?: string; // UUID
  related_work_order_id?: string; // UUID
  related_quote_id?: string; // UUID
  channel: string;
  direction: string;
  subject?: string;
  summary?: string;
  body?: string;
  metadata?: {
    [key: string]: any;
  };
  sentiment?: 'positive' | 'neutral' | 'negative';
  created_by_user_id?: string; // UUID, null for system-generated
  occurred_at: string; // ISO 8601 timestamp
  created_at: string; // ISO 8601 timestamp
}
```

#### 3.6.2 Error Responses

**400 Bad Request** (Validation Error):
```json
{
  "error": "customer_id is required and must be a valid UUID"
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
  "error": "Customer not found or access denied"
}
```

**500 Internal Server Error**:
```json
{
  "error": "Error logging interaction: <database error message>"
}
```

---

## 4. Story CRM-021: Fetch Interaction History API

### 4.1 Endpoint Specification

**Path**: `GET /crm/customers/:id/interactions`

**Method**: `GET`

**Authentication**: Required (Supabase JWT)

**Query Parameters**:
- `limit` (optional, default: 20, max: 100) - Number of interactions per page
- `offset` (optional, default: 0) - Pagination offset
- `channel` (optional) - Filter by channel (can be repeated for multiple channels)
- `sentiment` (optional) - Filter by sentiment: `positive`, `neutral`, `negative`
- `start_date` (optional) - ISO 8601 timestamp, filter interactions from this date onwards
- `end_date` (optional) - ISO 8601 timestamp, filter interactions up to this date
- `sort_order` (optional, default: `desc`) - Sort order: `asc`, `desc`

### 4.2 Response Schema

```typescript
interface GetInteractionHistoryResponse {
  data: Array<{
    id: string; // UUID
    customer_id: string; // UUID
    location_id?: string; // UUID
    related_work_order_id?: string; // UUID
    related_quote_id?: string; // UUID
    channel: string;
    direction?: string;
    subject?: string;
    summary?: string;
    body?: string; // May be truncated in list view
    metadata?: {
      [key: string]: any;
    };
    sentiment?: 'positive' | 'neutral' | 'negative';
    created_by_user_id?: string; // UUID
    created_by_user?: {
      id: string;
      first_name?: string;
      last_name?: string;
      email?: string;
    };
    occurred_at: string; // ISO 8601 timestamp
    created_at: string; // ISO 8601 timestamp
  }>;
  total: number;
  limit: number;
  offset: number;
  has_more: boolean;
}
```

### 4.3 Implementation: Edge Function

**File**: `supabase/functions/crm-interactions/index.ts` (add GET handler)

```typescript
// Add to existing Edge Function

if (req.method === 'GET') {
  const url = new URL(req.url);
  const pathParts = url.pathname.split('/');
  const customerId = pathParts[pathParts.length - 2] === 'customers' 
    ? pathParts[pathParts.length - 1] 
    : null;
  
  if (!customerId) {
    return new Response(
      JSON.stringify({ error: 'Customer ID is required' }),
      { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
  
  // Parse query parameters
  const searchParams = url.searchParams;
  const limit = Math.min(parseInt(searchParams.get('limit') || '20'), 100);
  const offset = parseInt(searchParams.get('offset') || '0');
  const channels = searchParams.getAll('channel');
  const sentiment = searchParams.get('sentiment');
  const startDate = searchParams.get('start_date');
  const endDate = searchParams.get('end_date');
  const sortOrder = searchParams.get('sort_order') || 'desc';
  
  // Get user's org_id
  const { data: profile } = await supabaseClient
    .from('profiles')
    .select('org_id')
    .eq('id', user.id)
    .single();
  
  // Call RPC function
  const { data: result, error: fetchError } = await supabaseClient.rpc(
    'crm_get_interaction_history',
    {
      p_customer_id: customerId,
      p_org_id: profile.org_id,
      p_limit: limit,
      p_offset: offset,
      p_channels: channels.length > 0 ? channels : null,
      p_sentiment: sentiment || null,
      p_start_date: startDate || null,
      p_end_date: endDate || null,
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
CREATE OR REPLACE FUNCTION crm_get_interaction_history(
  p_customer_id UUID,
  p_org_id UUID,
  p_limit INTEGER DEFAULT 20,
  p_offset INTEGER DEFAULT 0,
  p_channels TEXT[] DEFAULT NULL,
  p_sentiment TEXT DEFAULT NULL,
  p_start_date TIMESTAMPTZ DEFAULT NULL,
  p_end_date TIMESTAMPTZ DEFAULT NULL,
  p_sort_order TEXT DEFAULT 'desc'
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
  v_sort_direction TEXT;
BEGIN
  -- Get user's org_id
  SELECT org_id INTO v_user_org_id
  FROM profiles
  WHERE id = auth.uid() AND is_active = true;
  
  IF v_user_org_id IS NULL THEN
    RAISE EXCEPTION 'User not authenticated or inactive';
  END IF;
  
  -- Verify org_id matches (security check)
  IF v_user_org_id != p_org_id THEN
    RAISE EXCEPTION 'Access denied';
  END IF;
  
  -- Verify customer exists and belongs to org
  IF NOT EXISTS (
    SELECT 1 FROM customers
    WHERE id = p_customer_id AND org_id = p_org_id
  ) THEN
    RAISE EXCEPTION 'Customer not found or access denied';
  END IF;
  
  -- Validate and set limits
  IF p_limit > 100 THEN
    p_limit := 100;
  END IF;
  IF p_limit < 1 THEN
    p_limit := 20;
  END IF;
  
  -- Validate sort_order
  IF p_sort_order NOT IN ('asc', 'desc') THEN
    p_sort_order := 'desc';
  END IF;
  
  v_sort_direction := UPPER(p_sort_order);
  
  -- Get total count
  SELECT COUNT(*) INTO v_total
  FROM crm_interactions ci
  WHERE ci.customer_id = p_customer_id
  AND ci.org_id = p_org_id
  AND (p_channels IS NULL OR ci.channel = ANY(p_channels))
  AND (p_sentiment IS NULL OR ci.sentiment::TEXT = p_sentiment)
  AND (p_start_date IS NULL OR ci.occurred_at >= p_start_date)
  AND (p_end_date IS NULL OR ci.occurred_at <= p_end_date);
  
  -- Build result
  SELECT jsonb_build_object(
    'data', COALESCE(
      (
        SELECT jsonb_agg(jsonb_build_object(
          'id', ci.id,
          'customer_id', ci.customer_id,
          'location_id', ci.location_id,
          'related_work_order_id', ci.related_work_order_id,
          'related_quote_id', ci.related_quote_id,
          'channel', ci.channel,
          'direction', ci.direction,
          'subject', ci.subject,
          'summary', ci.summary,
          'body', ci.body,
          'metadata', ci.metadata,
          'sentiment', ci.sentiment,
          'created_by_user_id', ci.created_by_user_id,
          'created_by_user', CASE
            WHEN ci.created_by_user_id IS NOT NULL THEN jsonb_build_object(
              'id', p.id,
              'first_name', p.first_name,
              'last_name', p.last_name,
              'email', p.email
            )
            ELSE NULL
          END,
          'occurred_at', ci.occurred_at,
          'created_at', ci.created_at
        ) ORDER BY ci.occurred_at DESC)
        FROM crm_interactions ci
        LEFT JOIN profiles p ON p.id = ci.created_by_user_id
        WHERE ci.customer_id = p_customer_id
        AND ci.org_id = p_org_id
        AND (p_channels IS NULL OR ci.channel = ANY(p_channels))
        AND (p_sentiment IS NULL OR ci.sentiment::TEXT = p_sentiment)
        AND (p_start_date IS NULL OR ci.occurred_at >= p_start_date)
        AND (p_end_date IS NULL OR ci.occurred_at <= p_end_date)
        ORDER BY 
          CASE WHEN v_sort_direction = 'DESC' THEN ci.occurred_at END DESC,
          CASE WHEN v_sort_direction = 'ASC' THEN ci.occurred_at END ASC
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

### 4.5 Alternative: Direct Supabase Query (Frontend)

For simple use cases, frontend can query directly:

```typescript
// Frontend query example
const { data: interactions, error } = await supabase
  .from('crm_interactions')
  .select(`
    *,
    created_by_user:profiles!crm_interactions_created_by_user_id_fkey(
      id,
      first_name,
      last_name,
      email
    )
  `)
  .eq('customer_id', customerId)
  .order('occurred_at', { ascending: false })
  .range(offset, offset + limit - 1);
```

**Note**: RLS policies from Epic 2 automatically enforce org-scoping.

---

## 5. Webhook Integration Patterns

### 5.1 Email Webhook Integration

**Pattern**: Receive email webhook from provider (e.g., SendGrid, Mailgun)

**Edge Function**: `supabase/functions/webhook-email/index.ts`

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

serve(async (req) => {
  const supabaseAdmin = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!,
  );
  
  const webhookData = await req.json();
  
  // Extract email data from webhook
  const emailFrom = webhookData.from;
  const emailTo = webhookData.to;
  const emailSubject = webhookData.subject;
  const emailBody = webhookData.body;
  const emailMessageId = webhookData.message_id;
  const emailReceivedAt = webhookData.timestamp;
  
  // Find customer by email
  const { data: contact } = await supabaseAdmin
    .from('customer_contacts')
    .select('customer_id, org_id')
    .eq('type', 'email')
    .eq('value', emailTo)
    .single();
  
  if (!contact) {
    // Customer not found, handle accordingly
    return new Response(JSON.stringify({ error: 'Customer not found' }), { status: 404 });
  }
  
  // Create interaction
  const { data: interaction } = await supabaseAdmin
    .from('crm_interactions')
    .insert({
      org_id: contact.org_id,
      customer_id: contact.customer_id,
      channel: 'email_inbound',
      direction: 'inbound',
      subject: emailSubject,
      body: emailBody,
      metadata: {
        message_id: emailMessageId,
        from: emailFrom,
        provider: 'sendgrid'
      },
      occurred_at: emailReceivedAt,
    })
    .select()
    .single();
  
  return new Response(JSON.stringify({ success: true, interaction_id: interaction.id }), {
    status: 201,
  });
});
```

### 5.2 SMS Webhook Integration

**Pattern**: Receive SMS webhook from provider (e.g., Twilio)

**Edge Function**: `supabase/functions/webhook-sms/index.ts`

```typescript
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

serve(async (req) => {
  const supabaseAdmin = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!,
  );
  
  const formData = await req.formData();
  const phoneFrom = formData.get('From');
  const phoneTo = formData.get('To');
  const messageBody = formData.get('Body');
  const messageSid = formData.get('MessageSid');
  
  // Find customer by phone
  const { data: contact } = await supabaseAdmin
    .from('customer_contacts')
    .select('customer_id, org_id')
    .eq('type', 'mobile')
    .eq('value', phoneTo)
    .single();
  
  if (!contact) {
    return new Response(JSON.stringify({ error: 'Customer not found' }), { status: 404 });
  }
  
  // Create interaction
  const { data: interaction } = await supabaseAdmin
    .from('crm_interactions')
    .insert({
      org_id: contact.org_id,
      customer_id: contact.customer_id,
      channel: 'sms_inbound',
      direction: 'inbound',
      body: messageBody,
      metadata: {
        message_sid: messageSid,
        from: phoneFrom,
        provider: 'twilio'
      },
      occurred_at: new Date().toISOString(),
    })
    .select()
    .single();
  
  return new Response(JSON.stringify({ success: true, interaction_id: interaction.id }), {
    status: 201,
  });
});
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

- **200 OK**: Successful GET request
- **201 Created**: Successful POST request
- **400 Bad Request**: Validation error or invalid input
- **401 Unauthorized**: Missing or invalid authentication token
- **403 Forbidden**: User lacks permission (RLS blocked)
- **404 Not Found**: Resource not found
- **500 Internal Server Error**: Server error

### 6.3 Error Codes

- `VALIDATION_ERROR`: Request validation failed
- `NOT_FOUND`: Resource not found
- `UNAUTHORIZED`: Authentication required
- `FORBIDDEN`: Insufficient permissions
- `DATABASE_ERROR`: Database operation failed
- `CUSTOMER_NOT_FOUND`: Customer does not exist or access denied
- `LOCATION_NOT_FOUND`: Location does not belong to customer

---

## 7. Performance Considerations

### 7.1 Query Optimization

- Use indexes defined in Epic 1 for all filter and sort operations
- Index on `(customer_id, occurred_at DESC)` optimizes timeline queries
- Index on `(org_id, occurred_at DESC)` optimizes org-wide queries
- Limit result sets (default 20, max 100)
- Use pagination for large datasets

### 7.2 Caching Strategy

- Interaction history should not be cached (real-time data)
- Use Supabase real-time subscriptions for live updates
- Consider materialized views for aggregated interaction statistics (future)

### 7.3 Performance Targets

- Log Interaction: < 200ms
- Fetch Interaction History: < 300ms (for 20 interactions)
- Webhook processing: < 500ms (including customer lookup)

---

## 8. Testing Requirements

### 8.1 Unit Tests

- Request validation functions
- Direction inference logic
- Error handling paths
- Edge cases (empty metadata, null values, etc.)

### 8.2 Integration Tests

- End-to-end API calls
- RLS enforcement verification
- System-generated interaction creation
- Webhook integration flows
- Pagination and filtering

### 8.3 Performance Tests

- Load testing with realistic interaction volumes
- Query performance validation
- Concurrent request handling

---

## 9. Implementation Checklist

### Story CRM-020: Log Interaction API
- [ ] Edge Function implemented (`POST /crm/interactions`)
- [ ] RPC function implemented (`crm_log_interaction`)
- [ ] Request validation implemented
- [ ] Direction inference from channel
- [ ] Support for system-generated interactions (service role)
- [ ] Support for manual interactions (authenticated user)
- [ ] Location validation (belongs to customer)
- [ ] Timestamp validation (not too far in future/past)
- [ ] Error handling and rollback
- [ ] Response schema matches specification
- [ ] Tests written (happy path and error cases)
- [ ] API documentation with examples
- [ ] Webhook integration patterns documented

### Story CRM-021: Fetch Interaction History API
- [ ] Edge Function or RPC function implemented (`GET /crm/customers/:id/interactions`)
- [ ] Pagination support (limit/offset)
- [ ] Filter support (channel, sentiment, date range)
- [ ] Sort order support (asc/desc)
- [ ] User information included in response
- [ ] Performance validated
- [ ] Response schema matches specification
- [ ] Tests written
- [ ] API documentation with examples

---

## 10. Future Enhancements

### 10.1 Sentiment Analysis Integration

- Async sentiment analysis after interaction creation (Epic 8)
- Batch sentiment analysis for historical interactions
- Sentiment trends and analytics

### 10.2 Advanced Filtering

- Full-text search across subject/summary/body
- Filter by user who created interaction
- Filter by related work order or quote
- Date range presets (today, this week, this month, etc.)

### 10.3 Interaction Analytics

- Interaction volume over time
- Channel distribution
- Sentiment distribution
- Response time metrics
- Customer engagement scores

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 4 – Interaction & Communication Logging. All specifications are designed to be directly consumable by LLM-based code generators, with exact request/response schemas, validation rules, SQL functions, Edge Function implementations, and webhook integration patterns defined.

