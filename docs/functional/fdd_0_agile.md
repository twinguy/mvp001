## Platform Foundation – Agile User Stories (`fdd_0_agile.md`)

This document decomposes **FDD 0** (`fdd_0.md`) into epics and user stories suitable for implementation on **Supabase (Auth + Postgres + RLS + Edge Functions)** with a **Next.js frontend hosted on Vercel**, aligned with `functional.md`, `tooling.md`, and `overview.md`.

**Primary goal**: deliver the complete Authentication + Account Management foundation required for **CRM (`fdd_1.md`)** to function (multi-tenancy `org_id`, `profiles` mapping, roles/permissions, secure access patterns), without implementing CRM features themselves.

Each story includes:
- **User story** (role / intent / value)
- **Acceptance Criteria**
- **Definition of Done (DoD)**

Story IDs are for reference (not prescriptive for tooling).

---

## Epic 0 – Platform Identity Model & Tenancy Primitives (Required by CRM)

Create the foundational identity/tenancy primitives required by all modules (especially CRM): `orgs`, `profiles`, role model, and clear conventions for deriving `org_id` and role from authenticated users.

### Story AUTH-000 – Define Tenancy Strategy and Account Model

**As a** platform architect  
**I want** a documented tenancy strategy and account model  
**So that** all modules (CRM first) can consistently scope data and permissions.

**Acceptance Criteria**
- [ ] A decision is documented for tenancy shape:
  - [ ] single Supabase project, multi-tenant by `org_id` (default), or
  - [ ] one Supabase project per tenant (still keep `org_id` for portability).
- [ ] The decision includes implications for:
  - [ ] RLS strategy (org-scoped policies)
  - [ ] onboarding (org creation)
  - [ ] support/admin access (break-glass approach)
- [ ] Naming conventions for tables/columns are documented (e.g., `org_id` everywhere, `user_id` references `auth.users.id`).

**Definition of Done**
- [ ] Tenancy decision is written in a short doc section in this file (or a referenced doc) and linked from `fdd_1.md`/`fdd_1_agile.md`.
- [ ] A reviewer confirms the decision aligns with `tooling.md` (Supabase + Vercel) and future modules.

---

### Story AUTH-001 – Define `orgs` (Accounts) Concept and Lifecycle

**As an** owner/operator  
**I want** an account (“org”) to represent my company  
**So that** my data is isolated and my team can collaborate.

**Acceptance Criteria**
- [ ] The concept of an organization/account is defined:
  - [ ] minimal fields required now (e.g., org name, owner user, created_at)
  - [ ] ability to add org-level metadata later (address, timezone, billing)
- [ ] The org lifecycle is documented:
  - [ ] org creation at onboarding
  - [ ] org updates (name/contact)
  - [ ] org deactivation (soft delete) policy
- [ ] The org model is explicitly described as a prerequisite for CRM’s `org_id` scoping.

**Definition of Done**
- [ ] Org lifecycle is documented with at least one happy-path flow and one deactivation scenario.
- [ ] The design is referenced by CRM Epic 1 story about tenancy (CRM-001).

---

### Story AUTH-002 – Define User Profile Mapping (`profiles`) and Role Model

**As a** platform integrator  
**I want** a `profiles` model mapping `auth.users` to orgs and roles  
**So that** RLS and app logic can reliably scope access by tenant and permissions.

**Acceptance Criteria**
- [ ] A `profiles` concept is documented with required fields:
  - [ ] `user_id` (auth user id)
  - [ ] `org_id`
  - [ ] `role` (see role set below)
  - [ ] `is_active` (or equivalent)
  - [ ] basic contact fields used by app UI (name, phone) (optional but recommended)
- [ ] Role set is documented and includes, at minimum:
  - [ ] `owner` (billing + org admin)
  - [ ] `admin`
  - [ ] `manager`
  - [ ] `csr` (customer service / sales)
  - [ ] `dispatcher` (future dispatch console)
  - [ ] `technician` (future mobile)
  - [ ] `viewer` (read-only)
- [ ] The model explicitly supports CRM needs:
  - [ ] CSR/admin can manage customers
  - [ ] technicians may have limited read access to customer data in later modules
- [ ] A policy exists for multi-org users (choose one for MVP):
  - [ ] not supported (single org per user) with a clear error/UX, or
  - [ ] supported (one user can belong to multiple orgs) with active-org selection.

**Definition of Done**
- [ ] The `profiles` + role model is documented with example rows.
- [ ] A note exists describing how CRM tables will reference `org_id` and how RLS will use `profiles`.

---

### Story AUTH-003 – Define Auth Context Conventions (How `org_id` and `role` Are Derived)

**As a** developer  
**I want** a standardized way to derive current `org_id` and role for the logged-in user  
**So that** all modules enforce access consistently.

**Acceptance Criteria**
- [ ] A documented convention exists for deriving:
  - [ ] current user id (`auth.uid()`)
  - [ ] current org id (from `profiles.org_id` or JWT claims)
  - [ ] current role (from `profiles.role` or JWT claims)
- [ ] The convention specifies usage patterns for:
  - [ ] database RLS policies (SQL-side)
  - [ ] Edge Functions (server-side)
  - [ ] Next.js UI (client-side)
- [ ] The convention clarifies what is trusted vs untrusted:
  - [ ] client-provided `org_id` is not trusted
  - [ ] server derives org/role from auth context

**Definition of Done**
- [ ] Conventions are written as a “single source of truth” section in this doc (or a referenced platform conventions doc).
- [ ] The conventions are referenced by CRM stories where access control is discussed.

---

## Epic 1 – Authentication (Email/Password, Sessions, Verification, Recovery)

Implement secure user registration/login with email verification and password recovery, aligned with Supabase Auth.

### Story AUTH-010 – Email/Password Sign-Up

**As a** new owner/operator  
**I want** to register with email/password  
**So that** I can access the platform securely.

**Acceptance Criteria**
- [ ] A sign-up UI flow exists with:
  - [ ] email input
  - [ ] password input with strength guidance (minimum policy documented)
  - [ ] confirm password
  - [ ] acceptance of terms checkbox (optional if terms exist; otherwise explicitly deferred)
- [ ] Sign-up enforces:
  - [ ] unique email
  - [ ] password policy (min length + basic complexity)
  - [ ] rate limiting / bot protection strategy is documented (e.g., Supabase rate limits + optional CAPTCHA)
- [ ] After sign-up:
  - [ ] user is prompted to verify email (if email verification enabled)
  - [ ] user is routed into onboarding (org creation) after verification (or immediately with limited access until verified; policy documented)

**Definition of Done**
- [ ] Sign-up flow is documented with screenshots/wire description of key states (success, errors).
- [ ] A test checklist exists (valid signup, duplicate email, weak password, unverified flow).

---

### Story AUTH-011 – Email/Password Login and Session Management

**As a** returning user  
**I want** to log in securely and remain signed in  
**So that** I can use the app without frequent re-authentication.

**Acceptance Criteria**
- [ ] Login supports email/password authentication.
- [ ] Session behavior is documented:
  - [ ] token storage strategy (Supabase default; no custom localStorage hacks)
  - [ ] session refresh behavior (silent refresh)
  - [ ] logout behavior (local + server)
- [ ] Login errors are handled:
  - [ ] invalid credentials
  - [ ] unverified email (if enforced)
  - [ ] disabled user/profile
- [ ] A “Remember me” behavior is documented (even if Supabase handles it implicitly).

**Definition of Done**
- [ ] Login flow is documented including error states and redirects.
- [ ] A test checklist exists (login success, invalid password, logout, session persists on refresh).

---

### Story AUTH-012 – Email Verification

**As a** platform owner  
**I want** email verification  
**So that** accounts cannot be created with invalid email addresses.

**Acceptance Criteria**
- [ ] Email verification policy is defined (required vs optional) and consistent across the app.
- [ ] Verification email template requirements are documented:
  - [ ] sender identity
  - [ ] link expiration expectations
  - [ ] redirect URL after verification (to onboarding or dashboard)
- [ ] The UI handles:
  - [ ] “Check your email” state after signup
  - [ ] re-send verification email with throttling rules documented
  - [ ] verified confirmation and next step

**Definition of Done**
- [ ] Verification behavior and redirects are documented.
- [ ] A test checklist exists (verify link, expired link, resend link).

---

### Story AUTH-013 – Password Reset / Recovery

**As a** user  
**I want** to reset my password via email  
**So that** I can regain access if I forget it.

**Acceptance Criteria**
- [ ] “Forgot password” flow exists with email input.
- [ ] Reset email template requirements are documented (sender, link expiry, redirect URL).
- [ ] Reset form enforces password policy and confirms password.
- [ ] The UI handles:
  - [ ] unknown email (does not leak account existence; behavior documented)
  - [ ] expired/invalid reset link
  - [ ] successful reset and re-login

**Definition of Done**
- [ ] Reset flow is documented with key screens/states.
- [ ] A test checklist exists (reset success, expired link, weak password, unknown email behavior).

---

### Story AUTH-014 – Logout and Session Revocation

**As a** security-conscious user  
**I want** to log out and revoke my session  
**So that** my account cannot be accessed on shared devices.

**Acceptance Criteria**
- [ ] Logout is available from the UI.
- [ ] Logout clears local session state and invalidates session as appropriate.
- [ ] A policy is documented for “log out of all devices” (include as MVP or explicitly defer with rationale).

**Definition of Done**
- [ ] Logout behavior is verified and documented.
- [ ] Test checklist covers single-device logout and (if supported) global logout.

---

## Epic 2 – Social Login (OAuth) for Streamlined Access

### Story AUTH-020 – Google OAuth Login

**As a** user  
**I want** to sign in with Google  
**So that** I can access the platform without managing another password.

**Acceptance Criteria**
- [ ] Google OAuth is supported (Supabase Auth provider).
- [ ] The UI supports:
  - [ ] “Continue with Google”
  - [ ] account linking behavior (if email already exists; policy documented)
- [ ] Redirects after OAuth are defined:
  - [ ] new user → onboarding
  - [ ] existing user → app home
- [ ] Error handling covers:
  - [ ] user cancels OAuth
  - [ ] provider misconfiguration

**Definition of Done**
- [ ] OAuth login behavior is documented with example flows.
- [ ] Test checklist exists (new OAuth user, existing OAuth user, cancellation, misconfig).

---

### Story AUTH-021 – Social Login Extensibility

**As a** product owner  
**I want** a consistent pattern for adding additional social providers later  
**So that** we can expand login options without redesigning auth.

**Acceptance Criteria**
- [ ] A documented pattern exists for adding providers (e.g., Microsoft, Apple) with:
  - [ ] required provider configuration fields
  - [ ] redirect URL conventions
  - [ ] account linking rules
- [ ] UI is designed so new providers can be added without major layout changes.

**Definition of Done**
- [ ] A short “provider checklist” is included in this doc.
- [ ] Deferred providers are listed explicitly (if any).

---

## Epic 3 – Onboarding: Org Creation and Owner Profile Setup

### Story AUTH-030 – Owner Onboarding: Create Organization

**As a** new owner/operator  
**I want** to create my company account during onboarding  
**So that** my data is scoped to my organization and I can invite my team.

**Acceptance Criteria**
- [ ] Onboarding flow collects:
  - [ ] business name
  - [ ] owner name
  - [ ] owner contact (phone optional)
  - [ ] timezone (recommended for scheduling/reminders; if deferred, documented)
- [ ] On submit:
  - [ ] an org is created
  - [ ] the current user gets a `profiles` entry in that org with role `owner`
- [ ] The flow is idempotent (refresh/retry does not create duplicate orgs; behavior documented).
- [ ] Unverified email policy is enforced consistently (e.g., cannot create org until verified).

**Definition of Done**
- [ ] Onboarding flow is documented (happy path + failure cases).
- [ ] Test checklist exists (new org creation, duplicate submit, verified/unverified path).

---

### Story AUTH-031 – User Profile Management (Basic Info)

**As a** user  
**I want** to manage my profile information  
**So that** my team sees correct name and contact details.

**Acceptance Criteria**
- [ ] Users can view and update:
  - [ ] display name
  - [ ] phone (optional)
  - [ ] avatar (optional; if included, define storage approach; else explicitly deferred)
- [ ] Updates are permission-checked:
  - [ ] user can update their own profile
  - [ ] admins/owners can update team member profile fields (policy documented)

**Definition of Done**
- [ ] Profile fields and permissions are documented.
- [ ] Test checklist exists (self update, admin update, forbidden edits).

---

## Epic 4 – Team Management: Invites, Membership, Roles

### Story AUTH-040 – Invite Team Member by Email

**As an** owner/operator  
**I want** to invite team members by email  
**So that** my staff can access the platform under my org.

**Acceptance Criteria**
- [ ] An invite flow exists that:
  - [ ] collects invitee email
  - [ ] selects role (at least: admin, manager, csr, viewer; technician/dispatcher may be hidden until needed)
  - [ ] sends an invite email with an acceptance link
- [ ] Invite acceptance behavior is documented for:
  - [ ] invitee is a new Supabase user (creates auth user)
  - [ ] invitee already has an auth user with same email (links membership)
- [ ] Invites have a defined expiration window (e.g., 7 days) and resend policy.
- [ ] Invites are revocable by owner/admin.

**Definition of Done**
- [ ] Invite lifecycle is documented (create, resend, revoke, expire, accept).
- [ ] Test checklist exists (new user invite, existing user invite, revoked invite, expired invite).

---

### Story AUTH-041 – Manage Team Members (List, Activate/Deactivate)

**As an** owner/operator  
**I want** to view and manage my team members  
**So that** I can control access as staff changes.

**Acceptance Criteria**
- [ ] Team members list shows:
  - [ ] name/email
  - [ ] role
  - [ ] status (active/inactive)
  - [ ] last login (optional; if deferred, documented)
- [ ] Owner/admin can:
  - [ ] deactivate a member (prevents access)
  - [ ] reactivate a member
  - [ ] remove a member (policy documented: soft delete vs hard remove)
- [ ] Owner account protection is defined:
  - [ ] cannot deactivate the last remaining owner (or a safe transfer process exists)

**Definition of Done**
- [ ] Team management behaviors are documented.
- [ ] Test checklist exists (deactivate user blocks access, owner protection, reactivation).

---

### Story AUTH-042 – Change Member Role (RBAC)

**As an** owner/admin  
**I want** to change a user’s role  
**So that** I can grant or restrict permissions over time.

**Acceptance Criteria**
- [ ] Role change UI and behavior is defined.
- [ ] Allowed role transitions are documented (e.g., only owner can assign owner role; owner transfer process).
- [ ] Role changes take effect immediately for authorization decisions.

**Definition of Done**
- [ ] Role change scenarios are documented (owner transfer, downgrade, upgrade).
- [ ] Test checklist exists (role changes reflect in permissions, forbidden transitions blocked).

---

## Epic 5 – Authorization & Least-Privilege Access (RLS-Ready)

Define how permissions work across the platform and ensure the model supports CRM’s RLS requirements.

### Story AUTH-050 – Define Permission Matrix for Roles (MVP)

**As a** platform architect  
**I want** a permission matrix for roles  
**So that** product, engineering, and QA agree on who can do what (starting with CRM needs).

**Acceptance Criteria**
- [ ] A permission matrix is documented covering, at minimum:
  - [ ] CRM: customers/contacts/locations/interactions/followups/segments/automations
  - [ ] platform: team management, billing, security settings
- [ ] At least these decisions are explicit:
  - [ ] viewer is read-only across CRM
  - [ ] csr can CRUD customer data, interactions, followups
  - [ ] manager can manage automations/segments (or admin-only; decision documented)
  - [ ] technician has limited customer read access (future use; at least documented)
- [ ] “Least privilege” is applied: default roles do not get unnecessary write permissions.

**Definition of Done**
- [ ] Permission matrix is included in this doc (or linked) and referenced by `fdd_1_agile.md` where relevant.
- [ ] Stakeholders confirm the matrix supports CRM flows.

---

### Story AUTH-051 – Define RLS Policy Strategy for Org-Scoped Tables

**As a** developer  
**I want** a standard RLS strategy for org-scoped data  
**So that** all modules enforce tenant isolation consistently.

**Acceptance Criteria**
- [ ] An RLS strategy is documented including:
  - [ ] base predicate for org-scoped tables (rows must match user’s org)
  - [ ] role-based predicates for write operations
  - [ ] service-role usage rules (when and why it bypasses RLS)
- [ ] The strategy explicitly supports CRM’s tables and access patterns.

**Definition of Done**
- [ ] RLS strategy is documented with example pseudo-policies (no SQL required in this doc).
- [ ] A checklist exists that CRM epic implementation can follow table-by-table.

---

### Story AUTH-052 – Break-Glass / Support Access Policy (Non-Default)

**As a** platform operator  
**I want** a controlled “support access” policy  
**So that** issues can be debugged without weakening tenant isolation.

**Acceptance Criteria**
- [ ] A policy is documented for support access that includes:
  - [ ] when it can be used
  - [ ] who can use it
  - [ ] how it is audited
- [ ] The default runtime does not grant cross-org access to normal roles.

**Definition of Done**
- [ ] Support access policy is written and reviewed.
- [ ] Any needed future work is captured as backlog items (e.g., dedicated admin UI).

---

## Epic 6 – Subscription Management, Free Trial, Billing (Stripe)

Implement basic subscription lifecycle and usage-limit enforcement required by product packaging. This epic must not block CRM core usage for MVP unless explicitly required.

### Story AUTH-060 – Define Plans, Trial Rules, and Usage Limits (MVP)

**As a** product owner  
**I want** plan definitions and usage limits documented  
**So that** billing and enforcement can be implemented consistently.

**Acceptance Criteria**
- [ ] Plans are defined (at least: Free Trial, Paid Basic; optionally Pro).
- [ ] Trial rules are defined:
  - [ ] trial duration
  - [ ] conversion rules at trial end (grace period vs lockout)
  - [ ] what features are available during trial
- [ ] Usage limits are defined (per `fdd_0.md` examples):
  - [ ] number of facilities or units (even if those modules come later, the limit model is defined now)
  - [ ] user seats (optional; document if not enforced)
- [ ] A strategy is documented for enforcement points (UI warnings vs hard blocks).

**Definition of Done**
- [ ] Plan and limit definitions are documented in a stable format (table/list).
- [ ] Any deferred enforcement is explicitly stated with follow-up stories/backlog.

---

### Story AUTH-061 – Stripe Customer + Subscription Lifecycle (High-Level)

**As an** owner/operator  
**I want** to start a subscription and manage billing  
**So that** I can keep the account active after trial.

**Acceptance Criteria**
- [ ] Stripe integration responsibilities are documented:
  - [ ] customer creation
  - [ ] checkout session
  - [ ] webhook handling (subscription created/updated/canceled, payment failed)
- [ ] Billing states are defined:
  - [ ] trialing
  - [ ] active
  - [ ] past_due
  - [ ] canceled
- [ ] A policy exists for what happens when billing is past due (grace/lockout).

**Definition of Done**
- [ ] Billing lifecycle is documented with a state diagram or bullet state machine.
- [ ] A test checklist exists (trial → active, payment failure, cancel).

---

### Story AUTH-062 – Billing Portal / Subscription Management UI

**As an** owner/operator  
**I want** to view my subscription status and update payment method  
**So that** I can manage billing without support.

**Acceptance Criteria**
- [ ] A UI page exists to show:
  - [ ] current plan
  - [ ] trial end date or renewal date
  - [ ] billing status (active/past due/canceled)
- [ ] A documented approach exists to manage billing:
  - [ ] link to Stripe customer portal (preferred), or
  - [ ] in-app payment method update (defer if not MVP)

**Definition of Done**
- [ ] UI requirements and navigation are documented.
- [ ] Test checklist exists (view status, open portal, reflect updated status after webhook).

---

### Story AUTH-063 – Usage Limit Enforcement Hooks (Framework)

**As a** platform owner  
**I want** the system to enforce usage limits  
**So that** accounts respect plan constraints.

**Acceptance Criteria**
- [ ] A limit-enforcement framework is documented that describes:
  - [ ] where checks happen (server-side Edge Function / DB constraints / both)
  - [ ] how limits are calculated (counts by org)
  - [ ] how violations are reported (error codes for UI)
- [ ] At least one limit is implemented conceptually in the design:
  - [ ] e.g., “max users in org” or “max facilities” (even if facility module not built yet)

**Definition of Done**
- [ ] Enforcement rules and error contract are documented.
- [ ] Backlog item list exists for per-module limits (Facilities, Units, etc.).

---

## Epic 7 – MFA and Security Hardening

### Story AUTH-070 – MFA Enable/Disable (TOTP)

**As a** user  
**I want** to enable multi-factor authentication (MFA)  
**So that** my account is protected even if my password is compromised.

**Acceptance Criteria**
- [ ] MFA method is defined (TOTP via authenticator app for MVP).
- [ ] User can:
  - [ ] enroll MFA (show QR, verify code)
  - [ ] disable MFA (requires password re-check or MFA challenge; policy documented)
- [ ] Login flow supports MFA challenge when enabled.
- [ ] Recovery strategy is defined:
  - [ ] recovery codes supported, or
  - [ ] support-assisted recovery policy documented (break-glass with audit).

**Definition of Done**
- [ ] MFA UX flows are documented (enroll, challenge, disable, recover).
- [ ] Test checklist exists (enable MFA, login with MFA, disable MFA, lost device scenario).

---

### Story AUTH-071 – Security Events and Audit Logging (Auth & Account)

**As a** compliance-conscious operator  
**I want** security/audit events recorded  
**So that** we can investigate access issues and security incidents.

**Acceptance Criteria**
- [ ] A list of audited events is defined (at minimum):
  - [ ] sign-up
  - [ ] login success/failure (rate-limited; avoid storing secrets)
  - [ ] password reset requested/completed
  - [ ] MFA enabled/disabled
  - [ ] team member invited/accepted/role changed/deactivated
  - [ ] billing status changes
- [ ] PII handling is documented for audit logs (what is stored, what is not).
- [ ] Retention policy for audit logs is documented (e.g., 90 days+).

**Definition of Done**
- [ ] Audit event list and retention/PII policy are documented.
- [ ] Backlog items exist for a future “audit log viewer” UI (if not MVP).

---

### Story AUTH-072 – Rate Limiting and Abuse Prevention Policy

**As a** platform operator  
**I want** protection against brute force and abuse  
**So that** authentication endpoints and invite flows are resilient.

**Acceptance Criteria**
- [ ] Rate limiting strategy is documented for:
  - [ ] login attempts
  - [ ] password reset requests
  - [ ] invite sends/resends
  - [ ] MFA challenge attempts
- [ ] Bot mitigation policy is documented (CAPTCHA optional; explicitly state if deferred).

**Definition of Done**
- [ ] Rate limiting and abuse-prevention policy is documented and reviewed.
- [ ] Test checklist exists for basic abuse scenarios (excess resets, repeated login failures).

---

## Epic 8 – UX: Auth Pages, App Shell Guarding, and Settings

### Story AUTH-080 – Auth UI Pages and Routing (Next.js)

**As a** user  
**I want** clear authentication pages and redirects  
**So that** I can sign up, log in, and recover access smoothly.

**Acceptance Criteria**
- [ ] Pages/routes are defined for:
  - [ ] sign up
  - [ ] login
  - [ ] verify email
  - [ ] forgot password
  - [ ] reset password
  - [ ] MFA challenge (if applicable)
- [ ] Redirect rules are documented:
  - [ ] unauthenticated → login
  - [ ] authenticated but no org/profile → onboarding
  - [ ] authenticated with org/profile → app home
- [ ] Error UX is defined for common auth failures.

**Definition of Done**
- [ ] A simple route map and redirect table is included in this doc.
- [ ] Test checklist exists for navigation/redirect scenarios.

---

### Story AUTH-081 – Settings: Account, Team, Billing, Security

**As an** owner/operator  
**I want** a settings area for account/team/billing/security  
**So that** I can administer the platform without support.

**Acceptance Criteria**
- [ ] Settings sections are defined:
  - [ ] My Profile
  - [ ] Organization
  - [ ] Team Members
  - [ ] Billing
  - [ ] Security (MFA)
- [ ] Access rules are documented:
  - [ ] only owners/admins can see team and billing
  - [ ] all users can see my profile and MFA

**Definition of Done**
- [ ] Settings IA (information architecture) is documented with nav items and access rules.
- [ ] Test checklist exists (role-based visibility, forbidden direct navigation blocked).

---

## Epic 9 – Integration Readiness for CRM (Explicit Dependencies)

This epic ensures that the Auth foundation is explicitly “ready” for CRM stories and that assumptions are documented.

### Story AUTH-090 – CRM Readiness Checklist and Contracts

**As a** CRM implementer  
**I want** a readiness checklist for auth/tenancy primitives  
**So that** CRM epics can proceed without re-deciding identity and access patterns.

**Acceptance Criteria**
- [ ] A checklist exists covering:
  - [ ] org model is defined
  - [ ] profiles model is defined
  - [ ] role set and permission matrix exist
  - [ ] org_id derivation conventions exist
  - [ ] invite/team management flows exist
  - [ ] policy for technicians’ limited customer access is documented (even if not implemented yet)
- [ ] CRM stories that depend on these primitives are referenced (e.g., CRM-001 and CRM RLS stories).

**Definition of Done**
- [ ] Checklist is complete and referenced in `fdd_1_agile.md` (as a dependency note).
- [ ] Any gaps are captured as explicit backlog items with story IDs in this doc.

---

## Appendix A – Suggested Role Permission Summary (MVP)

This is a summary table for quick alignment; detailed permissions should be elaborated during technical design.

- **owner**: full access incl. billing, team, security settings
- **admin**: full app access except billing ownership transfer (policy-dependent)
- **manager**: CRM management + reporting; limited platform admin
- **csr**: CRM customers/interactions/followups; cannot manage billing/team
- **dispatcher**: (future) dispatch console access; limited CRM read
- **technician**: (future) limited customer read + assigned work visibility
- **viewer**: read-only access to CRM and other modules as permitted


