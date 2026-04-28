# Client Platform Service Level Objectives

## SLI Definitions

| Signal | SLI | Measurement Method |
|--------|-----|-------------------|
| Session Startup Success Rate | Percentage of play attempts that successfully start playback | Client-side telemetry: play_attempt vs play_start events |
| Crash-Free Session Rate | Percentage of sessions that complete without a client-side crash | Client crash reporting SDK |
| ABR Rebuffer Rate | Percentage of total watch time spent rebuffering | Client-side buffer state telemetry |
| Offline Playback Success Rate | Percentage of offline play attempts that succeed | Client-side offline playback telemetry |
| Client Telemetry Delivery Rate | Percentage of generated telemetry events received by the ingest pipeline | Event count reconciliation |

---

## SLO Targets

| SLI | Target | Measurement Window | Error Budget (30-day) |
|-----|--------|-------------------|----------------------|
| Session startup success rate | >= 99.9% | 30-day rolling | 0.1% of play attempts |
| Crash-free session rate | >= 99.9% | 30-day rolling | 0.1% of sessions |
| ABR rebuffer rate | <= 0.3% of watch time | 30-day rolling | 0.3% average |
| Offline playback success rate | >= 99.5% | 30-day rolling | 0.5% of offline attempts |
| Client telemetry delivery rate | >= 99.5% | 24-hour rolling | 0.5% event loss |

---

## Per-Platform SLO Targets

Some device platforms have different baseline characteristics due to hardware constraints:

| Platform | Session Startup SLO | Rebuffer SLO | Notes |
|---------|--------------------|-----------|----|
| Smart TV | >= 99.95% | <= 0.2% | High-power devices; tighter targets |
| iOS / Android (mobile) | >= 99.9% | <= 0.4% | Network variability is higher on mobile |
| Web Browser | >= 99.85% | <= 0.35% | Browser codec support variability |
| Gaming Console | >= 99.9% | <= 0.25% | Stable network connection typical |
| Partner Devices | >= 99.5% | <= 0.5% | Hardware heterogeneity; relaxed baseline |

---

## Error Budget Policy

| Error Budget Remaining | Action |
|-----------------------|--------|
| > 50% | Normal client release operations permitted |
| 25% - 50% | New app releases require QA sign-off and phased rollout |
| 10% - 25% | All client releases frozen; on-call review required |
| < 10% | Releases halted; client platform lead notified |
| 0% (exhausted) | Incident declared; mandatory postmortem |

---

## Alerting Thresholds

| Alert | Threshold | Severity | Responder |
|-------|-----------|----------|-----------|
| Playback error rate (global) | > 0.1% for 3 min | SEV-2 | Client SRE |
| Playback error rate (single platform) | > 1% for 3 min | SEV-2 | Client SRE |
| Crash rate spike (any app version) | > 0.5% crash rate for 5 min | SEV-2 | Client SRE |
| Offline playback failure rate | > 2% for 10 min | SEV-2 | Client SRE |
| Telemetry delivery drop | < 98% in 15-min window | SEV-3 | Observability team |

---

## Exclusions

- Playback failures caused by user-initiated interruptions (force-close, network airplane mode)
- Failures on unsupported OS versions or app versions older than the minimum supported version
- Device hardware failures (insufficient storage for download, GPU decode errors on end-of-life hardware)
- Third-party DRM platform outages (Widevine, FairPlay) where root cause is confirmed to be the DRM provider
