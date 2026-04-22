# ADR-0002: VMAF as the Primary Quality Optimization Metric

## Status
Accepted

## Context

Video quality assessment requires a metric that correlates with human perceptual experience. Traditional metrics used in encoding pipelines include:

- **PSNR (Peak Signal-to-Noise Ratio):** Measures pixel-level distortion. Fast to compute but poorly correlated with human perception; a blurry image can score well on PSNR if the blur is uniform.
- **SSIM (Structural Similarity Index):** Captures structural distortion better than PSNR, but still does not accurately model the human visual system's sensitivity to specific types of artifacts (blocking, ringing, banding).

Netflix's encoding team needed a metric that could reliably predict whether a human viewer would prefer one encode over another, enabling automated quality gates to replace or augment manual review.

Research into human perception-based video quality metrics produced VMAF (Video Multi-Method Assessment Fusion), developed by Netflix and open-sourced. VMAF trains a machine learning model on human quality ratings to fuse multiple elementary quality features into a single score that closely tracks subjective viewer preference.

## Decision

Adopt VMAF as the primary metric for:

1. **Quality validation gates:** All encode outputs must meet minimum VMAF score thresholds before being accepted into the content artifact store. Encodes that fail the gate are rejected and the encoding job is flagged for investigation.

2. **Per-title bitrate ladder optimization:** The per-title optimizer uses VMAF as the quality objective function when deriving the optimal bitrate for each resolution tier. The goal is to maximize VMAF score for a given bitrate target.

3. **Codec comparison:** When evaluating codec upgrades (e.g., migrating a resolution tier from H.264 to H.265 or AV1), VMAF scores are used to confirm the new codec delivers equal or better perceptual quality at the same or lower bitrate.

VMAF thresholds:
- 1080p content: minimum VMAF score >= 90
- 4K HDR content: minimum VMAF score >= 85 (acceptable range is lower due to source material characteristics and HDR tone mapping complexity)
- 720p and below: minimum VMAF score >= 88

## Consequences

### Positive
- Encode quality gates are objective and automated; manual review is reserved for edge cases flagged by the automation
- Per-title optimization has a clear objective function; the optimizer can be tuned and validated against ground truth
- VMAF is open-source and the model weights can be updated as perceptual research advances

### Negative
- VMAF computation is significantly more expensive than PSNR or SSIM; it adds compute time to the post-encode validation step
- VMAF models were trained on a specific distribution of content and viewer panels; they may not perfectly predict quality for edge cases (animation, screen-capture content, medical imaging footage)
- VMAF does not capture all quality dimensions: audio-visual synchronization, color accuracy, and HDR tone mapping quality require supplementary assessment methods

## Alternatives Considered

**PSNR only:** Fast and simple, but poorly correlated with human quality perception. Rejected due to known failure modes on artifacts that are perceptually objectionable but numerically acceptable.

**Manual quality review for all encodes:** Provides human-validated quality assurance but does not scale to the catalog size and encoding throughput required.

**SSIM + PSNR ensemble:** Better than PSNR alone, but still inferior to VMAF in human correlation studies. Using both adds compute without matching VMAF's predictive accuracy.
