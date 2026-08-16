# Warehouse AI Operational Supervisor — Research & Strategy Document

**Source Thread:** Research channel (#a0c6b1d4-de34-48f6-9b89-830010940a56)  
**Date:** August 2026  
**Contributors:** Amaan Singh, Nvidia Server, Flash  

---

## Executive Summary

Building an AI operational supervision layer for warehouses and manufacturing that sits **above** existing systems (WMS/WES/MES, robot fleets, equipment, CMMS, worker devices) rather than replacing them. The core value: **prevent downtime when possible, minimize recovery time when it happens**.

---

## Industry Landscape & Target Market

### Current State: Fragmented Systems
| System Type | Examples | Responsibility |
|-------------|----------|----------------|
| WMS/MES | ShipHero, Manhattan, Blue Yonder, Körber | Orders, inventory, production, workflows |
| WES/WCS | AutoStore, Dematic, Swisslog | Execution coordination, automation control |
| Robot Fleet Managers | Locus, 6 River Systems, Fetch, Geek+ | Single-vendor robot orchestration |
| CMMS | Fiix, UpKeep, Hippo | Maintenance work orders, asset tracking |
| Worker Devices | Zebra scanners, tablets, wearables, UWB | Human task execution |

**Key Insight:** Individual systems are getting smarter, but **cross-vendor coordination** is the gap. No single system sees the full picture: robots + inventory + labor + equipment + maintenance.

### Robotics Adoption Maturity
| Maturity | Technologies |
|----------|--------------|
| **Mature** | AMRs/AGVs, conveyors, sortation, goods-to-person |
| **Early** | Robotic piece picking, humanoid robots |

**Strategic Implication:** Build for **existing automation** — integrate with current robot fleets and equipment, don't wait for humanoids.

### Target Customer Profile
- **Primary:** Mid-market 3PLs (2,000+ facilities) running multiple clients on shared floors
- **Secondary:** Enterprise distribution centers with legacy WMS + mixed automation
- **Phase 3+:** Facilities with conveyor/sortation wanting equipment integration

---

## Core Problem: Manual Cross-System Coordination

Supervisors manually:
1. Notice an issue (robot stuck, inventory missing, conveyor down)
2. Determine downstream impact across systems
3. Decide response (reroute, reprioritize, dispatch human, redirect material)
4. Coordinate execution across disconnected tools

**Expensive failures:** AMR congestion, inventory mismatches, conveyor failures, robot errors, equipment breakdowns, labor shortages, dock congestion, slow exception handling.

---

## Product Strategy: Phased Approach

### Phase 1 — AI Exception Orchestration (Software-Only, Zero Hardware)
**Timeline:** ~14 weeks full / 8 weeks MVP  
**Deploy:** 2 weeks (API keys only)  
**Write Access:** Level 1 only — human notifications (Slack/Teams/SMS)

| Tier | Component | Vendor Target | Effort |
|------|-----------|---------------|--------|
| **Tier 0** | WMS Adapter | ShipHero (first) | 2 weeks |
| **Tier 1a** | Robot Fleet Adapter | Locus (first) | 2 weeks |
| **Tier 1b** | Labor Adapter | CSV/API | 1 week |
| **Tier 1c** | CMMS Adapter | Fiix/UpKeep (optional) | 1 week |
| **Tier 1d** | Camera Events | Verkada/Rhombus (opportunistic) | 0.5 weeks |
| **Core** | Canonical Data Model + Exception Engine | — | 2 weeks |
| **Core** | LLM Reasoning Layer | — | 3 weeks |
| **UI** | Dashboard (Exception Cards) | — | 2 weeks |
| **UI** | Notifications + Deep Links | Slack/Teams/SMS | 1 week |
| **UI** | Ground Truth Capture | Mobile Web | 1 week |
| **Core** | Drift Detection | — | 1 week |
| **Tool** | Map Importer (Logical Twin) | — | 1 week |
| **Core** | Trust Ladder Framework | — | 1 week |

### Phase 2 — Controlled Write-Back (Trust Ladder L2-L3)
- **L2 (Phase 2a):** CMMS ticket creation (low risk)
- **L3 (Phase 2b):** Robot reroute/pause (medium risk)
- Prerequisite: 8+ weeks L1 accuracy data

### Phase 3 — WMS Write-Back (Trust Ladder L4)
- Inventory transfers, wave short-ship (high risk)
- Prerequisite: 12+ weeks L1-L3 data + customer sign-off

### Phase 4 — Edge Gateway (Hardware)
- Conveyor/sorter sensors, PLC/OPC UA/Modbus
- Separate product, separate budget, Series B+

---

## The Trust Ladder (De-Risking Write Access)

| Level | Phase | Target | Action | Risk | Gate |
|-------|-------|--------|--------|------|------|
| **L1** | 1 | Humans (Slack/Teams/SMS) | Notify, Dispatch, Request Ground Truth | **Zero** | None — Day 1 |
| **L2** | 2a | CMMS | Create maintenance work order | Low | 4+ weeks L1 accuracy |
| **L3** | 2b | Robot Fleet | Reroute robot, pause mission | Medium | 8+ weeks L1+L2 accuracy |
| **L4** | 3 | WMS | Inventory transfer, wave short-ship | High | 12+ weeks L1-L3 + sign-off |

**Sales Pitch:** "We start at Level 1 — zero write access. After 8 weeks of audit-proven accuracy, YOU decide if we unlock Level 2."

---

## Canonical Data Model (The Implementation Moat)

```typescript
interface FacilityState {
  // From WMS
  waves: Wave[];              // {id, client, cutOff, status, priority, tasks[]}
  inventory: InventoryRecord[]; // {location, sku, onHand, reserved, variance, lastCount}
  locations: Location[];      // {id, type, zone, neighbors[], equipment[]}
  
  // From Robot Fleet(s)
  robots: Robot[];            // {id, vendor, pose, status, mission, battery, zone}
  
  // From Labor
  associates: Associate[];    // {id, name, certifications[], zone, activeTask, shift}
  
  // From CMMS (optional)
  maintenance: WorkOrder[];   // {id, asset, status, assignee, priority, location}
  
  // From Human-as-Sensor
  groundTruth: GroundTruthReport[]; // {timestamp, zone, reporter, finding, confidence}
  
  // Spatial Reasoning (Logical Twin)
  zoneGraph: ZoneGraph;       // {nodes: Zone[], edges: Adjacency[], constraints: Traversability[]}
  
  // Derived / Computed
  exceptions: Exception[];    // AI-detected anomalies with reasoning
  conflicts: Conflict[];      // Multi-vendor/resource conflicts
  driftFlags: DriftFlag[];    // WMS vs reality divergence
}
```

**Every integration = adapter to this model.** New WMS? New adapter. The AI only speaks this language.

---

## Logical Twin: Spatial Reasoning Layer

**Zone-Relationship Graph** — not a 3D digital twin, a topological map:
- `Aisle 4` → adjacent to `Zone B` → feeds `Sorter 1`
- Traversability rules: forklifts can't use robot charging zone, AGVs need 2m clearance
- Choke points, alternate routes, priority paths per vehicle type

**Enables:** "If Aisle 4 blocked, redirect to Aisle 3 — closest logical path for this vehicle type"

**Data Sources:** CSV/Excel from WMS, robot vendor map exports, AutoCAD/DWG (Phase 2)

**Map Importer:** 15-minute CSV upload → column mapping → Logical Twin generated

---

## LLM Reasoning Layer (The "Secret Sauce")

**Old Approach:** Correlate data → apply deterministic rules  
**New Approach:** Ingest heterogeneous logs (WMS events, robot telemetry, operator notes, CMMS tickets) → LLM reasoning → explanations, recommendations, confidence scores in natural language

### What This Enables
| Capability | Example |
|------------|---------|
| Snowflake WMS handling | No custom code per site — LLM reasons over canonical model |
| Plain-English WHY | "Robot R-12 blocked + wave cut-off 12 min + forklift 50m away = dispatch forklift (72% confidence)" |
| Learning from feedback | Operator taps "Conveyor Jammed" → facility model updates: Aisle 4 conveyor jam probability +15% |
| Cross-vendor correlation | Locus robot error + ShipHero wave at risk + labor shortage in zone = unified exception |

### Reasoning Output
```json
{
  "exception": "AMR Congestion — Aisle 4",
  "evidence": ["5 Locus robots blocked", "Wave W-14 (Shopify) cut-off 12 min", "Forklift FL-3 available 50m"],
  "reasoning": "Blocked robots are on critical pick path for high-priority wave. Forklift is nearest certified resource.",
  "recommendation": "Dispatch FL-3 to Aisle 4",
  "confidence": 0.72,
  "alternatives": ["Wait for self-recovery (risk: miss cut-off)", "Reroute robots via Aisle 3 (adds 8 min)"],
  "groundTruthNeeded": true
}
```

---

## Human-as-Sensor: Ground Truth Capture (Critical Feedback Loop)

### Mobile Web Interface (No App Install)
```
AI DETECTED: AMR Congestion — Aisle 4
5 robots blocked | Wave W-14 cut-off: 16:00 (12 min)

DISPATCHED: FL-3 (Forklift) to Aisle 4

FL-3, tap what you find:
[ PALLET CLEARED ]    [ CONVEYOR JAMMED ]
[ SENSOR ERROR    ]    [ OTHER: ________ ]

Optional note: ___________________________________
[SUBMIT]
```

### Why This Transforms Phase 1
1. **Solves missing sensors** — operator IS the sensor
2. **Trains facility-specific confidence** — "At Facility X, 'obstacle_detected' in Aisle 4 = pallet 78%"
3. **Creates ROI audit trail** — "AI flagged → operator confirmed pallet → resolved in 4 min"
4. **Builds trust** — supervisor sees AI + human collaborating, not AI guessing
5. **Shadow Mode** — AI records what it *would* do at L2/L3; supervisor dry-runs approval

---

## Actionable Notifications (The Wedge)

| Type | Channel | Example |
|------|---------|---------|
| **Exception Alert** | Slack/Teams/SMS | "Aisle 4: 5 Locus robots blocked. Wave W-14 (Shopify) cut-off 12 min. Forklift FL-3 available 50m. [VIEW] [DISPATCH]" |
| **Cut-off Risk** | Slack/Teams | "Wave W-14 at risk: 3 exceptions in pick path. Projected miss: 16:05. [DRILL DOWN]" |
| **Drift Warning** | Daily Digest | "PF-412 variance -12% over 3 counts. Recommend cycle count before peak." |
| **Resolution Confirmation** | Slack/Teams | "Aisle 4 cleared by FL-3 (pallet). Wave W-14 cut-off met. 4 min resolution." |

**Deep links:** [VIEW] → exception card with evidence/reasoning/actions | [DISPATCH] → sends task to LMS/operator mobile view  
**Key:** Write-back to **humans**, not systems — zero IT approval needed

---

## Multi-Client SLA Prioritization (Revenue Protection)

**Critical for 3PLs:** Multiple clients (Nike, Adidas) on same floor

**AI Intelligence:** 
> "Congestion in Aisle 4 blocking **High-Priority Nike wave** with 11:00 AM FedEx pickup, while robots in the way are for **Low-Priority Adidas wave** shipping tomorrow."

**Transforms from:** Exception tracker → **Revenue Protection Tool** (easier VP Ops sell)

---

## Safety & Compliance Metadata (Bonus Value)

Add to Logical Twin:
- **Safety/No-Go Zones:** Fire aisles, forklift-only zones, pedestrian paths
- **Trigger:** Robot/human lingering in restricted zone → High-Priority Safety Exception
- **Value:** Opens Insurance & Safety budgets (separate from Operations)

---

## Drift Detection (WMS vs Reality)

- Track inventory variance trends per location/SKU
- Flag: "PF-412 variance -12% over 3 counts → recommend cycle count before peak"
- Catches ghost SKU incidents **before** they cause pick failures

---

## Camera Event Ingestion (Tier 1d — Opportunistic)

**If prospect has Verkada/Rhombus/Arlo cloud cameras:**
- Ingest "person detected in zone" events via API
- Correlate: "Robot blocked + person detected same zone 30s ago = likely human obstruction"
- Zero hardware, privacy-safe (metadata only, no video)
- **Discovery question:** "Any cloud-connected cameras?"

---

## Discovery Questions (Qualification)

| Question | Tier | Purpose |
|----------|------|---------|
| API access to WMS? | 0 | Disqualify if No |
| Robot vendor(s)? REST API? | 1a | Primary integration target |
| Export locations/zones as CSV? | Map | 15-min Logical Twin |
| Cloud cameras (Verkada, Rhombus)? | 1d | Bonus CV correlation |
| Cloud WES with equipment API? | 1b | Bonus conveyor data |
| CMMS? Which one? | 1c | Phase 2 L2 target |
| On-prem WMS or cloud? | 0/2 | IT review timeline |
| Conveyor/sortation automation? | 3 | Phase 4 roadmap interest |

---

## Implementation Roadmap

| Sprint | Focus | Deliverable |
|--------|-------|-------------|
| 1-2 | ShipHero Adapter + FacilityState Core | Ingest orders, waves, inventory, locations |
| 3-4 | Locus Adapter + Robot State | Real-time robot poses, missions, errors |
| 5 | Map Importer + ZoneGraph | CSV upload → Logical Twin in 15 min |
| 6-7 | Exception Engine + Drift Detection | Correlated exceptions, variance flags |
| 8-10 | LLM Reasoning Layer | Evidence → Reasoning + Confidence + Actions |
| 11 | Dashboard (Exception Cards) | Evidence panel, reasoning, confidence, actions |
| 12 | Notifications (Slack/Teams/SMS) | Deep links, [VIEW], [DISPATCH] |
| 13 | Ground Truth Capture | Mobile web view, Pallet/Conveyor/Sensor buttons |
| 14 | Trust Ladder Framework | L1 active, L2-L4 gated, audit trail export |

**MVP (Sprints 1-8, ~8 weeks):** Dashboard + Notifications + Drift + Map Importer  
**Full Phase 1 (Sprints 1-14, ~14 weeks):** + Reasoning + Ground Truth + Trust Ladder

---

## Competitive Landscape & Differentiation

| Competitor Type | Strength | Our Wedge |
|-----------------|----------|-----------|
| **WMS Vendors** | Deep order/inventory | No robot/labor fusion, no spatial reasoning |
| **Robot Vendors** | Single-fleet orchestration | Cross-vendor, cross-system (human + robot + equipment) |
| **System Integrators** | Custom integration | One-off, no product, no compounding intelligence |
| **AI Agent Startups** | Black-box automation | No Trust Ladder, no human feedback loop, no audit trail |

**Our Moats:**
1. **Common Data Model + Adapters** — integrate with THEIR systems
2. **Logical Twin (Zone Graph)** — spatial reasoning WMS lacks
3. **Facility-specific confidence learning** via Human-as-Sensor
4. **Trust Ladder** — de-risks expansion, IT controls pace
5. **Compound intelligence** — every resolution trains the model

---

## Fundraising Answers (Complete)

| Investor Question | Answer |
|-------------------|--------|
| **Why now?** | LLMs enable reasoning over messy industrial logs; cloud APIs finally ubiquitous in mid-market 3PLs |
| **Moat?** | 1) Common Data Model + Adapters 2) Logical Twin (Zone Graph) 3) Facility-specific confidence learning via Human-as-Sensor 4) Trust Ladder de-risks expansion |
| **Hardware risk?** | Zero for Phase 1-3. Phase 4 edge gateway = separate product, separate round, Series B+ |
| **WMS lock-in?** | We don't replace WMS — we integrate with THEIRS via adapters to canonical model |
| **Pilot to scale?** | 2-week deploy, value day 1, self-serve onboarding, operator feedback compounds intelligence |
| **TAM?** | 2,000+ mid-market 3PLs (Phase 1) → 5,000+ with legacy WMS (Phase 3) → 10,000+ with equipment (Phase 4) |
| **Competition?** | WMS (no robot/labor fusion), Robot vendors (single-fleet), Integrators (one-off, no product), AI agents (black box, no trust ladder) |

---

## Final GTM Pitch — Phase 1

> **We deploy in 2 weeks with just API keys.**
>
> **Week 1:** You upload your locations CSV. Our Map Builder generates your Logical Twin — a spatial brain that knows Aisle 4 feeds Sorter 1, Aisle 3 is the alternate route, and forklifts can't use the robot charging zone.
>
> **Week 2:** We connect ShipHero + Locus. You get:
> 1. One screen — robots, orders, inventory, labor, exceptions correlated
> 2. AI explanations — 5 robots blocked in Aisle 4 + wave cut-off 12 min + forklift 50m away = dispatch forklift (72% confidence)
> 3. Actionable alerts — Slack/Teams with [VIEW] and [DISPATCH] buttons
> 4. Ground truth capture — forklift operator taps Pallet Cleared or Conveyor Jammed; trains our AI for YOUR facility
> 5. Drift detection — PF-412 variance -12% flagged BEFORE ghost SKU incident
>
> **Zero hardware. Zero PLC. Zero WCS dependency. Zero write access to your WMS.**
>
> **Trust Ladder:** We start at Level 1 (human notifications only). After 8 weeks of audit-proven accuracy, you decide if we unlock Level 2 (CMMS tickets), then Level 3 (robot reroute), then Level 4 (WMS write-back).
>
> When you want conveyor sensors, that's our Phase 4 edge gateway — separate product, separate budget.

---

## Key Decisions Made in This Thread

1. **Software-only Phase 1** — no hardware, no PLC, no edge gateway
2. **ShipHero + Locus as beachhead adapters** — Tier 0 + Tier 1a
3. **Canonical Data Model** as the integration moat
4. **LLM Reasoning Layer** over deterministic rules — handles messy real-world data
5. **Human-as-Sensor** mobile web view — solves missing sensors, builds trust, creates training data
6. **Actionable Notifications** (not passive dashboards) — the sales wedge
7. **Logical Twin (Zone Graph)** — spatial reasoning, not 3D digital twin
8. **Trust Ladder (L1→L4)** — IT controls write-access progression
9. **Map Importer** — 15-min CSV → Logical Twin, makes 2-week deploy real
10. **Multi-Client SLA Prioritization** — revenue protection for 3PLs
11. **Safety/Compliance Metadata** — opens separate budget
12. **Shadow Mode** — dry-run L2/L3 actions during L1 to build accuracy evidence
13. **Camera Events (Tier 1d)** — opportunistic, zero hardware bonus
14. **Drift Detection** — WMS vs reality divergence monitoring

---

## Data Requirements Summary

| Data Source | Required For | Integration Method |
|-------------|--------------|-------------------|
| WMS (orders, waves, inventory, locations) | Tier 0 — Core | REST API (ShipHero first) |
| Robot Fleet (poses, missions, errors, battery) | Tier 1a — Core | REST API / VDA 5050 (Locus first) |
| Labor (associates, certifications, zones, tasks) | Tier 1b — Core | CSV upload / API |
| CMMS (work orders, assets) | Tier 1c — Phase 2 | API (optional Phase 1) |
| Camera Events (person/zone detection) | Tier 1d — Bonus | Cloud API (Verkada/Rhombus) |
| Facility Map (locations, zones, adjacency) | Map Importer — Core | CSV/Excel upload |
| Conveyor/Equipment (PLC, OPC UA, Modbus) | Phase 4 | Edge gateway (future) |

---

## Scenario × Tier Impact Matrix

| Scenario | Tier 0 (WMS) | Tier 1a (Robots) | Tier 1b (Labor) | Tier 1c (CMMS) | Tier 1d (Cameras) | Map/Logical Twin |
|----------|--------------|------------------|-----------------|----------------|-------------------|------------------|
| AMR stuck in aisle | ❌ | ✅ Detect | ✅ Dispatch human | ❌ | ✅ Correlate human presence | ✅ Alternate route |
| Inventory missing at station | ✅ Detect variance | ❌ | ✅ Dispatch replenish | ❌ | ❌ | ✅ Nearest stock location |
| Conveyor jam | ❌ | ❌ | ✅ Dispatch tech | ✅ Create WO | ✅ Detect stopped flow | ✅ Upstream/downstream impact |
| Robot repeated failures | ❌ | ✅ Detect pattern | ✅ Escalate | ✅ Predictive WO | ❌ | ✅ Choke point analysis |
| Worker shortage | ✅ Wave at risk | ❌ | ✅ Detect gap | ❌ | ✅ Zone occupancy | ✅ Rebalance zones |
| Dock congestion | ✅ Inbound delayed | ✅ Staging full | ✅ Reassign labor | ❌ | ✅ Trailer detection | ✅ Dock-door mapping |
| Multi-client SLA conflict | ✅ Wave priority | ✅ Robot allocation | ✅ Labor allocation | ❌ | ❌ | ✅ Shared resource optimization |

---

## Human Interface Summary

| Interface | Audience | Purpose |
|-----------|----------|---------|
| **Dashboard (Web)** | Supervisors, Ops Leads | Exception cards, evidence, reasoning, confidence, actions |
| **Slack/Teams/SMS Notifications** | Floor Leads, Techs, Supervisors | Real-time alerts with [VIEW] [DISPATCH] deep links |
| **Mobile Web (Ground Truth)** | Forklift Ops, Techs, Associates | Tap findings: Pallet/Conveyor/Sensor/Other + notes |
| **Daily Digest (Email/Slack)** | Ops Managers | Drift warnings, variance trends, SLA risk summary |
| **Shadow Mode Review** | Supervisors (Phase 1→2 transition) | Dry-run approve AI-proposed L2/L3 actions |

---

## Next Steps

1. **Validate ShipHero + Locus API access** with design partner(s)
2. **Build Map Importer** — prove 15-min Logical Twin
3. **Implement Tier 0 + 1a adapters** — canonical data model ingestion
4. **Design Exception Engine** — correlation rules + LLM reasoning prompts
5. **Prototype Ground Truth mobile view** — test with real operators
6. **Define Trust Ladder audit metrics** — what accuracy % unlocks L2?
7. **Run Shadow Mode** in pilot — collect L2/L3 dry-run data
8. **Prepare fundraising deck** with complete GTM + moats + TAM

---

*Document compiled from Research channel thread ccb67e75be43cea8e49d8da12addbe57796c7bbc6d3759238ad018f7774a6dcf*
