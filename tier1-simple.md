# Tier 0 + 1a + 1b Architecture (Simple View)

```
┌─────────────────────────────────────────────────────────────────┐
│                    AI OPERATIONAL SUPERVISOR                     │
│                     (Your Software)                              │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ WMS Adapter  │  │ Robot Adapter│  │ Labor Adapter│           │
│  │  (ShipHero)  │  │   (Locus)    │  │  (Tier 1b)   │           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │
│         │                 │                 │                    │
│         └─────────────────┼─────────────────┘                    │
│                           ▼                                      │
│              ┌────────────────────────┐                          │
│              │   FACILITY STATE       │                          │
│              │  (Canonical Model)     │                          │
│              │  • Waves, Robots       │                          │
│              │  • Workers, Locations  │                          │
│              │  • ZoneGraph ref       │                          │
│              └───────────┬────────────┘                          │
│                          │                                       │
│         ┌────────────────┼────────────────┐                     │
│         ▼                ▼                ▼                     │
│  ┌─────────────┐ ┌───────────────┐ ┌─────────────┐             │
│  │  Exception  │ │  LLM Reasoning│ │ Action      │             │
│  │   Engine    │ │  (Confidence, │ │  Planner    │             │
│  │  (Detect)   │ │   Alternatives)│ │  (Dispatch) │             │
│  └──────┬──────┘ └───────┬───────┘ └──────┬──────┘             │
│         │                │                │                      │
│         └────────────────┼────────────────┘                      │
│                          ▼                                      │
│              ┌────────────────────────┐                          │
│              │  HUMAN-IN-THE-LOOP     │                          │
│              │  • Slack/Teams Alerts  │                          │
│              │  • Deep Links → Mobile │                          │
│              │  • Evidence Panel      │                          │
│              │  • Ground Truth Capture│                          │
│              └───────────┬────────────┘                          │
│                          │                                       │
│                          ▼                                       │
│              ┌────────────────────────┐                          │
│              │   LEARNING LOOP        │                          │
│              │  • Event Log (append)  │                          │
│              │  • Facility Tuning     │                          │
│              └────────────────────────┘                          │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
┌─────────────────────┐ ┌───────────────┐ ┌─────────────────┐
│     WMS (Tier 0)    │ │ ROBOT FLEET   │ │  LABOR (Tier 1b)│
│     ShipHero        │ │   (Tier 1a)   │ │                 │
│                     │ │   Locus       │ │  Worker API     │
│  • /waves           │ │               │ │  • location     │
│  • /wave-items      │ │  • /robots    │ │  • certs        │
│  • /locations       │ │  • /events    │ │  • tasks        │
│  • /inventory       │ │    (obstacle_ │ │                 │
│  • webhooks         │ │     detected) │ │  Mobile App     │
└─────────────────────┘ └───────────────┘ └─────────────────┘
              │               │               │
              └───────────────┼───────────────┘
                              ▼
                   ┌─────────────────────┐
                   │  NOTIFICATIONS      │
                   │  • Slack / Teams    │
                   │  • SMS / Push       │
                   └─────────────────────┘
```

## Data Flow (Scenario 1: AMR Congestion)

```
Locus: 5 robots → obstacle_detected
  │
  ▼
Robot Adapter → FacilityState: "5 robots blocked in Aisle 4"
                                                    │
ShipHero: Wave W-14 cutoff 17:00 (18 min)           │
  │                                                 │
  ▼                                                 │
WMS Adapter → FacilityState: "Wave W-14 HIGH, 47 picks at risk"
                                                    │
Labor: Forklift FL-3 idle, 50m away, certified ─────┘
  │
  ▼
Labor Adapter → FacilityState: "FL-3 available"
                                                    │
ZoneGraph: "Aisle 3 alternate, 2m wider, +8 min" ──┘
  │
  ▼
Exception Engine: PATTERN MATCH → "Critical wave + blocked robots + worker available"
  │
  ▼
LLM Reasoning: "Dispatch FL-3 (72%) | Reroute via Aisle 3 (+8 min, needs human first)"
  │
  ▼
Action Planner → Slack: Actionable message with [DISPATCH FL-3] [REROUTE] [VIEW EVIDENCE]
  │
  ▼
Floor Lead taps [DISPATCH FL-3] → Deep link → Mobile web on FL-3 operator phone
  │
  ▼
Operator taps [PALLET CLEARED] → Ground Truth: {finding: "pallet", confidence: 0.95}
  │
  ▼
Event Log ← Facility Model: "Aisle 4 obstacle → pallet probability +5%"
```

## Why This Tier First

| Tier | System | Scenario 1 | Scenario 2 | Scenario 3 |
|------|--------|------------|------------|------------|
| 0    | WMS (ShipHero) | ✅ Wave cutoff | ✅ Inventory drift | ✅ Wave priorities |
| 1a   | Robot Fleet (Locus) | ✅ Obstacle events | — | ✅ Queue status |
| 1b   | Labor | ✅ Forklift dispatch | ✅ Count team | ✅ Manual sort staff |
| 1c   | CMMS | — | — | ✅ Tech dispatch |
| 1d   | Cameras | — | — | ✅ Jam detection (optional) |

**Beachhead = Tier 0 + 1a + 1b** — covers Scenarios 1 & 2 with APIs most warehouses already have.
