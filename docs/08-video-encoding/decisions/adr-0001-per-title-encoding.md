# ADR-0001: Per-Title Encoding Strategy

## Status
Accepted

## Context

Traditional video encoding pipelines use a fixed, universal bitrate ladder: a predetermined set of resolution and bitrate combinations applied identically to all titles. For example, every 1080p encode targets 4,000 kbps regardless of whether the content is a simple talk show or a visually complex action film.

This approach has a fundamental mismatch with reality. A simple documentary with static shots can achieve excellent perceptual quality at 1,500 kbps. A complex action sequence with rapid motion and detailed backgrounds may require 6,000 kbps to maintain the same perceived quality. Applying the same fixed ladder to both wastes bandwidth on the documentary and delivers a subpar experience on the action film.

At Netflix's scale, this inefficiency is significant: even a 10% reduction in average streaming bitrate translates to a meaningful reduction in CDN costs and a corresponding improvement in quality for subscribers on constrained networks.

The encoding team needed a way to make the bitrate ladder data-driven, specific to each title's visual characteristics.

## Decision

Adopt per-title encoding, where the bitrate ladder for each title is derived from a machine-learning-based analysis of the title's visual complexity.

The per-title optimizer analyzes the mezzanine file before encoding begins. It segments the video into scene groups and computes spatial and temporal complexity scores (pixel variance, motion magnitude, edge density). These scores are fed into a model that predicts the quality-versus-bitrate curve for this specific title, and a custom bitrate ladder is generated to maximize VMAF quality at each bandwidth tier.

The default fixed bitrate ladder is retained as a fallback for cases where the per-title optimizer fails or produces an invalid result.

## Consequences

### Positive
- Measurable improvement in VMAF quality at equivalent bitrates compared to a fixed ladder; subscribers on constrained connections receive better quality
- CDN bandwidth savings: simpler content is encoded at lower bitrates without perceptible quality loss
- The per-title approach can adopt new codecs incrementally: per-title analysis can produce separate ladders for H.264, H.265, and AV1 using the same complexity inputs

### Negative
- Each title requires a complexity analysis pass before encoding can begin, adding time to the pipeline and requiring additional compute
- Per-title ladders make cross-title quality comparison harder: two titles at the same bitrate may look very different because they have different complexity profiles
- The complexity analysis model must be maintained and validated; model drift or incorrect scene segmentation can produce suboptimal ladders

## Alternatives Considered

**Fixed bitrate ladder with per-resolution tuning:** Simpler to operate and reason about, but does not account for title-specific visual characteristics. Quality and bandwidth efficiency are left on the table.

**Per-scene encoding (variable bitrate within a title):** More granular than per-title, but multiplies the number of encode decisions and the complexity of the ABR player's segment selection logic significantly. Per-title is a practical midpoint.
