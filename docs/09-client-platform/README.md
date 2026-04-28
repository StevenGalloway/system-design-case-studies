# Client Platform and Device Architecture

## Purpose

The Client Platform encompasses the software systems running on every device that plays Netflix content: smart TVs, mobile phones, tablets, web browsers, gaming consoles, and partner-embedded devices. It is responsible for adaptive bitrate streaming, device-specific performance optimization, offline download management, and the secure communication protocols that connect the player to the backend.

The client is the last mile of every architectural decision made in the platform. Poor client architecture degrades the subscriber experience even when backend infrastructure is perfectly healthy.

---

## Device Ecosystem

| Device Category | Platforms | Approximate Share |
|----------------|-----------|------------------|
| Smart TVs | Tizen, WebOS, Android TV, Fire TV, Roku | ~35% of streaming hours |
| Mobile | iOS, Android | ~25% of streaming hours |
| Web Browser | Chrome, Firefox, Safari, Edge | ~15% of streaming hours |
| Gaming Consoles | PlayStation, Xbox | ~10% of streaming hours |
| Partner Devices | Set-top boxes, blu-ray players | ~15% of streaming hours |

Each device category has distinct constraints: codec support, DRM stack (Widevine, FairPlay, PlayReady), memory limits, and network characteristics require per-platform optimization.

---

## Core Subsystems

| Subsystem | Purpose |
|-----------|---------|
| ABR Engine | Selects the optimal bitrate segment at each download interval based on network conditions and buffer state |
| Player Engine | Decodes and renders video segments; manages buffer lifecycle |
| Playback Session Manager | Coordinates the session initiation flow: auth, manifest fetch, DRM license, and first segment delivery |
| Offline Download Manager | Manages download scheduling, storage, license pre-fetching, and local playback for offline-enabled content |
| Client Telemetry SDK | Collects and buffers QoE events; reports to the telemetry ingest pipeline |
| Remote Configuration | Receives and applies server-side configuration updates without requiring an app update |
| DRM Stack Integration | Interfaces with the device-native DRM (Widevine, FairPlay, PlayReady) to request and manage playback licenses |

---

## Architecture Diagrams

- `diagrams/abr-state-machine.mmd` - ABR algorithm state transitions
- `diagrams/offline-download-flow.mmd` - offline download request through local playback sequence

---

## Key Design Decisions

| Decision | Rationale |
|---------|-----------|
| BOLA-based ABR algorithm | Buffer-occupancy-based ABR outperforms throughput-based algorithms in network variability and reduces rebuffering without sacrificing quality |
| Remote configuration (server-driven client behavior) | Allows ABR parameters, bitrate caps, and feature flags to be updated without an app release cycle |
| Offline download with pre-fetched licenses | DRM licenses are fetched at download time and stored on-device, enabling fully offline playback without a backend connection |
| Client-side telemetry buffering | Events are batched and transmitted to avoid network overhead on constrained connections; local buffer prevents event loss during connectivity gaps |
| Device capability negotiation at session start | The client declares its codec, resolution, HDR, and DRM capabilities at session initiation; the server selects the optimal manifest variant |

---

## Adaptive Bitrate Strategy

The ABR engine makes segment download decisions based on two primary inputs:

1. **Buffer occupancy:** How many seconds of video are currently buffered and ready to play
2. **Network throughput estimate:** Measured from recent segment download throughput

The BOLA (Buffer Occupancy-based Lyapunov Algorithm) approach prioritizes buffer health over instantaneous quality maximization. When the buffer is healthy, BOLA selects higher bitrates. When the buffer is depleted or recovering, BOLA selects lower bitrates aggressively to prevent a rebuffering event.

This produces a smoother quality trajectory under variable network conditions compared to pure throughput-based algorithms, which tend to overshoot on quality selection and cause cascading rebuffering.

---

## Pros / Cons

### Pros
- BOLA reduces rebuffering events relative to throughput-based ABR under real-world network variability
- Remote configuration allows rapid response to network conditions or CDN issues without waiting for client release cycles
- Offline download enables engagement in low-connectivity environments; critical for mobile subscribers in markets with unreliable networks
- Client telemetry is the primary source of truth for QoE measurement; rich device-side signals enable root cause analysis that server-side data cannot provide

### Cons
- Supporting 20+ device platforms creates significant per-platform engineering overhead; codec and DRM fragmentation across platforms is a persistent operational challenge
- ABR algorithms require continuous tuning; what performs well in a lab environment may behave differently in real-world network topologies
- Offline license management is complex: licenses expire, content is removed from the catalog, and device storage management requires graceful handling of edge cases
- Client telemetry completeness varies by device; resource-constrained devices may drop events under battery or memory pressure

---

## Failure Modes and Mitigation

| Failure | Impact | Mitigation |
|---------|--------|-----------|
| ABR stuck at low bitrate | Subscriber watching at unnecessarily low quality | Remote configuration cap override; ABR parameter adjustment via server-driven config |
| Rebuffering storm | Widespread rebuffering in a region or network | CDN traffic shift; bitrate cap issued via remote configuration; CDN PoP adjustment |
| Offline license expiry during playback | Offline content becomes unplayable | License expiry is surfaced in the UI before expiry; licenses are auto-renewed when connectivity is available |
| Client crash | Session terminates unexpectedly | Client crash reporting pipeline; crash correlation with device type and app version; forced upgrade for crash-rate hotspots |
| DRM license handshake failure | Playback blocked at startup | Retry with exponential backoff; fall back to lower security level if content policy permits |
| Telemetry SDK failure | Loss of QoE visibility for affected devices | Synthetic monitoring fills coverage gap; telemetry recovery is tracked in the client release pipeline |

---

## Service Level Objectives

| Metric | Target | Window |
|--------|--------|--------|
| Client crash-free session rate | >= 99.9% | 30-day rolling |
| ABR rebuffer rate | <= 0.3% of watch time | 30-day rolling |
| Offline playback success rate | >= 99.5% of offline play attempts | 30-day rolling |
| Client telemetry delivery rate | >= 99.5% of expected events | 24-hour rolling |
| Session startup success rate | >= 99.9% of play attempts | 30-day rolling |

---

## Security Considerations

- DRM stack integration uses device attestation to bind licenses to the specific device hardware
- Client application binary integrity is validated at launch; tampering attempts are detected and blocked
- All client-to-server communication uses TLS 1.3 with certificate pinning on mobile platforms
- Offline content is stored encrypted using device-specific keys; content cannot be copied between devices
- Client telemetry does not include PII; device identifiers are pseudonymized before transmission

---

## Operational Artifacts

- `decisions/adr-0001-bola-abr-algorithm.md`
- `decisions/adr-0002-offline-download-manager.md`
- `runbooks/client-playback-failure.md`
- `slos/slos.md`
