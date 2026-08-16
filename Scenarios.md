# Three Detailed Scenarios — How the AI Supervisor Solves Real Operational Problems

---

## Scenario 1: AMR Congestion Blocking a High-Priority Wave Cut-Off

### The Problem
**16:42** — Five Locus AMRs stall in Aisle 4 (obstacle_detected events). They're on the critical pick path for **Wave W-14 (Client: Nike, Priority: HIGH)** with a **FedEx pickup at 17:00** (18 minutes). The robots are blocking each other — none can reroute autonomously because the fleet manager only sees its own robots, not the wave context. Supervisor is on the other side of the 200K sq ft floor.

### What the AI Supervisor Does

| Time | Action | Systems Used |
|------|--------|--------------|
| **16:42:03** | Ingests 5× `obstacle_detected` from Locus API (Tier 1a) | Robot Fleet Adapter |
| **16:42:04** | Correlates with ShipHero: Wave W-14 cut-off 17:00, 47 picks remaining in Aisle 4/5 | WMS Adapter (Tier 0) |
| **16:42:05** | Queries Logical Twin (Zone Graph): Aisle 4 → feeds Sorter 1; alternate route = Aisle 3 (2m wider, forklift-accessible) | Map Importer / Logical Twin |
| **16:42:06** | Checks Labor: Forklift FL-3 (certified for pallet moves) is 50m away in Zone B, idle | Labor Adapter (Tier 1b) |
| **16:42:07** | LLM Reasoning: "5 robots blocked on critical path + 18 min to cut-off + forklift 50m away = dispatch forklift (72% confidence). Alternatives: wait (risk miss), reroute via Aisle 3 (adds 8 min, requires human to clear first)" | LLM Reasoning Layer |
| **16:42:08** | Sends **Actionable Notification** to Slack #ops-alerts + SMS to floor lead | Notifications (Slack/Teams/SMS) |

### Human Interface — What the Floor Lead Sees (Slack Message)

```
🚨 EXCEPTION: AMR Congestion — Aisle 4
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📍 Zone: Aisle 4 (feeds Sorter 1)
🤖 5 Locus robots blocked — obstacle_detected
📦 Wave W-14 (Nike) — HIGH PRIORITY
⏰ Cut-off: 17:00 (18 min) | 47 picks at risk
🚜 Forklift FL-3 available 50m (Zone B, certified)

🧠 AI RECOMMENDATION (72% confidence)
Dispatch FL-3 to Aisle 4 — clear obstruction, robots resume.
Alternative: Reroute via Aisle 3 (+8 min, needs human first)

[VIEW EVIDENCE]  [DISPATCH FL-3]  [REROUTE VIA AISLE 3]
```

### Human Action & Ground Truth Capture

Floor lead taps **[DISPATCH FL-3]** → deep link opens mobile web view on FL-3 operator's phone:

```
🎯 TASK ASSIGNED: Clear Aisle 4 Obstruction
Wave: W-14 (Nike) | Cut-off: 17:00 (14 min remaining)
Dispatched by: AI Supervisor + Floor Lead Maria

FL-3, tap what you find:
[ PALLET CLEARED ]     [ CONVEYOR JAMMED ]
[ SENSOR ERROR      ]  [ OTHER: ________ ]

Note: _______________________________________
[SUBMIT]
```

**16:46** — Operator taps **[PALLET CLEARED]**, adds note: "Stretch wrap on floor, removed."

**16:46:05** — System logs resolution: `groundTruth: {finding: "pallet", confidence: 0.95, resolver: "FL-3", duration: 4 min}`. Updates facility model: `Aisle 4 obstacle_detected → pallet probability +5%`. Sends resolution confirmation to Slack.

### Outcome
- **Before:** 15-30 min to notice, diagnose, dispatch, resolve → likely miss cut-off
- **After:** 4 min detection → dispatch → resolution → cut-off met
- **Data captured:** Exception + evidence + reasoning + human action + outcome = training data

---

## Scenario 2: Inventory Drift Creating Ghost SKU Before Peak

### The Problem
**Day 3 of peak season** — Location PF-412 (SKU: NIKE-AIR-10-BLK) shows 24 units in ShipHero. Physical count by associate during putaway: 18 units. Variance: **-25%**. This is the 3rd count in 2 weeks showing drift (-8%, -15%, -25%). If a wave picks this location tomorrow, 6 orders short-ship. No alert exists in WMS — it trusts its own numbers.

### What the AI Supervisor Does

| Time | Action | Systems Used |
|------|--------|--------------|
| **Daily 06:00** | Drift Detection job runs: compares last 3 cycle counts vs WMS onHand for all locations | Drift Detection (Core) |
| **06:00:12** | Flags PF-412: variance trend -8% → -15% → -25% (accelerating). SKU is in 3 waves tomorrow (2 Nike, 1 Adidas) | WMS Adapter + Canonical Model |
| **06:00:15** | LLM Reasoning: "Accelerating negative variance + high-velocity SKU in tomorrow's waves = ghost SKU risk. Recommend: emergency cycle count tonight, flag waves for picker awareness" | LLM Reasoning Layer |
| **06:00:18** | Sends **Daily Digest** to Ops Manager email + Slack #inventory-alerts | Notifications (Daily Digest) |

### Human Interface — Daily Digest (Email/Slack)

```
📊 DAILY DRIFT REPORT — 2026-11-15
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️ HIGH RISK: PF-412 (NIKE-AIR-10-BLK)
   WMS onHand: 24 | Last 3 counts: 22, 19, 18
   Variance trend: -8% → -15% → -25% ⬇️ ACCELERATING
   In tomorrow's waves: W-22 (Nike), W-23 (Nike), W-24 (Adidas)
   Projected short-ship: 6-8 units across 3 waves

🧠 AI RECOMMENDATION
1. Emergency cycle count PF-412 tonight (assign to count team)
2. Add "⚠️ VARIANCE FLAG" badge to pick tasks for this location tomorrow
3. If count confirms <15, trigger inter-warehouse transfer from WH-2

[VIEW LOCATION]  [ASSIGN CYCLE COUNT]  [FLAG TOMORROW'S WAVES]
```

### Human Action

Ops Manager taps **[ASSIGN CYCLE COUNT]** → creates task in Labor system (Tier 1b) for count team lead. Count team lead gets mobile notification.

**22:30** — Count team completes count: **16 units confirmed**. Taps **[CONFIRM COUNT: 16]** in mobile web.

**22:30:10** — System updates: `driftFlags: {location: PF-412, confirmed: 16, wms: 24, action: "waves flagged"}`. Sends update to Waves W-22/23/24 pick tasks: pickers see "⚠️ VARIANCE FLAG — verify quantity" on their scanners.

### Outcome
- **Before:** 6-8 short-ships discovered at pack-out → expedite fees, SLA breach, client escalation
- **After:** Proactive flag 18 hours before → pickers verify, short-ships prevented, client never knows
- **Data captured:** Drift trend + recommendation + human verification + prevention outcome

---

## Scenario 3: Conveyor Jam + Multi-Client SLA Conflict Resolution

### The Problem
**10:15** — Main sorter conveyor (Sorter 1) jams. Upstream: 12 Locus robots queued in Aisle 4/5 with totes. Downstream: 3 pack stations idle. Two waves affected:
- **Wave W-31 (Client: Nike, Priority: CRITICAL)** — 11:30 UPS pickup, 87 totes on sorter
- **Wave W-32 (Client: Adidas, Priority: STANDARD)** — 14:00 pickup, 156 totes on sorter

Robots are stuck because WES doesn't know wave priorities. Supervisor must manually decide: clear Nike totes first? Split resources? Call maintenance?

### What the AI Supervisor Does

| Time | Action | Systems Used |
|------|--------|--------------|
| **10:15:02** | Camera event (Verkada): "person detected Zone Sorter-1" + "motion stopped conveyor belt" | Camera Events (Tier 1d) |
| **10:15:03** | Robot telemetry: 12 robots `waiting_at_sorter` in Aisle 4/5 | Robot Fleet Adapter (Tier 1a) |
| **10:15:04** | WMS: W-31 (Nike, CRITICAL, 11:30 pickup), W-32 (Adidas, STANDARD, 14:00) | WMS Adapter (Tier 0) |
| **10:15:05** | CMMS: Maintenance tech MT-2 on shift, certified for sorter repair, currently in Zone C | CMMS Adapter (Tier 1c) |
| **10:15:06** | Logical Twin: Sorter 1 is single choke point. Alternate: manual sort at Station 7 (capacity 40 totes/hr) | Logical Twin (Zone Graph) |
| **10:15:07** | LLM Reasoning: "Sorter jam + 2 waves + CRITICAL Nike 75 min cutoff + STANDARD Adidas 225 min + 1 tech + manual sort capacity 40/hr = Dispatch MT-2 to Sorter 1 (ETA 8 min). Parallel: divert Nike totes to manual sort (40/hr × 1.25 hr = 50 totes). Hold Adidas totes at robots. Projected: Nike 87 totes cleared by 11:25 (meets cutoff), Adidas delayed 45 min (within window)." | LLM Reasoning Layer |
| **10:15:08** | Sends **Exception Alert** to Slack #ops-alerts + SMS to maintenance lead | Notifications |

### Human Interface — Exception Alert (Slack)

```
🚨 EXCEPTION: Sorter 1 Jam — Multi-Wave Impact
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📍 Zone: Sorter 1 (main choke point)
🔧 Conveyor jam detected (Verkada camera + robot queue)
🤖 12 Locus robots queued (Aisle 4/5)

📦 WAVE IMPACT
🔴 W-31 (NIKE) — CRITICAL — 11:30 UPS pickup (75 min) — 87 totes on sorter
🟢 W-32 (ADIDAS) — STANDARD — 14:00 pickup (225 min) — 156 totes on sorter

👷 MAINTENANCE: MT-2 available (Zone C, sorter-certified, ETA 8 min)
🔄 ALTERNATE: Manual Sort Station 7 (capacity 40 totes/hr)

🧠 AI RECOMMENDATION (78% confidence)
1. DISPATCH MT-2 → Sorter 1 (primary fix)
2. DIVERT NIKE totes → Manual Sort Station 7 (50 totes by 11:25)
3. HOLD ADIDAS totes at robots (release after Nike clears)
4. PROJECTED: Nike ✅ meets cutoff | Adidas ⏱️ +45 min delay (within SLA)

[VIEW EVIDENCE]  [DISPATCH MT-2]  [START MANUAL SORT]  [HOLD ADIDAS]
```

### Human Actions (Parallel)

**Floor Lead** taps **[DISPATCH MT-2]** → maintenance tech gets mobile task.

**Supervisor** taps **[START MANUAL SORT]** → assigns 2 associates to Station 7 via Labor system.

**10:18** — MT-2 arrives, taps mobile web: **[MECHANICAL JAM — CLEARED]**.

**10:23** — Sorter restarts. Nike totes flow. Manual sort processes 12 totes in first 15 min.

**10:38** — 62 Nike totes through sorter + 12 manual = 74 done. 13 remaining.

**10:55** — All 87 Nike totes processed. Wave W-31 **cut-off MET**.

**11:00** — Adidas totes released from robots. Sorter processes at full speed.

### Outcome
- **Before:** Supervisor guesses priority, likely misses Nike cutoff or delays Adidas excessively. No data on what worked.
- **After:** AI optimizes across **constraints** (tech availability, manual capacity, wave priorities, cutoffs). Both waves saved. Nike meets critical cutoff; Adidas absorbs minor delay within SLA.
- **Revenue Protection:** Nike (strategic client) never misses pickup → contract retained.
- **Data captured:** Full decision trace + human actions + outcomes = facility-specific optimization model

---

## Summary: What Each Scenario Uses

| Component | Scenario 1 (AMR Congestion) | Scenario 2 (Inventory Drift) | Scenario 3 (Sorter Jam) |
|-----------|------------------------------|------------------------------|--------------------------|
| **WMS (Tier 0)** | ✅ Wave cut-off, pick path | ✅ onHand, wave assignments | ✅ Wave priorities, totes |
| **Robot Fleet (Tier 1a)** | ✅ Poses, obstacle events | ❌ | ✅ Queue status, locations |
| **Labor (Tier 1b)** | ✅ Forklift location, certs | ✅ Count team assignment | ✅ Manual sort staffing |
| **CMMS (Tier 1c)** | ❌ | ❌ | ✅ Tech availability, certs |
| **Cameras (Tier 1d)** | ❌ | ❌ | ✅ Jam detection, person |
| **Logical Twin** | ✅ Alternate route (Aisle 3) | ❌ | ✅ Choke point, manual sort cap |
| **LLM Reasoning** | ✅ Dispatch vs reroute | ✅ Drift trend → action | ✅ Multi-wave optimization |
| **Notifications** | ✅ Slack + SMS + Deep links | ✅ Daily Digest + actions | ✅ Slack + SMS + actions |
| **Ground Truth** | ✅ Operator tap: pallet | ✅ Count team: confirmed 16 | ✅ Tech: mechanical jam |
| **Human Interface** | Mobile web (forklift op) | Mobile web (count team) | Mobile web (tech) + Slack (lead) |
| **Trust Ladder** | L1 (human dispatch) | L1 (human count) | L1 (human dispatch) → L2 ready (CMMS WO) |

---

## Key Pattern Across All Three

1. **Detect** — Multi-source ingestion (WMS + robots + cameras + labor)
2. **Correlate** — Canonical model + Logical Twin connects the dots
3. **Reason** — LLM explains WHY, weighs alternatives, gives confidence
4. **Act** — Actionable notification with deep links (not passive alert)
5. **Capture** — Human validates/acts → ground truth trains the model
6. **Learn** — Facility-specific confidence improves every cycle

This is the **compound intelligence loop** that becomes the moat.
