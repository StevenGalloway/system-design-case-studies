# ADR-0002: Unified Feature Store Architecture

## Status
Accepted

## Context

Before a centralized feature store existed, feature engineering in the organization was fragmented:

- Each team maintained their own feature computation pipelines
- The same feature (e.g., "user's watch history in the last 7 days") was computed by multiple teams with slightly different definitions, causing inconsistency between models that should be using the same signal
- Features computed in the training pipeline used different data snapshots than features served at inference time, creating training-serving skew that degraded model quality in ways that were hard to diagnose
- Online inference required low-latency feature reads that batch pipelines could not serve; teams built ad-hoc caches that were not monitored or governed

Training-serving skew is one of the most common and costly sources of model quality degradation in production ML systems. The lack of a shared feature definition layer meant that two models training on the "same" feature could produce different results based on subtle differences in how each team computed it.

## Decision

Build a unified Feature Store with separate access patterns optimized for online serving and offline training, backed by a shared feature definition and computation layer.

**Feature definition layer:** All features are defined in a central feature catalog with a schema, computation logic, versioning, and an owner. Teams cannot use a feature in production without it being registered in the catalog. Feature definitions are code-reviewed and versioned like software.

**Offline store (Iceberg-backed):** Feature values are computed by the data platform's batch and streaming pipelines and written to Iceberg tables partitioned by entity (user, device, content) and time. Training jobs read features from the offline store using point-in-time correct joins to prevent data leakage.

**Online store (low-latency key-value store):** The most recent feature values for each entity are materialized from the offline store into an in-memory key-value store optimized for sub-10ms reads. The inference fleet reads from the online store during live inference.

**Point-in-time correctness for training:** Training datasets are constructed using point-in-time joins: for each training example at timestamp T, the feature values are fetched from the state they were in at time T, not the current state. This prevents future data leakage from corrupting training labels.

## Consequences

### Positive
- Eliminates training-serving skew: offline training and online inference read from the same computed feature definitions
- Feature reuse across models: once a feature is in the store, any authorized model can use it without recomputing it
- Point-in-time correct training datasets are a first-class operation, not a custom engineering effort for each team
- Feature catalog provides discoverability: teams can find existing features before deciding to compute new ones

### Negative
- Centralizing feature computation creates a platform dependency; model teams cannot self-serve features that are not in the catalog without platform team involvement
- Online store must maintain consistency with the offline store under a bounded staleness model; the operational complexity of the synchronization pipeline must be actively managed
- Feature catalog governance overhead: feature registration, review, and versioning adds process that teams must adopt

## Alternatives Considered

**Per-team feature pipelines with shared schema registry:** Reduces centralization overhead but does not solve training-serving skew; each team still manages their own compute pipelines.

**Real-time feature computation at inference time:** Eliminates the need for a feature store by computing features on demand during inference. Not feasible at the required inference latency targets; feature computation is too expensive to run in the inference hot path for complex features.
