# 3PL AI Supervisor --- Layers 3, 5, 6, and 7

## Core Flow

**3. Normalize Data → 5. Calculate Warehouse State → 6. Detect Problems
→ 7. LLM Reasons About What to Do**

Layers 3, 5, and 6 should mostly **not** use a large language model.
They convert raw warehouse data into a reliable, structured operational
problem.

## 3. Canonical Event Stream

### Job

Different warehouse systems may represent the same event differently.
The adapter translates vendor-specific data into one internal format.

Example source A:

``` json
{"pick_id":8273,"state":"completed","completed_at":"10:31:22"}
```

Example source B:

``` json
{"taskNumber":8273,"status":"DONE","timestamp":"10:31:22"}
```

Canonical event:

``` json
{
  "event_type":"PICK_TASK_COMPLETED",
  "task_id":"8273",
  "timestamp":"10:31:22",
  "source":"WMS"
}
```

Everything downstream now understands the same schema regardless of the
WMS or other vendor.

### Technology

**No ML or LLM is normally needed.** Use standard software engineering:

-   API integrations and webhooks
-   Schema validation
-   Data transformation
-   Event queues/streams
-   Kafka, NATS, or Redis Streams
-   Python, TypeScript, or Go

Think of Layer 3 as the **translator and highway for warehouse events**.

## 5. Analytics & State Engine

### Job

Layer 5 answers:

> **What is happening in the warehouse right now?**

Raw events such as `PICK_COMPLETED` are accumulated into operational
state:

``` text
Zone A
Workers: 7
Remaining picks: 3,300
Picks completed last hour: 450
Current throughput: 450 picks/hour
Cutoff: 2:00 PM
```

It then calculates:

``` text
Required rate = remaining work / time remaining
3,300 / 4 hours = 825 picks/hour
```

The current 450 picks/hour is far below the required 825 picks/hour.

### Metrics

Layer 5 can calculate:

-   Picks/hour and packs/hour
-   Backlog and queue size
-   Backlog growth rate
-   Worker and equipment utilization
-   Current and required capacity
-   Expected completion time
-   Required throughput
-   Inventory depletion rate
-   Replenishment demand
-   Dock utilization
-   Robot utilization
-   Equipment throughput
-   Expected SLA completion

### Algorithms

For the MVP, use deterministic math and algorithms:

``` python
pick_rate = completed_picks / elapsed_time
required_rate = remaining_picks / time_until_cutoff
utilization = productive_time / available_time
backlog_growth = incoming_work_rate - completion_rate
```

Other useful methods include moving averages, rolling windows,
percentiles, queueing models, scheduling algorithms, optimization
algorithms, and graph algorithms.

### ML Later

This is a strong place for predictive ML later. Instead of estimating
completion time only as `remaining_orders / current_rate`, a model could
use order mix, SKU count, worker count, historical speeds, congestion,
time of day, replenishment state, travel distance, and equipment status.

For the MVP, **start deterministic**.

## 6. Exception Engine

### Job

Layer 6 answers:

> **Is something important going wrong?**

Suppose Layer 5 reports:

``` text
Zone A
Current rate: 450/hr
Required rate: 825/hr
Predicted completion: 3:18 PM
Cutoff: 2:00 PM
```

The exception engine applies deterministic rules.

``` python
if predicted_completion > cutoff:
    create_exception("SLA_RISK")
```

Or:

``` python
if current_pick_rate < expected_pick_rate * 0.75:
    create_exception("PICKING_SLOWDOWN")
```

A more robust condition might be:

``` text
IF throughput is >25% below baseline
AND it lasts >10 minutes
AND backlog is increasing
AND predicted SLA violation >15 minutes
THEN create a HIGH-priority exception
```

Output:

``` json
{
  "exception":"PICKING_ZONE_BEHIND",
  "zone":"A",
  "severity":"HIGH",
  "customer":"Client A",
  "predicted_late_orders":340,
  "predicted_delay_minutes":78
}
```

### Algorithms or ML?

For the MVP, use:

-   Rules
-   Thresholds
-   State machines
-   Rolling-window calculations
-   Deterministic anomaly conditions

This makes detection explainable and testable.

Later, ML can add anomaly detection, learned baselines, failure
prediction, dynamic thresholds, and root-cause probability models.

## 7. AI Supervisor

This is where a **large language model becomes useful**.

Instead of feeding the LLM thousands of raw events, Layer 6 gives it a
structured context package:

``` text
EXCEPTION
Client A picking SLA at risk.

Zone A:
7 workers
450 picks/hr
3,300 remaining
Cutoff: 14:00
Predicted finish: 15:18

Zone B:
8 workers
710 picks/hr
900 remaining
Cutoff: 17:00

AVAILABLE RESOURCES
Workers 14, 19, and 22 are cross-trained.
Replenishment worker 7 is available.
```

The LLM reasons about the operational response, for example:

> Zone B has excess capacity and a later deadline. Consider moving two
> qualified workers from Zone B to Zone A and prioritizing replenishment
> for Client A's critical SKUs.

The LLM is therefore used mainly for **reasoning about what to do**, not
basic arithmetic.

## Optimization / What-If Engine

The LLM should not necessarily have the final word. Its candidate
actions can be tested by simulation or optimization.

``` text
OPTION A
Move 2 workers B → A
Predicted Client A SLA: 97.8%

OPTION B
Move 4 workers B → A
Predicted Client A SLA: 100%
Client B becomes at risk

OPTION C
Move 2 workers B → A
+ prioritize replenishment
Predicted Client A SLA: 100%
Predicted Client B SLA: 100%
Lowest total disruption
```

The platform can recommend Option C.

## Intelligence Stack

``` text
RAW DATA
   ↓
3. CANONICAL EVENT STREAM
   Normal software — no AI
   ↓
5. ANALYTICS & STATE ENGINE
   Math + algorithms — predictive ML later
   ↓
6. EXCEPTION ENGINE
   Rules + thresholds — anomaly ML later
   ↓
PROBLEM DETECTED
   ↓
7. AI SUPERVISOR
   LLM — understand WHY and generate WHAT TO DO
   ↓
OPTIMIZER / SIMULATOR
   Test candidate actions and score consequences
   ↓
BEST ACTION
```

## Recommended Technology by Layer

  -----------------------------------------------------------------------
  Layer                   MVP                     Later
  ----------------------- ----------------------- -----------------------
  **3 --- Event Stream**  Regular software        Regular software

  **5 --- Analytics**     Math + deterministic    Algorithms + predictive
                          algorithms              ML

  **6 --- Exception       Rules + thresholds      Rules +
  Engine**                                        anomaly-detection ML

  **7 --- AI Supervisor** LLM                     LLM +
                                                  warehouse-specific
                                                  models/tools

  **Action Selection**    Simulation +            Advanced optimization /
                          optimization            learned policies
  -----------------------------------------------------------------------

## Design Principle

> **Do not use AI where normal software can produce a reliable, correct
> answer.**

Use deterministic algorithms to determine:

> Zone A is 37% below required throughput.

Use predictive ML when useful to estimate:

> There is an 87% probability Client A will miss its cutoff.

Use an LLM to reason:

> Given the warehouse situation, constraints, resources, and customer
> priorities, what recovery strategies make sense?

Then use an optimizer or simulator to verify:

> Which proposed action produces the lowest total operational impact?

The resulting architecture combines **traditional software for
reliability, ML for prediction, an LLM for flexible reasoning, and
optimization/simulation for decision verification**.
