# Video Encoding Platform Service Level Objectives

## SLI Definitions

| Signal | SLI | Measurement Method |
|--------|-----|-------------------|
| Encoding Success Rate | Percentage of encoding jobs that complete without terminal failure | Orchestrator job completion metrics |
| Encode Quality (VMAF) | VMAF score of all encodes that pass the quality gate | Post-encode VMAF scoring |
| Time to First Encode | Time from mezzanine ingest acceptance to first successfully encoded and packaged rendition | Orchestrator timestamp correlation |
| Full Catalog Publish Time | Time from mezzanine ingest to all renditions published and available for playback | Orchestrator + CDN pre-warming timestamps |
| Worker Availability | Percentage of time the encoding worker pool is healthy and accepting jobs | Worker health check monitoring |
| VMAF Gate Pass Rate | Percentage of encode outputs that pass the VMAF quality gate without manual override | Orchestrator quality gate metrics |

---

## SLO Targets

| SLI | Target | Measurement Window |
|-----|--------|-------------------|
| Encoding success rate | >= 99.9% | 30-day rolling |
| VMAF score (1080p encodes) | >= 90 for all published titles | Per-title at publish |
| VMAF score (4K HDR encodes) | >= 85 for all published titles | Per-title at publish |
| Time to first encode (standard title) | <= 24 hours from mezzanine accepted | Per-title measurement |
| Full catalog publish time (standard title) | <= 48 hours from mezzanine accepted | Per-title measurement |
| Worker pool availability | >= 99.5% | 30-day rolling |
| VMAF gate pass rate | >= 99.0% | 30-day rolling |

---

## Release Window SLOs

Encoding SLOs tighten during planned release windows (major title launches):

| SLI | Standard Target | Release Window Target |
|-----|----------------|----------------------|
| Time to first encode | <= 24 hours | <= 8 hours |
| Full catalog publish time | <= 48 hours | <= 24 hours |
| Worker pool availability | >= 99.5% | >= 99.9% |

Release windows are declared in the content schedule at least 72 hours in advance, triggering burst capacity reservation.

---

## Error Budget Policy

| Error Budget Remaining | Action |
|-----------------------|--------|
| > 50% | Normal pipeline operations; configuration changes permitted |
| 25% - 50% | Configuration changes require encoding team lead review |
| < 25% | All pipeline changes frozen; on-call review required |
| 0% (exhausted) | Incident declared; encoding platform lead notified |

---

## Alerting Thresholds

| Alert | Threshold | Severity | Responder |
|-------|-----------|----------|-----------|
| Encoding job failure rate | > 0.5% over 30 min | SEV-2 | Content Engineering SRE |
| Worker pool availability | < 90% | SEV-2 | Content Engineering SRE |
| VMAF gate rejection rate | > 2% over 1 hour | SEV-2 | Encoding Quality Team |
| Queue depth growth | > 50 jobs stacking for > 15 min | SEV-2 | Content Engineering SRE |
| Release window title at risk | Any title missing ETA | SEV-1 | Encoding Platform Lead |

---

## Exclusions

- Mezzanine files rejected at ingest due to source quality issues (SLO accountability transfers to the content supply chain)
- VMAF gate failures caused by source mezzanine quality, not encoding pipeline quality
- Encoding delays caused by studio delivery latency outside the declared mezzanine acceptance window
