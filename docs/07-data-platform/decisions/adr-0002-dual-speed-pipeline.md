# ADR-0002: Dual-Speed Data Pipeline Architecture

## Status
Accepted

## Context

Different consumers of the data platform have fundamentally different latency requirements:

- The **Personalization system** needs feature values refreshed within seconds of user behavior to power accurate homepage recommendations during a browsing session
- The **Alerting system** needs anomaly detection on QoE signals within 60 seconds to meet SLO alert latency targets
- The **Analytics team** needs accurate aggregations over large historical windows for A/B test analysis, business reporting, and ML training datasets
- The **ML training pipeline** needs complete, validated, and deduplicated datasets that may span months of history

These requirements cannot be served well by a single processing model. A pure streaming architecture (Flink-only) can handle real-time needs but is expensive at the scale needed for historical analytical queries. A pure batch architecture (Spark-only) has the throughput for historical workloads but cannot deliver sub-minute latency for real-time consumers.

## Decision

Operate a dual-speed data pipeline with clearly defined lanes for streaming and batch processing, sharing the same Kafka event bus as the source of truth.

**Streaming lane (Flink):**
- Consumes events from Kafka with end-to-end latency targets of <= 5 seconds
- Produces real-time feature updates to the online feature store
- Powers real-time anomaly detection and alerting
- Uses exactly-once semantics via Flink checkpointing + Iceberg transactional writes

**Batch lane (Spark):**
- Reads from Iceberg tables (materialized from the streaming lane or directly from Kafka via Iceberg streaming ingestion)
- Runs on a schedule (every 15 minutes to 24 hours depending on the dataset)
- Produces training datasets, historical aggregations, and compliance-grade reports
- Idempotent: re-running a Spark job for the same time window produces the same output

Both lanes write to Iceberg, providing a unified query layer for downstream consumers regardless of which lane produced the data.

## Consequences

### Positive
- Real-time and batch consumers are served with appropriate latency and cost characteristics
- Kafka acts as a durable replay buffer, enabling the batch lane to catch up after failures without data loss
- Streaming and batch outputs are both queryable via the same Iceberg tables, simplifying the consumer interface

### Negative
- Teams must understand which lane serves their use case; incorrect lane selection leads to either over-engineering (batch team using Flink) or unmet latency requirements (real-time team using Spark)
- Two compute frameworks (Flink and Spark) require distinct operational expertise, monitoring, and on-call coverage
- Lambda/Kappa architecture debates require clear governance: the platform defaults to Kappa (streaming-first) with Spark as the scheduled reprocessing layer, not a parallel computation path

## Alternatives Considered

**Streaming-only (Flink):** Eliminates operational complexity of running two frameworks, but makes large historical scans prohibitively expensive and complex in a streaming model.

**Batch-only (Spark):** Simpler operations, but cannot meet the sub-minute latency requirements of the alerting and online feature serving use cases.

**Separate systems per use case:** Each team builds their own pipeline, leading to duplicated infrastructure, inconsistent event semantics, and no shared governance model.
