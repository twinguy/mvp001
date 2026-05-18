# Servexa Functional Design v2 — Common Business Services Layer for Small & Medium Businesses

## Executive Summary

Servexa was originally scoped (see `functional.md`) as a vertical, AI-enabled field service platform for HVAC, Plumbing, and Electrical contractors. This v2 functional design **repositions Servexa as a horizontal "Common Business Services Layer" (CBSL)** purpose-built for the operational backbone shared by **most small and medium-sized business entities (SMBs)** — including but no longer limited to field-service trades.

The platform retains its multi-tenant, Supabase + Next.js foundation and its existing modules (CRM, Scheduling, Work Orders, Quoting/Invoicing, Inventory, Customer Portal, Communication, Insights, IoT). It generalizes them and adds first-class support for:

- **Appointment Scheduling** (online, on-site, virtual, multi-resource).
- **Digital Payments** (card, ACH, wallets, terminal, links, recurring).
- **Follow-Up Automation & No-Show Management** (cadence engine + risk scoring).
- **Staff & Shift Scheduling** (rosters, time clocks, availability, swaps).
- **Maintenance / Service Contracts** (recurring agreements, membership plans, SLAs).
- **Products & Services Catalog** (sellable goods, services, bundles, subscriptions, e-commerce checkout).

Servexa v2 is also **instrumented as a metered platform** so the same product can be monetized via a **consumption-based pricing model** layered on top of (or instead of) seat-based subscriptions. This document also specifies the metering, reporting, and pricing methodology.

---

## 1. Target Market & Vertical Coverage

### 1.1 Primary Target Segments

The CBSL is designed for SMBs (typically 1–250 employees) that share the same core operational primitives: a customer, an appointment/visit, a deliverable (service or product), an invoice, a payment, and ongoing engagement.

Representative verticals (non-exhaustive):

| Category | Examples |
|---|---|
| **Field Trades & Home Services** (legacy v1 focus) | HVAC, Plumbing, Electrical, Pest Control, Landscaping, Cleaning, Pool Service, Locksmith, Garage Door, Appliance Repair, Handyman, Solar Install/Service |
| **Health, Wellness & Personal Services** | Dental, Chiropractic, Physical Therapy, Optometry, Mental Health/Counseling, Med Spa, Salon, Barbershop, Massage, Pet Grooming/Boarding, Veterinary |
| **Professional & Consulting Services** | Legal practice, Accounting, Tax prep, Real Estate brokerages, Insurance agencies, Financial advisors, Marketing/Creative agencies |
| **Education & Coaching** | Tutoring, Music schools, Driving schools, Yoga/Pilates studios, Personal trainers, Test prep |
| **Auto & Equipment Services** | Auto repair, Detailing, Mobile mechanics, Tire shops, Equipment rental, Boat/Marine service |
| **Hospitality-Adjacent / Bookable Services** | Photographers, Event rentals, DJ/Entertainment, Catering, Wedding planners |
| **Light Retail with Service** | Bike shops, Music instrument stores, Computer repair, Phone repair, Specialty boutiques offering fittings/consults |

### 1.2 Out of Scope for v2

- Heavy industrial ERP (multi-plant manufacturing, advanced MRP).
- Full healthcare EHR / HIPAA-only specialty workflows (Servexa supports HIPAA-eligible deployments but is not an EHR).
- Restaurant POS (table management, KDS) — Servexa supports retail checkout, not table-service hospitality.
- Multi-entity GL / advanced accounting (continues to integrate with QuickBooks, Xero, NetSuite).

---

## 2. Product Branding & Component Map

**Product name**: Servexa — *The Common Business Services Layer*

### 2.1 Component Rename & Generalization

| v1 Component | v2 Component | Notes |
|---|---|---|
| Servexa Foreman | **Servexa Console** | Back-office workspace (formerly "Foreman" implied a job-site supervisor). |
| Servexa Auth | **Servexa Identity** | Unchanged in spirit; broader IdP/SSO scope. |
| Servexa CRM | **Servexa CRM** | Generalized; supports patients, clients, students, members, accounts. |
| Servexa Dispatch | **Servexa Schedule** | Unified appointment + dispatch + staff scheduling. |
| Servexa WorkOrders | **Servexa Jobs** | Generalized "job/case/engagement/visit" record. |
| Servexa Portal | **Servexa Portal** | Customer/patient/client self-service. |
| Servexa Insights | **Servexa Insights** | Analytics + AI. |
| *(new)* | **Servexa Commerce** | Catalog, cart, checkout, products, services, subscriptions. |
| *(new)* | **Servexa Pay** | Payments, payouts, refunds, recurring billing, terminals. |
| *(new)* | **Servexa Engage** | Follow-ups, reminders, cadences, no-show recovery, reviews. |
| *(new)* | **Servexa Workforce** | Staff profiles, shifts, rosters, time clock, availability, payroll exports. |
| *(new)* | **Servexa Plans** | Maintenance/service contracts, memberships, recurring deliverables, SLAs. |
| *(new)* | **Servexa Meter** | Usage metering, entitlements, rating, billing of the platform itself. |

### 2.2 High-Level Architecture (unchanged stack)

- **Backend**: Supabase (PostgreSQL, Auth, Storage, Realtime, Edge Functions).
- **Frontend**: Next.js (App Router) on Vercel, shadcn/ui.
- **Mobile**: Responsive web + future React Native apps sharing Supabase SDK.
- **Multi-tenancy**: Single Supabase project; every business-data row carries `org_id`; RLS enforces tenant isolation.
- **Industry packs**: A new layer of **vertical configuration bundles** (terminology, default catalog items, default checklists, default automations, default report packs) that tailor the generic platform to a vertical (e.g., "Dental Pack", "Pest Control Pack") without forking code.

---

## 3. Core Capability Catalog (Generalized)

The capabilities below supersede `functional.md` §1–§13. Items inherited from v1 are kept; new items are marked **(NEW)**. The capability list is intentionally organized so each line item is a candidate for being a **metered consumption dimension** (see §10).

### 3.1 Identity, Org & Access (Servexa Identity)

- Org provisioning, multi-org user membership, invite flows.
- Email/password, magic link, OAuth (Google, Microsoft, Apple), SSO/SAML (mid-market tier).
- MFA (TOTP, WebAuthn, SMS fallback).
- Role-based access control with a **generic role library**: `owner`, `admin`, `manager`, `front_desk`, `provider`, `dispatcher`, `field_staff`, `accountant`, `marketer`, `viewer`. Industry packs can rename these (e.g., `provider` → `dentist`, `field_staff` → `technician`).
- Audit log of identity & permission changes.

### 3.2 CRM — Contacts & Accounts (Servexa CRM)

Generalized from v1 §1.

- Unified **Contact** entity that can be a *Customer, Patient, Client, Student, Member, Lead, Prospect*. Terminology configurable per industry pack.
- **Account** entity for B2B contexts (companies, properties, households).
- Multiple addresses, contact methods, preferences, consent flags (GDPR/CCPA/TCPA).
- Tags, segments (rule-based, static, AI-generated).
- Activity timeline (calls, emails, SMS, notes, visits, payments).
- AI: segmentation, churn risk, lifetime value scoring, next-best-action.
- Custom fields per industry pack.
- Bulk import/export.

### 3.3 Appointment Scheduling **(NEW — first-class)** (Servexa Schedule)

A unified scheduling layer that subsumes v1 Dispatch.

- **Online booking** (customer-facing widget + portal): real-time slot availability, deposit/prepayment at booking.
- **Resource-aware scheduling**: book against any combination of *person* (provider, technician, instructor), *room/chair/bay*, *equipment*, *vehicle*, *service zone/territory*.
- **Multi-resource appointments** (e.g., a service that needs a dentist + hygienist + room).
- **Appointment types** with duration, buffer time, prep/cleanup time, required resources, eligible staff, default services.
- **Group appointments / classes** (yoga class, training session, group tour).
- **Virtual appointments** (telehealth/consult): auto-generates a video room link (integration with Zoom/Daily/Twilio Video).
- **On-site visits**: routing, ETAs, drive-time aware scheduling (inherits v1 dispatch routing).
- **Waitlist** with auto-promotion when slots open.
- **Recurring appointments** (e.g., weekly cleanings, monthly checkups, quarterly maintenance) tied to Plans (§3.7).
- **Conflict/double-book detection**, override with permission.
- **Hold/lock slots** during checkout to prevent race-condition double-bookings.
- **Calendar sync** (Google, Microsoft 365, Apple) — two-way.
- **Self-reschedule / self-cancel** via portal with policy controls (e.g., no cancel within 24h).

### 3.4 Jobs / Cases / Visits (Servexa Jobs)

Generalization of v1 Work Order Management.

- Generic "Job" record (`engagement`, `case`, `visit`, `appointment_outcome` are aliases per industry pack).
- Lifecycle state machine (configurable per industry).
- Tasks / checklists / forms (custom per job type).
- Attachments (photos, videos, documents, signed forms).
- Time tracking, materials/inventory consumption.
- Outcome capture, internal notes, customer-visible summary.
- Job ↔ Appointment ↔ Invoice linkage.
- AI categorization and prioritization.

### 3.5 Quoting, Invoicing & Estimates (Servexa Commerce + Servexa Pay)

Inherited from v1 §4 and generalized.

- Estimates / quotes / treatment plans (renamed per industry pack).
- Multi-tax, multi-currency, discounts, gratuities (NEW), service fees, surcharges.
- Deposits, partial / progress invoices, retainers (NEW for legal/professional).
- Statements, AR aging, dunning.
- Recurring invoices (subscription / membership billing — see §3.7).

### 3.6 Digital Payments **(NEW — first-class)** (Servexa Pay)

Promoted to a top-level capability rather than a sub-feature of invoicing.

- **Payment processors**: Stripe (primary), Adyen, Square, Authorize.net, ACH/Plaid; tenant chooses provider.
- **Payment methods**: card (CIT + MIT), ACH/bank transfer, digital wallets (Apple Pay, Google Pay), Buy-Now-Pay-Later (Affirm/Klarna), gift cards, account credits, cash/check tracking.
- **In-person**: card-present via terminal hardware (Stripe Terminal, Square Reader), QR-to-pay.
- **Online**: hosted checkout pages, embedded payment elements, pay-by-link via SMS/email, portal pay.
- **Recurring billing**: subscriptions, memberships, payment plans (split a balance over N installments).
- **Stored payment methods (tokens)** with consent tracking and last-used auditing.
- **Refunds, partial refunds, chargebacks/disputes** workflow.
- **Tipping / gratuities** at the point of payment.
- **Surcharging / convenience fees** where legally allowed; cash-discount programs.
- **Reconciliation**: payouts to bank, fee tracking, export to accounting.
- **Compliance**: PCI SAQ-A scope (tokenization only), 3DS/SCA, encryption.
- **AI**: fraud risk scoring on first-time card-not-present transactions; intelligent retry timing for failed recurring charges.

### 3.7 Maintenance / Service Contracts **(NEW)** (Servexa Plans)

A unified "Plan" entity covering: maintenance agreements (HVAC), memberships (gym, salon), retainers (legal/accounting), packages (10-class pass), warranties, subscription products.

- **Plan templates**: name, term length, renewal policy (auto/manual), billing cadence (one-time, monthly, quarterly, annual), price, included entitlements.
- **Entitlements**: a structured list of what the plan grants — e.g., *N visits per year*, *priority scheduling*, *discount % on catalog*, *free diagnostic*, *covered parts*, *response SLA hours*, *included class credits*.
- **Customer enrollment**: contracts (e-signed), start/end dates, renewal date, status (active, paused, expired, canceled, churned).
- **Automatic appointment generation**: scheduled visits auto-created on cadence (e.g., spring/fall HVAC tune-up); placed on dispatcher's review queue or auto-confirmed.
- **Entitlement consumption tracking**: each appointment/job/product can debit entitlements; remaining balance visible to staff and customer.
- **SLA enforcement**: priority-routing flags and response-time monitoring for contracted customers.
- **Renewal automation**: reminder cadence, auto-charge on renewal, upgrade/downgrade flows.
- **Revenue recognition helpers**: deferred revenue schedule export.
- **Reporting**: MRR/ARR, churn, retention, plan profitability.

### 3.8 Products & Services Catalog + E-Commerce **(NEW / Expanded)** (Servexa Commerce)

Expansion of v1 §4 service catalog to enable **direct sales** of products and services to customers.

- **Catalog**: items typed as `service`, `physical_product`, `digital_product`, `bundle`, `subscription_plan`, `gift_card`, `fee`.
- **Variants** (size, color, duration), SKUs, barcodes.
- **Pricing**: list price, cost, margin, price books (per customer tier or plan), bulk pricing, scheduled price changes.
- **Inventory link** (for physical goods) — pulls from v1 Inventory module; supports made-to-order and back-order.
- **Taxonomy**: categories, attributes, search, faceted filtering.
- **Storefront**:
  - **Hosted online store** (Next.js storefront generated per tenant, custom domain, SEO).
  - **Embeddable widgets** (book/buy/pay buttons for tenant's own website).
  - **In-portal store** (logged-in customer purchases).
  - **In-person POS** mode (staff sells at counter; pairs with Pay terminals).
- **Cart, promo codes, bundles, upsell/cross-sell rules**.
- **Order management**: order → fulfillment (ship/pickup/service-rendered) → invoice → payment.
- **Shipping**: rate shopping (USPS, UPS, FedEx via Shippo/EasyPost), labels, tracking.
- **Returns/RMAs** with policy windows.
- **Reviews & ratings** on products and services (feed into Engage's reputation tools).
- **AI**: product recommendations, dynamic pricing suggestions, abandoned-cart recovery.

### 3.9 Follow-Ups, Reminders & No-Show Management **(NEW — first-class)** (Servexa Engage)

Generalizes v1 §1 (CRM follow-ups) and §2 (reminders) into a unified engagement engine.

- **Cadences**: ordered, multi-step automation sequences (e.g., *Day -7 email, Day -2 SMS, Day -1 push, Day 0 morning SMS*).
- **Triggers**: appointment booked / confirmed / completed / canceled / no-show; invoice sent / overdue; plan expiring; no activity in N days; review request after service.
- **Channels**: SMS, email, push, in-app, voice (TTS or call campaign), postal mail (via Lob/Letterstream integrations).
- **Smart timing**: send-time optimization based on past engagement (AI).
- **Template library** with merge fields, attachments, AI-assisted writing.
- **Quiet hours, time-zone safety, opt-out enforcement** (TCPA, CAN-SPAM, CASL).
- **Two-way SMS** with reply handling (confirm, reschedule, opt-out keywords).
- **No-show & late-cancel handling**:
  - **Risk scoring**: AI predicts no-show probability per appointment using contact history, lead time, day-of-week, weather, prior behavior.
  - **Mitigation playbooks** triggered by risk threshold: extra reminder, required deposit/card-on-file, deposit forfeiture rule.
  - **Automated no-show fee charge** (configurable, with customer-visible policy disclosure).
  - **Recovery campaigns**: post-no-show cadence to rebook.
  - **Waitlist auto-promote** when a no-show or late cancel opens a slot.
- **Review & reputation requests**: post-service ask, route 4–5★ to Google/Yelp/Facebook, route 1–3★ to internal recovery.
- **Birthday/anniversary/win-back campaigns**.
- **A/B testing on subject lines, send times, and offers**.

### 3.10 Staff & Shift Scheduling **(NEW)** (Servexa Workforce)

A workforce-management layer adjacent to (but distinct from) appointment scheduling.

- **Staff profiles**: roles, skills/certifications, licenses (with expiry tracking), wage/cost rate, billable rate, color, photo, default location.
- **Availability templates**: weekly recurring schedules per staff member.
- **Shift scheduling**:
  - Drag-and-drop weekly roster.
  - Open shifts pool, shift bidding, shift swap requests with approval.
  - Multi-location staff (one person scheduled across sites/branches).
- **Time-off requests** (PTO, sick, unpaid), approval workflow, accrual tracking.
- **Time clock**:
  - Web clock-in, mobile geo-fenced clock-in, kiosk mode, QR code.
  - Break tracking, meal-break compliance per state/jurisdiction.
  - Auto clock-out / missed punch alerts.
- **Labor cost forecasting** vs. expected appointment revenue.
- **Compliance**: overtime alerts, minor-labor rules, predictive scheduling (Fair Workweek) where applicable.
- **Payroll exports**: ADP, Gusto, Paychex, Rippling, QuickBooks Payroll.
- **Coverage analytics**: capacity utilization, no-coverage warnings, overstaffing alerts.
- **Schedule publishing & notifications**: staff get push/SMS when schedule is posted or changed.

### 3.11 Customer Portal (Servexa Portal)

Inherited from v1 §7; extended with the new capabilities.

- Self-booking (§3.3), self-checkout (§3.8), self-pay (§3.6).
- Membership/plan management (view entitlements, renew, upgrade, cancel) (§3.7).
- Stored payment methods, autopay toggles.
- Order history, service history, document vault.
- Messaging with staff.
- Self-service AI assistant (chatbot grounded in tenant's FAQ and knowledge base).

### 3.12 Communication & Collaboration (Servexa Console + Engage)

Inherited from v1 §8.

- Unified inbox for SMS, email, voice (transcribed voicemail), portal messages, web chat.
- Internal team chat, per-job channels.
- AI: summarization, suggested replies, sentiment routing, escalation.

### 3.13 Insights, Reporting & AI (Servexa Insights)

- Pre-built dashboards per industry pack (e.g., dental: production per chair-hour; HVAC: revenue per route-hour).
- Custom report builder (no-code), scheduled email/Slack delivery.
- KPIs: bookings, fill rate, show rate, average ticket, NPS, CSAT, churn, MRR.
- AI: forecasting, anomaly detection, "ask your data" natural-language queries.
- Cohort analysis, customer LTV, marketing attribution.

### 3.14 Marketing Automation

- Campaigns: email, SMS, postcards, retargeting audiences.
- AI lead scoring, nurture journeys.
- Landing-page builder for promotions.
- Referral programs with tracking and rewards.
- Reputation management (§3.9 hooks).

### 3.15 Inventory & Asset Management (Servexa Console)

Inherited from v1 §5; remains optional per vertical (Health/Beauty may toggle off).

### 3.16 Integrations & Extensibility

- Open REST + Webhooks API for every entity.
- Native integrations: QuickBooks, Xero, Stripe, Twilio, SendGrid, Mailgun, Google/Microsoft Calendar, Zoom, Mapbox, Plaid, Shippo, Lob, Slack, HubSpot, Mailchimp.
- Marketplace/app directory model so vertical-specific tools (DrChrono, OpenDental, Mindbody migration, Jobber importer, etc.) can be installed per tenant.
- Zapier / Make connector.
- IoT layer (v1 §iot) remains for verticals needing it.

### 3.17 Security, Privacy & Compliance

- Role-based access, MFA, SSO/SAML, SCIM provisioning (mid-market).
- Encryption in transit/at rest; field-level encryption for sensitive PII.
- PCI SAQ-A via tokenization; SOC 2 Type II target.
- Privacy: GDPR, CCPA, CAN-SPAM, TCPA controls.
- HIPAA-eligible deployment (BAA-ready) for health verticals — requires elevated tier.
- Per-tenant data residency option (mid-market / enterprise).
- Comprehensive audit log of data access and changes.

---

## 4. Cross-Module User Journeys (Illustrative)

### 4.1 Dental Practice — Patient Recall to Paid Visit

1. **Plan** auto-generates a 6-month recall appointment for a Cleaning Plan member.
2. **Engage** runs the recall cadence (email Day 0, SMS Day 7, AI-timed retry on no-response).
3. Patient self-books via **Portal** (multi-resource: hygienist + chair + dentist for exam segment).
4. **Engage** sends pre-visit reminders; no-show risk score is medium → require card-on-file.
5. Patient arrives, **Jobs** records procedures, charts updates, X-ray attachments.
6. **Commerce** generates an invoice with treatment plan items; **Pay** collects co-pay via terminal.
7. **Engage** sends review request; **Insights** updates production-per-chair-hour KPI.

### 4.2 Mobile HVAC — Maintenance Plan Visit

1. **Plans**: residential customer's annual tune-up entitlement triggers a visit creation in **Schedule**.
2. **Schedule** assigns to a zone-eligible tech with capacity; **Engage** confirms via SMS.
3. Tech arrives, completes **Jobs** checklist, sells a humidifier upgrade from **Commerce**.
4. **Pay** collects payment in-field via mobile terminal; entitlement is debited in **Plans**.
5. **Workforce** logs tech's time; **Insights** updates revenue-per-route-hour.

### 4.3 Yoga Studio — Class Pass & Walk-In

1. New customer buys a 10-class pass online via **Commerce**; **Pay** processes the card.
2. **Plans** records 10 class credits on the contact.
3. Customer self-books a class; **Schedule** decrements available seats.
4. On check-in, **Jobs** marks attendance, **Plans** debits 1 credit.
5. **Engage** sends a win-back at 8 credits remaining and at expiration −14 days.
6. **Workforce** confirmed the instructor's shift covered the class slot.

---

## 5. Industry Packs (Configuration Bundles)

To avoid forking the codebase per vertical, an **Industry Pack** is a versioned bundle of:

- Terminology overrides (e.g., "Customer" → "Patient").
- Default custom fields (e.g., insurance carrier for dental).
- Default appointment types, job types, checklists, forms.
- Default catalog templates and price books.
- Default cadences (e.g., dental 6-month recall).
- Default report packs and KPIs.
- Default roles and permissions.
- Required compliance toggles (HIPAA for health packs).

Initial packs to ship: Home Services, Dental, Med Spa, Salon/Barber, Veterinary, Auto Repair, Legal, Tutoring/Studio, Generic SMB.

---

## 6. What's Removed or Repurposed from v1

| v1 Item | Disposition in v2 |
|---|---|
| HVAC/Plumbing/Electrical-only narrative | Repositioned as one of many supported verticals (Home Services pack). |
| Dispatch-centric routing as a separate module | Merged into Schedule; routing remains a feature toggled on for field-service verticals. |
| Work Order terminology | Generalized as "Jobs/Cases/Visits"; preserved for field-service via terminology overrides. |
| IoT/Predictive Maintenance | Remains as an opt-in MicroSaaS add-on (still vertical-specific). |

---

## 7. AI-Enabled Features (Generalized & Expanded)

- **Smart scheduling**: fill-rate optimization, double-book recommendations, no-show prediction.
- **Cadence intelligence**: send-time optimization, channel selection, fatigue management.
- **Catalog & pricing AI**: dynamic price recommendations, bundle suggestions, upsell ranking.
- **Conversational AI**: copilot for staff (drafts replies, summarizes accounts) and chatbot for customers (book, reschedule, FAQ).
- **Forecasting**: revenue, demand, labor, inventory.
- **Risk scoring**: no-show, churn, payment failure, fraud.
- **Document AI**: parse uploaded PDFs (insurance card, supplier invoice) into structured data.
- **Voice AI** (optional add-on): inbound call answering, appointment booking by phone.

---

## 8. Non-Functional Requirements

- **Performance**: P95 < 300ms for primary lookups at 100K contacts/org; appointment slot search < 600ms.
- **Scale**: 10K active tenants on a single Supabase project with read replicas; per-tenant data partitioned logically by `org_id`.
- **Availability**: 99.9% target for core platform; 99.95% for Pay critical path.
- **Recoverability**: RPO ≤ 5 min, RTO ≤ 1 hour for primary database.
- **Extensibility**: every domain entity has a `metadata jsonb` column for tenant-defined fields; every entity emits a webhook event on change.
- **Observability**: per-tenant usage telemetry feeds **Servexa Meter** (see §10).

---

## 9. Suggested Module Roadmap

| Phase | Modules |
|---|---|
| **Phase 1 (MVP, horizontal)** | Identity, CRM, Schedule (appointments + resources), Jobs, Commerce (catalog + invoicing), Pay (card + ACH), Engage (reminders + no-show), Portal. |
| **Phase 2** | Plans (memberships/contracts), Workforce (shifts + time clock), Insights v1, first 3 Industry Packs. |
| **Phase 3** | Online storefront, in-person POS terminals, marketing automation, advanced AI (voice, forecasting), Inventory, additional Industry Packs. |
| **Phase 4** | SSO/SAML, SCIM, HIPAA-eligible tier, IoT add-on re-launch, marketplace/app directory. |

---

## 10. Instrumentation & Consumption-Based Pricing

This is a foundational requirement for v2: every meaningful unit of value must be **measurable, attributable, and rateable**. Servexa Meter is the cross-cutting service that does this.

### 10.1 Design Principles

1. **Meter at the source of truth**: each module emits usage events via Postgres triggers and Edge Functions; events are written to an immutable `usage_events` table partitioned by month.
2. **Idempotent**: every event has a deterministic dedupe key (`org_id + meter + source_entity_id + event_uuid`).
3. **Real-time + aggregated**: hot-path counters in Redis (or Postgres advisory) for entitlement checks; cold aggregates in `usage_daily_rollups` for billing.
4. **Transparent**: every tenant can see their own usage in **Servexa Insights → Account Usage** with a meter-by-meter breakdown.
5. **Soft & hard limits**: each meter supports warning thresholds (notify) and hard caps (block) per plan; overage allowed with overage rate when configured.

### 10.2 Metered Dimensions (Catalog of Meters)

The following list is the canonical set of consumption meters. Each line names the meter, the unit, and the suggested rating model.

| Meter Code | Description | Unit | Rating Approach |
|---|---|---|---|
| `crm.contacts.active` | Distinct contacts touched in 30d | count | Tiered subscription with overage |
| `crm.contacts.total` | Total contacts stored | count | Storage-style tier (e.g., 0–10k included) |
| `crm.imports.records` | Bulk import records | count | Per-1000 records |
| `schedule.appointments.booked` | Appointments created | count | Per-appointment OR included pool + overage |
| `schedule.calendar_syncs.events` | Two-way calendar sync events | count | Included pool + overage |
| `jobs.created` | Job/visit/case records | count | Per-job overage above tier |
| `commerce.catalog.items` | Active catalog items | count | Tier |
| `commerce.orders.created` | Orders created (any channel) | count | Per-order or % of GMV |
| `commerce.gmv` | Gross Merchandise Value | USD | bps (basis points) on GMV |
| `pay.transactions.count` | Payment transactions (auth+capture) | count | Per-transaction fee |
| `pay.transactions.volume` | Payment volume processed | USD | % + per-txn (standard processor model) |
| `pay.terminals.active` | Active physical terminals | count | Per-terminal/month |
| `pay.payouts.count` | Bank payouts | count | Per-payout fee (optional) |
| `engage.messages.sms` | SMS sent | count | Per-message (carrier pass-through + margin) |
| `engage.messages.email` | Emails sent | count | Per-1000 emails |
| `engage.messages.voice_minutes` | Voice/IVR minutes | minutes | Per-minute |
| `engage.messages.push` | Push notifications | count | Per-1000 |
| `engage.cadences.active` | Active cadences | count | Tier |
| `engage.reviews.collected` | Review requests sent | count | Per-1000 |
| `plans.active_subscribers` | Active plan/contract subscribers | count | Per-subscriber/month |
| `plans.entitlement_redemptions` | Entitlement consumptions | count | Per-event (low cost) |
| `workforce.staff.scheduled` | Staff members on roster | count | Per-staff/month (a primary seat-equivalent meter) |
| `workforce.timeclock.punches` | Clock-in/out events | count | Per-1000 |
| `portal.bookings.online` | Self-service bookings | count | Per-booking |
| `portal.users.active` | Distinct portal logins/month | count | Tier |
| `insights.reports.scheduled` | Scheduled reports delivered | count | Per-100 |
| `insights.ai.queries` | AI/LLM queries from Insights ("ask your data") | count | Per-query (cost recovery) |
| `ai.tokens.consumed` | Total LLM tokens used (cross-module AI) | tokens | Per-1M tokens |
| `ai.voice.minutes` | Voice-AI minutes | minutes | Per-minute |
| `iot.assets.connected` | Connected IoT assets | count | Per-device/month |
| `storage.attachments.gb` | Storage usage | GB-month | Tier + per-GB overage |
| `api.requests` | External API calls | count | Per-million |
| `webhooks.deliveries` | Outbound webhook deliveries | count | Per-million |
| `integrations.sync.records` | Records synced to/from accounting | count | Per-10k |
| `industry_pack.installed` | Number of industry packs enabled | count | Flat per-pack/month |

### 10.3 Pricing Model Options

Three commercially viable pricing shapes are supported and can be combined per tenant:

**A. Tiered Subscription + Included Pool + Overage** (default)

- A monthly base price includes a pool for each meter (e.g., Starter: 1,000 SMS, 10,000 emails, 500 appointments).
- Overage charged on a per-unit basis above the pool.
- Pro: predictable for tenant; familiar SaaS shape.

**B. Pure Consumption / Pay-As-You-Go**

- No (or minimal) base fee; each meter rated per event/unit.
- Best for very small businesses or seasonal/episodic users.
- Pro: zero barrier to adoption; revenue scales with tenant activity.

**C. Outcome / GMV-Based**

- Servexa earns a % of payment volume (`pay.transactions.volume`) and/or a % of commerce GMV (`commerce.gmv`).
- Often combined with low/zero base fee; the platform's success is tied to the tenant's.
- Pro: aligns incentives; very compelling for the smallest businesses.

### 10.4 Suggested Packaged Plans

These are illustrative plan SKUs showing how the meters compose into market-ready offers.

| Plan | Monthly Base | Included Pools | Best For |
|---|---|---|---|
| **Servexa Solo** | $0 + usage | 100 appts, 200 SMS, 1,000 emails, 1 staff | Sole proprietors testing the waters |
| **Servexa Starter** | $49 | 500 appts, 1,000 SMS, 10,000 emails, 3 staff, 5 GB | Solo + 1–2 employees |
| **Servexa Growth** | $199 | 2,500 appts, 5,000 SMS, 50,000 emails, 10 staff, 25 GB, 1 Industry Pack | Established SMB |
| **Servexa Scale** | $599 | 10,000 appts, 25,000 SMS, 250,000 emails, 30 staff, 100 GB, 3 packs, SSO | Multi-location SMB |
| **Servexa Enterprise** | Custom | Negotiated | Mid-market, HIPAA, SAML, custom DPA |
| **Servexa Pay-as-you-go** | $0 | None — pure metered | Variable seasonal, very small biz |
| **Servexa GMV** | $0–$29 | Minimal + take rate on Pay & Commerce | Storefront-heavy tenants |

Per-unit overage rates (examples, illustrative):

- Appointments: $0.10 ea above pool.
- SMS: $0.012 ea (US) + carrier pass-through.
- Email: $0.50 per 1,000.
- Payment transactions: 2.9% + $0.30 (Stripe-equivalent), with Servexa take of +0.4% on the GMV plan.
- Storage: $0.25/GB-month above pool.
- AI tokens: $5 per 1M input / $15 per 1M output (passes through underlying cost + margin).
- Plan subscribers: $0.50/active subscriber/month.
- IoT assets: $2–$8/device/month depending on telemetry rate.
- Industry Pack: $25/pack/month.

### 10.5 Entitlement & Billing Architecture

Conceptual entities (to be detailed in a follow-up TDD):

- `meter_definitions` — catalog of all meter codes, units, descriptions, default rates.
- `plans` — packaged offers (Starter, Growth, etc.) with included pools per meter.
- `subscriptions` — tenant's active plan, billing cycle, status.
- `entitlements` — per-tenant per-meter pool size and reset cadence.
- `usage_events` — immutable raw events (high-volume, partitioned).
- `usage_daily_rollups` — aggregates per `(org_id, meter_code, date)`.
- `billing_periods` — closed periods, rated and invoiced.
- `rated_items` — per period per meter computed charges.
- `platform_invoices` — Servexa's invoice to the tenant (separate from tenant invoices to *their* customers).

Billing engine flow:

1. Modules emit events → `usage_events`.
2. Nightly job rolls up into `usage_daily_rollups`.
3. End-of-period job rates rollups against the tenant's plan and overage rates.
4. Generates `platform_invoice` and charges the tenant's stored Servexa payment method.
5. Pushes a portal-visible breakdown ("here's where your usage went") to the tenant's billing page.

### 10.6 Reporting & Transparency

Every tenant sees, in real-time:

- **Current period usage** vs. included pool for each meter.
- **Projected end-of-period bill**.
- **Drill-down**: e.g., which cadences drove SMS, which catalog items drove orders.
- **Alerts**: 50/80/100% threshold notifications, hard-cap warnings.
- **Cost-saver suggestions**: e.g., "you sent 4,200 SMS — moving to Growth plan would save $X."

### 10.7 Fairness, Anti-Abuse & Edge Cases

- **Test mode**: events tagged as test are not metered.
- **Soft-delete**: deleting a contact does not refund counts in past periods (consistent with most SaaS); explicit policies documented.
- **Bulk imports**: rated at a discounted rate vs. one-by-one creation.
- **Inbound vs. outbound**: inbound SMS is free to the tenant (Servexa absorbs carrier cost on the inbound side as a goodwill primitive); outbound is rated.
- **Failed sends / declined charges**: not metered.
- **AI cost passthrough**: tenants on Solo/Starter can be auto-throttled if their AI spend exceeds a "free AI" budget; upgrades unlock higher ceilings.

---

## 11. Open Questions

- **Default vs. opt-in modules**: should Workforce, Plans, and Commerce be on by default for all tenants or opt-in via Industry Pack?
- **Pay margins**: do we always pass payment processor fees through at cost (with platform fee on top), or absorb at lower tiers to drive adoption?
- **Industry Pack pricing**: bundled into Growth/Scale or always add-on?
- **Multi-currency / international**: how aggressively to support non-USD billing of the platform itself?
- **Data residency**: which regions in first 12 months (US, EU, AU)?
- **HIPAA**: ship in v2 or defer to Phase 4?

---

## 12. Conclusion

By generalizing the original HVAC/Plumbing/Electrical platform into a **Common Business Services Layer**, Servexa addresses an order-of-magnitude larger TAM while preserving and re-using nearly all of the engineering investment already documented in `fdd_0…fdd_8` and `fdd_iot.md`. The addition of first-class **Appointment Scheduling**, **Digital Payments**, **Follow-Up & No-Show Automation**, **Staff & Shift Scheduling**, **Maintenance Contracts/Plans**, and a **Products & Services Catalog + Commerce** rounds out the operational backbone that virtually every SMB needs.

Combined with a fully **instrumented, consumption-based pricing model** powered by Servexa Meter, the platform can be sold to a sole proprietor on pure pay-as-you-go pricing and scaled to a 250-person multi-location SMB on a tiered plan — all from a single product configuration, with Industry Packs supplying the vertical personality on top.
