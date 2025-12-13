# Technical Design Document – Section 5: Inventory & Asset Management

## 1. Purpose & Scope

This Technical Design Document (TDD) defines the **Inventory & Asset Management** capabilities for the HVAC, Plumbing, and Electrical services platform described in `functional.md`, implemented on the Supabase + Vercel stack recommended in `tooling.md`. It is designed to be explicit enough that an engineering team or an LLM can decompose it into concrete implementation tasks (DDL, Edge Functions, APIs, UI, and integrations).

- **In-scope capabilities (from `functional.md`, §5 Inventory & Asset Management)**:
  - Real-time inventory tracking.
  - Automated reordering and stock alerts.
  - Asset lifecycle management.
  - AI-based demand forecasting.
- **Additional derived capabilities needed for a production-ready module**:
  - Multi-location inventory (warehouses, vans/trucks, on-site stock rooms).
  - Serialized and non-serialized item tracking (e.g., compressors vs. pipe fittings).
  - Inventory transactions (receipts, issues/consumption, returns, transfers, adjustments).
  - Purchase orders, suppliers, and basic procurement flows.
  - Stock reservation and allocation for quotes/work orders.
  - Customer/site asset registry (equipment installed on-premise) with service history.
  - Warranty tracking (manufacturer and service warranties).
  - Optional IoT/telemetry hooks for connected assets (for future predictive maintenance).
  - Integration touchpoints with CRM, Scheduling & Dispatch, Work Order Management, and Quoting & Invoicing.
- **Out-of-scope for this TDD (but integrated)**:
  - Full procurement/ERP (AP, vendor invoices, landed cost, advanced costing methods).
  - Deep IoT platform design (device provisioning, OTA firmware, complex rules engines).
  - Detailed financial costing and margin analytics (handled in Reporting & Analytics TDD).

The module MUST be compatible with **Supabase (PostgreSQL + Auth + Storage + Edge Functions)** and a **Next.js web app on Vercel**, plus future mobile apps (technician app and customer portal).

## 2. Target Platform & High-Level Architecture

### 2.1 Platform Alignment (from `tooling.md`)

- **Backend: Supabase**
  - PostgreSQL for relational data (inventory items, locations, stock transactions, assets, POs).
  - Supabase Auth for users (admins, warehouse managers, technicians, CSRs).
  - Row-Level Security (RLS) enforcing multi-tenant isolation via `org_id`.
  - Supabase Storage for asset photos, manuals, and related documents.
  - Supabase Edge Functions for:
    - Transactional workflows (stock movements, PO lifecycle).
    - Reorder calculation and stock alert generation.
    - AI demand forecasting and recommended reorder quantities.
    - Asset lifecycle operations (commissioning, decommissioning, replacement suggestions).
    - Integration with Work Orders, Quoting & Invoicing, and external systems (e.g., vendor APIs, IoT services).
- **Frontend: Vercel**
  - Next.js back-office app containing inventory dashboards, PO/receiving screens, and asset registry.
  - Customer portal (read-only asset views where appropriate).
  - Uses Supabase JS client and HTTP Edge Functions.
- **Mobile**
  - Technician app (Flutter/React Native) with:
    - Truck stock views and simple adjustments.
    - Material usage capture directly on work orders.
    - Asset lookup at customer site and editing of limited fields (e.g., meter readings, notes, photos).

### 2.2 Logical Components

1. **Inventory Master Data**
   - Item catalog for stocked parts/materials, connected to `catalog_items` where sellable.
   - Stock locations (warehouses, trucks, staging areas) and optional bins.
2. **Stock Management**
   - Stock levels per item/location.
   - Stock transactions (receipts, issues, transfers, adjustments, returns).
   - Reservation/allocation for open work orders and quotes.
3. **Procurement & Replenishment**
   - Suppliers and item–supplier relationships.
   - Purchase orders and PO line items.
   - Automated reorder suggestions and alerts based on min/max, reorder points, and AI forecasts.
4. **Asset Management**
   - Asset models (equipment types and configurations).
   - Individual assets (e.g., specific rooftop unit at a customer site), including serials, install location, and current status.
   - Asset lifecycle events (installed, serviced, repaired, replaced, decommissioned).
   - Warranty records and service contracts.
   - Optional IoT bindings and telemetry snapshots.
5. **Integrations**
   - Tight coupling with:
     - CRM (customers, locations).
     - Scheduling & Dispatch and Work Orders (material usage and asset service).
     - Quoting & Invoicing (cost vs sale pricing, parts usage).
   - Optional ERP/accounting integrations for POs and vendor bills (future).

### 2.3 Multi-Tenancy & Access Control

- All Inventory & Asset tables include `org_id` (FK → `orgs.id`) and enforce RLS:
  - Staff can only see/manage data for their own `org_id`.
  - Technicians are further restricted to locations and assets relevant to their assigned regions/work orders.
- High-level roles:
  - **Owner/Admin**: full control over inventory, assets, configuration, and AI settings.
  - **Warehouse/Inventory Manager**: manage items, locations, POs, and stock transactions.
  - **Technician**: view item availability, consume materials on jobs, view/update assets on assigned work orders.
  - **CSR/Dispatcher**: view availability to support scheduling and quoting decisions.
  - **Customer (portal)**: view assets installed at their sites and related documentation (read-only).

## 3. Domain Model & Data Design

### 3.1 Entity Overview

- **Inventory Item**: Physical SKU kept in stock (e.g., filters, valves, compressors); may map to a sellable `catalog_item`.
- **Stock Location**: Physical/virtual location storing stock (e.g., main warehouse, technician truck, vendor-consigned stock).
- **Stock Level**: Current on-hand, allocated, and available quantities for an item at a specific location.
- **Stock Transaction**: Auditable movement/adjustment of quantity (receipt, issue to job, transfer, cycle count adjustment).
- **Supplier**: Vendor providing inventory items.
- **Purchase Order (PO)**: Request to supplier to purchase items; includes line items, statuses, and receiving information.
- **Reorder Rule**: Configuration for min/max levels, reorder point, and lead times per item/location.
- **Asset Model**: Template/specification for a class of equipment (brand, model, capacity).
- **Asset**: Individual piece of equipment tracked over lifecycle; often installed at a customer site.
- **Asset Event/History**: Time-stamped events for assets (install, service, fault, replacement).
- **Asset Warranty/Contract**: Warranty terms and coverage, linked to asset and optionally supplier or service agreement.
- **Asset Telemetry Binding** (optional/future): Link to external IoT device or data stream.

> Note: All tables implicitly include `created_at` and `updated_at` timestamps unless otherwise stated.

### 3.2 Core Inventory Tables

#### 3.2.1 `inventory_items`

Represents unique stock-keeping units (SKUs).

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `catalog_item_id` (UUID, FK → `catalog_items.id`, nullable) – link to sellable item.
- `sku` (text, not null) – internal or vendor SKU.
- `name` (text, not null).
- `description` (text, nullable).
- `category` (text, nullable) – e.g., "HVAC > Filters", "Plumbing > Valves".
- `unit_of_measure` (text, not null, e.g., "each", "box", "foot").
- `track_serial_numbers` (boolean, default false).
- `track_lots` (boolean, default false).
- `default_cost` (numeric, nullable) – estimated average cost.
- `default_supplier_id` (UUID, FK → `suppliers.id`, nullable).
- `is_active` (boolean, default true).
- `metadata` (jsonb, nullable) – e.g., hazardous material flags, storage requirements, barcodes.

Indexes:
- `idx_inventory_items_org_id_sku` (unique per `org_id`, `sku`).

#### 3.2.2 `stock_locations`

Defines locations where stock can be stored.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `name` (text, not null) – e.g., "Main Warehouse", "Truck #12".
- `location_type` (enum: `warehouse`, `truck`, `room`, `yard`, `other`).
- `parent_location_id` (UUID, FK → `stock_locations.id`, nullable) – for hierarchies (e.g., warehouse → aisle → bin).
- `address_id` (UUID, FK → `addresses.id`, nullable) – optional for physical address.
- `assigned_user_id` (UUID, FK → `auth.users`, nullable) – for trucks or user-specific lockers.
- `is_active` (boolean, default true).
- `metadata` (jsonb, nullable).

Indexes:
- `idx_stock_locations_org_id_type`.

#### 3.2.3 `stock_levels`

Snapshot of current quantity per item/location.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `inventory_item_id` (UUID, FK → `inventory_items.id`, not null, on delete cascade).
- `stock_location_id` (UUID, FK → `stock_locations.id`, not null, on delete cascade).
- `on_hand_quantity` (numeric, not null, default 0).
- `allocated_quantity` (numeric, not null, default 0) – reserved for open work orders/quotes.
- `available_quantity` (numeric, not null, default 0) – derived or maintained (`on_hand - allocated`).
- `minimum_quantity` (numeric, nullable) – static safety stock threshold.
- `maximum_quantity` (numeric, nullable).
- `reorder_point` (numeric, nullable).
- `reorder_quantity` (numeric, nullable).
- `last_replenished_at` (timestamptz, nullable).
- `last_counted_at` (timestamptz, nullable).

Unique:
- (`org_id`, `inventory_item_id`, `stock_location_id`).

#### 3.2.4 `stock_transactions`

Immutable log of all stock movements and adjustments.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `inventory_item_id` (UUID, FK → `inventory_items.id`, not null).
- `from_location_id` (UUID, FK → `stock_locations.id`, nullable).
- `to_location_id` (UUID, FK → `stock_locations.id`, nullable).
- `transaction_type` (enum: `receipt`, `issue_to_work_order`, `issue_to_quote`, `return_from_work_order`, `transfer`, `adjustment_positive`, `adjustment_negative`, `cycle_count`).
- `work_order_id` (UUID, FK → `work_orders.id`, nullable).
- `quote_id` (UUID, FK → `quotes.id`, nullable).
- `purchase_order_id` (UUID, FK → `purchase_orders.id`, nullable).
- `quantity` (numeric, not null) – positive value; direction inferred from type/from/to.
- `unit_cost` (numeric, nullable) – cost per unit at time of transaction.
- `total_cost` (numeric, nullable).
- `reason` (text, nullable).
- `created_by_user_id` (UUID, FK → `auth.users`, nullable).
- `occurred_at` (timestamptz, default now()).
- `metadata` (jsonb, nullable) – e.g., serials, lot numbers.

Indexes:
- `idx_stock_transactions_org_id_item_id_occurred_at`.
- `idx_stock_transactions_org_id_work_order_id`.

#### 3.2.5 `inventory_reservations`

Represents reserved quantities against future work (e.g., upcoming jobs).

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `inventory_item_id` (UUID, FK → `inventory_items.id`, not null).
- `stock_location_id` (UUID, FK → `stock_locations.id`, not null).
- `work_order_id` (UUID, FK → `work_orders.id`, nullable).
- `quote_id` (UUID, FK → `quotes.id`, nullable).
- `reserved_quantity` (numeric, not null).
- `status` (enum: `active`, `consumed`, `released`, `expired`).
- `created_by_user_id` (UUID, FK → `auth.users`, nullable).
- `created_at` (timestamptz, default now()).
- `released_at` (timestamptz, nullable).
- `metadata` (jsonb, nullable).

Unique:
- (`org_id`, `inventory_item_id`, `stock_location_id`, `work_order_id`, `quote_id`).

### 3.3 Procurement & Reorder Tables

#### 3.3.1 `suppliers`

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `name` (text, not null).
- `contact_name` (text, nullable).
- `email` (text, nullable).
- `phone` (text, nullable).
- `address_id` (UUID, FK → `addresses.id`, nullable).
- `payment_terms` (text, nullable).
- `is_active` (boolean, default true).
- `metadata` (jsonb, nullable).

#### 3.3.2 `inventory_item_suppliers`

Mapping of items to suppliers with pricing and lead times.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `inventory_item_id` (UUID, FK → `inventory_items.id`, not null).
- `supplier_id` (UUID, FK → `suppliers.id`, not null).
- `supplier_sku` (text, nullable).
- `purchase_unit_cost` (numeric, not null).
- `lead_time_days` (integer, nullable).
- `is_primary` (boolean, default false).

Unique:
- (`org_id`, `inventory_item_id`, `supplier_id`).

#### 3.3.3 `purchase_orders`

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `supplier_id` (UUID, FK → `suppliers.id`, not null).
- `ship_to_location_id` (UUID, FK → `stock_locations.id`, not null).
- `po_number` (text, not null).
- `status` (enum: `draft`, `sent`, `partially_received`, `received`, `canceled`, `closed`).
- `order_date` (timestamptz, not null, default now()).
- `expected_date` (timestamptz, nullable).
- `subtotal` (numeric, not null, default 0).
- `tax_total` (numeric, not null, default 0).
- `grand_total` (numeric, not null, default 0).
- `notes` (text, nullable).
- `created_by_user_id` (UUID, FK → `auth.users`, nullable).

Indexes:
- `idx_purchase_orders_org_id_status`.

#### 3.3.4 `purchase_order_lines`

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `purchase_order_id` (UUID, FK → `purchase_orders.id`, not null, on delete cascade).
- `inventory_item_id` (UUID, FK → `inventory_items.id`, not null).
- `ordered_quantity` (numeric, not null).
- `received_quantity` (numeric, not null, default 0).
- `unit_cost` (numeric, not null).
- `line_total` (numeric, not null, default 0).
- `metadata` (jsonb, nullable) – serial/lot info expectations.

Index:
- `idx_purchase_order_lines_org_id_po_id`.

#### 3.3.5 `reorder_rules`

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `inventory_item_id` (UUID, FK → `inventory_items.id`, not null).
- `stock_location_id` (UUID, FK → `stock_locations.id`, not null).
- `strategy` (enum: `static_min_max`, `static_reorder_point`, `ai_forecast`).
- `min_quantity` (numeric, nullable).
- `max_quantity` (numeric, nullable).
- `reorder_point` (numeric, nullable).
- `default_reorder_quantity` (numeric, nullable).
- `enabled` (boolean, default true).
- `metadata` (jsonb, nullable) – AI configuration (forecast horizon, seasonality hints).

Unique:
- (`org_id`, `inventory_item_id`, `stock_location_id`).

### 3.4 Asset Management Tables

#### 3.4.1 `asset_models`

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `manufacturer` (text, nullable).
- `model_number` (text, nullable).
- `name` (text, not null) – friendly name, e.g., "4-Ton Rooftop Unit".
- `category` (text, nullable) – HVAC/Plumbing/Electrical subcategory.
- `default_lifecycle_years` (integer, nullable).
- `spec_metadata` (jsonb, nullable) – capacity, voltage, tonnage, etc.
- `documentation_url` (text, nullable).
- `storage_bucket` (text, nullable) – for manuals/spec sheets in Supabase Storage.
- `storage_path_prefix` (text, nullable).
- `is_active` (boolean, default true).

Index:
- `idx_asset_models_org_id_category`.

#### 3.4.2 `assets`

Represents an individual piece of equipment.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `asset_model_id` (UUID, FK → `asset_models.id`, nullable).
- `customer_id` (UUID, FK → `customers.id`, nullable) – owner.
- `location_id` (UUID, FK → `customer_locations.id`, nullable) – physical install location.
- `external_ref` (text, nullable) – mapping to third-party system.
- `serial_number` (text, nullable).
- `installed_at` (timestamptz, nullable).
- `removed_at` (timestamptz, nullable).
- `status` (enum: `installed`, `out_of_service`, `needs_replacement`, `decommissioned`, `in_storage`).
- `description` (text, nullable).
- `initial_install_cost` (numeric, nullable).
- `current_estimated_value` (numeric, nullable).
- `metadata` (jsonb, nullable) – custom fields, tags.

Indexes:
- `idx_assets_org_id_customer_id`.
- `idx_assets_org_id_location_id_status`.

#### 3.4.3 `asset_events`

History of events for each asset.

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `asset_id` (UUID, FK → `assets.id`, not null, on delete cascade).
- `event_type` (enum: `installed`, `serviced`, `repaired`, `inspection`, `fault_reported`, `replaced`, `decommissioned`, `moved`).
- `work_order_id` (UUID, FK → `work_orders.id`, nullable).
- `event_time` (timestamptz, not null, default now()).
- `summary` (text, nullable).
- `details` (text, nullable).
- `created_by_user_id` (UUID, FK → `auth.users`, nullable).
- `metadata` (jsonb, nullable) – e.g., meter readings, sensor data snapshot.

Index:
- `idx_asset_events_org_id_asset_id_time`.

#### 3.4.4 `asset_warranties`

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `asset_id` (UUID, FK → `assets.id`, not null, on delete cascade).
- `warranty_type` (enum: `manufacturer`, `labor`, `extended`, `service_contract`).
- `provider_name` (text, nullable).
- `policy_number` (text, nullable).
- `coverage_start` (timestamptz, nullable).
- `coverage_end` (timestamptz, nullable).
- `coverage_details` (text, nullable).
- `metadata` (jsonb, nullable).

#### 3.4.5 `asset_documents`

- `id` (UUID, PK).
- `org_id` (UUID, FK → `orgs.id`, not null).
- `asset_id` (UUID, FK → `assets.id`, not null, on delete cascade).
- `storage_bucket` (text, not null).
- `storage_path` (text, not null).
- `file_name` (text, not null).
- `content_type` (text, nullable).
- `file_size_bytes` (bigint, nullable).
- `document_type` (enum: `photo`, `manual`, `inspection_report`, `other`).
- `is_customer_visible` (boolean, default true).

### 3.5 Relationships to Other Modules

- **CRM (Section 1)**:
  - `assets.customer_id` and `assets.location_id` link installed equipment to customers and their sites.
  - Asset events may create CRM interactions (e.g., "Asset X inspected").
- **Scheduling & Dispatch (Section 2)**:
  - Work orders reference `assets` to indicate which equipment is being serviced.
  - Scheduling logic can consider asset priority/criticality.
- **Work Order Management (Section 3)**:
  - Work orders consume inventory via `stock_transactions` (`issue_to_work_order`).
  - Work orders generate `asset_events` upon completion.
- **Quoting & Invoicing (Section 4)**:
  - Quote/invoice line items reference `catalog_items`, which in turn may link to `inventory_items`.
  - Inventory usage feeds costing and margin analytics for line items.
- **Reporting & Analytics (Section 9 of functional capabilities)**:
  - Inventory turns, stockouts, and asset performance metrics are surfaced via analytical views.

## 4. API & Edge Function Design

APIs are implemented via a mix of Supabase direct table access and HTTP Edge Functions. Complex multi-table workflows and business rules should be encapsulated in Edge Functions to keep clients thin and enforce invariants.

### 4.1 Authentication & Authorization

- All Inventory & Asset APIs require:
  - Authenticated Supabase user OR secure customer portal token (for read-only asset views).
  - `org_id` resolved from user profile/JWT and enforced by RLS.
- Role checks:
  - Only inventory/warehouse roles can create/modify `inventory_items`, `stock_locations`, and `purchase_orders`.
  - Technicians can only:
    - View stock in their assigned locations (e.g., trucks + regional warehouses).
    - Record materials usage on their own work orders (Edge Functions validate assignment).
  - Customers can only view `assets` for their `customer_id`/locations.

### 4.2 Inventory Item & Location APIs

#### 4.2.1 Create/Update Inventory Item

- **Endpoint**: `POST /inventory/items`, `PATCH /inventory/items/:id` (Edge Functions).
- **Behavior**:
  - Validate uniqueness of `sku` within `org_id`.
  - Optionally link to an existing `catalog_item_id` or auto-create a basic `catalog_item` when an item is marked as sellable.
  - Enforce that disabling an item (`is_active = false`) checks for in-flight dependencies (e.g., open POs or reservations).

#### 4.2.2 Create/Update Stock Location

- **Endpoint**: `POST /inventory/locations`, `PATCH /inventory/locations/:id`.
- **Behavior**:
  - Manage hierarchical locations (warehouse → zone → bin) via `parent_location_id`.
  - For truck locations, enforce presence of `assigned_user_id`.

### 4.3 Stock Transactions & Reservations

#### 4.3.1 Adjust Stock (Manual / Cycle Count)

- **Endpoint**: `POST /inventory/transactions/adjust`
- **Input**:
  - `inventory_item_id`, `stock_location_id`, `new_on_hand_quantity` OR delta quantity with `transaction_type`.
- **Behavior**:
  - Calculate difference vs current `stock_levels.on_hand_quantity`.
  - Insert `stock_transactions` record of type `adjustment_positive` or `adjustment_negative`.
  - Update `stock_levels` (`on_hand_quantity`, `available_quantity`).
  - Require elevated permission and optional approval for large adjustments.

#### 4.3.2 Receive Purchase Order

- **Endpoint**: `POST /inventory/transactions/receive_po`
- **Input**:
  - `purchase_order_id`, list of `{ purchase_order_line_id, received_quantity }`.
- **Behavior**:
  - Validate PO status (must be `sent` or `partially_received`).
  - For each line:
    - Increment `purchase_order_lines.received_quantity`.
    - Create `stock_transactions` with type `receipt` to `ship_to_location_id`.
    - Update `stock_levels` for each `inventory_item_id`.
  - Update PO status to `partially_received` or `received`.

#### 4.3.3 Consume Materials on Work Order

- **Endpoint**: `POST /inventory/transactions/consume_for_work_order`
- **Input**:
  - `work_order_id`, list of `{ inventory_item_id, stock_location_id, quantity, unit_cost_override? }`.
- **Behavior**:
  - Validate that caller is:
    - Assigned technician, or
    - Authorized CSR/manager.
  - For each item:
    - Ensure sufficient `available_quantity` at `stock_location_id`.
    - Insert `stock_transactions` of type `issue_to_work_order`, referencing `work_order_id`.
    - Reduce `stock_levels.on_hand_quantity` and `available_quantity`.
    - Optionally auto-create work-order material line entries in the Work Order module.
  - Optionally create or update `inventory_reservations` (consume active reservations).

#### 4.3.4 Transfer Stock Between Locations

- **Endpoint**: `POST /inventory/transactions/transfer`
- **Input**:
  - `inventory_item_id`, `from_location_id`, `to_location_id`, `quantity`.
- **Behavior**:
  - Validate `available_quantity` at `from_location_id`.
  - Insert one `stock_transactions` record with type `transfer` and both `from_location_id` and `to_location_id`.
  - Update `stock_levels` for both locations.

#### 4.3.5 Manage Reservations

- **Endpoint**: `POST /inventory/reservations`
- **Input**:
  - `inventory_item_id`, `stock_location_id`, `work_order_id` or `quote_id`, `reserved_quantity`.
- **Behavior**:
  - Ensure `available_quantity` ≥ requested reservation.
  - Insert `inventory_reservations` and increase `allocated_quantity`, decrease `available_quantity`.

- **Endpoint**: `POST /inventory/reservations/:id/release`
- **Behavior**:
  - Mark reservation as `released` and adjust `stock_levels`.
  - Typically invoked when a work order is canceled or materials are no longer needed.

### 4.4 Procurement & Reorder APIs

#### 4.4.1 Create/Manage Purchase Orders

- **Endpoints**:
  - `POST /inventory/purchase_orders`
  - `PATCH /inventory/purchase_orders/:id`
  - `POST /inventory/purchase_orders/:id/send`
- **Behavior**:
  - Allow creation of POs for one supplier with line items.
  - On `send`, mark PO as `sent`, optionally email PDF to supplier.
  - Enforce edit restrictions once PO is sent (allow only certain field changes).

#### 4.4.2 Reorder Suggestion & Alerts

- **Endpoint**: `POST /inventory/reorder_suggestions/run`
- **Behavior**:
  - For each `inventory_item_id` and `stock_location_id` with enabled `reorder_rules`:
    - Compute projected demand (using historical `stock_transactions` and AI forecast where configured).
    - Compare `stock_levels.available_quantity` vs `reorder_point`/`min_quantity`.
    - Generate suggestion rows (virtual or persisted) such as:
      - `suggested_order_quantity`, `suggested_supplier_id`, `urgency_score`.
  - Return structured list and optionally persist to a `inventory_reorder_suggestions` table for review.

- **Endpoint**: `POST /inventory/reorder_alerts/dispatch`
- **Behavior**:
  - Use suggestions to send alerts (email/in-app) to inventory managers.
  - Optionally auto-create POs for items above a configurable urgency threshold.

### 4.5 Asset Management APIs

#### 4.5.1 Create/Update Asset Models & Assets

- **Endpoints**:
  - `POST /assets/models`, `PATCH /assets/models/:id`
  - `POST /assets`, `PATCH /assets/:id`
- **Behavior**:
  - Allow admins to define standardized `asset_models`.
  - Support creation of assets:
    - From work orders (e.g., new install).
    - From bulk imports (CSV uploads via Storage + processing function).
  - Enforce consistent linking to `customer_id`/`location_id`.

#### 4.5.2 Log Asset Events

- **Endpoint**: `POST /assets/:id/events`
- **Behavior**:
  - Create `asset_events` tied to work orders or manual inspections.
  - For certain event types (e.g., `installed`, `decommissioned`), update `assets.status` and timestamps.

#### 4.5.3 Warranty & Documents

- **Endpoints**:
  - `POST /assets/:id/warranties`
  - `POST /assets/:id/documents`
- **Behavior**:
  - Store warranty details and upload associated docs to Storage with `org_id` + `asset_id`-scoped paths.
  - Expose read-only views to customers via portal, respecting `is_customer_visible`.

### 4.6 Background Jobs & Maintenance

- Cron-style Edge Functions:
  - **Reorder evaluation**: run periodically to refresh reorder suggestions and triggers.
  - **Reservation cleanup**: mark stale reservations as `expired` and release stock.
  - **Asset lifecycle monitor**: identify assets nearing end-of-life or warranty expiry and create notifications or proactive quote opportunities.

## 5. AI & Analytics Design

### 5.1 AI-Based Demand Forecasting

- **Edge Function**: `inventory_forecast_demand`
  - **Inputs**:
    - Historical `stock_transactions` for consumption (issues to work orders/quotes) by item/location.
    - Seasonality indicators (month, climate region, service type).
    - Configuration from `reorder_rules.metadata` (forecast horizon, confidence).
  - **Process**:
    - Aggregate historical usage into time series (e.g., weekly/monthly consumption).
    - Call an external AI/ML/LLM service or internal model to forecast demand for the next N periods.
    - Consider planned work orders (backlog) as additional demand.
  - **Outputs**:
    - Forecasted demand per item/location and suggested reorder quantities.
    - Persisted in a `inventory_demand_forecasts` table or `stock_levels.metadata`.

### 5.2 AI for Asset Health & Replacement Recommendations

- **Edge Function**: `asset_health_assessment`
  - **Inputs**:
    - Asset model/spec data, age, service history (`asset_events`), known faults.
    - Environmental context (location climate, usage intensity if available).
  - **Outputs**:
    - Health score (0–100), recommended actions (monitor, repair, replace), and expected remaining useful life.
    - Stored in `assets.metadata` and/or a dedicated `asset_health_snapshots` table.

- **Edge Function**: `asset_replacement_opportunities`
  - Scans assets approaching end-of-life or high failure risk and generates structured opportunities for outbound quoting (integrating with CRM and Quoting modules).

### 5.3 Reporting & KPIs

- Key KPIs and metrics:
  - Inventory turns by item/category.
  - Stockout incidents and fill rate.
  - Aged stock and slow-moving items.
  - Asset failure rates by model, age, and environment.
- Implementation:
  - SQL views/materialized views such as:
    - `vw_inventory_kpis`, `vw_asset_performance`.
  - These views are consumed by the Reporting & Analytics module and dashboards.

## 6. Frontend (Next.js / Mobile) UI/UX Design

### 6.1 Back-Office Inventory UI

- **Inventory Dashboard**:
  - Summary cards: total stock value, stockouts, items below reorder point.
  - Top slow-moving and fast-moving items.
- **Items List & Detail**:
  - Filters by category, supplier, active/inactive.
  - Detail view with:
    - Per-location stock levels (on hand, allocated, available).
    - Recent transactions and reorder rules.
    - Links to related POs and suppliers.
- **Locations Management**:
  - Tree/hierarchical view of locations and bins.
  - Per-location stock summary and configuration.
- **Purchase Orders**:
  - List with filters by status and supplier.
  - Detail page with line items, receive actions, and PDF export.

### 6.2 Technician UI

- **Truck Stock View**:
  - Quick list of items with quantities available on the technician’s truck.
  - Basic search and scanning (future: barcode/QR).
- **Material Usage on Work Order**:
  - Simple interface to add items used from selected locations.
  - Automatic stock update via `consume_for_work_order` Edge Function.
- **Asset Lookup & Update**:
  - Search by address, customer, or asset serial.
  - View asset details, history, and warranties.
  - Add notes, photos, and log events (e.g., inspection results).

### 6.3 Customer Portal UI

- **Installed Assets**:
  - List of assets per customer/location with status and basic specs.
  - Access to documents (manuals, inspection reports, photos) where `is_customer_visible`.
- **Proactive Recommendations**:
  - Notifications when assets are nearing warranty expiration or recommended replacement.
  - Deep links into quotes generated for replacement work.

### 6.4 Data Access Patterns

- Use Supabase JS client for:
  - Simple reads (inventory lists, stock levels, asset lists).
  - Low-risk updates (notes, metadata) when RLS rules are straightforward.
- Use Edge Functions for:
  - Multi-table workflows (stock transactions, PO lifecycle, AI forecasting).
  - Operations requiring strict role checks and business rules (consumption, adjustments, transfers).

## 7. Security, Privacy, and Compliance

### 7.1 RLS & Permissions

- All Inventory & Asset tables enforce:
  - `org_id = current_setting('app.current_org_id')::uuid` (or Supabase equivalent).
- Role-based policies:
  - Restrict write access to stock/PO/asset tables to appropriate internal roles.
  - Restrict technicians to stock and assets relevant to their assignments.
  - Restrict customer portal access to assets where `assets.customer_id` matches the authenticated customer.

### 7.2 Data Integrity & Auditing

- `stock_transactions` is append-only: no hard deletes; corrections done via counter-transactions.
- Critical operations (PO receipt, large adjustments, asset decommissioning) should:
  - Log the acting user.
  - Optionally require dual control (approval) in future iterations.
- Use Supabase logs for additional auditing of Edge Function calls and failures.

### 7.3 Storage & Document Access

- Store asset and inventory documents in dedicated Storage buckets with:
  - Path structure including `org_id` and entity IDs (e.g., `orgs/{org_id}/assets/{asset_id}/...`).
  - Signed URLs with short TTLs for customer-facing access.
  - Clear separation between internal-only and customer-visible docs.

## 8. Non-Functional Requirements

- **Performance**:
  - Inventory list and stock-level queries should return in < 500–800 ms for typical orgs (~50k stock records).
  - Stock transaction creation should be < 200–300 ms under normal load.
- **Scalability**:
  - Index on key fields: `inventory_item_id`, `stock_location_id`, `status`, `org_id`.
  - Materialized views for heavy analytics if needed.
- **Reliability**:
  - Idempotent transaction endpoints (e.g., safe re-submission of receipts).
  - Background jobs should be resilient to transient failures with retries/backoff.
- **Observability**:
  - Edge Functions must log correlation IDs, org and user context, and key payload fields (excluding sensitive data).
  - Monitoring for stockout events, failed PO receipts, and AI forecast errors.

## 9. Implementation Notes & Open Questions

- **Open Questions**:
  - Which costing method should be used initially (simple moving average vs. more advanced options)?
  - To what extent should POs and vendor bills integrate with external accounting (QuickBooks/Xero) vs. remain light-weight?
  - Required depth of IoT integration in v1 (simple asset bindings vs. continuous telemetry ingestion).
  - How strict should reservation behavior be (hard allocations vs. soft suggestions)?
  - Should technicians be allowed to create ad-hoc items on work orders that auto-create inventory/catalog records?
- **Extension Points**:
  - Add `serial_numbers` and `lots` tables for deeper traceability.
  - Introduce cycle counting programs with scheduling and compliance reporting.
  - Deeper integration with Quoting & Invoicing for true cost-of-goods tracking and margin analytics per job.
  - Additional AI capabilities such as vendor lead-time prediction and dynamic safety stock adjustments.

This TDD is structured to support direct translation into implementation tasks (database migrations, Edge Functions, RLS policies, UI flows, and integration work) for a robust, AI-aware Inventory & Asset Management module that fits cleanly into the broader HVAC/Plumbing/Electrical services platform.


