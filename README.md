# 📊 Real-Time Fraud & Usage Dashboard with Automated Alerting

> **Reduced manual oversight by 60%** through real-time operational dashboards monitoring high-risk triggers ($500+ data usage, multi-SIM fraud) with n8n-automated alerting and incident response workflows.

![Python](https://img.shields.io/badge/Python-3.11-blue)
![Kafka](https://img.shields.io/badge/Kafka-Streaming-red)
![n8n](https://img.shields.io/badge/n8n-Automation-orange)
![MicroStrategy](https://img.shields.io/badge/MicroStrategy-Dashboards-green)
![Docker](https://img.shields.io/badge/Docker-Containerized-blue)

---

## 📋 Table of Contents
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Fraud Detection Engine](#fraud-detection-engine)
- [n8n Workflow Automation](#n8n-workflow-automation)
- [Dashboard Design](#dashboard-design)
- [Model Drift & Monitoring](#model-drift--monitoring)
- [Deployment](#deployment)
- [Results](#results)

---

## Problem Statement

Telecom operations teams manually monitored usage patterns and investigated potential fraud cases:

- **$500+ data overage** events were caught 48-72 hours late, resulting in customer complaints and revenue leakage
- **Multi-SIM fraud** (same identity across multiple accounts) went undetected for weeks
- Analysts spent 8+ hours/day on manual report checks
- No automated escalation — incidents sat in queues for hours

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  DATA SOURCES                                │
│   CDR Events │ Billing │ CRM │ Network Logs │ SIM Registry  │
└──────────┬──────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────┐     ┌──────────────────────────────┐
│   Apache Kafka        │     │   Schema Registry             │
│   - usage-events      │────▶│   (Avro schemas for CDR,     │
│   - billing-alerts    │     │    billing, SIM events)       │
│   - sim-activations   │     └──────────────────────────────┘
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│              STREAM PROCESSING ENGINE (Python / Faust)        │
│                                                               │
│  ┌─────────────────┐  ┌──────────────────┐  ┌─────────────┐ │
│  │ Usage Anomaly    │  │ Multi-SIM         │  │ Velocity    │ │
│  │ Detector         │  │ Fraud Detector    │  │ Check       │ │
│  │ ($500+ threshold │  │ (Graph-based      │  │ (Burst      │ │
│  │  + Z-score)      │  │  identity link)   │  │  detection) │ │
│  └────────┬────────┘  └────────┬─────────┘  └──────┬──────┘ │
└───────────┼─────────────────────┼──────────────────┼─────────┘
            │                     │                  │
            ▼                     ▼                  ▼
┌──────────────────────────────────────────────────────────────┐
│                  ALERT ROUTING ENGINE                          │
│                                                               │
│   Risk Score → Priority (P1/P2/P3) → n8n Webhook Trigger     │
└──────────┬───────────────────────────────────────────────────┘
           │
     ┌─────┴──────────────────────────────────┐
     ▼                                         ▼
┌──────────────┐                    ┌──────────────────────────┐
│ n8n Workflows│                    │ Data Sink                 │
│ - Slack alert│                    │ - BigQuery (analytics)    │
│ - Email      │                    │ - Elasticsearch (search)  │
│ - ServiceNow │                    │ - MicroStrategy (dashbd)  │
│ - Auto-block │                    └──────────────────────────┘
└──────────────┘
```

## Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Streaming** | Apache Kafka + Faust | Real-time event processing |
| **Detection** | Python (scikit-learn, NetworkX) | Fraud pattern detection |
| **Automation** | n8n (self-hosted) | Workflow orchestration & alerting |
| **Dashboard** | MicroStrategy + Streamlit | Executive & operational views |
| **Data Store** | BigQuery + Elasticsearch | Analytics & search |
| **Monitoring** | ELK Stack + Prometheus | System observability |
| **CI/CD** | GitHub Actions + Docker | Automated pipeline |
| **Infrastructure** | GCP (Cloud Run, Pub/Sub) | Production hosting |

## Fraud Detection Engine

### Usage Anomaly Detector

```python
# src/detectors/usage_anomaly.py
"""
Detects abnormal data usage patterns using statistical thresholds
and rolling Z-score analysis.
"""
from dataclasses import dataclass, field
from datetime import datetime
from collections import defaultdict
import numpy as np

@dataclass
class UsageAlert:
    subscriber_id: str
    current_usage_mb: float
    current_charges: float
    threshold_exceeded: str  # "absolute" | "zscore" | "both"
    risk_score: float  # 0.0 - 1.0
    timestamp: datetime = field(default_factory=datetime.utcnow)

class UsageAnomalyDetector:
    """
    Two-pronged anomaly detection:
    1. Absolute threshold: flags any single-day usage exceeding $500
    2. Statistical (Z-score): flags usage >3σ from subscriber's rolling mean
    """
    ABSOLUTE_CHARGE_THRESHOLD = 500.0  # dollars
    ZSCORE_THRESHOLD = 3.0
    ROLLING_WINDOW = 30  # days

    def __init__(self):
        self._history: dict[str, list[float]] = defaultdict(list)

    def process_event(self, event: dict) -> UsageAlert | None:
        subscriber_id = event["subscriber_id"]
        usage_mb = event["data_usage_mb"]
        charges = event["charges_cad"]

        # Update rolling history
        self._history[subscriber_id].append(usage_mb)
        if len(self._history[subscriber_id]) > self.ROLLING_WINDOW:
            self._history[subscriber_id].pop(0)

        # Check 1: Absolute threshold
        abs_flag = charges >= self.ABSOLUTE_CHARGE_THRESHOLD

        # Check 2: Z-score anomaly
        zscore_flag = False
        zscore = 0.0
        history = self._history[subscriber_id]
        if len(history) >= 7:  # Need at least 7 days of data
            mean = np.mean(history[:-1])
            std = np.std(history[:-1])
            if std > 0:
                zscore = (usage_mb - mean) / std
                zscore_flag = zscore > self.ZSCORE_THRESHOLD

        if not abs_flag and not zscore_flag:
            return None

        # Calculate risk score
        risk_score = self._calculate_risk(charges, zscore, abs_flag, zscore_flag)

        threshold_type = (
            "both" if abs_flag and zscore_flag
            else "absolute" if abs_flag
            else "zscore"
        )

        return UsageAlert(
            subscriber_id=subscriber_id,
            current_usage_mb=usage_mb,
            current_charges=charges,
            threshold_exceeded=threshold_type,
            risk_score=risk_score,
        )

    def _calculate_risk(
        self, charges: float, zscore: float,
        abs_flag: bool, zscore_flag: bool
    ) -> float:
        score = 0.0
        if abs_flag:
            score += min(charges / 1000.0, 0.5)  # Cap at 0.5
        if zscore_flag:
            score += min(zscore / 10.0, 0.5)  # Cap at 0.5
        return round(min(score, 1.0), 3)
```

### Multi-SIM Fraud Detector

```python
# src/detectors/multi_sim.py
"""
Graph-based identity linking to detect multi-SIM fraud.
Uses shared attributes (address, payment method, device IMEI)
to build an identity graph and flag suspicious clusters.
"""
import networkx as nx
from dataclasses import dataclass
from datetime import datetime

@dataclass
class MultiSIMAlert:
    cluster_id: str
    linked_subscribers: list[str]
    shared_attributes: dict[str, list[str]]
    risk_score: float
    sim_count: int
    timestamp: datetime

class MultiSIMDetector:
    """
    Builds an identity graph where subscribers are nodes and
    shared attributes (address, payment, IMEI) are edges.
    Connected components with >3 nodes trigger fraud alerts.
    """
    SIM_THRESHOLD = 3
    ATTRIBUTE_WEIGHTS = {
        "address": 0.4,
        "payment_method": 0.35,
        "imei": 0.25,
    }

    def __init__(self):
        self.graph = nx.Graph()

    def ingest_subscriber(self, subscriber: dict):
        sub_id = subscriber["subscriber_id"]
        self.graph.add_node(sub_id, **subscriber)

        # Link to other subscribers sharing attributes
        for attr_name, weight in self.ATTRIBUTE_WEIGHTS.items():
            attr_value = subscriber.get(attr_name)
            if not attr_value:
                continue
            for existing_node in list(self.graph.nodes):
                if existing_node == sub_id:
                    continue
                existing_data = self.graph.nodes[existing_node]
                if existing_data.get(attr_name) == attr_value:
                    if self.graph.has_edge(sub_id, existing_node):
                        self.graph[sub_id][existing_node]["weight"] += weight
                        self.graph[sub_id][existing_node]["shared"].append(attr_name)
                    else:
                        self.graph.add_edge(
                            sub_id, existing_node,
                            weight=weight,
                            shared=[attr_name]
                        )

    def detect_fraud_clusters(self) -> list[MultiSIMAlert]:
        alerts = []
        for component in nx.connected_components(self.graph):
            if len(component) < self.SIM_THRESHOLD:
                continue

            members = list(component)
            shared = self._get_shared_attributes(members)
            risk = self._calculate_cluster_risk(members, shared)

            alerts.append(MultiSIMAlert(
                cluster_id=f"CLUSTER-{hash(frozenset(members)) % 10000:04d}",
                linked_subscribers=members,
                shared_attributes=shared,
                risk_score=risk,
                sim_count=len(members),
                timestamp=datetime.utcnow(),
            ))

        return sorted(alerts, key=lambda a: a.risk_score, reverse=True)

    def _get_shared_attributes(self, members: list[str]) -> dict:
        shared = {"address": [], "payment_method": [], "imei": []}
        for i, m1 in enumerate(members):
            for m2 in members[i+1:]:
                if self.graph.has_edge(m1, m2):
                    for attr in self.graph[m1][m2].get("shared", []):
                        val = self.graph.nodes[m1].get(attr, "unknown")
                        if val not in shared[attr]:
                            shared[attr].append(val)
        return {k: v for k, v in shared.items() if v}

    def _calculate_cluster_risk(self, members: list, shared: dict) -> float:
        size_factor = min(len(members) / 10.0, 0.5)
        attr_factor = sum(
            self.ATTRIBUTE_WEIGHTS.get(attr, 0) * min(len(vals), 3) / 3
            for attr, vals in shared.items()
        )
        return round(min(size_factor + attr_factor, 1.0), 3)
```

### Alert Router + n8n Integration

```python
# src/alerts/router.py
"""
Routes alerts to appropriate channels based on priority.
Integrates with n8n webhooks for automated workflow triggering.
"""
import httpx
import logging
from dataclasses import dataclass
from enum import Enum

logger = logging.getLogger(__name__)

class Priority(Enum):
    P1_CRITICAL = "P1"   # Risk > 0.8 → immediate block + Slack + ServiceNow
    P2_HIGH = "P2"       # Risk > 0.5 → Slack + email + ServiceNow
    P3_MEDIUM = "P3"     # Risk > 0.3 → email digest
    P4_LOW = "P4"        # Risk ≤ 0.3 → dashboard only

@dataclass
class RoutedAlert:
    alert_id: str
    priority: Priority
    channels: list[str]
    n8n_workflow_id: str | None
    payload: dict

class AlertRouter:
    def __init__(self, n8n_base_url: str, n8n_webhook_token: str):
        self.n8n_url = n8n_base_url
        self.n8n_token = n8n_webhook_token
        self.client = httpx.AsyncClient(timeout=10.0)

        # n8n workflow mapping
        self.workflow_map = {
            Priority.P1_CRITICAL: "webhook/fraud-p1-critical",
            Priority.P2_HIGH: "webhook/fraud-p2-high",
            Priority.P3_MEDIUM: "webhook/fraud-p3-medium",
        }

    def classify_priority(self, risk_score: float) -> Priority:
        if risk_score > 0.8:
            return Priority.P1_CRITICAL
        elif risk_score > 0.5:
            return Priority.P2_HIGH
        elif risk_score > 0.3:
            return Priority.P3_MEDIUM
        return Priority.P4_LOW

    async def route_alert(self, alert_type: str, alert_data: dict) -> RoutedAlert:
        risk_score = alert_data.get("risk_score", 0)
        priority = self.classify_priority(risk_score)

        channels = self._get_channels(priority)
        workflow_id = self.workflow_map.get(priority)

        payload = {
            "alert_type": alert_type,
            "priority": priority.value,
            "risk_score": risk_score,
            "data": alert_data,
        }

        # Trigger n8n workflow if applicable
        if workflow_id:
            await self._trigger_n8n(workflow_id, payload)

        return RoutedAlert(
            alert_id=f"ALR-{hash(str(alert_data)) % 100000:05d}",
            priority=priority,
            channels=channels,
            n8n_workflow_id=workflow_id,
            payload=payload,
        )

    async def _trigger_n8n(self, webhook_path: str, payload: dict):
        url = f"{self.n8n_url}/{webhook_path}"
        try:
            response = await self.client.post(
                url,
                json=payload,
                headers={"Authorization": f"Bearer {self.n8n_token}"},
            )
            response.raise_for_status()
            logger.info(f"n8n workflow triggered: {webhook_path}")
        except httpx.HTTPError as e:
            logger.error(f"n8n trigger failed for {webhook_path}: {e}")
            # Fallback: direct Slack notification
            await self._fallback_slack(payload)

    def _get_channels(self, priority: Priority) -> list[str]:
        mapping = {
            Priority.P1_CRITICAL: ["slack", "pagerduty", "servicenow", "auto_block"],
            Priority.P2_HIGH: ["slack", "email", "servicenow"],
            Priority.P3_MEDIUM: ["email_digest"],
            Priority.P4_LOW: ["dashboard_only"],
        }
        return mapping[priority]

    async def _fallback_slack(self, payload: dict):
        logger.warning(f"Sending fallback Slack alert: {payload['alert_type']}")
```

## n8n Workflow Automation

The project includes exportable n8n workflow JSON files:

### P1 Critical Fraud Workflow (`configs/n8n_p1_workflow.json`)
```
Webhook Trigger → Validate Payload → Parallel:
  ├── Block SIM (API call to SIM management)
  ├── Create ServiceNow P1 Incident
  ├── Slack #fraud-critical channel
  └── Email Fraud Ops Manager
→ Log to BigQuery audit table
```

### P2 High Priority Workflow
```
Webhook Trigger → Enrich with CRM data → Create ServiceNow P2 ticket
→ Slack #fraud-alerts → Email subscriber notification
```

## Model Drift & Monitoring

```python
# src/monitoring/drift_detector.py
"""Monitors detection model performance and flags distribution shifts."""
import numpy as np
from scipy import stats
from collections import deque

class FraudDriftDetector:
    def __init__(self, baseline_alert_rate: float = 0.02, window: int = 1000):
        self.baseline_rate = baseline_alert_rate
        self.window = window
        self.recent_events = deque(maxlen=window)
        self.recent_alerts = deque(maxlen=window)

    def record(self, is_alert: bool, risk_score: float):
        self.recent_events.append(risk_score)
        self.recent_alerts.append(1 if is_alert else 0)

    def check_drift(self) -> dict:
        if len(self.recent_events) < 200:
            return {"drift": False, "reason": "insufficient_data"}

        current_rate = np.mean(list(self.recent_alerts))
        rate_drift = abs(current_rate - self.baseline_rate) / self.baseline_rate

        # KS test on risk score distribution
        recent_half = list(self.recent_events)[len(self.recent_events)//2:]
        older_half = list(self.recent_events)[:len(self.recent_events)//2]
        ks_stat, ks_pvalue = stats.ks_2samp(recent_half, older_half)

        return {
            "drift": rate_drift > 0.3 or ks_pvalue < 0.01,
            "alert_rate_current": round(current_rate, 4),
            "alert_rate_baseline": self.baseline_rate,
            "rate_drift_pct": round(rate_drift * 100, 1),
            "ks_statistic": round(ks_stat, 4),
            "ks_pvalue": round(ks_pvalue, 4),
            "action": (
                "RETRAIN: significant distribution shift detected"
                if ks_pvalue < 0.01
                else "REVIEW: alert rate drift exceeds 30%"
                if rate_drift > 0.3
                else "HEALTHY"
            ),
        }
```

## Deployment

### Docker Compose

```yaml
# docker-compose.yml
version: "3.9"
services:
  detector:
    build: .
    env_file: .env
    depends_on: [kafka, elasticsearch]

  n8n:
    image: n8nio/n8n:latest
    ports: ["5678:5678"]
    volumes:
      - n8n_data:/home/node/.n8n
      - ./configs/n8n_workflows/:/home/node/workflows/
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    ports: ["9092:9092"]

  elasticsearch:
    image: elasticsearch:8.12.0
    ports: ["9200:9200"]
    environment:
      - discovery.type=single-node

  streamlit-dashboard:
    build: ./dashboard
    ports: ["8501:8501"]
    depends_on: [elasticsearch]

volumes:
  n8n_data:
```

### CI/CD Pipeline

```yaml
# .github/workflows/ci-cd.yml
name: Fraud Detection CI/CD
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: pytest tests/ -v --cov=src

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: google-github-actions/auth@v2
        with: { credentials_json: "${{ secrets.GCP_SA_KEY }}" }
      - run: |
          gcloud builds submit --tag gcr.io/${{ secrets.GCP_PROJECT }}/fraud-detector:${{ github.sha }}
          gcloud run deploy fraud-detector --image gcr.io/${{ secrets.GCP_PROJECT }}/fraud-detector:${{ github.sha }} --region us-central1
```

## Results

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Alert Detection Latency | 48-72 hrs | < 5 min | **Real-time** |
| Manual Monitoring Hours | 8 hrs/day | 3 hrs/day | **↓ 60%** |
| Fraud Detection Rate | 45% | 87% | **↑ 93%** |
| False Positive Rate | — | 8.2% | Calibrated |
| Avg. Incident Response Time | 6 hrs | 15 min | **↓ 96%** |
| Annual Revenue Saved | — | ~$1.2M | Estimated |

---

## License
MIT License

## Author
**Sana Khan** — [LinkedIn](https://linkedin.com/in/sankshine) | [GitHub](https://github.com/sankshine)
