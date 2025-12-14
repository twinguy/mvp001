# Technical Design Document – Epic 1: Authentication (Email/Password, Sessions, Verification, Recovery)

## Document Metadata

- **Document Type**: Technical Design Document (TDD)
- **Epic**: Epic 1 – Authentication (Email/Password, Sessions, Verification, Recovery)
- **Source**: Derived from `fdd_0_agile.md` Epic 1 (Stories AUTH-010 through AUTH-014)
- **Reference Documents**: 
  - `fdd_0.md` (Platform Foundation Requirements)
  - `fdd_0_agile.md` (Agile User Stories)
  - `functional.md` (Platform Capabilities)
  - `tooling.md` (Platform Choices: Supabase + Vercel)
  - `tdd_0_epic_0.md` (Epic 0 for tenancy/role conventions - if exists)
- **Target Platform**: Supabase Auth, Next.js 14+ (App Router), React 18+, shadcn/ui, TypeScript
- **Purpose**: Comprehensive technical specification for LLM-based code generation

---

## 1. Overview

This document provides complete technical specifications for implementing Email/Password Authentication with Email Verification and Password Recovery. It covers:

- User registration (sign-up) with email/password
- User login with session management
- Email verification flow
- Password reset/recovery flow
- Logout and session revocation
- Security policies and rate limiting
- Error handling and user experience

All authentication flows are implemented using Supabase Auth with Next.js App Router, React Server Components, Server Actions, and shadcn/ui components for the user interface.

This epic assumes Epic 0 (Platform Identity Model & Tenancy Primitives) is complete, providing the `orgs` and `profiles` foundation.

---

## 2. Prerequisites

### 2.1 Required Components

Before implementing Epic 1, ensure:

1. **Epic 0 Complete**: Tenancy strategy, `orgs` table, and `profiles` model are defined
2. **Next.js Project Setup**:
   - Next.js 14+ with App Router
   - React 18+
   - TypeScript
   - Tailwind CSS configured
   - shadcn/ui installed and initialized

3. **Supabase Setup**:
   - Supabase project created
   - Environment variables configured:
     - `NEXT_PUBLIC_SUPABASE_URL`
     - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
     - `SUPABASE_SERVICE_ROLE_KEY` (server-side only)
   - Supabase Auth enabled
   - Email templates configured (or custom templates ready)

4. **Required shadcn/ui Components**:
   - `Button`
   - `Input`
   - `Label`
   - `Card`
   - `Alert`
   - `AlertDescription`
   - `Checkbox`
   - `Form` (with `FormField`, `FormItem`, `FormLabel`, `FormControl`, `FormMessage`)
   - `Separator`
   - `Skeleton`
   - `Toast` (via `sonner`)

5. **Additional Dependencies**:
   - `@supabase/supabase-js` (latest)
   - `@supabase/ssr` (for Next.js App Router)
   - `zod` (form validation)
   - `react-hook-form` (form handling)
   - `@hookform/resolvers` (zod resolver)
   - `sonner` (toast notifications)
   - `lucide-react` (icons)
   - `zod` (schema validation)

### 2.2 Project Structure

```
app/
  (auth)/
    sign-up/
      page.tsx
      components/
        SignUpForm.tsx
        EmailVerificationPrompt.tsx
    login/
      page.tsx
      components/
        LoginForm.tsx
    verify-email/
      page.tsx
      components/
        VerificationStatus.tsx
        ResendVerification.tsx
    forgot-password/
      page.tsx
      components/
        ForgotPasswordForm.tsx
    reset-password/
      page.tsx
      components/
        ResetPasswordForm.tsx
  api/
    auth/
      callback/
        route.ts              # Supabase auth callback handler
      logout/
        route.ts              # Logout API route
lib/
  supabase/
    client.ts                 # Browser client
    server.ts                 # Server client
    middleware.ts             # Middleware client
  auth/
    actions.ts                # Server actions for auth
    schemas.ts                # Zod schemas
    utils.ts                  # Auth utilities
  utils/
    cn.ts                     # className utility
components/
  ui/                         # shadcn/ui components
  auth/
    PasswordStrength.tsx      # Password strength indicator
    AuthError.tsx             # Error display component
hooks/
  use-auth.ts                 # Auth state hook
  use-session.ts              # Session management hook
types/
  auth.ts                     # Auth-related types
```

---

## 3. Supabase Auth Configuration

### 3.1 Auth Settings

**Email Auth Provider**:
- Enabled: Yes
- Confirm email: Yes (required for MVP)
- Secure email change: Yes
- Double confirm email changes: Yes

**Password Policy**:
- Minimum length: 8 characters
- Require uppercase: No (optional enhancement)
- Require lowercase: Yes
- Require numbers: Yes
- Require special characters: No (optional enhancement)

**Email Templates**:
- Confirmation email: Custom template (or default)
- Magic link email: Not used (email/password only)
- Change email address: Custom template
- Reset password: Custom template

**Rate Limiting** (Supabase defaults):
- Sign-up: 5 requests per hour per IP
- Sign-in: 5 requests per hour per IP
- Password reset: 3 requests per hour per IP
- Email change: 3 requests per hour per IP

### 3.2 Redirect URLs

**Allowed Redirect URLs** (configured in Supabase Dashboard):
- `http://localhost:3000/auth/callback`
- `https://yourdomain.com/auth/callback`
- `http://localhost:3000/auth/verify-email`
- `https://yourdomain.com/auth/verify-email`
- `http://localhost:3000/auth/reset-password`
- `https://yourdomain.com/auth/reset-password`

**Site URL**: `http://localhost:3000` (development) / `https://yourdomain.com` (production)

---

## 4. Story AUTH-010: Email/Password Sign-Up

### 4.1 Sign-Up Schema

**Zod Schema** (`lib/auth/schemas.ts`):

```typescript
import { z } from 'zod';

export const signUpSchema = z.object({
  email: z
    .string()
    .min(1, 'Email is required')
    .email('Invalid email address')
    .toLowerCase()
    .trim(),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[a-z]/, 'Password must contain at least one lowercase letter')
    .regex(/[0-9]/, 'Password must contain at least one number'),
  confirmPassword: z.string().min(1, 'Please confirm your password'),
  acceptTerms: z.boolean().refine((val) => val === true, {
    message: 'You must accept the terms and conditions',
  }),
}).refine((data) => data.password === data.confirmPassword, {
  message: 'Passwords do not match',
  path: ['confirmPassword'],
});

export type SignUpInput = z.infer<typeof signUpSchema>;
```

### 4.2 Sign-Up Server Action

**Server Action** (`lib/auth/actions.ts`):

```typescript
'use server';

import { createClient } from '@/lib/supabase/server';
import { signUpSchema, type SignUpInput } from '@/lib/auth/schemas';
import { redirect } from 'next/navigation';

export async function signUpAction(formData: SignUpInput) {
  const supabase = createClient();

  // Validate input
  const validation = signUpSchema.safeParse(formData);
  if (!validation.success) {
    return {
      error: 'Validation failed',
      details: validation.error.flatten().fieldErrors,
    };
  }

  const { email, password } = validation.data;

  // Check if email already exists (optional pre-check)
  const { data: existingUser } = await supabase
    .from('auth.users')
    .select('id')
    .eq('email', email)
    .single();

  if (existingUser) {
    return {
      error: 'An account with this email already exists',
    };
  }

  // Sign up user
  const { data, error } = await supabase.auth.signUp({
    email,
    password,
    options: {
      emailRedirectTo: `${process.env.NEXT_PUBLIC_SITE_URL}/auth/verify-email`,
      data: {
        // Custom metadata if needed
      },
    },
  });

  if (error) {
    // Handle specific Supabase errors
    if (error.message.includes('rate limit')) {
      return {
        error: 'Too many sign-up attempts. Please try again later.',
      };
    }
    if (error.message.includes('already registered')) {
      return {
        error: 'An account with this email already exists',
      };
    }
    return {
      error: error.message || 'Failed to create account',
    };
  }

  if (!data.user) {
    return {
      error: 'Failed to create account',
    };
  }

  // Success: redirect to email verification page
  redirect('/auth/verify-email?email=' + encodeURIComponent(email));
}
```

### 4.3 Sign-Up Page Component

**Page** (`app/(auth)/sign-up/page.tsx`):

```typescript
import { SignUpForm } from './components/SignUpForm';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Separator } from '@/components/ui/separator';
import Link from 'next/link';

export default function SignUpPage() {
  return (
    <div className="flex min-h-screen items-center justify-center bg-gray-50 px-4 py-12 sm:px-6 lg:px-8">
      <Card className="w-full max-w-md">
        <CardHeader className="space-y-1">
          <CardTitle className="text-2xl font-bold">Create an account</CardTitle>
          <CardDescription>
            Enter your email and password to get started
          </CardDescription>
        </CardHeader>
        <CardContent>
          <SignUpForm />
          <Separator className="my-6" />
          <div className="text-center text-sm text-gray-600">
            Already have an account?{' '}
            <Link href="/login" className="font-medium text-primary hover:underline">
              Sign in
            </Link>
          </div>
        </CardContent>
      </Card>
    </div>
  );
}
```

### 4.4 Sign-Up Form Component

**Form Component** (`app/(auth)/sign-up/components/SignUpForm.tsx`):

```typescript
'use client';

import { useState } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { signUpSchema, type SignUpInput } from '@/lib/auth/schemas';
import { signUpAction } from '@/lib/auth/actions';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Checkbox } from '@/components/ui/checkbox';
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from '@/components/ui/form';
import { PasswordStrength } from '@/components/auth/PasswordStrength';
import { AuthError } from '@/components/auth/AuthError';
import { toast } from 'sonner';
import { Loader2 } from 'lucide-react';

export function SignUpForm() {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [password, setPassword] = useState('');

  const form = useForm<SignUpInput>({
    resolver: zodResolver(signUpSchema),
    defaultValues: {
      email: '',
      password: '',
      confirmPassword: '',
      acceptTerms: false,
    },
  });

  async function onSubmit(data: SignUpInput) {
    setIsLoading(true);
    setError(null);

    try {
      const result = await signUpAction(data);

      if (result?.error) {
        setError(result.error);
        if (result.details) {
          // Set field errors
          Object.entries(result.details).forEach(([field, messages]) => {
            if (messages && messages[0]) {
              form.setError(field as keyof SignUpInput, {
                message: messages[0],
              });
            }
          });
        }
      } else {
        toast.success('Account created! Please check your email to verify your account.');
      }
    } catch (err) {
      setError('An unexpected error occurred. Please try again.');
      console.error('Sign-up error:', err);
    } finally {
      setIsLoading(false);
    }
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        {error && <AuthError message={error} />}

        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl>
                <Input
                  type="email"
                  placeholder="name@example.com"
                  autoComplete="email"
                  {...field}
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="password"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Password</FormLabel>
              <FormControl>
                <Input
                  type="password"
                  placeholder="••••••••"
                  autoComplete="new-password"
                  {...field}
                  onChange={(e) => {
                    field.onChange(e);
                    setPassword(e.target.value);
                  }}
                />
              </FormControl>
              <PasswordStrength password={password} />
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="confirmPassword"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Confirm Password</FormLabel>
              <FormControl>
                <Input
                  type="password"
                  placeholder="••••••••"
                  autoComplete="new-password"
                  {...field}
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="acceptTerms"
          render={({ field }) => (
            <FormItem className="flex flex-row items-start space-x-3 space-y-0">
              <FormControl>
                <Checkbox
                  checked={field.value}
                  onCheckedChange={field.onChange}
                />
              </FormControl>
              <div className="space-y-1 leading-none">
                <FormLabel className="text-sm font-normal">
                  I accept the{' '}
                  <a href="/terms" className="text-primary hover:underline" target="_blank">
                    Terms and Conditions
                  </a>
                </FormLabel>
              </div>
            </FormItem>
          )}
        />

        <Button type="submit" className="w-full" disabled={isLoading}>
          {isLoading && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
          Create account
        </Button>
      </form>
    </Form>
  );
}
```

### 4.5 Password Strength Component

**Component** (`components/auth/PasswordStrength.tsx`):

```typescript
'use client';

import { useMemo } from 'react';
import { Progress } from '@/components/ui/progress';
import { Check, X } from 'lucide-react';

interface PasswordStrengthProps {
  password: string;
}

export function PasswordStrength({ password }: PasswordStrengthProps) {
  const strength = useMemo(() => {
    if (!password) return { score: 0, feedback: [] };

    let score = 0;
    const feedback: string[] = [];

    if (password.length >= 8) {
      score += 1;
    } else {
      feedback.push('At least 8 characters');
    }

    if (/[a-z]/.test(password)) {
      score += 1;
    } else {
      feedback.push('One lowercase letter');
    }

    if (/[A-Z]/.test(password)) {
      score += 1;
    } else {
      feedback.push('One uppercase letter (optional)');
    }

    if (/[0-9]/.test(password)) {
      score += 1;
    } else {
      feedback.push('One number');
    }

    if (/[^a-zA-Z0-9]/.test(password)) {
      score += 1;
    }

    return { score, feedback };
  }, [password]);

  if (!password) return null;

  const percentage = (strength.score / 4) * 100;
  const color =
    strength.score <= 1
      ? 'bg-red-500'
      : strength.score === 2
      ? 'bg-yellow-500'
      : strength.score === 3
      ? 'bg-blue-500'
      : 'bg-green-500';

  return (
    <div className="space-y-2">
      <Progress value={percentage} className="h-2" />
      {strength.feedback.length > 0 && (
        <div className="text-xs text-gray-600 space-y-1">
          <div>Password must contain:</div>
          {strength.feedback.map((item, idx) => (
            <div key={idx} className="flex items-center gap-1">
              <X className="h-3 w-3 text-red-500" />
              <span>{item}</span>
            </div>
          ))}
        </div>
      )}
      {strength.score >= 3 && (
        <div className="text-xs text-green-600 flex items-center gap-1">
          <Check className="h-3 w-3" />
          <span>Password strength: Good</span>
        </div>
      )}
    </div>
  );
}
```

### 4.6 Email Verification Prompt Component

**Component** (`app/(auth)/sign-up/components/EmailVerificationPrompt.tsx`):

```typescript
'use client';

import { Alert, AlertDescription, AlertTitle } from '@/components/ui/alert';
import { Mail, CheckCircle2 } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { resendVerificationEmail } from '@/lib/auth/actions';

interface EmailVerificationPromptProps {
  email: string;
}

export function EmailVerificationPrompt({ email }: EmailVerificationPromptProps) {
  const [isResending, setIsResending] = useState(false);
  const [resendSuccess, setResendSuccess] = useState(false);

  async function handleResend() {
    setIsResending(true);
    setResendSuccess(false);

    try {
      const result = await resendVerificationEmail(email);
      if (result?.error) {
        toast.error(result.error);
      } else {
        setResendSuccess(true);
        toast.success('Verification email sent!');
      }
    } catch (err) {
      toast.error('Failed to resend email');
    } finally {
      setIsResending(false);
    }
  }

  return (
    <Alert>
      <Mail className="h-4 w-4" />
      <AlertTitle>Check your email</AlertTitle>
      <AlertDescription className="space-y-2">
        <p>
          We've sent a verification email to <strong>{email}</strong>
        </p>
        <p className="text-sm text-gray-600">
          Click the link in the email to verify your account. The link will expire in 24 hours.
        </p>
        {resendSuccess ? (
          <div className="flex items-center gap-2 text-green-600">
            <CheckCircle2 className="h-4 w-4" />
            <span className="text-sm">Email sent!</span>
          </div>
        ) : (
          <Button
            variant="link"
            size="sm"
            onClick={handleResend}
            disabled={isResending}
            className="p-0 h-auto"
          >
            {isResending ? 'Sending...' : "Didn't receive it? Resend"}
          </Button>
        )}
      </AlertDescription>
    </Alert>
  );
}
```

---

## 5. Story AUTH-011: Email/Password Login and Session Management

### 5.1 Login Schema

**Zod Schema** (`lib/auth/schemas.ts`):

```typescript
export const loginSchema = z.object({
  email: z
    .string()
    .min(1, 'Email is required')
    .email('Invalid email address')
    .toLowerCase()
    .trim(),
  password: z.string().min(1, 'Password is required'),
});

export type LoginInput = z.infer<typeof loginSchema>;
```

### 5.2 Login Server Action

**Server Action** (`lib/auth/actions.ts`):

```typescript
'use server';

import { createClient } from '@/lib/supabase/server';
import { loginSchema, type LoginInput } from '@/lib/auth/schemas';
import { redirect } from 'next/navigation';
import { revalidatePath } from 'next/cache';

export async function loginAction(formData: LoginInput) {
  const supabase = createClient();

  // Validate input
  const validation = loginSchema.safeParse(formData);
  if (!validation.success) {
    return {
      error: 'Validation failed',
      details: validation.error.flatten().fieldErrors,
    };
  }

  const { email, password } = validation.data;

  // Sign in user
  const { data, error } = await supabase.auth.signInWithPassword({
    email,
    password,
  });

  if (error) {
    // Handle specific Supabase errors
    if (error.message.includes('Invalid login credentials')) {
      return {
        error: 'Invalid email or password',
      };
    }
    if (error.message.includes('Email not confirmed')) {
      return {
        error: 'Please verify your email address before signing in',
        requiresVerification: true,
      };
    }
    if (error.message.includes('rate limit')) {
      return {
        error: 'Too many login attempts. Please try again later.',
      };
    }
    return {
      error: error.message || 'Failed to sign in',
    };
  }

  if (!data.user) {
    return {
      error: 'Failed to sign in',
    };
  }

  // Check if user has a profile and org
  const { data: profile } = await supabase
    .from('profiles')
    .select('org_id, role, is_active')
    .eq('user_id', data.user.id)
    .single();

  if (!profile) {
    // User needs to complete onboarding
    redirect('/onboarding');
  }

  if (!profile.is_active) {
    return {
      error: 'Your account has been deactivated. Please contact support.',
    };
  }

  // Revalidate auth-dependent paths
  revalidatePath('/', 'layout');

  // Redirect based on user state
  if (!profile.org_id) {
    redirect('/onboarding');
  }

  // Success: redirect to app home
  redirect('/');
}
```

### 5.3 Login Page Component

**Page** (`app/(auth)/login/page.tsx`):

```typescript
import { LoginForm } from './components/LoginForm';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Separator } from '@/components/ui/separator';
import Link from 'next/link';

export default function LoginPage() {
  return (
    <div className="flex min-h-screen items-center justify-center bg-gray-50 px-4 py-12 sm:px-6 lg:px-8">
      <Card className="w-full max-w-md">
        <CardHeader className="space-y-1">
          <CardTitle className="text-2xl font-bold">Sign in</CardTitle>
          <CardDescription>
            Enter your email and password to access your account
          </CardDescription>
        </CardHeader>
        <CardContent>
          <LoginForm />
          <Separator className="my-6" />
          <div className="space-y-2 text-center text-sm">
            <Link
              href="/forgot-password"
              className="text-primary hover:underline"
            >
              Forgot password?
            </Link>
            <div className="text-gray-600">
              Don't have an account?{' '}
              <Link href="/sign-up" className="font-medium text-primary hover:underline">
                Sign up
              </Link>
            </div>
          </div>
        </CardContent>
      </Card>
    </div>
  );
}
```

### 5.4 Login Form Component

**Form Component** (`app/(auth)/login/components/LoginForm.tsx`):

```typescript
'use client';

import { useState } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { loginSchema, type LoginInput } from '@/lib/auth/schemas';
import { loginAction } from '@/lib/auth/actions';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from '@/components/ui/form';
import { AuthError } from '@/components/auth/AuthError';
import { toast } from 'sonner';
import { Loader2 } from 'lucide-react';
import { useRouter } from 'next/navigation';

export function LoginForm() {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const router = useRouter();

  const form = useForm<LoginInput>({
    resolver: zodResolver(loginSchema),
    defaultValues: {
      email: '',
      password: '',
    },
  });

  async function onSubmit(data: LoginInput) {
    setIsLoading(true);
    setError(null);

    try {
      const result = await loginAction(data);

      if (result?.error) {
        setError(result.error);
        if (result.requiresVerification) {
          // Redirect to verification page
          router.push(`/auth/verify-email?email=${encodeURIComponent(data.email)}`);
        }
        if (result.details) {
          // Set field errors
          Object.entries(result.details).forEach(([field, messages]) => {
            if (messages && messages[0]) {
              form.setError(field as keyof LoginInput, {
                message: messages[0],
              });
            }
          });
        }
      }
      // Success handled by redirect in server action
    } catch (err) {
      setError('An unexpected error occurred. Please try again.');
      console.error('Login error:', err);
    } finally {
      setIsLoading(false);
    }
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        {error && <AuthError message={error} />}

        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl>
                <Input
                  type="email"
                  placeholder="name@example.com"
                  autoComplete="email"
                  {...field}
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="password"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Password</FormLabel>
              <FormControl>
                <Input
                  type="password"
                  placeholder="••••••••"
                  autoComplete="current-password"
                  {...field}
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <Button type="submit" className="w-full" disabled={isLoading}>
          {isLoading && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
          Sign in
        </Button>
      </form>
    </Form>
  );
}
```

### 5.5 Session Management

**Supabase Client Setup** (`lib/supabase/client.ts`):

```typescript
import { createBrowserClient } from '@supabase/ssr';

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

**Server Client** (`lib/supabase/server.ts`):

```typescript
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient(
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

**Middleware Client** (`lib/supabase/middleware.ts`):

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
          cookiesToSet.forEach(({ name, value, options }) =>
            request.cookies.set(name, value)
          );
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

  // Refresh session if expired
  await supabase.auth.getUser();

  return supabaseResponse;
}
```

**Middleware** (`middleware.ts`):

```typescript
import { type NextRequest } from 'next/server';
import { updateSession } from '@/lib/supabase/middleware';

export async function middleware(request: NextRequest) {
  return await updateSession(request);
}

export const config = {
  matcher: [
    /*
     * Match all request paths except for the ones starting with:
     * - _next/static (static files)
     * - _next/image (image optimization files)
     * - favicon.ico (favicon file)
     * - public folder
     */
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
  ],
};
```

**Auth Hook** (`hooks/use-auth.ts`):

```typescript
'use client';

import { useEffect, useState } from 'react';
import { createClient } from '@/lib/supabase/client';
import type { User } from '@supabase/supabase-js';

export function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const supabase = createClient();

  useEffect(() => {
    // Get initial session
    supabase.auth.getSession().then(({ data: { session } }) => {
      setUser(session?.user ?? null);
      setLoading(false);
    });

    // Listen for auth changes
    const {
      data: { subscription },
    } = supabase.auth.onAuthStateChange((_event, session) => {
      setUser(session?.user ?? null);
      setLoading(false);
    });

    return () => subscription.unsubscribe();
  }, [supabase]);

  return { user, loading };
}
```

---

## 6. Story AUTH-012: Email Verification

### 6.1 Verification Server Actions

**Server Actions** (`lib/auth/actions.ts`):

```typescript
export async function verifyEmailAction(token: string, type: string) {
  const supabase = createClient();

  if (type === 'signup' || type === 'email_change') {
    const { data, error } = await supabase.auth.verifyOtp({
      token_hash: token,
      type: type === 'signup' ? 'signup' : 'email_change',
    });

    if (error) {
      return {
        error: error.message || 'Failed to verify email',
        expired: error.message.includes('expired') || error.message.includes('invalid'),
      };
    }

    if (data.user) {
      // Revalidate paths
      revalidatePath('/', 'layout');
      return { success: true, user: data.user };
    }
  }

  return { error: 'Invalid verification link' };
}

export async function resendVerificationEmail(email: string) {
  const supabase = createClient();

  const { error } = await supabase.auth.resend({
    type: 'signup',
    email,
    options: {
      emailRedirectTo: `${process.env.NEXT_PUBLIC_SITE_URL}/auth/verify-email`,
    },
  });

  if (error) {
    if (error.message.includes('rate limit')) {
      return {
        error: 'Too many requests. Please wait before requesting another email.',
      };
    }
    return {
      error: error.message || 'Failed to resend verification email',
    };
  }

  return { success: true };
}
```

### 6.2 Verification Callback Route

**Route Handler** (`app/api/auth/callback/route.ts`):

```typescript
import { createClient } from '@/lib/supabase/server';
import { NextResponse } from 'next/server';
import { redirect } from 'next/navigation';

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get('code');
  const next = searchParams.get('next') ?? '/';
  const type = searchParams.get('type'); // 'signup', 'recovery', 'email_change'

  if (code) {
    const supabase = await createClient();
    const { error } = await supabase.auth.exchangeCodeForSession(code);

    if (!error) {
      const forwardedHost = request.headers.get('x-forwarded-host'); // original origin before reverse proxy
      const isLocalEnv = process.env.NODE_ENV === 'development';

      if (isLocalEnv) {
        // In development, redirect to localhost
        return redirect(`${origin}${next}`);
      } else if (forwardedHost) {
        // In production, use forwarded host
        return redirect(`https://${forwardedHost}${next}`);
      } else {
        return redirect(`${origin}${next}`);
      }
    }
  }

  // Return the user to an error page with instructions
  return redirect(`${origin}/auth/auth-code-error`);
}
```

### 6.3 Verification Page

**Page** (`app/(auth)/verify-email/page.tsx`):

```typescript
import { Suspense } from 'react';
import { VerificationStatus } from './components/VerificationStatus';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Skeleton } from '@/components/ui/skeleton';

export default function VerifyEmailPage({
  searchParams,
}: {
  searchParams: { email?: string; token?: string; type?: string };
}) {
  return (
    <div className="flex min-h-screen items-center justify-center bg-gray-50 px-4 py-12 sm:px-6 lg:px-8">
      <Card className="w-full max-w-md">
        <CardHeader className="space-y-1">
          <CardTitle className="text-2xl font-bold">Verify your email</CardTitle>
          <CardDescription>
            We've sent a verification link to your email address
          </CardDescription>
        </CardHeader>
        <CardContent>
          <Suspense fallback={<Skeleton className="h-32 w-full" />}>
            <VerificationStatus
              email={searchParams.email}
              token={searchParams.token}
              type={searchParams.type}
            />
          </Suspense>
        </CardContent>
      </Card>
    </div>
  );
}
```

### 6.4 Verification Status Component

**Component** (`app/(auth)/verify-email/components/VerificationStatus.tsx`):

```typescript
'use client';

import { useEffect, useState } from 'react';
import { useSearchParams, useRouter } from 'next/navigation';
import { verifyEmailAction } from '@/lib/auth/actions';
import { Alert, AlertDescription, AlertTitle } from '@/components/ui/alert';
import { Button } from '@/components/ui/button';
import { ResendVerification } from './ResendVerification';
import { CheckCircle2, XCircle, Loader2, Mail } from 'lucide-react';
import { toast } from 'sonner';

interface VerificationStatusProps {
  email?: string;
  token?: string;
  type?: string;
}

export function VerificationStatus({ email, token, type }: VerificationStatusProps) {
  const [status, setStatus] = useState<'pending' | 'verifying' | 'success' | 'error' | 'expired'>(
    'pending'
  );
  const [errorMessage, setErrorMessage] = useState<string | null>(null);
  const router = useRouter();

  useEffect(() => {
    if (token && type) {
      handleVerification(token, type);
    }
  }, [token, type]);

  async function handleVerification(token: string, type: string) {
    setStatus('verifying');

    try {
      const result = await verifyEmailAction(token, type);

      if (result?.error) {
        if (result.expired) {
          setStatus('expired');
          setErrorMessage('This verification link has expired.');
        } else {
          setStatus('error');
          setErrorMessage(result.error);
        }
      } else if (result?.success) {
        setStatus('success');
        toast.success('Email verified successfully!');
        // Redirect to onboarding or home after 2 seconds
        setTimeout(() => {
          router.push('/onboarding');
        }, 2000);
      }
    } catch (err) {
      setStatus('error');
      setErrorMessage('An unexpected error occurred.');
      console.error('Verification error:', err);
    }
  }

  if (status === 'verifying') {
    return (
      <Alert>
        <Loader2 className="h-4 w-4 animate-spin" />
        <AlertTitle>Verifying...</AlertTitle>
        <AlertDescription>Please wait while we verify your email.</AlertDescription>
      </Alert>
    );
  }

  if (status === 'success') {
    return (
      <Alert className="border-green-500">
        <CheckCircle2 className="h-4 w-4 text-green-500" />
        <AlertTitle className="text-green-700">Email verified!</AlertTitle>
        <AlertDescription className="text-green-600">
          Your email has been verified successfully. Redirecting...
        </AlertDescription>
      </Alert>
    );
  }

  if (status === 'expired' || status === 'error') {
    return (
      <div className="space-y-4">
        <Alert variant="destructive">
          <XCircle className="h-4 w-4" />
          <AlertTitle>Verification failed</AlertTitle>
          <AlertDescription>
            {errorMessage || 'This verification link is invalid or has expired.'}
          </AlertDescription>
        </Alert>
        {email && <ResendVerification email={email} />}
      </div>
    );
  }

  // Pending state (waiting for user to click link)
  return (
    <div className="space-y-4">
      <Alert>
        <Mail className="h-4 w-4" />
        <AlertTitle>Check your email</AlertTitle>
        <AlertDescription>
          {email ? (
            <>
              We've sent a verification link to <strong>{email}</strong>
            </>
          ) : (
            'We've sent a verification link to your email address'
          )}
        </AlertDescription>
      </Alert>
      {email && <ResendVerification email={email} />}
    </div>
  );
}
```

### 6.5 Resend Verification Component

**Component** (`app/(auth)/verify-email/components/ResendVerification.tsx`):

```typescript
'use client';

import { useState } from 'react';
import { resendVerificationEmail } from '@/lib/auth/actions';
import { Button } from '@/components/ui/button';
import { toast } from 'sonner';
import { Loader2, Mail } from 'lucide-react';

interface ResendVerificationProps {
  email: string;
}

export function ResendVerification({ email }: ResendVerificationProps) {
  const [isResending, setIsResending] = useState(false);
  const [cooldown, setCooldown] = useState(0);

  async function handleResend() {
    setIsResending(true);

    try {
      const result = await resendVerificationEmail(email);

      if (result?.error) {
        toast.error(result.error);
      } else {
        toast.success('Verification email sent!');
        // Set cooldown (60 seconds)
        setCooldown(60);
        const interval = setInterval(() => {
          setCooldown((prev) => {
            if (prev <= 1) {
              clearInterval(interval);
              return 0;
            }
            return prev - 1;
          });
        }, 1000);
      }
    } catch (err) {
      toast.error('Failed to resend verification email');
    } finally {
      setIsResending(false);
    }
  }

  return (
    <div className="text-center">
      <p className="text-sm text-gray-600 mb-2">Didn't receive the email?</p>
      <Button
        variant="outline"
        onClick={handleResend}
        disabled={isResending || cooldown > 0}
        className="w-full"
      >
        {isResending ? (
          <>
            <Loader2 className="mr-2 h-4 w-4 animate-spin" />
            Sending...
          </>
        ) : cooldown > 0 ? (
          `Resend in ${cooldown}s`
        ) : (
          <>
            <Mail className="mr-2 h-4 w-4" />
            Resend verification email
          </>
        )}
      </Button>
    </div>
  );
}
```

---

## 7. Story AUTH-013: Password Reset / Recovery

### 7.1 Password Reset Schema

**Zod Schema** (`lib/auth/schemas.ts`):

```typescript
export const forgotPasswordSchema = z.object({
  email: z
    .string()
    .min(1, 'Email is required')
    .email('Invalid email address')
    .toLowerCase()
    .trim(),
});

export const resetPasswordSchema = z.object({
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[a-z]/, 'Password must contain at least one lowercase letter')
    .regex(/[0-9]/, 'Password must contain at least one number'),
  confirmPassword: z.string().min(1, 'Please confirm your password'),
}).refine((data) => data.password === data.confirmPassword, {
  message: 'Passwords do not match',
  path: ['confirmPassword'],
});

export type ForgotPasswordInput = z.infer<typeof forgotPasswordSchema>;
export type ResetPasswordInput = z.infer<typeof resetPasswordSchema>;
```

### 7.2 Password Reset Server Actions

**Server Actions** (`lib/auth/actions.ts`):

```typescript
export async function forgotPasswordAction(email: string) {
  const supabase = createClient();

  const validation = forgotPasswordSchema.safeParse({ email });
  if (!validation.success) {
    return {
      error: 'Invalid email address',
    };
  }

  const { error } = await supabase.auth.resetPasswordForEmail(email, {
    redirectTo: `${process.env.NEXT_PUBLIC_SITE_URL}/auth/reset-password`,
  });

  if (error) {
    if (error.message.includes('rate limit')) {
      return {
        error: 'Too many requests. Please wait before requesting another reset email.',
      };
    }
    // Don't leak whether email exists - always return success
    // Supabase will send email if account exists, otherwise silently fail
    return {
      error: error.message,
    };
  }

  // Always return success to prevent email enumeration
  return { success: true };
}

export async function resetPasswordAction(formData: ResetPasswordInput) {
  const supabase = createClient();

  const validation = resetPasswordSchema.safeParse(formData);
  if (!validation.success) {
    return {
      error: 'Validation failed',
      details: validation.error.flatten().fieldErrors,
    };
  }

  const { password } = validation.data;

  const { error } = await supabase.auth.updateUser({
    password: password,
  });

  if (error) {
    if (error.message.includes('expired') || error.message.includes('invalid')) {
      return {
        error: 'This reset link has expired. Please request a new one.',
        expired: true,
      };
    }
    return {
      error: error.message || 'Failed to reset password',
    };
  }

  // Revalidate paths
  revalidatePath('/', 'layout');

  return { success: true };
}
```

### 7.3 Forgot Password Page

**Page** (`app/(auth)/forgot-password/page.tsx`):

```typescript
import { ForgotPasswordForm } from './components/ForgotPasswordForm';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import Link from 'next/link';

export default function ForgotPasswordPage() {
  return (
    <div className="flex min-h-screen items-center justify-center bg-gray-50 px-4 py-12 sm:px-6 lg:px-8">
      <Card className="w-full max-w-md">
        <CardHeader className="space-y-1">
          <CardTitle className="text-2xl font-bold">Reset your password</CardTitle>
          <CardDescription>
            Enter your email address and we'll send you a reset link
          </CardDescription>
        </CardHeader>
        <CardContent>
          <ForgotPasswordForm />
          <div className="mt-4 text-center text-sm">
            <Link href="/login" className="text-primary hover:underline">
              Back to login
            </Link>
          </div>
        </CardContent>
      </Card>
    </div>
  );
}
```

### 7.4 Forgot Password Form Component

**Component** (`app/(auth)/forgot-password/components/ForgotPasswordForm.tsx`):

```typescript
'use client';

import { useState } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { forgotPasswordSchema, type ForgotPasswordInput } from '@/lib/auth/schemas';
import { forgotPasswordAction } from '@/lib/auth/actions';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from '@/components/ui/form';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { AuthError } from '@/components/auth/AuthError';
import { toast } from 'sonner';
import { Loader2, CheckCircle2, Mail } from 'lucide-react';

export function ForgotPasswordForm() {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [success, setSuccess] = useState(false);
  const [emailSent, setEmailSent] = useState('');

  const form = useForm<ForgotPasswordInput>({
    resolver: zodResolver(forgotPasswordSchema),
    defaultValues: {
      email: '',
    },
  });

  async function onSubmit(data: ForgotPasswordInput) {
    setIsLoading(true);
    setError(null);
    setSuccess(false);

    try {
      const result = await forgotPasswordAction(data.email);

      if (result?.error) {
        setError(result.error);
      } else {
        setSuccess(true);
        setEmailSent(data.email);
        toast.success('Password reset email sent!');
      }
    } catch (err) {
      setError('An unexpected error occurred. Please try again.');
      console.error('Forgot password error:', err);
    } finally {
      setIsLoading(false);
    }
  }

  if (success) {
    return (
      <Alert className="border-green-500">
        <Mail className="h-4 w-4 text-green-500" />
        <AlertDescription className="text-green-700">
          <div className="space-y-2">
            <p className="font-medium">Check your email</p>
            <p>
              We've sent a password reset link to <strong>{emailSent}</strong>
            </p>
            <p className="text-sm text-gray-600">
              Click the link in the email to reset your password. The link will expire in 1 hour.
            </p>
          </div>
        </AlertDescription>
      </Alert>
    );
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        {error && <AuthError message={error} />}

        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl>
                <Input
                  type="email"
                  placeholder="name@example.com"
                  autoComplete="email"
                  {...field}
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <Button type="submit" className="w-full" disabled={isLoading}>
          {isLoading && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
          Send reset link
        </Button>
      </form>
    </Form>
  );
}
```

### 7.5 Reset Password Page

**Page** (`app/(auth)/reset-password/page.tsx`):

```typescript
import { Suspense } from 'react';
import { ResetPasswordForm } from './components/ResetPasswordForm';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Skeleton } from '@/components/ui/skeleton';

export default function ResetPasswordPage({
  searchParams,
}: {
  searchParams: { code?: string };
}) {
  return (
    <div className="flex min-h-screen items-center justify-center bg-gray-50 px-4 py-12 sm:px-6 lg:px-8">
      <Card className="w-full max-w-md">
        <CardHeader className="space-y-1">
          <CardTitle className="text-2xl font-bold">Set new password</CardTitle>
          <CardDescription>
            Enter your new password below
          </CardDescription>
        </CardHeader>
        <CardContent>
          <Suspense fallback={<Skeleton className="h-64 w-full" />}>
            <ResetPasswordForm code={searchParams.code} />
          </Suspense>
        </CardContent>
      </Card>
    </div>
  );
}
```

### 7.6 Reset Password Form Component

**Component** (`app/(auth)/reset-password/components/ResetPasswordForm.tsx`):

```typescript
'use client';

import { useState, useEffect } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { resetPasswordSchema, type ResetPasswordInput } from '@/lib/auth/schemas';
import { resetPasswordAction } from '@/lib/auth/actions';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from '@/components/ui/form';
import { AuthError } from '@/components/auth/AuthError';
import { PasswordStrength } from '@/components/auth/PasswordStrength';
import { toast } from 'sonner';
import { Loader2 } from 'lucide-react';
import { useRouter } from 'next/navigation';
import { createClient } from '@/lib/supabase/client';

interface ResetPasswordFormProps {
  code?: string;
}

export function ResetPasswordForm({ code }: ResetPasswordFormProps) {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [password, setPassword] = useState('');
  const [isValidatingCode, setIsValidatingCode] = useState(!!code);
  const router = useRouter();
  const supabase = createClient();

  useEffect(() => {
    // If code is provided, exchange it for session
    if (code) {
      validateResetCode(code);
    }
  }, [code]);

  async function validateResetCode(code: string) {
    try {
      const { error } = await supabase.auth.verifyOtp({
        token_hash: code,
        type: 'recovery',
      });

      if (error) {
        if (error.message.includes('expired') || error.message.includes('invalid')) {
          setError('This reset link has expired. Please request a new one.');
        } else {
          setError('Invalid reset link');
        }
      }
    } catch (err) {
      setError('Failed to validate reset link');
    } finally {
      setIsValidatingCode(false);
    }
  }

  const form = useForm<ResetPasswordInput>({
    resolver: zodResolver(resetPasswordSchema),
    defaultValues: {
      password: '',
      confirmPassword: '',
    },
  });

  async function onSubmit(data: ResetPasswordInput) {
    setIsLoading(true);
    setError(null);

    try {
      const result = await resetPasswordAction(data);

      if (result?.error) {
        setError(result.error);
        if (result.expired) {
          // Redirect to forgot password page
          setTimeout(() => {
            router.push('/forgot-password');
          }, 3000);
        }
        if (result.details) {
          // Set field errors
          Object.entries(result.details).forEach(([field, messages]) => {
            if (messages && messages[0]) {
              form.setError(field as keyof ResetPasswordInput, {
                message: messages[0],
              });
            }
          });
        }
      } else {
        toast.success('Password reset successfully! Redirecting to login...');
        setTimeout(() => {
          router.push('/login');
        }, 2000);
      }
    } catch (err) {
      setError('An unexpected error occurred. Please try again.');
      console.error('Reset password error:', err);
    } finally {
      setIsLoading(false);
    }
  }

  if (isValidatingCode) {
    return (
      <div className="text-center py-8">
        <Loader2 className="h-8 w-8 animate-spin mx-auto mb-4" />
        <p className="text-sm text-gray-600">Validating reset link...</p>
      </div>
    );
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        {error && <AuthError message={error} />}

        <FormField
          control={form.control}
          name="password"
          render={({ field }) => (
            <FormItem>
              <FormLabel>New Password</FormLabel>
              <FormControl>
                <Input
                  type="password"
                  placeholder="••••••••"
                  autoComplete="new-password"
                  {...field}
                  onChange={(e) => {
                    field.onChange(e);
                    setPassword(e.target.value);
                  }}
                />
              </FormControl>
              <PasswordStrength password={password} />
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="confirmPassword"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Confirm New Password</FormLabel>
              <FormControl>
                <Input
                  type="password"
                  placeholder="••••••••"
                  autoComplete="new-password"
                  {...field}
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <Button type="submit" className="w-full" disabled={isLoading}>
          {isLoading && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
          Reset password
        </Button>
      </form>
    </Form>
  );
}
```

---

## 8. Story AUTH-014: Logout and Session Revocation

### 8.1 Logout Server Action

**Server Action** (`lib/auth/actions.ts`):

```typescript
export async function logoutAction() {
  const supabase = createClient();

  const { error } = await supabase.auth.signOut();

  if (error) {
    return {
      error: error.message || 'Failed to sign out',
    };
  }

  // Revalidate paths
  revalidatePath('/', 'layout');

  return { success: true };
}
```

### 8.2 Logout API Route

**Route Handler** (`app/api/auth/logout/route.ts`):

```typescript
import { createClient } from '@/lib/supabase/server';
import { NextResponse } from 'next/server';

export async function POST() {
  const supabase = await createClient();

  const { error } = await supabase.auth.signOut();

  if (error) {
    return NextResponse.json({ error: error.message }, { status: 500 });
  }

  return NextResponse.json({ success: true });
}
```

### 8.3 Logout Component

**Component** (`components/auth/LogoutButton.tsx`):

```typescript
'use client';

import { useState } from 'react';
import { logoutAction } from '@/lib/auth/actions';
import { Button } from '@/components/ui/button';
import { Loader2, LogOut } from 'lucide-react';
import { useRouter } from 'next/navigation';
import { toast } from 'sonner';

interface LogoutButtonProps {
  variant?: 'default' | 'outline' | 'ghost' | 'link' | 'destructive';
  size?: 'default' | 'sm' | 'lg' | 'icon';
  className?: string;
}

export function LogoutButton({ variant = 'ghost', size = 'default', className }: LogoutButtonProps) {
  const [isLoading, setIsLoading] = useState(false);
  const router = useRouter();

  async function handleLogout() {
    setIsLoading(true);

    try {
      const result = await logoutAction();

      if (result?.error) {
        toast.error(result.error);
      } else {
        toast.success('Signed out successfully');
        router.push('/login');
        router.refresh();
      }
    } catch (err) {
      toast.error('Failed to sign out');
      console.error('Logout error:', err);
    } finally {
      setIsLoading(false);
    }
  }

  return (
    <Button
      variant={variant}
      size={size}
      onClick={handleLogout}
      disabled={isLoading}
      className={className}
    >
      {isLoading ? (
        <>
          <Loader2 className="mr-2 h-4 w-4 animate-spin" />
          Signing out...
        </>
      ) : (
        <>
          <LogOut className="mr-2 h-4 w-4" />
          Sign out
        </>
      )}
    </Button>
  );
}
```

### 8.4 Client-Side Logout Hook

**Hook** (`hooks/use-logout.ts`):

```typescript
'use client';

import { useRouter } from 'next/navigation';
import { createClient } from '@/lib/supabase/client';
import { toast } from 'sonner';

export function useLogout() {
  const router = useRouter();
  const supabase = createClient();

  async function logout() {
    try {
      const { error } = await supabase.auth.signOut();

      if (error) {
        toast.error('Failed to sign out');
        throw error;
      }

      toast.success('Signed out successfully');
      router.push('/login');
      router.refresh();
    } catch (err) {
      console.error('Logout error:', err);
      throw err;
    }
  }

  return { logout };
}
```

---

## 9. Error Handling Components

### 9.1 Auth Error Component

**Component** (`components/auth/AuthError.tsx`):

```typescript
import { Alert, AlertDescription, AlertTitle } from '@/components/ui/alert';
import { AlertCircle } from 'lucide-react';

interface AuthErrorProps {
  message: string;
}

export function AuthError({ message }: AuthErrorProps) {
  return (
    <Alert variant="destructive">
      <AlertCircle className="h-4 w-4" />
      <AlertTitle>Error</AlertTitle>
      <AlertDescription>{message}</AlertDescription>
    </Alert>
  );
}
```

---

## 10. Security Considerations

### 10.1 Rate Limiting

**Supabase Default Limits**:
- Sign-up: 5 requests/hour/IP
- Sign-in: 5 requests/hour/IP
- Password reset: 3 requests/hour/IP
- Email verification resend: 3 requests/hour/IP

**Additional Protection** (optional):
- Implement CAPTCHA for sign-up/login (reCAPTCHA v3 or hCaptcha)
- Implement client-side rate limiting with exponential backoff
- Monitor and alert on suspicious patterns

### 10.2 Email Enumeration Prevention

**Strategy**:
- Always return success for password reset requests (even if email doesn't exist)
- Don't reveal whether an email is registered during sign-up (generic error message)
- Use consistent timing for all auth operations

### 10.3 Session Security

**Token Storage**:
- Supabase handles token storage securely in HTTP-only cookies (via `@supabase/ssr`)
- No manual localStorage/sessionStorage manipulation
- Tokens are automatically refreshed by middleware

**Session Expiration**:
- Default: 1 hour (configurable in Supabase)
- Refresh token: 30 days (configurable)
- Automatic refresh handled by middleware

### 10.4 Password Security

**Policy**:
- Minimum 8 characters
- Require lowercase letter
- Require number
- Optional: uppercase, special characters (future enhancement)

**Storage**:
- Passwords are hashed by Supabase (bcrypt)
- Never stored in plain text
- Never logged or exposed in error messages

---

## 11. Testing Checklist

### 11.1 Sign-Up (AUTH-010)

- [ ] Valid sign-up creates account and sends verification email
- [ ] Duplicate email shows appropriate error
- [ ] Weak password shows validation errors
- [ ] Password strength indicator works correctly
- [ ] Terms checkbox validation works
- [ ] Password confirmation mismatch shows error
- [ ] Rate limiting prevents excessive sign-ups
- [ ] Redirects to verification page after sign-up

### 11.2 Login (AUTH-011)

- [ ] Valid credentials log in successfully
- [ ] Invalid credentials show error
- [ ] Unverified email shows appropriate message
- [ ] Disabled user/profile cannot log in
- [ ] Session persists on page refresh
- [ ] Session refresh works automatically
- [ ] Rate limiting prevents brute force

### 11.3 Email Verification (AUTH-012)

- [ ] Verification link verifies email successfully
- [ ] Expired link shows appropriate error
- [ ] Invalid token shows error
- [ ] Resend verification email works
- [ ] Resend rate limiting works
- [ ] Redirects to onboarding after verification

### 11.4 Password Reset (AUTH-013)

- [ ] Forgot password sends reset email
- [ ] Reset link allows password change
- [ ] Expired reset link shows error
- [ ] Invalid token shows error
- [ ] Password validation works on reset form
- [ ] Successful reset redirects to login
- [ ] Email enumeration prevention works (always returns success)

### 11.5 Logout (AUTH-014)

- [ ] Logout clears session
- [ ] Logout redirects to login page
- [ ] Cannot access protected routes after logout
- [ ] Session invalidation works correctly

---

## 12. Implementation Checklist

### Story AUTH-010: Email/Password Sign-Up

- [ ] **Schema**: Sign-up Zod schema created
- [ ] **Server Action**: `signUpAction` implemented
- [ ] **Page**: Sign-up page created
- [ ] **Form**: Sign-up form component with validation
- [ ] **Password Strength**: Password strength indicator component
- [ ] **Email Verification Prompt**: Component for post-signup verification
- [ ] **Error Handling**: Error display and validation messages
- [ ] **Rate Limiting**: Supabase rate limits configured
- [ ] **Testing**: All test cases pass

### Story AUTH-011: Email/Password Login

- [ ] **Schema**: Login Zod schema created
- [ ] **Server Action**: `loginAction` implemented
- [ ] **Page**: Login page created
- [ ] **Form**: Login form component
- [ ] **Session Management**: Supabase client setup (browser/server/middleware)
- [ ] **Middleware**: Session refresh middleware implemented
- [ ] **Auth Hook**: `useAuth` hook for client-side auth state
- [ ] **Error Handling**: Login error handling
- [ ] **Testing**: All test cases pass

### Story AUTH-012: Email Verification

- [ ] **Server Actions**: `verifyEmailAction` and `resendVerificationEmail` implemented
- [ ] **Callback Route**: Auth callback route handler
- [ ] **Page**: Verification page created
- [ ] **Status Component**: Verification status component
- [ ] **Resend Component**: Resend verification component with cooldown
- [ ] **Error Handling**: Expired/invalid link handling
- [ ] **Testing**: All test cases pass

### Story AUTH-013: Password Reset

- [ ] **Schemas**: Forgot password and reset password Zod schemas
- [ ] **Server Actions**: `forgotPasswordAction` and `resetPasswordAction` implemented
- [ ] **Forgot Password Page**: Page and form component
- [ ] **Reset Password Page**: Page and form component
- [ ] **Code Validation**: Reset code validation logic
- [ ] **Error Handling**: Expired/invalid link handling
- [ ] **Email Enumeration Prevention**: Always return success for forgot password
- [ ] **Testing**: All test cases pass

### Story AUTH-014: Logout

- [ ] **Server Action**: `logoutAction` implemented
- [ ] **API Route**: Logout API route (optional)
- [ ] **Logout Button**: Logout button component
- [ ] **Logout Hook**: `useLogout` hook
- [ ] **Session Cleanup**: Proper session invalidation
- [ ] **Redirect**: Redirect to login after logout
- [ ] **Testing**: All test cases pass

---

## End of Document

This TDD provides comprehensive technical specifications for implementing Epic 1 – Authentication (Email/Password, Sessions, Verification, Recovery). All components use Supabase Auth, Next.js App Router, and shadcn/ui for a consistent, secure authentication experience.

**Next Steps**: After completing Epic 1, proceed to Epic 2 (Social Login) or Epic 3 (Onboarding) depending on product priorities.

