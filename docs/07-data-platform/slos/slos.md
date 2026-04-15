# Data Platform Service Level Objectives

## SLI Definitions

| Signal | SLI | Measurement Method |
|--------|-----|-------------------|
| Event Delivery Latency | Time from event production to availability in the downstream consumer | Kafka timestamp correlation |
| Batch Pipeline Freshness | Age of the latest committed partition in a managed Iceberg table | Partition metadata timestamp |
| Online Feature Serving Latency | p99 read latency for feature key lookups | Feature store instrumentation |
| Platform Availability | Percentage of time Kafka brokers and Flink jobs are fully operational | Health check monitoring |
| Data Completeness | Percentage of produced events that land in the downstream store within the SLA window | Event count reconciliation |

---

## SLO Targets

| SLI | Target | Measurement Window | Error Budget (30-day) |
|-----|--------|-------------------|----------------------|
| Event delivery latency (p99) | <= 5 seconds | 24-hour rolling | 0.1% of events may exceed |
| Batch pipeline freshness (tier-1 tables) | <= 15 minutes | Daily measurement | 0 breaches per month |
| Batch pipeline freshness (tier-2 tables) | <= 60 minutes | Daily measurement | <= 2 breaches per month |
| Online feature store latency (p99) | <= 10 ms | 30-day rolling | 0.1% of requests may exceed |
| Kafka + Flink availability | >= 99.9% | 30-day rolling | 43.8 minutes downtime |
| Data completeness | >= 99.99% | 24-hour rolling | 0.01% event loss |

---

## Dataset Tier Classification

| Tier | Freshness SLA | Examples | On-Call Priority |
|------|--------------|----------|-----------------|
| Tier 1 (Real-time critical) | <= 5 seconds (streaming) | QoE signals, alerting feeds, online features | SEV-1 on breach |
| Tier 2 (Near-real-time) | <= 15 minutes | Personalization training signals, engagement metrics | SEV-2 on breach |
| Tier 3 (Analytical) | <= 60 minutes | Business reporting, A/B test results | SEV-3 on breach |
| Tier 4 (Archival) | <= 24 hours | Compliance logs, cold analytics | No on-call response |

---

## Error Budget Policy

| Error Budget Remaining | Action |
|-----------------------|--------|
| > 50% | Normal operations; schema and pipeline changes permitted |
| 25% - 50% | Changes require data platform SRE review |
| 10% - 25% | Pipeline changes frozen; investigation required |
| < 10% | All changes halted; leadership notified |
| 0% (exhausted) | Incident declared; mandatory postmortem |

---

## SLO Alerting

| Alert | Threshold | Severity | Responder |
|-------|-----------|----------|-----------|
| Kafka consumer lag (real-time group) | > 10,000 messages for 3 min | SEV-2 | Data Platform SRE |
| Flink job failure | Any job in FAILED state | SEV-2 | Data Platform SRE |
| Tier-1 table freshness breach | Partition age > 5 minutes | SEV-1 | Data Platform SRE + domain team |
| Tier-2 table freshness breach | Partition age > 15 minutes | SEV-2 | Data Platform SRE |
| Online feature latency p99 | > 10 ms for 5 min | SEV-2 | Data Platform SRE |
| Data completeness drop | < 99.99% in 15-min window | SEV-2 | Data Platform SRE |

---

## SLO Exclusions

- Planned Kafka maintenance windows with >= 48 hours advance notice
- Spark cluster cold-start time following major cluster upgrades
- Input data unavailability caused by upstream source system incidents (SLO accountability transfers to source)
- Force majeure cloud provider outages affecting >= 2 availability zones simultaneously
