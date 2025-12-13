# Technical Design Document – Epic 9: Frontend (Next.js) CRM UI

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 9 – Frontend (Next.js) CRM UI
- **Source**: Derived from `fdd_1_agile.md` Epic 9 (Stories CRM-036 through CRM-041)
- **Reference Documents**: 
  - `fdd_1.md` (Functional Design Document for CRM, §6)
  - `fdd_1_agile.md` (Agile User Stories)
  - `tdd_1_epic_1.md` through `tdd_1_epic_8.md` (All backend APIs - prerequisites)
  - `tooling.md` (Platform Choices: Supabase + Vercel)
- **Target Platform**: Next.js 14+ (App Router) on Vercel with shadcn/ui
- **Purpose**: Comprehensive technical specification for LLM-based code generation
- **Prerequisites**: Epic 1-8 (All backend APIs) must be completed first

---

## 1. Overview

This document provides complete technical specifications for implementing the CRM frontend UI in Next.js. It covers:

- Next.js project structure and configuration
- shadcn/ui component library integration
- Supabase client setup and authentication
- Page implementations for all CRM features
- Reusable component library
- API integration patterns
- State management and data fetching
- Routing and navigation
- Loading and error states
- Role-based UI rendering

All specifications are designed to be directly implementable in Next.js with TypeScript, using shadcn/ui components and Supabase JS client.

---

## 2. Project Setup & Configuration

### 2.1 Next.js Project Structure

```
app/
  (auth)/
    login/
      page.tsx
    layout.tsx
  (crm)/
    layout.tsx                    # CRM layout with sidebar
    customers/
      page.tsx                    # Customers list
      [id]/
        page.tsx                  # Customer detail
        edit/
          page.tsx                # Edit customer
    followups/
      page.tsx                    # Follow-ups dashboard
    segments/
      page.tsx                    # Segments list
      [id]/
        page.tsx                  # Segment detail
      new/
        page.tsx                  # Create segment
    automation/
      page.tsx                    # Automation rules list
      [id]/
        page.tsx                  # Rule detail
      new/
        page.tsx                  # Create rule
    api/                          # API route handlers (if needed)
components/
  crm/
    customers/
      customer-list.tsx
      customer-card.tsx
      customer-avatar.tsx
      customer-filters.tsx
      customer-table.tsx
    followups/
      followup-card.tsx
      followup-filters.tsx
      followup-list.tsx
    segments/
      segment-card.tsx
      segment-rule-editor.tsx
      segment-members-list.tsx
    automation/
      automation-rule-card.tsx
      automation-rule-editor.tsx
      automation-runs-list.tsx
    shared/
      tag-selector.tsx
      tag-chip.tsx
      interaction-timeline-item.tsx
      status-badge.tsx
      priority-badge.tsx
  ui/                            # shadcn/ui components
    button.tsx
    input.tsx
    table.tsx
    card.tsx
    dialog.tsx
    tabs.tsx
    select.tsx
    badge.tsx
    # ... other shadcn components
lib/
  supabase/
    client.ts                    # Supabase client setup
    server.ts                    # Server-side Supabase client
    middleware.ts                 # Auth middleware
  api/
    customers.ts                 # Customer API functions
    followups.ts                 # Follow-up API functions
    segments.ts                  # Segment API functions
    automation.ts                # Automation API functions
    interactions.ts              # Interaction API functions
  utils/
    cn.ts                        # className utility
    format.ts                    # Date/number formatting
hooks/
  use-customers.ts
  use-followups.ts
  use-segments.ts
  use-automation-rules.ts
types/
  customer.ts
  followup.ts
  segment.ts
  automation.ts
```

### 2.2 Package Dependencies

**package.json**:

```json
{
  "dependencies": {
    "next": "^14.0.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "@supabase/supabase-js": "^2.38.0",
    "@supabase/ssr": "^0.0.10",
    "@radix-ui/react-dialog": "^1.0.5",
    "@radix-ui/react-dropdown-menu": "^2.0.6",
    "@radix-ui/react-label": "^2.0.2",
    "@radix-ui/react-select": "^2.0.0",
    "@radix-ui/react-tabs": "^1.0.4",
    "@radix-ui/react-toast": "^1.1.5",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.0.0",
    "date-fns": "^2.30.0",
    "lucide-react": "^0.294.0",
    "tailwind-merge": "^2.0.0",
    "tailwindcss-animate": "^1.0.7",
    "zod": "^3.22.4",
    "react-hook-form": "^7.48.2",
    "@hookform/resolvers": "^3.3.2"
  },
  "devDependencies": {
    "@types/node": "^20.9.0",
    "@types/react": "^18.2.37",
    "@types/react-dom": "^18.2.15",
    "typescript": "^5.2.2",
    "tailwindcss": "^3.3.5",
    "postcss": "^8.4.31",
    "autoprefixer": "^10.4.16",
    "eslint": "^8.53.0",
    "eslint-config-next": "^14.0.0"
  }
}
```

### 2.3 shadcn/ui Setup

**Installation** (via CLI):

```bash
npx shadcn-ui@latest init
npx shadcn-ui@latest add button input table card dialog tabs select badge label dropdown-menu toast
```

**components.json** (shadcn config):

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "default",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "tailwind.config.js",
    "css": "app/globals.css",
    "baseColor": "slate",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils"
  }
}
```

### 2.4 Supabase Client Setup

**lib/supabase/client.ts**:

```typescript
import { createBrowserClient } from '@supabase/ssr';
import { Database } from '@/types/database';

export function createClient() {
  return createBrowserClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

**lib/supabase/server.ts**:

```typescript
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';
import { Database } from '@/types/database';

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch {
            // The `setAll` method was called from a Server Component.
            // This can be ignored if you have middleware refreshing
            // user sessions.
          }
        },
      },
    }
  );
}
```

**lib/supabase/middleware.ts**:

```typescript
import { createServerClient } from '@supabase/ssr';
import { NextResponse, type NextRequest } from 'next/server';

export async function updateSession(request: NextRequest) {
  let supabaseResponse = NextResponse.next({
    request,
  });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) => request.cookies.set(name, value));
          supabaseResponse = NextResponse.next({
            request,
          });
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  const {
    data: { user },
  } = await supabase.auth.getUser();

  // Protect CRM routes
  if (request.nextUrl.pathname.startsWith('/crm') && !user) {
    const url = request.nextUrl.clone();
    url.pathname = '/login';
    return NextResponse.redirect(url);
  }

  return supabaseResponse;
}
```

**middleware.ts** (root):

```typescript
import { type NextRequest } from 'next/server';
import { updateSession } from '@/lib/supabase/middleware';

export async function middleware(request: NextRequest) {
  return await updateSession(request);
}

export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
  ],
};
```

---

## 3. Story CRM-036: Customers List Page

### 3.1 Page Specification

**File**: `app/(crm)/customers/page.tsx`

**Route**: `/crm/customers`

**Authentication**: Required (enforced by middleware)

### 3.2 Component Structure

```typescript
'use client';

import { useState, useEffect } from 'react';
import { useRouter } from 'next/navigation';
import { createClient } from '@/lib/supabase/client';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select';
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table';
import { CustomerAvatar } from '@/components/crm/customers/customer-avatar';
import { TagChip } from '@/components/crm/shared/tag-chip';
import { StatusBadge } from '@/components/crm/shared/status-badge';
import { Search, Plus, Filter } from 'lucide-react';
import type { Customer } from '@/types/customer';

export default function CustomersPage() {
  const router = useRouter();
  const supabase = createClient();
  
  const [customers, setCustomers] = useState<Customer[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [searchQuery, setSearchQuery] = useState('');
  const [filters, setFilters] = useState({
    status: '',
    lifecycle_stage: '',
    tag: '',
    segment_id: '',
  });
  const [pagination, setPagination] = useState({
    limit: 20,
    offset: 0,
    total: 0,
  });

  useEffect(() => {
    loadCustomers();
  }, [searchQuery, filters, pagination.offset]);

  const loadCustomers = async () => {
    try {
      setLoading(true);
      setError(null);

      const params = new URLSearchParams();
      if (searchQuery) params.set('q', searchQuery);
      if (filters.status) params.set('status', filters.status);
      if (filters.lifecycle_stage) params.set('lifecycle_stage', filters.lifecycle_stage);
      if (filters.tag) params.set('tag', filters.tag);
      if (filters.segment_id) params.set('segment_id', filters.segment_id);
      params.set('limit', pagination.limit.toString());
      params.set('offset', pagination.offset.toString());

      const { data, error: fetchError } = await supabase.functions.invoke('crm-search-customers', {
        body: { ...Object.fromEntries(params) },
      });

      if (fetchError) throw fetchError;

      setCustomers(data.data || []);
      setPagination(prev => ({ ...prev, total: data.total || 0 }));
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load customers');
    } finally {
      setLoading(false);
    }
  };

  const handleSearch = (value: string) => {
    setSearchQuery(value);
    setPagination(prev => ({ ...prev, offset: 0 }));
  };

  const handleFilterChange = (key: string, value: string) => {
    setFilters(prev => ({ ...prev, [key]: value }));
    setPagination(prev => ({ ...prev, offset: 0 }));
  };

  return (
    <div className="container mx-auto py-6 space-y-6">
      <div className="flex items-center justify-between">
        <h1 className="text-3xl font-bold">Customers</h1>
        <Button onClick={() => router.push('/crm/customers/new')}>
          <Plus className="mr-2 h-4 w-4" />
          New Customer
        </Button>
      </div>

      {/* Search and Filters */}
      <Card>
        <CardHeader>
          <CardTitle>Search & Filter</CardTitle>
        </CardHeader>
        <CardContent className="space-y-4">
          <div className="flex gap-4">
            <div className="flex-1 relative">
              <Search className="absolute left-3 top-1/2 transform -translate-y-1/2 h-4 w-4 text-muted-foreground" />
              <Input
                placeholder="Search by name, email, or phone..."
                value={searchQuery}
                onChange={(e) => handleSearch(e.target.value)}
                className="pl-10"
              />
            </div>
            <Select value={filters.status} onValueChange={(value) => handleFilterChange('status', value)}>
              <SelectTrigger className="w-[180px]">
                <SelectValue placeholder="Status" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="">All Statuses</SelectItem>
                <SelectItem value="active">Active</SelectItem>
                <SelectItem value="prospect">Prospect</SelectItem>
                <SelectItem value="inactive">Inactive</SelectItem>
                <SelectItem value="blacklisted">Blacklisted</SelectItem>
              </SelectContent>
            </Select>
            <Select value={filters.lifecycle_stage} onValueChange={(value) => handleFilterChange('lifecycle_stage', value)}>
              <SelectTrigger className="w-[180px]">
                <SelectValue placeholder="Lifecycle Stage" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="">All Stages</SelectItem>
                <SelectItem value="lead">Lead</SelectItem>
                <SelectItem value="opportunity">Opportunity</SelectItem>
                <SelectItem value="customer">Customer</SelectItem>
                <SelectItem value="former_customer">Former Customer</SelectItem>
              </SelectContent>
            </Select>
          </div>
        </CardContent>
      </Card>

      {/* Customers Table */}
      <Card>
        <CardHeader>
          <CardTitle>Customers ({pagination.total})</CardTitle>
        </CardHeader>
        <CardContent>
          {loading ? (
            <div className="text-center py-8">Loading...</div>
          ) : error ? (
            <div className="text-center py-8 text-destructive">{error}</div>
          ) : customers.length === 0 ? (
            <div className="text-center py-8 text-muted-foreground">No customers found</div>
          ) : (
            <>
              <Table>
                <TableHeader>
                  <TableRow>
                    <TableHead>Customer</TableHead>
                    <TableHead>Type</TableHead>
                    <TableHead>Status</TableHead>
                    <TableHead>Lifecycle Stage</TableHead>
                    <TableHead>Contact</TableHead>
                    <TableHead>Tags</TableHead>
                    <TableHead>Actions</TableHead>
                  </TableRow>
                </TableHeader>
                <TableBody>
                  {customers.map((customer) => (
                    <TableRow
                      key={customer.id}
                      className="cursor-pointer"
                      onClick={() => router.push(`/crm/customers/${customer.id}`)}
                    >
                      <TableCell>
                        <div className="flex items-center gap-3">
                          <CustomerAvatar customer={customer} />
                          <div>
                            <div className="font-medium">{customer.name}</div>
                            {customer.company_name && (
                              <div className="text-sm text-muted-foreground">{customer.company_name}</div>
                            )}
                          </div>
                        </div>
                      </TableCell>
                      <TableCell>
                        <Badge variant={customer.type === 'company' ? 'default' : 'secondary'}>
                          {customer.type}
                        </Badge>
                      </TableCell>
                      <TableCell>
                        <StatusBadge status={customer.status} />
                      </TableCell>
                      <TableCell>
                        <Badge variant="outline">{customer.lifecycle_stage}</Badge>
                      </TableCell>
                      <TableCell>
                        <div className="text-sm">
                          {customer.email && <div>{customer.email}</div>}
                          {customer.phone && <div className="text-muted-foreground">{customer.phone}</div>}
                        </div>
                      </TableCell>
                      <TableCell>
                        <div className="flex gap-1 flex-wrap">
                          {customer.tags?.map((tag) => (
                            <TagChip key={tag.id} tag={tag} />
                          ))}
                        </div>
                      </TableCell>
                      <TableCell>
                        <Button
                          variant="ghost"
                          size="sm"
                          onClick={(e) => {
                            e.stopPropagation();
                            router.push(`/crm/customers/${customer.id}/edit`);
                          }}
                        >
                          Edit
                        </Button>
                      </TableCell>
                    </TableRow>
                  ))}
                </TableBody>
              </Table>
              
              {/* Pagination */}
              <div className="flex items-center justify-between mt-4">
                <div className="text-sm text-muted-foreground">
                  Showing {pagination.offset + 1} to {Math.min(pagination.offset + pagination.limit, pagination.total)} of {pagination.total}
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
                    disabled={pagination.offset + pagination.limit >= pagination.total}
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

### 3.3 API Integration Function

**lib/api/customers.ts**:

```typescript
import { createClient } from '@/lib/supabase/client';
import type { Customer, SearchCustomersParams, SearchCustomersResponse } from '@/types/customer';

export async function searchCustomers(params: SearchCustomersParams): Promise<SearchCustomersResponse> {
  const supabase = createClient();
  
  const queryParams = new URLSearchParams();
  if (params.q) queryParams.set('q', params.q);
  if (params.status) queryParams.set('status', params.status);
  if (params.lifecycle_stage) queryParams.set('lifecycle_stage', params.lifecycle_stage);
  if (params.tag) queryParams.set('tag', params.tag);
  if (params.segment_id) queryParams.set('segment_id', params.segment_id);
  if (params.limit) queryParams.set('limit', params.limit.toString());
  if (params.offset) queryParams.set('offset', params.offset.toString());
  
  const { data, error } = await supabase.rpc('crm_search_customers', {
    p_search_query: params.q || null,
    p_status: params.status ? [params.status] : null,
    p_lifecycle_stage: params.lifecycle_stage ? [params.lifecycle_stage] : null,
    p_tag_ids: params.tag ? [params.tag] : null,
    p_segment_id: params.segment_id || null,
    p_limit: params.limit || 20,
    p_offset: params.offset || 0,
    p_sort_by: params.sort_by || 'name',
    p_sort_order: params.sort_order || 'asc',
  });
  
  if (error) throw error;
  return data;
}
```

---

## 4. Story CRM-037: Customer Detail Page

### 4.1 Page Specification

**File**: `app/(crm)/customers/[id]/page.tsx`

**Route**: `/crm/customers/[id]`

**Authentication**: Required

### 4.2 Component Structure

```typescript
'use client';

import { useState, useEffect } from 'react';
import { useParams, useRouter } from 'next/navigation';
import { createClient } from '@/lib/supabase/client';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { CustomerAvatar } from '@/components/crm/customers/customer-avatar';
import { TagChip } from '@/components/crm/shared/tag-chip';
import { StatusBadge } from '@/components/crm/shared/status-badge';
import { InteractionTimelineItem } from '@/components/crm/shared/interaction-timeline-item';
import { FollowUpCard } from '@/components/crm/shared/followup-card';
import { Edit, Mail, Phone, MapPin } from 'lucide-react';
import type { CustomerDetails } from '@/types/customer';

export default function CustomerDetailPage() {
  const params = useParams();
  const router = useRouter();
  const supabase = createClient();
  
  const [customer, setCustomer] = useState<CustomerDetails | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (params.id) {
      loadCustomerDetails(params.id as string);
    }
  }, [params.id]);

  const loadCustomerDetails = async (customerId: string) => {
    try {
      setLoading(true);
      setError(null);

      const { data, error: fetchError } = await supabase.rpc('crm_get_customer_details', {
        p_customer_id: customerId,
        p_interactions_limit: 10,
        p_interactions_offset: 0,
        p_followups_limit: 10,
        p_followups_offset: 0,
      });

      if (fetchError) throw fetchError;
      setCustomer(data);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load customer');
    } finally {
      setLoading(false);
    }
  };

  if (loading) {
    return <div className="container mx-auto py-6">Loading...</div>;
  }

  if (error || !customer) {
    return <div className="container mx-auto py-6 text-destructive">{error || 'Customer not found'}</div>;
  }

  return (
    <div className="container mx-auto py-6 space-y-6">
      {/* Header */}
      <Card>
        <CardHeader>
          <div className="flex items-start justify-between">
            <div className="flex items-center gap-4">
              <CustomerAvatar customer={customer} size="lg" />
              <div>
                <CardTitle className="text-2xl">{customer.name}</CardTitle>
                <div className="flex items-center gap-2 mt-2">
                  <StatusBadge status={customer.status} />
                  <Badge variant="outline">{customer.lifecycle_stage}</Badge>
                  {customer.tags?.map((tag) => (
                    <TagChip key={tag.id} tag={tag} />
                  ))}
                </div>
              </div>
            </div>
            <Button onClick={() => router.push(`/crm/customers/${customer.id}/edit`)}>
              <Edit className="mr-2 h-4 w-4" />
              Edit
            </Button>
          </div>
        </CardHeader>
        <CardContent>
          <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
            {customer.email && (
              <div className="flex items-center gap-2">
                <Mail className="h-4 w-4 text-muted-foreground" />
                <span className="text-sm">{customer.email}</span>
              </div>
            )}
            {customer.phone && (
              <div className="flex items-center gap-2">
                <Phone className="h-4 w-4 text-muted-foreground" />
                <span className="text-sm">{customer.phone}</span>
              </div>
            )}
            {customer.primary_location && (
              <div className="flex items-center gap-2">
                <MapPin className="h-4 w-4 text-muted-foreground" />
                <span className="text-sm">{customer.primary_location.city}, {customer.primary_location.state}</span>
              </div>
            )}
          </div>
        </CardContent>
      </Card>

      {/* Tabs */}
      <Tabs defaultValue="overview" className="space-y-4">
        <TabsList>
          <TabsTrigger value="overview">Overview</TabsTrigger>
          <TabsTrigger value="activity">Activity</TabsTrigger>
          <TabsTrigger value="segments">Segments</TabsTrigger>
          <TabsTrigger value="notes">Notes</TabsTrigger>
        </TabsList>

        <TabsContent value="overview" className="space-y-4">
          {/* Locations */}
          <Card>
            <CardHeader>
              <CardTitle>Locations</CardTitle>
            </CardHeader>
            <CardContent>
              {customer.locations && customer.locations.length > 0 ? (
                <div className="space-y-2">
                  {customer.locations.map((location) => (
                    <div key={location.id} className="p-3 border rounded">
                      <div className="font-medium">{location.label || 'Primary Location'}</div>
                      <div className="text-sm text-muted-foreground">
                        {location.address_line1}
                        {location.address_line2 && `, ${location.address_line2}`}
                        <br />
                        {location.city}, {location.state} {location.postal_code}
                      </div>
                      <Badge variant="outline" className="mt-1">{location.type}</Badge>
                    </div>
                  ))}
                </div>
              ) : (
                <div className="text-sm text-muted-foreground">No locations</div>
              )}
            </CardContent>
          </Card>

          {/* Contacts */}
          <Card>
            <CardHeader>
              <CardTitle>Contacts</CardTitle>
            </CardHeader>
            <CardContent>
              {customer.contacts && customer.contacts.length > 0 ? (
                <div className="space-y-2">
                  {customer.contacts.map((contact) => (
                    <div key={contact.id} className="flex items-center justify-between p-3 border rounded">
                      <div>
                        <div className="font-medium">{contact.type}</div>
                        <div className="text-sm text-muted-foreground">{contact.value}</div>
                      </div>
                      <div className="flex gap-2">
                        {contact.is_primary && <Badge>Primary</Badge>}
                        {contact.is_verified && <Badge variant="outline">Verified</Badge>}
                      </div>
                    </div>
                  ))}
                </div>
              ) : (
                <div className="text-sm text-muted-foreground">No contacts</div>
              )}
            </CardContent>
          </Card>

          {/* Preferences */}
          {customer.preferences && (
            <Card>
              <CardHeader>
                <CardTitle>Communication Preferences</CardTitle>
              </CardHeader>
              <CardContent>
                <div className="space-y-2">
                  <div className="flex items-center justify-between">
                    <span>Do Not Contact</span>
                    <Badge variant={customer.preferences.do_not_contact ? 'destructive' : 'outline'}>
                      {customer.preferences.do_not_contact ? 'Yes' : 'No'}
                    </Badge>
                  </div>
                  <div className="flex items-center justify-between">
                    <span>Do Not Email</span>
                    <Badge variant={customer.preferences.do_not_email ? 'destructive' : 'outline'}>
                      {customer.preferences.do_not_email ? 'Yes' : 'No'}
                    </Badge>
                  </div>
                  <div className="flex items-center justify-between">
                    <span>Do Not SMS</span>
                    <Badge variant={customer.preferences.do_not_sms ? 'destructive' : 'outline'}>
                      {customer.preferences.do_not_sms ? 'Yes' : 'No'}
                    </Badge>
                  </div>
                  <div className="flex items-center justify-between">
                    <span>Do Not Call</span>
                    <Badge variant={customer.preferences.do_not_call ? 'destructive' : 'outline'}>
                      {customer.preferences.do_not_call ? 'Yes' : 'No'}
                    </Badge>
                  </div>
                </div>
              </CardContent>
            </Card>
          )}
        </TabsContent>

        <TabsContent value="activity" className="space-y-4">
          {/* Interactions */}
          <Card>
            <CardHeader>
              <CardTitle>Recent Interactions ({customer.interactions.total})</CardTitle>
            </CardHeader>
            <CardContent>
              {customer.interactions.data && customer.interactions.data.length > 0 ? (
                <div className="space-y-2">
                  {customer.interactions.data.map((interaction) => (
                    <InteractionTimelineItem key={interaction.id} interaction={interaction} />
                  ))}
                </div>
              ) : (
                <div className="text-sm text-muted-foreground">No interactions</div>
              )}
            </CardContent>
          </Card>

          {/* Follow-ups */}
          <Card>
            <CardHeader>
              <CardTitle>Upcoming Follow-ups ({customer.followups.total})</CardTitle>
            </CardHeader>
            <CardContent>
              {customer.followups.data && customer.followups.data.length > 0 ? (
                <div className="space-y-2">
                  {customer.followups.data.map((followup) => (
                    <FollowUpCard key={followup.id} followup={followup} />
                  ))}
                </div>
              ) : (
                <div className="text-sm text-muted-foreground">No follow-ups</div>
              )}
            </CardContent>
          </Card>
        </TabsContent>

        <TabsContent value="segments">
          <Card>
            <CardHeader>
              <CardTitle>Segment Membership</CardTitle>
            </CardHeader>
            <CardContent>
              {customer.segments && customer.segments.length > 0 ? (
                <div className="space-y-2">
                  {customer.segments.map((segment) => (
                    <div key={segment.id} className="p-3 border rounded">
                      <div className="font-medium">{segment.name}</div>
                      {segment.score && (
                        <div className="text-sm text-muted-foreground">Score: {segment.score}</div>
                      )}
                    </div>
                  ))}
                </div>
              ) : (
                <div className="text-sm text-muted-foreground">Not in any segments</div>
              )}
            </CardContent>
          </Card>
        </TabsContent>

        <TabsContent value="notes">
          <Card>
            <CardHeader>
              <CardTitle>Notes</CardTitle>
            </CardHeader>
            <CardContent>
              <div className="text-sm whitespace-pre-wrap">{customer.notes || 'No notes'}</div>
            </CardContent>
          </Card>
        </TabsContent>
      </Tabs>
    </div>
  );
}
```

---

## 5. Story CRM-038: Follow-Ups Dashboard Page

### 5.1 Page Specification

**File**: `app/(crm)/followups/page.tsx`

**Route**: `/crm/followups`

**Authentication**: Required

### 5.2 Component Structure

```typescript
'use client';

import { useState, useEffect } from 'react';
import { useRouter } from 'next/navigation';
import { createClient } from '@/lib/supabase/client';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { FollowUpCard } from '@/components/crm/shared/followup-card';
import { PriorityBadge } from '@/components/crm/shared/priority-badge';
import { StatusBadge } from '@/components/crm/shared/status-badge';
import { Calendar, CheckCircle2, XCircle, Clock } from 'lucide-react';
import type { FollowUp } from '@/types/followup';

export default function FollowUpsPage() {
  const router = useRouter();
  const supabase = createClient();
  
  const [followups, setFollowups] = useState<FollowUp[]>([]);
  const [loading, setLoading] = useState(true);
  const [filters, setFilters] = useState({
    assigned_to_user_id: '',
    status: 'pending',
    priority: '',
    start_date: '',
    end_date: '',
    overdue_only: false,
  });
  const [pagination, setPagination] = useState({ limit: 20, offset: 0, total: 0 });

  useEffect(() => {
    loadFollowups();
  }, [filters, pagination.offset]);

  const loadFollowups = async () => {
    try {
      setLoading(true);
      const { data, error } = await supabase.rpc('crm_list_followups', {
        p_org_id: (await supabase.auth.getUser()).data.user?.id, // Will be resolved from profile
        p_assigned_to_user_id: filters.assigned_to_user_id || null,
        p_statuses: filters.status ? [filters.status] : null,
        p_priorities: filters.priority ? [filters.priority] : null,
        p_start_date: filters.start_date || null,
        p_end_date: filters.end_date || null,
        p_overdue_only: filters.overdue_only,
        p_limit: pagination.limit,
        p_offset: pagination.offset,
        p_sort_by: 'due_at',
        p_sort_order: 'asc',
      });

      if (error) throw error;
      setFollowups(data.data || []);
      setPagination(prev => ({ ...prev, total: data.total || 0 }));
    } catch (err) {
      console.error('Failed to load follow-ups:', err);
    } finally {
      setLoading(false);
    }
  };

  const handleComplete = async (followupId: string) => {
    try {
      const { error } = await supabase.rpc('crm_update_followup', {
        p_followup_id: followupId,
        p_org_id: '', // Will be resolved
        p_status: 'completed',
        p_completion_notes: 'Completed via dashboard',
      });

      if (error) throw error;
      loadFollowups();
    } catch (err) {
      console.error('Failed to complete follow-up:', err);
    }
  };

  return (
    <div className="container mx-auto py-6 space-y-6">
      <div className="flex items-center justify-between">
        <h1 className="text-3xl font-bold">Follow-Ups</h1>
      </div>

      {/* Filters */}
      <Card>
        <CardHeader>
          <CardTitle>Filters</CardTitle>
        </CardHeader>
        <CardContent className="space-y-4">
          <div className="grid grid-cols-1 md:grid-cols-4 gap-4">
            <Select value={filters.status} onValueChange={(value) => setFilters(prev => ({ ...prev, status: value }))}>
              <SelectTrigger>
                <SelectValue placeholder="Status" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="pending">Pending</SelectItem>
                <SelectItem value="completed">Completed</SelectItem>
                <SelectItem value="canceled">Canceled</SelectItem>
                <SelectItem value="expired">Expired</SelectItem>
              </SelectContent>
            </Select>
            <Select value={filters.priority} onValueChange={(value) => setFilters(prev => ({ ...prev, priority: value }))}>
              <SelectTrigger>
                <SelectValue placeholder="Priority" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="high">High</SelectItem>
                <SelectItem value="medium">Medium</SelectItem>
                <SelectItem value="low">Low</SelectItem>
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

      {/* Follow-ups List */}
      <div className="grid gap-4">
        {loading ? (
          <div className="text-center py-8">Loading...</div>
        ) : followups.length === 0 ? (
          <div className="text-center py-8 text-muted-foreground">No follow-ups found</div>
        ) : (
          followups.map((followup) => (
            <FollowUpCard
              key={followup.id}
              followup={followup}
              onComplete={() => handleComplete(followup.id)}
              onView={() => router.push(`/crm/customers/${followup.customer_id}`)}
            />
          ))
        )}
      </div>
    </div>
  );
}
```

---

## 6. Story CRM-039: Segments Management UI

### 6.1 Segments List Page

**File**: `app/(crm)/segments/page.tsx`

```typescript
'use client';

import { useState, useEffect } from 'react';
import { useRouter } from 'next/navigation';
import { createClient } from '@/lib/supabase/client';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table';
import { Plus, RefreshCw } from 'lucide-react';
import type { Segment } from '@/types/segment';

export default function SegmentsPage() {
  const router = useRouter();
  const supabase = createClient();
  const [segments, setSegments] = useState<Segment[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadSegments();
  }, []);

  const loadSegments = async () => {
    try {
      setLoading(true);
      const { data: { user } } = await supabase.auth.getUser();
      const { data: profile } = await supabase.from('profiles').select('org_id').eq('id', user?.id).single();
      
      const { data, error } = await supabase.rpc('crm_list_segments', {
        p_org_id: profile?.org_id,
      });

      if (error) throw error;
      setSegments(data || []);
    } catch (err) {
      console.error('Failed to load segments:', err);
    } finally {
      setLoading(false);
    }
  };

  const handleRecompute = async (segmentId: string) => {
    try {
      const { error } = await supabase.functions.invoke('crm-segments-recompute', {
        body: { segment_id: segmentId },
      });
      if (error) throw error;
      loadSegments();
    } catch (err) {
      console.error('Failed to recompute segment:', err);
    }
  };

  return (
    <div className="container mx-auto py-6 space-y-6">
      <div className="flex items-center justify-between">
        <h1 className="text-3xl font-bold">Segments</h1>
        <Button onClick={() => router.push('/crm/segments/new')}>
          <Plus className="mr-2 h-4 w-4" />
          New Segment
        </Button>
      </div>

      <Card>
        <CardHeader>
          <CardTitle>Segments</CardTitle>
        </CardHeader>
        <CardContent>
          <Table>
            <TableHeader>
              <TableRow>
                <TableHead>Name</TableHead>
                <TableHead>Type</TableHead>
                <TableHead>Members</TableHead>
                <TableHead>Status</TableHead>
                <TableHead>Last Computed</TableHead>
                <TableHead>Actions</TableHead>
              </TableRow>
            </TableHeader>
            <TableBody>
              {segments.map((segment) => (
                <TableRow key={segment.id}>
                  <TableCell className="font-medium">{segment.name}</TableCell>
                  <TableCell>
                    <Badge variant="outline">{segment.type}</Badge>
                  </TableCell>
                  <TableCell>{segment.member_count || 0}</TableCell>
                  <TableCell>
                    <Badge variant={segment.is_active ? 'default' : 'secondary'}>
                      {segment.is_active ? 'Active' : 'Inactive'}
                    </Badge>
                  </TableCell>
                  <TableCell>
                    {segment.last_computed_at
                      ? new Date(segment.last_computed_at).toLocaleDateString()
                      : 'Never'}
                  </TableCell>
                  <TableCell>
                    <div className="flex gap-2">
                      <Button
                        variant="ghost"
                        size="sm"
                        onClick={() => router.push(`/crm/segments/${segment.id}`)}
                      >
                        View
                      </Button>
                      {segment.type !== 'static' && (
                        <Button
                          variant="ghost"
                          size="sm"
                          onClick={() => handleRecompute(segment.id)}
                        >
                          <RefreshCw className="h-4 w-4" />
                        </Button>
                      )}
                    </div>
                  </TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </CardContent>
      </Card>
    </div>
  );
}
```

### 6.2 Segment Detail Page

**File**: `app/(crm)/segments/[id]/page.tsx`

```typescript
'use client';

import { useState, useEffect } from 'react';
import { useParams, useRouter } from 'next/navigation';
import { createClient } from '@/lib/supabase/client';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table';
import { RefreshCw } from 'lucide-react';
import type { SegmentDetails } from '@/types/segment';

export default function SegmentDetailPage() {
  const params = useParams();
  const router = useRouter();
  const supabase = createClient();
  const [segment, setSegment] = useState<SegmentDetails | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (params.id) {
      loadSegment(params.id as string);
    }
  }, [params.id]);

  const loadSegment = async (segmentId: string) => {
    try {
      setLoading(true);
      const { data: { user } } = await supabase.auth.getUser();
      const { data: profile } = await supabase.from('profiles').select('org_id').eq('id', user?.id).single();
      
      const { data: segmentData, error: segmentError } = await supabase
        .from('crm_segments')
        .select('*')
        .eq('id', segmentId)
        .eq('org_id', profile?.org_id)
        .single();

      if (segmentError) throw segmentError;

      const { data: membersData, error: membersError } = await supabase.rpc('crm_get_segment_members', {
        p_segment_id: segmentId,
        p_org_id: profile?.org_id,
        p_limit: 100,
        p_offset: 0,
      });

      if (membersError) throw membersError;

      setSegment({
        ...segmentData,
        members: membersData.data || [],
        member_count: membersData.total || 0,
      });
    } catch (err) {
      console.error('Failed to load segment:', err);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Loading...</div>;
  if (!segment) return <div>Segment not found</div>;

  return (
    <div className="container mx-auto py-6 space-y-6">
      <Card>
        <CardHeader>
          <div className="flex items-center justify-between">
            <div>
              <CardTitle>{segment.name}</CardTitle>
              <p className="text-sm text-muted-foreground mt-1">{segment.description}</p>
            </div>
            <div className="flex gap-2">
              <Badge variant="outline">{segment.type}</Badge>
              <Badge variant={segment.is_active ? 'default' : 'secondary'}>
                {segment.is_active ? 'Active' : 'Inactive'}
              </Badge>
              {segment.type !== 'static' && (
                <Button onClick={() => handleRecompute(segment.id)}>
                  <RefreshCw className="mr-2 h-4 w-4" />
                  Recompute
                </Button>
              )}
            </div>
          </div>
        </CardHeader>
      </Card>

      <Tabs defaultValue="members" className="space-y-4">
        <TabsList>
          <TabsTrigger value="members">Members ({segment.member_count})</TabsTrigger>
          <TabsTrigger value="definition">Definition</TabsTrigger>
        </TabsList>

        <TabsContent value="members">
          <Card>
            <CardHeader>
              <CardTitle>Segment Members</CardTitle>
            </CardHeader>
            <CardContent>
              <Table>
                <TableHeader>
                  <TableRow>
                    <TableHead>Customer</TableHead>
                    <TableHead>Status</TableHead>
                    <TableHead>Score</TableHead>
                  </TableRow>
                </TableHeader>
                <TableBody>
                  {segment.members?.map((member) => (
                    <TableRow
                      key={member.customer_id}
                      onClick={() => router.push(`/crm/customers/${member.customer_id}`)}
                      className="cursor-pointer"
                    >
                      <TableCell>{member.customer.name}</TableCell>
                      <TableCell>{member.customer.status}</TableCell>
                      <TableCell>{member.score || '-'}</TableCell>
                    </TableRow>
                  ))}
                </TableBody>
              </Table>
            </CardContent>
          </Card>
        </TabsContent>

        <TabsContent value="definition">
          <Card>
            <CardHeader>
              <CardTitle>Segment Definition</CardTitle>
            </CardHeader>
            <CardContent>
              {segment.type === 'rule_based' && segment.definition && (
                <pre className="bg-muted p-4 rounded overflow-auto">
                  {JSON.stringify(segment.definition, null, 2)}
                </pre>
              )}
              {segment.type === 'ai_generated' && segment.ai_prompt && (
                <div>
                  <div className="font-medium mb-2">AI Prompt:</div>
                  <div className="bg-muted p-4 rounded">{segment.ai_prompt}</div>
                  {segment.ai_explanation && (
                    <>
                      <div className="font-medium mb-2 mt-4">AI Explanation:</div>
                      <div className="bg-muted p-4 rounded">{segment.ai_explanation}</div>
                    </>
                  )}
                </div>
              )}
            </CardContent>
          </Card>
        </TabsContent>
      </Tabs>
    </div>
  );
}
```

---

## 7. Story CRM-040: Automation Rules Management UI

### 7.1 Automation Rules List Page

**File**: `app/(crm)/automation/page.tsx`

```typescript
'use client';

import { useState, useEffect } from 'react';
import { useRouter } from 'next/navigation';
import { createClient } from '@/lib/supabase/client';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { Switch } from '@/components/ui/switch';
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table';
import { Plus } from 'lucide-react';
import type { AutomationRule } from '@/types/automation';

export default function AutomationPage() {
  const router = useRouter();
  const supabase = createClient();
  const [rules, setRules] = useState<AutomationRule[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadRules();
  }, []);

  const loadRules = async () => {
    try {
      setLoading(true);
      const { data: { user } } = await supabase.auth.getUser();
      const { data: profile } = await supabase.from('profiles').select('org_id').eq('id', user?.id).single();
      
      const { data, error } = await supabase.rpc('crm_list_automation_rules', {
        p_org_id: profile?.org_id,
      });

      if (error) throw error;
      setRules(data || []);
    } catch (err) {
      console.error('Failed to load rules:', err);
    } finally {
      setLoading(false);
    }
  };

  const handleToggleEnabled = async (ruleId: string, enabled: boolean) => {
    try {
      const { error } = await supabase.rpc('crm_update_automation_rule', {
        p_rule_id: ruleId,
        p_org_id: '', // Will be resolved
        p_is_enabled: enabled,
      });

      if (error) throw error;
      loadRules();
    } catch (err) {
      console.error('Failed to update rule:', err);
    }
  };

  return (
    <div className="container mx-auto py-6 space-y-6">
      <div className="flex items-center justify-between">
        <h1 className="text-3xl font-bold">Automation Rules</h1>
        <Button onClick={() => router.push('/crm/automation/new')}>
          <Plus className="mr-2 h-4 w-4" />
          New Rule
        </Button>
      </div>

      <Card>
        <CardHeader>
          <CardTitle>Rules</CardTitle>
        </CardHeader>
        <CardContent>
          <Table>
            <TableHeader>
              <TableRow>
                <TableHead>Name</TableHead>
                <TableHead>Trigger Type</TableHead>
                <TableHead>Event Type</TableHead>
                <TableHead>Enabled</TableHead>
                <TableHead>Actions</TableHead>
              </TableRow>
            </TableHeader>
            <TableBody>
              {rules.map((rule) => (
                <TableRow key={rule.id}>
                  <TableCell className="font-medium">{rule.name}</TableCell>
                  <TableCell>
                    <Badge variant="outline">{rule.trigger_type}</Badge>
                  </TableCell>
                  <TableCell>{rule.event_type || '-'}</TableCell>
                  <TableCell>
                    <Switch
                      checked={rule.is_enabled}
                      onCheckedChange={(checked) => handleToggleEnabled(rule.id, checked)}
                    />
                  </TableCell>
                  <TableCell>
                    <Button
                      variant="ghost"
                      size="sm"
                      onClick={() => router.push(`/crm/automation/${rule.id}`)}
                    >
                      View
                    </Button>
                  </TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </CardContent>
      </Card>
    </div>
  );
}
```

---

## 8. Story CRM-041: Reusable CRM UI Components

### 8.1 Customer Avatar Component

**File**: `components/crm/customers/customer-avatar.tsx`

```typescript
import { Avatar, AvatarFallback } from '@/components/ui/avatar';
import { Badge } from '@/components/ui/badge';
import { cn } from '@/lib/utils';
import type { Customer } from '@/types/customer';

interface CustomerAvatarProps {
  customer: Customer;
  size?: 'sm' | 'md' | 'lg';
  showBadge?: boolean;
  className?: string;
}

export function CustomerAvatar({ customer, size = 'md', showBadge = true, className }: CustomerAvatarProps) {
  const initials = customer.type === 'individual'
    ? `${customer.first_name?.[0] || ''}${customer.last_name?.[0] || ''}`.toUpperCase() || customer.name[0].toUpperCase()
    : customer.company_name?.[0]?.toUpperCase() || customer.name[0].toUpperCase();

  const sizeClasses = {
    sm: 'h-8 w-8 text-xs',
    md: 'h-10 w-10 text-sm',
    lg: 'h-16 w-16 text-lg',
  };

  return (
    <div className={cn('relative inline-block', className)}>
      <Avatar className={sizeClasses[size]}>
        <AvatarFallback>{initials}</AvatarFallback>
      </Avatar>
      {showBadge && customer.status === 'active' && (
        <div className="absolute -bottom-1 -right-1 h-4 w-4 rounded-full bg-green-500 border-2 border-background" />
      )}
    </div>
  );
}
```

### 8.2 Tag Chip Component

**File**: `components/crm/shared/tag-chip.tsx`

```typescript
import { Badge } from '@/components/ui/badge';
import type { Tag } from '@/types/customer';

interface TagChipProps {
  tag: Tag;
  onRemove?: () => void;
}

export function TagChip({ tag, onRemove }: TagChipProps) {
  return (
    <Badge
      variant="secondary"
      style={tag.color ? { backgroundColor: tag.color, color: 'white' } : undefined}
      className="cursor-pointer"
    >
      {tag.name}
      {onRemove && (
        <span className="ml-1" onClick={(e) => { e.stopPropagation(); onRemove(); }}>
          ×
        </span>
      )}
    </Badge>
  );
}
```

### 8.3 Interaction Timeline Item Component

**File**: `components/crm/shared/interaction-timeline-item.tsx`

```typescript
import { Card, CardContent } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { format } from 'date-fns';
import { Mail, Phone, MessageSquare, FileText } from 'lucide-react';
import type { Interaction } from '@/types/interaction';

interface InteractionTimelineItemProps {
  interaction: Interaction;
}

const channelIcons = {
  email_inbound: Mail,
  email_outbound: Mail,
  phone_inbound: Phone,
  phone_outbound: Phone,
  sms_inbound: MessageSquare,
  sms_outbound: MessageSquare,
  portal_message: MessageSquare,
  note: FileText,
  in_person: MessageSquare,
};

export function InteractionTimelineItem({ interaction }: InteractionTimelineItemProps) {
  const Icon = channelIcons[interaction.channel as keyof typeof channelIcons] || FileText;
  const isInbound = interaction.direction === 'inbound';

  return (
    <Card>
      <CardContent className="p-4">
        <div className="flex items-start gap-4">
          <div className={`p-2 rounded ${isInbound ? 'bg-blue-100' : 'bg-green-100'}`}>
            <Icon className="h-4 w-4" />
          </div>
          <div className="flex-1">
            <div className="flex items-center gap-2 mb-1">
              <span className="font-medium">{interaction.channel.replace('_', ' ')}</span>
              <Badge variant="outline">{interaction.direction}</Badge>
              {interaction.sentiment && (
                <Badge variant={interaction.sentiment === 'negative' ? 'destructive' : 'default'}>
                  {interaction.sentiment}
                </Badge>
              )}
            </div>
            {interaction.subject && <div className="font-medium mb-1">{interaction.subject}</div>}
            {interaction.summary && <div className="text-sm text-muted-foreground">{interaction.summary}</div>}
            <div className="text-xs text-muted-foreground mt-2">
              {format(new Date(interaction.occurred_at), 'MMM d, yyyy h:mm a')}
            </div>
          </div>
        </div>
      </CardContent>
    </Card>
  );
}
```

### 8.4 Follow-Up Card Component

**File**: `components/crm/shared/followup-card.tsx`

```typescript
import { Card, CardContent, CardHeader } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { PriorityBadge } from './priority-badge';
import { format, isPast } from 'date-fns';
import { CheckCircle2, Clock, AlertCircle } from 'lucide-react';
import type { FollowUp } from '@/types/followup';

interface FollowUpCardProps {
  followup: FollowUp;
  onComplete?: () => void;
  onView?: () => void;
}

export function FollowUpCard({ followup, onComplete, onView }: FollowUpCardProps) {
  const isOverdue = isPast(new Date(followup.due_at)) && followup.status === 'pending';
  const isDueSoon = !isOverdue && new Date(followup.due_at).getTime() - Date.now() < 24 * 60 * 60 * 1000;

  return (
    <Card className={isOverdue ? 'border-destructive' : ''}>
      <CardHeader>
        <div className="flex items-start justify-between">
          <div className="flex-1">
            <div className="flex items-center gap-2 mb-2">
              <h3 className="font-semibold">{followup.title}</h3>
              <PriorityBadge priority={followup.priority} />
              {isOverdue && <Badge variant="destructive">Overdue</Badge>}
              {isDueSoon && <Badge variant="warning">Due Soon</Badge>}
            </div>
            {followup.description && (
              <p className="text-sm text-muted-foreground">{followup.description}</p>
            )}
          </div>
        </div>
      </CardHeader>
      <CardContent>
        <div className="flex items-center justify-between">
          <div className="flex items-center gap-4 text-sm text-muted-foreground">
            <div className="flex items-center gap-1">
              <Clock className="h-4 w-4" />
              {format(new Date(followup.due_at), 'MMM d, yyyy h:mm a')}
            </div>
            {followup.customer && (
              <div className="font-medium text-foreground">{followup.customer.name}</div>
            )}
          </div>
          <div className="flex gap-2">
            {onView && (
              <Button variant="outline" size="sm" onClick={onView}>
                View Customer
              </Button>
            )}
            {onComplete && followup.status === 'pending' && (
              <Button size="sm" onClick={onComplete}>
                <CheckCircle2 className="mr-2 h-4 w-4" />
                Complete
              </Button>
            )}
          </div>
        </div>
      </CardContent>
    </Card>
  );
}
```

### 8.5 Status Badge Component

**File**: `components/crm/shared/status-badge.tsx`

```typescript
import { Badge } from '@/components/ui/badge';
import { cn } from '@/lib/utils';

interface StatusBadgeProps {
  status: string;
  className?: string;
}

const statusVariants: Record<string, 'default' | 'secondary' | 'destructive' | 'outline'> = {
  active: 'default',
  prospect: 'secondary',
  inactive: 'outline',
  blacklisted: 'destructive',
  pending: 'secondary',
  completed: 'default',
  canceled: 'outline',
  expired: 'destructive',
};

export function StatusBadge({ status, className }: StatusBadgeProps) {
  return (
    <Badge variant={statusVariants[status] || 'outline'} className={className}>
      {status}
    </Badge>
  );
}
```

### 8.6 Priority Badge Component

**File**: `components/crm/shared/priority-badge.tsx`

```typescript
import { Badge } from '@/components/ui/badge';

interface PriorityBadgeProps {
  priority: 'low' | 'medium' | 'high';
}

const priorityVariants: Record<string, 'default' | 'secondary' | 'destructive'> = {
  high: 'destructive',
  medium: 'default',
  low: 'secondary',
};

export function PriorityBadge({ priority }: PriorityBadgeProps) {
  return (
    <Badge variant={priorityVariants[priority] || 'secondary'}>
      {priority}
    </Badge>
  );
}
```

### 8.7 Tag Selector Component

**File**: `components/crm/shared/tag-selector.tsx`

```typescript
'use client';

import { useState, useEffect } from 'react';
import { createClient } from '@/lib/supabase/client';
import { Button } from '@/components/ui/button';
import {
  Command,
  CommandEmpty,
  CommandGroup,
  CommandInput,
  CommandItem,
  CommandList,
} from '@/components/ui/command';
import {
  Popover,
  PopoverContent,
  PopoverTrigger,
} from '@/components/ui/popover';
import { TagChip } from './tag-chip';
import { Check, Plus } from 'lucide-react';
import { cn } from '@/lib/utils';
import type { Tag } from '@/types/customer';

interface TagSelectorProps {
  selectedTags: Tag[];
  onTagsChange: (tags: Tag[]) => void;
  orgId: string;
}

export function TagSelector({ selectedTags, onTagsChange, orgId }: TagSelectorProps) {
  const [open, setOpen] = useState(false);
  const [availableTags, setAvailableTags] = useState<Tag[]>([]);
  const supabase = createClient();

  useEffect(() => {
    loadTags();
  }, [orgId]);

  const loadTags = async () => {
    const { data } = await supabase
      .from('crm_tags')
      .select('*')
      .eq('org_id', orgId)
      .order('name');
    
    if (data) setAvailableTags(data);
  };

  const toggleTag = (tag: Tag) => {
    const isSelected = selectedTags.some(t => t.id === tag.id);
    if (isSelected) {
      onTagsChange(selectedTags.filter(t => t.id !== tag.id));
    } else {
      onTagsChange([...selectedTags, tag]);
    }
  };

  return (
    <div className="space-y-2">
      <div className="flex flex-wrap gap-2">
        {selectedTags.map((tag) => (
          <TagChip
            key={tag.id}
            tag={tag}
            onRemove={() => toggleTag(tag)}
          />
        ))}
      </div>
      <Popover open={open} onOpenChange={setOpen}>
        <PopoverTrigger asChild>
          <Button variant="outline" size="sm">
            <Plus className="mr-2 h-4 w-4" />
            Add Tag
          </Button>
        </PopoverTrigger>
        <PopoverContent className="w-[300px] p-0">
          <Command>
            <CommandInput placeholder="Search tags..." />
            <CommandList>
              <CommandEmpty>No tags found.</CommandEmpty>
              <CommandGroup>
                {availableTags.map((tag) => {
                  const isSelected = selectedTags.some(t => t.id === tag.id);
                  return (
                    <CommandItem
                      key={tag.id}
                      onSelect={() => toggleTag(tag)}
                    >
                      <Check
                        className={cn(
                          'mr-2 h-4 w-4',
                          isSelected ? 'opacity-100' : 'opacity-0'
                        )}
                      />
                      <TagChip tag={tag} />
                    </CommandItem>
                  );
                })}
              </CommandGroup>
            </CommandList>
          </Command>
        </PopoverContent>
      </Popover>
    </div>
  );
}
```

---

## 9. Type Definitions

### 9.1 Customer Types

**File**: `types/customer.ts`

```typescript
export interface Customer {
  id: string;
  org_id: string;
  type: 'individual' | 'company';
  name: string;
  first_name?: string;
  last_name?: string;
  company_name?: string;
  email?: string;
  phone?: string;
  status: 'active' | 'prospect' | 'inactive' | 'blacklisted';
  lifecycle_stage: 'lead' | 'opportunity' | 'customer' | 'former_customer';
  tags?: Tag[];
  created_at: string;
  updated_at: string;
}

export interface CustomerDetails extends Customer {
  primary_location?: Location;
  locations: Location[];
  contacts: Contact[];
  preferences?: Preferences;
  interactions: {
    data: Interaction[];
    total: number;
    limit: number;
    offset: number;
  };
  followups: {
    data: FollowUp[];
    total: number;
    limit: number;
    offset: number;
  };
  segments?: SegmentMembership[];
}

export interface Tag {
  id: string;
  name: string;
  color?: string;
}

export interface Location {
  id: string;
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
  is_primary: boolean;
}

export interface Contact {
  id: string;
  type: string;
  value: string;
  is_primary: boolean;
  is_verified: boolean;
}

export interface Preferences {
  do_not_contact: boolean;
  do_not_email: boolean;
  do_not_sms: boolean;
  do_not_call: boolean;
  preferred_contact_window_start?: string;
  preferred_contact_window_end?: string;
}

export interface SearchCustomersParams {
  q?: string;
  status?: string;
  lifecycle_stage?: string;
  tag?: string;
  segment_id?: string;
  limit?: number;
  offset?: number;
  sort_by?: string;
  sort_order?: 'asc' | 'desc';
}

export interface SearchCustomersResponse {
  data: Customer[];
  total: number;
  limit: number;
  offset: number;
}
```

### 9.2 Follow-Up Types

**File**: `types/followup.ts`

```typescript
export interface FollowUp {
  id: string;
  customer_id: string;
  customer?: {
    id: string;
    name: string;
  };
  assigned_to_user_id?: string;
  assigned_to_user?: {
    id: string;
    first_name?: string;
    last_name?: string;
    email?: string;
  };
  title: string;
  description?: string;
  due_at: string;
  status: 'pending' | 'completed' | 'canceled' | 'expired';
  priority: 'low' | 'medium' | 'high';
  origin: 'manual' | 'system_rule' | 'ai_recommendation';
  completed_at?: string;
  completion_notes?: string;
  created_at: string;
  updated_at: string;
  is_overdue?: boolean;
}
```

### 9.3 Segment Types

**File**: `types/segment.ts`

```typescript
export interface Segment {
  id: string;
  org_id: string;
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
  last_computed_at?: string;
  member_count?: number;
  created_at: string;
  updated_at: string;
}

export interface SegmentDetails extends Segment {
  members: SegmentMember[];
}

export interface SegmentMember {
  customer_id: string;
  customer: {
    id: string;
    name: string;
    status: string;
    lifecycle_stage: string;
  };
  score?: number;
  metadata?: any;
  created_at: string;
}
```

### 9.4 Automation Types

**File**: `types/automation.ts`

```typescript
export interface AutomationRule {
  id: string;
  org_id: string;
  name: string;
  description?: string;
  trigger_type: 'event' | 'time_based' | 'segment_membership';
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
  actions: Array<{
    type: string;
    [key: string]: any;
  }>;
  is_enabled: boolean;
  created_at: string;
  updated_at: string;
}

export interface AutomationRun {
  id: string;
  rule_id: string;
  customer_id?: string;
  status: 'pending' | 'success' | 'failed' | 'skipped';
  error_message?: string;
  started_at: string;
  completed_at?: string;
}
```

### 9.5 Interaction Types

**File**: `types/interaction.ts`

```typescript
export interface Interaction {
  id: string;
  customer_id: string;
  channel: string;
  direction?: string;
  subject?: string;
  summary?: string;
  body?: string;
  sentiment?: 'positive' | 'neutral' | 'negative';
  occurred_at: string;
  created_at: string;
}
```

---

## 10. Layout & Navigation

### 10.1 CRM Layout

**File**: `app/(crm)/layout.tsx`

```typescript
import { Sidebar } from '@/components/crm/layout/sidebar';
import { Header } from '@/components/crm/layout/header';

export default function CRMLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex h-screen">
      <Sidebar />
      <div className="flex-1 flex flex-col overflow-hidden">
        <Header />
        <main className="flex-1 overflow-y-auto bg-background">
          {children}
        </main>
      </div>
    </div>
  );
}
```

### 10.2 Sidebar Component

**File**: `components/crm/layout/sidebar.tsx`

```typescript
'use client';

import { usePathname } from 'next/navigation';
import Link from 'next/link';
import { cn } from '@/lib/utils';
import { Users, Calendar, Layers, Zap, Home } from 'lucide-react';

const navItems = [
  { href: '/crm/customers', label: 'Customers', icon: Users },
  { href: '/crm/followups', label: 'Follow-Ups', icon: Calendar },
  { href: '/crm/segments', label: 'Segments', icon: Layers },
  { href: '/crm/automation', label: 'Automation', icon: Zap },
];

export function Sidebar() {
  const pathname = usePathname();

  return (
    <aside className="w-64 border-r bg-card">
      <div className="p-4">
        <Link href="/crm" className="flex items-center gap-2 font-bold text-lg">
          <Home className="h-6 w-6" />
          CRM
        </Link>
      </div>
      <nav className="px-4 space-y-1">
        {navItems.map((item) => {
          const Icon = item.icon;
          const isActive = pathname.startsWith(item.href);
          return (
            <Link
              key={item.href}
              href={item.href}
              className={cn(
                'flex items-center gap-3 px-3 py-2 rounded-md transition-colors',
                isActive
                  ? 'bg-primary text-primary-foreground'
                  : 'hover:bg-muted'
              )}
            >
              <Icon className="h-5 w-5" />
              {item.label}
            </Link>
          );
        })}
      </nav>
    </aside>
  );
}
```

### 10.3 Header Component

**File**: `components/crm/layout/header.tsx`

```typescript
'use client';

import { useState, useEffect } from 'react';
import { createClient } from '@/lib/supabase/client';
import { Button } from '@/components/ui/button';
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu';
import { Avatar, AvatarFallback } from '@/components/ui/avatar';
import { LogOut, User } from 'lucide-react';

export function Header() {
  const supabase = createClient();
  const [user, setUser] = useState<any>(null);

  useEffect(() => {
    supabase.auth.getUser().then(({ data }) => {
      setUser(data.user);
    });
  }, []);

  const handleSignOut = async () => {
    await supabase.auth.signOut();
    window.location.href = '/login';
  };

  return (
    <header className="border-b bg-card">
      <div className="flex items-center justify-between px-6 py-4">
        <div className="flex items-center gap-4">
          {/* Breadcrumbs or page title can go here */}
        </div>
        <DropdownMenu>
          <DropdownMenuTrigger asChild>
            <Button variant="ghost" className="relative h-10 w-10 rounded-full">
              <Avatar>
                <AvatarFallback>
                  {user?.email?.[0]?.toUpperCase() || 'U'}
                </AvatarFallback>
              </Avatar>
            </Button>
          </DropdownMenuTrigger>
          <DropdownMenuContent align="end">
            <DropdownMenuItem>
              <User className="mr-2 h-4 w-4" />
              Profile
            </DropdownMenuItem>
            <DropdownMenuItem onClick={handleSignOut}>
              <LogOut className="mr-2 h-4 w-4" />
              Sign Out
            </DropdownMenuItem>
          </DropdownMenuContent>
        </DropdownMenu>
      </div>
    </header>
  );
}
```

---

## 11. API Integration Functions

### 11.1 Customers API

**File**: `lib/api/customers.ts`

```typescript
import { createClient } from '@/lib/supabase/client';
import type { Customer, CustomerDetails, SearchCustomersParams, SearchCustomersResponse } from '@/types/customer';

export async function searchCustomers(params: SearchCustomersParams): Promise<SearchCustomersResponse> {
  const supabase = createClient();
  const { data: { user } } = await supabase.auth.getUser();
  const { data: profile } = await supabase.from('profiles').select('org_id').eq('id', user?.id).single();
  
  const { data, error } = await supabase.rpc('crm_search_customers', {
    p_search_query: params.q || null,
    p_status: params.status ? [params.status] : null,
    p_lifecycle_stage: params.lifecycle_stage ? [params.lifecycle_stage] : null,
    p_tag_ids: params.tag ? [params.tag] : null,
    p_segment_id: params.segment_id || null,
    p_limit: params.limit || 20,
    p_offset: params.offset || 0,
    p_sort_by: params.sort_by || 'name',
    p_sort_order: params.sort_order || 'asc',
  });
  
  if (error) throw error;
  return data;
}

export async function getCustomerDetails(customerId: string): Promise<CustomerDetails> {
  const supabase = createClient();
  const { data: { user } } = await supabase.auth.getUser();
  const { data: profile } = await supabase.from('profiles').select('org_id').eq('id', user?.id).single();
  
  const { data, error } = await supabase.rpc('crm_get_customer_details', {
    p_customer_id: customerId,
    p_interactions_limit: 10,
    p_interactions_offset: 0,
    p_followups_limit: 10,
    p_followups_offset: 0,
  });
  
  if (error) throw error;
  return data;
}

export async function createCustomer(customerData: any): Promise<Customer> {
  const supabase = createClient();
  
  const { data, error } = await supabase.functions.invoke('crm-customers', {
    body: customerData,
  });
  
  if (error) throw error;
  return data;
}

export async function updateCustomer(customerId: string, updates: any): Promise<Customer> {
  const supabase = createClient();
  
  const { data, error } = await supabase.functions.invoke(`crm-customers/${customerId}`, {
    method: 'PATCH',
    body: updates,
  });
  
  if (error) throw error;
  return data;
}
```

### 11.2 Follow-Ups API

**File**: `lib/api/followups.ts`

```typescript
import { createClient } from '@/lib/supabase/client';
import type { FollowUp } from '@/types/followup';

export async function listFollowups(params: any): Promise<{ data: FollowUp[]; total: number }> {
  const supabase = createClient();
  const { data: { user } } = await supabase.auth.getUser();
  const { data: profile } = await supabase.from('profiles').select('org_id').eq('id', user?.id).single();
  
  const { data, error } = await supabase.rpc('crm_list_followups', {
    p_org_id: profile?.org_id,
    ...params,
  });
  
  if (error) throw error;
  return data;
}

export async function createFollowup(followupData: any): Promise<FollowUp> {
  const supabase = createClient();
  
  const { data, error } = await supabase.functions.invoke('crm-followups', {
    body: followupData,
  });
  
  if (error) throw error;
  return data;
}

export async function updateFollowup(followupId: string, updates: any): Promise<FollowUp> {
  const supabase = createClient();
  
  const { data, error } = await supabase.functions.invoke(`crm-followups/${followupId}`, {
    method: 'PATCH',
    body: updates,
  });
  
  if (error) throw error;
  return data;
}
```

---

## 12. Environment Variables

**.env.local**:

```bash
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```

---

## 13. Implementation Checklist

### Story CRM-036: Customers List Page
- [ ] Next.js page created (`/crm/customers`)
- [ ] Search bar implemented
- [ ] Filters implemented (status, lifecycle_stage, tags, segments)
- [ ] Table/card view with customer details
- [ ] Pagination implemented
- [ ] Loading states handled
- [ ] Error states handled
- [ ] Empty states handled
- [ ] Role-based access enforced
- [ ] shadcn/ui components integrated

### Story CRM-037: Customer Detail Page
- [ ] Next.js page created (`/crm/customers/[id]`)
- [ ] Summary header implemented
- [ ] Tabs implemented (Overview, Activity, Segments, Notes)
- [ ] Locations section
- [ ] Contacts section
- [ ] Preferences section
- [ ] Interactions timeline
- [ ] Follow-ups list
- [ ] Segments membership
- [ ] Inline editing (or edit flows)
- [ ] Role-based field visibility
- [ ] Performance optimized

### Story CRM-038: Follow-Ups Dashboard
- [ ] Next.js page created (`/crm/followups`)
- [ ] Filters implemented (assignee, status, priority, date range)
- [ ] Visual indicators (overdue, high-priority)
- [ ] Quick actions (complete, reschedule)
- [ ] Role-based visibility
- [ ] Loading and error states

### Story CRM-039: Segments Management UI
- [ ] Segments list page
- [ ] Segment detail page
- [ ] Rule editor (for rule_based)
- [ ] AI prompt editor (for ai_generated)
- [ ] Members list
- [ ] Recompute action
- [ ] Role-based access (admin/manager only)
- [ ] Error handling

### Story CRM-040: Automation Rules Management UI
- [ ] Automation rules list page
- [ ] Rule detail page
- [ ] Rule editor (trigger, conditions, actions)
- [ ] Enable/disable toggle
- [ ] Recent runs display
- [ ] Role-based access (admin/manager only)
- [ ] Error handling

### Story CRM-041: Reusable Components
- [ ] CustomerAvatar component
- [ ] TagChip component
- [ ] TagSelector component
- [ ] InteractionTimelineItem component
- [ ] FollowUpCard component
- [ ] StatusBadge component
- [ ] PriorityBadge component
- [ ] Components handle loading/error states
- [ ] Components documented

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 9 – Frontend (Next.js) CRM UI. All specifications are designed to be directly consumable by LLM-based code generators, with exact component structures, TypeScript interfaces, shadcn/ui component usage, API integration patterns, and routing configurations defined.

