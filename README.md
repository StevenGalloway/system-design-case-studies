# Netflix Architecture Case Study

This repository presents reference architectures for a large-scale streaming platform modeled after Netflix. It is designed to demonstrate real-world Solutions Architect thinking across the full breadth of systems that power a global streaming service.

The documentation follows the conventions used in production engineering organizations: Architecture Decision Records (ADRs) capture design rationale, runbooks provide actionable operational procedures, threat models surface security considerations, and SLOs define measurable reliability targets.

---

## What This Repository Demonstrates

- Tradeoff analysis across distributed system design decisions
- Architecture Decision Records with context, rationale, and consequences
- SLO-driven reliability engineering with error budget policies
- Operational runbooks written for real incident response
- Security-first design through Zero Trust and STRIDE-based threat modeling
- Resilience engineering through cell architecture, chaos testing, and progressive delivery
- ML platform design covering the full model lifecycle
- Data platform design from raw event streams to analytical and ML-ready datasets
- Client-side architecture from ABR algorithms to offline download management

---

## Core Design Principles

| Principle | Description |
|-----------|-------------|
| Separation of control plane and data plane | Playback reliability is never compromised by changes in personalization, experimentation, or configuration systems |
| Multi-CDN and global traffic steering | No single CDN provider is a point of failure; real-time steering responds to degradation automatically |
| Cell-based architecture and blast radius containment | Failures are bounded to isolated cells; no single failure can cascade globally |
| SLO-driven reliability engineering | Error budgets govern deployment risk; SLOs are not aspirational, they are operational contracts |
| Zero Trust security | No implicit trust between services; mutual TLS, short-lived credentials, and explicit authorization for all calls |
| Progressive delivery and safe rollouts | Canary analysis gates every deployment; automatic rollback is the first line of defense against regressions |
| Graceful degradation with playback first | Non-critical systems fail open; playback continues even when personalization, recommendations, or experiments are unavailable |
| Event-driven content supply chain | Parallelism, fault isolation, and idempotent retries enable massive scale and high release velocity |
| Data-driven personalization and experimentation | Every product decision is validated through controlled experiments with automated guardrails |
| Chaos engineering and continuous resilience validation | The system is exercised against failure continuously; confidence in resilience comes from practice, not assumptions |

---

## Architecture Domains

| # | Domain | Description |
|---|--------|-------------|
| 01 | [Playback Delivery](docs/01-playback-delivery/) | Global low-latency streaming, CDN steering, QoE optimization |
| 02 | [Content Supply Chain](docs/02-content-supply-chain/) | Ingest, encode, package, and publish pipeline |
| 03 | [Identity, Entitlements & DRM](docs/03-identity-entitlements-drm/) | Authentication, authorization, and content protection |
| 04 | [Personalization & Control Plane](docs/04-personalization-control-plane/) | Recommendations, ML serving, and experimentation |
| 05 | [Global Resilience](docs/05-global-resilience/) | Multi-region fault tolerance, chaos testing, and disaster recovery |
| 06 | [Observability & QoE](docs/06-observability-qoe/) | Telemetry pipelines, SLO monitoring, and incident response |
| 07 | [Data Platform](docs/07-data-platform/) | Kafka, Flink, Spark, Iceberg lakehouse, and feature store infrastructure |
| 08 | [Video Encoding](docs/08-video-encoding/) | Per-title encoding, VMAF quality optimization, and codec strategy |
| 09 | [Client Platform](docs/09-client-platform/) | ABR streaming, offline downloads, and device architecture |
| 10 | [MLOps Platform](docs/10-mlops-platform/) | Model training, shadow deployment, drift monitoring, and inference infrastructure |

---

## Repository Structure

Each domain folder contains the following artifacts, depending on domain scope:

```
docs/<domain>/
├── README.md                 Domain overview, design decisions, SLOs, failure modes
├── decisions/                Architecture Decision Records (ADRs)
│   └── adr-NNNN-<topic>.md  Context, decision, rationale, and consequences
├── diagrams/                 Mermaid diagrams for key flows and topologies
│   └── <name>.mmd
├── runbooks/                 Operational response procedures for on-call engineers
│   └── <incident-type>.md
├── slos/                     SLI definitions, SLO targets, and error budget policy
│   └── slos.md
└── threat-model/             STRIDE-based threat analysis and abuse case documentation
    └── threat-model-<domain>.md
```

---

## Architecture Decision Records

ADRs capture the context behind each significant architectural choice. They are written for the audience that will inherit and maintain the system, not just the audience that participated in the original decision.

Each ADR follows the structure: **Status | Context | Decision | Consequences | Alternatives Considered**

| Domain | ADR | Decision |
|--------|-----|---------|
| Playback Delivery | ADR-0001 | Multi-CDN active steering over primary/failover CDN |
| Content Supply Chain | ADR-0001 | Event-driven pipeline architecture |
| Content Supply Chain | ADR-0002 | Immutable content artifacts for auditability |
| Content Supply Chain | ADR-0003 | Policy-based publishing for safe global releases |
| Identity & DRM | ADR-0001 | Zero Trust service architecture |
| Identity & DRM | ADR-0002 | Short-lived playback tokens |
| Personalization | ADR-0001 | Control plane isolation from playback data plane |
| Personalization | ADR-0002 | Graceful degradation with static fallbacks |
| Data Platform | ADR-0001 | Apache Iceberg as the lakehouse table format |
| Data Platform | ADR-0002 | Dual-speed pipeline (streaming + batch) |
| Data Platform | ADR-0003 | Data mesh ownership model |
| Video Encoding | ADR-0001 | Per-title encoding strategy |
| Video Encoding | ADR-0002 | VMAF as the primary quality optimization metric |
| Client Platform | ADR-0001 | BOLA buffer-based ABR algorithm |
| Client Platform | ADR-0002 | Offline download manager with pre-fetched DRM licenses |
| MLOps Platform | ADR-0001 | Shadow deployment for model validation |
| MLOps Platform | ADR-0002 | Unified feature store architecture |
