# Video Encoding and Packaging Platform

## Purpose

The Video Encoding Platform transforms raw studio-delivered mezzanine files into the optimized, multi-bitrate, multi-codec video segments served to every device in the Netflix catalog.

Encoding quality directly impacts subscriber experience: a poorly optimized bitrate ladder wastes bandwidth and degrades picture quality simultaneously. This domain applies machine perception research at scale to ensure every title is encoded with the optimal quality-per-bit for its visual complexity.

---

## Core Subsystems

| Subsystem | Purpose |
|-----------|---------|
| Encoding Orchestrator | Manages job scheduling, worker allocation, and dependency tracking across the encoding pipeline |
| Per-Title Optimizer | Analyzes scene complexity to derive a custom bitrate ladder for each title |
| Encoding Workers | Horizontally scalable compute nodes running FFmpeg-based encoding with hardware acceleration |
| Quality Validation | VMAF-based perceptual quality scoring applied to every encode output |
| Packaging Service | Segments encoded streams into CMAF/DASH/HLS formats with DRM encryption |
| Codec Manager | Governs codec selection strategy (H.264, H.265, AV1) per device tier and content type |

---

## Architecture Diagrams

- `diagrams/encoding-pipeline.mmd` - end-to-end pipeline from mezzanine file to packaged segments
- `diagrams/per-title-optimizer.mmd` - per-title complexity analysis and bitrate ladder derivation flow

---

## Encoding Pipeline Flow

```
Mezzanine File (from Content Supply Chain)
       |
  Scene Complexity Analysis
       |
  Per-Title Bitrate Ladder Generation
       |
  Parallel Encoding (multiple codecs, multiple bitrates)
       |
  VMAF Quality Validation
       |
  Packaging + DRM Encryption (CMAF)
       |
  Content Artifact Store (Iceberg + Object Storage)
       |
  CDN Distribution
```

---

## Key Design Decisions

| Decision | Rationale |
|---------|-----------|
| Per-title encoding | Static bitrate ladders waste bits on simple scenes and starve complex scenes; per-title optimization maximizes quality at every bandwidth tier |
| VMAF as quality gate | VMAF correlates more strongly with human perceptual quality than PSNR or SSIM; it enables objective quality enforcement across codec generations |
| Multi-codec strategy | AV1 for new-generation devices reduces bandwidth by 30-50% vs H.264; maintaining H.264 support for older devices prevents catalog fragmentation |
| Idempotent encode jobs | Encoding workers can be safely retried or pre-empted; outputs are written atomically to object storage upon job completion |
| Hardware-accelerated encoding | GPU encoding accelerates throughput for H.265 and AV1 on high-resolution content; CPU encoding retained for codecs without mature GPU support |

---

## Bitrate Ladder Strategy

Netflix uses a per-title, per-resolution bitrate ladder derived from scene complexity analysis rather than a fixed target bitrate schedule.

| Resolution | Codec | Adaptive Bitrate Range |
|-----------|-------|----------------------|
| 240p | H.264 | 100 - 300 kbps |
| 480p | H.264 | 300 - 750 kbps |
| 720p | H.264 / H.265 | 1,000 - 2,500 kbps |
| 1080p | H.264 / H.265 | 2,000 - 5,800 kbps |
| 4K HDR | H.265 / AV1 | 7,000 - 16,000 kbps |

Actual bitrates for a given title are determined by the per-title optimizer based on spatial and temporal complexity scores computed from the mezzanine file.

---

## Pros / Cons

### Pros
- Measurable improvement in perceived quality at a given bandwidth compared to fixed-rung ladders
- Enables significant CDN bandwidth savings across the catalog without perceptible quality reduction
- VMAF quality gates prevent quality regressions from reaching subscribers
- Multi-codec strategy positions the platform to absorb next-generation codecs (AV1, VVC) incrementally

### Cons
- Per-title analysis adds compute time and cost to the encoding pipeline before encoding begins
- More complex quality control process: each title has a unique bitrate ladder, making cross-title quality comparison require normalization
- Multi-codec support multiplies the number of encoded artifacts per title (H.264 + H.265 + AV1), increasing storage costs
- AV1 encoding is significantly slower than H.264 and H.265 at equivalent hardware; requires careful job scheduling to meet release deadlines

---

## Failure Modes and Mitigation

| Failure | Impact | Mitigation |
|---------|--------|-----------|
| Encoding worker failure | Single encode job fails | Job retry with exponential backoff; work is reassigned to a healthy worker |
| Per-title optimizer failure | Title falls back to default bitrate ladder | Default ladder is maintained and kept current as a safe fallback |
| VMAF quality gate failure | Encode rejected; title not available | Alert encoding team; inspect source mezzanine for quality issues; allow manual override with senior review |
| Packaging failure | Encoded segments cannot be packaged | Packaging service retry; if persistent, inspect DRM key delivery to packaging service |
| Storage write failure | Encoding output lost | Encoding job is retriable; artifact store write is atomic via Iceberg; job is re-queued |
| Release deadline miss | Title unavailable at scheduled launch | Priority queue escalation; encode worker burst scaling; CDN pre-warming of partial catalog |

---

## Service Level Objectives

| Metric | Target | Window |
|--------|--------|--------|
| Encoding success rate | >= 99.9% | 30-day rolling |
| VMAF score (all published encodes) | >= 90 for 1080p; >= 85 for 4K HDR | Per-title at publish |
| Time from ingest to first encode complete | <= 24 hours for standard titles | Per-title measurement |
| Time from ingest to full catalog publish | <= 48 hours for standard titles | Per-title measurement |
| Encoding worker availability | >= 99.5% | 30-day rolling |

---

## Security Considerations

- DRM keys are injected at the packaging stage, not during encoding; encoding workers never have access to content encryption keys
- Encoding jobs run in isolated compute environments; cross-title data access is not possible by design
- VMAF output and bitrate ladder metadata are stored with the content artifact and are auditable
- Mezzanine files are stored encrypted at rest and are deleted from worker scratch storage after job completion

---

## Operational Artifacts

- `decisions/adr-0001-per-title-encoding.md`
- `decisions/adr-0002-vmaf-quality-optimization.md`
- `runbooks/encoding-failure.md`
- `runbooks/quality-regression.md`
- `slos/slos.md`
