# Technical Design Document – Epic 3: Customer Management APIs & Workflows

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 3 – Customer Management APIs & Workflows
- **Source**: Derived from `fdd_1_agile.md` Epic 3 (Stories CRM-016 through CRM-019)
- **Reference Documents**: 
  - `fdd_1.md` (Functional Design Document for CRM, §4.2)
  - `fdd_1_agile.md` (Agile User Stories)
  - `tdd_1_epic_1.md` (CRM Core Data Model - prerequisite)
  - `tdd_1_epic_2.md` (Authentication, Authorization & RLS Policies - prerequisite)
  - `functional.md` (Platform Capabilities)
  - `tooling.md` (Platform Choices: Supabase + Vercel)
- **Target Platform**: Supabase (PostgreSQL 15+ with Edge Functions)
- **Purpose**: Comprehensive technical specification for LLM-based code generation
- **Prerequisites**: Epic 1 (CRM Core Data Model) and Epic 2 (RLS Policies) must be completed first

---

## 1. Overview

This document provides complete technical specifications for implementing customer management APIs and workflows in Supabase. It covers:

- Create Customer API (Edge Function and RPC options)
- Update Customer API with nested data support
- Get Customer Details API (composed view)
- Search & List Customers API with filtering and pagination
- Request/response schemas with exact JSON structures
- Validation rules and error handling
- Transaction management for data consistency
- Performance optimizations and query strategies

All specifications are designed to be directly implementable as Supabase Edge Functions (Deno/TypeScript) or PostgreSQL RPC functions, with exact schemas, validation rules, and error codes defined.

---

## 2. Architecture Decisions

### 2.1 Implementation Approach

**Decision**: Support both Edge Functions and PostgreSQL RPC functions:

- **Edge Functions** (`/crm/customers`): Recommended for complex workflows, external integrations, and better error handling
- **RPC Functions** (`crm_create_customer()`): Alternative for simple operations, better performance for direct database access

**Rationale**: 
- Edge Functions provide better HTTP semantics, error handling, and integration with external services
- RPC functions provide lower latency and simpler deployment for database-only operations
- Frontend can choose based on use case

### 2.2 Authentication & Authorization

- All endpoints require authenticated Supabase user (JWT token)
- `org_id` is automatically derived from user's profile (via RLS helper functions from Epic 2)
- RLS policies enforce org-scoping automatically
- Role-based access control enforced via RLS (from Epic 2)

### 2.3 Transaction Management

- Multi-table operations (create customer with locations/contacts) use database transactions
- Edge Functions use Supabase client transactions
- RPC functions use PostgreSQL `BEGIN/COMMIT/ROLLBACK`
- Partial failures roll back entire operation

---

## 3. Story CRM-016: Create Customer API

### 3.1 Endpoint Specification

#### 3.1.1 Edge Function Endpoint

**Path**: `POST /crm/customers`

**Method**: `POST`

**Authentication**: Required (Supabase JWT)

**Content-Type**: `application/json`

#### 3.1.2 RPC Function Alternative

**Function Name**: `crm_create_customer`

**Schema**: `public`

**Parameters**: JSONB input parameter

### 3.2 Request Schema

#### 3.2.1 Request Body Structure

```typescript
interface CreateCustomerRequest {
  // Required fields
  type: 'individual' | 'company';
  name: string; // Full name or company name
  
  // Optional individual fields
  first_name?: string;
  last_name?: string;
  
  // Optional company fields
  company_name?: string;
  
  // Optional customer metadata
  external_ref?: string;
  status?: 'active' | 'prospect' | 'inactive' | 'blacklisted'; // Default: 'prospect'
  lifecycle_stage?: 'lead' | 'opportunity' | 'customer' | 'former_customer'; // Default: 'lead'
  source?: string; // e.g., 'web', 'phone', 'referral'
  preferred_language?: string; // ISO code, e.g., 'en', 'es'
  notes?: string;
  
  // Primary contact (optional but recommended)
  primary_contact?: {
    type: 'email' | 'mobile' | 'phone' | 'fax' | 'whatsapp' | 'telegram' | 'portal';
    value: string; // Email address or phone number
    is_verified?: boolean; // Default: false
    opt_in_marketing?: boolean; // Default: true
    opt_in_transactional?: boolean; // Default: true
    preferred_channel?: boolean; // Default: false
    notes?: string;
  };
  
  // Primary location (optional but recommended)
  primary_location?: {
    label?: string; // e.g., 'Home', 'Office'
    type: 'billing' | 'service' | 'both'; // Default: 'both'
    address_line1: string;
    address_line2?: string;
    city: string;
    state?: string;
    postal_code?: string;
    country?: string; // Default: 'US'
    latitude?: number; // Decimal degrees
    longitude?: number; // Decimal degrees
  };
  
  // Initial tags (optional)
  tags?: Array<{
    tag_id?: string; // UUID of existing tag
    tag_name?: string; // Name of tag (will be created if doesn't exist)
  }>;
}
```

#### 3.2.2 Request Validation Rules

**Required Fields**:
- `type` must be `'individual'` or `'company'`
- `name` must be non-empty string (max 255 characters)

**Conditional Validation**:
- If `type = 'individual'`: `first_name` and `last_name` should be provided (recommended, not enforced at API level if business allows)
- If `type = 'company'`: `company_name` should be provided (recommended)

**Format Validation**:
- `primary_contact.value`:
  - If `type = 'email'`: Must match email regex `^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$`
  - If `type IN ('mobile', 'phone', 'fax')`: Must match E.164 format `^\+?[1-9]\d{1,14}$`
- `primary_location`:
  - `address_line1` required if `primary_location` provided
  - `city` required if `primary_location` provided
  - `latitude` and `longitude` must both be provided or both omitted
  - `latitude` must be between -90 and 90
  - `longitude` must be between -180 and 180

**Tag Validation**:
- Either `tag_id` or `tag_name` must be provided (not both)
- `tag_id` must be valid UUID and exist in `crm_tags` for user's org
- `tag_name` will be created if doesn't exist (max 50 characters)

### 3.3 Implementation: Edge Function

#### 3.3.1 Edge Function Structure

**File**: `supabase/functions/crm-customers/index.ts`

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

    // Parse request body
    const body = await req.json();

    // Validate request
    const validationError = validateCreateCustomerRequest(body);
    if (validationError) {
      return new Response(
        JSON.stringify({ error: validationError }),
        { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
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

    // Create customer using RPC function (or inline transaction)
    const { data: customer, error: createError } = await supabaseClient.rpc(
      'crm_create_customer',
      {
        p_org_id: orgId,
        p_customer_data: body,
      }
    );

    if (createError) {
      return new Response(
        JSON.stringify({ error: createError.message }),
        { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
      );
    }

    return new Response(
      JSON.stringify(customer),
      { status: 201, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 500, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
});
```

#### 3.3.2 Validation Function

```typescript
function validateCreateCustomerRequest(body: any): string | null {
  // Required fields
  if (!body.type || !['individual', 'company'].includes(body.type)) {
    return 'type is required and must be "individual" or "company"';
  }
  
  if (!body.name || typeof body.name !== 'string' || body.name.trim().length === 0) {
    return 'name is required and must be a non-empty string';
  }
  
  if (body.name.length > 255) {
    return 'name must be 255 characters or less';
  }
  
  // Validate primary contact if provided
  if (body.primary_contact) {
    if (!body.primary_contact.type || !body.primary_contact.value) {
      return 'primary_contact.type and primary_contact.value are required if primary_contact is provided';
    }
    
    const contactType = body.primary_contact.type;
    const contactValue = body.primary_contact.value;
    
    if (contactType === 'email') {
      const emailRegex = /^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$/;
      if (!emailRegex.test(contactValue)) {
        return 'primary_contact.value must be a valid email address';
      }
    } else if (['mobile', 'phone', 'fax'].includes(contactType)) {
      const phoneRegex = /^\+?[1-9]\d{1,14}$/;
      if (!phoneRegex.test(contactValue)) {
        return 'primary_contact.value must be a valid phone number in E.164 format';
      }
    }
  }
  
  // Validate primary location if provided
  if (body.primary_location) {
    if (!body.primary_location.address_line1 || !body.primary_location.city) {
      return 'primary_location.address_line1 and primary_location.city are required if primary_location is provided';
    }
    
    if (body.primary_location.latitude !== undefined || body.primary_location.longitude !== undefined) {
      if (body.primary_location.latitude === undefined || body.primary_location.longitude === undefined) {
        return 'latitude and longitude must both be provided or both omitted';
      }
      
      if (body.primary_location.latitude < -90 || body.primary_location.latitude > 90) {
        return 'latitude must be between -90 and 90';
      }
      
      if (body.primary_location.longitude < -180 || body.primary_location.longitude > 180) {
        return 'longitude must be between -180 and 180';
      }
    }
  }
  
  // Validate tags if provided
  if (body.tags && Array.isArray(body.tags)) {
    for (const tag of body.tags) {
      if (!tag.tag_id && !tag.tag_name) {
        return 'Each tag must have either tag_id or tag_name';
      }
      if (tag.tag_id && tag.tag_name) {
        return 'Each tag cannot have both tag_id and tag_name';
      }
      if (tag.tag_name && tag.tag_name.length > 50) {
        return 'tag_name must be 50 characters or less';
      }
    }
  }
  
  return null; // Validation passed
}
```

### 3.4 Implementation: PostgreSQL RPC Function

#### 3.4.1 RPC Function Definition

**Function Name**: `crm_create_customer`

**Parameters**:
- `p_org_id UUID` - Organization ID (derived from authenticated user)
- `p_customer_data JSONB` - Customer data (validated request body)

**Return Type**: `JSONB` - Created customer with nested data

**DDL**:

```sql
CREATE OR REPLACE FUNCTION crm_create_customer(
  p_org_id UUID,
  p_customer_data JSONB
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_customer_id UUID;
  v_location_id UUID;
  v_contact_id UUID;
  v_tag_id UUID;
  v_tag_record RECORD;
  v_result JSONB;
  v_created_by_user_id UUID;
BEGIN
  -- Get authenticated user ID
  v_created_by_user_id := auth.uid();
  
  -- Validate org_id matches user's org (security check)
  IF NOT EXISTS (
    SELECT 1 FROM profiles
    WHERE id = v_created_by_user_id
    AND org_id = p_org_id
    AND is_active = true
  ) THEN
    RAISE EXCEPTION 'User does not belong to specified organization';
  END IF;
  
  -- Begin transaction (implicit in function, but explicit for clarity)
  BEGIN
    -- Insert customer
    INSERT INTO customers (
      org_id,
      type,
      name,
      first_name,
      last_name,
      company_name,
      external_ref,
      status,
      lifecycle_stage,
      source,
      preferred_language,
      notes,
      email, -- Set from primary contact if email type
      phone  -- Set from primary contact if phone type
    ) VALUES (
      p_org_id,
      (p_customer_data->>'type')::customer_type_enum,
      p_customer_data->>'name',
      NULLIF(p_customer_data->>'first_name', ''),
      NULLIF(p_customer_data->>'last_name', ''),
      NULLIF(p_customer_data->>'company_name', ''),
      NULLIF(p_customer_data->>'external_ref', ''),
      COALESCE((p_customer_data->>'status')::customer_status_enum, 'prospect'),
      COALESCE((p_customer_data->>'lifecycle_stage')::customer_lifecycle_stage_enum, 'lead'),
      NULLIF(p_customer_data->>'source', ''),
      NULLIF(p_customer_data->>'preferred_language', ''),
      NULLIF(p_customer_data->>'notes', ''),
      CASE 
        WHEN p_customer_data->'primary_contact'->>'type' = 'email' 
        THEN p_customer_data->'primary_contact'->>'value'
        ELSE NULL
      END,
      CASE 
        WHEN p_customer_data->'primary_contact'->>'type' IN ('mobile', 'phone', 'fax')
        THEN p_customer_data->'primary_contact'->>'value'
        ELSE NULL
      END
    )
    RETURNING id INTO v_customer_id;
    
    -- Create primary location if provided
    IF p_customer_data->'primary_location' IS NOT NULL THEN
      INSERT INTO customer_locations (
        org_id,
        customer_id,
        label,
        type,
        address_line1,
        address_line2,
        city,
        state,
        postal_code,
        country,
        latitude,
        longitude,
        is_primary
      ) VALUES (
        p_org_id,
        v_customer_id,
        NULLIF(p_customer_data->'primary_location'->>'label', ''),
        COALESCE((p_customer_data->'primary_location'->>'type')::location_type_enum, 'both'),
        p_customer_data->'primary_location'->>'address_line1',
        NULLIF(p_customer_data->'primary_location'->>'address_line2', ''),
        p_customer_data->'primary_location'->>'city',
        NULLIF(p_customer_data->'primary_location'->>'state', ''),
        NULLIF(p_customer_data->'primary_location'->>'postal_code', ''),
        COALESCE(NULLIF(p_customer_data->'primary_location'->>'country', ''), 'US'),
        (p_customer_data->'primary_location'->>'latitude')::NUMERIC,
        (p_customer_data->'primary_location'->>'longitude')::NUMERIC,
        true
      )
      RETURNING id INTO v_location_id;
      
      -- Update customer with primary_location_id
      UPDATE customers
      SET primary_location_id = v_location_id
      WHERE id = v_customer_id;
    END IF;
    
    -- Create primary contact if provided
    IF p_customer_data->'primary_contact' IS NOT NULL THEN
      INSERT INTO customer_contacts (
        org_id,
        customer_id,
        type,
        value,
        is_primary,
        is_verified,
        opt_in_marketing,
        opt_in_transactional,
        preferred_channel,
        notes
      ) VALUES (
        p_org_id,
        v_customer_id,
        (p_customer_data->'primary_contact'->>'type')::contact_type_enum,
        p_customer_data->'primary_contact'->>'value',
        true,
        COALESCE((p_customer_data->'primary_contact'->>'is_verified')::BOOLEAN, false),
        COALESCE((p_customer_data->'primary_contact'->>'opt_in_marketing')::BOOLEAN, true),
        COALESCE((p_customer_data->'primary_contact'->>'opt_in_transactional')::BOOLEAN, true),
        COALESCE((p_customer_data->'primary_contact'->>'preferred_channel')::BOOLEAN, false),
        NULLIF(p_customer_data->'primary_contact'->>'notes', '')
      )
      RETURNING id INTO v_contact_id;
      
      -- Update customer with primary_contact_id
      UPDATE customers
      SET primary_contact_id = v_contact_id
      WHERE id = v_customer_id;
    END IF;
    
    -- Create default preferences
    INSERT INTO crm_preferences (
      org_id,
      customer_id,
      do_not_contact,
      do_not_email,
      do_not_sms,
      do_not_call
    ) VALUES (
      p_org_id,
      v_customer_id,
      false,
      false,
      false,
      false
    );
    
    -- Create/link tags if provided
    IF p_customer_data->'tags' IS NOT NULL THEN
      FOR v_tag_record IN 
        SELECT * FROM jsonb_array_elements(p_customer_data->'tags')
      LOOP
        -- Resolve tag ID
        IF v_tag_record.value->>'tag_id' IS NOT NULL THEN
          v_tag_id := (v_tag_record.value->>'tag_id')::UUID;
          
          -- Verify tag exists and belongs to org
          IF NOT EXISTS (
            SELECT 1 FROM crm_tags
            WHERE id = v_tag_id AND org_id = p_org_id
          ) THEN
            RAISE EXCEPTION 'Tag % does not exist or does not belong to organization', v_tag_id;
          END IF;
        ELSIF v_tag_record.value->>'tag_name' IS NOT NULL THEN
          -- Find or create tag
          SELECT id INTO v_tag_id
          FROM crm_tags
          WHERE org_id = p_org_id
          AND name = v_tag_record.value->>'tag_name';
          
          IF v_tag_id IS NULL THEN
            INSERT INTO crm_tags (org_id, name)
            VALUES (p_org_id, v_tag_record.value->>'tag_name')
            RETURNING id INTO v_tag_id;
          END IF;
        END IF;
        
        -- Link tag to customer (ignore if already linked)
        INSERT INTO crm_customer_tags (org_id, customer_id, tag_id, assigned_by_user_id)
        VALUES (p_org_id, v_customer_id, v_tag_id, v_created_by_user_id)
        ON CONFLICT (org_id, customer_id, tag_id) DO NOTHING;
      END LOOP;
    END IF;
    
    -- Build and return result
    SELECT jsonb_build_object(
      'id', c.id,
      'org_id', c.org_id,
      'type', c.type,
      'name', c.name,
      'first_name', c.first_name,
      'last_name', c.last_name,
      'company_name', c.company_name,
      'status', c.status,
      'lifecycle_stage', c.lifecycle_stage,
      'created_at', c.created_at,
      'primary_location', CASE 
        WHEN cl.id IS NOT NULL THEN jsonb_build_object(
          'id', cl.id,
          'label', cl.label,
          'type', cl.type,
          'address_line1', cl.address_line1,
          'address_line2', cl.address_line2,
          'city', cl.city,
          'state', cl.state,
          'postal_code', cl.postal_code,
          'country', cl.country,
          'latitude', cl.latitude,
          'longitude', cl.longitude
        )
        ELSE NULL
      END,
      'primary_contact', CASE
        WHEN cc.id IS NOT NULL THEN jsonb_build_object(
          'id', cc.id,
          'type', cc.type,
          'value', cc.value,
          'is_verified', cc.is_verified,
          'opt_in_marketing', cc.opt_in_marketing,
          'opt_in_transactional', cc.opt_in_transactional
        )
        ELSE NULL
      END,
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
      )
    )
    INTO v_result
    FROM customers c
    LEFT JOIN customer_locations cl ON cl.id = c.primary_location_id
    LEFT JOIN customer_contacts cc ON cc.id = c.primary_contact_id
    WHERE c.id = v_customer_id;
    
    RETURN v_result;
    
  EXCEPTION
    WHEN OTHERS THEN
      -- Rollback is automatic in function
      RAISE EXCEPTION 'Error creating customer: %', SQLERRM;
  END;
END;
$$;
```

### 3.5 Response Schema

#### 3.5.1 Success Response (201 Created)

```typescript
interface CreateCustomerResponse {
  id: string; // UUID
  org_id: string; // UUID
  type: 'individual' | 'company';
  name: string;
  first_name?: string;
  last_name?: string;
  company_name?: string;
  status: 'active' | 'prospect' | 'inactive' | 'blacklisted';
  lifecycle_stage: 'lead' | 'opportunity' | 'customer' | 'former_customer';
  created_at: string; // ISO 8601 timestamp
  primary_location?: {
    id: string; // UUID
    label?: string;
    type: 'billing' | 'service' | 'both';
    address_line1: string;
    address_line2?: string;
    city: string;
    state?: string;
    postal_code?: string;
    country: string;
    latitude?: number;
    longitude?: number;
  };
  primary_contact?: {
    id: string; // UUID
    type: string;
    value: string;
    is_verified: boolean;
    opt_in_marketing: boolean;
    opt_in_transactional: boolean;
  };
  tags: Array<{
    id: string; // UUID
    name: string;
    color?: string;
  }>;
}
```

#### 3.5.2 Error Responses

**400 Bad Request** (Validation Error):
```json
{
  "error": "type is required and must be \"individual\" or \"company\""
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
  "error": "Error creating customer: <database error message>"
}
```

---

## 4. Story CRM-017: Update Customer API

### 4.1 Endpoint Specification

**Path**: `PATCH /crm/customers/:id`

**Method**: `PATCH`

**Authentication**: Required (Supabase JWT)

**Content-Type**: `application/json`

### 4.2 Request Schema

```typescript
interface UpdateCustomerRequest {
  // Core customer fields (all optional)
  name?: string;
  first_name?: string;
  last_name?: string;
  company_name?: string;
  external_ref?: string;
  status?: 'active' | 'prospect' | 'inactive' | 'blacklisted';
  lifecycle_stage?: 'lead' | 'opportunity' | 'customer' | 'former_customer';
  source?: string;
  preferred_language?: string;
  notes?: string;
  
  // Nested updates
  locations?: {
    create?: Array<{
      label?: string;
      type: 'billing' | 'service' | 'both';
      address_line1: string;
      address_line2?: string;
      city: string;
      state?: string;
      postal_code?: string;
      country?: string;
      latitude?: number;
      longitude?: number;
      is_primary?: boolean;
    }>;
    update?: Array<{
      id: string; // UUID
      label?: string;
      type?: 'billing' | 'service' | 'both';
      address_line1?: string;
      address_line2?: string;
      city?: string;
      state?: string;
      postal_code?: string;
      country?: string;
      latitude?: number;
      longitude?: number;
      is_primary?: boolean;
    }>;
    delete?: Array<string>; // Array of location IDs (UUIDs)
  };
  
  contacts?: {
    create?: Array<{
      type: 'email' | 'mobile' | 'phone' | 'fax' | 'whatsapp' | 'telegram' | 'portal';
      value: string;
      is_primary?: boolean;
      is_verified?: boolean;
      opt_in_marketing?: boolean;
      opt_in_transactional?: boolean;
      preferred_channel?: boolean;
      notes?: string;
    }>;
    update?: Array<{
      id: string; // UUID
      type?: string;
      value?: string;
      is_primary?: boolean;
      is_verified?: boolean;
      opt_in_marketing?: boolean;
      opt_in_transactional?: boolean;
      preferred_channel?: boolean;
      notes?: string;
    }>;
    delete?: Array<string>; // Array of contact IDs (UUIDs)
  };
  
  // Options
  log_interaction?: boolean; // Default: false
  interaction_summary?: string; // Required if log_interaction = true
}
```

### 4.3 Implementation: Edge Function

**File**: `supabase/functions/crm-customers/index.ts` (add PATCH handler)

```typescript
// Add to existing Edge Function

if (req.method === 'PATCH') {
  const url = new URL(req.url);
  const customerId = url.pathname.split('/').pop();
  
  if (!customerId) {
    return new Response(
      JSON.stringify({ error: 'Customer ID is required' }),
      { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
  
  const body = await req.json();
  
  // Validate request
  const validationError = validateUpdateCustomerRequest(body);
  if (validationError) {
    return new Response(
      JSON.stringify({ error: validationError }),
      { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
  
  // Get user's org_id
  const { data: profile } = await supabaseClient
    .from('profiles')
    .select('org_id')
    .eq('id', user.id)
    .single();
  
  // Call RPC function
  const { data: customer, error: updateError } = await supabaseClient.rpc(
    'crm_update_customer',
    {
      p_customer_id: customerId,
      p_org_id: profile.org_id,
      p_update_data: body,
    }
  );
  
  if (updateError) {
    return new Response(
      JSON.stringify({ error: updateError.message }),
      { status: 400, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
    );
  }
  
  return new Response(
    JSON.stringify(customer),
    { status: 200, headers: { ...corsHeaders, 'Content-Type': 'application/json' } }
  );
}
```

### 4.4 Implementation: PostgreSQL RPC Function

```sql
CREATE OR REPLACE FUNCTION crm_update_customer(
  p_customer_id UUID,
  p_org_id UUID,
  p_update_data JSONB
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_user_id UUID;
  v_location_record RECORD;
  v_contact_record RECORD;
  v_result JSONB;
  v_changed_fields TEXT[];
BEGIN
  v_user_id := auth.uid();
  
  -- Verify customer exists and belongs to org
  IF NOT EXISTS (
    SELECT 1 FROM customers
    WHERE id = p_customer_id AND org_id = p_org_id
  ) THEN
    RAISE EXCEPTION 'Customer not found or access denied';
  END IF;
  
  BEGIN
    -- Update core customer fields
    UPDATE customers
    SET
      name = COALESCE(p_update_data->>'name', name),
      first_name = CASE WHEN p_update_data ? 'first_name' THEN NULLIF(p_update_data->>'first_name', '') ELSE first_name END,
      last_name = CASE WHEN p_update_data ? 'last_name' THEN NULLIF(p_update_data->>'last_name', '') ELSE last_name END,
      company_name = CASE WHEN p_update_data ? 'company_name' THEN NULLIF(p_update_data->>'company_name', '') ELSE company_name END,
      external_ref = CASE WHEN p_update_data ? 'external_ref' THEN NULLIF(p_update_data->>'external_ref', '') ELSE external_ref END,
      status = COALESCE((p_update_data->>'status')::customer_status_enum, status),
      lifecycle_stage = COALESCE((p_update_data->>'lifecycle_stage')::customer_lifecycle_stage_enum, lifecycle_stage),
      source = CASE WHEN p_update_data ? 'source' THEN NULLIF(p_update_data->>'source', '') ELSE source END,
      preferred_language = CASE WHEN p_update_data ? 'preferred_language' THEN NULLIF(p_update_data->>'preferred_language', '') ELSE preferred_language END,
      notes = CASE WHEN p_update_data ? 'notes' THEN NULLIF(p_update_data->>'notes', '') ELSE notes END,
      updated_at = now()
    WHERE id = p_customer_id;
    
    -- Handle locations
    IF p_update_data->'locations' IS NOT NULL THEN
      -- Create new locations
      IF p_update_data->'locations'->'create' IS NOT NULL THEN
        FOR v_location_record IN 
          SELECT * FROM jsonb_array_elements(p_update_data->'locations'->'create')
        LOOP
          INSERT INTO customer_locations (
            org_id, customer_id, label, type, address_line1, address_line2,
            city, state, postal_code, country, latitude, longitude, is_primary
          ) VALUES (
            p_org_id, p_customer_id,
            NULLIF(v_location_record.value->>'label', ''),
            (v_location_record.value->>'type')::location_type_enum,
            v_location_record.value->>'address_line1',
            NULLIF(v_location_record.value->>'address_line2', ''),
            v_location_record.value->>'city',
            NULLIF(v_location_record.value->>'state', ''),
            NULLIF(v_location_record.value->>'postal_code', ''),
            COALESCE(NULLIF(v_location_record.value->>'country', ''), 'US'),
            (v_location_record.value->>'latitude')::NUMERIC,
            (v_location_record.value->>'longitude')::NUMERIC,
            COALESCE((v_location_record.value->>'is_primary')::BOOLEAN, false)
          );
        END LOOP;
      END IF;
      
      -- Update existing locations
      IF p_update_data->'locations'->'update' IS NOT NULL THEN
        FOR v_location_record IN 
          SELECT * FROM jsonb_array_elements(p_update_data->'locations'->'update')
        LOOP
          UPDATE customer_locations
          SET
            label = COALESCE(NULLIF(v_location_record.value->>'label', ''), label),
            type = COALESCE((v_location_record.value->>'type')::location_type_enum, type),
            address_line1 = COALESCE(v_location_record.value->>'address_line1', address_line1),
            address_line2 = CASE WHEN v_location_record.value ? 'address_line2' THEN NULLIF(v_location_record.value->>'address_line2', '') ELSE address_line2 END,
            city = COALESCE(v_location_record.value->>'city', city),
            state = CASE WHEN v_location_record.value ? 'state' THEN NULLIF(v_location_record.value->>'state', '') ELSE state END,
            postal_code = CASE WHEN v_location_record.value ? 'postal_code' THEN NULLIF(v_location_record.value->>'postal_code', '') ELSE postal_code END,
            country = CASE WHEN v_location_record.value ? 'country' THEN NULLIF(v_location_record.value->>'country', '') ELSE country END,
            latitude = CASE WHEN v_location_record.value ? 'latitude' THEN (v_location_record.value->>'latitude')::NUMERIC ELSE latitude END,
            longitude = CASE WHEN v_location_record.value ? 'longitude' THEN (v_location_record.value->>'longitude')::NUMERIC ELSE longitude END,
            is_primary = COALESCE((v_location_record.value->>'is_primary')::BOOLEAN, is_primary),
            updated_at = now()
          WHERE id = (v_location_record.value->>'id')::UUID
          AND org_id = p_org_id
          AND customer_id = p_customer_id;
        END LOOP;
      END IF;
      
      -- Delete locations
      IF p_update_data->'locations'->'delete' IS NOT NULL THEN
        DELETE FROM customer_locations
        WHERE id = ANY(
          SELECT jsonb_array_elements_text(p_update_data->'locations'->'delete')::UUID
        )
        AND org_id = p_org_id
        AND customer_id = p_customer_id;
      END IF;
    END IF;
    
    -- Handle contacts (similar pattern to locations)
    IF p_update_data->'contacts' IS NOT NULL THEN
      -- Create, update, delete contacts (similar to locations logic)
      -- ... (implementation similar to locations)
    END IF;
    
    -- Log interaction if requested
    IF COALESCE((p_update_data->>'log_interaction')::BOOLEAN, false) THEN
      IF p_update_data->>'interaction_summary' IS NULL THEN
        RAISE EXCEPTION 'interaction_summary is required when log_interaction is true';
      END IF;
      
      INSERT INTO crm_interactions (
        org_id, customer_id, channel, direction, summary, created_by_user_id, occurred_at
      ) VALUES (
        p_org_id, p_customer_id, 'note', 'system_generated',
        p_update_data->>'interaction_summary', v_user_id, now()
      );
    END IF;
    
    -- Return updated customer (similar to create function)
    SELECT jsonb_build_object(
      'id', c.id,
      'name', c.name,
      'updated_at', c.updated_at
      -- ... (include all fields)
    )
    INTO v_result
    FROM customers c
    WHERE c.id = p_customer_id;
    
    RETURN v_result;
    
  EXCEPTION
    WHEN OTHERS THEN
      RAISE EXCEPTION 'Error updating customer: %', SQLERRM;
  END;
END;
$$;
```

### 4.5 Response Schema

**Success Response (200 OK)**: Same structure as CreateCustomerResponse

**Error Responses**: Same as Create Customer API

---

## 5. Story CRM-018: Get Customer Details API

### 5.1 Endpoint Specification

**Path**: `GET /crm/customers/:id`

**Method**: `GET`

**Authentication**: Required (Supabase JWT)

**Query Parameters**:
- `interactions_limit` (optional, default: 10, max: 100) - Number of recent interactions to return
- `interactions_offset` (optional, default: 0) - Offset for interactions pagination
- `followups_limit` (optional, default: 10, max: 100) - Number of upcoming follow-ups to return
- `followups_offset` (optional, default: 0) - Offset for follow-ups pagination

### 5.2 Response Schema

```typescript
interface GetCustomerDetailsResponse {
  // Core customer fields
  id: string;
  org_id: string;
  type: 'individual' | 'company';
  name: string;
  first_name?: string;
  last_name?: string;
  company_name?: string;
  external_ref?: string;
  email?: string;
  phone?: string;
  status: string;
  lifecycle_stage: string;
  source?: string;
  preferred_language?: string;
  notes?: string;
  created_at: string;
  updated_at: string;
  
  // Primary location
  primary_location?: {
    id: string;
    label?: string;
    type: string;
    address_line1: string;
    address_line2?: string;
    city: string;
    state?: string;
    postal_code?: string;
    country: string;
    latitude?: number;
    longitude?: number;
    is_primary: boolean;
  };
  
  // All locations
  locations: Array<{
    id: string;
    label?: string;
    type: string;
    address_line1: string;
    address_line2?: string;
    city: string;
    state?: string;
    postal_code?: string;
    country: string;
    latitude?: number;
    longitude?: number;
    is_primary: boolean;
    created_at: string;
    updated_at: string;
  }>;
  
  // All contacts
  contacts: Array<{
    id: string;
    type: string;
    value: string;
    is_primary: boolean;
    is_verified: boolean;
    opt_in_marketing: boolean;
    opt_in_transactional: boolean;
    preferred_channel: boolean;
    notes?: string;
    created_at: string;
    updated_at: string;
  }>;
  
  // Preferences
  preferences?: {
    do_not_contact: boolean;
    do_not_email: boolean;
    do_not_sms: boolean;
    do_not_call: boolean;
    preferred_contact_window_start?: string; // TIME format
    preferred_contact_window_end?: string; // TIME format
    notes?: string;
  };
  
  // Recent interactions (paginated)
  interactions: {
    data: Array<{
      id: string;
      channel: string;
      direction?: string;
      subject?: string;
      summary?: string;
      sentiment?: string;
      occurred_at: string;
      created_by_user_id?: string;
    }>;
    total: number;
    limit: number;
    offset: number;
  };
  
  // Upcoming follow-ups (paginated)
  followups: {
    data: Array<{
      id: string;
      title: string;
      description?: string;
      due_at: string;
      status: string;
      priority: string;
      assigned_to_user_id?: string;
    }>;
    total: number;
    limit: number;
    offset: number;
  };
  
  // Tags
  tags: Array<{
    id: string;
    name: string;
    color?: string;
  }>;
}
```

### 5.3 Implementation: PostgreSQL RPC Function

```sql
CREATE OR REPLACE FUNCTION crm_get_customer_details(
  p_customer_id UUID,
  p_interactions_limit INTEGER DEFAULT 10,
  p_interactions_offset INTEGER DEFAULT 0,
  p_followups_limit INTEGER DEFAULT 10,
  p_followups_offset INTEGER DEFAULT 0
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_org_id UUID;
  v_result JSONB;
BEGIN
  -- Get user's org_id
  SELECT org_id INTO v_org_id
  FROM profiles
  WHERE id = auth.uid() AND is_active = true;
  
  IF v_org_id IS NULL THEN
    RAISE EXCEPTION 'User not authenticated or inactive';
  END IF;
  
  -- Validate limits
  IF p_interactions_limit > 100 THEN
    p_interactions_limit := 100;
  END IF;
  IF p_followups_limit > 100 THEN
    p_followups_limit := 100;
  END IF;
  
  -- Build comprehensive customer view
  SELECT jsonb_build_object(
    -- Core customer
    'id', c.id,
    'org_id', c.org_id,
    'type', c.type,
    'name', c.name,
    'first_name', c.first_name,
    'last_name', c.last_name,
    'company_name', c.company_name,
    'external_ref', c.external_ref,
    'email', c.email,
    'phone', c.phone,
    'status', c.status,
    'lifecycle_stage', c.lifecycle_stage,
    'source', c.source,
    'preferred_language', c.preferred_language,
    'notes', c.notes,
    'created_at', c.created_at,
    'updated_at', c.updated_at,
    
    -- Primary location
    'primary_location', CASE
      WHEN pl.id IS NOT NULL THEN jsonb_build_object(
        'id', pl.id,
        'label', pl.label,
        'type', pl.type,
        'address_line1', pl.address_line1,
        'address_line2', pl.address_line2,
        'city', pl.city,
        'state', pl.state,
        'postal_code', pl.postal_code,
        'country', pl.country,
        'latitude', pl.latitude,
        'longitude', pl.longitude,
        'is_primary', pl.is_primary
      )
      ELSE NULL
    END,
    
    -- All locations
    'locations', COALESCE(
      (
        SELECT jsonb_agg(jsonb_build_object(
          'id', cl.id,
          'label', cl.label,
          'type', cl.type,
          'address_line1', cl.address_line1,
          'address_line2', cl.address_line2,
          'city', cl.city,
          'state', cl.state,
          'postal_code', cl.postal_code,
          'country', cl.country,
          'latitude', cl.latitude,
          'longitude', cl.longitude,
          'is_primary', cl.is_primary,
          'created_at', cl.created_at,
          'updated_at', cl.updated_at
        ) ORDER BY cl.is_primary DESC, cl.created_at)
        FROM customer_locations cl
        WHERE cl.customer_id = c.id
      ),
      '[]'::jsonb
    ),
    
    -- All contacts
    'contacts', COALESCE(
      (
        SELECT jsonb_agg(jsonb_build_object(
          'id', cc.id,
          'type', cc.type,
          'value', cc.value,
          'is_primary', cc.is_primary,
          'is_verified', cc.is_verified,
          'opt_in_marketing', cc.opt_in_marketing,
          'opt_in_transactional', cc.opt_in_transactional,
          'preferred_channel', cc.preferred_channel,
          'notes', cc.notes,
          'created_at', cc.created_at,
          'updated_at', cc.updated_at
        ) ORDER BY cc.is_primary DESC, cc.created_at)
        FROM customer_contacts cc
        WHERE cc.customer_id = c.id
      ),
      '[]'::jsonb
    ),
    
    -- Preferences
    'preferences', CASE
      WHEN cp.id IS NOT NULL THEN jsonb_build_object(
        'do_not_contact', cp.do_not_contact,
        'do_not_email', cp.do_not_email,
        'do_not_sms', cp.do_not_sms,
        'do_not_call', cp.do_not_call,
        'preferred_contact_window_start', cp.preferred_contact_window_start,
        'preferred_contact_window_end', cp.preferred_contact_window_end,
        'notes', cp.notes
      )
      ELSE NULL
    END,
    
    -- Interactions (paginated)
    'interactions', jsonb_build_object(
      'data', COALESCE(
        (
          SELECT jsonb_agg(jsonb_build_object(
            'id', ci.id,
            'channel', ci.channel,
            'direction', ci.direction,
            'subject', ci.subject,
            'summary', ci.summary,
            'sentiment', ci.sentiment,
            'occurred_at', ci.occurred_at,
            'created_by_user_id', ci.created_by_user_id
          ) ORDER BY ci.occurred_at DESC)
          FROM crm_interactions ci
          WHERE ci.customer_id = c.id
          ORDER BY ci.occurred_at DESC
          LIMIT p_interactions_limit
          OFFSET p_interactions_offset
        ),
        '[]'::jsonb
      ),
      'total', (
        SELECT COUNT(*)::INTEGER
        FROM crm_interactions ci
        WHERE ci.customer_id = c.id
      ),
      'limit', p_interactions_limit,
      'offset', p_interactions_offset
    ),
    
    -- Follow-ups (paginated)
    'followups', jsonb_build_object(
      'data', COALESCE(
        (
          SELECT jsonb_agg(jsonb_build_object(
            'id', cf.id,
            'title', cf.title,
            'description', cf.description,
            'due_at', cf.due_at,
            'status', cf.status,
            'priority', cf.priority,
            'assigned_to_user_id', cf.assigned_to_user_id
          ) ORDER BY cf.due_at ASC)
          FROM crm_followups cf
          WHERE cf.customer_id = c.id
          AND cf.status = 'pending'
          ORDER BY cf.due_at ASC
          LIMIT p_followups_limit
          OFFSET p_followups_offset
        ),
        '[]'::jsonb
      ),
      'total', (
        SELECT COUNT(*)::INTEGER
        FROM crm_followups cf
        WHERE cf.customer_id = c.id
        AND cf.status = 'pending'
      ),
      'limit', p_followups_limit,
      'offset', p_followups_offset
    ),
    
    -- Tags
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
    )
  )
  INTO v_result
  FROM customers c
  LEFT JOIN customer_locations pl ON pl.id = c.primary_location_id
  LEFT JOIN crm_preferences cp ON cp.customer_id = c.id
  WHERE c.id = p_customer_id
  AND c.org_id = v_org_id; -- RLS check
  
  IF v_result IS NULL THEN
    RAISE EXCEPTION 'Customer not found or access denied';
  END IF;
  
  RETURN v_result;
END;
$$;
```

---

## 6. Story CRM-019: Search & List Customers API

### 6.1 Endpoint Specification

**Path**: `GET /crm/customers`

**Method**: `GET`

**Authentication**: Required (Supabase JWT)

**Query Parameters**:
- `q` (optional) - Search query (searches name, email, phone, address)
- `status` (optional) - Filter by status (comma-separated for multiple)
- `lifecycle_stage` (optional) - Filter by lifecycle stage (comma-separated)
- `tag` (optional) - Filter by tag ID (UUID, can be repeated)
- `segment_id` (optional) - Filter by segment ID (UUID)
- `limit` (optional, default: 20, max: 100) - Results per page
- `offset` (optional, default: 0) - Pagination offset
- `sort_by` (optional, default: 'name') - Sort field: 'name', 'created_at', 'updated_at'
- `sort_order` (optional, default: 'asc') - Sort order: 'asc', 'desc'

### 6.2 Implementation: PostgreSQL RPC Function

```sql
CREATE OR REPLACE FUNCTION crm_search_customers(
  p_search_query TEXT DEFAULT NULL,
  p_status TEXT[] DEFAULT NULL,
  p_lifecycle_stage TEXT[] DEFAULT NULL,
  p_tag_ids UUID[] DEFAULT NULL,
  p_segment_id UUID DEFAULT NULL,
  p_limit INTEGER DEFAULT 20,
  p_offset INTEGER DEFAULT 0,
  p_sort_by TEXT DEFAULT 'name',
  p_sort_order TEXT DEFAULT 'asc'
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_org_id UUID;
  v_result JSONB;
  v_sort_expr TEXT;
BEGIN
  -- Get user's org_id
  SELECT org_id INTO v_org_id
  FROM profiles
  WHERE id = auth.uid() AND is_active = true;
  
  IF v_org_id IS NULL THEN
    RAISE EXCEPTION 'User not authenticated or inactive';
  END IF;
  
  -- Validate and set limits
  IF p_limit > 100 THEN
    p_limit := 100;
  END IF;
  IF p_limit < 1 THEN
    p_limit := 20;
  END IF;
  
  -- Validate sort_by
  IF p_sort_by NOT IN ('name', 'created_at', 'updated_at') THEN
    p_sort_by := 'name';
  END IF;
  
  -- Validate sort_order
  IF p_sort_order NOT IN ('asc', 'desc') THEN
    p_sort_order := 'asc';
  END IF;
  
  -- Build sort expression
  v_sort_expr := p_sort_by || ' ' || UPPER(p_sort_order);
  
  -- Build and execute query
  SELECT jsonb_build_object(
    'data', COALESCE(
      (
        SELECT jsonb_agg(jsonb_build_object(
          'id', c.id,
          'type', c.type,
          'name', c.name,
          'first_name', c.first_name,
          'last_name', c.last_name,
          'company_name', c.company_name,
          'email', c.email,
          'phone', c.phone,
          'status', c.status,
          'lifecycle_stage', c.lifecycle_stage,
          'created_at', c.created_at,
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
          )
        ))
        FROM customers c
        WHERE c.org_id = v_org_id
        -- Search query filter
        AND (
          p_search_query IS NULL OR
          p_search_query = '' OR
          (
            c.name ILIKE '%' || p_search_query || '%' OR
            c.email ILIKE '%' || p_search_query || '%' OR
            c.phone ILIKE '%' || p_search_query || '%' OR
            EXISTS (
              SELECT 1 FROM customer_locations cl
              WHERE cl.customer_id = c.id
              AND (
                cl.address_line1 ILIKE '%' || p_search_query || '%' OR
                cl.city ILIKE '%' || p_search_query || '%' OR
                cl.postal_code ILIKE '%' || p_search_query || '%'
              )
            )
          )
        )
        -- Status filter
        AND (p_status IS NULL OR c.status = ANY(p_status))
        -- Lifecycle stage filter
        AND (p_lifecycle_stage IS NULL OR c.lifecycle_stage = ANY(p_lifecycle_stage))
        -- Tag filter
        AND (
          p_tag_ids IS NULL OR
          EXISTS (
            SELECT 1 FROM crm_customer_tags cct
            WHERE cct.customer_id = c.id
            AND cct.tag_id = ANY(p_tag_ids)
          )
        )
        -- Segment filter
        AND (
          p_segment_id IS NULL OR
          EXISTS (
            SELECT 1 FROM crm_segment_members csm
            WHERE csm.customer_id = c.id
            AND csm.segment_id = p_segment_id
          )
        )
        ORDER BY 
          CASE WHEN v_sort_expr LIKE 'name%' THEN c.name END,
          CASE WHEN v_sort_expr LIKE 'created_at%' THEN c.created_at END,
          CASE WHEN v_sort_expr LIKE 'updated_at%' THEN c.updated_at END
        LIMIT p_limit
        OFFSET p_offset
      ),
      '[]'::jsonb
    ),
    'total', (
      SELECT COUNT(*)
      FROM customers c
      WHERE c.org_id = v_org_id
      -- Same filters as above
      AND (
        p_search_query IS NULL OR
        p_search_query = '' OR
        (
          c.name ILIKE '%' || p_search_query || '%' OR
          c.email ILIKE '%' || p_search_query || '%' OR
          c.phone ILIKE '%' || p_search_query || '%' OR
          EXISTS (
            SELECT 1 FROM customer_locations cl
            WHERE cl.customer_id = c.id
            AND (
              cl.address_line1 ILIKE '%' || p_search_query || '%' OR
              cl.city ILIKE '%' || p_search_query || '%' OR
              cl.postal_code ILIKE '%' || p_search_query || '%'
            )
          )
        )
      )
      AND (p_status IS NULL OR c.status = ANY(p_status))
      AND (p_lifecycle_stage IS NULL OR c.lifecycle_stage = ANY(p_lifecycle_stage))
      AND (
        p_tag_ids IS NULL OR
        EXISTS (
          SELECT 1 FROM crm_customer_tags cct
          WHERE cct.customer_id = c.id
          AND cct.tag_id = ANY(p_tag_ids)
        )
      )
      AND (
        p_segment_id IS NULL OR
        EXISTS (
          SELECT 1 FROM crm_segment_members csm
          WHERE csm.customer_id = c.id
          AND csm.segment_id = p_segment_id
        )
      )
    ),
    'limit', p_limit,
    'offset', p_offset
  )
  INTO v_result;
  
  RETURN v_result;
END;
$$;
```

### 6.3 Response Schema

```typescript
interface SearchCustomersResponse {
  data: Array<{
    id: string;
    type: string;
    name: string;
    first_name?: string;
    last_name?: string;
    company_name?: string;
    email?: string;
    phone?: string;
    status: string;
    lifecycle_stage: string;
    created_at: string;
    tags: Array<{
      id: string;
      name: string;
      color?: string;
    }>;
  }>;
  total: number;
  limit: number;
  offset: number;
}
```

---

## 7. Error Handling

### 7.1 Standard Error Response Format

All APIs return errors in this format:

```typescript
interface ErrorResponse {
  error: string; // Human-readable error message
  code?: string; // Optional error code (e.g., 'VALIDATION_ERROR', 'NOT_FOUND')
  details?: any; // Optional additional error details
}
```

### 7.2 HTTP Status Codes

- **200 OK**: Successful GET/PATCH request
- **201 Created**: Successful POST request
- **400 Bad Request**: Validation error or invalid input
- **401 Unauthorized**: Missing or invalid authentication token
- **403 Forbidden**: User lacks permission (RLS blocked)
- **404 Not Found**: Resource not found
- **500 Internal Server Error**: Server error

### 7.3 Error Codes

- `VALIDATION_ERROR`: Request validation failed
- `NOT_FOUND`: Resource not found
- `UNAUTHORIZED`: Authentication required
- `FORBIDDEN`: Insufficient permissions
- `DATABASE_ERROR`: Database operation failed
- `CONSTRAINT_VIOLATION`: Database constraint violation

---

## 8. Performance Considerations

### 8.1 Query Optimization

- Use indexes defined in Epic 1 for all filter and sort operations
- Limit result sets (default 20, max 100)
- Use pagination for large datasets
- Consider materialized views for complex composed queries if needed

### 8.2 Caching Strategy

- Customer details can be cached (TTL: 5 minutes)
- Search results should not be cached (real-time data)
- Use Supabase real-time subscriptions for live updates

### 8.3 Performance Targets

- Create Customer: < 500ms
- Update Customer: < 300ms
- Get Customer Details: < 400ms (with 10 interactions, 10 follow-ups)
- Search Customers: < 500ms (for up to 50k customers per org)

---

## 9. Testing Requirements

### 9.1 Unit Tests

- Request validation functions
- Error handling paths
- Edge cases (empty arrays, null values, etc.)

### 9.2 Integration Tests

- End-to-end API calls
- RLS enforcement verification
- Transaction rollback on errors
- Multi-table operations

### 9.3 Performance Tests

- Load testing with realistic data volumes
- Query performance validation
- Concurrent request handling

---

## 10. Implementation Checklist

### Story CRM-016: Create Customer
- [ ] Edge Function implemented (`POST /crm/customers`)
- [ ] RPC function implemented (`crm_create_customer`)
- [ ] Request validation implemented
- [ ] Transaction management for multi-table operations
- [ ] Error handling and rollback
- [ ] Response schema matches specification
- [ ] Tests written (happy path and error cases)
- [ ] API documentation with examples

### Story CRM-017: Update Customer
- [ ] Edge Function implemented (`PATCH /crm/customers/:id`)
- [ ] RPC function implemented (`crm_update_customer`)
- [ ] Partial update support
- [ ] Nested updates (locations, contacts)
- [ ] Optional interaction logging
- [ ] RLS enforcement verified
- [ ] Tests written
- [ ] API documentation with examples

### Story CRM-018: Get Customer Details
- [ ] Edge Function or RPC function implemented (`GET /crm/customers/:id`)
- [ ] Composed view with all nested data
- [ ] Pagination for interactions and follow-ups
- [ ] Performance validated
- [ ] Response schema matches specification
- [ ] Tests written
- [ ] API documentation with examples

### Story CRM-019: Search & List Customers
- [ ] Edge Function or RPC function implemented (`GET /crm/customers`)
- [ ] Search query implementation (name, email, phone, address)
- [ ] Filter support (status, lifecycle_stage, tag, segment)
- [ ] Pagination support
- [ ] Sorting support
- [ ] Performance validated
- [ ] Tests written
- [ ] API documentation with examples

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 3 – Customer Management APIs & Workflows. All specifications are designed to be directly consumable by LLM-based code generators, with exact request/response schemas, validation rules, SQL functions, and Edge Function implementations defined.

