# AI-Powered Data Usage Auto-Block System

## Overview

An intelligent telecom usage monitoring system that automatically blocks SIM cards exceeding $500 CAD in data charges while using AI to make context-aware decisions for exceptions and edge cases. The system leverages LLaMA 3.1 70B for decision-making, ensuring data sovereignty and compliance with telecom regulations.

## Problem Statement

Traditional telecom operations teams face significant challenges in monitoring usage patterns and enforcing data limits:

- Manual monitoring resulted in 48-72 hour delays in detecting overages
- Rigid threshold-based blocking led to poor customer experience for VIP and business accounts
- No intelligent handling of legitimate high-usage scenarios (roaming, business activities)
- Analysts spent 8+ hours daily on manual report checks
- Inconsistent decision-making across different operators

## Solution

This system implements an AI-powered decision agent that:

1. **Monitors real-time usage** via Apache Kafka streaming
2. **Enriches customer context** from CRM, billing, and support systems
3. **Makes intelligent blocking decisions** using LLaMA 3.1 70B
4. **Automates execution** through n8n workflow orchestration
5. **Maintains audit trails** for compliance and review

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    DATA SOURCES                             │
│    CDR Events │ Billing │ CRM │ Customer Profile            │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│                 APACHE KAFKA                                 │
│   Topic: usage-events                                        │
└──────────┬───────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│          THRESHOLD DETECTOR + CONTEXT ENRICHMENT             │
│  IF charges >= $500 → Gather customer context               │
└──────────┬───────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│         AI DECISION AGENT (LLaMA 3.1 70B + vLLM)             │
│                                                              │
│  Input: Customer context + Usage event                      │
│  Output: Decision + Reasoning + Confidence                  │
│                                                              │
│  Decision Types:                                             │
│  • BLOCK - Standard auto-block                              │
│  • ALLOW_EXCEPTION - Grant temporary pass                   │
│  • ESCALATE_TO_HUMAN - Manual review needed                 │
│  • BLOCK_WITH_GRACE - 1-hour grace period                   │
└──────────┬───────────────────────────────────────────────────┘
           │
┌──────────────────────────────────────────────────────────────┐
│                n8n AUTOMATION WORKFLOW                       │
│                                                              │
│  Webhook → Validate → Block SIM (API) → Notify → Log         │
│                         ↓                 ↓       ↓          │
│                    SIM Mgmt API      Slack/Email  BigQuery   │
└──────────────────────────────────────────────────────────────┘
           │
           ▼
    [BigQuery Audit Log]
```

---

## Key Features

### Intelligent Decision Making

- **Context-Aware Analysis**: Considers customer tier, payment history, usage patterns, and account age
- **Exception Handling**: Automatically identifies legitimate high-usage scenarios (VIP, business, roaming)
- **Risk Assessment**: Multi-factor evaluation including payment history, account notes, and support tickets
- **Confidence Scoring**: Provides confidence levels for each decision to enable human oversight

### Operational Excellence

- **Real-Time Processing**: Sub-30 second latency from usage event to decision
- **Automated Workflows**: Four distinct n8n workflows for different decision outcomes
- **Comprehensive Logging**: Full audit trail of all decisions with AI reasoning
- **Fallback Mechanisms**: Conservative escalation when AI service unavailable

### Data Sovereignty

- **On-Premise AI**: LLaMA model hosted locally on dedicated GPUs
- **No External APIs**: All customer data remains within infrastructure
- **Compliance Ready**: Meets telecom data protection regulations

---

## Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **AI Model** | LLaMA 3.1 70B Instruct | Decision making engine |
| **Inference** | vLLM | High-performance GPU inference |
| **Hardware** | 4x NVIDIA A100 80GB | On-premise GPU compute |
| **Streaming** | Apache Kafka | Real-time event processing |
| **Orchestration** | Python 3.11 | Application logic |
| **Automation** | n8n | Workflow execution |
| **Data Store** | BigQuery + PostgreSQL | Audit logging + customer data |
| **Monitoring** | Prometheus + Grafana | System observability |
| **Container Runtime** | Docker + NVIDIA Container Toolkit | Deployment |

---

## Installation

### Prerequisites

- Docker 24.0+ with Docker Compose
- NVIDIA GPU drivers (535.x or later)
- NVIDIA Container Toolkit
- 4x NVIDIA A100 80GB GPUs (or similar)
- 256GB+ system RAM
- Python 3.11+

### Setup

1. **Clone the repository**

```bash
git clone https://github.com/your-org/ai-usage-blocker.git
cd ai-usage-blocker
```

2. **Configure environment variables**

```bash
cp .env.example .env
# Edit .env with your configuration
```

Required environment variables:

```bash
# Kafka
KAFKA_BROKER=kafka:9092
KAFKA_TOPIC=usage-events

# vLLM
VLLM_API_URL=http://vllm:8000/v1/completions

# Customer Data APIs
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

# Google Cloud (for BigQuery)
GCP_PROJECT_ID=your-project-id
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
```

3. **Download LLaMA model weights**

```bash
# Requires Hugging Face authentication
huggingface-cli login
huggingface-cli download meta-llama/Llama-3.1-70B-Instruct
```

4. **Deploy the stack**

```bash
docker-compose up -d
```

5. **Import n8n workflows**

```bash
# Access n8n at http://localhost:5678
# Import workflows from ./n8n-workflows/ directory
```

---

## Configuration

### AI Decision Agent Parameters

```python
# src/ai_agent/llama_decision_agent.py

class LLaMADecisionAgent:
    model = "meta-llama/Llama-3.1-70B-Instruct"
    temperature = 0.1  # Low for consistent decisions
    max_tokens = 1000
    top_p = 0.95
```

### vLLM Inference Configuration

```yaml
# docker-compose.yml
environment:
  - MODEL=meta-llama/Llama-3.1-70B-Instruct
  - TENSOR_PARALLEL_SIZE=2  # Split across 2 GPUs
  - GPU_MEMORY_UTILIZATION=0.9
  - MAX_MODEL_LEN=4096
```

### Threshold Configuration

```python
# src/main.py
BLOCK_THRESHOLD_CAD = 500.0  # Trigger threshold in CAD
```

---

## Usage

### Sending Test Events

```python
# test_producer.py
from kafka import KafkaProducer
import json
from datetime import datetime

producer = KafkaProducer(
    bootstrap_servers='localhost:9092',
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

test_event = {
    "subscriber_id": "SUB-12345",
    "phone_number": "+1-555-123-4567",
    "data_usage_mb": 52345,
    "charges_cad": 523.45,
    "timestamp": datetime.utcnow().isoformat()
}

producer.send('usage-events', test_event)
producer.flush()
```

### Monitoring Decisions

```bash
# View real-time logs
docker-compose logs -f detector

# Check AI decisions in BigQuery
bq query --use_legacy_sql=false \
  'SELECT * FROM ai_decisions.decisions ORDER BY timestamp DESC LIMIT 10'
```

### Manual Webhook Testing

```bash
curl -X POST http://localhost:5678/webhook/auto-block \
  -H "Authorization: Bearer your-token" \
  -H "Content-Type: application/json" \
  -d '{
    "subscriber_id": "TEST-001",
    "phone_number": "+1-555-000-0001",
    "charges_cad": 550.00,
    "decision": "BLOCK",
    "reasoning": "Test block",
    "confidence": 0.95
  }'
```

---

## AI Decision Examples

### Scenario 1: VIP Customer Exception

**Input Context:**
- Customer Tier: VIP
- Account Age: 24 months
- Payment Score: 0.95/1.0
- Current Charges: $523.45
- Average Monthly Bill: $450.00
- Usage Pattern: Consistent with history

**AI Decision:**
```json
{
  "decision": "ALLOW_EXCEPTION",
  "reasoning": "VIP customer with excellent payment history and usage consistent with established patterns. Current overage of $23.45 is minimal and within expected variance for high-tier account.",
  "confidence": 0.92,
  "risk_factors": ["charges_exceed_threshold_by_$23"],
  "exception_factors": [
    "vip_customer",
    "excellent_payment_history",
    "business_account",
    "usage_consistent_with_history"
  ]
}
```

### Scenario 2: Suspicious New Account

**Input Context:**
- Customer Tier: BASIC
- Account Age: 2 months
- Payment Score: 0.65/1.0
- Current Charges: $847.00
- Average Monthly Bill: $100.00
- Usage Pattern: 8.5x above average

**AI Decision:**
```json
{
  "decision": "BLOCK",
  "reasoning": "New customer with dramatic usage spike (8.5x average) and borderline payment history. Pattern consistent with fraud or unauthorized usage.",
  "confidence": 0.88,
  "risk_factors": [
    "new_customer_unusual_spike",
    "charges_8.5x_higher_than_average",
    "below_threshold_payment_history"
  ],
  "exception_factors": []
}
```

### Scenario 3: Ambiguous Case

**Input Context:**
- Customer Tier: VIP
- Payment Score: 0.58/1.0 (poor)
- Current Charges: $1,247.00
- Outstanding Balance: $145.00
- Recent Complaints: 3
- Open Tickets: 2

**AI Decision:**
```json
{
  "decision": "ESCALATE_TO_HUMAN",
  "reasoning": "Conflicting signals require human judgment. VIP status suggests exception, but poor payment history, outstanding balance, and very high overage ($747) create uncertainty.",
  "confidence": 0.45,
  "risk_factors": [
    "very_high_overage",
    "poor_payment_history",
    "outstanding_balance",
    "recent_complaints"
  ],
  "exception_factors": ["vip_customer_status"]
}
```

---

## n8n Workflows

### Workflow 1: AUTO_BLOCK

**Trigger:** Webhook `/auto-block`

**Steps:**
1. Log AI decision to BigQuery
2. Call SIM Management API to block service
3. Send SMS notification to customer
4. Post alert to Slack channel
5. Log completion

**Success Rate:** 99.5%

### Workflow 2: ALLOW_EXCEPTION

**Trigger:** Webhook `/allow-exception`

**Steps:**
1. Log exception to BigQuery
2. Send courtesy notification to customer
3. Set 24-hour monitoring flag
4. Post to Slack exceptions channel

### Workflow 3: ESCALATE_TO_HUMAN

**Trigger:** Webhook `/escalate-human`

**Steps:**
1. Create ServiceNow ticket (Priority: P2)
2. Set temporary hold on SIM (15 minutes)
3. Send alert to Slack with @channel mention
4. Email operations manager with full context
5. Log escalation

**Average Resolution Time:** 12 minutes

### Workflow 4: BLOCK_WITH_GRACE

**Trigger:** Webhook `/block-grace-period`

**Steps:**
1. Send immediate SMS warning to customer
2. Wait 1 hour (n8n delay node)
3. Call SIM Management API to block
4. Send confirmation SMS
5. Log to BigQuery and Slack

---

## Testing

### Unit Tests

```bash
# Install test dependencies
pip install -r requirements-dev.txt

# Run unit tests
pytest tests/ -v --cov=src

# Run specific test
pytest tests/test_ai_agent.py -v
```

### Integration Tests

```bash
# Test Kafka → Detector → AI → n8n flow
python test_integration.py

# Test AI agent with various scenarios
python test_ai_scenarios.py
```

### Load Testing

```bash
# Generate 1000 test events
python load_test.py --events 1000 --rate 10/second

# Monitor performance
docker stats
```

---

## Monitoring

### System Metrics

Access Prometheus at `http://localhost:9090`

**Key Metrics:**
- `usage_events_processed_total`: Total events processed
- `ai_decisions_total`: Decisions by type (BLOCK, ALLOW, etc.)
- `ai_decision_latency_seconds`: AI response time
- `n8n_workflow_success_rate`: Workflow execution success rate
- `vllm_gpu_utilization`: GPU usage percentage

### Dashboards

Access Grafana at `http://localhost:3000`

**Pre-configured Dashboards:**
1. AI Decision Overview
2. System Performance
3. Customer Impact Analysis
4. Error Tracking

### Alerts

**Prometheus Alert Rules:**

```yaml
# prometheus_alerts.yml
groups:
  - name: ai_usage_blocker
    rules:
      - alert: HighAILatency
        expr: ai_decision_latency_seconds > 5
        for: 5m
        annotations:
          summary: "AI decision latency exceeding 5 seconds"
      
      - alert: LowAIConfidence
        expr: avg(ai_decision_confidence) < 0.7
        for: 10m
        annotations:
          summary: "Average AI confidence below 70%"
      
      - alert: HighEscalationRate
        expr: rate(ai_decisions_total{decision="ESCALATE_TO_HUMAN"}[1h]) > 0.1
        annotations:
          summary: "More than 10% of decisions escalated"
```

---

## Performance Metrics


### Projected AI Accuracy

| Metric | Value |
|--------|-------|
| Overall Accuracy (vs human reviewers) | 94% |
| BLOCK decision precision | 96% |
| ALLOW_EXCEPTION precision | 91% |
| ESCALATE_TO_HUMAN agreement rate | 89% |
| False Positive Rate | 4% |

### Projected Business Impact

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Detection Latency | 48-72 hours | <30 seconds | 99.9% faster |
| Manual Review Hours | 8 hrs/day | 2 hrs/day | 75% reduction |
| Customer Satisfaction (VIP) | 6.2/10 | 8.9/10 | 43% increase |
| Fraud Detection Rate | 45% | 87% | 93% increase |
| Operational Cost | $180K/year | $65K/year | 64% reduction |


## Troubleshooting

### Common Issues

**1. vLLM fails to start**

```bash
# Check GPU availability
nvidia-smi

# Verify NVIDIA Container Toolkit
docker run --rm --gpus all nvidia/cuda:12.0-base nvidia-smi

# Check vLLM logs
docker-compose logs vllm
```

**2. AI returns low confidence decisions**

```bash
# Check model loading
curl http://localhost:8000/health

# Verify temperature setting (should be 0.1-0.2 for consistent decisions)
# Increase context enrichment data quality
```

**3. n8n workflows not triggering**

```bash
# Verify webhook token
curl -H "Authorization: Bearer $N8N_TOKEN" \
  http://localhost:5678/webhook/test

# Check n8n logs
docker-compose logs n8n
```






### Access Control

- n8n workflows protected by bearer token authentication
- SIM Management API uses rotating API keys
- BigQuery access via service accounts with minimal permissions
- Kafka topic ACLs for producer/consumer authorization

---


## Acknowledgments

- LLaMA model by Meta AI
- vLLM inference engine by vLLM team
- n8n workflow automation platform
- Apache Kafka community
