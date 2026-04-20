# ADR-0003: Data Mesh Ownership Model

## Status
Accepted

## Context

As the platform scaled, centralized data engineering became a bottleneck. A small central team was responsible for building and maintaining pipelines on behalf of all product and engineering domains. This created several structural problems:

- Pipeline development was slow: business teams waited weeks for new data products
- Quality issues were often discovered by downstream consumers, not by the team that produced the data
- The central team lacked domain context needed to understand whether a metric was correct
- Ownership of data quality was ambiguous: producers blamed the pipeline, pipeline team blamed the schema, consumers blamed the source
- The team's operational load scaled with the number of pipelines, not the number of engineers

The data platform needed an ownership model that decentralized responsibility for data quality to the teams with the most domain context, while the central team focused on platform infrastructure.

## Decision

Adopt a Data Mesh model where domain teams own their data products end-to-end.

**Domain team responsibilities:**
- Define and version the schema of their output datasets
- Maintain schema contracts and notify downstream consumers of breaking changes with >= 2 sprint lead time
- Monitor data quality and freshness SLAs for their data products
- Respond to data quality incidents for data they produce
- Publish metadata (ownership, schema, SLA, lineage) to the central data catalog

**Central platform team responsibilities:**
- Operate Kafka, Flink, Spark, and Iceberg infrastructure
- Provide self-service tooling for schema registration, pipeline deployment, and SLA monitoring
- Define and enforce platform-level standards (schema registry, data classification, access control)
- Provide cost attribution by domain to enable teams to optimize their pipeline footprint

**Data product contract requirements (enforced at publish time):**
- Schema registered in the platform schema registry with semantic versioning
- Declared owner (team + individual DRI)
- Declared SLA (freshness and completeness targets)
- Lineage documented (upstream datasets this product depends on)

## Consequences

### Positive
- Data quality ownership moves to the team with the most context; incidents resolve faster
- Pipeline development velocity increases because domain teams are not blocked on the central team
- Platform team can focus on infrastructure improvements rather than per-domain pipeline work
- Schema contracts make breaking changes explicit and prevent surprise downstream failures

### Negative
- Domain teams need baseline data engineering skills; the platform team must provide training and templates to lower the barrier
- Without active governance enforcement, data mesh can degrade into data swamp: undiscoverable datasets with no clear owner
- Central data catalog must be actively maintained; stale catalog entries are worse than no catalog

## Alternatives Considered

**Centralized data engineering:** High consistency and quality enforcement, but does not scale; creates bottlenecks and context loss as the number of domains grows.

**Self-serve pipelines with no standards:** Maximum team autonomy, but leads to duplicated infrastructure, inconsistent semantics across datasets, and no platform-level observability.
