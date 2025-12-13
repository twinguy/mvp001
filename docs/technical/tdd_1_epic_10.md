# Technical Design Document – Epic 10: Security, Privacy & Compliance

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 10 – Security, Privacy & Compliance
- **Source**: Derived from `fdd_1_agile.md` Epic 10 (Stories CRM-042 through CRM-043)
- **Reference Documents**: 
  - `fdd_1.md` (Functional Design Document for CRM, §7)
  - `fdd_1_agile.md` (Agile User Stories)
  - `tdd_1_epic_1.md` (CRM Core Data Model - prerequisite)
  - `tdd_1_epic_2.md` (Authentication, Authorization & RLS Policies - prerequisite)
  - `tdd_1_epic_4.md` (Interaction & Communication Logging - prerequisite)
  - `tdd_1_epic_5.md` (Follow-Ups & Reminders - prerequisite)
  - `tdd_1_epic_7.md` (Automation Engine - prerequisite)
  - `tdd_1_epic_9.md` (Frontend UI - prerequisite)
  - `functional.md` (Platform Capabilities)
  - `tooling.md` (Platform Choices: Supabase + Vercel)
- **Target Platform**: Supabase (PostgreSQL 15+ with Edge Functions) + Next.js (Frontend)
- **Purpose**: Comprehensive technical specification for LLM-based code generation
- **Prerequisites**: Epic 1-9 must be completed first

---

## 1. Overview

This document provides complete technical specifications for implementing security, privacy, and compliance features in the CRM module. It covers:

- Communication preference enforcement in automations and messaging
- Preference checking utility functions
- Manual messaging UI warnings
- Audit logging system for CRM data changes
- Audit log access controls and retention policies
- Request/response schemas with exact JSON structures
- Database triggers and Edge Function implementations
- Frontend components with shadcn/ui

All specifications are designed to be directly implementable in Supabase (PostgreSQL and Edge Functions) and Next.js, with exact schemas, validation rules, and error codes defined.

---

## 2. Architecture Decisions

### 2.1 Preference Enforcement Strategy

**Decision**: Centralize preference checking in reusable functions:

- **PostgreSQL Functions**: For database-level preference checks
- **Edge Function Utilities**: For API-level preference validation
- **Frontend Components**: For UI warnings before sending messages

**Rationale**: 
- Centralized logic ensures consistency across all communication channels
- Database functions provide efficient checks without multiple queries
- Frontend warnings prevent user errors before API calls

### 2.2 Audit Logging Strategy

**Decision**: Use dedicated audit table with triggers:

- **Audit Table**: Separate `crm_audit_log` table for all audit events
- **Database Triggers**: Automatic logging on INSERT/UPDATE/DELETE
- **Edge Function Logging**: For API-level events and external integrations
- **Retention Policy**: Configurable retention period (default: 7 years)

**Rationale**:
- Dedicated table provides better query performance and isolation
- Triggers ensure all changes are logged automatically
- Separate table allows independent retention and archival policies

### 2.3 Access Control for Audit Logs

**Decision**: Role-based access with RLS:

- **View Access**: `admin` and `manager` roles only
- **RLS Policies**: Enforce org-level isolation
- **No Modification**: Audit logs are append-only (no UPDATE/DELETE allowed)

---

## 3. Story CRM-042: Enforce Communication Preferences

### 3.1 Preference Checking Utility Functions

#### 3.1.1 PostgreSQL Function: Check Communication Allowed

**File**: `supabase/migrations/YYYYMMDDHHMMSS_create_preference_check_functions.sql`

```sql
-- Function to check if a communication channel is allowed for a customer
CREATE OR REPLACE FUNCTION crm_check_communication_allowed(
  p_customer_id UUID,
  p_org_id UUID,
  p_channel TEXT -- 'email', 'sms', 'phone', 'any'
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_preferences RECORD;
  v_contact RECORD;
  v_result JSONB;
  v_allowed BOOLEAN := true;
  v_reason TEXT;
BEGIN
  -- Get customer preferences
  SELECT 
    do_not_contact,
    do_not_email,
    do_not_sms,
    do_not_call
  INTO v_preferences
  FROM crm_preferences
  WHERE customer_id = p_customer_id
    AND org_id = p_org_id;
  
  -- If no preferences record exists, allow communication
  IF NOT FOUND THEN
    RETURN jsonb_build_object(
      'allowed', true,
      'reason', 'no_preferences_set'
    );
  END IF;
  
  -- Check global do_not_contact flag
  IF v_preferences.do_not_contact THEN
    RETURN jsonb_build_object(
      'allowed', false,
      'reason', 'do_not_contact_flag_set',
      'channel', p_channel
    );
  END IF;
  
  -- Check channel-specific flags
  IF p_channel = 'email' AND v_preferences.do_not_email THEN
    RETURN jsonb_build_object(
      'allowed', false,
      'reason', 'do_not_email_flag_set',
      'channel', p_channel
    );
  END IF;
  
  IF p_channel = 'sms' AND v_preferences.do_not_sms THEN
    RETURN jsonb_build_object(
      'allowed', false,
      'reason', 'do_not_sms_flag_set',
      'channel', p_channel
    );
  END IF;
  
  IF p_channel = 'phone' AND v_preferences.do_not_call THEN
    RETURN jsonb_build_object(
      'allowed', false,
      'reason', 'do_not_call_flag_set',
      'channel', p_channel
    );
  END IF;
  
  -- If checking 'any' channel, check all flags
  IF p_channel = 'any' THEN
    IF v_preferences.do_not_email OR v_preferences.do_not_sms OR v_preferences.do_not_call THEN
      RETURN jsonb_build_object(
        'allowed', false,
        'reason', 'one_or_more_channel_flags_set',
        'channels_blocked', jsonb_build_object(
          'email', v_preferences.do_not_email,
          'sms', v_preferences.do_not_sms,
          'phone', v_preferences.do_not_call
        )
      );
    END IF;
  END IF;
  
  -- Check contact-level opt-in flags if specific contact is provided
  -- This would require an additional parameter p_contact_id
  -- For now, return allowed if preference flags pass
  
  RETURN jsonb_build_object(
    'allowed', true,
    'reason', 'preferences_allow_communication',
    'channel', p_channel
  );
END;
$$;

-- Function to check contact-level opt-in flags
CREATE OR REPLACE FUNCTION crm_check_contact_opt_in(
  p_contact_id UUID,
  p_org_id UUID,
  p_communication_type TEXT -- 'marketing', 'transactional'
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_contact RECORD;
BEGIN
  SELECT 
    opt_in_marketing,
    opt_in_transactional
  INTO v_contact
  FROM customer_contacts
  WHERE id = p_contact_id
    AND org_id = p_org_id;
  
  IF NOT FOUND THEN
    RETURN jsonb_build_object(
      'allowed', false,
      'reason', 'contact_not_found'
    );
  END IF;
  
  IF p_communication_type = 'marketing' AND NOT COALESCE(v_contact.opt_in_marketing, false) THEN
    RETURN jsonb_build_object(
      'allowed', false,
      'reason', 'marketing_opt_in_not_granted',
      'contact_id', p_contact_id
    );
  END IF;
  
  IF p_communication_type = 'transactional' AND NOT COALESCE(v_contact.opt_in_transactional, false) THEN
    RETURN jsonb_build_object(
      'allowed', false,
      'reason', 'transactional_opt_in_not_granted',
      'contact_id', p_contact_id
    );
  END IF;
  
  RETURN jsonb_build_object(
    'allowed', true,
    'reason', 'opt_in_granted',
    'communication_type', p_communication_type
  );
END;
$$;

-- Comprehensive function that checks both preferences and contact opt-ins
CREATE OR REPLACE FUNCTION crm_check_communication_permission(
  p_customer_id UUID,
  p_org_id UUID,
  p_channel TEXT,
  p_contact_id UUID DEFAULT NULL,
  p_communication_type TEXT DEFAULT 'transactional' -- 'marketing' or 'transactional'
)
RETURNS JSONB
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_preference_check JSONB;
  v_contact_check JSONB;
  v_result JSONB;
BEGIN
  -- First check customer-level preferences
  v_preference_check := crm_check_communication_allowed(
    p_customer_id,
    p_org_id,
    p_channel
  );
  
  IF NOT (v_preference_check->>'allowed')::boolean THEN
    RETURN v_preference_check;
  END IF;
  
  -- If contact_id is provided, check contact-level opt-ins
  IF p_contact_id IS NOT NULL THEN
    v_contact_check := crm_check_contact_opt_in(
      p_contact_id,
      p_org_id,
      p_communication_type
    );
    
    IF NOT (v_contact_check->>'allowed')::boolean THEN
      RETURN v_contact_check;
    END IF;
  END IF;
  
  -- All checks passed
  RETURN jsonb_build_object(
    'allowed', true,
    'reason', 'all_checks_passed',
    'channel', p_channel,
    'communication_type', p_communication_type
  );
END;
$$;
```

### 3.2 Integration with Automation Engine

#### 3.2.1 Updated Automation Action Execution

**File**: `supabase/functions/crm-automation-handle-event/index.ts` (Update)

Add preference checking before executing `send_message` actions:

```typescript
// In executeActions function, before send_message action:

if (action.type === 'send_message') {
  // Check communication preferences
  const { data: permissionCheck } = await supabaseAdmin.rpc(
    'crm_check_communication_permission',
    {
      p_customer_id: customerId,
      p_org_id: orgId,
      p_channel: action.channel === 'email' ? 'email' : action.channel === 'sms' ? 'sms' : 'phone',
      p_contact_id: action.contact_id || null,
      p_communication_type: action.communication_type || 'transactional',
    }
  );
  
  if (!permissionCheck.allowed) {
    results.push({
      type: action.type,
      status: 'skipped',
      error: `Communication blocked: ${permissionCheck.reason}`,
      reason: permissionCheck.reason,
    });
    continue;
  }
  
  // Proceed with sending message
  // ... existing send_message logic
}
```

#### 3.2.2 Updated Time-Based Automation Processor

**File**: `supabase/functions/crm-automation-process-time-based/index.ts` (Update)

Add preference checking in the same way as event-triggered automations.

#### 3.2.3 Updated Segment-Based Automation Processor

**File**: `supabase/functions/crm-automation-process-segment-membership/index.ts` (Update)

Add preference checking before executing actions for each customer.

### 3.3 Frontend: Manual Messaging UI Warnings

#### 3.3.1 Communication Preference Check Hook

**File**: `hooks/use-communication-permission.ts`

```typescript
import { useState, useCallback } from 'react';
import { createClient } from '@/lib/supabase/client';
import type { CommunicationPermissionResult } from '@/types/compliance';

interface CheckPermissionParams {
  customerId: string;
  channel: 'email' | 'sms' | 'phone';
  contactId?: string;
  communicationType?: 'marketing' | 'transactional';
}

export function useCommunicationPermission() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const supabase = createClient();

  const checkPermission = useCallback(async (
    params: CheckPermissionParams
  ): Promise<CommunicationPermissionResult | null> => {
    try {
      setLoading(true);
      setError(null);

      // Get user's org_id from profile
      const { data: { user } } = await supabase.auth.getUser();
      if (!user) {
        throw new Error('User not authenticated');
      }

      const { data: profile, error: profileError } = await supabase
        .from('profiles')
        .select('org_id')
        .eq('id', user.id)
        .single();

      if (profileError || !profile) {
        throw new Error('Failed to load user profile');
      }

      const { data, error: checkError } = await supabase.rpc(
        'crm_check_communication_permission',
        {
          p_customer_id: params.customerId,
          p_org_id: profile.org_id,
          p_channel: params.channel,
          p_contact_id: params.contactId || null,
          p_communication_type: params.communicationType || 'transactional',
        }
      );

      if (checkError) throw checkError;
      return data;
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to check permission');
      return null;
    } finally {
      setLoading(false);
    }
  }, [supabase]);

  return { checkPermission, loading, error };
}
```

#### 3.3.2 Communication Warning Component

**File**: `components/crm/compliance/communication-warning.tsx`

```typescript
'use client';

import { useEffect, useState } from 'react';
import { Alert, AlertDescription, AlertTitle } from '@/components/ui/alert';
import { AlertTriangle, Mail, Phone, MessageSquare } from 'lucide-react';
import { useCommunicationPermission } from '@/hooks/use-communication-permission';
import type { CommunicationPermissionResult } from '@/types/compliance';

interface CommunicationWarningProps {
  customerId: string;
  channel: 'email' | 'sms' | 'phone';
  contactId?: string;
  communicationType?: 'marketing' | 'transactional';
  onPermissionChecked?: (result: CommunicationPermissionResult) => void;
}

const channelIcons = {
  email: Mail,
  sms: MessageSquare,
  phone: Phone,
};

const channelLabels = {
  email: 'Email',
  sms: 'SMS',
  phone: 'Phone Call',
};

const reasonMessages: Record<string, string> = {
  do_not_contact_flag_set: 'This customer has opted out of all communications.',
  do_not_email_flag_set: 'This customer has opted out of email communications.',
  do_not_sms_flag_set: 'This customer has opted out of SMS communications.',
  do_not_call_flag_set: 'This customer has opted out of phone calls.',
  marketing_opt_in_not_granted: 'This contact has not opted in to marketing communications.',
  transactional_opt_in_not_granted: 'This contact has not opted in to transactional communications.',
  one_or_more_channel_flags_set: 'This customer has opted out of one or more communication channels.',
};

export function CommunicationWarning({
  customerId,
  channel,
  contactId,
  communicationType = 'transactional',
  onPermissionChecked,
}: CommunicationWarningProps) {
  const { checkPermission, loading } = useCommunicationPermission();
  const [permissionResult, setPermissionResult] = useState<CommunicationPermissionResult | null>(null);

  useEffect(() => {
    const check = async () => {
      const result = await checkPermission({
        customerId,
        channel,
        contactId,
        communicationType,
      });
      
      if (result) {
        setPermissionResult(result);
        onPermissionChecked?.(result);
      }
    };

    check();
  }, [customerId, channel, contactId, communicationType, checkPermission, onPermissionChecked]);

  if (loading) {
    return (
      <Alert>
        <AlertTitle>Checking communication preferences...</AlertTitle>
      </Alert>
    );
  }

  if (!permissionResult || permissionResult.allowed) {
    return null;
  }

  const Icon = channelIcons[channel];
  const reason = permissionResult.reason || 'unknown_reason';
  const message = reasonMessages[reason] || `Communication is not allowed: ${reason}`;

  return (
    <Alert variant="destructive">
      <AlertTriangle className="h-4 w-4" />
      <AlertTitle>Communication Not Allowed</AlertTitle>
      <AlertDescription>
        <div className="flex items-center gap-2 mb-2">
          <Icon className="h-4 w-4" />
          <span className="font-medium">{channelLabels[channel]} communication is blocked</span>
        </div>
        <p>{message}</p>
        {permissionResult.channels_blocked && (
          <div className="mt-2 text-sm">
            <p>Blocked channels:</p>
            <ul className="list-disc list-inside">
              {permissionResult.channels_blocked.email && <li>Email</li>}
              {permissionResult.channels_blocked.sms && <li>SMS</li>}
              {permissionResult.channels_blocked.phone && <li>Phone</li>}
            </ul>
          </div>
        )}
      </AlertDescription>
    </Alert>
  );
}
```

#### 3.3.3 Send Message Dialog Component

**File**: `components/crm/messaging/send-message-dialog.tsx`

```typescript
'use client';

import { useState, useEffect } from 'react';
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
} from '@/components/ui/dialog';
import { Button } from '@/components/ui/button';
import { Label } from '@/components/ui/label';
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select';
import { Textarea } from '@/components/ui/textarea';
import { CommunicationWarning } from '@/components/crm/compliance/communication-warning';
import { useCommunicationPermission } from '@/hooks/use-communication-permission';
import { createClient } from '@/lib/supabase/client';
import { toast } from '@/components/ui/use-toast';

interface SendMessageDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  customerId: string;
  customerName: string;
  contacts?: Array<{
    id: string;
    type: string;
    value: string;
  }>;
}

export function SendMessageDialog({
  open,
  onOpenChange,
  customerId,
  customerName,
  contacts = [],
}: SendMessageDialogProps) {
  const supabase = createClient();
  const { checkPermission } = useCommunicationPermission();
  
  const [channel, setChannel] = useState<'email' | 'sms' | 'phone'>('email');
  const [selectedContactId, setSelectedContactId] = useState<string>('');
  const [communicationType, setCommunicationType] = useState<'marketing' | 'transactional'>('transactional');
  const [message, setMessage] = useState('');
  const [permissionAllowed, setPermissionAllowed] = useState(true);
  const [sending, setSending] = useState(false);

  useEffect(() => {
    if (open && customerId && channel) {
      checkPermission({
        customerId,
        channel,
        contactId: selectedContactId || undefined,
        communicationType,
      }).then((result) => {
        setPermissionAllowed(result?.allowed ?? false);
      });
    }
  }, [open, customerId, channel, selectedContactId, communicationType, checkPermission]);

  const handleSend = async () => {
    if (!permissionAllowed) {
      toast({
        title: 'Cannot Send Message',
        description: 'Communication is not allowed due to customer preferences.',
        variant: 'destructive',
      });
      return;
    }

    try {
      setSending(true);
      
      // Call send message API
      const { error } = await supabase.functions.invoke('crm-send-message', {
        body: {
          customer_id: customerId,
          channel,
          contact_id: selectedContactId || null,
          message,
          communication_type: communicationType,
        },
      });

      if (error) throw error;

      toast({
        title: 'Message Sent',
        description: `Message sent to ${customerName} via ${channel}.`,
      });

      onOpenChange(false);
      setMessage('');
    } catch (err) {
      toast({
        title: 'Failed to Send Message',
        description: err instanceof Error ? err.message : 'An error occurred',
        variant: 'destructive',
      });
    } finally {
      setSending(false);
    }
  };

  const availableContacts = contacts.filter((c) => {
    if (channel === 'email') return c.type === 'email';
    if (channel === 'sms') return c.type === 'mobile' || c.type === 'sms';
    if (channel === 'phone') return c.type === 'phone' || c.type === 'mobile';
    return false;
  });

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="sm:max-w-[500px]">
        <DialogHeader>
          <DialogTitle>Send Message to {customerName}</DialogTitle>
          <DialogDescription>
            Send a message to this customer. Communication preferences will be checked automatically.
          </DialogDescription>
        </DialogHeader>

        <div className="space-y-4 py-4">
          <div className="space-y-2">
            <Label>Channel</Label>
            <Select value={channel} onValueChange={(value: 'email' | 'sms' | 'phone') => setChannel(value)}>
              <SelectTrigger>
                <SelectValue />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="email">Email</SelectItem>
                <SelectItem value="sms">SMS</SelectItem>
                <SelectItem value="phone">Phone Call</SelectItem>
              </SelectContent>
            </Select>
          </div>

          {availableContacts.length > 0 && (
            <div className="space-y-2">
              <Label>Contact</Label>
              <Select value={selectedContactId} onValueChange={setSelectedContactId}>
                <SelectTrigger>
                  <SelectValue placeholder="Select contact" />
                </SelectTrigger>
                <SelectContent>
                  {availableContacts.map((contact) => (
                    <SelectItem key={contact.id} value={contact.id}>
                      {contact.value} ({contact.type})
                    </SelectItem>
                  ))}
                </SelectContent>
              </Select>
            </div>
          )}

          <div className="space-y-2">
            <Label>Communication Type</Label>
            <Select value={communicationType} onValueChange={(value: 'marketing' | 'transactional') => setCommunicationType(value)}>
              <SelectTrigger>
                <SelectValue />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="transactional">Transactional</SelectItem>
                <SelectItem value="marketing">Marketing</SelectItem>
              </SelectContent>
            </Select>
          </div>

          <CommunicationWarning
            customerId={customerId}
            channel={channel}
            contactId={selectedContactId || undefined}
            communicationType={communicationType}
            onPermissionChecked={(result) => setPermissionAllowed(result.allowed)}
          />

          <div className="space-y-2">
            <Label>Message</Label>
            <Textarea
              value={message}
              onChange={(e) => setMessage(e.target.value)}
              placeholder="Enter your message..."
              rows={5}
            />
          </div>
        </div>

        <DialogFooter>
          <Button variant="outline" onClick={() => onOpenChange(false)}>
            Cancel
          </Button>
          <Button onClick={handleSend} disabled={!permissionAllowed || sending || !message.trim()}>
            {sending ? 'Sending...' : 'Send Message'}
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}
```

---

## 4. Story CRM-043: CRM Data Access Audit Logging

### 4.1 Audit Log Table Schema

**File**: `supabase/migrations/YYYYMMDDHHMMSS_create_audit_log_table.sql`

```sql
-- Audit log table for tracking CRM data changes
CREATE TABLE IF NOT EXISTS crm_audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES orgs(id) ON DELETE CASCADE,
  table_name TEXT NOT NULL, -- e.g., 'customers', 'crm_preferences'
  record_id UUID NOT NULL, -- ID of the affected record
  operation TEXT NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE', 'SELECT')),
  user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  user_email TEXT, -- Denormalized for easier querying
  user_role TEXT, -- Denormalized role from profiles
  old_values JSONB, -- Previous values (for UPDATE/DELETE)
  new_values JSONB, -- New values (for INSERT/UPDATE)
  changed_fields TEXT[], -- Array of field names that changed
  ip_address INET,
  user_agent TEXT,
  metadata JSONB, -- Additional context (e.g., API endpoint, Edge Function name)
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  -- Constraints
  CONSTRAINT chk_crm_audit_log_operation_values CHECK (
    operation IN ('INSERT', 'UPDATE', 'DELETE', 'SELECT')
  )
);

-- Indexes for common query patterns
CREATE INDEX idx_crm_audit_log_org_id_created_at ON crm_audit_log(org_id, created_at DESC);
CREATE INDEX idx_crm_audit_log_table_name_record_id ON crm_audit_log(table_name, record_id);
CREATE INDEX idx_crm_audit_log_user_id_created_at ON crm_audit_log(user_id, created_at DESC);
CREATE INDEX idx_crm_audit_log_operation_created_at ON crm_audit_log(operation, created_at DESC);
CREATE INDEX idx_crm_audit_log_org_id_table_name ON crm_audit_log(org_id, table_name);

-- GIN index for JSONB queries
CREATE INDEX idx_crm_audit_log_metadata ON crm_audit_log USING gin(metadata);
CREATE INDEX idx_crm_audit_log_old_values ON crm_audit_log USING gin(old_values);
CREATE INDEX idx_crm_audit_log_new_values ON crm_audit_log USING gin(new_values);

-- RLS Policy: Only admins and managers can view audit logs
ALTER TABLE crm_audit_log ENABLE ROW LEVEL SECURITY;

CREATE POLICY crm_audit_log_select_policy ON crm_audit_log
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM profiles
      WHERE profiles.id = auth.uid()
      AND profiles.org_id = crm_audit_log.org_id
      AND profiles.role IN ('admin', 'manager')
      AND profiles.is_active = true
    )
  );

-- Prevent modifications to audit logs (append-only)
CREATE POLICY crm_audit_log_insert_policy ON crm_audit_log
  FOR INSERT
  WITH CHECK (true); -- Only service role can insert

CREATE POLICY crm_audit_log_no_update ON crm_audit_log
  FOR UPDATE
  USING (false); -- No updates allowed

CREATE POLICY crm_audit_log_no_delete ON crm_audit_log
  FOR DELETE
  USING (false); -- No deletes allowed (except via retention policy)
```

### 4.2 Audit Logging Function

**File**: `supabase/migrations/YYYYMMDDHHMMSS_create_audit_logging_function.sql`

```sql
-- Function to create audit log entries
CREATE OR REPLACE FUNCTION crm_create_audit_log(
  p_org_id UUID,
  p_table_name TEXT,
  p_record_id UUID,
  p_operation TEXT,
  p_user_id UUID DEFAULT NULL,
  p_old_values JSONB DEFAULT NULL,
  p_new_values JSONB DEFAULT NULL,
  p_changed_fields TEXT[] DEFAULT NULL,
  p_ip_address INET DEFAULT NULL,
  p_user_agent TEXT DEFAULT NULL,
  p_metadata JSONB DEFAULT NULL
)
RETURNS UUID
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_user_email TEXT;
  v_user_role TEXT;
  v_audit_log_id UUID;
BEGIN
  -- Get user email and role if user_id provided
  IF p_user_id IS NOT NULL THEN
    SELECT 
      p.email,
      pr.role
    INTO v_user_email, v_user_role
    FROM auth.users u
    LEFT JOIN profiles pr ON pr.id = u.id
    WHERE u.id = p_user_id;
  END IF;
  
  -- Insert audit log entry
  INSERT INTO crm_audit_log (
    org_id,
    table_name,
    record_id,
    operation,
    user_id,
    user_email,
    user_role,
    old_values,
    new_values,
    changed_fields,
    ip_address,
    user_agent,
    metadata
  ) VALUES (
    p_org_id,
    p_table_name,
    p_record_id,
    p_operation,
    p_user_id,
    v_user_email,
    v_user_role,
    p_old_values,
    p_new_values,
    p_changed_fields,
    p_ip_address,
    p_user_agent,
    p_metadata
  )
  RETURNING id INTO v_audit_log_id;
  
  RETURN v_audit_log_id;
END;
$$;
```

### 4.3 Database Triggers for Audit Logging

**File**: `supabase/migrations/YYYYMMDDHHMMSS_create_audit_triggers.sql`

```sql
-- Generic trigger function for audit logging
CREATE OR REPLACE FUNCTION crm_audit_trigger_function()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_org_id UUID;
  v_user_id UUID;
  v_old_values JSONB;
  v_new_values JSONB;
  v_changed_fields TEXT[];
  v_operation TEXT;
BEGIN
  -- Determine operation type
  IF TG_OP = 'INSERT' THEN
    v_operation := 'INSERT';
    v_new_values := to_jsonb(NEW);
    v_old_values := NULL;
    v_changed_fields := NULL;
  ELSIF TG_OP = 'UPDATE' THEN
    v_operation := 'UPDATE';
    v_old_values := to_jsonb(OLD);
    v_new_values := to_jsonb(NEW);
    
    -- Calculate changed fields
    SELECT array_agg(key)
    INTO v_changed_fields
    FROM jsonb_each(v_old_values)
    WHERE value IS DISTINCT FROM v_new_values->key;
  ELSIF TG_OP = 'DELETE' THEN
    v_operation := 'DELETE';
    v_old_values := to_jsonb(OLD);
    v_new_values := NULL;
    v_changed_fields := NULL;
  END IF;
  
  -- Get org_id from the record
  IF TG_OP = 'INSERT' THEN
    v_org_id := NEW.org_id;
  ELSE
    v_org_id := OLD.org_id;
  END IF;
  
  -- Get user_id from JWT claims
  v_user_id := auth.uid();
  
  -- Create audit log entry
  PERFORM crm_create_audit_log(
    p_org_id := v_org_id,
    p_table_name := TG_TABLE_NAME,
    p_record_id := COALESCE(NEW.id, OLD.id),
    p_operation := v_operation,
    p_user_id := v_user_id,
    p_old_values := v_old_values,
    p_new_values := v_new_values,
    p_changed_fields := v_changed_fields,
    p_metadata := jsonb_build_object(
      'trigger_name', TG_NAME,
      'trigger_when', TG_WHEN,
      'trigger_level', TG_LEVEL
    )
  );
  
  IF TG_OP = 'DELETE' THEN
    RETURN OLD;
  ELSE
    RETURN NEW;
  END IF;
END;
$$;

-- Create triggers for key CRM tables
CREATE TRIGGER crm_audit_customers_trigger
  AFTER INSERT OR UPDATE OR DELETE ON customers
  FOR EACH ROW
  EXECUTE FUNCTION crm_audit_trigger_function();

CREATE TRIGGER crm_audit_crm_preferences_trigger
  AFTER INSERT OR UPDATE OR DELETE ON crm_preferences
  FOR EACH ROW
  EXECUTE FUNCTION crm_audit_trigger_function();

CREATE TRIGGER crm_audit_customer_contacts_trigger
  AFTER INSERT OR UPDATE OR DELETE ON customer_contacts
  FOR EACH ROW
  EXECUTE FUNCTION crm_audit_trigger_function();

CREATE TRIGGER crm_audit_customer_locations_trigger
  AFTER INSERT OR UPDATE OR DELETE ON customer_locations
  FOR EACH ROW
  EXECUTE FUNCTION crm_audit_trigger_function();

CREATE TRIGGER crm_audit_crm_interactions_trigger
  AFTER INSERT OR UPDATE OR DELETE ON crm_interactions
  FOR EACH ROW
  EXECUTE FUNCTION crm_audit_trigger_function();

CREATE TRIGGER crm_audit_crm_followups_trigger
  AFTER INSERT OR UPDATE OR DELETE ON crm_followups
  FOR EACH ROW
  EXECUTE FUNCTION crm_audit_trigger_function();

CREATE TRIGGER crm_audit_crm_segments_trigger
  AFTER INSERT OR UPDATE OR DELETE ON crm_segments
  FOR EACH ROW
  EXECUTE FUNCTION crm_audit_trigger_function();

CREATE TRIGGER crm_audit_crm_automation_rules_trigger
  AFTER INSERT OR UPDATE OR DELETE ON crm_automation_rules
  FOR EACH ROW
  EXECUTE FUNCTION crm_audit_trigger_function();
```

### 4.4 Edge Function Audit Logging Helper

**File**: `supabase/functions/_shared/audit-logger.ts`

```typescript
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

export interface AuditLogEntry {
  org_id: string;
  table_name: string;
  record_id: string;
  operation: 'INSERT' | 'UPDATE' | 'DELETE' | 'SELECT';
  user_id?: string;
  old_values?: any;
  new_values?: any;
  changed_fields?: string[];
  ip_address?: string;
  user_agent?: string;
  metadata?: any;
}

export async function createAuditLog(
  supabaseAdmin: any,
  entry: AuditLogEntry
): Promise<void> {
  try {
    const { error } = await supabaseAdmin.rpc('crm_create_audit_log', {
      p_org_id: entry.org_id,
      p_table_name: entry.table_name,
      p_record_id: entry.record_id,
      p_operation: entry.operation,
      p_user_id: entry.user_id || null,
      p_old_values: entry.old_values || null,
      p_new_values: entry.new_values || null,
      p_changed_fields: entry.changed_fields || null,
      p_ip_address: entry.ip_address || null,
      p_user_agent: entry.user_agent || null,
      p_metadata: entry.metadata || null,
    });

    if (error) {
      console.error('Failed to create audit log:', error);
      // Don't throw - audit logging failures shouldn't break operations
    }
  } catch (err) {
    console.error('Audit logging error:', err);
    // Don't throw - audit logging failures shouldn't break operations
  }
}

export function getClientInfo(req: Request): { ip_address?: string; user_agent?: string } {
  return {
    ip_address: req.headers.get('x-forwarded-for')?.split(',')[0] || 
                req.headers.get('x-real-ip') || 
                undefined,
    user_agent: req.headers.get('user-agent') || undefined,
  };
}
```

### 4.5 Audit Log Query Functions

**File**: `supabase/migrations/YYYYMMDDHHMMSS_create_audit_query_functions.sql`

```sql
-- Function to query audit logs with filters
CREATE OR REPLACE FUNCTION crm_query_audit_logs(
  p_org_id UUID,
  p_table_name TEXT DEFAULT NULL,
  p_record_id UUID DEFAULT NULL,
  p_user_id UUID DEFAULT NULL,
  p_operation TEXT DEFAULT NULL,
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
  -- Check user role (must be admin or manager)
  SELECT role INTO v_user_role
  FROM profiles
  WHERE id = auth.uid()
    AND org_id = p_org_id
    AND is_active = true;
  
  IF v_user_role NOT IN ('admin', 'manager') THEN
    RAISE EXCEPTION 'Access denied: Only admins and managers can view audit logs';
  END IF;
  
  -- Build query result
  SELECT jsonb_build_object(
    'data', jsonb_agg(
      jsonb_build_object(
        'id', cal.id,
        'table_name', cal.table_name,
        'record_id', cal.record_id,
        'operation', cal.operation,
        'user_id', cal.user_id,
        'user_email', cal.user_email,
        'user_role', cal.user_role,
        'old_values', cal.old_values,
        'new_values', cal.new_values,
        'changed_fields', cal.changed_fields,
        'ip_address', cal.ip_address,
        'user_agent', cal.user_agent,
        'metadata', cal.metadata,
        'created_at', cal.created_at
      )
    ),
    'total', COUNT(*)
  )
  INTO v_result
  FROM crm_audit_log cal
  WHERE cal.org_id = p_org_id
    AND (p_table_name IS NULL OR cal.table_name = p_table_name)
    AND (p_record_id IS NULL OR cal.record_id = p_record_id)
    AND (p_user_id IS NULL OR cal.user_id = p_user_id)
    AND (p_operation IS NULL OR cal.operation = p_operation)
    AND (p_start_date IS NULL OR cal.created_at >= p_start_date)
    AND (p_end_date IS NULL OR cal.created_at <= p_end_date)
  ORDER BY cal.created_at DESC
  LIMIT p_limit
  OFFSET p_offset;
  
  RETURN COALESCE(v_result, '{"data": [], "total": 0}'::jsonb);
END;
$$;

-- Function to get audit log summary statistics
CREATE OR REPLACE FUNCTION crm_audit_log_summary(
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
    RAISE EXCEPTION 'Access denied: Only admins and managers can view audit logs';
  END IF;
  
  SELECT jsonb_build_object(
    'total_events', COUNT(*),
    'by_operation', jsonb_object_agg(
      operation,
      COUNT(*)
    ),
    'by_table', jsonb_object_agg(
      table_name,
      COUNT(*)
    ),
    'by_user', (
      SELECT jsonb_agg(
        jsonb_build_object(
          'user_id', user_id,
          'user_email', user_email,
          'event_count', COUNT(*)
        )
      )
      FROM crm_audit_log
      WHERE org_id = p_org_id
        AND (p_start_date IS NULL OR created_at >= p_start_date)
        AND (p_end_date IS NULL OR created_at <= p_end_date)
      GROUP BY user_id, user_email
      ORDER BY COUNT(*) DESC
      LIMIT 10
    )
  )
  INTO v_result
  FROM crm_audit_log
  WHERE org_id = p_org_id
    AND (p_start_date IS NULL OR created_at >= p_start_date)
    AND (p_end_date IS NULL OR created_at <= p_end_date);
  
  RETURN COALESCE(v_result, '{}'::jsonb);
END;
$$;
```

### 4.6 Audit Log Retention Policy

**File**: `supabase/migrations/YYYYMMDDHHMMSS_create_audit_retention_policy.sql`

```sql
-- Function to delete old audit logs (retention policy)
CREATE OR REPLACE FUNCTION crm_cleanup_old_audit_logs(
  p_retention_days INTEGER DEFAULT 2555 -- 7 years default
)
RETURNS INTEGER
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_deleted_count INTEGER;
  v_cutoff_date TIMESTAMPTZ;
BEGIN
  v_cutoff_date := now() - (p_retention_days || ' days')::INTERVAL;
  
  DELETE FROM crm_audit_log
  WHERE created_at < v_cutoff_date;
  
  GET DIAGNOSTICS v_deleted_count = ROW_COUNT;
  
  RETURN v_deleted_count;
END;
$$;

-- Create a scheduled job to run retention cleanup (if pg_cron is available)
-- This would typically be configured via Supabase dashboard or Edge Function cron
-- Example cron job (runs monthly):
-- SELECT cron.schedule(
--   'cleanup-old-audit-logs',
--   '0 0 1 * *', -- First day of each month at midnight
--   $$SELECT crm_cleanup_old_audit_logs(2555);$$
-- );
```

### 4.7 Frontend: Audit Log Viewer

#### 4.7.1 Audit Log Page

**File**: `app/(crm)/audit/page.tsx`

```typescript
'use client';

import { useState, useEffect } from 'react';
import { createClient } from '@/lib/supabase/client';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select';
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table';
import { Badge } from '@/components/ui/badge';
import { format } from 'date-fns';
import { Search, Download } from 'lucide-react';
import type { AuditLogEntry } from '@/types/compliance';

export default function AuditLogPage() {
  const supabase = createClient();
  const [logs, setLogs] = useState<AuditLogEntry[]>([]);
  const [loading, setLoading] = useState(true);
  const [total, setTotal] = useState(0);
  const [filters, setFilters] = useState({
    table_name: '',
    operation: '',
    user_id: '',
    start_date: '',
    end_date: '',
  });
  const [pagination, setPagination] = useState({ limit: 50, offset: 0 });

  useEffect(() => {
    loadAuditLogs();
  }, [filters, pagination]);

  const loadAuditLogs = async () => {
    try {
      setLoading(true);
      const { data: { user } } = await supabase.auth.getUser();
      const { data: profile } = await supabase.from('profiles').select('org_id').eq('id', user?.id).single();

      const { data, error } = await supabase.rpc('crm_query_audit_logs', {
        p_org_id: profile?.org_id,
        p_table_name: filters.table_name || null,
        p_operation: filters.operation || null,
        p_user_id: filters.user_id || null,
        p_start_date: filters.start_date || null,
        p_end_date: filters.end_date || null,
        p_limit: pagination.limit,
        p_offset: pagination.offset,
      });

      if (error) throw error;
      setLogs(data.data || []);
      setTotal(data.total || 0);
    } catch (err) {
      console.error('Failed to load audit logs:', err);
    } finally {
      setLoading(false);
    }
  };

  const operationColors: Record<string, 'default' | 'secondary' | 'destructive' | 'outline'> = {
    INSERT: 'default',
    UPDATE: 'secondary',
    DELETE: 'destructive',
    SELECT: 'outline',
  };

  return (
    <div className="container mx-auto py-6 space-y-6">
      <div className="flex items-center justify-between">
        <h1 className="text-3xl font-bold">Audit Logs</h1>
        <Button variant="outline">
          <Download className="mr-2 h-4 w-4" />
          Export
        </Button>
      </div>

      {/* Filters */}
      <Card>
        <CardHeader>
          <CardTitle>Filters</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
            <Select value={filters.table_name} onValueChange={(value) => setFilters(prev => ({ ...prev, table_name: value }))}>
              <SelectTrigger>
                <SelectValue placeholder="Table" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="">All Tables</SelectItem>
                <SelectItem value="customers">Customers</SelectItem>
                <SelectItem value="crm_preferences">Preferences</SelectItem>
                <SelectItem value="customer_contacts">Contacts</SelectItem>
                <SelectItem value="crm_interactions">Interactions</SelectItem>
                <SelectItem value="crm_followups">Follow-ups</SelectItem>
                <SelectItem value="crm_segments">Segments</SelectItem>
                <SelectItem value="crm_automation_rules">Automation Rules</SelectItem>
              </SelectContent>
            </Select>
            <Select value={filters.operation} onValueChange={(value) => setFilters(prev => ({ ...prev, operation: value }))}>
              <SelectTrigger>
                <SelectValue placeholder="Operation" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="">All Operations</SelectItem>
                <SelectItem value="INSERT">Insert</SelectItem>
                <SelectItem value="UPDATE">Update</SelectItem>
                <SelectItem value="DELETE">Delete</SelectItem>
                <SelectItem value="SELECT">Select</SelectItem>
              </SelectContent>
            </Select>
            <Input
              type="date"
              placeholder="Start Date"
              value={filters.start_date}
              onChange={(e) => setFilters(prev => ({ ...prev, start_date: e.target.value }))}
            />
            <Input
              type="date"
              placeholder="End Date"
              value={filters.end_date}
              onChange={(e) => setFilters(prev => ({ ...prev, end_date: e.target.value }))}
            />
          </div>
        </CardContent>
      </Card>

      {/* Audit Logs Table */}
      <Card>
        <CardHeader>
          <CardTitle>Audit Logs ({total})</CardTitle>
        </CardHeader>
        <CardContent>
          {loading ? (
            <div className="text-center py-8">Loading...</div>
          ) : logs.length === 0 ? (
            <div className="text-center py-8 text-muted-foreground">No audit logs found</div>
          ) : (
            <>
              <Table>
                <TableHeader>
                  <TableRow>
                    <TableHead>Timestamp</TableHead>
                    <TableHead>Table</TableHead>
                    <TableHead>Operation</TableHead>
                    <TableHead>User</TableHead>
                    <TableHead>Record ID</TableHead>
                    <TableHead>Changes</TableHead>
                  </TableRow>
                </TableHeader>
                <TableBody>
                  {logs.map((log) => (
                    <TableRow key={log.id}>
                      <TableCell>
                        {format(new Date(log.created_at), 'MMM d, yyyy h:mm a')}
                      </TableCell>
                      <TableCell className="font-mono text-sm">{log.table_name}</TableCell>
                      <TableCell>
                        <Badge variant={operationColors[log.operation] || 'outline'}>
                          {log.operation}
                        </Badge>
                      </TableCell>
                      <TableCell>
                        <div>
                          <div className="font-medium">{log.user_email || 'System'}</div>
                          {log.user_role && (
                            <div className="text-xs text-muted-foreground">{log.user_role}</div>
                          )}
                        </div>
                      </TableCell>
                      <TableCell className="font-mono text-xs">{log.record_id}</TableCell>
                      <TableCell>
                        {log.changed_fields && log.changed_fields.length > 0 ? (
                          <div className="text-sm">
                            {log.changed_fields.length} field(s): {log.changed_fields.join(', ')}
                          </div>
                        ) : (
                          <span className="text-muted-foreground">-</span>
                        )}
                      </TableCell>
                    </TableRow>
                  ))}
                </TableBody>
              </Table>

              {/* Pagination */}
              <div className="flex items-center justify-between mt-4">
                <div className="text-sm text-muted-foreground">
                  Showing {pagination.offset + 1} to {Math.min(pagination.offset + pagination.limit, total)} of {total}
                </div>
                <div className="flex gap-2">
                  <Button
                    variant="outline"
                    disabled={pagination.offset === 0}
                    onClick={() => setPagination(prev => ({ ...prev, offset: Math.max(0, prev.offset - prev.limit) }))}
                  >
                    Previous
                  </Button>
                  <Button
                    variant="outline"
                    disabled={pagination.offset + pagination.limit >= total}
                    onClick={() => setPagination(prev => ({ ...prev, offset: prev.offset + prev.limit }))}
                  >
                    Next
                  </Button>
                </div>
              </div>
            </>
          )}
        </CardContent>
      </Card>
    </div>
  );
}
```

#### 4.7.2 Audit Log Detail Dialog

**File**: `components/crm/compliance/audit-log-detail-dialog.tsx`

```typescript
'use client';

import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
} from '@/components/ui/dialog';
import { Badge } from '@/components/ui/badge';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { format } from 'date-fns';
import type { AuditLogEntry } from '@/types/compliance';

interface AuditLogDetailDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  log: AuditLogEntry | null;
}

export function AuditLogDetailDialog({
  open,
  onOpenChange,
  log,
}: AuditLogDetailDialogProps) {
  if (!log) return null;

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="max-w-4xl max-h-[80vh] overflow-y-auto">
        <DialogHeader>
          <DialogTitle>Audit Log Details</DialogTitle>
          <DialogDescription>
            {format(new Date(log.created_at), 'PPpp')}
          </DialogDescription>
        </DialogHeader>

        <div className="space-y-4">
          <div className="grid grid-cols-2 gap-4">
            <div>
              <div className="text-sm font-medium text-muted-foreground">Table</div>
              <div className="font-mono">{log.table_name}</div>
            </div>
            <div>
              <div className="text-sm font-medium text-muted-foreground">Operation</div>
              <Badge>{log.operation}</Badge>
            </div>
            <div>
              <div className="text-sm font-medium text-muted-foreground">User</div>
              <div>{log.user_email || 'System'}</div>
              {log.user_role && (
                <div className="text-xs text-muted-foreground">{log.user_role}</div>
              )}
            </div>
            <div>
              <div className="text-sm font-medium text-muted-foreground">Record ID</div>
              <div className="font-mono text-xs">{log.record_id}</div>
            </div>
          </div>

          <Tabs defaultValue="changes" className="w-full">
            <TabsList>
              <TabsTrigger value="changes">Changes</TabsTrigger>
              <TabsTrigger value="old">Old Values</TabsTrigger>
              <TabsTrigger value="new">New Values</TabsTrigger>
              <TabsTrigger value="metadata">Metadata</TabsTrigger>
            </TabsList>

            <TabsContent value="changes">
              {log.changed_fields && log.changed_fields.length > 0 ? (
                <div className="space-y-2">
                  {log.changed_fields.map((field) => (
                    <div key={field} className="p-2 bg-muted rounded">
                      {field}
                    </div>
                  ))}
                </div>
              ) : (
                <div className="text-muted-foreground">No changes recorded</div>
              )}
            </TabsContent>

            <TabsContent value="old">
              {log.old_values ? (
                <pre className="bg-muted p-4 rounded overflow-auto text-xs">
                  {JSON.stringify(log.old_values, null, 2)}
                </pre>
              ) : (
                <div className="text-muted-foreground">No old values</div>
              )}
            </TabsContent>

            <TabsContent value="new">
              {log.new_values ? (
                <pre className="bg-muted p-4 rounded overflow-auto text-xs">
                  {JSON.stringify(log.new_values, null, 2)}
                </pre>
              ) : (
                <div className="text-muted-foreground">No new values</div>
              )}
            </TabsContent>

            <TabsContent value="metadata">
              {log.metadata ? (
                <pre className="bg-muted p-4 rounded overflow-auto text-xs">
                  {JSON.stringify(log.metadata, null, 2)}
                </pre>
              ) : (
                <div className="text-muted-foreground">No metadata</div>
              )}
            </TabsContent>
          </Tabs>
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

---

## 5. Type Definitions

### 5.1 Compliance Types

**File**: `types/compliance.ts`

```typescript
export interface CommunicationPermissionResult {
  allowed: boolean;
  reason: string;
  channel?: string;
  communication_type?: string;
  channels_blocked?: {
    email?: boolean;
    sms?: boolean;
    phone?: boolean;
  };
  contact_id?: string;
}

export interface AuditLogEntry {
  id: string;
  org_id: string;
  table_name: string;
  record_id: string;
  operation: 'INSERT' | 'UPDATE' | 'DELETE' | 'SELECT';
  user_id?: string;
  user_email?: string;
  user_role?: string;
  old_values?: any;
  new_values?: any;
  changed_fields?: string[];
  ip_address?: string;
  user_agent?: string;
  metadata?: any;
  created_at: string;
}

export interface AuditLogQueryParams {
  table_name?: string;
  record_id?: string;
  user_id?: string;
  operation?: 'INSERT' | 'UPDATE' | 'DELETE' | 'SELECT';
  start_date?: string;
  end_date?: string;
  limit?: number;
  offset?: number;
}

export interface AuditLogQueryResponse {
  data: AuditLogEntry[];
  total: number;
}
```

---

## 6. Implementation Checklist

### Story CRM-042: Enforce Communication Preferences
- [ ] PostgreSQL preference checking functions implemented
- [ ] Contact opt-in checking functions implemented
- [ ] Comprehensive permission check function implemented
- [ ] Automation engine updated to check preferences before actions
- [ ] Event-triggered automation updated
- [ ] Time-based automation updated
- [ ] Segment-based automation updated
- [ ] Frontend communication permission hook implemented
- [ ] Communication warning component implemented
- [ ] Send message dialog with warnings implemented
- [ ] Test scenarios verify prohibited communications are blocked
- [ ] Documentation for preference-check logic

### Story CRM-043: CRM Data Access Audit Logging
- [ ] Audit log table created with indexes
- [ ] RLS policies for audit log access (admin/manager only)
- [ ] Audit logging function implemented
- [ ] Database triggers created for key tables
- [ ] Edge Function audit logging helper implemented
- [ ] Audit log query functions implemented
- [ ] Audit log summary function implemented
- [ ] Retention policy function implemented
- [ ] Frontend audit log page implemented
- [ ] Audit log detail dialog implemented
- [ ] Export functionality (optional)
- [ ] Documentation for audit log queries and retention

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 10 – Security, Privacy & Compliance. All specifications are designed to be directly consumable by LLM-based code generators, with exact SQL functions, TypeScript interfaces, database triggers, Edge Function utilities, and frontend components using shadcn/ui defined.

