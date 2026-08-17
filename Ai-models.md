## AI Model Architecture for the Operational Supervisor

### Model Types (Hybrid Approach)

| Component | Model Type | Purpose |
|-----------|------------|---------|
| **Reasoning / Planning** | Frontier LLM (GPT-4o, Claude 3.5 Sonnet, etc.) via API | Understand context, weigh options, produce structured action plans |
| **Exception Detection** | Lightweight classifiers (XGBoost, small transformers) | Real-time pattern matching on telemetry — "5 robots blocked + wave cutoff in 18 min" |
| **Anomaly / Drift Detection** | Statistical models + isolation forests | Inventory drift, throughput degradation, equipment health trends |
| **Routing / Optimization** | OR-Tools / custom heuristics + LLM-guided search | Forklift dispatch, robot rerouting, zone assignment |
| **Ground-Truth Learning** | Fine-tuned smaller model (Llama 3.1 8B, Phi-3, etc.) | Distill human operator decisions into a specialist model over time |

---

### Data Collection for Training

**What gets collected (append-only event log):**
- Every exception detected (type, severity, context snapshot)
- Every AI recommendation (structured JSON plan + confidence)
- Every human action taken (dispatch, reroute, escalate, defer)
- Every outcome (resolved, escalated, failed, false positive)
- Facility state at decision time (robot poses, inventory, wave status, labor availability)

**What does NOT get collected:**
- Raw PII (worker names, faces — use IDs only)
- Full video streams (only derived events from cameras)
- Customer order data beyond what's needed for priority context

---

### Data Storage

| Data | Storage |
|------|---------|
| **Event log (immutable)** | PostgreSQL (append-only table) + object storage (S3/MinIO) for large context snapshots |
| **Facility model (ZoneGraph, certs, widths)** | PostgreSQL + versioned migrations |
| **Training datasets** | Parquet files in object storage, partitioned by facility + time |
| **Model artifacts** | MLflow / W&B / S3 with versioning |
| **Inference cache** | Redis (recent decisions for deduplication) |

---

### Deployment Model

**Phase 1 (Beachhead): API-only — no model deployment needed**
- Call frontier models (OpenAI, Anthropic) via HTTPS
- Zero ML ops burden, best reasoning quality, pay-per-token
- All prompt engineering, few-shot examples, and tool definitions in your codebase

**Phase 2 (Scale + Privacy): Hybrid**
- Frontier LLM for complex reasoning (rare, high-stakes decisions)
- Self-hosted fine-tuned 8B model for high-volume exception classification + routine planning
- Run on GPU inference server (vLLM / TGI) in customer VPC or your cloud

**Phase 3 (Maturity): Fully self-hosted**
- Distilled specialist model handles 95%+ of decisions
- Frontier model only for novel/ambiguous cases (human-escalation tier)

---

### RAG? Yes — but "Operational RAG"

Not document retrieval. **Context retrieval:**

| Query | Retrieved Context |
|-------|-------------------|
| "AMR congestion in Zone B" | ZoneGraph alt routes, forklift certifications, current robot poses, active wave priorities, similar past exceptions + outcomes |
| "Inventory drift SKU-12345" | Recent cycle counts, pick paths, putaway history, replenishment rules, last 50 drift resolutions |

Implementation: Vector store (pgvector / Qdrant) for semantic search over past exceptions + structured SQL for real-time facility state. Both injected into LLM prompt as "context pack."

---

### Can you use ChatGPT? 

**Yes, via API — not ChatGPT the consumer product.**
- Use  /  /  via OpenAI API (or Anthropic / Google / Azure equivalents)
- Data processing agreements (DPAs) + zero-retention endpoints available for enterprise
- **Do not** paste warehouse data into chat.openai.com

---

### Summary Recommendation

| Stage | Strategy |
|-------|----------|
| **MVP (0-3 customers)** | 100% API (GPT-4o / Claude Sonnet). Prompt engineering + few-shot + structured output. Collect event log religiously. |
| **Product-Market Fit (3-10 customers)** | Add self-hosted classifier for exception detection (high volume, low latency). Keep planner on API. |
| **Scale (10+ customers, data moat)** | Fine-tune 8B specialist on your event log. Route 80%+ of decisions to it. API for edge cases. |

The event log you build from Day 1 is your future training data — design its schema carefully (facility_id, exception_type, context_json, recommendation_json, human_action, outcome, timestamp).