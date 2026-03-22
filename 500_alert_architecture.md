# 500 Alert Automation with n8n — Detailed Architecture Document

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Problem Statement](#2-problem-statement)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Component Deep-Dive](#4-component-deep-dive)
   - 4.1 [Data Sources & CDR Events](#41-data-sources--cdr-events)
   - 4.2 [Apache Kafka — Event Streaming](#42-apache-kafka--event-streaming)
   - 4.3 [Threshold Detector & Context Enrichment](#43-threshold-detector--context-enrichment)
   - 4.4 [AI Decision Agent — LLaMA 3.1 70B](#44-ai-decision-agent--llama-31-70b)
   - 4.5 [vLLM Inference Server](#45-vllm-inference-server)
   - 4.6 [GPU Infrastructure — 4× NVIDIA A100 80GB](#46-gpu-infrastructure--4-nvidia-a100-80gb)
   - 4.7 [n8n Workflow Orchestration](#47-n8n-workflow-orchestration)
   - 4.8 [SIM Management API](#48-sim-management-api)
   - 4.9 [Notification Layer](#49-notification-layer)
   - 4.10 [Audit & Data Storage](#410-audit--data-storage)
   - 4.11 [Observability Stack](#411-observability-stack)
5. [Data Flow — End to End](#5-data-flow--end-to-end)
6. [AI Decision Logic](#6-ai-decision-logic)
   - 6.1 [Prompt Structure](#61-prompt-structure)
   - 6.2 [Decision Types & Routing](#62-decision-types--routing)
   - 6.3 [Confidence Scoring](#63-confidence-scoring)
   - 6.4 [Example Scenarios](#64-example-scenarios)
7. [n8n Workflows — Detailed](#7-n8n-workflows--detailed)
   - 7.1 [AUTO_BLOCK](#71-auto_block)
   - 7.2 [ALLOW_EXCEPTION](#72-allow_exception)
   - 7.3 [ESCALATE_TO_HUMAN](#73-escalate_to_human)
   - 7.4 [BLOCK_WITH_GRACE](#74-block_with_grace)
8. [LLaMA 3.1 70B — Model Details](#8-llama-31-70b--model-details)
   - 8.1 [Architecture Internals](#81-architecture-internals)
   - 8.2 [Token Embeddings](#82-token-embeddings)
   - 8.3 [Training Pipeline](#83-training-pipeline)
   - 8.4 [RLHF & Alignment](#84-rlhf--alignment)
   - 8.5 [Key Configuration Parameters](#85-key-configuration-parameters)
9. [vLLM & GPU Setup](#9-vllm--gpu-setup)
   - 9.1 [Why vLLM](#91-why-vllm)
   - 9.2 [PagedAttention](#92-pagedattention)
   - 9.3 [Tensor Parallelism](#93-tensor-parallelism)
   - 9.4 [GPU Memory Layout](#94-gpu-memory-layout)
10. [Security & Compliance](#10-security--compliance)
11. [Failure Modes & Fallbacks](#11-failure-modes--fallbacks)
12. [Monitoring & Alerting](#12-monitoring--alerting)
13. [Deployment](#13-deployment)
14. [Performance Metrics](#14-performance-metrics)
15. [Potential Enhancements](#15-potential-enhancements)

---

## 1. System Overview

The 500 Alert Automation system is an **AI-powered telecom usage monitoring platform** that automatically detects when a subscriber's data charges exceed $500 CAD and makes an intelligent, context-aware decision about whether to block the SIM card, grant an exception, escalate to a human reviewer, or issue a grace period warning.

The system replaces a manual, report-driven process that previously took 48–72 hours to detect overages with a real-time pipeline that makes a decision in under 30 seconds.

**Core principle:** Rather than a hard threshold rule ("always block at $500"), the system enriches each event with full customer context — payment history, account tier, usage patterns, support tickets — and uses a large language model to make a nuanced, human-like decision at machine speed.

---

## 2. Problem Statement

| Pain Point | Before | After |
|---|---|---|
| Detection latency | 48–72 hours | < 30 seconds |
| Manual review hours | 8 hrs/day | 2 hrs/day |
| Fraud detection rate | 45% | 87% |
| VIP customer satisfaction | 6.2 / 10 | 8.9 / 10 |
| Operational cost | $180,000/year | $65,000/year |
| Decision consistency | Variable by operator | AI-enforced uniformity |

Traditional approaches failed because:

- Rigid threshold rules blocked legitimate VIP and business accounts
- No intelligent handling of roaming, corporate plans, or usage anomalies
- Human operators were inconsistent across shifts
- No real-time signal — only batch report checks

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          DATA SOURCES                                   │
│   CDR Events  |  Billing API  |  CRM System  |  Customer Profile DB     │
└────────────────────────────┬────────────────────────────────────────────┘
                             │ usage events (real-time)
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        APACHE KAFKA                                     │
│                   Topic: usage-events                                   │
│              Durable · Ordered · Replayable                             │
└────────────────────────────┬────────────────────────────────────────────┘
                             │ consumed by detector
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              THRESHOLD DETECTOR + CONTEXT ENRICHMENT                   │
│                        (Python 3.11)                                    │
│   if charges_cad >= 500.0:                                              │
│       fetch CRM + Billing + Support + Payment history                   │
│       assemble enriched prompt                                          │
└────────────────────────────┬────────────────────────────────────────────┘
                             │ enriched prompt (HTTP POST)
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    vLLM INFERENCE SERVER                                 │
│              PagedAttention · Continuous Batching                        │
│         ┌─────────────────┐   ┌─────────────────┐                      │
│         │  Replica 1      │   │  Replica 2      │                      │
│         │  GPU 0 + GPU 1  │   │  GPU 2 + GPU 3  │                      │
│         │  80 GB + 80 GB  │   │  80 GB + 80 GB  │                      │
│         └─────────────────┘   └─────────────────┘                      │
│                 LLaMA 3.1 70B Instruct (140 GB fp16)                    │
└────────────────────────────┬────────────────────────────────────────────┘
                             │ JSON decision
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                       AI DECISION OUTPUT                                 │
│  { decision, reasoning, confidence, risk_factors, exception_factors }    │
└─────┬──────────────┬────────────────┬────────────────────┬──────────────┘
      │              │                │                    │
      ▼              ▼                ▼                    ▼
  BLOCK       ALLOW_EXCEPTION   ESCALATE_TO_HUMAN   BLOCK_WITH_GRACE
      │              │                │                    │
      └──────────────┴────────────────┴────────────────────┘
                             │ webhook POST
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    n8n WORKFLOW ORCHESTRATION                           │
│     4 separate webhook workflows, bearer token auth                     │
│     SIM API  ·  SMS  ·  Slack  ·  Email  ·  ServiceNow                  │
└────────────────────────────┬────────────────────────────────────────────┘
                             │ audit write
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              AUDIT & OBSERVABILITY                                      │
│        BigQuery (decisions)  ·  PostgreSQL (customers)                  │
│        Prometheus (metrics)  ·  Grafana (dashboards)                    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Component Deep-Dive

### 4.1 Data Sources & CDR Events

**Call Detail Records (CDRs)** are the raw usage events emitted by the telecom billing system each time a subscriber consumes data. Each event carries:

```json
{
  "subscriber_id": "SUB-12345",
  "phone_number": "+1-555-123-4567",
  "data_usage_mb": 52345,
  "charges_cad": 523.45,
  "timestamp": "2024-11-15T14:23:00Z"
}
```

Additional data pulled during enrichment from separate APIs:

| Source | Fields Fetched |
|---|---|
| CRM API | customer tier, account age, account type (personal/business) |
| Billing API | average monthly bill, outstanding balance, billing cycle |
| Payment history | payment score (0.0–1.0), late payments, defaults |
| Support API | open tickets count, complaints last 90 days, last ticket reason |
| Usage history | last 3 months average MB, deviation from average |

---

### 4.2 Apache Kafka — Event Streaming

**Role:** Decouples the telecom billing system from the detection system. Events are persisted to disk and consumed at the detector's pace.

**Configuration:**

| Parameter | Value |
|---|---|
| Broker address | `kafka:9092` |
| Topic | `usage-events` |
| Retention | 7 days |
| Partitions | 12 (allows parallel consumers) |
| Replication factor | 3 (fault tolerance) |

**Why Kafka over direct webhooks:**

- If the detector restarts, events queue up safely rather than being dropped
- Multiple detector instances can be added; Kafka auto-balances partitions
- Historical events can be replayed for debugging or re-processing
- The telecom billing system has zero knowledge of the AI pipeline — clean decoupling

---

### 4.3 Threshold Detector & Context Enrichment

**Language:** Python 3.11

**Trigger condition:**
```python
BLOCK_THRESHOLD_CAD = 500.0

if event["charges_cad"] >= BLOCK_THRESHOLD_CAD:
    context = enrich_customer_context(event["subscriber_id"])
    prompt = build_prompt(event, context)
    decision = query_llama(prompt)
    route_to_n8n(decision)
```

**Enrichment flow:**

```
CDR event received
        │
        ├── GET /crm/{subscriber_id}          → tier, account_age, account_type
        ├── GET /billing/{subscriber_id}       → avg_bill, outstanding_balance
        ├── GET /payments/{subscriber_id}      → payment_score, late_count
        ├── GET /support/{subscriber_id}       → tickets, complaints
        └── GET /usage-history/{subscriber_id} → avg_mb, deviation_%
                │
                ▼
        Assembled into structured prompt
        Pre-compute deviation: (current_mb - avg_mb) / avg_mb × 100
        (arithmetic offloaded to Python — LLMs unreliable at maths)
```

**Fallback behaviour:** If any enrichment API is unavailable, the field defaults to `null` and the prompt instructs LLaMA to escalate rather than decide when key fields are missing.

---

### 4.4 AI Decision Agent — LLaMA 3.1 70B

**Role:** Receives the enriched prompt and produces a structured JSON decision with reasoning, confidence score, and enumerated risk/exception factors.

**Model:** `meta-llama/Llama-3.1-70B-Instruct`

**Why this model:**
- 70B parameters provide strong multi-step reasoning — necessary for weighing competing signals (VIP status vs. poor payment history)
- The Instruct variant is fine-tuned via RLHF to follow structured instructions and output valid JSON reliably
- Open weights — no API cost, no per-token billing at scale, no data leaving the infrastructure
- Fits within 4× A100 80GB GPU setup (2 GPUs per replica × 2 replicas)

---

### 4.5 vLLM Inference Server

**Role:** Serves LLaMA 3.1 70B over an OpenAI-compatible HTTP API. Manages GPU memory, request queuing, batching, and multi-GPU coordination.

**Key capabilities:**

| Feature | What it does |
|---|---|
| PagedAttention | Manages KV cache in fixed-size pages — no wasted pre-allocated memory |
| Continuous batching | New requests slot into the running batch as soon as a slot frees up |
| Tensor parallelism | Splits model weights across multiple GPUs seamlessly |
| OpenAI-compatible API | Python agent calls `/v1/completions` exactly like it would call OpenAI |

**Configuration:**
```yaml
MODEL: meta-llama/Llama-3.1-70B-Instruct
TENSOR_PARALLEL_SIZE: 2      # split each replica across 2 GPUs
GPU_MEMORY_UTILIZATION: 0.9  # 90% of VRAM reserved for KV cache
MAX_MODEL_LEN: 4096           # max prompt + output tokens
```

**Endpoint used by the detector:**
```
POST http://vllm:8000/v1/completions
Authorization: Bearer <token>
Content-Type: application/json
```

---

### 4.6 GPU Infrastructure — 4× NVIDIA A100 80GB

**Why GPUs:** Running 70B model inference on a CPU would take 10–20 minutes per request. A100 GPUs with HBM2e memory (2 TB/s bandwidth) reduce this to 5–15 seconds.

**Why 4 GPUs specifically:**

```
LLaMA 3.1 70B in fp16 ≈ 140 GB VRAM required

1× A100 (80 GB)  → does not fit
2× A100 (160 GB) → fits, but only 1 request at a time
4× A100 (320 GB) → 2 replicas running in parallel
                   = 2 simultaneous $500 decisions
```

**Physical layout:**

```
┌─────────────────────────────────────────────────────────┐
│                    Server Node                          │
│                                                         │
│  ┌─────────────┐   NVLink   ┌─────────────┐           │
│  │   GPU 0     │◄──────────►│   GPU 1     │  Replica 1 │
│  │   80 GB     │            │   80 GB     │           │
│  └─────────────┘            └─────────────┘           │
│                                                         │
│  ┌─────────────┐   NVLink   ┌─────────────┐           │
│  │   GPU 2     │◄──────────►│   GPU 3     │  Replica 2 │
│  │   80 GB     │            │   80 GB     │           │
│  └─────────────┘            └─────────────┘           │
│                                                         │
│  System RAM: 256 GB+    |    NVMe SSD: model weights   │
└─────────────────────────────────────────────────────────┘
```

**NVLink** is a high-bandwidth direct GPU-to-GPU interconnect (up to 600 GB/s) that allows the two GPUs in each replica to share tensor computations at near-memory speeds during each forward pass.

---

### 4.7 n8n Workflow Orchestration

**Role:** Executes the decision produced by the AI agent. Handles all external side effects — API calls, notifications, logging — without any changes needed to the Python codebase.

**Why n8n over hardcoded Python:**
- Visual no-code editor — ops team can modify notification channels or timing without a deployment
- Each decision type is an independent webhook endpoint, cleanly separated
- Built-in retry logic, error handling, and execution history
- Bearer token authentication on all webhooks

**Authentication:**
```
Authorization: Bearer ${N8N_TOKEN}
```

All four workflow webhooks share the same token. The token is rotated via environment variable without code changes.

---

### 4.8 SIM Management API

**Role:** The telecom carrier's internal API that physically blocks or modifies SIM card service.

**Endpoints called by n8n:**

| Action | Endpoint | Used by |
|---|---|---|
| Full block | `POST /sim/{id}/block` | AUTO_BLOCK, BLOCK_WITH_GRACE (after delay) |
| Temporary hold | `POST /sim/{id}/hold?duration=15m` | ESCALATE_TO_HUMAN |
| Monitor flag | `POST /sim/{id}/monitor?duration=24h` | ALLOW_EXCEPTION |

**Authentication:** Rotating API key injected via environment variable `SIM_API_KEY`.

---

### 4.9 Notification Layer

Three notification channels, each triggered by different workflows:

| Channel | Trigger | Content |
|---|---|---|
| SMS to customer | All four workflows | Decision-appropriate message (warning, confirmation, courtesy notice) |
| Slack `#auto-block` | AUTO_BLOCK, BLOCK_WITH_GRACE | Block details, subscriber ID, AI reasoning summary |
| Slack `#exceptions` | ALLOW_EXCEPTION | Exception granted, monitoring period started |
| Slack `@channel` | ESCALATE_TO_HUMAN | Urgent alert, full context, link to case |
| Email to ops manager | ESCALATE_TO_HUMAN | Full JSON context, AI reasoning, suggested action |
| ServiceNow ticket | ESCALATE_TO_HUMAN | P2 priority, auto-assigned to on-call team |

---

### 4.10 Audit & Data Storage

**BigQuery** — append-only audit log of every AI decision:

```sql
CREATE TABLE ai_decisions.decisions (
  timestamp         TIMESTAMP,
  subscriber_id     STRING,
  phone_number      STRING,
  charges_cad       FLOAT64,
  decision          STRING,   -- BLOCK / ALLOW_EXCEPTION / etc.
  reasoning         STRING,   -- full AI reasoning text
  confidence        FLOAT64,
  risk_factors      ARRAY<STRING>,
  exception_factors ARRAY<STRING>,
  n8n_workflow_id   STRING,
  execution_ms      INT64
);
```

**PostgreSQL** — operational customer data:

- Customer profiles (CRM data)
- Billing records
- Payment history
- Support ticket history

PostgreSQL is the source of truth for enrichment API calls. BigQuery is write-only from the perspective of the AI pipeline — no decisions are ever modified, only appended.

---

### 4.11 Observability Stack

**Prometheus** scrapes metrics from the detector service every 15 seconds.

**Key metrics exported:**

| Metric | Type | Description |
|---|---|---|
| `usage_events_processed_total` | Counter | Total CDR events consumed from Kafka |
| `usage_events_above_threshold_total` | Counter | Events that triggered the AI agent |
| `ai_decisions_total{decision="..."}` | Counter | Decisions broken down by type |
| `ai_decision_latency_seconds` | Histogram | Time from enrichment start to decision |
| `n8n_workflow_success_rate` | Gauge | Workflow execution success percentage |
| `vllm_gpu_utilization` | Gauge | GPU utilisation per device |
| `enrichment_api_latency_seconds{api="..."}` | Histogram | Latency per enrichment API call |

**Grafana** — four pre-built dashboards:

1. **AI Decision Overview** — decision type breakdown, rates over time, confidence distribution
2. **System Performance** — Kafka lag, AI latency percentiles, GPU utilisation
3. **Customer Impact** — blocks per hour, exceptions granted, escalation volume
4. **Error Tracking** — failed API calls, JSON parse errors, workflow failures

**Prometheus alert rules:**

```yaml
groups:
  - name: ai_usage_blocker
    rules:
      - alert: HighAILatency
        expr: ai_decision_latency_seconds > 5
        for: 5m
        annotations:
          summary: "AI decision latency exceeding 5 seconds"
          action: "Check vLLM logs and GPU memory utilisation"

      - alert: LowAIConfidence
        expr: avg(ai_decision_confidence) < 0.7
        for: 10m
        annotations:
          summary: "Average AI confidence below 70%"
          action: "Check enrichment API data quality"

      - alert: HighEscalationRate
        expr: rate(ai_decisions_total{decision="ESCALATE_TO_HUMAN"}[1h]) > 0.1
        for: 0m
        annotations:
          summary: "More than 10% of decisions being escalated"
          action: "Investigate unusual usage pattern or enrichment data issue"

      - alert: KafkaConsumerLag
        expr: kafka_consumer_lag > 1000
        for: 5m
        annotations:
          summary: "Detector falling behind event stream"
```

---

## 5. Data Flow — End to End

```
1. Subscriber uses data
        │
        ▼
2. Telecom billing system emits CDR event → Kafka topic: usage-events
        │
        ▼
3. Python detector consumes event
   → checks: charges_cad >= 500.0?
   → NO  → discard (no action)
   → YES → proceed
        │
        ▼
4. Context enrichment (parallel API calls, ~200–500ms total)
   → CRM: tier=VIP, age=24mo, type=BUSINESS
   → Billing: avg_bill=450, outstanding=0
   → Payments: score=0.95
   → Support: tickets=0, complaints=0
   → Usage: avg_mb=48200, deviation=+8.6%
        │
        ▼
5. Prompt assembly
   → Structured text with === section headers
   → Pre-computed deviation %
   → Null fields flagged explicitly
        │
        ▼
6. HTTP POST to vLLM /v1/completions
   → vLLM scheduler tokenises prompt
   → Assigns to free replica (GPU 0+1 or GPU 2+3)
   → 80-layer forward pass executes
   → Token-by-token generation (temperature=0.1)
   → Returns when EOS token generated or max_tokens reached
        │
        ▼
7. Python agent receives response
   → json.loads() parses output
   → Validates schema (decision field present and valid)
   → On parse failure: retry once, then default to ESCALATE_TO_HUMAN
        │
        ▼
8. Route to n8n webhook based on decision
   BLOCK              → POST /webhook/auto-block
   ALLOW_EXCEPTION    → POST /webhook/allow-exception
   ESCALATE_TO_HUMAN  → POST /webhook/escalate-human
   BLOCK_WITH_GRACE   → POST /webhook/block-grace-period
        │
        ▼
9. n8n workflow executes
   → Log to BigQuery
   → Call SIM Management API
   → Send SMS / Slack / Email / ServiceNow
        │
        ▼
10. Prometheus metrics updated
    Grafana dashboard reflects new decision
```

**Total wall time:** 5–30 seconds (dominated by vLLM inference latency)

---

## 6. AI Decision Logic

### 6.1 Prompt Structure

The prompt sent to LLaMA has two parts:

**System prompt** (sets role, defines output schema, enforces JSON-only output):
```
You are a telecom fraud and usage management agent. Your task is to
decide whether to block a SIM card that has exceeded $500 CAD in
data charges.

You must weigh risk factors against exception factors and output a
JSON decision.

Decision types:
- BLOCK: clear fraud or abuse signal, low risk of false positive
- ALLOW_EXCEPTION: legitimate high usage, VIP or business context
- ESCALATE_TO_HUMAN: conflicting signals, confidence below threshold
- BLOCK_WITH_GRACE: borderline case, customer deserves warning first

Always respond with ONLY valid JSON. No preamble. No explanation
outside the JSON.
```

**User prompt** (assembled dynamically from enrichment data):
```
=== USAGE EVENT ===
subscriber_id: SUB-12345
phone: +1-555-123-4567
charges_cad: 523.45
data_usage_mb: 52345
timestamp: 2024-11-15T14:23:00Z

=== CUSTOMER PROFILE ===
tier: VIP
account_age_months: 24
account_type: BUSINESS
payment_score: 0.95
avg_monthly_bill_cad: 450.00
outstanding_balance_cad: 0.00

=== USAGE HISTORY ===
last_3_months_avg_mb: 48200
current_mb: 52345
deviation_from_avg: +8.6%

=== SUPPORT CONTEXT ===
open_tickets: 0
complaints_last_90d: 0
last_ticket_reason: null

=== ACCOUNT NOTES ===
"Corporate account - field sales team, frequent roaming"

Evaluate and respond with JSON.
```

### 6.2 Decision Types & Routing

| Decision | Condition | n8n Webhook | SIM Action |
|---|---|---|---|
| `BLOCK` | High confidence fraud/abuse signal | `/auto-block` | Immediate block |
| `ALLOW_EXCEPTION` | Legitimate high usage, VIP/business | `/allow-exception` | 24h monitoring flag |
| `ESCALATE_TO_HUMAN` | Conflicting signals, low confidence | `/escalate-human` | 15-minute temporary hold |
| `BLOCK_WITH_GRACE` | Borderline — customer deserves warning | `/block-grace-period` | SMS warning, block after 1 hour |

### 6.3 Confidence Scoring

The confidence field (0.0–1.0) is the model's self-reported certainty. It is used as a routing gate:

- Confidence ≥ 0.80 → proceed with BLOCK or ALLOW_EXCEPTION as decided
- Confidence 0.60–0.79 → system considers BLOCK_WITH_GRACE as a safer option
- Confidence < 0.60 → auto-route to ESCALATE_TO_HUMAN regardless of primary decision

> **Important:** Confidence is not a calibrated probability. It is the model's subjective certainty. Over time, human override rates tracked in BigQuery provide ground truth for model calibration.

### 6.4 Example Scenarios

**Scenario A — VIP business customer, minor overage:**
```json
{
  "decision": "ALLOW_EXCEPTION",
  "reasoning": "VIP business account with 24 months tenure and excellent payment score of 0.95. Current overage of $23.45 is 5.2% above threshold and within expected variance for a high-tier corporate account with consistent usage history.",
  "confidence": 0.92,
  "risk_factors": ["charges_exceed_threshold_by_$23"],
  "exception_factors": ["vip_customer", "excellent_payment_history", "business_account", "usage_consistent_with_history"]
}
```

**Scenario B — New basic account, 8.5× usage spike:**
```json
{
  "decision": "BLOCK",
  "reasoning": "New customer (2 months) with borderline payment score (0.65) showing dramatic usage spike of 8.5× average monthly consumption. Current charges of $847 represent $347 overage. Pattern is consistent with SIM compromise or unauthorised resale.",
  "confidence": 0.88,
  "risk_factors": ["new_account_high_spike", "charges_8.5x_above_average", "borderline_payment_score"],
  "exception_factors": []
}
```

**Scenario C — VIP account, conflicting signals:**
```json
{
  "decision": "ESCALATE_TO_HUMAN",
  "reasoning": "VIP status suggests exception warranted, but poor payment score (0.58), existing outstanding balance ($145), very large overage ($747), and 3 recent complaints create significant uncertainty. Human judgment required to weigh these conflicting signals.",
  "confidence": 0.45,
  "risk_factors": ["very_high_overage_$747", "poor_payment_score", "outstanding_balance", "recent_complaints"],
  "exception_factors": ["vip_customer_status"]
}
```

---

## 7. n8n Workflows — Detailed

### 7.1 AUTO_BLOCK

**Webhook:** `POST /webhook/auto-block`

**Trigger payload:**
```json
{
  "subscriber_id": "SUB-12345",
  "phone_number": "+1-555-123-4567",
  "charges_cad": 523.45,
  "decision": "BLOCK",
  "reasoning": "...",
  "confidence": 0.88
}
```

**Steps:**

1. Validate bearer token
2. Log full decision payload to BigQuery `ai_decisions.decisions`
3. `POST /sim/{subscriber_id}/block` → SIM Management API
4. Send SMS to `phone_number`: "Your data service has been temporarily suspended due to unusual usage. Contact support to restore."
5. Post to Slack `#auto-block` channel with subscriber ID, charges, and AI reasoning
6. Write completion log entry to BigQuery

**Success rate:** 99.5%

---

### 7.2 ALLOW_EXCEPTION

**Webhook:** `POST /webhook/allow-exception`

**Steps:**

1. Validate bearer token
2. Log exception grant to BigQuery
3. Send courtesy SMS: "We noticed high data usage on your account. No action needed — your account is in good standing."
4. `POST /sim/{subscriber_id}/monitor?duration=24h` → flags account for elevated monitoring
5. Post to Slack `#exceptions` with exception reason summary
6. Log completion

---

### 7.3 ESCALATE_TO_HUMAN

**Webhook:** `POST /webhook/escalate-human`

**Steps:**

1. Validate bearer token
2. `POST /sim/{subscriber_id}/hold?duration=15m` → temporary SIM hold (buys time for review)
3. Create ServiceNow ticket: Priority P2, Category "Usage Anomaly", assigned to on-call team
4. Post to Slack `#escalations` with `@channel` mention, full context, and direct ticket link
5. Send email to operations manager with complete JSON payload and AI reasoning
6. Log escalation to BigQuery
7. If no human action taken within 15 minutes → auto-extend hold or escalate to P1

**Average human resolution time:** 12 minutes

---

### 7.4 BLOCK_WITH_GRACE

**Webhook:** `POST /webhook/block-grace-period`

**Steps:**

1. Validate bearer token
2. Send immediate SMS: "Warning: Your data charges have reached $500 CAD. Service will be suspended in 1 hour unless you contact support."
3. Log warning issued to BigQuery
4. **n8n delay node: wait 60 minutes**
5. Check if human override was issued (query BigQuery for cancellation flag)
6. If no override: `POST /sim/{subscriber_id}/block` → SIM Management API
7. Send confirmation SMS: "Your data service has been suspended. Contact support to discuss your account."
8. Post to Slack `#auto-block`
9. Log completion to BigQuery

---

## 8. LLaMA 3.1 70B — Model Details

### 8.1 Architecture Internals

LLaMA 3.1 70B is a decoder-only transformer with the following specifications:

| Parameter | Value |
|---|---|
| Total parameters | 70 billion |
| Transformer layers | 80 |
| Model dimension | 8,192 |
| Query heads | 64 |
| Key-Value heads | 8 (Grouped-Query Attention) |
| Head dimension | 128 |
| FFN hidden dimension | 28,672 |
| Vocabulary size | 128,000 tokens |
| Context window | 128,000 tokens |
| Activation function | SwiGLU |
| Positional encoding | RoPE (Rotary Position Embedding) |
| Normalisation | RMSNorm |
| VRAM required (fp16) | ~140 GB |

**Grouped-Query Attention (GQA):** 64 query heads share 8 key-value heads. This dramatically reduces KV cache memory per request, enabling higher concurrency without proportionally larger VRAM.

### 8.2 Token Embeddings

Every forward pass begins with the **token embedding layer** — a lookup table that converts each token ID into a vector of 8,192 floating-point numbers. These vectors encode the learned meaning and relationships between words from pre-training.

This is an internal mechanism of LLaMA and is invisible to the Python agent. The agent sends raw text; LLaMA tokenises it and runs it through the embedding lookup internally.

> This project does NOT use standalone embedding models (e.g. for semantic search or retrieval). There is no vector database. All reasoning is done by LLaMA directly from the structured text prompt.

### 8.3 Training Pipeline

LLaMA 3.1 70B was trained by Meta in three phases:

**Phase 1 — Pre-training:**
- Dataset: ~15 trillion tokens from web text, code, books, papers, Wikipedia
- Objective: next-token prediction (cross-entropy loss)
- Compute: ~16,000 NVIDIA H100 GPUs for approximately 3–4 months
- Estimated total compute: ~6 × 10²⁴ FLOPs
- Result: a base model with strong text completion but no instruction-following behaviour

**Phase 2 — Supervised Fine-Tuning (SFT):**
- Dataset: millions of human-written (prompt, ideal response) pairs
- Covers: coding, reasoning, safety, maths, conversations, structured output
- Objective: same next-token loss, but on structured chat format
- Result: a model that follows instructions and produces chat-format responses

**Phase 3 — RLHF (Reinforcement Learning from Human Feedback):**
- See section 8.4

### 8.4 RLHF & Alignment

RLHF transforms the SFT model into the Instruct model used in this project. It has three sub-phases:

**Step 1 — Preference data collection:**
- Human annotators compare pairs of model responses for the same prompt
- They select which response is better (chosen vs. rejected)
- Millions of (prompt, chosen, rejected) triples collected

**Step 2 — Reward model training:**
- A separate model trained on the preference pairs using Bradley-Terry loss:
  `L = -log sigmoid(r_chosen - r_rejected)`
- Output: a scalar score predicting human preference for any (prompt, response) pair

**Step 3 — PPO optimisation with KL penalty:**
- The SFT model (policy) generates responses
- The reward model scores them
- PPO updates the policy to maximise reward
- A KL divergence penalty prevents the policy from drifting too far from the SFT baseline:
  `Objective = E[reward] - β × KL(policy || reference)`
- The KL term prevents reward hacking — the model cannot exploit reward model blind spots by drifting to degenerate outputs

**Additional techniques used by Meta:**
- RLAIF (AI feedback supplementing human feedback for scale)
- DPO (Direct Preference Optimisation) in some stages as a more stable PPO alternative
- Iterative RLHF — multiple rounds of preference collection and optimisation

**Why RLHF matters for this project:** The Instruct model reliably outputs valid JSON, hedges uncertainty with lower confidence scores, and escalates ambiguous cases — all behaviours shaped by RLHF that a base model would not exhibit.

### 8.5 Key Configuration Parameters

| Parameter | Value | Rationale |
|---|---|---|
| `temperature` | 0.1 | Near-deterministic sampling — same input should produce same decision |
| `max_tokens` | 1000 | Generous headroom for JSON output (typical output: 150–300 tokens) |
| `top_p` | 0.95 | Nucleus sampling — cuts off very low-probability token tail |
| `tensor_parallel_size` | 2 | Splits model across 2 GPUs per replica |
| `gpu_memory_utilization` | 0.9 | 90% VRAM for KV cache pages |
| `max_model_len` | 4096 | Max prompt + output — well above the ~800-token typical prompt |

---

## 9. vLLM & GPU Setup

### 9.1 Why vLLM

Without vLLM, running LLaMA 3.1 70B in production would require writing custom code for:
- GPU memory management and allocation
- Splitting model weights across multiple GPUs
- Request queuing and prioritisation
- KV cache management
- Tokenisation and detokenisation
- Concurrent request handling

vLLM provides all of this out-of-the-box, exposing a simple OpenAI-compatible REST API. The Python agent only needs to make an HTTP POST request.

### 9.2 PagedAttention

Traditional LLM servers pre-allocate a contiguous block of VRAM for each request's KV cache (the stored key-value tensors from attention layers). This wastes 60–90% of allocated memory because the full maximum sequence length is reserved even for short requests.

PagedAttention divides VRAM into fixed-size pages (analogous to virtual memory in an OS). Pages are allocated on demand as tokens are generated, and pages from different requests can be interleaved in physical memory. This allows:

- 3–5× more concurrent requests for the same VRAM
- No wasted pre-allocated memory
- Efficient handling of variable-length prompts

### 9.3 Tensor Parallelism

With `tensor_parallel_size=2`, each GPU holds half of every weight matrix in the model. During each forward pass:

1. Input activations are broadcast to both GPUs
2. Each GPU computes its half of the matrix multiplication
3. Results are summed via an all-reduce operation over NVLink
4. Output passes to the next layer

NVLink (up to 600 GB/s bandwidth) makes this inter-GPU communication fast enough that the overhead is small relative to the compute time.

### 9.4 GPU Memory Layout

For each replica (2 GPUs):

```
GPU 0 (80 GB):
  ├── Model weights (layers 1–40): ~70 GB
  └── KV cache pages: ~9 GB (gpu_memory_utilization=0.9, minus overhead)

GPU 1 (80 GB):
  ├── Model weights (layers 41–80): ~70 GB
  └── KV cache pages: ~9 GB
```

Combined KV cache of ~18 GB per replica, divided into pages of ~256 KB each, gives roughly 70,000 pages available for concurrent requests.

---

## 10. Security & Compliance

| Control | Implementation |
|---|---|
| Data sovereignty | LLaMA runs on-premise — no customer data sent to external APIs |
| n8n webhook auth | Bearer token authentication on all four webhook endpoints |
| SIM API auth | Rotating API keys via environment variable |
| BigQuery access | Service accounts with minimum required IAM permissions |
| Kafka ACLs | Per-topic producer/consumer authorisation |
| Secrets management | All keys injected via environment variables, never hardcoded |
| Audit trail | Every AI decision persisted to BigQuery with full reasoning text |
| Compliance | Meets telecom data protection regulations (PIPEDA for Canadian data) |

**No external model APIs are used.** The entire inference pipeline runs within the company's own infrastructure. Customer subscriber data, usage patterns, and payment history never leave the network perimeter.

---

## 11. Failure Modes & Fallbacks

| Failure | Impact | Fallback |
|---|---|---|
| vLLM service down | No AI decisions possible | Default to `ESCALATE_TO_HUMAN` — never default to ALLOW or BLOCK without AI |
| Malformed JSON output | Decision cannot be parsed | Retry once with explicit JSON-only instruction; if still fails → ESCALATE_TO_HUMAN |
| GPU OOM during spike | New requests queued or rejected | Kafka buffers events; Prometheus alert fires on latency > 5s |
| Enrichment API failure | Incomplete context | Null fields with escalation instruction in prompt |
| n8n workflow failure | Decision not executed | Retry with exponential backoff; alert on repeated failure |
| Kafka consumer lag | Events processed late | Add detector instances; Kafka auto-rebalances partitions |
| Prompt injection in notes | Model manipulation attempt | Sanitise account notes field before interpolation; label as data-only in prompt |

**Conservative fallback principle:** When uncertain, the system always escalates to a human rather than making an automatic block or allow decision. Blocking wrongly harms legitimate customers; allowing wrongly enables fraud. Human escalation is the safe middle ground.

---

## 12. Monitoring & Alerting

### Alert Thresholds

| Alert | Condition | Severity | Action |
|---|---|---|---|
| `HighAILatency` | p95 latency > 5s for 5 minutes | Warning | Check vLLM / GPU |
| `LowAIConfidence` | Rolling avg confidence < 0.70 for 10 minutes | Warning | Check enrichment data quality |
| `HighEscalationRate` | Escalation rate > 10% over 1 hour | Warning | Investigate unusual patterns |
| `KafkaConsumerLag` | Consumer lag > 1,000 events | Critical | Scale detector or check vLLM |
| `vLLMDown` | No successful inference for 60 seconds | Critical | Page on-call engineer |
| `WorkflowFailureRate` | n8n failure rate > 2% | Warning | Check n8n logs and external APIs |

### Dashboard Panels

**AI Decision Overview:**
- Decisions per hour (stacked by type)
- Confidence score distribution (histogram)
- Block rate vs. exception rate over time
- Top 10 escalation reasons

**System Performance:**
- Kafka consumer lag (line chart)
- AI decision latency p50 / p95 / p99 (line chart)
- GPU utilisation per device (line chart)
- Enrichment API latency by source

**Customer Impact:**
- SIMs blocked per hour
- Exceptions granted per hour
- Escalations resolved vs. pending
- VIP decisions breakdown

---

## 13. Deployment

### Prerequisites

- Docker 24.0+ with Docker Compose
- NVIDIA GPU drivers 535.x or later
- NVIDIA Container Toolkit
- 4× NVIDIA A100 80GB GPUs
- 256 GB+ system RAM
- Python 3.11+
- Hugging Face account with LLaMA 3.1 access

### Setup Steps

```bash
# 1. Clone repository
git clone https://github.com/sankshine/500_alert_automation_with_n8n
cd 500_alert_automation_with_n8n

# 2. Configure environment
cp .env.example .env
# Edit .env with your values (see Environment Variables below)

# 3. Download model weights
huggingface-cli login
huggingface-cli download meta-llama/Llama-3.1-70B-Instruct

# 4. Deploy full stack
docker-compose up -d

# 5. Import n8n workflows
# Access n8n at http://localhost:5678
# Import JSON workflow files from ./n8n-workflows/ directory

# 6. Verify deployment
docker-compose logs -f detector
curl http://localhost:8000/health   # vLLM health check
```

### Environment Variables

```bash
# Kafka
KAFKA_BROKER=kafka:9092
KAFKA_TOPIC=usage-events

# vLLM
VLLM_API_URL=http://vllm:8000/v1/completions

# Enrichment APIs
CRM_API_URL=https://crm.yourcompany.com
BILLING_API_URL=https://billing.yourcompany.com
SUPPORT_API_URL=https://support.yourcompany.com

# n8n
N8N_BASE_URL=http://n8n:5678/webhook
N8N_TOKEN=your-secure-token
N8N_PASSWORD=admin-password

# SIM Management
SIM_API_BASE_URL=https://sim-api.yourcompany.com
SIM_API_KEY=your-api-key

# Google Cloud (BigQuery)
GCP_PROJECT_ID=your-project-id
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json

# Thresholds
BLOCK_THRESHOLD_CAD=500.0
```

### Services Started by docker-compose

| Service | Port | Purpose |
|---|---|---|
| `kafka` | 9092 | Event streaming |
| `zookeeper` | 2181 | Kafka coordination |
| `vllm` | 8000 | LLM inference server |
| `detector` | — | Python threshold detector (no external port) |
| `n8n` | 5678 | Workflow orchestration UI + webhooks |
| `postgres` | 5432 | Customer data |
| `prometheus` | 9090 | Metrics collection |
| `grafana` | 3000 | Dashboards |

---

## 14. Performance Metrics

### Latency Breakdown

| Stage | Typical Latency |
|---|---|
| Kafka consume → threshold check | < 100 ms |
| Context enrichment (parallel API calls) | 200–500 ms |
| Prompt assembly | < 10 ms |
| vLLM inference (tokenise + forward pass + generate) | 5–15 seconds |
| JSON parse + decision routing | < 50 ms |
| n8n workflow execution | 1–3 seconds |
| **Total end-to-end** | **~8–20 seconds** |

### AI Decision Accuracy

| Metric | Value |
|---|---|
| Overall accuracy vs. human reviewers | 94% |
| BLOCK decision precision | 96% |
| ALLOW_EXCEPTION precision | 91% |
| ESCALATE_TO_HUMAN agreement rate | 89% |
| False positive rate (wrongly blocked) | 4% |

### Throughput

| Configuration | Max concurrent decisions |
|---|---|
| 2× A100 (1 replica) | 1 at a time, ~4–10/min |
| 4× A100 (2 replicas) | 2 concurrent, ~8–20/min |
| 8× A100 (4 replicas) | 4 concurrent, ~16–40/min |

For most telecom networks, 2 replicas (4 GPUs) provides comfortable headroom. Kafka buffers any burst above this rate.

---

## 15. Potential Enhancements

### 1. RAG — Retrieval-Augmented Generation
Add a vector database (pgvector, Pinecone) storing past human-reviewed decisions as embeddings. At inference time, retrieve the 3 most similar past cases and inject them as examples into the LLaMA prompt. This would improve consistency on edge cases by showing the model how humans handled near-identical situations.

### 2. Fine-tuning on Domain Data
Once 500+ human-reviewed decisions are accumulated in BigQuery, fine-tune a smaller model (LLaMA 8B) on that domain-specific data. A fine-tuned 8B model on this task may match or exceed a raw 70B model while using 1/4 the GPU resources.

### 3. Feedback Loop
Track human override rates per decision type in BigQuery. When humans override AI decisions at > 15%, trigger a model re-evaluation or fine-tuning cycle. This closes the loop between AI decisions and ground truth.

### 4. Streaming Decisions
Use vLLM's streaming endpoint to begin routing partial JSON output as tokens arrive. This reduces perceived latency but requires more complex JSON parsing (handle partial responses).

### 5. GCP Migration
Replace on-premise components with managed GCP services:

| Current | GCP Replacement |
|---|---|
| Apache Kafka | Google Cloud Pub/Sub |
| LLaMA on A100s | Vertex AI Gemini via API |
| vLLM | Vertex AI managed endpoints |
| n8n | Google Cloud Workflows + Cloud Functions |
| PostgreSQL | Cloud SQL |
| BigQuery | BigQuery (no change) |
| Prometheus + Grafana | Cloud Monitoring + Cloud Logging |

This eliminates GPU management overhead at the cost of data sovereignty (customer data would leave on-premise infrastructure) and introduces per-token API costs.

### 6. Multi-Currency Support
The threshold is currently hardcoded to CAD. Parameterise the threshold with currency-aware conversion to support international deployments.

### 7. Explainability Dashboard
Build a Grafana panel (or separate UI) showing the AI's reasoning text for each decision. Let ops managers browse decisions, filter by confidence, and mark overrides — feeding directly back into the fine-tuning pipeline.

---

*Document generated from the project: `github.com/sankshine/500_alert_automation_with_n8n`*  
*Architecture version: 1.0 | Last updated: March 2026*
