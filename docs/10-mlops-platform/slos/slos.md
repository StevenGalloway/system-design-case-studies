# MLOps Platform Service Level Objectives

## SLI Definitions

| Signal | SLI | Measurement Method |
|--------|-----|-------------------|
| Online Feature Store Latency | p99 read latency for feature key lookups during live inference | Feature store instrumentation |
| Inference Fleet Latency | p99 latency for model inference requests | Inference service instrumentation |
| Inference Fleet Availability | Percentage of time the inference fleet is healthy and serving requests | Health check monitoring |
| Feature Freshness | Age of the most recently materialized feature partition in the online store | Feature store partition metadata |
| Training Pipeline Success Rate | Percentage of scheduled training runs that complete without terminal failure | Training orchestrator metrics |
| Model Deployment Lead Time | Time from model artifact registration to full production promotion | Model registry + deployment timestamps |
| Drift Detection Latency | Time from drift onset to alert delivery | Monitoring system timestamps |

---

## SLO Targets

| SLI | Target | Measurement Window | Error Budget (30-day) |
|-----|--------|-------------------|----------------------|
| Online feature store latency (p99) | <= 10 ms | 30-day rolling | 0.1% of requests |
| Inference fleet latency (p99) | <= 50 ms | 30-day rolling | 0.1% of requests |
| Inference fleet availability | >= 99.99% | 30-day rolling | 4.4 minutes downtime |
| Feature freshness (critical features) | <= 5 minutes | 24-hour rolling | 0 breaches |
| Feature freshness (standard features) | <= 30 minutes | 24-hour rolling | <= 2 breaches per month |
| Training pipeline success rate | >= 99% | 30-day rolling | 1% of scheduled runs |
| Model deployment lead time | <= 4 hours (shadow + canary) | Per-deployment | Tracked as process metric |
| Drift detection latency (p99) | <= 30 minutes from drift onset | 30-day rolling | 0.1% of drift events |

---

## Model Tier Classification

| Tier | Staleness Tolerance | Retraining Cadence | On-Call Priority |
|------|--------------------|--------------------|-----------------|
| Tier 1 (Business-critical) | <= 24 hours | Daily | SEV-1 on inference outage |
| Tier 2 (High-value) | <= 72 hours | 2-3 times per week | SEV-2 on inference outage |
| Tier 3 (Standard) | <= 7 days | Weekly | SEV-3 on inference outage |
| Tier 4 (Experimental) | No SLA | As needed | Best effort |

Tier 1 models include: personalization ranking, fraud detection, QoE anomaly detection.

---

## Error Budget Policy

| Error Budget Remaining | Action |
|-----------------------|--------|
| > 50% | Normal platform operations; model deployments proceed on standard schedule |
| 25% - 50% | High-risk model updates require MLOps platform lead approval |
| 10% - 25% | All model promotions paused; infrastructure changes frozen |
| < 10% | Incident declared; leadership notified; model deployments halted |
| 0% (exhausted) | Full incident response; mandatory postmortem; 2-week stability window |

---

## Alerting Thresholds

| Alert | Threshold | Severity | Responder |
|-------|-----------|----------|-----------|
| Inference fleet latency p99 | > 50 ms for 5 min | SEV-2 | MLOps SRE |
| Inference fleet availability | < 99.9% for 2 min | SEV-1 | MLOps SRE |
| Feature freshness breach (Tier 1) | > 5 min stale | SEV-1 | MLOps SRE + Data Platform |
| Feature freshness breach (Tier 2) | > 30 min stale | SEV-2 | MLOps SRE |
| Training pipeline failure | Job FAILED after retries | SEV-2 | MLOps SRE |
| Feature drift (PSI) | PSI > 0.2 for key feature | SEV-2 | MLOps SRE + Model Owner |
| Prediction drift | Distribution change > 20% for 30 min | SEV-2 | MLOps SRE + Model Owner |

---

## Exclusions

- Training pipeline delays caused by upstream feature store outages (SLO accountability transfers to data platform)
- Inference latency increases caused by model complexity changes explicitly approved by the model owner
- Drift events caused by known external events (major content releases, global streaming peaks) that are pre-declared in the platform calendar
