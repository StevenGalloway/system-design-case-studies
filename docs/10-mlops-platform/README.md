# MLOps Platform

## Purpose

The MLOps Platform provides the infrastructure for training, validating, deploying, and monitoring machine learning models across Netflix. It governs the full lifecycle from raw feature engineering to production inference, ensuring that models are safe to deploy, performing within expectations, and governed by clear ownership and rollback procedures.

Without a structured MLOps platform, the personalization, content ranking, encoding optimization, and anomaly detection systems cannot safely operate at production scale. Model drift goes undetected. Feature pipelines go stale. Bad model updates reach production without meaningful validation.

---

## Scope

This platform serves all ML model workloads across the organization:

| Domain | Model Types |
|--------|------------|
| Personalization | Recommendation ranking, homepage layout, content affinity |
| Content Strategy | Demand forecasting, content scoring, licensed content prioritization |
| Video Encoding | Per-title complexity analysis, bitrate ladder optimization |
| Observability | Anomaly detection, QoE degradation prediction |
| Security | Fraud detection, credential stuffing pattern recognition |

---

## Core Subsystems

| Subsystem | Purpose |
|-----------|---------|
| Feature Store | Centralized storage for ML features; serves both online (low-latency) and offline (batch training) access patterns |
| Training Pipeline | Managed Spark and GPU-based training workflows with reproducible experiment tracking |
| Model Registry | Versioned storage for trained model artifacts with metadata, lineage, and deployment history |
| Shadow Deployment Controller | Routes a configurable percentage of production traffic to a new model for validation before full promotion |
| Inference Fleet | Horizontally scalable model serving infrastructure; supports synchronous and asynchronous inference |
| Model Monitoring | Tracks feature distribution drift, prediction drift, and business metric impact for deployed models |
| Experiment Tracker | Records hyperparameters, training data versions, and evaluation metrics for every training run |

---

## Architecture Diagrams

- `diagrams/model-lifecycle.mmd` - full model lifecycle from feature engineering to production inference and monitoring
- `diagrams/feature-store-topology.mmd` - feature store architecture covering online and offline access patterns

---

## Model Lifecycle

```
Feature Engineering (Spark / Flink)
         |
  Feature Store (Offline)
         |
  Training Run (Spark / GPU cluster)
         |
  Experiment Tracker (metrics, parameters, artifacts)
         |
  Model Registry (versioned artifact)
         |
  Shadow Deployment (production traffic slice)
         |
  Canary Promotion (5% -> 25% -> 50% -> 100%)
         |
  Production Inference Fleet
         |
  Model Monitoring (drift, latency, business metrics)
         |
  Rollback if drift or regression detected
```

---

## Key Design Decisions

| Decision | Rationale |
|---------|-----------|
| Shadow deployment before promotion | New models receive production traffic without affecting the user experience; online evaluation eliminates the gap between offline metrics and production behavior |
| Feature store separation (online vs offline) | Online serving requires sub-10ms reads; offline training requires full-scan throughput; a single store cannot optimize for both |
| Versioned model artifacts in a central registry | Enables reproducible rollbacks, audit trails, and A/B comparisons between model versions |
| Drift-based automatic rollback | Model performance degrades gradually; automated drift detection catches regressions before they impact subscriber metrics |
| Training reproducibility requirements | Every training run records its data version, code version, and hyperparameters; any run can be reproduced exactly |

---

## Pros / Cons

### Pros
- Shadow deployment provides production-fidelity validation that offline evaluation cannot replicate
- Centralized feature store eliminates feature duplication and ensures consistent feature definitions across teams
- Versioned model registry enables rollback to any previous model version within minutes
- Drift monitoring catches model performance degradation before it becomes a subscriber experience issue

### Cons
- Shadow deployment increases inference fleet cost proportionally to the shadow traffic percentage
- Feature store consistency guarantees add latency to online feature writes; teams must accept bounded staleness
- Centralized platform creates a dependency: teams must align with platform deployment schedules and cannot operate independently
- Drift detection thresholds require careful calibration; overly sensitive thresholds trigger unnecessary rollbacks, while insensitive thresholds miss real regressions

---

## Failure Modes and Mitigation

| Failure | Impact | Mitigation |
|---------|--------|-----------|
| Model serving latency spike | Downstream latency increase for personalization and other consumers | Auto-scaling; load shedding with stale prediction fallback; circuit breaker to cached predictions |
| Feature staleness | Models receiving stale features; prediction quality degrades | Feature freshness SLA monitoring; fallback to population-level feature defaults on stale partition detection |
| Training pipeline failure | No new model versions produced during the outage window | Retry with idempotent training jobs; alert the owning team; existing production model remains active |
| Model drift detected | Predictions degrading; downstream business metric impact | Automatic rollback to the previous model version; drift alert sent to model owner |
| Feature store outage | Inference falls back to cached or default features | Stale feature TTL cache in the inference fleet; graceful degradation to fallback predictions |
| Shadow deployment resource exhaustion | Unable to run shadow deployments simultaneously | Priority queue for shadow deployments; lower-priority experiments deferred |

---

## Service Level Objectives

| Metric | Target | Window |
|--------|--------|--------|
| Online feature store latency (p99) | <= 10 ms | 30-day rolling |
| Inference fleet latency (p99) | <= 50 ms | 30-day rolling |
| Inference fleet availability | >= 99.99% | 30-day rolling |
| Feature freshness (online store) | <= 5 minutes | 24-hour rolling |
| Training pipeline success rate | >= 99% of scheduled runs | 30-day rolling |
| Model deployment lead time | Model registry to production <= 4 hours | Per-deployment measurement |

---

## Security Considerations

- Model artifacts are signed and checksummed in the model registry; tampering is detectable before deployment
- Training data access is governed by the data platform's role-based access control; models cannot access data outside their authorized scope
- Inference endpoints use mTLS with service identity certificates; unauthenticated inference requests are rejected
- PII features are prohibited in models without explicit data governance approval and are enforced at the feature store access layer
- Audit logs record every model artifact access, training run, and production promotion event

---

## Operational Artifacts

- `decisions/adr-0001-shadow-model-deployment.md`
- `decisions/adr-0002-feature-store-architecture.md`
- `runbooks/model-drift-response.md`
- `runbooks/training-pipeline-failure.md`
- `slos/slos.md`
