# 3PL AI Supervisor --- End-to-End Scenario

## Scenario

It is 11:00 AM. Client A has a 2:00 PM shipping cutoff.

Initial state: - Zone A: 8 pickers - Zone B: 6 pickers - 12 AMRs
assisting picking - Client A: 2,800 picks remaining - Client B: 1,100
picks remaining

Then **two Zone A workers become unavailable and three AMRs become
blocked in Aisle 14**. Together these disruptions cause Client A's
picking operation to fall behind.

This scenario follows the disruption through the entire architecture.

------------------------------------------------------------------------

## 1. Data Sources

### WMS

The WMS sees the operational result:

``` text
Zone A:
remaining_picks = 2,800
recent_pick_rate = 610/hr
cutoff = 14:00
```

It also continuously produces pick-task events.

### Labor System

``` text
Worker 17 → CLOCKED_OUT
Worker 24 → UNAVAILABLE

Zone A:
scheduled = 8
available = 6
```

### Robot Fleet Manager

``` text
AMR-04 → BLOCKED
AMR-07 → BLOCKED
AMR-09 → BLOCKED

location = Aisle 14
reason = congestion/obstacle
```

No individual system necessarily understands the complete operational
problem. The WMS sees falling throughput, the labor system sees
unavailable workers, and the fleet manager sees blocked robots.

------------------------------------------------------------------------

## 2. Adapter Layer

Each vendor has its own API/schema.

A labor API might provide:

``` json
{"employee":"17","status":"OUT"}
```

A robot fleet API might provide:

``` json
{"bot":"AMR-04","state":37}
```

Your adapters translate them into your internal language:

``` text
WORKER_UNAVAILABLE
worker_id: 17
zone: A
```

``` text
ROBOT_BLOCKED
robot_id: AMR-04
zone: A
```

``` text
PICK_TASK_COMPLETED
zone: A
customer: Client-A
```

This is primarily normal software engineering, not an LLM.

------------------------------------------------------------------------

## 3. Canonical Event Stream

All normalized events enter one stream:

``` text
11:01:04  WORKER_UNAVAILABLE    Worker-17     Zone-A
11:02:17  WORKER_UNAVAILABLE    Worker-24     Zone-A

11:03:02  ROBOT_BLOCKED         AMR-04        Aisle-14
11:03:08  ROBOT_BLOCKED         AMR-07        Aisle-14
11:03:15  ROBOT_BLOCKED         AMR-09        Aisle-14

11:04:01  PICK_COMPLETED        Order-821
11:04:07  PICK_COMPLETED        Order-934
...
```

Everything downstream now consumes the same event format regardless of
vendor.

------------------------------------------------------------------------

## 4. System of Record

Every event is persisted for history, auditing, analysis, and replay.

The platform also maintains current state:

``` text
CURRENT WAREHOUSE STATE

Zone A
────────────────────
Workers available:       6
Workers scheduled:       8

Robots available:        9
Robots blocked:          3

Remaining picks:         2,800
Current throughput:      610/hr

Client A cutoff:         14:00
```

Historical state can show:

``` text
Zone A normal throughput
with 8 workers + 12 robots:
~1,050 picks/hr
```

The system of record becomes the operational memory of the platform.

------------------------------------------------------------------------

## 5. Analytics & State Engine

Algorithms calculate:

``` text
Normal throughput: 1,050/hr
Current throughput: 610/hr
Change: ↓ 42%
```

Then:

``` text
Remaining work:       2,800 picks
Current rate:         610 picks/hr
Estimated duration:   4.59 hours
Current time:         11:05
Projected completion: ~15:40
Customer cutoff:      14:00
```

Therefore:

> **Client A is projected to miss cutoff by approximately 100 minutes.**

The engine can correlate:

``` text
Worker capacity ↓
Robot capacity ↓
Throughput ↓
Backlog ↑
```

This stage primarily uses deterministic algorithms/statistics, with
predictive ML potentially added later.

------------------------------------------------------------------------

## 6. Exception Engine

A rule might be:

``` text
IF projected_completion > SLA_cutoff
AND throughput < required_throughput
AND condition persists > 3 minutes
THEN create SLA_RISK_EXCEPTION
```

The output becomes:

``` text
EXCEPTION #1842

Type:
PICKING_CAPACITY_SHORTAGE

Severity:
HIGH

Customer:
Client A

Affected area:
Zone A

Projected delay:
100 minutes

Likely contributing factors:
- 2 workers unavailable
- 3 AMRs blocked
- throughput down 42%

Evidence confidence:
HIGH
```

This is the structured handoff to the AI supervisor.

------------------------------------------------------------------------

## 7. AI Supervisor

The LLM receives a structured context pack rather than thousands of raw
events:

``` text
ZONE A
6 available workers
610 picks/hr
2,800 remaining
Client A cutoff: 14:00
Projected completion: 15:40

ZONE B
Workers: 6
Required: 4
Ahead of schedule: 74 minutes

Available qualified workers:
Worker 31
Worker 38

ROBOTS
3 blocked in Aisle 14
Alternative route: Aisle 16
Expected additional travel: +11%

CLIENT A
Priority: Same-day
Cutoff: 14:00

CLIENT B
Priority: Standard
Cutoff: 17:00
```

The LLM generates candidate recovery strategies:

-   **Option A:** Move two Zone B workers to Zone A.
-   **Option B:** Reroute the three blocked AMRs through Aisle 16.
-   **Option C:** Do both.
-   **Option D:** Do both and prioritize Client A's pick tasks until
    risk clears.

The LLM's job is flexible reasoning and strategy generation.

------------------------------------------------------------------------

## 8. What-If Simulator / Optimizer

The system does not blindly trust the LLM.

It evaluates candidate actions:

``` text
OPTION A — Move 2 workers
Client A finish: 14:31
❌ SLA missed
```

``` text
OPTION B — Reroute robots
Client A finish: 14:18
❌ SLA missed
```

``` text
OPTION C — Move 2 workers + reroute robots
Client A finish: 13:48
✅ SLA

Client B finish: 15:51
✅ SLA
```

``` text
OPTION D — Option C + prioritize Client A
Client A finish: 13:39
Client B finish: 16:02
Additional disruption cost: higher than C
```

The optimizer concludes:

> **Option C provides the lowest intervention while protecting both
> customer SLAs.**

------------------------------------------------------------------------

## 9. Human in the Loop

The warehouse supervisor receives:

``` text
HIGH — Client A SLA Risk

Zone A projected 100 minutes late.

CAUSE
• 2 workers unavailable
• 3 AMRs blocked
• Picking throughput ↓42%

RECOMMENDED RECOVERY
• Move Workers 31 & 38 from Zone B → Zone A
• Reroute AMR-04, AMR-07, AMR-09 through Aisle 16

PREDICTED RESULT
Client A completion: 1:48 PM
Client B remains on schedule

Confidence: 91%

[ APPROVE ] [ MODIFY ] [ REJECT ] [ VIEW EVIDENCE ]
```

The supervisor selects **APPROVE**.

------------------------------------------------------------------------

## 10. Action Dispatcher

The LLM does not directly make arbitrary calls to warehouse systems.

The approved plan becomes validated structured commands.

### Labor Adapter

``` text
REASSIGN

Worker 31:
Zone B → Zone A

Worker 38:
Zone B → Zone A
```

### Robot Fleet Adapter

``` text
REROUTE

AMR-04
AMR-07
AMR-09

avoid: Aisle-14
route: Aisle-16
```

The robot fleet manager---not the LLM---handles navigation, collision
avoidance, and robot safety.

### WMS

The dispatcher may also update:

``` text
Worker assignments
Task priorities
Wave priorities
Operational tasks
```

The Action Dispatcher is the controlled bridge between AI
recommendations and operational execution.

------------------------------------------------------------------------

## 11. Warehouse Responds

Five minutes later:

``` text
Workers 31 + 38 → Picking Zone A

AMR-04 → rerouted
AMR-07 → rerouted
AMR-09 → rerouted
```

The WMS begins reporting increased throughput:

``` text
610/hr
 ↓
720/hr
 ↓
890/hr
 ↓
1,080/hr
```

------------------------------------------------------------------------

## 12. Everything Loops Back

The new events re-enter the same architecture:

``` text
WMS / Labor / Fleet
        ↓
     Adapters
        ↓
Canonical Event Stream
        ↓
 System of Record
        ↓
Analytics & State Engine
```

Analytics sees:

``` text
11:05    610 picks/hr
11:10    720 picks/hr
11:15    890 picks/hr
11:20  1,080 picks/hr
```

Projected completion changes:

``` text
15:40
 ↓
14:52
 ↓
14:11
 ↓
13:47
```

At 11:20:

> **SLA risk cleared.**

------------------------------------------------------------------------

## 13. Feedback Loop

The complete incident is stored:

``` text
EXCEPTION
Picking capacity shortage — Client A

CAUSE
2 workers unavailable
3 AMRs blocked

AI RECOMMENDATION
Move 2 workers B → A
Reroute 3 AMRs

HUMAN DECISION
Approved

PREDICTED RECOVERY
13:48 completion

ACTUAL RECOVERY
13:51 completion

RESULT
SLA saved
0 late orders

Recovery time:
19 minutes
```

This produces the long-term data structure:

``` text
Situation
    ↓
Recommendation
    ↓
Human Decision
    ↓
Executed Action
    ↓
Actual Outcome
```

Over time this history can improve forecasting, action scoring, incident
evaluation, recurring-pattern detection, and future warehouse-specific
models.

The platform should not blindly retrain itself on every incident.
Model/policy improvements should go through controlled evaluation and
deployment.

------------------------------------------------------------------------

# Complete End-to-End Flow

``` text
                    REAL / SIMULATED WAREHOUSE
                              │
               ┌──────────────┼──────────────┐
               │              │              │
              WMS           LABOR       FLEET MANAGER
               │              │              │
               ▼              ▼              ▼
          WMS Adapter    Labor Adapter   Robot Adapter
               │              │              │
               └──────────────┼──────────────┘
                              ▼
                   CANONICAL EVENT STREAM
                              │
                              ▼
                     SYSTEM OF RECORD
                              │
                              ▼
                  ANALYTICS / STATE ENGINE
                              │
                   "Throughput ↓42%"
                   "SLA miss predicted"
                              │
                              ▼
                     EXCEPTION ENGINE
                              │
                  PICKING_CAPACITY_RISK
                              │
                              ▼
                       AI SUPERVISOR
                           [LLM]
                              │
                    Generates candidate
                    recovery strategies
                              │
                              ▼
                  WHAT-IF / OPTIMIZER
                     /       |       \
                    A        B        C
                                     │
                                   ✓ BEST
                                     │
                                     ▼
                         HUMAN SUPERVISOR
                     APPROVE / MODIFY / REJECT
                                     │
                                  APPROVE
                                     │
                                     ▼
                          ACTION DISPATCHER
                         /        |        \
                        /         |         \
                      WMS       LABOR      FLEET
                       │          │          │
                       │       Move 2     Reroute
                       │       workers     3 AMRs
                       │          │          │
                       └──────────┼──────────┘
                                  ▼
                         WAREHOUSE CHANGES
                                  │
                                  ▼
                          NEW OPERATIONAL EVENTS
                                  │
                                  ▼
                              ADAPTERS
                                  │
                                  ▼
                         CANONICAL STREAM
                                  │
                                  ▼
                             ANALYTICS
                                  │
                       Throughput recovers
                                  │
                                  ▼
                         EXCEPTION CLEARED
                                  │
                                  ▼
                           FEEDBACK LOOP
                                  │
               ┌──────────────────┼──────────────────┐
               │                  │                  │
           Prediction           Action            Outcome
               │                  │                  │
               └──────────────────┼──────────────────┘
                                  ▼
                        OPERATIONAL HISTORY
                                  │
                                  ▼
                      MODEL / POLICY IMPROVEMENT
```

------------------------------------------------------------------------

## What Technology Does What?

  -----------------------------------------------------------------------
  Component               Main Technology         Job
  ----------------------- ----------------------- -----------------------
  WMS/Labor/Fleet         Normal code             Connect to vendor APIs
  adapters                                        and translate data

  Canonical event stream  Event infrastructure    Standardized
                                                  operational event
                                                  transport

  System of record        Database/event store    Persist history and
                                                  operational state

  Analytics/state engine  Math + algorithms       Calculate throughput,
                                                  backlog, capacity, ETA,
                                                  SLA risk

  Exception engine        Rules + thresholds      Determine when
                                                  intervention is needed

  AI supervisor           LLM                     Understand context and
                                                  generate recovery
                                                  strategies

  What-if/optimizer       Algorithms + simulation Test strategies and
                                                  quantify tradeoffs

  Human-in-loop           UI/workflow             Approve, modify,
                                                  reject, inspect
                                                  evidence

  Action dispatcher       Controlled API layer    Validate and execute
                                                  approved commands

  Feedback loop           Data + evaluation       Compare prediction,
                          pipeline                action, and actual
                                                  outcome
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# Core Product Loop

The product is not simply an LLM connected to warehouse APIs.

It is a **closed-loop operational system**:

> **Observe → Normalize → Store → Calculate → Detect → Reason → Simulate
> → Human Approves → Execute → Observe Recovery → Evaluate Outcome →
> Improve**

That closed loop is the core architecture of the 3PL AI operations
supervisor.
