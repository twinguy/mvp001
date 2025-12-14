# Technical Design Document – Epic 9: Calendar Integration (OAuth, Sync, Webhooks)

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 9 – Calendar Integration (OAuth, Sync, Webhooks)
- **Source**: Derived from `fdd_2_agile.md` Epic 9 (Stories DISP-044 through DISP-047)
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
  - `tdd_2_epic_8.md` (Dispatch Epic 8 for emergency job handling)
- **Target Platform**: Supabase (PostgreSQL 15+, Edge Functions)
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing Calendar Integration with OAuth, sync, and webhooks. It covers:

- OAuth flow initiation and callback handling for Google and Microsoft calendars
- Secure token storage and encryption
- Synchronization of internal appointments to external calendars
- Webhook ingestion for external calendar changes
- Reconciliation modes (apply vs flag) for external edits

All APIs are implemented as Supabase Edge Functions (Deno/TypeScript) that integrate with Google Calendar API and Microsoft Graph API, handle OAuth flows securely, and provide bidirectional calendar synchronization.

This epic assumes Epic 1 (tenancy/roles), Epic 2 (tables), Epic 3 (RLS policies), Epic 4 (technician APIs), Epic 5 (job lifecycle APIs), Epic 6 (technician mobile hooks), Epic 7 (auto-scheduling and route optimization), and Epic 8 (emergency job handling) are complete.

---

## 2. Prerequisites

### 2.1 Required Components

Before implementing Epic 9, ensure:

1. **Epic 1-8 Complete**: All previous epics are implemented
2. **Required Tables**: All dispatch tables exist:
   - `calendar_integrations`
   - `calendar_events`
   - `job_assignments`
   - `dispatch_jobs`
   - `technician_profiles`

3. **OAuth Provider Setup**:
   - Google Cloud Project with OAuth 2.0 credentials
   - Microsoft Azure App Registration with OAuth 2.0 credentials
   - Redirect URIs configured in provider dashboards

4. **Supabase Vault**: Configured for token encryption (or application-level encryption)

### 2.2 Helper Functions

From Epic 1:
- `get_user_org_id()` - Returns authenticated user's org_id
- `get_user_role()` - Returns authenticated user's role

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

### 3.3 Token Encryption Helper

**Critical**: Tokens must be encrypted at rest.

**Option 1: Supabase Vault** (Recommended):

```typescript
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

async function encryptToken(
  supabase: SupabaseClient,
  plaintext: string
): Promise<string> {
  // Use Supabase Vault for encryption
  // Note: Requires service_role key and Vault extension enabled
  const { data, error } = await supabase.rpc('encrypt', {
    data: plaintext,
    key_id: 'calendar_tokens_key' // Configured in Vault
  });

  if (error) {
    throw new Error(`Encryption failed: ${error.message}`);
  }

  return data;
}

async function decryptToken(
  supabase: SupabaseClient,
  ciphertext: string
): Promise<string> {
  const { data, error } = await supabase.rpc('decrypt', {
    data: ciphertext,
    key_id: 'calendar_tokens_key'
  });

  if (error) {
    throw new Error(`Decryption failed: ${error.message}`);
  }

  return data;
}
```

**Option 2: Application-Level Encryption** (Alternative):

```typescript
import { crypto } from 'https://deno.land/std@0.208.0/crypto/mod.ts';

const ENCRYPTION_KEY = Deno.env.get('CALENDAR_ENCRYPTION_KEY'); // 32-byte key

async function encryptToken(plaintext: string): Promise<string> {
  const key = await crypto.subtle.importKey(
    'raw',
    new TextEncoder().encode(ENCRYPTION_KEY!),
    { name: 'AES-GCM', length: 256 },
    false,
    ['encrypt']
  );

  const iv = crypto.getRandomValues(new Uint8Array(12));
  const encoded = new TextEncoder().encode(plaintext);

  const encrypted = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv },
    key,
    encoded
  );

  // Combine IV and encrypted data
  const combined = new Uint8Array(iv.length + encrypted.byteLength);
  combined.set(iv);
  combined.set(new Uint8Array(encrypted), iv.length);

  return btoa(String.fromCharCode(...combined));
}

async function decryptToken(ciphertext: string): Promise<string> {
  const combined = Uint8Array.from(atob(ciphertext), c => c.charCodeAt(0));
  const iv = combined.slice(0, 12);
  const encrypted = combined.slice(12);

  const key = await crypto.subtle.importKey(
    'raw',
    new TextEncoder().encode(ENCRYPTION_KEY!),
    { name: 'AES-GCM', length: 256 },
    false,
    ['decrypt']
  );

  const decrypted = await crypto.subtle.decrypt(
    { name: 'AES-GCM', iv },
    key,
    encrypted
  );

  return new TextDecoder().decode(decrypted);
}
```

---

## 4. Story DISP-044: Calendar Connect Endpoint (OAuth Initiation)

### 4.1 POST /dispatch/calendar/connect

**Purpose**: Initiate OAuth flow for connecting Google or Microsoft calendar.

**Authorization**: `admin`, `dispatcher`, `technician`

**Request Schema**:

```typescript
interface ConnectCalendarRequest {
  provider: 'google' | 'microsoft';
  redirect_uri?: string; // Optional, defaults to configured redirect URI
}
```

**Request Example**:

```json
{
  "provider": "google"
}
```

**Response Schema**:

```typescript
interface ConnectCalendarResponse {
  authorization_url: string;
  state: string; // CSRF token for validation
  expires_at: string; // ISO 8601 timestamp
}
```

**Response Example**:

```json
{
  "data": {
    "authorization_url": "https://accounts.google.com/o/oauth2/v2/auth?client_id=...&redirect_uri=...&response_type=code&scope=...&state=...&access_type=offline&prompt=consent",
    "state": "abc123xyz789",
    "expires_at": "2024-01-20T10:05:00Z"
  }
}
```

### 4.2 OAuth State Management

**State Storage Table**:

```sql
CREATE TABLE IF NOT EXISTS oauth_states (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  state TEXT NOT NULL UNIQUE,
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  provider TEXT NOT NULL,
  code_verifier TEXT, -- For PKCE
  redirect_uri TEXT,
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_oauth_states_state ON oauth_states(state);
CREATE INDEX idx_oauth_states_expires_at ON oauth_states(expires_at);

-- Cleanup expired states (run periodically)
CREATE OR REPLACE FUNCTION cleanup_expired_oauth_states()
RETURNS void AS $$
BEGIN
  DELETE FROM oauth_states WHERE expires_at < now();
END;
$$ LANGUAGE plpgsql;
```

**State Generation**:

```typescript
function generateState(): string {
  const randomBytes = crypto.getRandomValues(new Uint8Array(32));
  return Array.from(randomBytes, byte => byte.toString(16).padStart(2, '0')).join('');
}

function generateCodeVerifier(): string {
  const randomBytes = crypto.getRandomValues(new Uint8Array(32));
  return btoa(String.fromCharCode(...randomBytes))
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=/g, '');
}

async function sha256(plaintext: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(plaintext);
  const hashBuffer = await crypto.subtle.digest('SHA-256', data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
}

async function generateCodeChallenge(verifier: string): Promise<string> {
  const hash = await sha256(verifier);
  return btoa(String.fromCharCode(...Uint8Array.from(hash, c => parseInt(c, 16))))
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=/g, '');
}
```

### 4.3 Google OAuth Flow

**OAuth Configuration**:

```typescript
interface GoogleOAuthConfig {
  clientId: string;
  clientSecret: string;
  redirectUri: string;
  scopes: string[];
}

const GOOGLE_OAUTH_CONFIG: GoogleOAuthConfig = {
  clientId: Deno.env.get('GOOGLE_CLIENT_ID')!,
  clientSecret: Deno.env.get('GOOGLE_CLIENT_SECRET')!,
  redirectUri: Deno.env.get('GOOGLE_REDIRECT_URI') || 'https://your-domain.com/api/calendar/oauth/callback/google',
  scopes: [
    'https://www.googleapis.com/auth/calendar',
    'https://www.googleapis.com/auth/calendar.events'
  ]
};
```

**Authorization URL Generation**:

```typescript
async function generateGoogleAuthUrl(
  state: string,
  codeVerifier: string,
  redirectUri: string
): Promise<string> {
  const codeChallenge = await generateCodeChallenge(codeVerifier);
  
  const params = new URLSearchParams({
    client_id: GOOGLE_OAUTH_CONFIG.clientId,
    redirect_uri: redirectUri,
    response_type: 'code',
    scope: GOOGLE_OAUTH_CONFIG.scopes.join(' '),
    state: state,
    access_type: 'offline', // Required for refresh token
    prompt: 'consent', // Force consent screen to get refresh token
    code_challenge: codeChallenge,
    code_challenge_method: 'S256'
  });

  return `https://accounts.google.com/o/oauth2/v2/auth?${params.toString()}`;
}
```

### 4.4 Microsoft OAuth Flow

**OAuth Configuration**:

```typescript
interface MicrosoftOAuthConfig {
  clientId: string;
  clientSecret: string;
  redirectUri: string;
  tenantId: string; // Optional, 'common' for multi-tenant
  scopes: string[];
}

const MICROSOFT_OAUTH_CONFIG: MicrosoftOAuthConfig = {
  clientId: Deno.env.get('MICROSOFT_CLIENT_ID')!,
  clientSecret: Deno.env.get('MICROSOFT_CLIENT_SECRET')!,
  redirectUri: Deno.env.get('MICROSOFT_REDIRECT_URI') || 'https://your-domain.com/api/calendar/oauth/callback/microsoft',
  tenantId: Deno.env.get('MICROSOFT_TENANT_ID') || 'common',
  scopes: [
    'Calendars.ReadWrite',
    'offline_access' // Required for refresh token
  ]
};
```

**Authorization URL Generation**:

```typescript
async function generateMicrosoftAuthUrl(
  state: string,
  codeVerifier: string,
  redirectUri: string
): Promise<string> {
  const codeChallenge = await generateCodeChallenge(codeVerifier);
  const tenant = MICROSOFT_OAUTH_CONFIG.tenantId;

  const params = new URLSearchParams({
    client_id: MICROSOFT_OAUTH_CONFIG.clientId,
    response_type: 'code',
    redirect_uri: redirectUri,
    response_mode: 'query',
    scope: MICROSOFT_OAUTH_CONFIG.scopes.join(' '),
    state: state,
    code_challenge: codeChallenge,
    code_challenge_method: 'S256'
  });

  return `https://login.microsoftonline.com/${tenant}/oauth2/v2.0/authorize?${params.toString()}`;
}
```

### 4.5 Implementation (Edge Function)

**POST /dispatch/calendar/connect**:

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

  const auth = await authorizeUser(supabase, user.id, ['admin', 'dispatcher', 'technician']);
  if (!auth) {
    return errorResponse('Forbidden', 403, 'INSUFFICIENT_PERMISSIONS');
  }

  const body = await req.json();
  const provider = body.provider;

  if (provider !== 'google' && provider !== 'microsoft') {
    return errorResponse('Invalid provider', 400, 'INVALID_PROVIDER');
  }

  // Generate state and code verifier (PKCE)
  const state = generateState();
  const codeVerifier = generateCodeVerifier();
  const redirectUri = body.redirect_uri || 
    (provider === 'google' ? GOOGLE_OAUTH_CONFIG.redirectUri : MICROSOFT_OAUTH_CONFIG.redirectUri);

  // Store state (expires in 10 minutes)
  const expiresAt = new Date();
  expiresAt.setMinutes(expiresAt.getMinutes() + 10);

  const { error: stateError } = await supabase
    .from('oauth_states')
    .insert({
      state: state,
      user_id: user.id,
      org_id: auth.orgId,
      provider: provider,
      code_verifier: codeVerifier,
      redirect_uri: redirectUri,
      expires_at: expiresAt.toISOString()
    });

  if (stateError) {
    return errorResponse('Failed to store OAuth state', 500, 'STATE_STORAGE_ERROR');
  }

  // Generate authorization URL
  let authorizationUrl: string;
  if (provider === 'google') {
    authorizationUrl = await generateGoogleAuthUrl(state, codeVerifier, redirectUri);
  } else {
    authorizationUrl = await generateMicrosoftAuthUrl(state, codeVerifier, redirectUri);
  }

  return successResponse({
    authorization_url: authorizationUrl,
    state: state,
    expires_at: expiresAt.toISOString()
  });
});
```

### 4.6 OAuth Callback Handler

**GET /dispatch/calendar/oauth/callback/:provider**

**Purpose**: Handle OAuth callback and exchange authorization code for tokens.

**Implementation**:

```typescript
Deno.serve(async (req) => {
  const url = new URL(req.url);
  const provider = url.pathname.split('/').pop();

  if (provider !== 'google' && provider !== 'microsoft') {
    return errorResponse('Invalid provider', 400, 'INVALID_PROVIDER');
  }

  const code = url.searchParams.get('code');
  const state = url.searchParams.get('state');
  const error = url.searchParams.get('error');

  if (error) {
    return errorResponse(`OAuth error: ${error}`, 400, 'OAUTH_ERROR');
  }

  if (!code || !state) {
    return errorResponse('Missing code or state', 400, 'MISSING_PARAMS');
  }

  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')! // Need service role to access oauth_states
  );

  // Verify state
  const { data: stateRecord, error: stateError } = await supabase
    .from('oauth_states')
    .select('*')
    .eq('state', state)
    .single();

  if (stateError || !stateRecord) {
    return errorResponse('Invalid or expired state', 400, 'INVALID_STATE');
  }

  if (new Date(stateRecord.expires_at) < new Date()) {
    return errorResponse('State expired', 400, 'STATE_EXPIRED');
  }

  if (stateRecord.provider !== provider) {
    return errorResponse('Provider mismatch', 400, 'PROVIDER_MISMATCH');
  }

  // Exchange code for tokens
  let tokens: { access_token: string; refresh_token?: string; expires_in?: number };
  
  if (provider === 'google') {
    tokens = await exchangeGoogleCodeForTokens(
      code,
      stateRecord.code_verifier!,
      stateRecord.redirect_uri
    );
  } else {
    tokens = await exchangeMicrosoftCodeForTokens(
      code,
      stateRecord.code_verifier!,
      stateRecord.redirect_uri
    );
  }

  // Get user info from provider
  const userInfo = await getUserInfoFromProvider(provider, tokens.access_token);

  // Encrypt tokens
  const encryptedAccessToken = await encryptToken(supabase, tokens.access_token);
  const encryptedRefreshToken = tokens.refresh_token 
    ? await encryptToken(supabase, tokens.refresh_token)
    : null;

  // Calculate token expiration
  const expiresAt = tokens.expires_in
    ? new Date(Date.now() + tokens.expires_in * 1000)
    : null;

  // Get calendar ID (primary calendar)
  const calendarId = await getPrimaryCalendarId(provider, tokens.access_token);

  // Store integration
  const { data: integration, error: integrationError } = await supabase
    .from('calendar_integrations')
    .upsert({
      org_id: stateRecord.org_id,
      user_id: stateRecord.user_id,
      provider: provider,
      provider_account_id: userInfo.id,
      access_token: encryptedAccessToken,
      refresh_token: encryptedRefreshToken,
      expires_at: expiresAt?.toISOString() || null,
      calendar_id: calendarId,
      scope: provider === 'google' 
        ? GOOGLE_OAUTH_CONFIG.scopes.join(' ')
        : MICROSOFT_OAUTH_CONFIG.scopes.join(' '),
      is_active: true,
      metadata: {
        email: userInfo.email,
        name: userInfo.name
      }
    }, {
      onConflict: 'org_id,user_id,provider'
    })
    .select()
    .single();

  if (integrationError) {
    return errorResponse('Failed to save integration', 500, 'SAVE_ERROR');
  }

  // Delete used state
  await supabase
    .from('oauth_states')
    .delete()
    .eq('id', stateRecord.id);

  // Redirect to success page
  return new Response(null, {
    status: 302,
    headers: {
      'Location': `${stateRecord.redirect_uri}?success=true&integration_id=${integration.id}`
    }
  });
});

async function exchangeGoogleCodeForTokens(
  code: string,
  codeVerifier: string,
  redirectUri: string
): Promise<{ access_token: string; refresh_token?: string; expires_in?: number }> {
  const response = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded'
    },
    body: new URLSearchParams({
      client_id: GOOGLE_OAUTH_CONFIG.clientId,
      client_secret: GOOGLE_OAUTH_CONFIG.clientSecret,
      code: code,
      grant_type: 'authorization_code',
      redirect_uri: redirectUri,
      code_verifier: codeVerifier
    })
  });

  if (!response.ok) {
    const error = await response.text();
    throw new Error(`Token exchange failed: ${error}`);
  }

  return await response.json();
}

async function exchangeMicrosoftCodeForTokens(
  code: string,
  codeVerifier: string,
  redirectUri: string
): Promise<{ access_token: string; refresh_token?: string; expires_in?: number }> {
  const tenant = MICROSOFT_OAUTH_CONFIG.tenantId;
  
  const response = await fetch(`https://login.microsoftonline.com/${tenant}/oauth2/v2.0/token`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded'
    },
    body: new URLSearchParams({
      client_id: MICROSOFT_OAUTH_CONFIG.clientId,
      client_secret: MICROSOFT_OAUTH_CONFIG.clientSecret,
      code: code,
      grant_type: 'authorization_code',
      redirect_uri: redirectUri,
      code_verifier: codeVerifier
    })
  });

  if (!response.ok) {
    const error = await response.text();
    throw new Error(`Token exchange failed: ${error}`);
  }

  return await response.json();
}

async function getUserInfoFromProvider(
  provider: 'google' | 'microsoft',
  accessToken: string
): Promise<{ id: string; email: string; name: string }> {
  if (provider === 'google') {
    const response = await fetch('https://www.googleapis.com/oauth2/v2/userinfo', {
      headers: {
        'Authorization': `Bearer ${accessToken}`
      }
    });

    if (!response.ok) {
      throw new Error('Failed to get user info');
    }

    const data = await response.json();
    return {
      id: data.id,
      email: data.email,
      name: data.name
    };
  } else {
    const response = await fetch('https://graph.microsoft.com/v1.0/me', {
      headers: {
        'Authorization': `Bearer ${accessToken}`
      }
    });

    if (!response.ok) {
      throw new Error('Failed to get user info');
    }

    const data = await response.json();
    return {
      id: data.id,
      email: data.mail || data.userPrincipalName,
      name: data.displayName
    };
  }
}

async function getPrimaryCalendarId(
  provider: 'google' | 'microsoft',
  accessToken: string
): Promise<string> {
  if (provider === 'google') {
    const response = await fetch('https://www.googleapis.com/calendar/v3/users/me/calendarList/primary', {
      headers: {
        'Authorization': `Bearer ${accessToken}`
      }
    });

    if (!response.ok) {
      throw new Error('Failed to get primary calendar');
    }

    const data = await response.json();
    return data.id;
  } else {
    // Microsoft uses 'calendar' as the default calendar ID
    return 'calendar';
  }
}
```

### 4.7 Security Considerations

**CSRF Protection**:
- State parameter validates callback authenticity
- State expires after 10 minutes
- State is single-use (deleted after use)

**PKCE (Proof Key for Code Exchange)**:
- Code verifier generated client-side
- Code challenge sent in authorization request
- Code verifier required for token exchange
- Prevents authorization code interception attacks

**Token Security**:
- Tokens encrypted at rest using Supabase Vault or application-level encryption
- Tokens never exposed in API responses
- Refresh tokens stored securely
- Token expiration tracked and refreshed automatically

**Least Scope**:
- Request only necessary scopes (`calendar`, `calendar.events` for Google)
- Document required scopes in configuration

---

## 5. Story DISP-045: Sync Internal Appointments to Calendar

### 5.1 POST /dispatch/calendar/sync

**Purpose**: Sync internal job assignments to external calendars.

**Authorization**: `admin`, `dispatcher` (or system/cron)

**Request Schema**:

```typescript
interface SyncCalendarRequest {
  integration_id?: string; // Optional: sync specific integration
  assignment_id?: string; // Optional: sync specific assignment
  force_full_sync?: boolean; // default: false, incremental sync
}
```

**Response Schema**:

```typescript
interface SyncCalendarResponse {
  synced_count: number;
  created_count: number;
  updated_count: number;
  canceled_count: number;
  errors: Array<{
    assignment_id: string;
    error: string;
  }>;
}
```

### 5.2 Sync Logic

**Incremental Sync Strategy**:

1. **Find Pending Assignments**:
   - Assignments without `calendar_events` records
   - Assignments with `calendar_events.status = 'updated'`
   - Assignments modified since last sync

2. **For Each Integration**:
   - Get active integrations for technicians with assignments
   - Refresh access token if expired
   - Sync assignments to provider calendar

3. **Create/Update/Cancel Events**:
   - Create events for new assignments
   - Update events for modified assignments
   - Cancel events for canceled assignments

**Implementation**:

```typescript
Deno.serve(async (req) => {
  // ... authorization ...

  const body = await req.json();
  const integrationId = body.integration_id;
  const assignmentId = body.assignment_id;
  const forceFullSync = body.force_full_sync || false;

  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')! // Need service role for sync
  );

  // Get integrations to sync
  let integrationsQuery = supabase
    .from('calendar_integrations')
    .select('*')
    .eq('org_id', auth.orgId)
    .eq('is_active', true);

  if (integrationId) {
    integrationsQuery = integrationsQuery.eq('id', integrationId);
  }

  const { data: integrations } = await integrationsQuery;

  if (!integrations || integrations.length === 0) {
    return successResponse({
      synced_count: 0,
      created_count: 0,
      updated_count: 0,
      canceled_count: 0,
      errors: []
    });
  }

  let syncedCount = 0;
  let createdCount = 0;
  let updatedCount = 0;
  let canceledCount = 0;
  const errors: Array<{ assignment_id: string; error: string }> = [];

  for (const integration of integrations) {
    try {
      // Refresh token if expired
      const decryptedAccessToken = await decryptToken(supabase, integration.access_token);
      const decryptedRefreshToken = integration.refresh_token 
        ? await decryptToken(supabase, integration.refresh_token)
        : null;

      let accessToken = decryptedAccessToken;
      
      if (integration.expires_at && new Date(integration.expires_at) < new Date()) {
        if (!decryptedRefreshToken) {
          console.error(`Integration ${integration.id} token expired and no refresh token`);
          continue;
        }

        // Refresh token
        const refreshed = await refreshProviderToken(integration.provider, decryptedRefreshToken);
        accessToken = refreshed.access_token;

        // Update stored tokens
        const encryptedAccessToken = await encryptToken(supabase, refreshed.access_token);
        const encryptedRefreshToken = refreshed.refresh_token
          ? await encryptToken(supabase, refreshed.refresh_token)
          : integration.refresh_token;

        await supabase
          .from('calendar_integrations')
          .update({
            access_token: encryptedAccessToken,
            refresh_token: encryptedRefreshToken,
            expires_at: refreshed.expires_in
              ? new Date(Date.now() + refreshed.expires_in * 1000).toISOString()
              : null
          })
          .eq('id', integration.id);
      }

      // Get assignments to sync
      let assignmentsQuery = supabase
        .from('job_assignments')
        .select(`
          *,
          dispatch_jobs!inner(
            id,
            title,
            description,
            customer_locations!inner(
              address_line1,
              city,
              state,
              postal_code
            )
          ),
          technician_profiles!inner(
            user_id
          )
        `)
        .eq('org_id', auth.orgId)
        .eq('technician_profiles.user_id', integration.user_id)
        .in('status', ['assigned', 'accepted', 'en_route', 'on_site']);

      if (assignmentId) {
        assignmentsQuery = assignmentsQuery.eq('id', assignmentId);
      }

      const { data: assignments } = await assignmentsQuery;

      if (!assignments || assignments.length === 0) {
        continue;
      }

      // Sync each assignment
      for (const assignment of assignments) {
        try {
          const result = await syncAssignmentToCalendar(
            supabase,
            integration,
            assignment,
            accessToken,
            forceFullSync
          );

          syncedCount++;
          if (result.action === 'created') createdCount++;
          else if (result.action === 'updated') updatedCount++;
          else if (result.action === 'canceled') canceledCount++;
        } catch (error) {
          errors.push({
            assignment_id: assignment.id,
            error: error.message
          });
        }
      }
    } catch (error) {
      console.error(`Failed to sync integration ${integration.id}:`, error);
    }
  }

  return successResponse({
    synced_count: syncedCount,
    created_count: createdCount,
    updated_count: updatedCount,
    canceled_count: canceledCount,
    errors: errors
  });
});

async function syncAssignmentToCalendar(
  supabase: SupabaseClient,
  integration: any,
  assignment: any,
  accessToken: string,
  forceFullSync: boolean
): Promise<{ action: 'created' | 'updated' | 'canceled' | 'skipped' }> {
  // Check if calendar event exists
  const { data: existingEvent } = await supabase
    .from('calendar_events')
    .select('*')
    .eq('calendar_integration_id', integration.id)
    .eq('job_assignment_id', assignment.id)
    .single();

  // Check if assignment was canceled
  if (assignment.status === 'canceled' || assignment.status === 'no_show') {
    if (existingEvent) {
      // Cancel external event
      await cancelProviderEvent(
        integration.provider,
        accessToken,
        integration.calendar_id,
        existingEvent.provider_event_id
      );

      // Update calendar_events
      await supabase
        .from('calendar_events')
        .update({
          status: 'canceled',
          last_synced_at: new Date().toISOString()
        })
        .eq('id', existingEvent.id);

      return { action: 'canceled' };
    }
    return { action: 'skipped' };
  }

  // Check if update needed
  if (existingEvent && !forceFullSync) {
    const needsUpdate = 
      new Date(assignment.updated_at) > new Date(existingEvent.last_synced_at || '1970-01-01');

    if (!needsUpdate) {
      return { action: 'skipped' };
    }
  }

  // Build event data
  const eventData = buildCalendarEventData(assignment);

  let providerEventId: string;
  let action: 'created' | 'updated';

  if (existingEvent) {
    // Update existing event
    providerEventId = await updateProviderEvent(
      integration.provider,
      accessToken,
      integration.calendar_id,
      existingEvent.provider_event_id,
      eventData
    );
    action = 'updated';
  } else {
    // Create new event
    providerEventId = await createProviderEvent(
      integration.provider,
      accessToken,
      integration.calendar_id,
      eventData
    );
    action = 'created';
  }

  // Update calendar_events table
  if (existingEvent) {
    await supabase
      .from('calendar_events')
      .update({
        provider_event_id: providerEventId,
        status: 'scheduled',
        last_synced_at: new Date().toISOString(),
        sync_direction: 'internal_to_external',
        metadata: {
          last_sync: {
            assignment_id: assignment.id,
            scheduled_start_at: assignment.scheduled_start_at,
            scheduled_end_at: assignment.scheduled_end_at,
            status: assignment.status
          }
        }
      })
      .eq('id', existingEvent.id);
  } else {
    await supabase
      .from('calendar_events')
      .insert({
        org_id: integration.org_id,
        calendar_integration_id: integration.id,
        job_assignment_id: assignment.id,
        dispatch_job_id: assignment.dispatch_job_id,
        provider_event_id: providerEventId,
        status: 'scheduled',
        last_synced_at: new Date().toISOString(),
        sync_direction: 'internal_to_external',
        metadata: {
          last_sync: {
            assignment_id: assignment.id,
            scheduled_start_at: assignment.scheduled_start_at,
            scheduled_end_at: assignment.scheduled_end_at,
            status: assignment.status
          }
        }
      });
  }

  return { action };
}

function buildCalendarEventData(assignment: any): any {
  const job = assignment.dispatch_jobs;
  const location = job.customer_locations;

  const address = [
    location.address_line1,
    location.city,
    location.state,
    location.postal_code
  ].filter(Boolean).join(', ');

  return {
    summary: job.title || 'Service Appointment',
    description: job.description || '',
    start: {
      dateTime: assignment.scheduled_start_at,
      timeZone: 'UTC'
    },
    end: {
      dateTime: assignment.scheduled_end_at,
      timeZone: 'UTC'
    },
    location: address,
    reminders: {
      useDefault: false,
      overrides: [
        { method: 'popup', minutes: 60 },
        { method: 'popup', minutes: 15 }
      ]
    }
  };
}

async function createProviderEvent(
  provider: 'google' | 'microsoft',
  accessToken: string,
  calendarId: string,
  eventData: any
): Promise<string> {
  if (provider === 'google') {
    const response = await fetch(
      `https://www.googleapis.com/calendar/v3/calendars/${calendarId}/events`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${accessToken}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(eventData)
      }
    );

    if (!response.ok) {
      const error = await response.text();
      throw new Error(`Failed to create Google event: ${error}`);
    }

    const event = await response.json();
    return event.id;
  } else {
    const response = await fetch(
      `https://graph.microsoft.com/v1.0/me/calendars/${calendarId}/events`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${accessToken}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(eventData)
      }
    );

    if (!response.ok) {
      const error = await response.text();
      throw new Error(`Failed to create Microsoft event: ${error}`);
    }

    const event = await response.json();
    return event.id;
  }
}

async function updateProviderEvent(
  provider: 'google' | 'microsoft',
  accessToken: string,
  calendarId: string,
  eventId: string,
  eventData: any
): Promise<string> {
  if (provider === 'google') {
    const response = await fetch(
      `https://www.googleapis.com/calendar/v3/calendars/${calendarId}/events/${eventId}`,
      {
        method: 'PUT',
        headers: {
          'Authorization': `Bearer ${accessToken}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(eventData)
      }
    );

    if (!response.ok) {
      const error = await response.text();
      throw new Error(`Failed to update Google event: ${error}`);
    }

    const event = await response.json();
    return event.id;
  } else {
    const response = await fetch(
      `https://graph.microsoft.com/v1.0/me/calendars/${calendarId}/events/${eventId}`,
      {
        method: 'PATCH',
        headers: {
          'Authorization': `Bearer ${accessToken}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(eventData)
      }
    );

    if (!response.ok) {
      const error = await response.text();
      throw new Error(`Failed to update Microsoft event: ${error}`);
    }

    const event = await response.json();
    return event.id;
  }
}

async function cancelProviderEvent(
  provider: 'google' | 'microsoft',
  accessToken: string,
  calendarId: string,
  eventId: string
): Promise<void> {
  if (provider === 'google') {
    const response = await fetch(
      `https://www.googleapis.com/calendar/v3/calendars/${calendarId}/events/${eventId}`,
      {
        method: 'DELETE',
        headers: {
          'Authorization': `Bearer ${accessToken}`
        }
      }
    );

    if (!response.ok && response.status !== 404) {
      const error = await response.text();
      throw new Error(`Failed to cancel Google event: ${error}`);
    }
  } else {
    const response = await fetch(
      `https://graph.microsoft.com/v1.0/me/calendars/${calendarId}/events/${eventId}`,
      {
        method: 'DELETE',
        headers: {
          'Authorization': `Bearer ${accessToken}`
        }
      }
    );

    if (!response.ok && response.status !== 404) {
      const error = await response.text();
      throw new Error(`Failed to cancel Microsoft event: ${error}`);
    }
  }
}

async function refreshProviderToken(
  provider: 'google' | 'microsoft',
  refreshToken: string
): Promise<{ access_token: string; refresh_token?: string; expires_in?: number }> {
  if (provider === 'google') {
    const response = await fetch('https://oauth2.googleapis.com/token', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded'
      },
      body: new URLSearchParams({
        client_id: GOOGLE_OAUTH_CONFIG.clientId,
        client_secret: GOOGLE_OAUTH_CONFIG.clientSecret,
        refresh_token: refreshToken,
        grant_type: 'refresh_token'
      })
    });

    if (!response.ok) {
      const error = await response.text();
      throw new Error(`Token refresh failed: ${error}`);
    }

    return await response.json();
  } else {
    const tenant = MICROSOFT_OAUTH_CONFIG.tenantId;
    const response = await fetch(`https://login.microsoftonline.com/${tenant}/oauth2/v2.0/token`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded'
      },
      body: new URLSearchParams({
        client_id: MICROSOFT_OAUTH_CONFIG.clientId,
        client_secret: MICROSOFT_OAUTH_CONFIG.clientSecret,
        refresh_token: refreshToken,
        grant_type: 'refresh_token',
        scope: MICROSOFT_OAUTH_CONFIG.scopes.join(' ')
      })
    });

    if (!response.ok) {
      const error = await response.text();
      throw new Error(`Token refresh failed: ${error}`);
    }

    return await response.json();
  }
}
```

### 5.3 Idempotency

**Idempotency Strategy**:
- Use `calendar_events.provider_event_id` as idempotency key
- Check for existing event before creating
- Update existing event if assignment changed
- Cancel event if assignment canceled

---

## 6. Story DISP-046: Calendar Provider Webhooks Ingestion

### 6.1 POST /webhooks/calendar/google

**Purpose**: Receive webhook notifications from Google Calendar.

**Webhook Verification**:

Google Calendar webhooks use push notifications. Verification requires:

1. **Verification Token**: Google sends `X-Goog-Channel-Token` header (must match configured token)
2. **Channel ID**: Google sends `X-Goog-Channel-Id` header
3. **Resource State**: Google sends `X-Goog-Resource-State` header (`sync`, `exists`, `not_exists`)

**Implementation**:

```typescript
Deno.serve(async (req) => {
  if (req.method === 'GET') {
    // Google webhook verification (initial subscription)
    const token = req.headers.get('X-Goog-Channel-Token');
    const expectedToken = Deno.env.get('GOOGLE_WEBHOOK_TOKEN');

    if (token !== expectedToken) {
      return errorResponse('Invalid token', 401);
    }

    // Return 200 to confirm subscription
    return new Response('OK', { status: 200 });
  }

  if (req.method !== 'POST') {
    return errorResponse('Method not allowed', 405);
  }

  // Verify webhook authenticity
  const token = req.headers.get('X-Goog-Channel-Token');
  const expectedToken = Deno.env.get('GOOGLE_WEBHOOK_TOKEN');

  if (token !== expectedToken) {
    return errorResponse('Invalid token', 401);
  }

  const resourceState = req.headers.get('X-Goog-Resource-State');
  const channelId = req.headers.get('X-Goog-Channel-Id');
  const resourceId = req.headers.get('X-Goog-Resource-Id');

  // Handle sync notification (initial or periodic)
  if (resourceState === 'sync') {
    return new Response('OK', { status: 200 });
  }

  // Handle event change
  if (resourceState === 'exists' || resourceState === 'not_exists') {
    // Get calendar event from database
    const supabase = createClient(
      Deno.env.get('SUPABASE_URL')!,
      Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
    );

    // Find calendar integration by channel ID (stored in metadata)
    const { data: integration } = await supabase
      .from('calendar_integrations')
      .select('*')
      .contains('metadata', { webhook_channel_id: channelId })
      .single();

    if (!integration) {
      console.error(`Integration not found for channel ${channelId}`);
      return new Response('OK', { status: 200 }); // Acknowledge to prevent retries
    }

    // Fetch event details from Google
    const decryptedAccessToken = await decryptToken(supabase, integration.access_token);
    
    if (resourceState === 'exists') {
      // Event was created or updated
      await handleGoogleEventChange(
        supabase,
        integration,
        resourceId!,
        decryptedAccessToken
      );
    } else {
      // Event was deleted
      await handleGoogleEventDeletion(
        supabase,
        integration,
        resourceId!
      );
    }
  }

  return new Response('OK', { status: 200 });
});
```

### 6.2 POST /webhooks/calendar/microsoft

**Purpose**: Receive webhook notifications from Microsoft Graph.

**Webhook Verification**:

Microsoft Graph uses subscription validation:

1. **Validation Token**: Sent in `validationToken` query parameter on initial subscription
2. **Signature Verification**: Uses HMAC-SHA256 signature in `Authorization` header

**Implementation**:

```typescript
Deno.serve(async (req) => {
  const url = new URL(req.url);
  const validationToken = url.searchParams.get('validationToken');

  if (validationToken) {
    // Initial subscription validation
    return new Response(validationToken, {
      status: 200,
      headers: { 'Content-Type': 'text/plain' }
    });
  }

  if (req.method !== 'POST') {
    return errorResponse('Method not allowed', 405);
  }

  // Verify webhook signature (simplified, would implement full verification)
  const signature = req.headers.get('Authorization');
  // ... signature verification logic ...

  const body = await req.json();
  const notifications = body.value || [];

  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
  );

  for (const notification of notifications) {
    const subscriptionId = notification.subscriptionId;
    const resource = notification.resource;
    const changeType = notification.changeType;

    // Find integration by subscription ID
    const { data: integration } = await supabase
      .from('calendar_integrations')
      .select('*')
      .contains('metadata', { webhook_subscription_id: subscriptionId })
      .single();

    if (!integration) {
      console.error(`Integration not found for subscription ${subscriptionId}`);
      continue;
    }

    if (changeType === 'created' || changeType === 'updated') {
      await handleMicrosoftEventChange(
        supabase,
        integration,
        resource
      );
    } else if (changeType === 'deleted') {
      await handleMicrosoftEventDeletion(
        supabase,
        integration,
        resource
      );
    }
  }

  return new Response('OK', { status: 200 });
});
```

### 6.3 Event Change Handlers

```typescript
async function handleGoogleEventChange(
  supabase: SupabaseClient,
  integration: any,
  eventId: string,
  accessToken: string
): Promise<void> {
  // Fetch event from Google
  const response = await fetch(
    `https://www.googleapis.com/calendar/v3/calendars/${integration.calendar_id}/events/${eventId}`,
    {
      headers: {
        'Authorization': `Bearer ${accessToken}`
      }
    }
  );

  if (!response.ok) {
    console.error(`Failed to fetch Google event ${eventId}`);
    return;
  }

  const googleEvent = await response.json();

  // Find calendar_events record
  const { data: calendarEvent } = await supabase
    .from('calendar_events')
    .select('*')
    .eq('calendar_integration_id', integration.id)
    .eq('provider_event_id', eventId)
    .single();

  if (!calendarEvent) {
    // External event not linked to internal assignment
    // Could be a new event created externally
    console.log(`External event ${eventId} not linked to internal assignment`);
    return;
  }

  // Check reconciliation mode
  const reconciliationMode = await getReconciliationMode(supabase, integration.org_id);

  if (reconciliationMode === 'apply') {
    // Apply external changes to internal schedule
    await applyExternalEventChanges(
      supabase,
      calendarEvent,
      googleEvent
    );
  } else {
    // Flag for manual reconciliation
    await flagForReconciliation(
      supabase,
      calendarEvent,
      googleEvent
    );
  }
}

async function handleGoogleEventDeletion(
  supabase: SupabaseClient,
  integration: any,
  eventId: string
): Promise<void> {
  const { data: calendarEvent } = await supabase
    .from('calendar_events')
    .select('*')
    .eq('calendar_integration_id', integration.id)
    .eq('provider_event_id', eventId)
    .single();

  if (!calendarEvent) {
    return;
  }

  const reconciliationMode = await getReconciliationMode(supabase, integration.org_id);

  if (reconciliationMode === 'apply') {
    // Mark assignment as canceled
    await supabase
      .from('job_assignments')
      .update({ status: 'canceled' })
      .eq('id', calendarEvent.job_assignment_id);

    await supabase
      .from('calendar_events')
      .update({
        status: 'deleted_by_user',
        last_synced_at: new Date().toISOString()
      })
      .eq('id', calendarEvent.id);
  } else {
    await supabase
      .from('calendar_events')
      .update({
        status: 'deleted_by_user',
        last_synced_at: new Date().toISOString()
      })
      .eq('id', calendarEvent.id);
  }
}

async function applyExternalEventChanges(
  supabase: SupabaseClient,
  calendarEvent: any,
  externalEvent: any
): Promise<void> {
  // Parse external event times
  const externalStart = new Date(externalEvent.start.dateTime || externalEvent.start.date);
  const externalEnd = new Date(externalEvent.end.dateTime || externalEvent.end.date);

  // Update assignment times
  await supabase
    .from('job_assignments')
    .update({
      scheduled_start_at: externalStart.toISOString(),
      scheduled_end_at: externalEnd.toISOString(),
      updated_at: new Date().toISOString()
    })
    .eq('id', calendarEvent.job_assignment_id);

  // Update calendar_events
  await supabase
    .from('calendar_events')
    .update({
      status: 'updated',
      last_synced_at: new Date().toISOString(),
      sync_direction: 'external_to_internal',
      metadata: {
        ...calendarEvent.metadata,
        external_change: {
          timestamp: new Date().toISOString(),
          old_start: calendarEvent.metadata?.last_sync?.scheduled_start_at,
          new_start: externalStart.toISOString(),
          old_end: calendarEvent.metadata?.last_sync?.scheduled_end_at,
          new_end: externalEnd.toISOString()
        }
      }
    })
    .eq('id', calendarEvent.id);

  // Create notification for dispatcher (deferred to Epic 10)
  // await createDispatcherNotification(...);
}

async function flagForReconciliation(
  supabase: SupabaseClient,
  calendarEvent: any,
  externalEvent: any
): Promise<void> {
  await supabase
    .from('calendar_events')
    .update({
      status: 'deleted_by_user',
      last_synced_at: new Date().toISOString(),
      metadata: {
        ...calendarEvent.metadata,
        external_change: {
          timestamp: new Date().toISOString(),
          external_event: externalEvent,
          requires_reconciliation: true
        }
      }
    })
    .eq('id', calendarEvent.id);
}

async function handleMicrosoftEventChange(
  supabase: SupabaseClient,
  integration: any,
  resource: string
): Promise<void> {
  // Similar to Google handler
  // Fetch event from Microsoft Graph
  // Apply or flag based on reconciliation mode
}

async function handleMicrosoftEventDeletion(
  supabase: SupabaseClient,
  integration: any,
  resource: string
): Promise<void> {
  // Similar to Google handler
}
```

### 6.4 Webhook Subscription Setup

**Google Calendar Subscription**:

```typescript
async function subscribeToGoogleCalendar(
  integration: any,
  accessToken: string
): Promise<string> {
  const channelId = crypto.randomUUID();
  const webhookUrl = Deno.env.get('GOOGLE_WEBHOOK_URL') || 'https://your-domain.com/webhooks/calendar/google';

  const response = await fetch(
    `https://www.googleapis.com/calendar/v3/calendars/${integration.calendar_id}/events/watch`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${accessToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        id: channelId,
        type: 'web_hook',
        address: webhookUrl,
        token: Deno.env.get('GOOGLE_WEBHOOK_TOKEN')
      })
    }
  );

  if (!response.ok) {
    const error = await response.text();
    throw new Error(`Failed to subscribe: ${error}`);
  }

  const data = await response.json();
  
  // Store channel ID and expiration in integration metadata
  return channelId;
}
```

**Microsoft Graph Subscription**:

```typescript
async function subscribeToMicrosoftCalendar(
  integration: any,
  accessToken: string
): Promise<string> {
  const webhookUrl = Deno.env.get('MICROSOFT_WEBHOOK_URL') || 'https://your-domain.com/webhooks/calendar/microsoft';

  const response = await fetch('https://graph.microsoft.com/v1.0/subscriptions', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${accessToken}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      changeType: 'created,updated,deleted',
      notificationUrl: webhookUrl,
      resource: `/me/calendars/${integration.calendar_id}/events`,
      expirationDateTime: new Date(Date.now() + 3 * 24 * 60 * 60 * 1000).toISOString(), // 3 days
      clientState: Deno.env.get('MICROSOFT_WEBHOOK_SECRET')
    })
  });

  if (!response.ok) {
    const error = await response.text();
    throw new Error(`Failed to subscribe: ${error}`);
  }

  const data = await response.json();
  return data.id; // Subscription ID
}
```

---

## 7. Story DISP-047: Calendar Reconciliation Modes

### 7.1 Reconciliation Mode Configuration

**Org-Level Setting**:

```sql
-- Add to orgs table or create org_settings table
ALTER TABLE orgs ADD COLUMN IF NOT EXISTS calendar_reconciliation_mode TEXT 
  DEFAULT 'flag' CHECK (calendar_reconciliation_mode IN ('apply', 'flag'));

-- Or create separate settings table
CREATE TABLE IF NOT EXISTS org_calendar_settings (
  org_id UUID PRIMARY KEY REFERENCES orgs(id) ON DELETE CASCADE,
  reconciliation_mode TEXT NOT NULL DEFAULT 'flag' CHECK (reconciliation_mode IN ('apply', 'flag')),
  auto_notify_dispatchers BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Reconciliation Modes**:

1. **`apply`**: Automatically apply external calendar changes to internal schedule
   - Update assignment times when event moved
   - Cancel assignment when event deleted
   - Notify dispatchers of changes

2. **`flag`**: Flag external changes for manual reconciliation
   - Mark `calendar_events.status = 'deleted_by_user'`
   - Store external change details in metadata
   - Require dispatcher approval before applying

**Get Reconciliation Mode**:

```typescript
async function getReconciliationMode(
  supabase: SupabaseClient,
  orgId: string
): Promise<'apply' | 'flag'> {
  const { data: setting } = await supabase
    .from('org_calendar_settings')
    .select('reconciliation_mode')
    .eq('org_id', orgId)
    .single();

  return setting?.reconciliation_mode || 'flag'; // Default to 'flag'
}
```

### 7.2 Audit Trail for External Changes

**Audit Record Structure**:

```typescript
interface ExternalChangeAudit {
  calendar_event_id: string;
  change_type: 'moved' | 'deleted' | 'updated';
  old_scheduled_start_at: string | null;
  new_scheduled_start_at: string | null;
  old_scheduled_end_at: string | null;
  new_scheduled_end_at: string | null;
  reconciliation_mode: 'apply' | 'flag';
  applied: boolean;
  applied_by_user_id: string | null;
  applied_at: string | null;
  external_event_snapshot: any;
}
```

**Store in `calendar_events.metadata`**:

```typescript
{
  external_changes: [
    {
      timestamp: "2024-01-20T10:00:00Z",
      change_type: "moved",
      old_start: "2024-01-20T09:00:00Z",
      new_start: "2024-01-20T10:00:00Z",
      reconciliation_mode: "apply",
      applied: true,
      applied_by: "user_id",
      applied_at: "2024-01-20T10:05:00Z"
    }
  ]
}
```

---

## 8. Error Handling

### 8.1 Standard Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid authentication token |
| `FORBIDDEN` | 403 | User not authorized |
| `INVALID_PROVIDER` | 400 | Invalid calendar provider |
| `INVALID_STATE` | 400 | Invalid or expired OAuth state |
| `OAUTH_ERROR` | 400 | OAuth provider error |
| `TOKEN_REFRESH_FAILED` | 500 | Failed to refresh access token |
| `SYNC_ERROR` | 500 | Calendar sync failed |

---

## 9. Implementation Checklist

### Story DISP-044: Calendar Connect Endpoint

- [ ] **POST /dispatch/calendar/connect**:
  - [ ] Endpoint implemented
  - [ ] Google OAuth flow implemented
  - [ ] Microsoft OAuth flow implemented
  - [ ] PKCE implementation
  - [ ] State management implemented
  - [ ] OAuth state table created
  - [ ] Security considerations documented
  - [ ] API documentation with examples

- [ ] **OAuth Callback Handler**:
  - [ ] GET /dispatch/calendar/oauth/callback/:provider implemented
  - [ ] State validation implemented
  - [ ] Token exchange implemented
  - [ ] Token encryption implemented
  - [ ] Integration storage implemented
  - [ ] Error handling implemented

- [ ] **Security**:
  - [ ] CSRF protection (state parameter)
  - [ ] PKCE implementation
  - [ ] Token encryption at rest
  - [ ] Least scope principle
  - [ ] Security documentation

### Story DISP-045: Sync Internal Appointments

- [ ] **POST /dispatch/calendar/sync**:
  - [ ] Endpoint implemented
  - [ ] Incremental sync logic implemented
  - [ ] Token refresh logic implemented
  - [ ] Event creation implemented
  - [ ] Event update implemented
  - [ ] Event cancellation implemented
  - [ ] Idempotency ensured
  - [ ] Error handling implemented
  - [ ] API documentation with examples

- [ ] **Provider Integration**:
  - [ ] Google Calendar API integration
  - [ ] Microsoft Graph API integration
  - [ ] Event data mapping
  - [ ] Error handling for API failures

### Story DISP-046: Calendar Provider Webhooks

- [ ] **POST /webhooks/calendar/google**:
  - [ ] Webhook endpoint implemented
  - [ ] Webhook verification implemented
  - [ ] Event change handler implemented
  - [ ] Event deletion handler implemented
  - [ ] Error handling implemented

- [ ] **POST /webhooks/calendar/microsoft**:
  - [ ] Webhook endpoint implemented
  - [ ] Subscription validation implemented
  - [ ] Signature verification implemented
  - [ ] Event change handler implemented
  - [ ] Event deletion handler implemented
  - [ ] Error handling implemented

- [ ] **Webhook Subscription**:
  - [ ] Google subscription setup implemented
  - [ ] Microsoft subscription setup implemented
  - [ ] Subscription renewal logic documented

### Story DISP-047: Calendar Reconciliation Modes

- [ ] **Reconciliation Mode Configuration**:
  - [ ] Org-level setting implemented
  - [ ] `apply` mode implementation
  - [ ] `flag` mode implementation
  - [ ] Mode retrieval function implemented

- [ ] **Audit Trail**:
  - [ ] External change tracking implemented
  - [ ] Audit record structure defined
  - [ ] Metadata storage implemented

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 9 – Calendar Integration. All APIs are designed as Supabase Edge Functions with complete OAuth flows, secure token handling, calendar synchronization, webhook processing, and reconciliation strategies.

**Next Steps**: After completing Epic 9, proceed to Epic 10 (Notifications & Reminder Orchestration) which will implement notification scheduling and dispatch.

