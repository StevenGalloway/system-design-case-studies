# ADR-0001: BOLA-Based Adaptive Bitrate Algorithm

## Status
Accepted

## Context

Adaptive bitrate (ABR) streaming allows the client player to switch between multiple encoded quality levels during playback based on current network conditions. The choice of ABR algorithm directly determines the tradeoff between playback quality, rebuffering frequency, and quality stability.

Two dominant families of ABR algorithms exist:

**Throughput-based algorithms:** Estimate available network bandwidth from recent segment download speeds and select the highest bitrate that fits within that estimate. These algorithms react quickly to improving conditions but tend to overshoot on quality selection, leading to rebuffering when throughput estimates are optimistic (common in bursty mobile networks).

**Buffer-based algorithms:** Select bitrate based on the current playback buffer occupancy rather than throughput estimates. When the buffer is full, select higher quality. When the buffer is draining, select lower quality. These algorithms are more conservative but produce fewer rebuffering events under variable network conditions.

The previous throughput-based ABR algorithm in production was producing acceptable average quality but had elevated rebuffering rates in mobile network conditions, where throughput can vary by an order of magnitude within seconds.

## Decision

Adopt BOLA (Buffer Occupancy-based Lyapunov Algorithm) as the primary ABR algorithm across all client platforms.

BOLA uses a utility function grounded in Lyapunov optimization theory to select the bitrate that maximizes quality utility over time while maintaining buffer occupancy above a target threshold. The algorithm does not require accurate throughput prediction; it makes decisions based entirely on buffer state.

Key configuration parameters (tunable via remote configuration):
- `buffer_target_seconds`: target buffer occupancy level that triggers quality increases (default: 12 seconds)
- `buffer_minimum_seconds`: buffer level below which quality is reduced aggressively (default: 4 seconds)
- `quality_utility_weights`: per-bitrate utility values, tunable to reflect the platform's quality ladder

All BOLA parameters are served via the Remote Configuration Service, enabling rapid adjustment without a client release.

## Consequences

### Positive
- Reduced rebuffering events in mobile network conditions where throughput estimation is unreliable
- Quality trajectory is smoother (fewer upward/downward switches) compared to aggressive throughput-based algorithms
- Buffer-based decisions are more predictable and easier to reason about during incident debugging
- Algorithm parameters can be tuned per device type, region, and network condition without a client release

### Negative
- BOLA is conservative by design; under ideal network conditions, it may not select the highest available quality as quickly as a throughput-based algorithm
- Buffer-based algorithms are less responsive to sudden network improvements; the system waits for the buffer to recover before increasing quality
- Requires careful parameter tuning per device class; a single global configuration does not optimize all scenarios

## Alternatives Considered

**Throughput-based ABR (MPC / FESTIVE):** Maximizes quality under stable network conditions but produces higher rebuffering rates under variability. Not well-suited for mobile subscribers as a primary algorithm.

**Hybrid throughput + buffer (Pensieve / reinforcement learning):** ML-trained ABR shows promise in research environments but introduces inference overhead on resource-constrained devices and requires continuous model training and deployment infrastructure.
