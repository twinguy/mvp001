# Technical Design Document – Section 4: Quoting & Invoicing

## 1. Purpose & Scope

This Technical Design Document (TDD) defines the **Quoting & Invoicing** capabilities for the HVAC, Plumbing, and Electrical services platform described in `functional.md`. It is intended to be detailed enough for engineers and LLM-based task generators to break the module into concrete implementation tasks.

- **In-scope capabilities (from `functional.md`, §4 Quoting & Invoicing)**:
  - Automated quote generation based on service catalog.
  - E-signature support for approvals.
  - Integrated invoicing and payment processing.
  - AI-driven pricing recommendations.
- **Additional derived capabilities needed for a production-ready module**:
  - Service catalog and price book management (labor, materials, bundles, fees).
  - Taxes, surcharges, and discounts (line-level and document-level).
  - Deposits and progress billing (partial invoicing vs quote total).
  - Flexible payment terms (due on receipt, Net 15/30, etc.).
  - Refunds, credits, and write-offs (basic AR operations).
  - PDF and email generation for quotes/invoices.
  - Customer-facing quote acceptance/decline and invoice payment UX (portal and mobile).
  - Integration hooks with Work Orders, Scheduling, Inventory, and external accounting systems.
- **Out-of-scope for this TDD (but integrated)**:
  - Full accounting/GL, complex tax jurisdiction logic, and financial reporting (handled by external accounting/ERP systems).
  - Detailed inventory costing and asset lifecycle (covered in Inventory & Asset Management TDD).
  - Full dunning/collections workflows beyond basic reminder scheduling.

The Quoting & Invoicing module must be deployable on **Supabase (PostgreSQL + Auth + Storage + Edge Functions)** and consumed by a **Next.js web app on Vercel**, plus future mobile apps, in alignment with `tooling.md`.

## 2. Target Platform & High-Level Architecture

### 2.1 Platform Alignment (from `tooling.md`)

- **Backend: Supabase**
  - PostgreSQL for relational data (service catalog, quotes, invoices, payments).
  - Supabase Auth for users (office staff, technicians, customers via portal).
  - Row-Level Security (RLS) for multi-tenant and role-based access control using `org_id`.
  - Supabase Storage for rendered quote/invoice PDFs and related attachments.
  - Supabase Edge Functions for:
    - Quote auto-generation and recomputation logic.
    - Payment provider orchestration (e.g., Stripe or similar).
    - AI-driven pricing and discount recommendations.
    - PDF rendering and email dispatch.
    - Webhook handling from payment gateways and (optionally) accounting systems.
- **Frontend: Vercel**
  - Next.js application (back-office quoting, AR dashboard, and customer portal experiences).
  - Uses Supabase JS client for auth, database, storage, and real-time updates.
  - Deployed via Vercel with CI/CD and environment separation (dev/stage/prod).
- **Mobile**
  - Future Flutter or React Native technician app using Supabase SDKs.
  - Supports on-site quote creation, signature capture, and collecting payments.

### 2.2 Logical Components

1. **Data Layer (Supabase PostgreSQL)**
   - Service catalog & pricing:
     - `catalog_items`, `catalog_item_prices`, `price_books`, `tax_rates`, `discount_schedules`, `fees`.
   - Quoting:
     - `quotes`, `quote_line_items`, `quote_status_history`, `quote_attachments`, `quote_signatures`.
   - Invoicing & AR:
     - `invoices`, `invoice_line_items`, `invoice_status_history`, `invoice_attachments`.
   - Payments:
     - `payments`, `payment_intents`, `payment_methods` (token references), `payout_accounts` (optional).
   - Integrations:
     - `accounting_integrations`, `external_accounting_links`, `payment_provider_logs`.

2. **Service Layer (Supabase Edge Functions + SQL/RPC)**
   - Encapsulates complex workflows:
     - Quote generation from service catalog, work orders, or maintenance plans.
     - Quote lifecycle enforcement (draft → sent → accepted/declined → converted).
     - Invoice creation from quotes or work order completion (including partial/progress invoices).
     - Tax and total calculation engine.
     - Payment intent creation and handling for cards/ACH/digital wallets.
     - AI-based pricing and discount suggestions.
     - Synchronization with external accounting (QuickBooks, Xero, etc.) where configured.

3. **API Layer**
   - Direct table access via Supabase client for simple reads/writes (e.g., viewing quote lists).
   - Higher-level orchestration via HTTP Edge Functions:
     - `create_quote`, `recalculate_quote`, `send_quote`, `accept_quote`, `decline_quote`.
     - `create_invoice`, `recalculate_invoice`, `send_invoice`.
     - `create_payment_intent`, `record_payment`, `refund_payment`.
     - `sync_invoice_to_accounting`, etc.

4. **UI Layer (Next.js + mobile)**
   - Office/CSR quoting UI (desktop-optimized).
   - Technician mobile on-site quoting and invoicing screens.
   - Customer portal experiences for viewing/approving quotes and paying invoices.

5. **Integrations**
   - Payment providers (e.g., Stripe) via Edge Functions and webhooks.
   - Accounting systems (QuickBooks, Xero, etc.) via Edge Functions and API/webhooks (optional, configurable per org).
   - Shared communication/messaging engine for sending emails/SMS with quote/invoice links.

### 2.3 Multi-Tenancy & Access Control

Assume a multi-tenant Supabase project with `org_id` used across all finance-related tables:

- Every Quoting & Invoicing table includes `org_id` (FK → `orgs.id`).
- **Roles** (high level):
  - **Admin/Owner**: full control over catalog, pricing rules, quotes, invoices, and AR.
  - **CSR/Sales**: create/send quotes, convert to work orders/invoices within their `org_id`.
  - **Technician**: create/adjust on-site quotes within limits (discount caps, price book restrictions); collect payments on invoices assigned to their jobs.
  - **AR/Billing Specialist**: manage invoicing, payments, refunds, and accounting sync.
  - **Customer (portal)**: view their own quotes/invoices, approve/decline, and pay.
- RLS policies:
  - Enforce `org_id` isolation across tables.
  - Restrict quote/invoice visibility for staff based on role (e.g., techs see only their assigned or work-order-linked artifacts).
  - Customers restricted to quotes/invoices where `customer_id` matches their CRM identity.

## 3. Domain Model & Data Design

### 3.1 Entity Overview

Primary domain concepts:

- **Catalog Item**: Sellable unit—service, product/part, fee, or bundle—with base pricing and tax treatment.
- **Price Book**: Contextual pricing set (e.g., residential vs commercial, standard vs after-hours).
- **Quote**: Offer to perform work for a specific price and terms; may be generated from catalog, AI suggestions, or work orders.
- **Quote Line Item**: Individual line (labor, parts, discount, fee) that composes a quote total.
- **Invoice**: Bill for work performed (may be derived from a quote or directly from work order data).
- **Invoice Line Item**: Individual billable lines on an invoice.
- **Payment Intent/Transaction**: Representation of an attempted or completed payment via a provider.
- **Payment**: Confirmed settlement of funds applied to one or more invoices.
- **Signature**: E-signature artifacts on quotes or invoices (customer approvals, technician confirmations).
- **External Accounting Link**: Mapping between internal quotes/invoices/payments and external accounting/ERP entities.

### 3.2 Tables & Columns (Conceptual Schema)

> Note: types are conceptual; actual implementation uses Supabase/Postgres DDL and constraints. All tables include `created_at` and `updated_at` timestamps unless noted.

#### 3.2.1 `orgs`

Shared tenant table as defined in other modules; reused here.

- `id` (UUID, PK).
- `name` (text, not null).
- `created_at` (timestamptz, default now()).

#### 3.2.2 `catalog_items`

Master record for services, parts, fees, and bundles used in quotes/invoices.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `code` (text, nullable) – SKU or internal code.
- `name` (text, not null).
- `description` (text, nullable).
- `item_type` (enum: `service`, `part`, `fee`, `bundle`).
- `unit_of_measure` (text, nullable) – e.g., "hour", "unit", "visit".
- `base_unit_price` (numeric, nullable) – fallback price.
- `is_taxable` (boolean, default true).
- `default_tax_rate_id` (UUID, FK → `tax_rates.id`, nullable).
- `default_price_book_id` (UUID, FK → `price_books.id`, nullable).
- `is_active` (boolean, default true).
- `metadata` (jsonb, nullable) – extra attributes (warranty flags, category mapping).

Indexes:
- `idx_catalog_items_org_id_code`.
- Unique: (`org_id`, `code`) where `code` is not null.

#### 3.2.3 `price_books`

Named sets of prices (e.g., "Standard Residential", "Commercial", "After Hours").

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `name` (text, not null).
- `description` (text, nullable).
- `currency` (text, not null, default `USD`).
- `is_default` (boolean, default false).
- `applies_to_customer_type` (enum: `any`, `residential`, `commercial`).
- `is_active` (boolean, default true).

Unique:
- (`org_id`, `name`).

#### 3.2.4 `catalog_item_prices`

Overrides and price rules per catalog item and price book.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `catalog_item_id` (UUID, FK → `catalog_items.id`, not null, on delete cascade).
- `price_book_id` (UUID, FK → `price_books.id`, not null, on delete cascade).
- `unit_price` (numeric, not null).
- `min_quantity` (numeric, nullable) – for volume pricing.
- `max_quantity` (numeric, nullable).
- `effective_from` (timestamptz, nullable).
- `effective_to` (timestamptz, nullable).
- `metadata` (jsonb, nullable).

Unique:
- (`org_id`, `catalog_item_id`, `price_book_id`, `effective_from`, `effective_to`).

#### 3.2.5 `tax_rates`

Basic tax rate definitions; complex jurisdictional logic is out of scope.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `name` (text, not null) – e.g., "NY State + Local".
- `rate_percent` (numeric, not null) – e.g., 8.875.
- `tax_code` (text, nullable) – for external systems.
- `is_default` (boolean, default false).
- `is_active` (boolean, default true).

Unique:
- (`org_id`, `name`).

#### 3.2.6 `discount_schedules`

Predefined discount structures for quotes and invoices.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `name` (text, not null) – e.g., "Military Discount 10%".
- `discount_type` (enum: `percentage`, `fixed_amount`).
- `value` (numeric, not null).
- `applies_to_scope` (enum: `line_item`, `document_total`).
- `max_uses_per_quote` (integer, nullable).
- `requires_approval` (boolean, default false).
- `is_active` (boolean, default true).

Unique:
- (`org_id`, `name`).

#### 3.2.7 `quotes`

Core quote entity.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `customer_id` (UUID, FK → `customers.id`, not null).
- `location_id` (UUID, FK → `customer_locations.id`, nullable).
- `work_order_id` (UUID, FK → `work_orders.id`, nullable) – if derived from job.
- `external_ref` (text, nullable) – link to legacy or accounting system.
- `quote_number` (text, not null) – human-readable identifier.
- `status` (enum: `draft`, `awaiting_approval`, `sent`, `viewed`, `accepted`, `declined`, `expired`, `converted_to_work_order`).
- `title` (text, nullable).
- `description` (text, nullable) – scope of work summary.
- `price_book_id` (UUID, FK → `price_books.id`, nullable).
- `subtotal` (numeric, not null, default 0).
- `discount_total` (numeric, not null, default 0).
- `tax_total` (numeric, not null, default 0).
- `grand_total` (numeric, not null, default 0).
- `currency` (text, not null, default `USD`).
- `valid_until` (timestamptz, nullable).
- `terms_and_conditions` (text, nullable).
- `customer_notes` (text, nullable).
- `internal_notes` (text, nullable).
- `created_by_user_id` (UUID, FK → `auth.users`, nullable).
- `approved_by_user_id` (UUID, FK → `auth.users`, nullable).
- `approved_at` (timestamptz, nullable).
- `sent_at` (timestamptz, nullable).
- `viewed_at` (timestamptz, nullable).
- `accepted_at` (timestamptz, nullable).
- `declined_at` (timestamptz, nullable).

Indexes:
- `idx_quotes_org_id_status`.
- `idx_quotes_org_id_customer_id`.
- `idx_quotes_org_id_quote_number`.

#### 3.2.8 `quote_line_items`

Line items composing a quote total.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `quote_id` (UUID, FK → `quotes.id`, not null, on delete cascade).
- `catalog_item_id` (UUID, FK → `catalog_items.id`, nullable).
- `line_type` (enum: `product`, `service`, `fee`, `discount`, `custom_text`).
- `description` (text, not null) – may override catalog name.
- `quantity` (numeric, not null, default 1).
- `unit_price` (numeric, not null, default 0).
- `line_subtotal` (numeric, not null, default 0).
- `discount_type` (enum: `none`, `percentage`, `fixed_amount`).
- `discount_value` (numeric, not null, default 0).
- `discount_amount` (numeric, not null, default 0).
- `tax_rate_id` (UUID, FK → `tax_rates.id`, nullable).
- `tax_amount` (numeric, not null, default 0).
- `line_total` (numeric, not null, default 0).
- `sort_order` (integer, default 0).
- `is_optional` (boolean, default false) – optional upsell items.
- `is_selected` (boolean, default true) – customer selection for optional items.
- `metadata` (jsonb, nullable) – links to inventory SKUs, warranty, etc.

Index:
- `idx_quote_line_items_quote_id_sort_order`.

#### 3.2.9 `quote_status_history`

Audit of quote status transitions.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `quote_id` (UUID, FK → `quotes.id`, not null, on delete cascade).
- `previous_status` (enum same as `quotes.status`, nullable).
- `new_status` (enum same as `quotes.status`, not null).
- `changed_by_user_id` (UUID, FK → `auth.users`, nullable).
- `changed_at` (timestamptz, default now()).
- `reason` (text, nullable).
- `metadata` (jsonb, nullable) – context (channel, IP, device).

Index:
- `idx_quote_status_history_quote_id_changed_at`.

#### 3.2.10 `quote_signatures`

Stores e-signature metadata and artifacts for quotes.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `quote_id` (UUID, FK → `quotes.id`, not null, on delete cascade).
- `signed_by_role` (enum: `customer`, `internal_approver`, `other`).
- `signed_by_name` (text, nullable).
- `signed_by_email` (text, nullable).
- `signature_image_path` (text, nullable) – Supabase Storage path.
- `signature_payload` (jsonb, nullable) – vector data or drawing captures.
- `signed_at` (timestamptz, not null).
- `metadata` (jsonb, nullable) – IP, user-agent, geo, etc.

#### 3.2.11 `quote_attachments`

Attachments associated with a quote (e.g., spec sheets).

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `quote_id` (UUID, FK → `quotes.id`, not null, on delete cascade).
- `storage_bucket` (text, not null).
- `storage_path` (text, not null).
- `file_name` (text, not null).
- `content_type` (text, nullable).
- `file_size_bytes` (bigint, nullable).
- `is_customer_visible` (boolean, default true).

Index:
- `idx_quote_attachments_quote_id`.

#### 3.2.12 `invoices`

Core invoice entity.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `customer_id` (UUID, FK → `customers.id`, not null).
- `location_id` (UUID, FK → `customer_locations.id`, nullable).
- `work_order_id` (UUID, FK → `work_orders.id`, nullable).
- `quote_id` (UUID, FK → `quotes.id`, nullable).
- `external_ref` (text, nullable) – accounting system invoice ID/number.
- `invoice_number` (text, not null).
- `status` (enum: `draft`, `sent`, `viewed`, `partial_paid`, `paid`, `overdue`, `void`, `refunded`, `written_off`).
- `title` (text, nullable).
- `description` (text, nullable).
- `subtotal` (numeric, not null, default 0).
- `discount_total` (numeric, not null, default 0).
- `tax_total` (numeric, not null, default 0).
- `grand_total` (numeric, not null, default 0).
- `amount_paid` (numeric, not null, default 0).
- `amount_due` (numeric, not null, default 0).
- `currency` (text, not null, default `USD`).
- `due_date` (timestamptz, nullable).
- `payment_terms` (text, nullable) – e.g., "Due on receipt", "Net 30".
- `terms_and_conditions` (text, nullable).
- `customer_notes` (text, nullable).
- `internal_notes` (text, nullable).
- `created_by_user_id` (UUID, FK → `auth.users`, nullable).
- `sent_at` (timestamptz, nullable).
- `viewed_at` (timestamptz, nullable).
- `paid_at` (timestamptz, nullable).
- `written_off_at` (timestamptz, nullable).

Indexes:
- `idx_invoices_org_id_status`.
- `idx_invoices_org_id_customer_id`.
- `idx_invoices_org_id_invoice_number`.
- `idx_invoices_org_id_due_date_status`.

#### 3.2.13 `invoice_line_items`

Line items composing the invoice total (often derived from quotes or work orders).

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `invoice_id` (UUID, FK → `invoices.id`, not null, on delete cascade).
- `quote_line_item_id` (UUID, FK → `quote_line_items.id`, nullable).
- `catalog_item_id` (UUID, FK → `catalog_items.id`, nullable).
- `line_type` (enum: `product`, `service`, `fee`, `discount`, `custom_text`).
- `description` (text, not null).
- `quantity` (numeric, not null, default 1).
- `unit_price` (numeric, not null, default 0).
- `line_subtotal` (numeric, not null, default 0).
- `discount_type` (enum: `none`, `percentage`, `fixed_amount`).
- `discount_value` (numeric, not null, default 0).
- `discount_amount` (numeric, not null, default 0).
- `tax_rate_id` (UUID, FK → `tax_rates.id`, nullable).
- `tax_amount` (numeric, not null, default 0).
- `line_total` (numeric, not null, default 0).
- `sort_order` (integer, default 0).
- `metadata` (jsonb, nullable) – links to work order tasks, materials logs.

Index:
- `idx_invoice_line_items_invoice_id_sort_order`.

#### 3.2.14 `invoice_status_history`

Audit of invoice status transitions.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `invoice_id` (UUID, FK → `invoices.id`, not null, on delete cascade).
- `previous_status` (enum same as `invoices.status`, nullable).
- `new_status` (enum same as `invoices.status`, not null).
- `changed_by_user_id` (UUID, FK → `auth.users`, nullable).
- `changed_at` (timestamptz, default now()).
- `reason` (text, nullable).
- `metadata` (jsonb, nullable).

Index:
- `idx_invoice_status_history_invoice_id_changed_at`.

#### 3.2.15 `payments`

Confirmed payments applied to invoices.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `customer_id` (UUID, FK → `customers.id`, not null).
- `invoice_id` (UUID, FK → `invoices.id`, nullable) – may be split across multiple invoices.
- `amount` (numeric, not null).
- `currency` (text, not null, default `USD`).
- `payment_method_type` (enum: `card`, `ach`, `cash`, `check`, `other`).
- `payment_provider` (text, nullable) – e.g., `stripe`, `square`.
- `payment_provider_payment_id` (text, nullable).
- `status` (enum: `pending`, `succeeded`, `failed`, `refunded`, `partially_refunded`, `canceled`).
- `received_at` (timestamptz, nullable).
- `refunded_amount` (numeric, not null, default 0).
- `notes` (text, nullable).
- `metadata` (jsonb, nullable) – provider payload snapshot.

Indexes:
- `idx_payments_org_id_customer_id`.
- `idx_payments_org_id_invoice_id`.

> For payments that need to be allocated across multiple invoices, a separate `payment_allocations` table can be added in a future iteration.

#### 3.2.16 `payment_intents`

Represents intents/sessions created with payment providers before confirmation.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `invoice_id` (UUID, FK → `invoices.id`, not null).
- `customer_id` (UUID, FK → `customers.id`, not null).
- `payment_provider` (text, not null).
- `provider_intent_id` (text, not null).
- `client_secret` (text, nullable) – if used by provider.
- `amount` (numeric, not null).
- `currency` (text, not null, default `USD`).
- `status` (enum: `created`, `requires_action`, `succeeded`, `canceled`, `expired`).
- `created_by_user_id` (UUID, FK → `auth.users`, nullable).
- `metadata` (jsonb, nullable).

Index:
- `idx_payment_intents_org_id_invoice_id`.

#### 3.2.17 `payment_methods`

Stored references to customer payment methods via provider tokens (no raw card data).

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `customer_id` (UUID, FK → `customers.id`, not null).
- `payment_provider` (text, not null).
- `provider_customer_id` (text, not null).
- `provider_payment_method_id` (text, not null).
- `method_type` (enum: `card`, `ach`, `wallet`, `other`).
- `last4` (text, nullable).
- `expiry_month` (integer, nullable).
- `expiry_year` (integer, nullable).
- `brand` (text, nullable).
- `is_default` (boolean, default false).
- `is_active` (boolean, default true).

Unique:
- (`org_id`, `customer_id`, `payment_provider`, `provider_payment_method_id`).

#### 3.2.18 `accounting_integrations`

Configuration for external accounting system connections.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `provider` (enum: `quickbooks_online`, `xero`, `other`).
- `access_token` (text, encrypted/managed via secrets).
- `refresh_token` (text, encrypted, nullable).
- `expires_at` (timestamptz, nullable).
- `company_id` (text, nullable) – external account identifier.
- `is_active` (boolean, default true).
- `metadata` (jsonb, nullable).

#### 3.2.19 `external_accounting_links`

Mapping of internal entities to external accounting IDs.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `entity_type` (enum: `invoice`, `payment`, `customer`, `item`).
- `entity_id` (UUID, not null) – internal ID (e.g., `invoices.id`).
- `accounting_integration_id` (UUID, FK → `accounting_integrations.id`, not null).
- `external_id` (text, not null).
- `external_number` (text, nullable).
- `last_synced_at` (timestamptz, nullable).
- `sync_status` (enum: `synced`, `pending`, `failed`).
- `error_message` (text, nullable).

Index:
- `idx_external_accounting_links_org_id_entity_type_entity_id`.

### 3.3 Relationships to Other Modules

- **CRM (TDD §1)**:
  - `quotes.customer_id`, `invoices.customer_id` → `customers.id`.
  - Quote/invoice creation and acceptance/pay events can create `crm_interactions` (e.g., "Quote sent", "Invoice paid").
- **Work Order Management (TDD §3)**:
  - `quotes.work_order_id`, `invoices.work_order_id` link financial docs to jobs.
  - Work order completion can trigger automatic invoice creation.
- **Scheduling & Dispatch (TDD §2)**:
  - Appointment completion and visit outcomes can inform billable time/fees in invoices.
- **Inventory & Asset Management (future TDD)**:
  - `catalog_items` and line items reference parts and materials; inventory module consumes invoice/quote data for usage estimates.
- **Customer Portal (future TDD)**:
  - Portal exposes filtered views for customer quotes/invoices and payment actions.

## 4. API & Edge Function Design

This section defines logical APIs and flows; implementation may use Supabase RPC or HTTP Edge Functions.

### 4.1 Authentication & Authorization

- All Quoting & Invoicing APIs require:
  - Authenticated Supabase user (staff/technician) or secure customer portal token.
  - `org_id` derived from user profile/JWT and enforced via RLS.
- Role-based access:
  - Only authorized roles can modify pricing, send quotes/invoices, and process payments/refunds.
  - Customers can only view/act on their own quotes/invoices with limited fields.

### 4.2 Quote Lifecycle APIs

#### 4.2.1 Create Quote

- **Endpoint**: Edge Function `POST /quotes`
- **Input**:
  - `customer_id`, optional `location_id`.
  - Optional `work_order_id` (for scope-from-job).
  - Initial line items: list of `{ catalog_item_id | custom_description, quantity, price_book_id, discount }`.
  - Optional `price_book_id`, `valid_until`, `terms_and_conditions`, `customer_notes`.
- **Behavior**:
  - Validate customer and location belong to caller’s `org_id`.
  - Select appropriate price book (from input, customer type, or org default).
  - Resolve catalog prices using `catalog_item_prices` and apply overrides.
  - Insert `quotes` (status = `draft` by default) and `quote_line_items`.
  - Compute and persist financial fields (`subtotal`, `discount_total`, `tax_total`, `grand_total`).
  - Optionally trigger AI pricing suggestions (see §5.1).
  - Emit `quote.created` internal event.

#### 4.2.2 Update Quote & Recalculate

- **Endpoint**: `PATCH /quotes/:id`
- **Input**:
  - Partial updates: header fields (title, notes, terms, price book) and line-item changes (add/update/remove).
- **Behavior**:
  - Enforce that only quotes in allowed statuses (`draft`, `awaiting_approval`) can be edited (configurable).
  - On structural changes, recompute line item totals and quote-level totals via shared calc engine.
  - Optionally re-run AI pricing to propose updated discounts/alternatives.
  - Append to `quote_status_history` if status changes.

#### 4.2.3 Send Quote

- **Endpoint**: `POST /quotes/:id/send`
- **Behavior**:
  - Generate or refresh a PDF via Edge Function and store in Supabase Storage.
  - Generate a secure customer-facing URL (with token) for online quote view/approval.
  - Trigger email/SMS send via messaging service using quote templates.
  - Update `quotes.status` to `sent` and set `sent_at`.
  - Record history in `quote_status_history`.

#### 4.2.4 Customer View & Accept/Decline Quote

- **Endpoint**: `GET /portal/quotes/:public_token`
  - Returns a read-only view for the customer: line items, totals, terms, and approval controls.
- **Endpoint**: `POST /portal/quotes/:public_token/accept`
- **Endpoint**: `POST /portal/quotes/:public_token/decline`
- **Behavior**:
  - Validate token and map to `quote_id`, ensuring it belongs to the correct `customer_id`.
  - On **accept**:
    - Update `quotes.status` to `accepted`, set `accepted_at`.
    - Capture e-signature via `quote_signatures` (name, signature image, IP, timestamp).
    - Optionally create or update a `work_order` and/or schedule jobs based on org rules.
    - Emit events: `quote.accepted`, `work_order.created_from_quote` (if applicable).
  - On **decline**:
    - Set `quotes.status = declined`, set `declined_at`, capture reason if provided.
    - Emit `quote.declined` event.

### 4.3 Invoice Lifecycle APIs

#### 4.3.1 Create Invoice

- **Endpoint**: `POST /invoices`
- **Input**:
  - `customer_id`, optional `location_id`.
  - Optional `quote_id` and/or `work_order_id`.
  - Optional mode: `from_quote_full`, `from_quote_partial`, `from_work_order_summary`, or `manual`.
  - For partial billing: mapping of which quote line items and proportions/quantities to include.
- **Behavior**:
  - For quote-based invoices:
    - Pull accepted quote line items; filter/transform per requested mode.
    - Maintain mapping to `quote_line_items` for reconciliation.
  - For work-order-based invoices:
    - Pull checklist outputs/materials/labor logs from Work Order module (where available).
  - Create `invoices` and associated `invoice_line_items`.
  - Run pricing/tax calculations to populate totals and `amount_due`.
  - Set `status = draft`.
  - Emit `invoice.created` event.

#### 4.3.2 Update & Recalculate Invoice

- **Endpoint**: `PATCH /invoices/:id`
- **Behavior**:
  - Similar to quote update: restrict edits to allowed statuses (`draft`).
  - Recalculate totals and `amount_due` after changes.
  - If linked to external accounting and already synced, record that invoice has been modified and may require resync.

#### 4.3.3 Send Invoice

- **Endpoint**: `POST /invoices/:id/send`
- **Behavior**:
  - Generate or refresh invoice PDF and store in Supabase Storage.
  - Generate a secure payment link (public token) for customer portal.
  - Dispatch email/SMS via messaging engine with invoice and payment link.
  - Update status to `sent`, set `sent_at`, and append to `invoice_status_history`.

#### 4.3.4 View & Pay Invoice (Customer)

- **Endpoint**: `GET /portal/invoices/:public_token`
  - Returns invoice details and payment options.
- **Endpoint**: `POST /portal/invoices/:public_token/pay`
- **Behavior**:
  - For pay endpoint:
    - Delegate to payment intent creation (see §4.4) and return provider client secret/session info to frontend.
    - Upon provider confirmation (via webhook), record `payments` and update `invoices.amount_paid`, `amount_due`, and `status` (`partial_paid` or `paid`).
    - Emit `invoice.paid` event.

### 4.4 Payment & Billing APIs

#### 4.4.1 Create Payment Intent

- **Endpoint**: `POST /payments/intent`
- **Input**:
  - `invoice_id`, `amount` (optional; defaults to `amount_due`), `payment_method_type`.
  - Optional `save_payment_method` flag.
- **Behavior**:
  - Validate invoice belongs to caller’s `org_id` and is payable (not void/written off).
  - Create `payment_intents` row.
  - Call external payment provider API to create an intent/session.
  - Store provider IDs and `client_secret`.
  - Return client-side configuration for hosted checkout/SDKs.

#### 4.4.2 Payment Provider Webhooks

- **Endpoints**:
  - `POST /webhooks/payments/<provider>`
- **Behavior**:
  - Verify signature/secret from provider.
  - Parse events such as `payment_intent.succeeded`, `payment_intent.canceled`, `charge.refunded`.
  - Update `payment_intents.status`.
  - Insert/Update `payments` with final `status`, `amount`, provider IDs, and timestamps.
  - Adjust `invoices.amount_paid` and `amount_due`, and update invoice `status` appropriately.
  - Optionally store new `payment_methods` tokens if customer consented.

#### 4.4.3 Manual Payments & Adjustments

- **Endpoint**: `POST /invoices/:id/payments`
  - For recording offline payments (cash/check).
- **Endpoint**: `POST /payments/:id/refund`
  - For initiating a full or partial refund via provider and creating a credit note effect.
- **Behavior**:
  - Enforce role-based permissions.
  - For refunds, ensure the provider call is successful before updating internal `payments` and invoice balances.

### 4.5 Accounting Integration APIs

#### 4.5.1 Connect Accounting System

- **Endpoint**: `POST /accounting/connect`
- **Behavior**:
  - Initiate OAuth with accounting provider.
  - Store credentials in `accounting_integrations`.

#### 4.5.2 Sync Invoice & Payment

- **Endpoints**:
  - `POST /accounting/invoices/:id/sync`
  - `POST /accounting/payments/:id/sync`
- **Behavior**:
  - Fetch invoice/payment and map to provider’s schema (customer, items, tax).
  - Call provider API to create/update invoice/payment records.
  - Store identifiers in `external_accounting_links`.
  - Update `sync_status` and `last_synced_at`.

### 4.6 Background Jobs & Maintenance

- Cron-style Edge Functions:
  - **Invoice due/overdue monitor**: periodically mark invoices as `overdue` when past `due_date` and create reminder notifications.
  - **Quote expiry monitor**: mark quotes as `expired` when `valid_until` is in the past.
  - **Accounting sync retries**: retry failed syncs logged in `external_accounting_links`.

## 5. AI & Analytics Design

### 5.1 AI-Driven Pricing & Discount Recommendations

- **Edge Function**: `suggest_quote_pricing`
  - **Inputs**:
    - Draft quote details: line items, customer type, region, historical pricing for similar jobs.
    - Organizational settings: target margin ranges, discount policies.
    - Competitive/market data (optional, if available).
  - **Process**:
    - Call external LLM/AI with structured schema for catalog items and pricing constraints.
    - Receive suggested:
      - Unit prices and bundles.
      - Additional upsell items (optional line items).
      - Discount levels with explanations.
  - **Outputs**:
    - Suggested adjustments stored in `quotes` metadata (e.g., `ai_suggestions`) or a dedicated `quote_ai_suggestions` table (future extension).
    - Optionally auto-apply within admin-defined constraints (e.g., discount ≤ 10%).

### 5.2 AI for Invoicing Summaries & Line Item Derivation

- **Edge Function**: `generate_invoice_summary`
  - Summarizes work order checklists, materials, and labor logs into an invoice-ready narrative (e.g., for customer invoice notes).
- **Edge Function**: `derive_line_items_from_work_order`
  - Given `work_order_id`, uses structured data plus free-text notes to propose invoice line items (services, parts, fees).

### 5.3 Analytics & KPIs

- Derived views and materialized views:
  - Quote conversion rates by job type, technician, channel.
  - Average discount percentages and margin impact.
  - Days sales outstanding (DSO) and aging buckets.
  - Payment method mix (card vs ACH vs others).
- Implementation tasks:
  - Create SQL views such as `vw_quote_kpis`, `vw_invoice_aging`.

## 6. Frontend (Next.js / Mobile) UI/UX Design

### 6.1 Back-Office / CSR Web UI

- **Quotes List Page**:
  - Filters by status, customer, creation date, value range, technician/CSR.
  - Indicators for quotes expiring soon and awaiting customer decision.
- **Quote Detail/Edit Page**:
  - Header with customer, location, status, validity, and totals.
  - Line item editor with catalog search, price book selection, discounts, and tax.
  - AI suggestion panel (e.g., "Apply recommended bundle", "Offer 5% discount").
  - Buttons: Save draft, Send to customer, Convert to work order.
- **Invoices List Page**:
  - Filters by status, due date, aging bucket, customer, work order.
  - Overdue badges and quick links to send reminders.
- **Invoice Detail Page**:
  - Header with customer, status, due date, totals, amount paid/due.
  - Line item grid (read-only or editable for drafts).
  - Payment history and links to create new payment or refund.
  - Indicators of accounting sync state.

### 6.2 Technician UI (Web or Mobile)

- **On-Site Quote Creation**:
  - Simple flow to add catalog items, adjust quantities, apply limited discounts, and show total to customer.
  - Capture customer e-signature on device.
  - Option to convert accepted quote directly into a scheduled work order/visit.
- **On-Site Invoicing & Payment**:
  - View invoices linked to assigned work orders.
  - Collect payment via card reader or hosted payment page (deep link to provider).
  - Provide digital receipt and optionally print/email.

### 6.3 Customer Portal UI

- **Quotes**:
  - List of open and past quotes with statuses.
  - Detail view with line items, optional/selected items toggles, terms, and acceptance controls.
  - Ability to leave notes for CSR (e.g., "Can we remove this line?").
- **Invoices**:
  - List of open, paid, and overdue invoices.
  - Detail view with payment history and secure "Pay Now" CTA.
  - Download PDF and view invoice summary.

### 6.4 Data Access Patterns

- Use Supabase JS client for:
  - Reading quote/invoice lists and details with appropriate filters.
  - Light-weight updates to non-critical fields (notes, internal flags).
- Use Edge Functions for:
  - Creation/update flows that require validation and multi-table writes (quote/invoice create + line items).
  - Payment intent creation and provider webhooks.
  - AI-powered pricing and work-order-to-invoice derivation.

## 7. Security, Privacy, and Compliance

### 7.1 RLS & Role Policies

- All tables enforce:
  - `org_id = current_setting('app.current_org_id')::uuid` (or equivalent Supabase pattern).
- Role-level checks:
  - Only allowed staff can modify pricing, apply large discounts, or void/write off invoices.
  - Technicians limited to quotes/invoices linked to their assigned work orders.
  - Customers limited to their own `customer_id` via portal tokens or authenticated sessions.

### 7.2 Payment & PCI Considerations

- No raw card or bank account numbers stored in Postgres.
- All sensitive payment details reside with provider (e.g., Stripe) and are referenced only by provider tokens/IDs.
- Payment webhook endpoints must:
  - Validate signatures/secrets.
  - Avoid logging full provider payloads if they may contain PII; mask as needed.

### 7.3 Media & Document Protection

- Store PDFs and attachments in dedicated Storage buckets with paths including `org_id` and `quote_id`/`invoice_id`.
- Use signed URLs with short TTLs for customer downloads.
- Ensure internal-only documents are not exposed through portal URLs.

### 7.4 Auditing & Traceability

- Maintain immutable histories for:
  - Quote and invoice status changes (`quote_status_history`, `invoice_status_history`).
  - Key payment events via `payments` and `payment_intents`.
- Consider additional audit tables or Supabase logs for critical operations (refunds, write-offs, discount overrides).

## 8. Non-Functional Requirements

- **Performance**:
  - Quote and invoice list views should respond in < 500–800ms for typical orgs (~50k documents).
  - Payment intent creation and confirmation should complete in < 2–3 seconds under normal conditions (excluding external provider latency spikes).
- **Scalability**:
  - Supabase Pro tier recommended for production.
  - Indexes on key filter fields (`status`, `due_date`, `customer_id`, `quote_number`, `invoice_number`) to support large datasets.
- **Reliability**:
  - Idempotent webhooks and payment handling to prevent duplicate charges or double-applied payments.
  - Safe handling of partial failures (e.g., provider success but local DB write fails → retry with idempotency keys).
- **Observability**:
  - Edge Functions must log key events (quote sent, invoice paid, sync errors) with correlation IDs.
  - Metrics for payment success rates, webhook errors, and sync failures.

## 9. Implementation Notes & Open Questions

- **Open Questions**:
  - Exact levels of coupling with external accounting (source of truth for AR vs the platform acting as light AR).
  - How aggressive AI pricing should be and how orgs configure guardrails (max discount, margin thresholds).
  - Support for more advanced billing models (subscriptions, recurring maintenance plans, retainers).
  - Detailed tax compliance needs by region (may require external tax service integration).
  - Whether to implement multi-currency per org or per invoice/quote.
- **Extension Points**:
  - Add `payment_allocations` and `credit_notes` tables for multi-invoice allocations and fine-grained AR.
  - Deeper integration with Inventory for automatic costing and margin analytics.
  - Richer dunning workflows with multi-step reminder sequences and customer self-service payment plans.

This TDD is structured to be consumed by an LLM or engineering team to generate concrete tasks such as DDL migrations, Edge Function implementations, RLS policies, backend services, and frontend/mobile UI for robust Quoting & Invoicing, while integrating cleanly with CRM, Work Order Management, Scheduling, Inventory, and external accounting systems.


