# Technical Design Document – Epic 11: Dispatch Console UI (Next.js)

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 11 – Dispatch Console UI (Next.js)
- **Source**: Derived from `fdd_2_agile.md` Epic 11 (Stories DISP-051 through DISP-055)
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
  - `tdd_2_epic_10.md` (Dispatch Epic 10 for notifications)
- **Target Platform**: Next.js 14+ (App Router), React 18+, shadcn/ui, Supabase JS Client
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing the Dispatch Console UI using Next.js and shadcn/ui components. It covers:

- Schedule Board (Timeline View) with drag-and-drop functionality
- Map-Based Dispatch View with job pins and route overlays
- Job Creation UI and Detail Drawer
- Capacity & Utilization View per technician
- Optimization Actions UI with progress tracking

All components are built using Next.js 14+ App Router, React 18+, shadcn/ui component library, and Supabase JS Client for real-time data synchronization.

This epic assumes Epic 1-10 are complete and all backend APIs are available.

---

## 2. Prerequisites

### 2.1 Required Components

Before implementing Epic 11, ensure:

1. **Epic 1-10 Complete**: All backend epics are implemented
2. **Next.js Project Setup**:
   - Next.js 14+ with App Router
   - React 18+
   - TypeScript
   - Tailwind CSS
   - shadcn/ui installed and configured

3. **Supabase Client**:
   - `@supabase/supabase-js` installed
   - Supabase client configured with environment variables
   - Authentication setup complete

4. **Required shadcn/ui Components**:
   - `Button`
   - `Card`
   - `Dialog` / `Drawer` (from Radix UI)
   - `Select`
   - `Input`
   - `Label`
   - `Badge`
   - `Table`
   - `Tabs`
   - `Tooltip`
   - `Popover`
   - `Calendar`
   - `Skeleton`
   - `Alert`
   - `Toast` (via `sonner` or similar)

5. **Additional Dependencies**:
   - `@hello-pangea/dnd` for drag-and-drop (recommended, maintained fork of react-beautiful-dnd)
   - `react-map-gl` and `mapbox-gl` for map view (or `leaflet` as alternative)
   - `date-fns` for date formatting
   - `zod` for form validation
   - `react-hook-form` for form handling
   - `@tanstack/react-query` for data fetching and caching
   - `sonner` for toast notifications
   - `lucide-react` for icons

### 2.2 Project Structure

```
app/
  dispatch/
    schedule-board/
      page.tsx
      components/
        ScheduleBoard.tsx
        TechnicianRow.tsx
        AssignmentBlock.tsx
        TimeAxis.tsx
    map-view/
      page.tsx
      components/
        MapView.tsx
        JobMarker.tsx
        RouteOverlay.tsx
        MapFilters.tsx
    jobs/
      [id]/
        page.tsx
      components/
        JobDetailDrawer.tsx
        JobForm.tsx
        AssignmentList.tsx
    capacity/
      page.tsx
      components/
        CapacityView.tsx
        TechnicianCapacityCard.tsx
        UtilizationChart.tsx
    optimize/
      components/
        OptimizeActions.tsx
        OptimizationProgress.tsx
  components/
    ui/                    # shadcn/ui components
    dispatch/              # Shared dispatch components
      AssignmentCard.tsx
      JobCard.tsx
      StatusBadge.tsx
      PriorityBadge.tsx
lib/
  supabase/
    client.ts
    queries.ts
    mutations.ts
  hooks/
    useAssignments.ts
    useJobs.ts
    useRealtime.ts
  utils/
    date.ts
    formatting.ts
types/
  dispatch.ts
```

---

## 3. Common Patterns

### 3.1 Supabase Client Setup

**File**: `lib/supabase/client.ts`

```typescript
import { createClientComponentClient } from '@supabase/auth-helpers-nextjs';
import { createClient } from '@supabase/supabase-js';

// For client components
export const createSupabaseClient = () => {
  return createClientComponentClient();
};

// For server components and API routes
export const createSupabaseServerClient = () => {
  return createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
};
```

### 3.2 Type Definitions

**File**: `types/dispatch.ts`

```typescript
export interface DispatchJob {
  id: string;
  org_id: string;
  customer_id: string;
  location_id: string;
  title: string;
  description: string | null;
  job_type: string | null;
  priority: 'low' | 'normal' | 'high' | 'emergency';
  status: 'unscheduled' | 'scheduled' | 'dispatched' | 'in_progress' | 'completed' | 'canceled';
  estimated_duration_minutes: number;
  required_skills: string[] | null;
  required_crew_size: number;
  service_zone_id: string | null;
  sla_start_at: string | null;
  sla_end_at: string | null;
  is_customer_booked: boolean;
  notes_internal: string | null;
  created_at: string;
  updated_at: string;
  // Joined data
  customer?: {
    id: string;
    name: string;
    phone: string | null;
    email: string | null;
  };
  location?: {
    id: string;
    address_line1: string;
    city: string;
    state: string;
    postal_code: string;
    latitude: number | null;
    longitude: number | null;
  };
  time_windows?: JobTimeWindow[];
  assignments?: JobAssignment[];
}

export interface JobAssignment {
  id: string;
  org_id: string;
  dispatch_job_id: string;
  technician_id: string;
  scheduled_start_at: string;
  scheduled_end_at: string;
  arrival_window_start: string | null;
  arrival_window_end: string | null;
  status: 'assigned' | 'accepted' | 'declined' | 'en_route' | 'on_site' | 'completed' | 'no_show' | 'canceled';
  tech_eta_at: string | null;
  sequence_in_route: number | null;
  is_primary_technician: boolean;
  notes: string | null;
  created_at: string;
  updated_at: string;
  // Joined data
  technician?: {
    id: string;
    display_name: string;
    user_id: string;
  };
  job?: DispatchJob;
}

export interface JobTimeWindow {
  id: string;
  dispatch_job_id: string;
  window_start: string;
  window_end: string;
  source: 'system_suggested' | 'dispatcher_selected' | 'customer_selected';
  is_selected: boolean;
  created_at: string;
}

export interface TechnicianProfile {
  id: string;
  org_id: string;
  user_id: string;
  display_name: string;
  employment_type: string | null;
  is_active: boolean;
  max_daily_work_minutes: number | null;
  max_concurrent_jobs: number | null;
  vehicle_type: string | null;
  home_base_location_id: string | null;
  created_at: string;
  updated_at: string;
}

export interface RoutePlan {
  id: string;
  org_id: string;
  technician_id: string;
  date: string;
  status: 'draft' | 'finalized' | 'in_progress' | 'completed';
  optimization_strategy: string | null;
  optimization_metadata: any;
  created_at: string;
  updated_at: string;
  stops?: RouteStop[];
}

export interface RouteStop {
  id: string;
  route_plan_id: string;
  job_assignment_id: string | null;
  stop_type: 'job' | 'depot' | 'break';
  sequence: number;
  planned_arrival_at: string;
  planned_departure_at: string;
  actual_arrival_at: string | null;
  actual_departure_at: string | null;
  travel_time_minutes_from_prev: number | null;
  distance_km_from_prev: number | null;
}
```

### 3.3 React Query Hooks

**File**: `lib/hooks/useAssignments.ts`

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { createSupabaseClient } from '@/lib/supabase/client';
import type { JobAssignment } from '@/types/dispatch';

export function useAssignments(date: string, technicianId?: string) {
  const supabase = createSupabaseClient();

  return useQuery({
    queryKey: ['assignments', date, technicianId],
    queryFn: async () => {
      let query = supabase
        .from('job_assignments')
        .select(`
          *,
          dispatch_jobs!inner(
            *,
            customers(id, name),
            customer_locations(address_line1, city, state, postal_code, latitude, longitude)
          ),
          technician_profiles(id, display_name)
        `)
        .eq('org_id', (await supabase.auth.getUser()).data.user?.id) // Simplified, would use org_id helper
        .gte('scheduled_start_at', `${date}T00:00:00Z`)
        .lt('scheduled_start_at', `${date}T23:59:59Z`)
        .in('status', ['assigned', 'accepted', 'en_route', 'on_site']);

      if (technicianId) {
        query = query.eq('technician_id', technicianId);
      }

      const { data, error } = await query.order('scheduled_start_at', { ascending: true });

      if (error) throw error;
      return data as JobAssignment[];
    }
  });
}

export function useUpdateAssignment() {
  const supabase = createSupabaseClient();
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({
      assignmentId,
      updates
    }: {
      assignmentId: string;
      updates: Partial<JobAssignment>;
    }) => {
      const { data, error } = await supabase
        .from('job_assignments')
        .update(updates)
        .eq('id', assignmentId)
        .select()
        .single();

      if (error) throw error;
      return data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['assignments'] });
    }
  });
}
```

### 3.4 Real-Time Subscriptions

**File**: `lib/hooks/useRealtime.ts`

```typescript
import { useEffect } from 'react';
import { createSupabaseClient } from '@/lib/supabase/client';
import { useQueryClient } from '@tanstack/react-query';

export function useRealtimeAssignments(date: string) {
  const supabase = createSupabaseClient();
  const queryClient = useQueryClient();

  useEffect(() => {
    const channel = supabase
      .channel(`assignments-${date}`)
      .on(
        'postgres_changes',
        {
          event: '*',
          schema: 'public',
          table: 'job_assignments',
          filter: `scheduled_start_at=gte.${date}T00:00:00Z`
        },
        (payload) => {
          queryClient.invalidateQueries({ queryKey: ['assignments', date] });
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [date, supabase, queryClient]);
}
```

---

## 4. Story DISP-051: Dispatch Schedule Board (Timeline View)

### 4.1 Component Structure

**File**: `app/dispatch/schedule-board/page.tsx`

```typescript
'use client';

import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { createSupabaseClient } from '@/lib/supabase/client';
import { ScheduleBoard } from './components/ScheduleBoard';
import { Button } from '@/components/ui/button';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { Calendar } from '@/components/ui/calendar';
import { Popover, PopoverContent, PopoverTrigger } from '@/components/ui/popover';
import { format } from 'date-fns';
import { CalendarIcon } from 'lucide-react';
import { cn } from '@/lib/utils';

export default function ScheduleBoardPage() {
  const [selectedDate, setSelectedDate] = useState<Date>(new Date());
  const [selectedZone, setSelectedZone] = useState<string>('all');

  const dateString = format(selectedDate, 'yyyy-MM-dd');

  return (
    <div className="flex flex-col h-screen">
      <div className="border-b p-4 flex items-center justify-between">
        <h1 className="text-2xl font-bold">Schedule Board</h1>
        <div className="flex items-center gap-4">
          <Popover>
            <PopoverTrigger asChild>
              <Button
                variant="outline"
                className={cn(
                  'w-[280px] justify-start text-left font-normal',
                  !selectedDate && 'text-muted-foreground'
                )}
              >
                <CalendarIcon className="mr-2 h-4 w-4" />
                {selectedDate ? format(selectedDate, 'PPP') : <span>Pick a date</span>}
              </Button>
            </PopoverTrigger>
            <PopoverContent className="w-auto p-0">
              <Calendar
                mode="single"
                selected={selectedDate}
                onSelect={(date) => date && setSelectedDate(date)}
                initialFocus
              />
            </PopoverContent>
          </Popover>
          <Select value={selectedZone} onValueChange={setSelectedZone}>
            <SelectTrigger className="w-[180px]">
              <SelectValue placeholder="Service Zone" />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="all">All Zones</SelectItem>
              {/* Zones would be loaded from API */}
            </SelectContent>
          </Select>
        </div>
      </div>
      <div className="flex-1 overflow-hidden">
        <ScheduleBoard date={dateString} zoneFilter={selectedZone} />
      </div>
    </div>
  );
}
```

### 4.2 Schedule Board Component

**File**: `app/dispatch/schedule-board/components/ScheduleBoard.tsx`

```typescript
'use client';

import { useMemo } from 'react';
import { useAssignments } from '@/lib/hooks/useAssignments';
import { useTechnicians } from '@/lib/hooks/useTechnicians';
import { useRealtimeAssignments } from '@/lib/hooks/useRealtime';
import { TechnicianRow } from './TechnicianRow';
import { TimeAxis } from './TimeAxis';
import { Skeleton } from '@/components/ui/skeleton';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { AlertCircle } from 'lucide-react';

interface ScheduleBoardProps {
  date: string;
  zoneFilter: string;
}

export function ScheduleBoard({ date, zoneFilter }: ScheduleBoardProps) {
  useRealtimeAssignments(date);

  const { data: technicians, isLoading: techniciansLoading } = useTechnicians(zoneFilter);
  const { data: assignments, isLoading: assignmentsLoading, error } = useAssignments(date);

  const assignmentsByTechnician = useMemo(() => {
    if (!assignments) return {};
    
    return assignments.reduce((acc, assignment) => {
      const techId = assignment.technician_id;
      if (!acc[techId]) {
        acc[techId] = [];
      }
      acc[techId].push(assignment);
      return acc;
    }, {} as Record<string, typeof assignments>);
  }, [assignments]);

  const timeSlots = useMemo(() => {
    // Generate hourly slots from 6 AM to 8 PM
    const slots = [];
    for (let hour = 6; hour <= 20; hour++) {
      slots.push(`${hour.toString().padStart(2, '0')}:00`);
    }
    return slots;
  }, []);

  if (error) {
    return (
      <Alert variant="destructive" className="m-4">
        <AlertCircle className="h-4 w-4" />
        <AlertDescription>
          Failed to load schedule data. Please try again.
        </AlertDescription>
      </Alert>
    );
  }

  if (techniciansLoading || assignmentsLoading) {
    return (
      <div className="p-4 space-y-4">
        {[1, 2, 3].map((i) => (
          <Skeleton key={i} className="h-24 w-full" />
        ))}
      </div>
    );
  }

  return (
    <div className="flex flex-col h-full overflow-auto">
      <div className="sticky top-0 z-10 bg-background border-b">
        <TimeAxis slots={timeSlots} />
      </div>
      <div className="flex-1">
        {technicians && technicians.length > 0 ? (
          technicians.map((technician) => (
            <TechnicianRow
              key={technician.id}
              technician={technician}
              assignments={assignmentsByTechnician[technician.id] || []}
              date={date}
              timeSlots={timeSlots}
            />
          ))
        ) : (
          <div className="flex items-center justify-center h-full text-muted-foreground">
            No technicians found
          </div>
        )}
      </div>
    </div>
  );
}
```

### 4.3 Technician Row Component

**File**: `app/dispatch/schedule-board/components/TechnicianRow.tsx`

```typescript
'use client';

import { useMemo } from 'react';
import { Droppable, Draggable } from '@hello-pangea/dnd';
import { AssignmentBlock } from './AssignmentBlock';
import { Badge } from '@/components/ui/badge';
import { format, parseISO } from 'date-fns';
import type { TechnicianProfile, JobAssignment } from '@/types/dispatch';

interface TechnicianRowProps {
  technician: TechnicianProfile;
  assignments: JobAssignment[];
  date: string;
  timeSlots: string[];
}

export function TechnicianRow({ technician, assignments, date, timeSlots }: TechnicianRowProps) {
  const dateObj = parseISO(date);
  const startOfDay = new Date(dateObj);
  startOfDay.setHours(6, 0, 0, 0);
  const endOfDay = new Date(dateObj);
  endOfDay.setHours(20, 0, 0, 0);

  const totalMinutes = useMemo(() => {
    return assignments.reduce((total, assignment) => {
      const start = parseISO(assignment.scheduled_start_at);
      const end = parseISO(assignment.scheduled_end_at);
      return total + (end.getTime() - start.getTime()) / (1000 * 60);
    }, 0);
  }, [assignments]);

  const utilizationPercent = technician.max_daily_work_minutes
    ? Math.round((totalMinutes / technician.max_daily_work_minutes) * 100)
    : 0;

  return (
    <div className="border-b">
      <div className="flex">
        <div className="w-48 border-r p-3 bg-muted/50 flex flex-col justify-center">
          <div className="font-medium">{technician.display_name}</div>
          <div className="text-sm text-muted-foreground flex items-center gap-2 mt-1">
            <Badge variant={utilizationPercent > 100 ? 'destructive' : utilizationPercent > 80 ? 'warning' : 'default'}>
              {utilizationPercent}% utilized
            </Badge>
            <span>{assignments.length} jobs</span>
          </div>
        </div>
        <Droppable droppableId={technician.id} direction="horizontal" type="assignment">
          {(provided, snapshot) => (
            <div
              ref={provided.innerRef}
              {...provided.droppableProps}
              className={cn(
                'flex-1 relative min-h-[80px]',
                snapshot.isDraggingOver && 'bg-muted/50'
              )}
              style={{
                backgroundImage: `repeating-linear-gradient(
                  90deg,
                  transparent,
                  transparent calc(100% / ${timeSlots.length} - 1px),
                  hsl(var(--border)) calc(100% / ${timeSlots.length} - 1px),
                  hsl(var(--border)) calc(100% / ${timeSlots.length})
                )`
              }}
            >
              {assignments.map((assignment, index) => (
                <Draggable
                  key={assignment.id}
                  draggableId={assignment.id}
                  index={index}
                >
                  {(provided, snapshot) => (
                    <AssignmentBlock
                      ref={provided.innerRef}
                      {...provided.draggableProps}
                      {...provided.dragHandleProps}
                      assignment={assignment}
                      isDragging={snapshot.isDragging}
                      startOfDay={startOfDay}
                      endOfDay={endOfDay}
                    />
                  )}
                </Draggable>
              ))}
              {provided.placeholder}
            </div>
          )}
        </Droppable>
      </div>
    </div>
  );
}
```

### 4.4 Assignment Block Component

**File**: `app/dispatch/schedule-board/components/AssignmentBlock.tsx`

```typescript
'use client';

import { forwardRef } from 'react';
import { format, parseISO } from 'date-fns';
import { Badge } from '@/components/ui/badge';
import { cn } from '@/lib/utils';
import type { JobAssignment } from '@/types/dispatch';

interface AssignmentBlockProps {
  assignment: JobAssignment;
  isDragging: boolean;
  startOfDay: Date;
  endOfDay: Date;
}

export const AssignmentBlock = forwardRef<HTMLDivElement, AssignmentBlockProps & any>(
  ({ assignment, isDragging, startOfDay, endOfDay, ...props }, ref) => {
    const start = parseISO(assignment.scheduled_start_at);
    const end = parseISO(assignment.scheduled_end_at);
    
    const startOffset = ((start.getTime() - startOfDay.getTime()) / (endOfDay.getTime() - startOfDay.getTime())) * 100;
    const width = ((end.getTime() - start.getTime()) / (endOfDay.getTime() - startOfDay.getTime())) * 100;

    const priorityColors = {
      low: 'bg-blue-500',
      normal: 'bg-green-500',
      high: 'bg-orange-500',
      emergency: 'bg-red-500'
    };

    const statusColors = {
      assigned: 'border-blue-300',
      accepted: 'border-green-300',
      en_route: 'border-yellow-300',
      on_site: 'border-purple-300',
      completed: 'border-gray-300',
      canceled: 'border-red-300'
    };

    return (
      <div
        ref={ref}
        {...props}
        className={cn(
          'absolute top-1 bottom-1 rounded-md border-2 p-2 cursor-move shadow-sm',
          'hover:shadow-md transition-shadow',
          priorityColors[assignment.job?.priority || 'normal'],
          statusColors[assignment.status],
          isDragging && 'opacity-50 rotate-2'
        )}
        style={{
          left: `${startOffset}%`,
          width: `${width}%`,
          minWidth: '120px'
        }}
      >
        <div className="text-xs font-medium text-white truncate">
          {assignment.job?.title || 'Untitled Job'}
        </div>
        <div className="text-xs text-white/80 mt-1">
          {format(start, 'h:mm a')} - {format(end, 'h:mm a')}
        </div>
        {assignment.job?.customer && (
          <div className="text-xs text-white/70 truncate mt-1">
            {assignment.job.customer.name}
          </div>
        )}
        <div className="flex gap-1 mt-1">
          <Badge variant="secondary" className="text-xs h-4 px-1">
            {assignment.status}
          </Badge>
          {assignment.tech_eta_at && (
            <Badge variant="outline" className="text-xs h-4 px-1">
              ETA: {format(parseISO(assignment.tech_eta_at), 'h:mm a')}
            </Badge>
          )}
        </div>
      </div>
    );
  }
);

AssignmentBlock.displayName = 'AssignmentBlock';
```

### 4.5 Time Axis Component

**File**: `app/dispatch/schedule-board/components/TimeAxis.tsx`

```typescript
'use client';

import { cn } from '@/lib/utils';

interface TimeAxisProps {
  slots: string[];
}

export function TimeAxis({ slots }: TimeAxisProps) {
  return (
    <div className="flex">
      <div className="w-48 border-r p-2 text-sm font-medium text-muted-foreground">
        Time
      </div>
      <div className="flex-1 flex">
        {slots.map((slot, index) => (
          <div
            key={slot}
            className={cn(
              'flex-1 border-r p-2 text-xs text-muted-foreground text-center',
              index === 0 && 'border-l'
            )}
          >
            {slot}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 4.6 Drag-and-Drop Handler

**File**: `app/dispatch/schedule-board/components/ScheduleBoard.tsx` (updated)

```typescript
'use client';

import { DragDropContext, DropResult } from '@hello-pangea/dnd';
import { useUpdateAssignment } from '@/lib/hooks/useAssignments';
import { parseISO, addMinutes } from 'date-fns';
import { toast } from 'sonner';
import type { JobAssignment } from '@/types/dispatch';

// Updated ScheduleBoard component with drag-and-drop:
export function ScheduleBoard({ date, zoneFilter }: ScheduleBoardProps) {
  useRealtimeAssignments(date);

  const { data: technicians, isLoading: techniciansLoading } = useTechnicians(zoneFilter);
  const { data: assignments, isLoading: assignmentsLoading, error } = useAssignments(date);
  const updateAssignment = useUpdateAssignment();

  // ... existing code ...

  const dateObj = parseISO(date);
  const startOfDay = new Date(dateObj);
  startOfDay.setHours(6, 0, 0, 0);
  const endOfDay = new Date(dateObj);
  endOfDay.setHours(20, 0, 0, 0);

  const handleDragEnd = async (result: DropResult) => {
    if (!result.destination || !assignments) return;

    const assignmentId = result.draggableId;
    const sourceTechId = result.source.droppableId;
    const destTechId = result.destination.droppableId;

    const assignment = assignments.find(a => a.id === assignmentId);
    if (!assignment) return;

    // Calculate new time based on drop position
    // Use time slot index to determine position
    const slotIndex = result.destination.index;
    const totalSlots = timeSlots.length;
    const slotDuration = (endOfDay.getTime() - startOfDay.getTime()) / totalSlots;
    
    const newStartTime = new Date(startOfDay.getTime() + (slotIndex * slotDuration));
    
    // Snap to 15-minute intervals
    const minutes = newStartTime.getMinutes();
    const snappedMinutes = Math.round(minutes / 15) * 15;
    newStartTime.setMinutes(snappedMinutes);
    newStartTime.setSeconds(0);
    newStartTime.setMilliseconds(0);
    
    const duration = (parseISO(assignment.scheduled_end_at).getTime() - parseISO(assignment.scheduled_start_at).getTime()) / (1000 * 60);
    const newEndTime = addMinutes(newStartTime, duration);

    const updates: Partial<JobAssignment> = {
      scheduled_start_at: newStartTime.toISOString(),
      scheduled_end_at: newEndTime.toISOString()
    };

    if (destTechId !== sourceTechId) {
      updates.technician_id = destTechId;
    }

    try {
      await updateAssignment.mutateAsync({
        assignmentId,
        updates
      });
      toast.success('Assignment updated successfully');
    } catch (error) {
      console.error('Failed to update assignment:', error);
      toast.error('Failed to update assignment');
    }
  };

  return (
    <DragDropContext onDragEnd={handleDragEnd}>
      {/* ... existing content ... */}
    </DragDropContext>
  );
}
```

**Snap Intervals**: Assignments snap to 15-minute intervals when dragged. This can be configured via a constant or user preference.

---

## 5. Story DISP-052: Map-Based Dispatch View

### 5.1 Map View Page

**File**: `app/dispatch/map-view/page.tsx`

```typescript
'use client';

import { useState } from 'react';
import { MapView } from './components/MapView';
import { MapFilters } from './components/MapFilters';
import { Button } from '@/components/ui/button';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';

export default function MapViewPage() {
  const [filters, setFilters] = useState({
    zone: 'all',
    skill: 'all',
    status: 'all',
    priority: 'all'
  });

  return (
    <div className="flex flex-col h-screen">
      <div className="border-b p-4">
        <h1 className="text-2xl font-bold mb-4">Map View</h1>
        <MapFilters filters={filters} onFiltersChange={setFilters} />
      </div>
      <div className="flex-1 relative">
        <MapView filters={filters} />
      </div>
    </div>
  );
}
```

### 5.2 Map View Component

**File**: `app/dispatch/map-view/components/MapView.tsx`

```typescript
'use client';

import { useMemo } from 'react';
import Map, { Marker, Source, Layer } from 'react-map-gl';
import 'mapbox-gl/dist/mapbox-gl.css';
import { useJobs } from '@/lib/hooks/useJobs';
import { useRoutePlans } from '@/lib/hooks/useRoutes';
import { JobMarker } from './JobMarker';
import { RouteOverlay } from './RouteOverlay';
import { Skeleton } from '@/components/ui/skeleton';

interface MapViewProps {
  filters: {
    zone: string;
    skill: string;
    status: string;
    priority: string;
  };
}

export function MapView({ filters }: MapViewProps) {
  const { data: jobs, isLoading } = useJobs(filters);
  const { data: routes } = useRoutePlans(new Date().toISOString().split('T')[0]);

  const mapboxToken = process.env.NEXT_PUBLIC_MAPBOX_TOKEN;

  if (!mapboxToken) {
    return (
      <div className="flex items-center justify-center h-full">
        <div className="text-center">
          <p className="text-muted-foreground">Mapbox token not configured</p>
        </div>
      </div>
    );
  }

  const filteredJobs = useMemo(() => {
    if (!jobs) return [];
    
    return jobs.filter(job => {
      if (filters.zone !== 'all' && job.service_zone_id !== filters.zone) return false;
      if (filters.status !== 'all' && job.status !== filters.status) return false;
      if (filters.priority !== 'all' && job.priority !== filters.priority) return false;
      return true;
    });
  }, [jobs, filters]);

  if (isLoading) {
    return <Skeleton className="w-full h-full" />;
  }

  return (
    <Map
      initialViewState={{
        longitude: -89.6501, // Default center (Springfield, IL)
        latitude: 39.7817,
        zoom: 11
      }}
      style={{ width: '100%', height: '100%' }}
      mapStyle="mapbox://styles/mapbox/streets-v12"
      mapboxAccessToken={mapboxToken}
    >
      {filteredJobs.map((job) => {
        if (!job.location?.latitude || !job.location?.longitude) return null;
        
        return (
          <JobMarker
            key={job.id}
            job={job}
            longitude={job.location.longitude}
            latitude={job.location.latitude}
          />
        );
      })}
      
      {routes?.map((route) => (
        <RouteOverlay key={route.id} route={route} />
      ))}
    </Map>
  );
}
```

### 5.3 Job Marker Component

**File**: `app/dispatch/map-view/components/JobMarker.tsx`

```typescript
'use client';

import { Marker, Popup } from 'react-map-gl';
import { useState } from 'react';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { format, parseISO } from 'date-fns';
import type { DispatchJob } from '@/types/dispatch';

interface JobMarkerProps {
  job: DispatchJob;
  longitude: number;
  latitude: number;
}

const priorityColors = {
  low: 'bg-blue-500',
  normal: 'bg-green-500',
  high: 'bg-orange-500',
  emergency: 'bg-red-500'
};

export function JobMarker({ job, longitude, latitude }: JobMarkerProps) {
  const [showPopup, setShowPopup] = useState(false);

  return (
    <>
      <Marker
        longitude={longitude}
        latitude={latitude}
        anchor="bottom"
        onClick={() => setShowPopup(true)}
      >
        <div
          className={cn(
            'w-6 h-6 rounded-full border-2 border-white shadow-lg cursor-pointer',
            priorityColors[job.priority]
          )}
        />
      </Marker>
      {showPopup && (
        <Popup
          longitude={longitude}
          latitude={latitude}
          anchor="top"
          onClose={() => setShowPopup(false)}
          closeButton={true}
        >
          <Card className="w-64">
            <CardHeader className="pb-2">
              <CardTitle className="text-sm">{job.title}</CardTitle>
            </CardHeader>
            <CardContent className="space-y-2">
              <div className="text-xs">
                <Badge variant="outline" className="mr-1">{job.priority}</Badge>
                <Badge variant="secondary">{job.status}</Badge>
              </div>
              {job.customer && (
                <div className="text-xs text-muted-foreground">
                  {job.customer.name}
                </div>
              )}
              {job.location && (
                <div className="text-xs text-muted-foreground">
                  {job.location.address_line1}, {job.location.city}
                </div>
              )}
              {job.assignments && job.assignments.length > 0 && (
                <div className="text-xs">
                  <div className="font-medium">Scheduled:</div>
                  {job.assignments.map((assignment) => (
                    <div key={assignment.id} className="text-muted-foreground">
                      {format(parseISO(assignment.scheduled_start_at), 'MMM d, h:mm a')}
                    </div>
                  ))}
                </div>
              )}
              <Button size="sm" className="w-full mt-2">
                View Details
              </Button>
            </CardContent>
          </Card>
        </Popup>
      )}
    </>
  );
}
```

### 5.4 Map Filters Component

**File**: `app/dispatch/map-view/components/MapFilters.tsx`

```typescript
'use client';

import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { Label } from '@/components/ui/label';

interface MapFiltersProps {
  filters: {
    zone: string;
    skill: string;
    status: string;
    priority: string;
  };
  onFiltersChange: (filters: MapFiltersProps['filters']) => void;
}

export function MapFilters({ filters, onFiltersChange }: MapFiltersProps) {
  return (
    <div className="flex gap-4">
      <div className="flex items-center gap-2">
        <Label htmlFor="zone">Zone</Label>
        <Select
          value={filters.zone}
          onValueChange={(value) => onFiltersChange({ ...filters, zone: value })}
        >
          <SelectTrigger id="zone" className="w-[180px]">
            <SelectValue />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="all">All Zones</SelectItem>
            {/* Zones loaded from API */}
          </SelectContent>
        </Select>
      </div>
      
      <div className="flex items-center gap-2">
        <Label htmlFor="status">Status</Label>
        <Select
          value={filters.status}
          onValueChange={(value) => onFiltersChange({ ...filters, status: value })}
        >
          <SelectTrigger id="status" className="w-[180px]">
            <SelectValue />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="all">All Statuses</SelectItem>
            <SelectItem value="unscheduled">Unscheduled</SelectItem>
            <SelectItem value="scheduled">Scheduled</SelectItem>
            <SelectItem value="in_progress">In Progress</SelectItem>
            <SelectItem value="completed">Completed</SelectItem>
          </SelectContent>
        </Select>
      </div>
      
      <div className="flex items-center gap-2">
        <Label htmlFor="priority">Priority</Label>
        <Select
          value={filters.priority}
          onValueChange={(value) => onFiltersChange({ ...filters, priority: value })}
        >
          <SelectTrigger id="priority" className="w-[180px]">
            <SelectValue />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="all">All Priorities</SelectItem>
            <SelectItem value="low">Low</SelectItem>
            <SelectItem value="normal">Normal</SelectItem>
            <SelectItem value="high">High</SelectItem>
            <SelectItem value="emergency">Emergency</SelectItem>
          </SelectContent>
        </Select>
      </div>
    </div>
  );
}
```

---

## 6. Story DISP-053: Job Creation UI + Detail Drawer

### 6.1 Job Form Component

**File**: `app/dispatch/jobs/components/JobForm.tsx`

```typescript
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { Textarea } from '@/components/ui/textarea';
import { Calendar } from '@/components/ui/calendar';
import { Popover, PopoverContent, PopoverTrigger } from '@/components/ui/popover';
import { format } from 'date-fns';
import { CalendarIcon } from 'lucide-react';
import { cn } from '@/lib/utils';
import { useCreateJob } from '@/lib/hooks/useJobs';

const jobSchema = z.object({
  customer_id: z.string().uuid(),
  location_id: z.string().uuid(),
  title: z.string().min(1).max(255),
  description: z.string().optional(),
  job_type: z.string().optional(),
  priority: z.enum(['low', 'normal', 'high', 'emergency']),
  estimated_duration_minutes: z.number().min(1),
  required_skills: z.array(z.string()).optional(),
  required_crew_size: z.number().min(1).default(1),
  service_zone_id: z.string().uuid().optional(),
  sla_start_at: z.string().optional(),
  sla_end_at: z.string().optional(),
  is_customer_booked: z.boolean().default(false),
  notes_internal: z.string().optional()
});

type JobFormData = z.infer<typeof jobSchema>;

interface JobFormProps {
  onSuccess?: () => void;
  initialData?: Partial<JobFormData>;
}

export function JobForm({ onSuccess, initialData }: JobFormProps) {
  const createJob = useCreateJob();
  
  const {
    register,
    handleSubmit,
    formState: { errors },
    setValue,
    watch
  } = useForm<JobFormData>({
    resolver: zodResolver(jobSchema),
    defaultValues: {
      priority: 'normal',
      estimated_duration_minutes: 60,
      required_crew_size: 1,
      is_customer_booked: false,
      ...initialData
    }
  });

  const onSubmit = async (data: JobFormData) => {
    try {
      await createJob.mutateAsync(data);
      onSuccess?.();
    } catch (error) {
      console.error('Failed to create job:', error);
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <div className="grid grid-cols-2 gap-4">
        <div className="space-y-2">
          <Label htmlFor="title">Job Title *</Label>
          <Input
            id="title"
            {...register('title')}
            placeholder="e.g., AC Maintenance"
          />
          {errors.title && (
            <p className="text-sm text-destructive">{errors.title.message}</p>
          )}
        </div>

        <div className="space-y-2">
          <Label htmlFor="priority">Priority *</Label>
          <Select
            value={watch('priority')}
            onValueChange={(value) => setValue('priority', value as any)}
          >
            <SelectTrigger id="priority">
              <SelectValue />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="low">Low</SelectItem>
              <SelectItem value="normal">Normal</SelectItem>
              <SelectItem value="high">High</SelectItem>
              <SelectItem value="emergency">Emergency</SelectItem>
            </SelectContent>
          </Select>
        </div>
      </div>

      <div className="space-y-2">
        <Label htmlFor="description">Description</Label>
        <Textarea
          id="description"
          {...register('description')}
          placeholder="Job description..."
          rows={3}
        />
      </div>

      <div className="grid grid-cols-2 gap-4">
        <div className="space-y-2">
          <Label htmlFor="estimated_duration_minutes">Duration (minutes) *</Label>
          <Input
            id="estimated_duration_minutes"
            type="number"
            {...register('estimated_duration_minutes', { valueAsNumber: true })}
            min={1}
          />
        </div>

        <div className="space-y-2">
          <Label htmlFor="required_crew_size">Crew Size *</Label>
          <Input
            id="required_crew_size"
            type="number"
            {...register('required_crew_size', { valueAsNumber: true })}
            min={1}
          />
        </div>
      </div>

      <div className="grid grid-cols-2 gap-4">
        <div className="space-y-2">
          <Label htmlFor="sla_start_at">SLA Start</Label>
          <Popover>
            <PopoverTrigger asChild>
              <Button
                variant="outline"
                className={cn(
                  'w-full justify-start text-left font-normal',
                  !watch('sla_start_at') && 'text-muted-foreground'
                )}
              >
                <CalendarIcon className="mr-2 h-4 w-4" />
                {watch('sla_start_at') ? (
                  format(new Date(watch('sla_start_at')!), 'PPP')
                ) : (
                  <span>Pick a date</span>
                )}
              </Button>
            </PopoverTrigger>
            <PopoverContent className="w-auto p-0">
              <Calendar
                mode="single"
                selected={watch('sla_start_at') ? new Date(watch('sla_start_at')!) : undefined}
                onSelect={(date) => setValue('sla_start_at', date?.toISOString())}
              />
            </PopoverContent>
          </Popover>
        </div>

        <div className="space-y-2">
          <Label htmlFor="sla_end_at">SLA End</Label>
          <Popover>
            <PopoverTrigger asChild>
              <Button
                variant="outline"
                className={cn(
                  'w-full justify-start text-left font-normal',
                  !watch('sla_end_at') && 'text-muted-foreground'
                )}
              >
                <CalendarIcon className="mr-2 h-4 w-4" />
                {watch('sla_end_at') ? (
                  format(new Date(watch('sla_end_at')!), 'PPP')
                ) : (
                  <span>Pick a date</span>
                )}
              </Button>
            </PopoverTrigger>
            <PopoverContent className="w-auto p-0">
              <Calendar
                mode="single"
                selected={watch('sla_end_at') ? new Date(watch('sla_end_at')!) : undefined}
                onSelect={(date) => setValue('sla_end_at', date?.toISOString())}
              />
            </PopoverContent>
          </Popover>
        </div>
      </div>

      <div className="flex justify-end gap-2">
        <Button type="button" variant="outline" onClick={onSuccess}>
          Cancel
        </Button>
        <Button type="submit" disabled={createJob.isPending}>
          {createJob.isPending ? 'Creating...' : 'Create Job'}
        </Button>
      </div>
    </form>
  );
}
```

### 6.2 Assignment List Component

**File**: `app/dispatch/jobs/components/AssignmentList.tsx`

```typescript
'use client';

import { format, parseISO } from 'date-fns';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table';
import type { JobAssignment } from '@/types/dispatch';

interface AssignmentListProps {
  jobId: string;
  assignments: JobAssignment[];
}

export function AssignmentList({ assignments }: AssignmentListProps) {
  if (assignments.length === 0) {
    return (
      <div className="text-sm text-muted-foreground py-4">
        No assignments yet
      </div>
    );
  }

  return (
    <Table>
      <TableHeader>
        <TableRow>
          <TableHead>Technician</TableHead>
          <TableHead>Start Time</TableHead>
          <TableHead>End Time</TableHead>
          <TableHead>Status</TableHead>
          <TableHead>ETA</TableHead>
          <TableHead>Actions</TableHead>
        </TableRow>
      </TableHeader>
      <TableBody>
        {assignments.map((assignment) => (
          <TableRow key={assignment.id}>
            <TableCell>
              {assignment.technician?.display_name || 'Unknown'}
            </TableCell>
            <TableCell>
              {format(parseISO(assignment.scheduled_start_at), 'MMM d, h:mm a')}
            </TableCell>
            <TableCell>
              {format(parseISO(assignment.scheduled_end_at), 'MMM d, h:mm a')}
            </TableCell>
            <TableCell>
              <Badge variant="outline">{assignment.status}</Badge>
            </TableCell>
            <TableCell>
              {assignment.tech_eta_at
                ? format(parseISO(assignment.tech_eta_at), 'h:mm a')
                : 'N/A'}
            </TableCell>
            <TableCell>
              <Button variant="ghost" size="sm">
                Edit
              </Button>
            </TableCell>
          </TableRow>
        ))}
      </TableBody>
    </Table>
  );
}
```

### 6.3 Job Detail Drawer

**File**: `app/dispatch/jobs/components/JobDetailDrawer.tsx`

```typescript
'use client';

import { useState } from 'react';
import {
  Sheet,
  SheetContent,
  SheetDescription,
  SheetHeader,
  SheetTitle,
} from '@/components/ui/sheet'; // shadcn/ui Sheet component (based on Radix UI Dialog)
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { AssignmentList } from './AssignmentList';
import { format, parseISO } from 'date-fns';
import type { DispatchJob } from '@/types/dispatch';
import { useAutoScheduleJob } from '@/lib/hooks/useJobs';

interface JobDetailDrawerProps {
  job: DispatchJob | null;
  open: boolean;
  onOpenChange: (open: boolean) => void;
}

export function JobDetailDrawer({ job, open, onOpenChange }: JobDetailDrawerProps) {
  const autoSchedule = useAutoScheduleJob();

  if (!job) return null;

  const handleAutoSchedule = async () => {
    try {
      await autoSchedule.mutateAsync({ jobId: job.id, mode: 'commit' });
    } catch (error) {
      console.error('Failed to auto-schedule:', error);
    }
  };

  return (
    <Sheet open={open} onOpenChange={onOpenChange}>
      <SheetContent className="w-full sm:max-w-2xl overflow-y-auto">
        <SheetHeader>
          <SheetTitle className="flex items-center gap-2">
            {job.title}
            <Badge variant="outline">{job.priority}</Badge>
            <Badge variant="secondary">{job.status}</Badge>
          </SheetTitle>
          <SheetDescription>
            Job ID: {job.id}
          </SheetDescription>
        </SheetHeader>

        <Tabs defaultValue="details" className="mt-4">
          <TabsList>
            <TabsTrigger value="details">Details</TabsTrigger>
            <TabsTrigger value="assignments">Assignments</TabsTrigger>
            <TabsTrigger value="time-windows">Time Windows</TabsTrigger>
          </TabsList>

          <TabsContent value="details" className="space-y-4">
            <div className="grid grid-cols-2 gap-4">
              <div>
                <div className="text-sm font-medium text-muted-foreground">Customer</div>
                <div className="text-sm">{job.customer?.name || 'N/A'}</div>
              </div>
              <div>
                <div className="text-sm font-medium text-muted-foreground">Location</div>
                <div className="text-sm">
                  {job.location?.address_line1}, {job.location?.city}
                </div>
              </div>
              <div>
                <div className="text-sm font-medium text-muted-foreground">Duration</div>
                <div className="text-sm">{job.estimated_duration_minutes} minutes</div>
              </div>
              <div>
                <div className="text-sm font-medium text-muted-foreground">Crew Size</div>
                <div className="text-sm">{job.required_crew_size}</div>
              </div>
            </div>

            {job.description && (
              <div>
                <div className="text-sm font-medium text-muted-foreground mb-1">Description</div>
                <div className="text-sm">{job.description}</div>
              </div>
            )}

            {job.sla_start_at && job.sla_end_at && (
              <div>
                <div className="text-sm font-medium text-muted-foreground mb-1">SLA Window</div>
                <div className="text-sm">
                  {format(parseISO(job.sla_start_at), 'PPP p')} - {format(parseISO(job.sla_end_at), 'PPP p')}
                </div>
              </div>
            )}

            <div className="flex gap-2 pt-4 border-t">
              <Button onClick={handleAutoSchedule} disabled={autoSchedule.isPending}>
                Auto-Schedule
              </Button>
              <Button variant="outline">Manual Assign</Button>
            </div>
          </TabsContent>

          <TabsContent value="assignments">
            <AssignmentList jobId={job.id} assignments={job.assignments || []} />
          </TabsContent>

          <TabsContent value="time-windows">
            <div className="space-y-2">
              {job.time_windows && job.time_windows.length > 0 ? (
                job.time_windows.map((window) => (
                  <div
                    key={window.id}
                    className={cn(
                      'p-3 border rounded-md',
                      window.is_selected && 'bg-muted'
                    )}
                  >
                    <div className="flex items-center justify-between">
                      <div>
                        <div className="text-sm font-medium">
                          {format(parseISO(window.window_start), 'PPP p')}
                        </div>
                        <div className="text-xs text-muted-foreground">
                          to {format(parseISO(window.window_end), 'PPP p')}
                        </div>
                      </div>
                      {window.is_selected && (
                        <Badge variant="default">Selected</Badge>
                      )}
                    </div>
                  </div>
                ))
              ) : (
                <div className="text-sm text-muted-foreground">No time windows</div>
              )}
            </div>
          </TabsContent>
        </Tabs>
      </SheetContent>
    </Sheet>
  );
}
```

---

## 7. Story DISP-054: Capacity & Utilization View

### 7.1 Capacity View Page

**File**: `app/dispatch/capacity/page.tsx`

```typescript
'use client';

import { useState } from 'react';
import { CapacityView } from './components/CapacityView';
import { Calendar } from '@/components/ui/calendar';
import { Popover, PopoverContent, PopoverTrigger } from '@/components/ui/popover';
import { Button } from '@/components/ui/button';
import { format } from 'date-fns';
import { CalendarIcon } from 'lucide-react';
import { cn } from '@/lib/utils';

export default function CapacityPage() {
  const [selectedDate, setSelectedDate] = useState<Date>(new Date());

  return (
    <div className="container mx-auto py-6">
      <div className="flex items-center justify-between mb-6">
        <h1 className="text-2xl font-bold">Capacity & Utilization</h1>
        <Popover>
          <PopoverTrigger asChild>
            <Button
              variant="outline"
              className={cn(
                'w-[280px] justify-start text-left font-normal',
                !selectedDate && 'text-muted-foreground'
              )}
            >
              <CalendarIcon className="mr-2 h-4 w-4" />
              {selectedDate ? format(selectedDate, 'PPP') : <span>Pick a date</span>}
            </Button>
          </PopoverTrigger>
          <PopoverContent className="w-auto p-0">
            <Calendar
              mode="single"
              selected={selectedDate}
              onSelect={(date) => date && setSelectedDate(date)}
              initialFocus
            />
          </PopoverContent>
        </Popover>
      </div>
      <CapacityView date={format(selectedDate, 'yyyy-MM-dd')} />
    </div>
  );
}
```

### 7.2 Capacity View Component

**File**: `app/dispatch/capacity/components/CapacityView.tsx`

```typescript
'use client';

import { useMemo } from 'react';
import { useTechnicians } from '@/lib/hooks/useTechnicians';
import { useAssignments } from '@/lib/hooks/useAssignments';
import { TechnicianCapacityCard } from './TechnicianCapacityCard';
import { Skeleton } from '@/components/ui/skeleton';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { AlertTriangle } from 'lucide-react';

interface CapacityViewProps {
  date: string;
}

export function CapacityView({ date }: CapacityViewProps) {
  const { data: technicians, isLoading: techniciansLoading } = useTechnicians();
  const { data: assignments, isLoading: assignmentsLoading } = useAssignments(date);

  const capacityData = useMemo(() => {
    if (!technicians || !assignments) return [];

    return technicians.map((technician) => {
      const techAssignments = assignments.filter(
        a => a.technician_id === technician.id
      );

      const scheduledMinutes = techAssignments.reduce((total, assignment) => {
        const start = new Date(assignment.scheduled_start_at);
        const end = new Date(assignment.scheduled_end_at);
        return total + (end.getTime() - start.getTime()) / (1000 * 60);
      }, 0);

      const availableMinutes = technician.max_daily_work_minutes || 480; // Default 8 hours
      const utilizationPercent = Math.round((scheduledMinutes / availableMinutes) * 100);

      // Check for SLA risks
      const slaRisks = techAssignments.filter((assignment) => {
        if (!assignment.job?.sla_end_at) return false;
        const slaEnd = new Date(assignment.job.sla_end_at);
        const scheduledEnd = new Date(assignment.scheduled_end_at);
        const hoursUntilSLA = (slaEnd.getTime() - scheduledEnd.getTime()) / (1000 * 60 * 60);
        return hoursUntilSLA < 2 && hoursUntilSLA > 0; // Less than 2 hours buffer
      });

      return {
        technician,
        scheduledMinutes,
        availableMinutes,
        utilizationPercent,
        assignmentCount: techAssignments.length,
        slaRisks: slaRisks.length
      };
    });
  }, [technicians, assignments]);

  if (techniciansLoading || assignmentsLoading) {
    return (
      <div className="space-y-4">
        {[1, 2, 3].map((i) => (
          <Skeleton key={i} className="h-32 w-full" />
        ))}
      </div>
    );
  }

  return (
    <div className="space-y-4">
      {capacityData.map((data) => (
        <TechnicianCapacityCard key={data.technician.id} data={data} />
      ))}
    </div>
  );
}
```

### 7.3 Technician Capacity Card

**File**: `app/dispatch/capacity/components/TechnicianCapacityCard.tsx`

```typescript
'use client';

import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Progress } from '@/components/ui/progress';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { AlertTriangle } from 'lucide-react';

interface TechnicianCapacityCardProps {
  data: {
    technician: any;
    scheduledMinutes: number;
    availableMinutes: number;
    utilizationPercent: number;
    assignmentCount: number;
    slaRisks: number;
  };
}

export function TechnicianCapacityCard({ data }: TechnicianCapacityCardProps) {
  const { technician, scheduledMinutes, availableMinutes, utilizationPercent, assignmentCount, slaRisks } = data;

  const isOverCapacity = utilizationPercent > 100;
  const isNearCapacity = utilizationPercent > 80 && utilizationPercent <= 100;

  return (
    <Card>
      <CardHeader>
        <CardTitle className="flex items-center justify-between">
          <span>{technician.display_name}</span>
          <Badge
            variant={
              isOverCapacity
                ? 'destructive'
                : isNearCapacity
                ? 'default'
                : 'secondary'
            }
          >
            {utilizationPercent}% utilized
          </Badge>
        </CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        <div className="space-y-2">
          <div className="flex justify-between text-sm">
            <span className="text-muted-foreground">Scheduled</span>
            <span className="font-medium">
              {Math.round(scheduledMinutes)} / {availableMinutes} minutes
            </span>
          </div>
          <Progress value={Math.min(utilizationPercent, 100)} />
        </div>

        <div className="grid grid-cols-2 gap-4 text-sm">
          <div>
            <div className="text-muted-foreground">Assignments</div>
            <div className="font-medium">{assignmentCount}</div>
          </div>
          <div>
            <div className="text-muted-foreground">Remaining</div>
            <div className="font-medium">
              {Math.max(0, availableMinutes - Math.round(scheduledMinutes))} minutes
            </div>
          </div>
        </div>

        {isOverCapacity && (
          <Alert variant="destructive">
            <AlertTriangle className="h-4 w-4" />
            <AlertDescription>
              Technician is over capacity by {Math.round(scheduledMinutes - availableMinutes)} minutes
            </AlertDescription>
          </Alert>
        )}

        {slaRisks > 0 && (
          <Alert>
            <AlertTriangle className="h-4 w-4" />
            <AlertDescription>
              {slaRisks} assignment{slaRisks > 1 ? 's' : ''} near SLA deadline
            </AlertDescription>
          </Alert>
        )}
      </CardContent>
    </Card>
  );
}
```

---

## 8. Story DISP-055: Optimization Actions UI

### 8.1 Optimize Actions Component

**File**: `app/dispatch/optimize/components/OptimizeActions.tsx`

```typescript
'use client';

import { useState } from 'react';
import { Button } from '@/components/ui/button';
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from '@/components/ui/dialog';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { Label } from '@/components/ui/label';
import { useOptimizeRoute } from '@/lib/hooks/useOptimization';
import { useOptimizeDay } from '@/lib/hooks/useOptimization';
import { OptimizationProgress } from './OptimizationProgress';
import { format } from 'date-fns';
import { Loader2 } from 'lucide-react';

interface OptimizeActionsProps {
  date: string;
  technicianId?: string;
}

export function OptimizeActions({ date, technicianId }: OptimizeActionsProps) {
  const [strategy, setStrategy] = useState<'balanced' | 'time_minimization' | 'distance_minimization' | 'priority_first'>('balanced');
  const [showProgress, setShowProgress] = useState(false);
  
  const optimizeRoute = useOptimizeRoute();
  const optimizeDay = useOptimizeDay();

  const handleOptimizeRoute = async () => {
    if (!technicianId) return;
    
    setShowProgress(true);
    try {
      await optimizeRoute.mutateAsync({
        technicianId,
        date,
        strategy,
        mode: 'commit'
      });
    } catch (error) {
      console.error('Optimization failed:', error);
    } finally {
      setShowProgress(false);
    }
  };

  const handleOptimizeDay = async () => {
    setShowProgress(true);
    try {
      await optimizeDay.mutateAsync({
        date,
        strategy,
        mode: 'commit'
      });
    } catch (error) {
      console.error('Optimization failed:', error);
    } finally {
      setShowProgress(false);
    }
  };

  return (
    <div className="space-y-4">
      <div className="flex items-center gap-4">
        <div className="flex items-center gap-2">
          <Label htmlFor="strategy">Strategy</Label>
          <Select value={strategy} onValueChange={(value: any) => setStrategy(value)}>
            <SelectTrigger id="strategy" className="w-[200px]">
              <SelectValue />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="balanced">Balanced</SelectItem>
              <SelectItem value="time_minimization">Minimize Time</SelectItem>
              <SelectItem value="distance_minimization">Minimize Distance</SelectItem>
              <SelectItem value="priority_first">Priority First</SelectItem>
            </SelectContent>
          </Select>
        </div>

        {technicianId ? (
          <Button
            onClick={handleOptimizeRoute}
            disabled={optimizeRoute.isPending}
          >
            {optimizeRoute.isPending ? (
              <>
                <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                Optimizing...
              </>
            ) : (
              'Optimize Route'
            )}
          </Button>
        ) : (
          <Button
            onClick={handleOptimizeDay}
            disabled={optimizeDay.isPending}
          >
            {optimizeDay.isPending ? (
              <>
                <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                Optimizing...
              </>
            ) : (
              'Optimize All Routes'
            )}
          </Button>
        )}
      </div>

      {showProgress && (
        <OptimizationProgress
          runId={optimizeRoute.data?.run_id || optimizeDay.data?.run_id}
          onComplete={() => setShowProgress(false)}
        />
      )}

      {(optimizeRoute.data || optimizeDay.data) && (
        <div className="p-4 border rounded-md bg-muted">
          <div className="text-sm font-medium mb-2">Optimization Results</div>
          <div className="text-sm space-y-1">
            <div>
              Distance saved: {optimizeRoute.data?.estimated_savings_km || optimizeDay.data?.summary?.total_distance_saved_km || 0} km
            </div>
            <div>
              Time saved: {optimizeRoute.data?.estimated_savings_minutes || optimizeDay.data?.summary?.total_time_saved_minutes || 0} minutes
            </div>
          </div>
        </div>
      )}
    </div>
  );
}
```

### 8.2 Optimization Progress Component

**File**: `app/dispatch/optimize/components/OptimizationProgress.tsx`

```typescript
'use client';

import { useEffect } from 'react';
import { useQuery } from '@tanstack/react-query';
import { createSupabaseClient } from '@/lib/supabase/client';
import { Progress } from '@/components/ui/progress';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { CheckCircle2, XCircle, Loader2 } from 'lucide-react';

interface OptimizationProgressProps {
  runId: string | undefined;
  onComplete: () => void;
}

export function OptimizationProgress({ runId, onComplete }: OptimizationProgressProps) {
  const supabase = createSupabaseClient();

  const { data: runStatus } = useQuery({
    queryKey: ['optimization-run', runId],
    queryFn: async () => {
      if (!runId) return null;

      const { data, error } = await supabase
        .from('optimization_runs')
        .select('*')
        .eq('id', runId)
        .single();

      if (error) throw error;
      return data;
    },
    enabled: !!runId,
    refetchInterval: (query) => {
      const data = query.state.data;
      if (data?.status === 'succeeded' || data?.status === 'failed') {
        return false;
      }
      return 2000; // Poll every 2 seconds
    }
  });

  useEffect(() => {
    if (runStatus?.status === 'succeeded' || runStatus?.status === 'failed') {
      setTimeout(onComplete, 3000); // Auto-close after 3 seconds
    }
  }, [runStatus?.status, onComplete]);

  if (!runStatus) return null;

  const progress = runStatus.status === 'running' ? 50 : runStatus.status === 'succeeded' ? 100 : 0;

  return (
    <Alert>
      <div className="space-y-2">
        <div className="flex items-center gap-2">
          {runStatus.status === 'running' && <Loader2 className="h-4 w-4 animate-spin" />}
          {runStatus.status === 'succeeded' && <CheckCircle2 className="h-4 w-4 text-green-500" />}
          {runStatus.status === 'failed' && <XCircle className="h-4 w-4 text-red-500" />}
          <span className="font-medium">
            {runStatus.status === 'running' && 'Optimization in progress...'}
            {runStatus.status === 'succeeded' && 'Optimization completed'}
            {runStatus.status === 'failed' && 'Optimization failed'}
          </span>
        </div>
        {runStatus.status === 'running' && (
          <Progress value={progress} className="w-full" />
        )}
        {runStatus.status === 'failed' && runStatus.error_message && (
          <AlertDescription className="text-destructive">
            {runStatus.error_message}
          </AlertDescription>
        )}
        {runStatus.status === 'succeeded' && runStatus.result_metadata && (
          <AlertDescription>
            <div className="text-sm space-y-1">
              <div>Technicians processed: {runStatus.result_metadata.technicians_processed}</div>
              <div>Distance saved: {runStatus.result_metadata.total_distance_saved_km} km</div>
              <div>Time saved: {runStatus.result_metadata.total_time_saved_minutes} minutes</div>
            </div>
          </AlertDescription>
        )}
      </div>
    </Alert>
  );
}
```

---

## 9. Additional Hooks and Utilities

### 9.1 useTechnicians Hook

**File**: `lib/hooks/useTechnicians.ts`

```typescript
import { useQuery } from '@tanstack/react-query';
import { createSupabaseClient } from '@/lib/supabase/client';
import type { TechnicianProfile } from '@/types/dispatch';

export function useTechnicians(zoneFilter?: string) {
  const supabase = createSupabaseClient();

  return useQuery({
    queryKey: ['technicians', zoneFilter],
    queryFn: async () => {
      let query = supabase
        .from('technician_profiles')
        .select('*')
        .eq('is_active', true)
        .order('display_name', { ascending: true });

      if (zoneFilter && zoneFilter !== 'all') {
        query = query.eq('technician_service_zones.service_zone_id', zoneFilter);
      }

      const { data, error } = await query;

      if (error) throw error;
      return data as TechnicianProfile[];
    }
  });
}
```

### 9.2 useRoutes Hook

**File**: `lib/hooks/useRoutes.ts`

```typescript
import { useQuery } from '@tanstack/react-query';
import { createSupabaseClient } from '@/lib/supabase/client';
import type { RoutePlan } from '@/types/dispatch';

export function useRoutePlans(date: string) {
  const supabase = createSupabaseClient();

  return useQuery({
    queryKey: ['routes', date],
    queryFn: async () => {
      const { data, error } = await supabase
        .from('route_plans')
        .select(`
          *,
          route_stops(*),
          technician_profiles(id, display_name)
        `)
        .eq('date', date)
        .order('created_at', { ascending: false });

      if (error) throw error;
      return data as RoutePlan[];
    }
  });
}
```

### 9.3 useJobs Hook

**File**: `lib/hooks/useJobs.ts`

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { createSupabaseClient } from '@/lib/supabase/client';
import type { DispatchJob } from '@/types/dispatch';

export function useJobs(filters?: {
  zone?: string;
  skill?: string;
  status?: string;
  priority?: string;
}) {
  const supabase = createSupabaseClient();

  return useQuery({
    queryKey: ['jobs', filters],
    queryFn: async () => {
      let query = supabase
        .from('dispatch_jobs')
        .select(`
          *,
          customers(id, name, phone, email),
          customer_locations(address_line1, city, state, postal_code, latitude, longitude),
          job_time_windows(*),
          job_assignments(*)
        `)
        .order('created_at', { ascending: false });

      if (filters?.zone && filters.zone !== 'all') {
        query = query.eq('service_zone_id', filters.zone);
      }
      if (filters?.status && filters.status !== 'all') {
        query = query.eq('status', filters.status);
      }
      if (filters?.priority && filters.priority !== 'all') {
        query = query.eq('priority', filters.priority);
      }

      const { data, error } = await query;

      if (error) throw error;
      return data as DispatchJob[];
    }
  });
}

export function useCreateJob() {
  const supabase = createSupabaseClient();
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (jobData: Partial<DispatchJob>) => {
      const { data, error } = await supabase
        .from('dispatch_jobs')
        .insert(jobData)
        .select()
        .single();

      if (error) throw error;
      return data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['jobs'] });
    }
  });
}

export function useAutoScheduleJob() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ jobId, mode }: { jobId: string; mode: 'propose' | 'commit' }) => {
      const response = await fetch(`/api/dispatch/jobs/${jobId}/auto_schedule`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ mode })
      });

      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.message);
      }

      return response.json();
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['jobs'] });
      queryClient.invalidateQueries({ queryKey: ['assignments'] });
    }
  });
}
```

### 9.2 useOptimization Hook

**File**: `lib/hooks/useOptimization.ts`

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';

export function useOptimizeRoute() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({
      technicianId,
      date,
      strategy,
      mode
    }: {
      technicianId: string;
      date: string;
      strategy: string;
      mode: 'propose' | 'commit';
    }) => {
      const response = await fetch(`/api/dispatch/technicians/${technicianId}/optimize_route`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ date, strategy, mode })
      });

      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.message);
      }

      return response.json();
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['assignments'] });
      queryClient.invalidateQueries({ queryKey: ['routes'] });
    }
  });
}

export function useOptimizeDay() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({
      date,
      strategy,
      mode
    }: {
      date: string;
      strategy: string;
      mode: 'propose' | 'commit';
    }) => {
      const response = await fetch(`/api/dispatch/days/${date}/optimize_all`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ strategy, mode })
      });

      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.message);
      }

      return response.json();
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['assignments'] });
      queryClient.invalidateQueries({ queryKey: ['routes'] });
    }
  });
}
```

### 9.4 API Route Patterns

**File**: `app/api/dispatch/jobs/[id]/auto_schedule/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';

export async function POST(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const supabase = createRouteHandlerClient({ cookies });
  
  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const body = await request.json();
  const { mode } = body;

  // Call Supabase Edge Function
  const { data, error } = await supabase.functions.invoke('dispatch-auto-schedule', {
    body: {
      job_id: params.id,
      mode: mode || 'propose',
    },
  });

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json({ data });
}
```

**File**: `app/api/dispatch/technicians/[id]/optimize_route/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';

export async function POST(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  const supabase = createRouteHandlerClient({ cookies });
  
  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const body = await request.json();
  const { date, strategy, mode } = body;

  const { data, error } = await supabase.functions.invoke('dispatch-optimize-route', {
    body: {
      technician_id: params.id,
      date,
      strategy: strategy || 'balanced',
      mode: mode || 'propose',
    },
  });

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json({ data });
}
```

**File**: `app/api/dispatch/days/[date]/optimize_all/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { createRouteHandlerClient } from '@supabase/auth-helpers-nextjs';
import { cookies } from 'next/headers';

export async function POST(
  request: NextRequest,
  { params }: { params: { date: string } }
) {
  const supabase = createRouteHandlerClient({ cookies });
  
  const {
    data: { user },
  } = await supabase.auth.getUser();

  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const body = await request.json();
  const { strategy, mode } = body;

  const { data, error } = await supabase.functions.invoke('dispatch-optimize-day', {
    body: {
      date: params.date,
      strategy: strategy || 'balanced',
      mode: mode || 'propose',
    },
  });

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json({ data });
}
```

---

## 10. Error Handling

### 10.1 Error Boundaries

**File**: `app/dispatch/error.tsx`

```typescript
'use client';

import { useEffect } from 'react';
import { Button } from '@/components/ui/button';
import { Alert, AlertDescription, AlertTitle } from '@/components/ui/alert';
import { AlertCircle } from 'lucide-react';

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    console.error(error);
  }, [error]);

  return (
    <div className="flex items-center justify-center min-h-screen p-4">
      <Alert variant="destructive" className="max-w-md">
        <AlertCircle className="h-4 w-4" />
        <AlertTitle>Something went wrong</AlertTitle>
        <AlertDescription className="mt-2">
          {error.message || 'An unexpected error occurred'}
        </AlertDescription>
        <Button onClick={reset} className="mt-4">
          Try again
        </Button>
      </Alert>
    </div>
  );
}
```

### 10.2 Toast Notifications

**File**: `components/providers/ToastProvider.tsx`

```typescript
'use client';

import { Toaster } from 'sonner';

export function ToastProvider() {
  return <Toaster />;
}
```

---

## 11. Performance Considerations

### 11.1 Virtualization

For large lists (100+ technicians), consider using `@tanstack/react-virtual`:

```typescript
import { useVirtualizer } from '@tanstack/react-virtual';

// In ScheduleBoard component
const parentRef = useRef<HTMLDivElement>(null);

const virtualizer = useVirtualizer({
  count: technicians.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 80,
  overscan: 5
});
```

### 11.2 Memoization

Use `React.memo` and `useMemo` for expensive computations:

```typescript
const filteredAssignments = useMemo(() => {
  return assignments.filter(/* ... */);
}, [assignments, filters]);
```

### 11.3 Code Splitting

Use dynamic imports for heavy components:

```typescript
const MapView = dynamic(() => import('./components/MapView'), {
  loading: () => <Skeleton className="w-full h-full" />,
  ssr: false
});
```

---

## 12. Implementation Checklist

### Story DISP-051: Dispatch Schedule Board

- [ ] **Schedule Board Page**:
  - [ ] Page component created
  - [ ] Date picker implemented
  - [ ] Zone filter implemented
  - [ ] Layout structure implemented

- [ ] **Schedule Board Component**:
  - [ ] Component created
  - [ ] Technician rows rendered
  - [ ] Time axis rendered
  - [ ] Real-time subscriptions integrated
  - [ ] Loading states implemented
  - [ ] Error states implemented

- [ ] **Drag-and-Drop**:
  - [ ] @hello-pangea/dnd installed
  - [ ] DragDropContext implemented
  - [ ] Draggable assignment blocks
  - [ ] Droppable technician rows
  - [ ] Time calculation on drop
  - [ ] API call on drop end
  - [ ] Optimistic updates

- [ ] **Assignment Block**:
  - [ ] Component created
  - [ ] Position calculation
  - [ ] Width calculation
  - [ ] Color coding by priority
  - [ ] Status badges
  - [ ] Hover effects

### Story DISP-052: Map-Based Dispatch View

- [ ] **Map View Page**:
  - [ ] Page component created
  - [ ] Filters integrated
  - [ ] Layout implemented

- [ ] **Map Component**:
  - [ ] react-map-gl installed
  - [ ] Mapbox token configured
  - [ ] Initial view state set
  - [ ] Job markers rendered
  - [ ] Route overlays rendered
  - [ ] Popups implemented

- [ ] **Filters**:
  - [ ] Zone filter
  - [ ] Status filter
  - [ ] Priority filter
  - [ ] Skill filter (if applicable)

### Story DISP-053: Job Creation UI

- [ ] **Job Form**:
  - [ ] Form component created
  - [ ] react-hook-form integrated
  - [ ] zod validation
  - [ ] All fields implemented
  - [ ] Date pickers for SLA
  - [ ] Submit handler
  - [ ] Error handling

- [ ] **Job Detail Drawer**:
  - [ ] Sheet component from shadcn/ui
  - [ ] Tabs for details/assignments/time-windows
  - [ ] Customer/location display
  - [ ] Assignment list
  - [ ] Time windows display
  - [ ] Auto-schedule button
  - [ ] Manual assign button

### Story DISP-054: Capacity View

- [ ] **Capacity Page**:
  - [ ] Page component created
  - [ ] Date picker
  - [ ] Layout implemented

- [ ] **Capacity View Component**:
  - [ ] Component created
  - [ ] Technician data fetched
  - [ ] Assignment data fetched
  - [ ] Utilization calculation
  - [ ] SLA risk detection
  - [ ] Cards rendered

- [ ] **Capacity Card**:
  - [ ] Card component
  - [ ] Progress bar
  - [ ] Utilization percentage
  - [ ] Warning alerts
  - [ ] SLA risk indicators

### Story DISP-055: Optimization Actions

- [ ] **Optimize Actions Component**:
  - [ ] Component created
  - [ ] Strategy selector
  - [ ] Optimize route button
  - [ ] Optimize day button
  - [ ] API integration
  - [ ] Loading states
  - [ ] Results display

- [ ] **Progress Component**:
  - [ ] Component created
  - [ ] Run status polling
  - [ ] Progress bar
  - [ ] Success/error states
  - [ ] Results display

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 11 – Dispatch Console UI using Next.js and shadcn/ui components. All components are designed with real-time updates, proper error handling, and performance optimizations.

**Next Steps**: After completing Epic 11, proceed to Epic 12 (Customer Portal and Booking Hooks) which will implement portal-safe APIs for customer appointment management.

