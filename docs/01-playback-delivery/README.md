# Playback Delivery Platform

## Purpose
Deliver globally low-latency, high-availability streaming with predictable startup times and minimal buffering.

## User Journey
Launch App → Request Playback Session → Receive Signed Manifest → Fetch Segments from CDN → Stream → Send QoE Telemetry

## Architecture
See diagrams:
- playback-sequence.mmd
- multi-cdn-routing.mmd

## Design Decisions
- Multi-CDN with real-time steering
- Tokenized manifests & segments
- Origin shielding & cache hierarchy
- Closed-loop QoE optimization

## Pros / Cons

Pros:
- Extremely fast startup
- High resilience
- Vendor risk reduction

Cons:
- Operational complexity
- Cache fragmentation risks

## Failure Modes & Mitigation

| Failure | Mitigation |
|--------|------------|
| CDN outage | Automatic traffic shift |
| Origin overload | Shield + request collapsing |
| Regional outage | Global failover |

## SLOs
TTFF ≤ 1.5s  
Rebuffer Ratio ≤ 0.3%  
Playback Error Rate ≤ 0.1%

## Security
- Signed URLs
- DRM enforcement
- Zero Trust service auth

## Artifacts
- ADR: adr-0001-multi-cdn-steering.md
- Runbook: cdn-degradation.md
