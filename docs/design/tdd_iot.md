## TDD – Predictive Maintenance & IoT Orchestrator (MicroSaaS Layer)

### 1. Scope & Objectives

**Goal**: Design a pluggable **IoT + predictive maintenance orchestrator** that:

- Connects low-cost field devices and OEM gateways to the existing FSM platform.
- Normalizes and stores telemetry, faults, and events in a shared time-series/IoT layer.
- Runs **rules + ML/AI models** to detect anomalies and predict failures.
- Orchestrates actions into the core platform: **work orders, quotes, notifications, portal updates**.
- Is deliverable as a **MicroSaaS add-on**, priced primarily **per connected asset/device**.

This TDD focuses on:

- Capabilities, user journeys, and data flows.
- Technical architecture (cloud + onsite edge/IoT infra).
- Device classes, connectivity options, and **estimated per-device cost envelope**.

---

### 2. Context & Dependencies

- **Existing platform (from `tech_features.md`)**
  - **Core**: Supabase (PostgreSQL, Auth, RLS, Storage, Realtime, Edge Functions) + Next.js on Vercel.
  - **Modules**: CRM, Scheduling & Dispatch, Work Orders, Inventory & Assets, Customer Portal, Communication & Collaboration, AI/LLM infra.
- **Positioning**
  - This IoT orchestrator is a **MicroSaaS layer**; it can be:
    - A **separate Supabase project** (or time-series-oriented backend) with its own DB.
    - Exposed to the main FSM via REST/gRPC APIs and event webhooks.
  - Commercial model: **per-connected-asset/device pricing** plus optional premium analytics tier.

---

### 3. Core Capabilities

#### 3.1 Device & Gateway Onboarding

**User stories**

- As a technician, I can **scan a QR code** on a sensor or gateway and bind it to an `asset` and `location`.
- As an admin, I can configure **OEM cloud integrations** (e.g., smart thermostats, chillers, BMS) via API keys/tenant IDs.
- As a partner, I can run **bulk onboarding** for fleets of similar devices with minimal manual work.

**Functional requirements**

- **Device registry**
  - Tables: `iot_devices`, `iot_gateways`, `iot_device_bindings`.
  - Fields:
    - Device identifier (hardware ID, MAC, serial, or OEM-provided ID).
    - Device type (sensor, gateway, thermostat, controller, OEM cloud virtual device).
    - Protocol and connectivity (MQTT, HTTP, LoRaWAN, Modbus/TCP, BACnet/IP via gateway).
    - Binding to `assets.id` and `locations.id` in the core FSM DB.
- **Claim / pairing workflow**
  - QR code or alphanumeric claim code printed on device/gateway.
  - Web/mobile flow:
    - Scan → look up device in `iot_devices` (unclaimed state).
    - Choose or create asset and site.
    - Set metadata: measurement types, units, update interval.
  - Role-based permissions: only authorized roles can bind or rebind devices.
- **OEM cloud connectors**
  - Pluggable connectors for major OEMs:
    - Persist credentials and tenant metadata securely.
    - Synchronize OEM device inventory → map to `iot_devices`.
    - Subscribe to OEM webhook topics or poll APIs for telemetry and fault codes.

#### 3.2 Telemetry Ingestion & Normalization

**User stories**

- As the orchestrator, I can **ingest high-volume telemetry** (temperatures, currents, pressures, runtimes) from diverse protocols reliably.
- As a data consumer, I can query telemetry in a **normalized schema** irrespective of source.

**Functional requirements**

- **Ingestion endpoints**
  - MQTT broker for device and gateway clients (TLS, token-based auth).
  - HTTPS ingest endpoint for OEM clouds and constrained devices.
  - Optional WebSocket channel for lab/test devices.
- **Time-series storage**
  - Options:
    - **TimescaleDB** extension on Postgres (if staying in Supabase ecosystem).
    - Or dedicated time-series DB (InfluxDB, AWS Timestream, etc.) with summarized data synced back to Supabase.
  - Base schema:
    - `iot_timeseries (id, device_id, ts, metric_key, value_num, value_str, unit, quality_flag, ingest_source)`.
    - Partitioned by time (e.g., monthly) and optionally by tenant.
- **Normalization layer**
  - Map raw payloads into canonical metric keys:
    - E.g., `coil_temp_c`, `suction_pressure_psi`, `line_current_a`, `vibration_mm_s`, `runtime_hours`.
  - Maintain mapping table: `iot_metric_mappings` (vendor_field → canonical metric key, unit, scaling factor).
- **Data compaction & retention**
  - Short-interval raw data (e.g., 1–5 minutes) retained for 30–90 days (configurable).
  - Aggregated rollups (e.g., 15-min, hourly, daily) computed via jobs and retained long-term.

#### 3.3 Event, Fault & Anomaly Detection Engine

**User stories**

- As a planner, I want the system to **detect anomalies and fault patterns** and turn them into actionable events.
- As a service manager, I want **configurable rules** per device type and customer SLA tier.

**Functional requirements**

- **Rule engine (deterministic)**
  - Rule definitions stored in `iot_rules`:
    - Scope: device type, asset model, customer, location, or global.
    - Conditions: threshold, duration, rate of change, combination rules (e.g., temp + current).
    - Output: event type (`warning`, `maintenance_due`, `likely_failure`, `critical_fault`), severity, recommended action template.
  - Execution:
    - Stream processing on ingestion (e.g., via serverless functions or lightweight stream processor).
    - Or periodic batch evaluation for more complex patterns.
- **ML/AI-based anomaly & prediction models**
  - Phase 1:
    - Univariate anomaly detection (e.g., Z-score, EWMA thresholds) for individual metrics.
    - Simple survival/regression models per asset family to estimate failure risk window (7–30 days).
  - Phase 2:
    - Multivariate models using historical labeled data:
      - Input: sequences of telemetry + asset metadata + past failures.
      - Output: probability of failure in a given horizon, recommended intervention (inspect/repair/replace).
  - Model hosting:
    - Inference via serverless functions or containerized microservices (e.g., small CPU instances, no GPU needed).
    - Models versioned and referenced in `iot_models` with lineage.
- **Event stream**
  - Table: `iot_events (id, asset_id, device_id, rule_id, model_version, ts, severity, event_type, payload_json, status)`.
  - States: `open`, `acknowledged`, `linked_to_work_order`, `dismissed`.

#### 3.4 Orchestration into FSM: Work Orders, Quotes, Tasks

**User stories**

- As a dispatcher, I want high-confidence, high-severity events to **auto-create proactive maintenance jobs**.
- As a sales/estimating team, I want severe degradation events to **auto-create replacement opportunity quotes**.

**Functional requirements**

- **Event-to-action mapping**
  - Configurable mapping per tenant:
    - Event type + severity → action:
      - Create work order with template (job type, checklist, priority).
      - Create draft quote with recommended line items.
      - Create CRM follow-up task (e.g., call customer to discuss options).
      - Notify specific teams/channels (Slack, email, in-app).
  - Respect working hours and SLAs when choosing scheduling windows.
- **Work order creation**
  - Uses core FSM APIs:
    - `work_orders`:
      - Links to `asset_id`, `location_id`, `customer_id`.
      - Includes IoT event reference and context.
    - `work_order_checklists`:
      - Attach IoT-specific diagnostic checklists by asset model/issue type.
  - Auto-assign **priority** based on risk and SLA tier.
- **Quote creation**
  - Draft quotes:
    - For high-risk events or repeatedly failing assets.
    - Pull parts and labor templates from price books.
    - Include optional “repair vs replace” options with suggested narratives.
- **Idempotency & spam prevention**
  - Suppress duplicate events:
    - Event grouping window (e.g., no more than one proactive job per asset per X hours for same event family).
    - Event state machine ensures one work order/quote per unique problem until resolved.

#### 3.5 Customer & Technician Communication

**User stories**

- As a customer, I want **clear, non-technical alerts** about detected issues and my options.
- As a technician, I want **IoT-derived context** in my work order briefing.

**Functional requirements**

- **Customer notifications**
  - Use Communication & Collaboration module for messaging:
    - Channels: email, SMS, in-app/portal.
    - Templates with dynamic content:
      - Summaries of detected behavior (e.g., “compressor short cycling 3× normal rate”).
      - Simple explanation of risk (e.g., “Higher chance of no-cool event in next 7–14 days”).
      - Proposed next steps (inspection vs replacement proposal).
  - Localization and branding via tenant-specific templates.
- **Technician views**
  - Work order briefing includes:
    - Sparkline charts of key metrics in the days/weeks leading up to the event.
    - List of anomalies and relevant rule/model hits.
    - Suggested diagnostic flow and parts likely needed (integrates with inventory).
- **Customer portal integration**
  - Portal card per connected asset:
    - Current status: `Healthy`, `Degrading`, `At Risk`, `Critical`.
    - Recent IoT events and actions taken.
    - Optional “request second opinion” or “accept recommendation” actions.

#### 3.6 Monitoring, Admin & Tenancy

**Functional requirements**

- **Fleet monitoring**
  - Dashboards for internal teams:
    - Device online/offline status.
    - Data freshness and telemetry volumes.
    - Rule/alert rates per tenant and per asset type.
- **Tenant configuration**
  - Per-tenant settings:
    - What event types can auto-create jobs.
    - Notification preferences and escalation rules.
    - Data retention preferences (within global limits).
- **Billing & usage**
  - Metrics per tenant:
    - Number of active connected assets/devices.
    - Telemetry volume.
    - Number of IoT-generated work orders/quotes.
  - Exposed to billing engine for per-device pricing.

---

### 4. Technical Architecture

#### 4.1 High-Level Logical Architecture

- **Device Layer (Onsite / OEM Cloud)**
  - Low-power sensors (temperature, pressure, current, vibration, humidity, flow, occupancy).
  - Gateways (onsite edge devices aggregating local fieldbus/protocols).
  - OEM cloud platforms (thermostats, BMS, packaged equipment).
- **IoT Orchestrator Backend**
  - Ingestion endpoints (MQTT, HTTPS).
  - Time-series database and metric normalization.
  - Rules engine + ML inference services.
  - Event store and orchestration logic to FSM.
- **Core FSM Platform**
  - Supabase/Postgres (assets, work orders, schedules, CRM, etc.).
  - API layer, Next.js frontend, mobile apps.
  - Communication & AI layers consuming IoT events and context.

#### 4.2 Component Breakdown

- **MQTT Broker**
  - Options: managed MQTT (e.g., AWS IoT Core, EMQX Cloud, HiveMQ Cloud) or self-hosted.
  - Provides secure topics per tenant and device.
- **Ingestion Services**
  - Serverless functions triggered by MQTT messages, HTTP requests, and OEM webhooks.
  - Parse payloads, authenticate device, enrich with device metadata, write to time-series store.
- **Time-Series Storage**
  - Preferred path:
    - Supabase + TimescaleDB extension for unified management.
    - If scale/throughput becomes a concern, offload to a dedicated InfluxDB/Timestream cluster.
- **Rules & ML Services**
  - Stateless microservices:
    - Consume telemetry streams or scheduled batches.
    - Emit `iot_events` and orchestrator commands.
- **Orchestrator**
  - Central service that:
    - Consumes `iot_events`.
    - Applies tenant-specific orchestration policies.
    - Calls core FSM APIs to create work orders, quotes, tasks, and messages.

#### 4.3 Data Flow (Example)

1. Sensor captures data (e.g., discharge air temp, suction pressure, compressor current) every 60 seconds.
2. Sensor sends data to onsite gateway via Modbus/LoRa/Bluetooth.
3. Gateway publishes normalized JSON to MQTT topic over LTE or Ethernet.
4. Ingestion function validates token, looks up `iot_devices` entry, and writes records to time-series store.
5. Rules/ML services evaluate stream; an anomaly is detected (short cycling + increased current).
6. Service writes `iot_events` row (`likely_failure`, `medium` severity) with context.
7. Orchestrator checks tenant policies: allowed to create proactive work order and notify customer.
8. Orchestrator calls FSM APIs:
   - Creates `work_order` linked to asset with IoT context.
   - Triggers communication templates via communication module.

---

### 5. Onsite Infrastructure & Implementation Options

#### 5.1 Device Classes

- **Retrofit sensors**
  - Clamp-on current sensors (CTs) for compressor/fan motors.
  - Temperature probes (supply/return air, refrigerant lines, ambient).
  - Vibration/accelerometers for rotating equipment (pumps, motors).
  - Differential pressure sensors (filters, coils).
  - Digital inputs (e.g., door open/closed, occupancy).
- **Smart controllers/thermostats**
  - Wi-Fi or Zigbee thermostats with cloud APIs.
  - Packaged unit controllers with IP connectivity.
- **Industrial I/O modules**
  - DIN-rail modules for analog (4–20 mA, 0–10V) and digital signals.

#### 5.2 Gateway Options (Edge Hardware)

**Option A – Low-Cost SBC Gateway (for light commercial/SMB)**

- Hardware examples:
  - Raspberry Pi-class SBC (or equivalent industrial SBC).
  - Specs:
    - Quad-core ARM, 2–4 GB RAM.
    - Onboard Wi-Fi/Ethernet; optional LTE modem via USB/hat.
    - 16–32 GB microSD or eMMC.
- Software:
  - Containerized stack (e.g., lightweight Linux + Docker/Podman).
  - Components:
    - Local MQTT or protocol adapters (Modbus RTU/TCP, BACnet/IP, BLE).
    - Agent that batches and forwards normalized payloads to cloud MQTT.
    - Basic local buffering when offline.
- Approx hardware cost:
  - SBC + case + power supply: **\$60–\$100**.
  - Optional LTE modem and SIM provisioning: **\$40–\$80** one-time + ongoing data plan.

**Option B – Industrial Rugged Gateway (for harsh environments/enterprises)**

- Specs:
  - Fanless, industrial temperature range.
  - Multiple serial ports (RS-485), digital I/O, dual Ethernet.
  - DIN-rail mount.
- Costs:
  - Device: typically **\$250–\$600** per gateway depending on ports and certifications.
  - Better suited for multi-asset sites with long lifetimes and harsh conditions.

**Option C – No Onsite Gateway (OEM Cloud Integrations Only)**

- For sites with existing smart equipment that already streams to OEM clouds:
  - No hardware cost.
  - Implementation cost shifts to **API integration and security**.
  - Limited physical telemetry (restricted to vendor’s provided data).

#### 5.3 Connectivity Options

- **Ethernet/Wi-Fi**
  - Cheapest ongoing cost.
  - Best when customer network cooperation is high and VLAN/security rules can be met.
- **Cellular (LTE/5G)**
  - Higher ongoing cost but isolates from customer IT.
  - Recommended for quick deployments where network integration is a barrier.
- **LoRaWAN**
  - Suitable for large campuses or multi-building sites where long-range, low-bandwidth sensing is required.
  - Requires local LoRaWAN gateway(s) and LoRa sensors.

---

### 6. Estimated Cost Per Device (IoT Capability)

The following are **order-of-magnitude estimates** suitable for pricing and business casing. Actual costs will vary by region, volume, and vendor.

#### 6.1 Baseline Retrofit Scenario (HVAC Unit or Similar Asset)

Assume:

- 1 gateway serves **4–8 assets** (e.g., rooftop units).
- Each asset uses **2–4 sensors** (current, temperature, vibration/pressure).

**Hardware BOM per asset (amortized)**

- Sensors:
  - 2–4 retrofit sensors @ \$10–\$30 each → **\$20–\$120** per asset.
- Gateway share:
  - Low-cost SBC gateway at \$80; shared across 4–8 assets:
    - **\$10–\$20** per asset.
- Miscellaneous:
  - Wiring, conduit, mounting hardware: **\$5–\$15** per asset.

**Approximate one-time hardware cost per asset**

- **Low end (minimal sensing)**: \$35–\$50 per asset.
- **Typical (3–4 high-quality sensors)**: **\$60–\$120 per asset**.
- **High end (ruggedized + extra channels)**: \$120–\$180 per asset.

#### 6.2 Connectivity & Cloud Operating Cost (Per Device/Asset)

- **Connectivity (if cellular)**
  - Shared data plan for gateway: \$5–\$15/month.
  - Amortized per asset (4–8 assets per gateway): **\$1–\$4/month per asset**.
- **Cloud IoT & storage**
  - Assuming:
    - 1 reading/minute, ~10 metrics per reading → ~14,400 data points/day/asset.
    - Rollups and retention (90 days raw, 1–2 years aggregated).
  - At modest cloud prices for storage, compute, and IoT broker:
    - Roughly **\$0.20–\$0.80/month per asset** for telemetry storage + processing at small scale.
  - Plus orchestrator overhead (rules, events, work order creation):
    - **\$0.20–\$0.50/month per asset**.

**Approximate recurring cost envelope per asset**

- **Without cellular (using site network)**:
  - Cloud + orchestrator: **\$0.40–\$1.30/month per asset**.
- **With cellular**:
  - Cloud + orchestrator + connectivity: **\$1.40–\$5.30/month per asset**.

These numbers provide room for:

- A **retail price** of e.g., **\$5–\$15/month per connected asset**, leaving healthy margin for support, development, and risk.

#### 6.3 OEM-Only Integration Scenario (No Hardware)

- No onsite hardware cost per asset.
- Cloud and orchestrator costs similar to above telemetry scenario (depends on volume).
- Net recurring marginal cost could be:
  - **\$0.20–\$1.00/month per active asset** (mainly processing and data storage).
- Leaves room to price OEM-connected assets at a slight discount vs. retrofit hardware assets.

---

### 7. Non-Functional Requirements

#### 7.1 Scalability

- Design for:
  - 10s of devices per site, 1000s–100,000s of devices across tenants.
  - Burst ingestion and backfill scenarios (devices reconnecting after outages).
- Techniques:
  - Horizontal scaling of ingestion services.
  - Time-series DB partitioning and retention policies.
  - Async processing for heavy analytics.

#### 7.2 Reliability & Offline Behavior

- Gateways must:
  - Buffer data locally for **at least several hours** when offline.
  - Resume uploads with correct ordering when connectivity returns.
- Cloud services:
  - Use at-least-once semantics with idempotent writes to avoid duplication.

#### 7.3 Security & Multi-Tenancy

- Per-device credentials/keys; rotateable and revocable.
- TLS for all device-to-cloud and cloud-to-cloud comms.
- Strong tenant isolation:
  - Per-tenant topics in MQTT.
  - RLS policies in Supabase for all IoT tables.
- PII minimization:
  - Device and telemetry data contain **no customer-identifying information** except via secure references to `assets` and `locations` in the FSM.

#### 7.4 Observability

- Metrics:
  - Device online/offline counts.
  - Telemetry ingestion rates and error rates.
  - Rule and event generation rates.
  - IoT-driven work order and quote volumes.
- Logging:
  - Structured logs for ingestion, rule evaluation, and orchestration steps.

---

### 8. Rollout & Integration Plan (High-Level)

- **Phase 1: Narrow pilot**
  - Focus on 1–2 asset types (e.g., rooftop HVAC units, critical pumps).
  - Use low-cost SBC gateway option and 2–3 key sensors per asset.
  - Implement simple deterministic rules; limited ML.
- **Phase 2: Broader device coverage + ML**
  - Add more sensor types and asset families.
  - Train and deploy first supervised failure-risk models.
- **Phase 3: Full SaaS packaging**
  - Harden onboarding flows, billing integration, and multi-tenant management.
  - Offer per-asset subscription tiers (monitoring only vs. monitoring + predictive + automated orchestration).

This design supports a practical **IoT MicroSaaS** that can start small, leverage existing FSM modules, and scale into a differentiated predictive maintenance offering with clear per-device cost structure. 



