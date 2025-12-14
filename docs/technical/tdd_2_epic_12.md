# Technical Design Document – Epic 12: Customer Portal and Booking Hooks (Scheduling-Adjacent)

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 12 – Customer Portal and Booking Hooks (Scheduling-Adjacent)
- **Source**: Derived from `fdd_2_agile.md` Epic 12 (Stories DISP-056 through DISP-057)
- **Reference Documents**: 
  - `fdd_2.md` (Functional Design Document for Scheduling & Dispatch)
  - `fdd_2_agile.md` (Agile User Stories)
  - `tdd_2_epic_1.md` (Dispatch Epic 1 for tenancy/role conventions)
  - `tdd_2_epic_2.md` (Dispatch Epic 2 for table schemas)
  - `tdd_2_epic_3.md` (Dispatch Epic 3 for RLS policies - portal access strategy)
  - `tdd_2_epic_5.md` (Dispatch Epic 5 for job lifecycle APIs)
  - `tdd_2_epic_11.md` (Dispatch Epic 11 for Next.js UI patterns)
- **Target Platform**: Next.js 14+ (App Router), React 18+, shadcn/ui, Supabase Edge Functions
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing Customer Portal and Booking Hooks for scheduling. It covers:

- Portal-safe APIs for listing and selecting appointment time windows
- Customer appointment status and ETA read endpoints
- Next.js UI components for customer portal using shadcn/ui
- Real-time updates for appointment status and ETA
- Security patterns for customer-scoped access

All APIs are implemented as Supabase Edge Functions with strict customer authorization, and UI components are built using Next.js 14+ App Router with shadcn/ui components.

This epic assumes Epic 1-11 are complete and customer authentication/portal user mapping is established.

---

## 2. Prerequisites

### 2.1 Required Components

Before implementing Epic 12, ensure:

1. **Epic 1-11 Complete**: All previous epics are implemented
2. **Customer Portal Authentication**:
   - Customer users can authenticate via Supabase Auth
   - `portal_user_contacts` table exists mapping `auth.users.id` to `customer_contacts.id`
   - Customer role is assigned in `profiles` table

3. **Required Tables**:
   - `dispatch_jobs`
   - `job_time_windows`
   - `job_assignments`
   - `customers`
   - `customer_contacts`
   - `portal_user_contacts` (from Portal module)

4. **Next.js Project Setup**:
   - Next.js 14+ with App Router
   - React 18+
   - TypeScript
   - Tailwind CSS
   - shadcn/ui installed and configured

### 2.2 Customer Portal User Mapping

**Table**: `portal_user_contacts` (from Portal module)

```sql
CREATE TABLE IF NOT EXISTS portal_user_contacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  contact_id UUID NOT NULL REFERENCES customer_contacts(id) ON DELETE CASCADE,
  is_primary BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  UNIQUE(user_id, contact_id)
);

CREATE INDEX idx_portal_user_contacts_user_id ON portal_user_contacts(user_id);
CREATE INDEX idx_portal_user_contacts_contact_id ON portal_user_contacts(contact_id);
```

**Purpose**: Maps authenticated portal users to customer contacts, enabling customer-scoped data access.

---

## 3. Common Patterns

### 3.1 Customer Authorization Helper

**File**: `supabase/functions/_shared/customer-auth.ts`

```typescript
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

interface CustomerAuthResult {
  userId: string;
  customerId: string;
  contactId: string;
  orgId: string;
}

/**
 * Verify user is authenticated and get customer information
 */
export async function authorizeCustomer(
  supabase: SupabaseClient,
  authHeader: string | null
): Promise<CustomerAuthResult | null> {
  if (!authHeader) {
    return null;
  }

  const { data: { user }, error: userError } = await supabase.auth.getUser(
    authHeader.replace('Bearer ', '')
  );

  if (userError || !user) {
    return null;
  }

  // Get customer contact mapping
  const { data: portalMapping, error: mappingError } = await supabase
    .from('portal_user_contacts')
    .select('contact_id, customer_contacts!inner(customer_id, org_id)')
    .eq('user_id', user.id)
    .eq('is_primary', true)
    .single();

  if (mappingError || !portalMapping) {
    return null;
  }

  // Verify user has customer role
  const { data: profile, error: profileError } = await supabase
    .from('profiles')
    .select('role, org_id')
    .eq('user_id', user.id)
    .single();

  if (profileError || !profile || profile.role !== 'customer') {
    return null;
  }

  return {
    userId: user.id,
    customerId: portalMapping.customer_contacts.customer_id,
    contactId: portalMapping.contact_id,
    orgId: profile.org_id
  };
}

/**
 * Verify customer owns a job
 */
export async function verifyCustomerOwnsJob(
  supabase: SupabaseClient,
  jobId: string,
  customerId: string,
  orgId: string
): Promise<boolean> {
  const { data: job, error } = await supabase
    .from('dispatch_jobs')
    .select('customer_id, org_id')
    .eq('id', jobId)
    .eq('customer_id', customerId)
    .eq('org_id', orgId)
    .single();

  return !error && job !== null;
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

---

## 4. Story DISP-056: Expose Appointment Time Windows for Portal Selection

### 4.1 GET /portal/jobs/:job_id/time-windows

**Purpose**: List available time windows for a customer's job.

**Authorization**: Customer (must own the job)

**Edge Function**: `portal-list-time-windows`

**Request**: No body, job ID in URL path

**Response Schema**:

```typescript
interface TimeWindowResponse {
  id: string;
  window_start: string; // ISO 8601 timestamp
  window_end: string; // ISO 8601 timestamp
  source: 'system_suggested' | 'dispatcher_selected' | 'customer_selected';
  is_selected: boolean;
  is_available: boolean; // true if window hasn't passed and job isn't assigned
  created_at: string;
}
```

**Response Example**:

```json
{
  "data": {
    "job_id": "444e4567-e89b-12d3-a456-426614174000",
    "time_windows": [
      {
        "id": "666e4567-e89b-12d3-a456-426614174000",
        "window_start": "2024-01-22T09:00:00Z",
        "window_end": "2024-01-22T11:00:00Z",
        "source": "system_suggested",
        "is_selected": false,
        "is_available": true,
        "created_at": "2024-01-15T10:30:00Z"
      },
      {
        "id": "777e4567-e89b-12d3-a456-426614174000",
        "window_start": "2024-01-22T13:00:00Z",
        "window_end": "2024-01-22T15:00:00Z",
        "source": "system_suggested",
        "is_selected": true,
        "is_available": true,
        "created_at": "2024-01-15T10:30:00Z"
      }
    ]
  }
}
```

**Implementation** (Edge Function):

```typescript
Deno.serve(async (req) => {
  if (req.method !== 'GET') {
    return errorResponse('Method not allowed', 405);
  }

  const authHeader = req.headers.get('Authorization');
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_ANON_KEY')!,
    { global: { headers: { Authorization: authHeader } } }
  );

  const customerAuth = await authorizeCustomer(supabase, authHeader);
  if (!customerAuth) {
    return errorResponse('Unauthorized', 401);
  }

  // Extract job ID from URL
  const url = new URL(req.url);
  const jobId = url.pathname.split('/').pop();

  if (!jobId) {
    return errorResponse('Job ID required', 400, 'MISSING_JOB_ID');
  }

  // Verify customer owns the job
  const ownsJob = await verifyCustomerOwnsJob(
    supabase,
    jobId,
    customerAuth.customerId,
    customerAuth.orgId
  );

  if (!ownsJob) {
    return errorResponse('Job not found or access denied', 403, 'FORBIDDEN');
  }

  // Get time windows for the job
  const { data: timeWindows, error: windowsError } = await supabase
    .from('job_time_windows')
    .select('*')
    .eq('dispatch_job_id', jobId)
    .eq('org_id', customerAuth.orgId)
    .order('window_start', { ascending: true });

  if (windowsError) {
    return errorResponse('Failed to fetch time windows', 500, 'FETCH_ERROR');
  }

  // Check if job is already assigned (windows not available if assigned)
  const { data: assignments } = await supabase
    .from('job_assignments')
    .select('id')
    .eq('dispatch_job_id', jobId)
    .eq('org_id', customerAuth.orgId)
    .in('status', ['assigned', 'accepted', 'en_route', 'on_site'])
    .limit(1);

  const isAssigned = assignments && assignments.length > 0;
  const now = new Date();

  // Enrich with availability status
  const enrichedWindows = (timeWindows || []).map((window: any) => ({
    ...window,
    is_available: !isAssigned && new Date(window.window_start) > now
  }));

  return successResponse({
    job_id: jobId,
    time_windows: enrichedWindows
  });
});
```

### 4.2 POST /portal/jobs/:job_id/time-windows/:window_id/select

**Purpose**: Select a time window for a customer's job.

**Authorization**: Customer (must own the job)

**Edge Function**: `portal-select-time-window`

**Request Schema**:

```typescript
interface SelectTimeWindowRequest {
  trigger_auto_schedule?: boolean; // default: false, trigger auto-scheduling after selection
}
```

**Request Example**:

```json
{
  "trigger_auto_schedule": true
}
```

**Response Schema**:

```typescript
interface SelectTimeWindowResponse {
  job_id: string;
  selected_window_id: string;
  auto_scheduled: boolean;
  assignment_id?: string; // If auto-scheduled
}
```

**Response Example**:

```json
{
  "data": {
    "job_id": "444e4567-e89b-12d3-a456-426614174000",
    "selected_window_id": "777e4567-e89b-12d3-a456-426614174000",
    "auto_scheduled": true,
    "assignment_id": "999e4567-e89b-12d3-a456-426614174000"
  }
}
```

**Implementation** (Edge Function):

```typescript
Deno.serve(async (req) => {
  if (req.method !== 'POST') {
    return errorResponse('Method not allowed', 405);
  }

  const authHeader = req.headers.get('Authorization');
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_ANON_KEY')!,
    { global: { headers: { Authorization: authHeader } } }
  );

  const customerAuth = await authorizeCustomer(supabase, authHeader);
  if (!customerAuth) {
    return errorResponse('Unauthorized', 401);
  }

  // Extract IDs from URL
  const url = new URL(req.url);
  const pathParts = url.pathname.split('/');
  const jobId = pathParts[pathParts.length - 3];
  const windowId = pathParts[pathParts.length - 1];

  if (!jobId || !windowId) {
    return errorResponse('Job ID and Window ID required', 400, 'MISSING_IDS');
  }

  // Verify customer owns the job
  const ownsJob = await verifyCustomerOwnsJob(
    supabase,
    jobId,
    customerAuth.customerId,
    customerAuth.orgId
  );

  if (!ownsJob) {
    return errorResponse('Job not found or access denied', 403, 'FORBIDDEN');
  }

  // Verify window belongs to job
  const { data: window, error: windowError } = await supabase
    .from('job_time_windows')
    .select('*')
    .eq('id', windowId)
    .eq('dispatch_job_id', jobId)
    .eq('org_id', customerAuth.orgId)
    .single();

  if (windowError || !window) {
    return errorResponse('Time window not found', 404, 'WINDOW_NOT_FOUND');
  }

  // Check if window is still available
  if (new Date(window.window_start) < new Date()) {
    return errorResponse('Time window has passed', 400, 'WINDOW_EXPIRED');
  }

  // Check if job is already assigned
  const { data: existingAssignments } = await supabase
    .from('job_assignments')
    .select('id')
    .eq('dispatch_job_id', jobId)
    .eq('org_id', customerAuth.orgId)
    .in('status', ['assigned', 'accepted', 'en_route', 'on_site'])
    .limit(1);

  if (existingAssignments && existingAssignments.length > 0) {
    return errorResponse('Job is already assigned', 400, 'ALREADY_ASSIGNED');
  }

  const body = await req.json();
  const triggerAutoSchedule = body.trigger_auto_schedule || false;

  // Unselect all other windows
  await supabase
    .from('job_time_windows')
    .update({ is_selected: false })
    .eq('dispatch_job_id', jobId)
    .eq('org_id', customerAuth.orgId);

  // Select the chosen window
  const { error: updateError } = await supabase
    .from('job_time_windows')
    .update({
      is_selected: true,
      source: 'customer_selected'
    })
    .eq('id', windowId);

  if (updateError) {
    return errorResponse('Failed to select time window', 500, 'UPDATE_ERROR');
  }

  let assignmentId: string | undefined;
  let autoScheduled = false;

  // Trigger auto-scheduling if requested
  if (triggerAutoSchedule) {
    try {
      // Call auto-schedule Edge Function
      const autoScheduleResponse = await fetch(
        `${Deno.env.get('SUPABASE_URL')}/functions/v1/dispatch-auto-schedule`,
        {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')}`,
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({
            job_id: jobId,
            mode: 'commit'
          })
        }
      );

      if (autoScheduleResponse.ok) {
        const autoScheduleData = await autoScheduleResponse.json();
        assignmentId = autoScheduleData.data?.assignment_id;
        autoScheduled = true;
      }
    } catch (error) {
      console.error('Auto-schedule failed:', error);
      // Continue without failing the window selection
    }
  }

  return successResponse({
    job_id: jobId,
    selected_window_id: windowId,
    auto_scheduled: autoScheduled,
    assignment_id: assignmentId
  });
});
```

### 4.3 Next.js UI Component: Time Window Selection

**File**: `app/portal/appointments/[jobId]/time-windows/page.tsx`

```typescript
'use client';

import { useState } from 'react';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { useParams, useRouter } from 'next/navigation';
import { createSupabaseClient } from '@/lib/supabase/client';
import { Button } from '@/components/ui/button';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Skeleton } from '@/components/ui/skeleton';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { format, parseISO, isPast } from 'date-fns';
import { Calendar, Clock, CheckCircle2, AlertCircle } from 'lucide-react';
import { toast } from 'sonner';
import { cn } from '@/lib/utils';

interface TimeWindow {
  id: string;
  window_start: string;
  window_end: string;
  source: string;
  is_selected: boolean;
  is_available: boolean;
}

export default function TimeWindowSelectionPage() {
  const params = useParams();
  const router = useRouter();
  const jobId = params.jobId as string;
  const [selectedWindowId, setSelectedWindowId] = useState<string | null>(null);
  const [triggerAutoSchedule, setTriggerAutoSchedule] = useState(false);

  const supabase = createSupabaseClient();

  const { data, isLoading, error } = useQuery({
    queryKey: ['time-windows', jobId],
    queryFn: async () => {
      const { data: { session } } = await supabase.auth.getSession();
      
      const response = await fetch(`/api/portal/jobs/${jobId}/time-windows`, {
        headers: {
          'Authorization': `Bearer ${session?.access_token}`
        }
      });

      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.error?.message || 'Failed to load time windows');
      }

      return response.json();
    }
  });

  const selectMutation = useMutation({
    mutationFn: async (windowId: string) => {
      const { data: { session } } = await supabase.auth.getSession();
      
      const response = await fetch(`/api/portal/jobs/${jobId}/time-windows/${windowId}/select`, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${session?.access_token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          trigger_auto_schedule: triggerAutoSchedule
        })
      });

      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.error?.message || 'Failed to select time window');
      }

      return response.json();
    },
    onSuccess: (data) => {
      toast.success('Time window selected successfully');
      if (data.data.auto_scheduled) {
        toast.info('Appointment has been automatically scheduled');
        router.push(`/portal/appointments/${jobId}`);
      } else {
        router.refresh();
      }
    },
    onError: (error: Error) => {
      toast.error(error.message);
    }
  });

  if (isLoading) {
    return (
      <div className="container mx-auto py-8 space-y-4">
        <Skeleton className="h-12 w-64" />
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
          {[1, 2, 3].map((i) => (
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
          <AlertCircle className="h-4 w-4" />
          <AlertDescription>
            Failed to load time windows. Please try again.
          </AlertDescription>
        </Alert>
      </div>
    );
  }

  const timeWindows: TimeWindow[] = data?.data?.time_windows || [];
  const hasSelectedWindow = timeWindows.some(w => w.is_selected);

  return (
    <div className="container mx-auto py-8 max-w-4xl">
      <div className="mb-6">
        <h1 className="text-2xl font-bold mb-2">Select Appointment Time</h1>
        <p className="text-muted-foreground">
          Choose a time window that works best for you
        </p>
      </div>

      {hasSelectedWindow && (
        <Alert className="mb-6">
          <CheckCircle2 className="h-4 w-4" />
          <AlertDescription>
            You have already selected a time window. You can change your selection below.
          </AlertDescription>
        </Alert>
      )}

      <div className="mb-4">
        <label className="flex items-center gap-2">
          <input
            type="checkbox"
            checked={triggerAutoSchedule}
            onChange={(e) => setTriggerAutoSchedule(e.target.checked)}
            className="rounded"
          />
          <span className="text-sm">Automatically schedule appointment after selection</span>
        </label>
      </div>

      {timeWindows.length === 0 ? (
        <Card>
          <CardContent className="py-8 text-center text-muted-foreground">
            No time windows available. Please contact support.
          </CardContent>
        </Card>
      ) : (
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
          {timeWindows.map((window) => {
            const start = parseISO(window.window_start);
            const end = parseISO(window.window_end);
            const isSelected = window.is_selected;
            const isUnavailable = !window.is_available || isPast(end);

            return (
              <Card
                key={window.id}
                className={cn(
                  'cursor-pointer transition-all hover:shadow-md',
                  isSelected && 'ring-2 ring-primary',
                  isUnavailable && 'opacity-50 cursor-not-allowed'
                )}
                onClick={() => {
                  if (!isUnavailable) {
                    setSelectedWindowId(window.id);
                    selectMutation.mutate(window.id);
                  }
                }}
              >
                <CardHeader>
                  <CardTitle className="text-lg flex items-center justify-between">
                    <span className="flex items-center gap-2">
                      <Calendar className="h-4 w-4" />
                      {format(start, 'MMM d')}
                    </span>
                    {isSelected && (
                      <Badge variant="default">
                        <CheckCircle2 className="h-3 w-3 mr-1" />
                        Selected
                      </Badge>
                    )}
                  </CardTitle>
                  <CardDescription>
                    {format(start, 'EEEE')}
                  </CardDescription>
                </CardHeader>
                <CardContent>
                  <div className="flex items-center gap-2 text-sm">
                    <Clock className="h-4 w-4 text-muted-foreground" />
                    <span>
                      {format(start, 'h:mm a')} - {format(end, 'h:mm a')}
                    </span>
                  </div>
                  {isUnavailable && !isSelected && (
                    <Badge variant="secondary" className="mt-2">
                      Unavailable
                    </Badge>
                  )}
                  {!isUnavailable && !isSelected && (
                    <Button
                      className="w-full mt-4"
                      variant="outline"
                      disabled={selectMutation.isPending}
                      onClick={(e) => {
                        e.stopPropagation();
                        setSelectedWindowId(window.id);
                        selectMutation.mutate(window.id);
                      }}
                    >
                      {selectMutation.isPending && selectedWindowId === window.id
                        ? 'Selecting...'
                        : 'Select This Time'}
                    </Button>
                  )}
                </CardContent>
              </Card>
            );
          })}
        </div>
      )}
    </div>
  );
}
```

### 4.4 API Route: Time Windows

**File**: `app/api/portal/jobs/[jobId]/time-windows/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';

export async function GET(
  request: NextRequest,
  { params }: { params: { jobId: string } }
) {
  const supabase = createRouteHandlerClient({ cookies });
  
  const {
    data: { session },
  } = await supabase.auth.getSession();

  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Call Edge Function
  const { data, error } = await supabase.functions.invoke('portal-list-time-windows', {
    body: {
      job_id: params.jobId,
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

**File**: `app/api/portal/jobs/[jobId]/time-windows/[windowId]/select/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';

export async function POST(
  request: NextRequest,
  { params }: { params: { jobId: string; windowId: string } }
) {
  const supabase = createRouteHandlerClient({ cookies });
  
  const {
    data: { session },
  } = await supabase.auth.getSession();

  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const body = await request.json();

  // Call Edge Function
  const { data, error } = await supabase.functions.invoke('portal-select-time-window', {
    body: {
      job_id: params.jobId,
      window_id: params.windowId,
      trigger_auto_schedule: body.trigger_auto_schedule || false,
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

## 5. Story DISP-057: Customer Appointment Status + ETA Read Model

### 5.1 GET /portal/appointments

**Purpose**: List all appointments for the authenticated customer.

**Authorization**: Customer

**Edge Function**: `portal-list-appointments`

**Query Parameters**:

```typescript
interface ListAppointmentsQuery {
  status?: 'unscheduled' | 'scheduled' | 'in_progress' | 'completed' | 'canceled';
  limit?: number; // default: 50, max: 100
  offset?: number; // default: 0
}
```

**Response Schema**:

```typescript
interface AppointmentResponse {
  job_id: string;
  title: string;
  description: string | null;
  status: string;
  priority: string;
  scheduled_start_at: string | null; // From assignment
  scheduled_end_at: string | null; // From assignment
  arrival_window_start: string | null; // From assignment
  arrival_window_end: string | null; // From assignment
  tech_eta_at: string | null; // Current ETA from technician
  technician_name: string | null;
  location: {
    address_line1: string;
    city: string;
    state: string;
    postal_code: string;
  };
  time_windows: Array<{
    id: string;
    window_start: string;
    window_end: string;
    is_selected: boolean;
  }>;
  created_at: string;
  updated_at: string;
}
```

**Response Example**:

```json
{
  "data": {
    "appointments": [
      {
        "job_id": "444e4567-e89b-12d3-a456-426614174000",
        "title": "AC Maintenance",
        "description": "Routine AC maintenance check",
        "status": "scheduled",
        "priority": "normal",
        "scheduled_start_at": "2024-01-22T10:00:00Z",
        "scheduled_end_at": "2024-01-22T11:00:00Z",
        "arrival_window_start": "2024-01-22T10:00:00Z",
        "arrival_window_end": "2024-01-22T10:30:00Z",
        "tech_eta_at": "2024-01-22T10:15:00Z",
        "technician_name": "John Smith",
        "location": {
          "address_line1": "123 Main St",
          "city": "Springfield",
          "state": "IL",
          "postal_code": "62701"
        },
        "time_windows": [
          {
            "id": "777e4567-e89b-12d3-a456-426614174000",
            "window_start": "2024-01-22T09:00:00Z",
            "window_end": "2024-01-22T11:00:00Z",
            "is_selected": true
          }
        ],
        "created_at": "2024-01-15T10:30:00Z",
        "updated_at": "2024-01-20T09:15:00Z"
      }
    ],
    "pagination": {
      "total": 1,
      "limit": 50,
      "offset": 0,
      "has_more": false
    }
  }
}
```

**Implementation** (Edge Function):

```typescript
Deno.serve(async (req) => {
  if (req.method !== 'GET') {
    return errorResponse('Method not allowed', 405);
  }

  const authHeader = req.headers.get('Authorization');
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_ANON_KEY')!,
    { global: { headers: { Authorization: authHeader } } }
  );

  const customerAuth = await authorizeCustomer(supabase, authHeader);
  if (!customerAuth) {
    return errorResponse('Unauthorized', 401);
  }

  const url = new URL(req.url);
  const statusFilter = url.searchParams.get('status');
  const limit = Math.min(parseInt(url.searchParams.get('limit') || '50'), 100);
  const offset = parseInt(url.searchParams.get('offset') || '0');

  // Get customer's jobs
  let jobsQuery = supabase
    .from('dispatch_jobs')
    .select(`
      id,
      title,
      description,
      status,
      priority,
      created_at,
      updated_at,
      customer_locations!inner(
        address_line1,
        city,
        state,
        postal_code
      ),
      job_time_windows(
        id,
        window_start,
        window_end,
        is_selected
      ),
      job_assignments(
        id,
        scheduled_start_at,
        scheduled_end_at,
        arrival_window_start,
        arrival_window_end,
        status,
        tech_eta_at,
        technician_profiles(
          display_name
        )
      )
    `)
    .eq('customer_id', customerAuth.customerId)
    .eq('org_id', customerAuth.orgId)
    .order('created_at', { ascending: false })
    .range(offset, offset + limit - 1);

  if (statusFilter) {
    jobsQuery = jobsQuery.eq('status', statusFilter);
  }

  const { data: jobs, error: jobsError } = await jobsQuery;

  if (jobsError) {
    return errorResponse('Failed to fetch appointments', 500, 'FETCH_ERROR');
  }

  // Get total count for pagination
  let countQuery = supabase
    .from('dispatch_jobs')
    .select('id', { count: 'exact', head: true })
    .eq('customer_id', customerAuth.customerId)
    .eq('org_id', customerAuth.orgId);

  if (statusFilter) {
    countQuery = countQuery.eq('status', statusFilter);
  }

  const { count } = await countQuery;

  // Transform to appointment response format
  const appointments = (jobs || []).map((job: any) => {
    // Get primary assignment (first active assignment)
    const primaryAssignment = job.job_assignments?.find((a: any) =>
      ['assigned', 'accepted', 'en_route', 'on_site'].includes(a.status)
    ) || job.job_assignments?.[0];

    return {
      job_id: job.id,
      title: job.title,
      description: job.description,
      status: job.status,
      priority: job.priority,
      scheduled_start_at: primaryAssignment?.scheduled_start_at || null,
      scheduled_end_at: primaryAssignment?.scheduled_end_at || null,
      arrival_window_start: primaryAssignment?.arrival_window_start || null,
      arrival_window_end: primaryAssignment?.arrival_window_end || null,
      tech_eta_at: primaryAssignment?.tech_eta_at || null,
      technician_name: primaryAssignment?.technician_profiles?.display_name || null,
      location: {
        address_line1: job.customer_locations.address_line1,
        city: job.customer_locations.city,
        state: job.customer_locations.state,
        postal_code: job.customer_locations.postal_code
      },
      time_windows: job.job_time_windows || [],
      created_at: job.created_at,
      updated_at: job.updated_at
    };
  });

  return successResponse({
    appointments,
    pagination: {
      total: count || 0,
      limit,
      offset,
      has_more: (count || 0) > offset + limit
    }
  });
});
```

### 5.2 GET /portal/appointments/:job_id

**Purpose**: Get detailed appointment information including status and ETA.

**Authorization**: Customer (must own the job)

**Edge Function**: `portal-get-appointment`

**Response Schema**: Same as single appointment from list endpoint

**Implementation** (Edge Function):

```typescript
Deno.serve(async (req) => {
  if (req.method !== 'GET') {
    return errorResponse('Method not allowed', 405);
  }

  const authHeader = req.headers.get('Authorization');
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_ANON_KEY')!,
    { global: { headers: { Authorization: authHeader } } }
  );

  const customerAuth = await authorizeCustomer(supabase, authHeader);
  if (!customerAuth) {
    return errorResponse('Unauthorized', 401);
  }

  const url = new URL(req.url);
  const jobId = url.pathname.split('/').pop();

  if (!jobId) {
    return errorResponse('Job ID required', 400, 'MISSING_JOB_ID');
  }

  // Verify customer owns the job
  const ownsJob = await verifyCustomerOwnsJob(
    supabase,
    jobId,
    customerAuth.customerId,
    customerAuth.orgId
  );

  if (!ownsJob) {
    return errorResponse('Appointment not found or access denied', 403, 'FORBIDDEN');
  }

  // Get job details
  const { data: job, error: jobError } = await supabase
    .from('dispatch_jobs')
    .select(`
      id,
      title,
      description,
      status,
      priority,
      created_at,
      updated_at,
      customer_locations!inner(
        address_line1,
        city,
        state,
        postal_code
      ),
      job_time_windows(
        id,
        window_start,
        window_end,
        is_selected
      ),
      job_assignments(
        id,
        scheduled_start_at,
        scheduled_end_at,
        arrival_window_start,
        arrival_window_end,
        status,
        tech_eta_at,
        technician_profiles(
          display_name
        )
      )
    `)
    .eq('id', jobId)
    .single();

  if (jobError || !job) {
    return errorResponse('Appointment not found', 404, 'NOT_FOUND');
  }

  // Transform to appointment response format
  const primaryAssignment = job.job_assignments?.find((a: any) =>
    ['assigned', 'accepted', 'en_route', 'on_site'].includes(a.status)
  ) || job.job_assignments?.[0];

  const appointment = {
    job_id: job.id,
    title: job.title,
    description: job.description,
    status: job.status,
    priority: job.priority,
    scheduled_start_at: primaryAssignment?.scheduled_start_at || null,
    scheduled_end_at: primaryAssignment?.scheduled_end_at || null,
    arrival_window_start: primaryAssignment?.arrival_window_start || null,
    arrival_window_end: primaryAssignment?.arrival_window_end || null,
    tech_eta_at: primaryAssignment?.tech_eta_at || null,
    technician_name: primaryAssignment?.technician_profiles?.display_name || null,
    location: {
      address_line1: job.customer_locations.address_line1,
      city: job.customer_locations.city,
      state: job.customer_locations.state,
      postal_code: job.customer_locations.postal_code
    },
    time_windows: job.job_time_windows || [],
    created_at: job.created_at,
    updated_at: job.updated_at
  };

  return successResponse({ appointment });
});
```

### 5.3 Next.js UI Component: Appointment List

**File**: `app/portal/appointments/page.tsx`

```typescript
'use client';

import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { useRouter } from 'next/navigation';
import { createSupabaseClient } from '@/lib/supabase/client';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { Skeleton } from '@/components/ui/skeleton';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { format, parseISO } from 'date-fns';
import { Calendar, Clock, MapPin, User, AlertCircle, ChevronRight } from 'lucide-react';
import { cn } from '@/lib/utils';

interface Appointment {
  job_id: string;
  title: string;
  description: string | null;
  status: string;
  priority: string;
  scheduled_start_at: string | null;
  scheduled_end_at: string | null;
  arrival_window_start: string | null;
  arrival_window_end: string | null;
  tech_eta_at: string | null;
  technician_name: string | null;
  location: {
    address_line1: string;
    city: string;
    state: string;
    postal_code: string;
  };
  time_windows: Array<{
    id: string;
    window_start: string;
    window_end: string;
    is_selected: boolean;
  }>;
  created_at: string;
  updated_at: string;
}

export default function AppointmentsPage() {
  const router = useRouter();
  const [statusFilter, setStatusFilter] = useState<string>('all');
  const supabase = createSupabaseClient();

  const { data, isLoading, error } = useQuery({
    queryKey: ['appointments', statusFilter],
    queryFn: async () => {
      const { data: { session } } = await supabase.auth.getSession();
      
      const params = new URLSearchParams();
      if (statusFilter !== 'all') {
        params.append('status', statusFilter);
      }

      const response = await fetch(`/api/portal/appointments?${params.toString()}`, {
        headers: {
          'Authorization': `Bearer ${session?.access_token}`
        }
      });

      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.error?.message || 'Failed to load appointments');
      }

      return response.json();
    }
  });

  const appointments: Appointment[] = data?.data?.appointments || [];

  const getStatusColor = (status: string) => {
    switch (status) {
      case 'scheduled':
        return 'bg-blue-500';
      case 'in_progress':
        return 'bg-yellow-500';
      case 'completed':
        return 'bg-green-500';
      case 'canceled':
        return 'bg-red-500';
      default:
        return 'bg-gray-500';
    }
  };

  if (isLoading) {
    return (
      <div className="container mx-auto py-8 space-y-4">
        <Skeleton className="h-12 w-64" />
        {[1, 2, 3].map((i) => (
          <Skeleton key={i} className="h-32" />
        ))}
      </div>
    );
  }

  if (error) {
    return (
      <div className="container mx-auto py-8">
        <Alert variant="destructive">
          <AlertCircle className="h-4 w-4" />
          <AlertDescription>
            Failed to load appointments. Please try again.
          </AlertDescription>
        </Alert>
      </div>
    );
  }

  return (
    <div className="container mx-auto py-8 max-w-6xl">
      <div className="flex items-center justify-between mb-6">
        <div>
          <h1 className="text-2xl font-bold mb-2">My Appointments</h1>
          <p className="text-muted-foreground">
            View and manage your scheduled appointments
          </p>
        </div>
        <Select value={statusFilter} onValueChange={setStatusFilter}>
          <SelectTrigger className="w-[180px]">
            <SelectValue placeholder="Filter by status" />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="all">All Statuses</SelectItem>
            <SelectItem value="unscheduled">Unscheduled</SelectItem>
            <SelectItem value="scheduled">Scheduled</SelectItem>
            <SelectItem value="in_progress">In Progress</SelectItem>
            <SelectItem value="completed">Completed</SelectItem>
            <SelectItem value="canceled">Canceled</SelectItem>
          </SelectContent>
        </Select>
      </div>

      {appointments.length === 0 ? (
        <Card>
          <CardContent className="py-8 text-center text-muted-foreground">
            No appointments found
          </CardContent>
        </Card>
      ) : (
        <div className="space-y-4">
          {appointments.map((appointment) => (
            <Card
              key={appointment.job_id}
              className="cursor-pointer hover:shadow-md transition-shadow"
              onClick={() => router.push(`/portal/appointments/${appointment.job_id}`)}
            >
              <CardHeader>
                <div className="flex items-start justify-between">
                  <div className="flex-1">
                    <CardTitle className="flex items-center gap-2 mb-2">
                      {appointment.title}
                      <Badge variant="outline">{appointment.priority}</Badge>
                      <div className={cn('w-2 h-2 rounded-full', getStatusColor(appointment.status))} />
                    </CardTitle>
                    <CardDescription>{appointment.description || 'No description'}</CardDescription>
                  </div>
                  <Badge variant="secondary">{appointment.status}</Badge>
                </div>
              </CardHeader>
              <CardContent>
                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                  {appointment.scheduled_start_at && (
                    <div className="flex items-center gap-2 text-sm">
                      <Calendar className="h-4 w-4 text-muted-foreground" />
                      <div>
                        <div className="font-medium">
                          {format(parseISO(appointment.scheduled_start_at), 'MMM d, yyyy')}
                        </div>
                        <div className="text-muted-foreground">
                          {format(parseISO(appointment.scheduled_start_at), 'h:mm a')} - {appointment.scheduled_end_at && format(parseISO(appointment.scheduled_end_at), 'h:mm a')}
                        </div>
                      </div>
                    </div>
                  )}

                  {appointment.technician_name && (
                    <div className="flex items-center gap-2 text-sm">
                      <User className="h-4 w-4 text-muted-foreground" />
                      <div>
                        <div className="font-medium">Technician</div>
                        <div className="text-muted-foreground">{appointment.technician_name}</div>
                      </div>
                    </div>
                  )}

                  <div className="flex items-center gap-2 text-sm">
                    <MapPin className="h-4 w-4 text-muted-foreground" />
                    <div>
                      <div className="font-medium">Location</div>
                      <div className="text-muted-foreground">
                        {appointment.location.address_line1}, {appointment.location.city}
                      </div>
                    </div>
                  </div>

                  {appointment.tech_eta_at && (
                    <div className="flex items-center gap-2 text-sm">
                      <Clock className="h-4 w-4 text-muted-foreground" />
                      <div>
                        <div className="font-medium">ETA</div>
                        <div className="text-muted-foreground">
                          {format(parseISO(appointment.tech_eta_at), 'h:mm a')}
                        </div>
                      </div>
                    </div>
                  )}
                </div>

                {appointment.arrival_window_start && appointment.arrival_window_end && (
                  <div className="mt-4 p-3 bg-muted rounded-md">
                    <div className="text-sm font-medium mb-1">Arrival Window</div>
                    <div className="text-sm text-muted-foreground">
                      {format(parseISO(appointment.arrival_window_start), 'h:mm a')} - {format(parseISO(appointment.arrival_window_end), 'h:mm a')}
                    </div>
                  </div>
                )}

                <Button
                  variant="ghost"
                  className="w-full mt-4"
                  onClick={(e) => {
                    e.stopPropagation();
                    router.push(`/portal/appointments/${appointment.job_id}`);
                  }}
                >
                  View Details
                  <ChevronRight className="ml-2 h-4 w-4" />
                </Button>
              </CardContent>
            </Card>
          ))}
        </div>
      )}
    </div>
  );
}
```

### 5.4 Next.js UI Component: Appointment Detail with Real-Time Updates

**File**: `app/portal/appointments/[jobId]/page.tsx`

```typescript
'use client';

import { useEffect } from 'react';
import { useQuery, useQueryClient } from '@tanstack/react-query';
import { useParams } from 'next/navigation';
import { createSupabaseClient } from '@/lib/supabase/client';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Skeleton } from '@/components/ui/skeleton';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { format, parseISO } from 'date-fns';
import { Calendar, Clock, MapPin, User, AlertCircle, CheckCircle2, Truck } from 'lucide-react';
import { cn } from '@/lib/utils';
import { useRealtimeAppointment } from '@/lib/hooks/useRealtime';

interface Appointment {
  job_id: string;
  title: string;
  description: string | null;
  status: string;
  priority: string;
  scheduled_start_at: string | null;
  scheduled_end_at: string | null;
  arrival_window_start: string | null;
  arrival_window_end: string | null;
  tech_eta_at: string | null;
  technician_name: string | null;
  location: {
    address_line1: string;
    city: string;
    state: string;
    postal_code: string;
  };
  time_windows: Array<{
    id: string;
    window_start: string;
    window_end: string;
    is_selected: boolean;
  }>;
  created_at: string;
  updated_at: string;
}

export default function AppointmentDetailPage() {
  const params = useParams();
  const jobId = params.jobId as string;
  const supabase = createSupabaseClient();
  const queryClient = useQueryClient();

  // Set up real-time subscription
  useRealtimeAppointment(jobId);

  const { data, isLoading, error } = useQuery({
    queryKey: ['appointment', jobId],
    queryFn: async () => {
      const { data: { session } } = await supabase.auth.getSession();
      
      const response = await fetch(`/api/portal/appointments/${jobId}`, {
        headers: {
          'Authorization': `Bearer ${session?.access_token}`
        }
      });

      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.error?.message || 'Failed to load appointment');
      }

      return response.json();
    }
  });

  const appointment: Appointment | null = data?.data?.appointment || null;

  const getStatusBadge = (status: string) => {
    const variants: Record<string, 'default' | 'secondary' | 'destructive' | 'outline'> = {
      scheduled: 'default',
      in_progress: 'secondary',
      completed: 'outline',
      canceled: 'destructive',
      unscheduled: 'secondary'
    };

    return (
      <Badge variant={variants[status] || 'secondary'}>
        {status.replace('_', ' ')}
      </Badge>
    );
  };

  if (isLoading) {
    return (
      <div className="container mx-auto py-8 max-w-4xl space-y-4">
        <Skeleton className="h-12 w-full" />
        <Skeleton className="h-64 w-full" />
      </div>
    );
  }

  if (error || !appointment) {
    return (
      <div className="container mx-auto py-8 max-w-4xl">
        <Alert variant="destructive">
          <AlertCircle className="h-4 w-4" />
          <AlertDescription>
            Failed to load appointment. Please try again.
          </AlertDescription>
        </Alert>
      </div>
    );
  }

  const selectedWindow = appointment.time_windows.find(w => w.is_selected);

  return (
    <div className="container mx-auto py-8 max-w-4xl space-y-6">
      <div>
        <h1 className="text-2xl font-bold mb-2">{appointment.title}</h1>
        <div className="flex items-center gap-2">
          {getStatusBadge(appointment.status)}
          <Badge variant="outline">{appointment.priority}</Badge>
        </div>
      </div>

      {appointment.description && (
        <Card>
          <CardHeader>
            <CardTitle>Description</CardTitle>
          </CardHeader>
          <CardContent>
            <p className="text-sm">{appointment.description}</p>
          </CardContent>
        </Card>
      )}

      <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
        {appointment.scheduled_start_at && (
          <Card>
            <CardHeader>
              <CardTitle className="text-base flex items-center gap-2">
                <Calendar className="h-4 w-4" />
                Scheduled Time
              </CardTitle>
            </CardHeader>
            <CardContent>
              <div className="text-sm space-y-1">
                <div className="font-medium">
                  {format(parseISO(appointment.scheduled_start_at), 'EEEE, MMMM d, yyyy')}
                </div>
                <div className="text-muted-foreground">
                  {format(parseISO(appointment.scheduled_start_at), 'h:mm a')} - {appointment.scheduled_end_at && format(parseISO(appointment.scheduled_end_at), 'h:mm a')}
                </div>
              </div>
            </CardContent>
          </Card>
        )}

        {appointment.technician_name && (
          <Card>
            <CardHeader>
              <CardTitle className="text-base flex items-center gap-2">
                <User className="h-4 w-4" />
                Technician
              </CardTitle>
            </CardHeader>
            <CardContent>
              <div className="text-sm font-medium">{appointment.technician_name}</div>
            </CardContent>
          </Card>
        )}

        <Card>
          <CardHeader>
            <CardTitle className="text-base flex items-center gap-2">
              <MapPin className="h-4 w-4" />
              Location
            </CardTitle>
          </CardHeader>
          <CardContent>
            <div className="text-sm">
              <div>{appointment.location.address_line1}</div>
              <div className="text-muted-foreground">
                {appointment.location.city}, {appointment.location.state} {appointment.location.postal_code}
              </div>
            </div>
          </CardContent>
        </Card>

        {appointment.tech_eta_at && (
          <Card className="border-primary">
            <CardHeader>
              <CardTitle className="text-base flex items-center gap-2">
                <Truck className="h-4 w-4 text-primary" />
                Technician ETA
              </CardTitle>
            </CardHeader>
            <CardContent>
              <div className="text-sm">
                <div className="font-medium text-primary">
                  {format(parseISO(appointment.tech_eta_at), 'h:mm a')}
                </div>
                <div className="text-muted-foreground text-xs mt-1">
                  Updated in real-time
                </div>
              </div>
            </CardContent>
          </Card>
        )}
      </div>

      {appointment.arrival_window_start && appointment.arrival_window_end && (
        <Card>
          <CardHeader>
            <CardTitle className="text-base flex items-center gap-2">
              <Clock className="h-4 w-4" />
              Arrival Window
            </CardTitle>
          </CardHeader>
          <CardContent>
            <div className="text-sm">
              <div className="font-medium">
                {format(parseISO(appointment.arrival_window_start), 'h:mm a')} - {format(parseISO(appointment.arrival_window_end), 'h:mm a')}
              </div>
              <div className="text-muted-foreground text-xs mt-1">
                Expected arrival time range
              </div>
            </div>
          </CardContent>
        </Card>
      )}

      {selectedWindow && (
        <Card>
          <CardHeader>
            <CardTitle className="text-base flex items-center gap-2">
              <CheckCircle2 className="h-4 w-4 text-green-500" />
              Selected Time Window
            </CardTitle>
          </CardHeader>
          <CardContent>
            <div className="text-sm">
              <div className="font-medium">
                {format(parseISO(selectedWindow.window_start), 'EEEE, MMMM d')}
              </div>
              <div className="text-muted-foreground">
                {format(parseISO(selectedWindow.window_start), 'h:mm a')} - {format(parseISO(selectedWindow.window_end), 'h:mm a')}
              </div>
            </div>
          </CardContent>
        </Card>
      )}

      {appointment.status === 'unscheduled' && (
        <Alert>
          <AlertCircle className="h-4 w-4" />
          <AlertDescription>
            This appointment has not been scheduled yet. Please select a time window to proceed.
          </AlertDescription>
        </Alert>
      )}
    </div>
  );
}
```

### 5.5 Real-Time Subscription Hook

**File**: `lib/hooks/useRealtime.ts` (add to existing file)

```typescript
import { useEffect } from 'react';
import { createSupabaseClient } from '@/lib/supabase/client';
import { useQueryClient } from '@tanstack/react-query';

export function useRealtimeAppointment(jobId: string) {
  const supabase = createSupabaseClient();
  const queryClient = useQueryClient();

  useEffect(() => {
    // Subscribe to job updates
    const jobChannel = supabase
      .channel(`appointment-job-${jobId}`)
      .on(
        'postgres_changes',
        {
          event: 'UPDATE',
          schema: 'public',
          table: 'dispatch_jobs',
          filter: `id=eq.${jobId}`
        },
        () => {
          queryClient.invalidateQueries({ queryKey: ['appointment', jobId] });
        }
      )
      .subscribe();

    // Subscribe to assignment updates (for ETA and status)
    const assignmentChannel = supabase
      .channel(`appointment-assignments-${jobId}`)
      .on(
        'postgres_changes',
        {
          event: '*',
          schema: 'public',
          table: 'job_assignments',
          filter: `dispatch_job_id=eq.${jobId}`
        },
        () => {
          queryClient.invalidateQueries({ queryKey: ['appointment', jobId] });
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(jobChannel);
      supabase.removeChannel(assignmentChannel);
    };
  }, [jobId, supabase, queryClient]);
}
```

### 5.6 API Routes

**File**: `app/api/portal/appointments/route.ts`

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
  const status = searchParams.get('status');
  const limit = searchParams.get('limit');
  const offset = searchParams.get('offset');

  const { data, error } = await supabase.functions.invoke('portal-list-appointments', {
    body: {
      status: status || undefined,
      limit: limit ? parseInt(limit) : undefined,
      offset: offset ? parseInt(offset) : undefined,
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

**File**: `app/api/portal/appointments/[jobId]/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';

export async function GET(
  request: NextRequest,
  { params }: { params: { jobId: string } }
) {
  const supabase = createRouteHandlerClient({ cookies });
  
  const {
    data: { session },
  } = await supabase.auth.getSession();

  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { data, error } = await supabase.functions.invoke('portal-get-appointment', {
    body: {
      job_id: params.jobId,
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

## 6. Security Considerations

### 6.1 Customer Authorization

**Critical**: All portal endpoints must:

1. **Verify Authentication**: Check for valid Supabase Auth session
2. **Verify Customer Role**: Ensure user has `role = 'customer'` in profiles
3. **Verify Job Ownership**: Ensure job belongs to customer via `portal_user_contacts` mapping
4. **Enforce Org Isolation**: Ensure customer can only access jobs in their org

### 6.2 Data Filtering

**Portal responses must NOT include**:
- Internal notes (`notes_internal`)
- Dispatcher information
- Other customers' data
- System metadata (e.g., `created_by_user_id`)
- Sensitive technician details (beyond display name)

### 6.3 Rate Limiting

Consider implementing rate limiting on portal endpoints to prevent abuse:

```typescript
// Example: Limit to 100 requests per minute per customer
const rateLimitKey = `portal:${customerAuth.userId}:${endpoint}`;
// Use Redis or similar for rate limiting
```

---

## 7. Error Handling

### 7.1 Standard Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid authentication token |
| `FORBIDDEN` | 403 | Customer does not own the job or access denied |
| `NOT_FOUND` | 404 | Job or time window not found |
| `WINDOW_EXPIRED` | 400 | Time window has passed |
| `ALREADY_ASSIGNED` | 400 | Job is already assigned |
| `MISSING_JOB_ID` | 400 | Job ID required |
| `MISSING_WINDOW_ID` | 400 | Window ID required |

---

## 8. Testing Requirements

### 8.1 Security Tests

1. **Customer Isolation**:
   - Customer A cannot access Customer B's jobs
   - Customer cannot access jobs from different orgs
   - Unauthenticated users cannot access portal endpoints

2. **Authorization Tests**:
   - Non-customer roles cannot access portal endpoints
   - Customers can only select windows for their own jobs
   - Customers cannot modify assignment times or technicians

3. **Data Filtering Tests**:
   - Portal responses do not include sensitive fields
   - Internal notes are excluded
   - Dispatcher information is excluded

### 8.2 Functional Tests

1. **Time Window Selection**:
   - Customer can view available windows
   - Customer can select a window
   - Selection updates `is_selected` flag
   - Auto-scheduling works when enabled

2. **Appointment Status**:
   - Customer can view appointment status
   - ETA updates appear in real-time
   - Status changes are reflected immediately

---

## 9. Implementation Checklist

### Story DISP-056: Expose Appointment Time Windows

- [ ] **Edge Function: portal-list-time-windows**:
  - [ ] Function implemented
  - [ ] Customer authorization implemented
  - [ ] Job ownership verification implemented
  - [ ] Availability calculation implemented
  - [ ] Error handling implemented
  - [ ] API documentation

- [ ] **Edge Function: portal-select-time-window**:
  - [ ] Function implemented
  - [ ] Customer authorization implemented
  - [ ] Job ownership verification implemented
  - [ ] Window validation implemented
  - [ ] Selection update logic implemented
  - [ ] Auto-schedule integration implemented
  - [ ] Error handling implemented
  - [ ] API documentation

- [ ] **Next.js UI Component**:
  - [ ] Time window selection page created
  - [ ] shadcn/ui components used
  - [ ] Window cards with availability status
  - [ ] Selection handler implemented
  - [ ] Auto-schedule checkbox implemented
  - [ ] Loading states implemented
  - [ ] Error states implemented
  - [ ] Success feedback implemented

- [ ] **API Routes**:
  - [ ] GET /api/portal/jobs/[jobId]/time-windows route implemented
  - [ ] POST /api/portal/jobs/[jobId]/time-windows/[windowId]/select route implemented
  - [ ] Error handling implemented

### Story DISP-057: Customer Appointment Status + ETA

- [ ] **Edge Function: portal-list-appointments**:
  - [ ] Function implemented
  - [ ] Customer authorization implemented
  - [ ] Job filtering by customer implemented
  - [ ] Status filtering implemented
  - [ ] Pagination implemented
  - [ ] Response transformation implemented
  - [ ] Error handling implemented
  - [ ] API documentation

- [ ] **Edge Function: portal-get-appointment**:
  - [ ] Function implemented
  - [ ] Customer authorization implemented
  - [ ] Job ownership verification implemented
  - [ ] Detailed appointment data returned
  - [ ] Error handling implemented
  - [ ] API documentation

- [ ] **Next.js UI Components**:
  - [ ] Appointment list page created
  - [ ] Appointment detail page created
  - [ ] shadcn/ui components used
  - [ ] Status badges implemented
  - [ ] ETA display implemented
  - [ ] Real-time updates implemented
  - [ ] Loading states implemented
  - [ ] Error states implemented

- [ ] **Real-Time Subscriptions**:
  - [ ] useRealtimeAppointment hook implemented
  - [ ] Job updates subscription
  - [ ] Assignment updates subscription
  - [ ] Query invalidation on updates

- [ ] **API Routes**:
  - [ ] GET /api/portal/appointments route implemented
  - [ ] GET /api/portal/appointments/[jobId] route implemented
  - [ ] Error handling implemented

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 12 – Customer Portal and Booking Hooks. All APIs are designed as portal-safe Edge Functions with strict customer authorization, and UI components are built using Next.js with shadcn/ui for a consistent customer experience.

**Next Steps**: After completing Epic 12, proceed to Epic 13 (Analytics & KPI Foundations) which will define scheduling KPI specifications for downstream reporting.

