Here's the architecture diagram for **Tier 0 + 1a + 1b** (Scenario 1: AMR Congestion) — the beachhead you're building first.

Saved to  as a Mermaid diagram. You can render it in any Mermaid viewer (GitHub, Notion, Mermaid Live Editor, VS Code with Mermaid extension).

## What the diagram shows

**External Systems (Customer Environment):**
- **WMS — ShipHero (Tier 0)**: Wave cut-offs, pick paths, inventory — polled 30s + webhooks
- **Robot Fleet — Locus (Tier 1a)**: Robot poses, status,  events — polled 15s + webhooks
- **Labor System (Tier 1b)**: Worker location, certifications, task dispatch — polled 30s + webhooks
- **Notifications**: Slack/Teams + SMS for actionable alerts

**Your Software (AI Operational Supervisor):**

| Layer | Components | Key Responsibility |
|-------|------------|-------------------|
| **Adapter Layer** | WMS/Robot/Labor Adapters | Normalize vendor-specific APIs → Canonical types |
| **Core Facility State** | FacilityState, ZoneGraph, Drift Detector | Single source of truth + spatial reasoning |
| **Intelligence Layer** | Exception Engine, LLM Reasoning, Action Planner | Detect → Reason (with confidence) → Plan |
| **Human-in-the-Loop** | Notification Dispatcher, Ground Truth Capture, Evidence Panel | Actionable Slack messages with deep links → mobile web for operator response |
| **Learning Loop** | Append-Only Event Log, Facility Model Tuning | Every exception + human action + outcome = training data |

## Data Flow for Scenario 1

1. **Ingest**: 5×  from Locus (15s poll/webhook) + Wave W-14 cutoff from ShipHero (30s poll)
2. **Correlate**: Exception Engine matches pattern "5 robots blocked + critical wave cutoff 18 min"
3. **Reason**: LLM gets full context — wave priority, forklift FL-3 50m away (idle, certified), ZoneGraph alt route (Aisle 3, +8 min)
4. **Act**: Structured JSON recommendation → Actionable Slack message with **[DISPATCH FL-3]** / **[REROUTE]** / **[VIEW EVIDENCE]**
5. **Capture**: Floor lead taps dispatch → deep link opens mobile web on FL-3 operator's phone → operator taps **[PALLET CLEARED]**
6. **Learn**: Ground truth () appended to event log → facility model updates 

## Why this tier first

- **Tier 0 (WMS)** + **Tier 1a (Robots)** + **Tier 1b (Labor)** = Scenario 1 **and** Scenario 2 (Inventory Drift)
- No cameras, no CMMS needed — works with APIs most warehouses already have
- High ROI: prevents missed cut-offs (Scenario 1) and ghost SKUs (Scenario 2)
- Scenario 3 (Sorter Jam) adds Tier 1c (CMMS) + Tier 1d (Cameras) later as expansion

The  is the spatial brain — built once per facility during onboarding (import CAD/floor plan → nodes/edges with widths, cert requirements, alternate routes). This is what lets the LLM know "Aisle 3 is 2m wider and forklift-accessible" without hardcoding.