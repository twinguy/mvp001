# Technical Design Document – Epic 10: Notifications & Reminder Orchestration

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 10 – Notifications & Reminder Orchestration
- **Source**: Derived from `fdd_2_agile.md` Epic 10 (Stories DISP-048 through DISP-050)
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
  - `tdd_2_epic_9.md` (Dispatch Epic 9 for calendar integration)
- **Target Platform**: Supabase (PostgreSQL 15+, Edge Functions, Cron Jobs)
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing Notifications & Reminder Orchestration. It covers:

- Scheduling standard notifications for jobs with configurable reminder rules
- Cron-based processor for sending pending notifications
- Template variable contracts for each notification type
- Integration with SMS, email, and push notification providers
- Idempotency and retry logic
- PII handling rules per channel

All APIs are implemented as Supabase Edge Functions (Deno/TypeScript) that schedule notifications, process them via cron jobs, and integrate with messaging providers to deliver reminders and updates to customers and technicians.

This epic assumes Epic 1 (tenancy/roles), Epic 2 (tables), Epic 3 (RLS policies), Epic 4 (technician APIs), Epic 5 (job lifecycle APIs), Epic 6 (technician mobile hooks), Epic 7 (auto-scheduling and route optimization), Epic 8 (emergency job handling), and Epic 9 (calendar integration) are complete.

---

## 2. Prerequisites

### 2.1 Required Components

Before implementing Epic 10, ensure:

1. **Epic 1-9 Complete**: All previous epics are implemented
2. **Required Tables**: All dispatch tables exist:
   - `job_notifications`
   - `job_assignments`
   - `dispatch_jobs`
   - `technician_profiles`
   - `customer_contacts` (from CRM)
   - `customers` (from CRM)
   - `customer_locations` (from CRM)

3. **Messaging Providers** (configured):
   - SMS provider (Twilio, AWS SNS, etc.)
   - Email provider (SendGrid, AWS SES, etc.)
   - Push notification provider (Firebase Cloud Messaging, OneSignal, etc.)

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

---

## 4. Story DISP-048: Schedule Standard Notifications for a Job

### 4.1 POST /dispatch/jobs/:id/schedule_notifications

**Purpose**: Schedule standard reminder notifications for a job assignment.

**Authorization**: `admin`, `dispatcher` (or automatic on assignment creation)

**Request Schema**:

```typescript
interface ScheduleNotificationsRequest {
  assignment_id?: string; // UUID, optional: schedule for specific assignment (default: all active assignments)
  override_rules?: Array<{
    notification_type: 'pre_appointment_reminder' | 'same_day_reminder' | 'tech_on_the_way';
    offset_minutes: number; // Minutes before scheduled_start_at
    channels?: ('sms' | 'email' | 'push')[]; // Optional: override default channels
  }>;
  skip_existing?: boolean; // default: false, skip if notifications already exist
}
```

**Request Example**:

```json
{
  "assignment_id": "999e4567-e89b-12d3-a456-426614174000",
  "override_rules": [
    {
      "notification_type": "pre_appointment_reminder",
      "offset_minutes": 1440,
      "channels": ["sms", "email"]
    }
  ]
}
```

**Response Schema**:

```typescript
interface ScheduleNotificationsResponse {
  job_id: string;
  notifications_created: number;
  notifications: Array<{
    id: string;
    notification_type: string;
    channel: string;
    recipient_type: string;
    scheduled_send_at: string;
  }>;
}
```

**Response Example**:

```json
{
  "data": {
    "job_id": "444e4567-e89b-12d3-a456-426614174000",
    "notifications_created": 6,
    "notifications": [
      {
        "id": "aaa11111-1111-1111-1111-111111111111",
        "notification_type": "pre_appointment_reminder",
        "channel": "sms",
        "recipient_type": "customer",
        "scheduled_send_at": "2024-01-19T10:00:00Z"
      },
      {
        "id": "aaa22222-2222-2222-2222-222222222222",
        "notification_type": "pre_appointment_reminder",
        "channel": "email",
        "recipient_type": "customer",
        "scheduled_send_at": "2024-01-19T10:00:00Z"
      }
    ]
  }
}
```

### 4.2 Notification Reminder Rules

**Default Rules** (MVP):

| Notification Type | Offset | Recipients | Channels | Description |
|-------------------|--------|------------|----------|-------------|
| `pre_appointment_reminder` | 24 hours (1440 min) | Customer, Technician | SMS, Email | Reminder 24h before appointment |
| `same_day_reminder` | 2 hours (120 min) | Customer, Technician | SMS, Email | Reminder 2h before appointment |
| `tech_on_the_way` | On ETA update | Customer | SMS, Push | Sent when technician updates ETA |
| `booking_confirmation` | Immediately | Customer | SMS, Email | Sent when job is booked |
| `reschedule_notice` | Immediately | Customer | SMS, Email | Sent when appointment is rescheduled |
| `cancellation_notice` | Immediately | Customer | SMS, Email | Sent when appointment is canceled |

**Org-Level Configuration**:

```sql
CREATE TABLE IF NOT EXISTS org_notification_rules (
  org_id UUID PRIMARY KEY REFERENCES orgs(id) ON DELETE CASCADE,
  rules JSONB NOT NULL DEFAULT '{
    "pre_appointment_reminder": {
      "offset_minutes": 1440,
      "recipients": ["customer", "technician"],
      "channels": ["sms", "email"],
      "enabled": true
    },
    "same_day_reminder": {
      "offset_minutes": 120,
      "recipients": ["customer"],
      "channels": ["sms", "email"],
      "enabled": true
    },
    "tech_on_the_way": {
      "offset_minutes": 0,
      "recipients": ["customer"],
      "channels": ["sms", "push"],
      "enabled": true,
      "trigger": "eta_update"
    },
    "booking_confirmation": {
      "offset_minutes": 0,
      "recipients": ["customer"],
      "channels": ["sms", "email"],
      "enabled": true,
      "trigger": "job_created"
    },
    "reschedule_notice": {
      "offset_minutes": 0,
      "recipients": ["customer"],
      "channels": ["sms", "email"],
      "enabled": true,
      "trigger": "assignment_rescheduled"
    },
    "cancellation_notice": {
      "offset_minutes": 0,
      "recipients": ["customer"],
      "channels": ["sms", "email"],
      "enabled": true,
      "trigger": "assignment_canceled"
    }
  }'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Get Notification Rules**:

```typescript
async function getNotificationRules(
  supabase: SupabaseClient,
  orgId: string
): Promise<any> {
  const { data: orgRules } = await supabase
    .from('org_notification_rules')
    .select('rules')
    .eq('org_id', orgId)
    .single();

  if (orgRules) {
    return orgRules.rules;
  }

  // Return default rules
  return {
    pre_appointment_reminder: {
      offset_minutes: 1440,
      recipients: ['customer', 'technician'],
      channels: ['sms', 'email'],
      enabled: true
    },
    same_day_reminder: {
      offset_minutes: 120,
      recipients: ['customer'],
      channels: ['sms', 'email'],
      enabled: true
    },
    tech_on_the_way: {
      offset_minutes: 0,
      recipients: ['customer'],
      channels: ['sms', 'push'],
      enabled: true,
      trigger: 'eta_update'
    },
    booking_confirmation: {
      offset_minutes: 0,
      recipients: ['customer'],
      channels: ['sms', 'email'],
      enabled: true,
      trigger: 'job_created'
    },
    reschedule_notice: {
      offset_minutes: 0,
      recipients: ['customer'],
      channels: ['sms', 'email'],
      enabled: true,
      trigger: 'assignment_rescheduled'
    },
    cancellation_notice: {
      offset_minutes: 0,
      recipients: ['customer'],
      channels: ['sms', 'email'],
      enabled: true,
      trigger: 'assignment_canceled'
    }
  };
}
```

### 4.3 Implementation

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

  const auth = await authorizeUser(supabase, user.id, ['admin', 'dispatcher']);
  if (!auth) {
    return errorResponse('Forbidden', 403, 'INSUFFICIENT_PERMISSIONS');
  }

  // Extract job ID from URL
  const url = new URL(req.url);
  const jobId = url.pathname.split('/').pop();

  if (!jobId) {
    return errorResponse('Job ID required', 400, 'MISSING_JOB_ID');
  }

  const body = await req.json();
  const assignmentId = body.assignment_id;
  const overrideRules = body.override_rules || [];
  const skipExisting = body.skip_existing || false;

  // Get job and assignments
  let assignmentsQuery = supabase
    .from('job_assignments')
    .select(`
      *,
      dispatch_jobs!inner(
        id,
        title,
        description,
        customer_id,
        location_id,
        customer_locations!inner(
          address_line1,
          city,
          state,
          postal_code
        ),
        customers!inner(
          id,
          name
        )
      ),
      technician_profiles!inner(
        id,
        display_name,
        user_id
      )
    `)
    .eq('org_id', auth.orgId)
    .eq('dispatch_jobs.id', jobId)
    .in('status', ['assigned', 'accepted', 'en_route', 'on_site']);

  if (assignmentId) {
    assignmentsQuery = assignmentsQuery.eq('id', assignmentId);
  }

  const { data: assignments, error: assignmentsError } = await assignmentsQuery;

  if (assignmentsError || !assignments || assignments.length === 0) {
    return errorResponse('No active assignments found', 404, 'NO_ASSIGNMENTS');
  }

  // Get notification rules
  const rules = await getNotificationRules(supabase, auth.orgId);

  const notificationsToCreate: Array<{
    org_id: string;
    dispatch_job_id: string;
    job_assignment_id: string;
    recipient_type: string;
    recipient_contact_id: string | null;
    channel: string;
    notification_type: string;
    scheduled_send_at: string;
    status: string;
    metadata: any;
  }> = [];

  for (const assignment of assignments) {
    const job = assignment.dispatch_jobs;
    const scheduledStart = new Date(assignment.scheduled_start_at);

    // Process each notification type
    for (const [notificationType, rule] of Object.entries(rules)) {
      if (!rule.enabled) {
        continue;
      }

      // Check for override
      const override = overrideRules.find((o: any) => o.notification_type === notificationType);
      const offsetMinutes = override?.offset_minutes ?? rule.offset_minutes;
      const channels = override?.channels ?? rule.channels;
      const recipients = rule.recipients;

      // Calculate scheduled send time
      const scheduledSendAt = new Date(scheduledStart.getTime() - offsetMinutes * 60000);

      // Skip if in the past (unless immediate notification)
      if (offsetMinutes > 0 && scheduledSendAt < new Date()) {
        continue;
      }

      // Check if already exists
      if (skipExisting) {
        const { data: existing } = await supabase
          .from('job_notifications')
          .select('id')
          .eq('job_assignment_id', assignment.id)
          .eq('notification_type', notificationType)
          .eq('status', 'pending')
          .single();

        if (existing) {
          continue;
        }
      }

      // Create notifications for each recipient and channel
      for (const recipient of recipients) {
        for (const channel of channels) {
          // Get recipient contact
          const contactId = await getRecipientContactId(
            supabase,
            auth.orgId,
            recipient,
            assignment,
            channel
          );

          if (!contactId) {
            continue; // Skip if no contact info
          }

          // Build metadata (template variables)
          const metadata = await buildNotificationMetadata(
            supabase,
            assignment,
            job,
            notificationType
          );

          notificationsToCreate.push({
            org_id: auth.orgId,
            dispatch_job_id: job.id,
            job_assignment_id: assignment.id,
            recipient_type: recipient,
            recipient_contact_id: contactId,
            channel: channel,
            notification_type: notificationType,
            scheduled_send_at: scheduledSendAt.toISOString(),
            status: 'pending',
            metadata: metadata
          });
        }
      }
    }
  }

  if (notificationsToCreate.length === 0) {
    return successResponse({
      job_id: jobId,
      notifications_created: 0,
      notifications: []
    });
  }

  // Insert notifications
  const { data: createdNotifications, error: createError } = await supabase
    .from('job_notifications')
    .insert(notificationsToCreate)
    .select();

  if (createError) {
    return errorResponse('Failed to create notifications', 500, 'CREATE_ERROR', { error: createError.message });
  }

  return successResponse({
    job_id: jobId,
    notifications_created: createdNotifications?.length || 0,
    notifications: createdNotifications?.map(n => ({
      id: n.id,
      notification_type: n.notification_type,
      channel: n.channel,
      recipient_type: n.recipient_type,
      scheduled_send_at: n.scheduled_send_at
    })) || []
  });
});

async function getRecipientContactId(
  supabase: SupabaseClient,
  orgId: string,
  recipientType: string,
  assignment: any,
  channel: string
): Promise<string | null> {
  if (recipientType === 'customer') {
    const job = assignment.dispatch_jobs;
    
    // Get customer contact matching channel
    const contactType = channel === 'sms' ? 'mobile' : channel === 'email' ? 'email' : null;
    
    if (!contactType) {
      return null;
    }

    const { data: contact } = await supabase
      .from('customer_contacts')
      .select('id')
      .eq('customer_id', job.customer_id)
      .eq('org_id', orgId)
      .eq('type', contactType)
      .eq('is_primary', true)
      .single();

    return contact?.id || null;
  } else if (recipientType === 'technician') {
    // Get technician contact (via user profile or technician_profiles)
    const techProfile = assignment.technician_profiles;
    
    // Simplified: would resolve via user profile or technician contact info
    // For MVP, return null if not available
    return null;
  } else if (recipientType === 'dispatcher') {
    // Get dispatcher contact (via user profile)
    // For MVP, return null
    return null;
  }

  return null;
}

async function buildNotificationMetadata(
  supabase: SupabaseClient,
  assignment: any,
  job: any,
  notificationType: string
): Promise<any> {
  const location = job.customer_locations;
  const customer = job.customers;
  const technician = assignment.technician_profiles;

  const address = [
    location.address_line1,
    location.city,
    location.state,
    location.postal_code
  ].filter(Boolean).join(', ');

  const scheduledStart = new Date(assignment.scheduled_start_at);
  const scheduledEnd = new Date(assignment.scheduled_end_at);

  return {
    // Common fields
    job_id: job.id,
    job_title: job.title || 'Service Appointment',
    job_description: job.description || '',
    customer_name: customer.name || 'Customer',
    customer_id: customer.id,
    location_address: address,
    scheduled_start_at: assignment.scheduled_start_at,
    scheduled_end_at: assignment.scheduled_end_at,
    scheduled_start_formatted: formatDateTime(scheduledStart),
    scheduled_end_formatted: formatDateTime(scheduledEnd),
    technician_name: technician.display_name || 'Technician',
    technician_id: technician.id,
    
    // Type-specific fields
    ...(notificationType === 'tech_on_the_way' && assignment.tech_eta_at ? {
      tech_eta_at: assignment.tech_eta_at,
      tech_eta_formatted: formatDateTime(new Date(assignment.tech_eta_at))
    } : {}),
    
    ...(notificationType === 'reschedule_notice' ? {
      old_scheduled_start_at: assignment.metadata?.old_scheduled_start_at,
      old_scheduled_start_formatted: assignment.metadata?.old_scheduled_start_at 
        ? formatDateTime(new Date(assignment.metadata.old_scheduled_start_at))
        : null,
      delay_minutes: assignment.metadata?.delay_minutes || 0
    } : {}),
    
    ...(notificationType === 'cancellation_notice' ? {
      cancellation_reason: assignment.metadata?.cancellation_reason || 'Appointment canceled'
    } : {})
  };
}

function formatDateTime(date: Date): string {
  return date.toLocaleString('en-US', {
    weekday: 'long',
    year: 'numeric',
    month: 'long',
    day: 'numeric',
    hour: 'numeric',
    minute: '2-digit',
    hour12: true
  });
}
```

---

## 5. Story DISP-049: Notification Processor (Cron)

### 5.1 Edge Function: process_job_notifications

**Purpose**: Cron-based processor to send pending notifications.

**Schedule**: Run every minute (configured in Supabase Dashboard or via pg_cron)

**Implementation**:

```typescript
Deno.serve(async (req) => {
  // This function is called by Supabase cron
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')! // Need service role for cron
  );

  const now = new Date();
  const nowISO = now.toISOString();

  // Fetch pending notifications due now (with small buffer for processing time)
  const bufferMinutes = 1;
  const cutoffTime = new Date(now.getTime() + bufferMinutes * 60000);

  const { data: pendingNotifications, error: fetchError } = await supabase
    .from('job_notifications')
    .select(`
      *,
      dispatch_jobs(
        id,
        title,
        customer_id,
        customers(id, name),
        customer_locations(address_line1, city, state, postal_code)
      ),
      job_assignments(
        id,
        scheduled_start_at,
        scheduled_end_at,
        tech_eta_at,
        technician_profiles(display_name)
      ),
      customer_contacts(value, type)
    `)
    .eq('status', 'pending')
    .lte('scheduled_send_at', cutoffTime.toISOString())
    .order('scheduled_send_at', { ascending: true })
    .limit(100); // Process in batches

  if (fetchError) {
    console.error('Failed to fetch pending notifications:', fetchError);
    return new Response('Error', { status: 500 });
  }

  if (!pendingNotifications || pendingNotifications.length === 0) {
    return new Response('No notifications to process', { status: 200 });
  }

  let sentCount = 0;
  let failedCount = 0;
  const errors: Array<{ notification_id: string; error: string }> = [];

  for (const notification of pendingNotifications) {
    try {
      // Check idempotency: verify notification is still pending
      const { data: currentNotification } = await supabase
        .from('job_notifications')
        .select('status')
        .eq('id', notification.id)
        .single();

      if (!currentNotification || currentNotification.status !== 'pending') {
        continue; // Already processed or canceled
      }

      // Mark as processing (optional: add 'processing' status)
      await supabase
        .from('job_notifications')
        .update({ updated_at: nowISO })
        .eq('id', notification.id);

      // Resolve template content
      const content = await resolveNotificationContent(notification);

      // Send via provider
      const result = await sendNotification(
        notification.channel,
        notification.recipient_contact_id,
        notification.customer_contacts?.value,
        content,
        notification.metadata
      );

      if (result.success) {
        // Update as sent
        await supabase
          .from('job_notifications')
          .update({
            status: 'sent',
            sent_at: nowISO,
            metadata: {
              ...notification.metadata,
              provider_message_id: result.messageId,
              sent_at: nowISO
            }
          })
          .eq('id', notification.id);

        sentCount++;
      } else {
        // Update as failed
        await supabase
          .from('job_notifications')
          .update({
            status: 'failed',
            error_message: result.error,
            metadata: {
              ...notification.metadata,
              error: result.error,
              failed_at: nowISO,
              retry_count: (notification.metadata?.retry_count || 0) + 1
            }
          })
          .eq('id', notification.id);

        failedCount++;
        errors.push({
          notification_id: notification.id,
          error: result.error || 'Unknown error'
        });
      }
    } catch (error) {
      console.error(`Failed to process notification ${notification.id}:`, error);
      
      await supabase
        .from('job_notifications')
        .update({
          status: 'failed',
          error_message: error.message,
          metadata: {
            ...notification.metadata,
            error: error.message,
            failed_at: nowISO,
            retry_count: (notification.metadata?.retry_count || 0) + 1
          }
        })
        .eq('id', notification.id);

      failedCount++;
      errors.push({
        notification_id: notification.id,
        error: error.message
      });
    }
  }

  return new Response(
    JSON.stringify({
      processed: pendingNotifications.length,
      sent: sentCount,
      failed: failedCount,
      errors: errors
    }),
    {
      status: 200,
      headers: { 'Content-Type': 'application/json' }
    }
  );
});
```

### 5.2 Template Resolution

```typescript
async function resolveNotificationContent(
  notification: any
): Promise<{ subject?: string; body: string }> {
  const metadata = notification.metadata || {};
  const notificationType = notification.notification_type;
  const channel = notification.channel;

  // Get template (would be stored in database or config)
  const template = await getTemplate(notificationType, channel);

  if (!template) {
    throw new Error(`Template not found for ${notificationType}/${channel}`);
  }

  // Resolve template variables
  const resolved = resolveTemplate(template, metadata);

  return resolved;
}

async function getTemplate(
  notificationType: string,
  channel: string
): Promise<any> {
  // Simplified: would fetch from database or config
  // For MVP, use inline templates
  
  const templates: Record<string, Record<string, any>> = {
    pre_appointment_reminder: {
      sms: {
        body: 'Hi {{customer_name}}, reminder: Your appointment "{{job_title}}" is scheduled for {{scheduled_start_formatted}} at {{location_address}}. See you then!'
      },
      email: {
        subject: 'Appointment Reminder: {{job_title}}',
        body: 'Hi {{customer_name}},\n\nThis is a reminder that your appointment "{{job_title}}" is scheduled for:\n\nDate & Time: {{scheduled_start_formatted}}\nLocation: {{location_address}}\nTechnician: {{technician_name}}\n\nWe look forward to seeing you!'
      }
    },
    same_day_reminder: {
      sms: {
        body: 'Hi {{customer_name}}, your appointment "{{job_title}}" is in 2 hours at {{scheduled_start_formatted}}. Location: {{location_address}}'
      },
      email: {
        subject: 'Reminder: Your Appointment is Today',
        body: 'Hi {{customer_name}},\n\nYour appointment "{{job_title}}" is scheduled for today at {{scheduled_start_formatted}}.\n\nLocation: {{location_address}}\nTechnician: {{technician_name}}\n\nSee you soon!'
      }
    },
    tech_on_the_way: {
      sms: {
        body: 'Hi {{customer_name}}, {{technician_name}} is on the way! ETA: {{tech_eta_formatted}}'
      },
      push: {
        title: 'Technician On The Way',
        body: '{{technician_name}} is on the way. ETA: {{tech_eta_formatted}}'
      }
    },
    booking_confirmation: {
      sms: {
        body: 'Hi {{customer_name}}, your appointment "{{job_title}}" is confirmed for {{scheduled_start_formatted}} at {{location_address}}'
      },
      email: {
        subject: 'Appointment Confirmed: {{job_title}}',
        body: 'Hi {{customer_name}},\n\nYour appointment has been confirmed:\n\nService: {{job_title}}\nDate & Time: {{scheduled_start_formatted}}\nLocation: {{location_address}}\nTechnician: {{technician_name}}\n\nWe look forward to serving you!'
      }
    },
    reschedule_notice: {
      sms: {
        body: 'Hi {{customer_name}}, your appointment "{{job_title}}" has been rescheduled to {{scheduled_start_formatted}}. We apologize for any inconvenience.'
      },
      email: {
        subject: 'Appointment Rescheduled: {{job_title}}',
        body: 'Hi {{customer_name}},\n\nYour appointment "{{job_title}}" has been rescheduled:\n\nPrevious Time: {{old_scheduled_start_formatted}}\nNew Time: {{scheduled_start_formatted}}\n\nWe apologize for any inconvenience this may cause.'
      }
    },
    cancellation_notice: {
      sms: {
        body: 'Hi {{customer_name}}, your appointment "{{job_title}}" scheduled for {{scheduled_start_formatted}} has been canceled. Reason: {{cancellation_reason}}'
      },
      email: {
        subject: 'Appointment Canceled: {{job_title}}',
        body: 'Hi {{customer_name}},\n\nYour appointment "{{job_title}}" scheduled for {{scheduled_start_formatted}} has been canceled.\n\nReason: {{cancellation_reason}}\n\nPlease contact us if you need to reschedule.'
      }
    }
  };

  return templates[notificationType]?.[channel] || null;
}

function resolveTemplate(
  template: { subject?: string; body: string },
  metadata: any
): { subject?: string; body: string } {
  const resolve = (text: string): string => {
    return text.replace(/\{\{(\w+)\}\}/g, (match, key) => {
      return metadata[key] || match;
    });
  };

  return {
    subject: template.subject ? resolve(template.subject) : undefined,
    body: resolve(template.body)
  };
}
```

### 5.3 Provider Integration

```typescript
interface SendResult {
  success: boolean;
  messageId?: string;
  error?: string;
}

async function sendNotification(
  channel: string,
  contactId: string | null,
  contactValue: string | undefined,
  content: { subject?: string; body: string },
  metadata: any
): Promise<SendResult> {
  if (!contactValue) {
    return {
      success: false,
      error: 'Contact value not provided'
    };
  }

  try {
    if (channel === 'sms') {
      return await sendSMS(contactValue, content.body, metadata);
    } else if (channel === 'email') {
      return await sendEmail(contactValue, content.subject || '', content.body, metadata);
    } else if (channel === 'push') {
      return await sendPushNotification(contactId, content.body, metadata);
    } else {
      return {
        success: false,
        error: `Unsupported channel: ${channel}`
      };
    }
  } catch (error) {
    return {
      success: false,
      error: error.message
    };
  }
}

// SMS Provider (Twilio example)
async function sendSMS(
  phoneNumber: string,
  body: string,
  metadata: any
): Promise<SendResult> {
  const accountSid = Deno.env.get('TWILIO_ACCOUNT_SID');
  const authToken = Deno.env.get('TWILIO_AUTH_TOKEN');
  const fromNumber = Deno.env.get('TWILIO_PHONE_NUMBER');

  if (!accountSid || !authToken || !fromNumber) {
    return {
      success: false,
      error: 'Twilio not configured'
    };
  }

  const response = await fetch(
    `https://api.twilio.com/2010-04-01/Accounts/${accountSid}/Messages.json`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Basic ${btoa(`${accountSid}:${authToken}`)}`,
        'Content-Type': 'application/x-www-form-urlencoded'
      },
      body: new URLSearchParams({
        From: fromNumber,
        To: phoneNumber,
        Body: body
      })
    }
  );

  if (!response.ok) {
    const error = await response.text();
    return {
      success: false,
      error: `Twilio error: ${error}`
    };
  }

  const data = await response.json();
  return {
    success: true,
    messageId: data.sid
  };
}

// Email Provider (SendGrid example)
async function sendEmail(
  emailAddress: string,
  subject: string,
  body: string,
  metadata: any
): Promise<SendResult> {
  const apiKey = Deno.env.get('SENDGRID_API_KEY');
  const fromEmail = Deno.env.get('SENDGRID_FROM_EMAIL') || 'noreply@yourdomain.com';

  if (!apiKey) {
    return {
      success: false,
      error: 'SendGrid not configured'
    };
  }

  const response = await fetch('https://api.sendgrid.com/v3/mail/send', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${apiKey}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      personalizations: [{
        to: [{ email: emailAddress }],
        subject: subject
      }],
      from: { email: fromEmail },
      content: [{
        type: 'text/plain',
        value: body
      }]
    })
  });

  if (!response.ok) {
    const error = await response.text();
    return {
      success: false,
      error: `SendGrid error: ${error}`
    };
  }

  // SendGrid doesn't return message ID in response, use timestamp
  return {
    success: true,
    messageId: `sg_${Date.now()}`
  };
}

// Push Notification Provider (Firebase Cloud Messaging example)
async function sendPushNotification(
  contactId: string | null,
  body: string,
  metadata: any
): Promise<SendResult> {
  // Would resolve FCM token from contactId or user profile
  // Simplified for MVP
  
  const fcmServerKey = Deno.env.get('FCM_SERVER_KEY');
  
  if (!fcmServerKey) {
    return {
      success: false,
      error: 'FCM not configured'
    };
  }

  // Get FCM token from contact/user (would be stored in database)
  const fcmToken = await getFCMToken(contactId);
  
  if (!fcmToken) {
    return {
      success: false,
      error: 'FCM token not found'
    };
  }

  const response = await fetch('https://fcm.googleapis.com/fcm/send', {
    method: 'POST',
    headers: {
      'Authorization': `key=${fcmServerKey}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      to: fcmToken,
      notification: {
        title: metadata.job_title || 'Notification',
        body: body
      },
      data: {
        job_id: metadata.job_id,
        assignment_id: metadata.assignment_id
      }
    })
  });

  if (!response.ok) {
    const error = await response.text();
    return {
      success: false,
      error: `FCM error: ${error}`
    };
  }

  const data = await response.json();
  return {
    success: true,
    messageId: data.message_id || `fcm_${Date.now()}`
  };
}

async function getFCMToken(contactId: string | null): Promise<string | null> {
  // Would query database for FCM token associated with contact/user
  // Simplified for MVP
  return null;
}
```

### 5.4 Retry Logic

**Retry Strategy**:

```typescript
async function shouldRetry(
  notification: any,
  maxRetries: number = 3
): Promise<boolean> {
  const retryCount = notification.metadata?.retry_count || 0;
  
  if (retryCount >= maxRetries) {
    return false; // Max retries exceeded
  }

  // Exponential backoff: 5min, 15min, 30min
  const backoffMinutes = [5, 15, 30][retryCount] || 30;
  const lastFailedAt = notification.metadata?.failed_at 
    ? new Date(notification.metadata.failed_at)
    : null;

  if (!lastFailedAt) {
    return true; // No previous failure, can retry
  }

  const retryAfter = new Date(lastFailedAt.getTime() + backoffMinutes * 60000);
  
  return new Date() >= retryAfter;
}

// Update scheduled_send_at for retry
async function scheduleRetry(
  supabase: SupabaseClient,
  notificationId: string,
  backoffMinutes: number
): Promise<void> {
  const newScheduledTime = new Date(Date.now() + backoffMinutes * 60000);
  
  await supabase
    .from('job_notifications')
    .update({
      status: 'pending',
      scheduled_send_at: newScheduledTime.toISOString(),
      error_message: null
    })
    .eq('id', notificationId);
}
```

### 5.5 Cron Configuration

**Supabase Cron Setup**:

```sql
-- Create cron job to run every minute
SELECT cron.schedule(
  'process-job-notifications',
  '* * * * *', -- Every minute
  $$
  SELECT net.http_post(
    url := 'https://your-project.supabase.co/functions/v1/process_job_notifications',
    headers := jsonb_build_object(
      'Content-Type', 'application/json',
      'Authorization', 'Bearer YOUR_SERVICE_ROLE_KEY'
    ),
    body := '{}'::jsonb
  );
  $$
);
```

---

## 6. Story DISP-050: Notification Template Variables Contract

### 6.1 Template Variable Schema

**Common Variables** (available for all notification types):

```typescript
interface CommonTemplateVariables {
  // Job Information
  job_id: string;
  job_title: string;
  job_description: string;
  
  // Customer Information
  customer_name: string;
  customer_id: string;
  
  // Location Information
  location_address: string; // Full formatted address
  
  // Scheduling Information
  scheduled_start_at: string; // ISO 8601 timestamp
  scheduled_end_at: string; // ISO 8601 timestamp
  scheduled_start_formatted: string; // Human-readable format
  scheduled_end_formatted: string; // Human-readable format
  
  // Technician Information
  technician_name: string;
  technician_id: string;
}
```

**Type-Specific Variables**:

**`pre_appointment_reminder`**:
```typescript
interface PreAppointmentReminderVariables extends CommonTemplateVariables {
  // No additional fields
}
```

**`same_day_reminder`**:
```typescript
interface SameDayReminderVariables extends CommonTemplateVariables {
  // No additional fields
}
```

**`tech_on_the_way`**:
```typescript
interface TechOnTheWayVariables extends CommonTemplateVariables {
  tech_eta_at: string; // ISO 8601 timestamp
  tech_eta_formatted: string; // Human-readable format
}
```

**`booking_confirmation`**:
```typescript
interface BookingConfirmationVariables extends CommonTemplateVariables {
  // No additional fields
}
```

**`reschedule_notice`**:
```typescript
interface RescheduleNoticeVariables extends CommonTemplateVariables {
  old_scheduled_start_at: string; // ISO 8601 timestamp
  old_scheduled_start_formatted: string; // Human-readable format
  new_scheduled_start_at: string; // ISO 8601 timestamp (same as scheduled_start_at)
  delay_minutes: number; // Delay in minutes
}
```

**`cancellation_notice`**:
```typescript
interface CancellationNoticeVariables extends CommonTemplateVariables {
  cancellation_reason: string; // Reason for cancellation
}
```

### 6.2 PII Handling Rules

**SMS Channel**:
- ✅ Safe: Customer name, job title, scheduled time, location address, technician name
- ❌ Avoid: Full customer address details, phone numbers, email addresses
- ⚠️ Limit: Keep messages under 160 characters when possible

**Email Channel**:
- ✅ Safe: All common variables, full address, detailed descriptions
- ✅ Can include: Links to customer portal, cancellation/reschedule links
- ⚠️ Limit: Avoid sensitive financial information

**Push Notification Channel**:
- ✅ Safe: Customer name (first name only), job title, scheduled time, technician name
- ❌ Avoid: Full addresses, detailed descriptions
- ⚠️ Limit: Keep title under 50 characters, body under 200 characters

**In-App Channel**:
- ✅ Safe: All variables (user is authenticated)
- ✅ Can include: Full details, action buttons, links

### 6.3 Template Examples

**SMS Templates**:

```typescript
const smsTemplates = {
  pre_appointment_reminder: 'Hi {{customer_name}}, reminder: Your appointment "{{job_title}}" is scheduled for {{scheduled_start_formatted}} at {{location_address}}. See you then!',
  
  same_day_reminder: 'Hi {{customer_name}}, your appointment "{{job_title}}" is in 2 hours at {{scheduled_start_formatted}}. Location: {{location_address}}',
  
  tech_on_the_way: 'Hi {{customer_name}}, {{technician_name}} is on the way! ETA: {{tech_eta_formatted}}',
  
  booking_confirmation: 'Hi {{customer_name}}, your appointment "{{job_title}}" is confirmed for {{scheduled_start_formatted}} at {{location_address}}',
  
  reschedule_notice: 'Hi {{customer_name}}, your appointment "{{job_title}}" has been rescheduled to {{scheduled_start_formatted}}. We apologize for any inconvenience.',
  
  cancellation_notice: 'Hi {{customer_name}}, your appointment "{{job_title}}" scheduled for {{scheduled_start_formatted}} has been canceled. Reason: {{cancellation_reason}}'
};
```

**Email Templates**:

```typescript
const emailTemplates = {
  pre_appointment_reminder: {
    subject: 'Appointment Reminder: {{job_title}}',
    body: `Hi {{customer_name}},

This is a reminder that your appointment "{{job_title}}" is scheduled for:

Date & Time: {{scheduled_start_formatted}}
Location: {{location_address}}
Technician: {{technician_name}}

We look forward to seeing you!

Best regards,
Your Service Team`
  },
  
  same_day_reminder: {
    subject: 'Reminder: Your Appointment is Today',
    body: `Hi {{customer_name}},

Your appointment "{{job_title}}" is scheduled for today at {{scheduled_start_formatted}}.

Location: {{location_address}}
Technician: {{technician_name}}

See you soon!

Best regards,
Your Service Team`
  },
  
  tech_on_the_way: {
    subject: 'Technician On The Way',
    body: `Hi {{customer_name}},

{{technician_name}} is on the way to your location!

Estimated Arrival: {{tech_eta_formatted}}

We'll see you soon!

Best regards,
Your Service Team`
  },
  
  booking_confirmation: {
    subject: 'Appointment Confirmed: {{job_title}}',
    body: `Hi {{customer_name}},

Your appointment has been confirmed:

Service: {{job_title}}
Date & Time: {{scheduled_start_formatted}}
Location: {{location_address}}
Technician: {{technician_name}}

We look forward to serving you!

Best regards,
Your Service Team`
  },
  
  reschedule_notice: {
    subject: 'Appointment Rescheduled: {{job_title}}',
    body: `Hi {{customer_name}},

Your appointment "{{job_title}}" has been rescheduled:

Previous Time: {{old_scheduled_start_formatted}}
New Time: {{scheduled_start_formatted}}

We apologize for any inconvenience this may cause.

Best regards,
Your Service Team`
  },
  
  cancellation_notice: {
    subject: 'Appointment Canceled: {{job_title}}',
    body: `Hi {{customer_name}},

Your appointment "{{job_title}}" scheduled for {{scheduled_start_formatted}} has been canceled.

Reason: {{cancellation_reason}}

Please contact us if you need to reschedule.

Best regards,
Your Service Team`
  }
};
```

### 6.4 Template Storage

**Database Table** (Optional, for dynamic templates):

```sql
CREATE TABLE IF NOT EXISTS notification_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID REFERENCES orgs(id) ON DELETE CASCADE,
  notification_type notification_type_enum NOT NULL,
  channel notification_channel_enum NOT NULL,
  subject TEXT,
  body TEXT NOT NULL,
  is_default BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  UNIQUE(org_id, notification_type, channel) WHERE org_id IS NOT NULL,
  UNIQUE(notification_type, channel, is_default) WHERE org_id IS NULL AND is_default = true
);

CREATE INDEX idx_notification_templates_org_type_channel 
  ON notification_templates(org_id, notification_type, channel) 
  WHERE org_id IS NOT NULL;
```

---

## 7. Error Handling

### 7.1 Standard Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid authentication token |
| `FORBIDDEN` | 403 | User not authorized |
| `MISSING_JOB_ID` | 400 | Job ID required |
| `NO_ASSIGNMENTS` | 404 | No active assignments found |
| `CREATE_ERROR` | 500 | Failed to create notifications |
| `TEMPLATE_NOT_FOUND` | 500 | Template not found for notification type/channel |
| `PROVIDER_ERROR` | 500 | Messaging provider error |

---

## 8. Implementation Checklist

### Story DISP-048: Schedule Standard Notifications

- [ ] **POST /dispatch/jobs/:id/schedule_notifications**:
  - [ ] Endpoint implemented
  - [ ] Notification rules configuration implemented
  - [ ] Org-level rules table created
  - [ ] Default rules defined
  - [ ] Override rules support implemented
  - [ ] Recipient contact resolution implemented
  - [ ] Metadata building implemented
  - [ ] Notification creation implemented
  - [ ] Skip existing logic implemented
  - [ ] API documentation with examples

- [ ] **Notification Rules**:
  - [ ] Default rules documented
  - [ ] Org-level configuration implemented
  - [ ] Rule evaluation logic implemented

### Story DISP-049: Notification Processor

- [ ] **process_job_notifications Edge Function**:
  - [ ] Function implemented
  - [ ] Pending notification fetching implemented
  - [ ] Idempotency check implemented
  - [ ] Template resolution implemented
  - [ ] Provider integration implemented
  - [ ] Status update logic implemented
  - [ ] Error handling implemented
  - [ ] Retry logic implemented

- [ ] **Provider Integration**:
  - [ ] SMS provider integration (Twilio or alternative)
  - [ ] Email provider integration (SendGrid or alternative)
  - [ ] Push notification provider integration (FCM or alternative)
  - [ ] Provider error handling

- [ ] **Cron Configuration**:
  - [ ] Cron job configured
  - [ ] Schedule documented
  - [ ] Operational runbook created

- [ ] **Retry Logic**:
  - [ ] Retry strategy implemented
  - [ ] Exponential backoff implemented
  - [ ] Max retries configuration

### Story DISP-050: Template Variables Contract

- [ ] **Template Variable Schema**:
  - [ ] Common variables documented
  - [ ] Type-specific variables documented
  - [ ] JSON schema created

- [ ] **PII Handling Rules**:
  - [ ] SMS PII rules documented
  - [ ] Email PII rules documented
  - [ ] Push notification PII rules documented
  - [ ] In-app PII rules documented

- [ ] **Template Examples**:
  - [ ] SMS templates provided
  - [ ] Email templates provided
  - [ ] Push notification templates provided

- [ ] **Template Storage** (Optional):
  - [ ] Template table created
  - [ ] Template CRUD APIs implemented

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 10 – Notifications & Reminder Orchestration. All APIs are designed as Supabase Edge Functions with complete notification scheduling, template resolution, provider integration, and cron-based processing.

**Next Steps**: After completing Epic 10, proceed to Epic 11 (Dispatch Console UI) which will implement the dispatcher-facing Next.js UI components.

