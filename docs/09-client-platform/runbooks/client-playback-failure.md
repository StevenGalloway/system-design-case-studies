# Runbook: Client Playback Failure

**Severity:** SEV-2 (device-type or version-scoped) / SEV-1 (global client playback failure)
**On-Call Team:** Client Platform SRE, Playback Engineering
**Escalation:** Client Platform Engineering Lead, VP of Engineering

---

## Overview

Client playback failures occur when a significant percentage of play attempts fail to start or abort during playback on one or more device types. Because the client is the closest layer to the subscriber, client failures are immediately visible in QoE metrics and customer support volume.

---

## Detection

- Automated alert: Playback Error Rate exceeds 0.1% globally or >= 1% on a specific device segment for 3 consecutive minutes
- Automated alert: Session startup success rate drops below 99.9%
- Client crash rate alert: crash-free session rate drops below 99.5% for a specific app version
- Customer support spike: volume of playback failure contacts increases by > 50% above baseline
- Client telemetry alert: a specific error code appears with disproportionate frequency (e.g., DRM license failure, manifest parse error)

---

## Initial Assessment (< 10 minutes)

1. Open the **Client QoE Dashboard** and segment the failure rate by:
   - Device type / platform (smart TV, iOS, Android, web, console)
   - App version
   - Geographic region
   - Error code (manifest error, DRM error, network error, decode error)

2. Determine the scope:
   - Is this a new app version that was recently deployed?
   - Is it scoped to a specific device model or OS version?
   - Is the error code pointing to a backend service (DRM, manifest, CDN) rather than the client itself?

3. Check backend service health:
   - DRM License Service: is license issuance latency elevated?
   - Manifest Service: is manifest generation error rate elevated?
   - CDN: is the CDN degradation runbook more appropriate?

---

## Mitigation Steps

### App Version Rollback

If the failure is correlated with a specific recently deployed app version:

1. Evaluate the feasibility of a forced upgrade or server-side kill switch:
   - The Remote Configuration Service can block a specific app version from initiating playback and redirect users to the update prompt.
   - Coordinate with the app store release team to understand rollback options (Google Play staged rollout rollback, Apple expedited review).

2. Issue a server-side block on the affected version:
   ```
   remote-config set --key client_version_block --value "<affected-version>" --platform <platform>
   ```

3. Monitor affected device count and error rate after the block is applied.

### DRM-Related Failure

4. If the error code indicates DRM license failure (not a client crash):
   - Switch to the DRM License Service degradation runbook.
   - Issue a server-side configuration to use a shorter license validity period to force faster re-validation.

### Network-Related Failure

5. If errors are network-related (segment download timeouts, manifest fetch failures):
   - Issue a remote configuration to reduce the ABR starting bitrate, reducing the first-segment download time:
     ```
     remote-config set --key abr_startup_bitrate_cap --value <lower-bitrate-kbps> --platform <platform>
     ```
   - Evaluate whether a CDN issue is the underlying cause.

### Offline Playback Failure

6. If offline playback failures are reported (license expired, content removed):
   - Check whether a recent catalog change removed titles that subscribers have downloaded.
   - Check whether a DRM license renewal campaign failed for a specific device type.
   - Issue a server-side notification to affected subscribers via the in-app notification service.

---

## Communication

- **At detection:** Post scope, error code breakdown, and affected device segment to `#incidents-client`.
- **If app version rollback is considered:** Notify app store partnerships immediately; rollback timelines are platform-dependent.
- **Every 15 minutes:** Post updated impact metrics.
- **At resolution:** Post root cause, mitigation, and recovery timeline.

---

## Validation

1. Confirm Playback Error Rate returns to <= 0.1% on the affected device segment.
2. Confirm Session Startup Success Rate returns to >= 99.9%.
3. Confirm client crash rate returns to baseline on the affected app version.
4. Sustain monitoring for 30 minutes before declaring resolution.

---

## Post-Incident

1. Document which app version, device type, and error code were involved.
2. Identify whether the failure mode was covered by the client pre-release test suite.
3. Evaluate whether the Remote Configuration kill switch was applied quickly enough; if not, improve the trigger automation.
4. File a requirement to add the failure scenario to the client regression test suite.
5. Schedule a postmortem within 5 business days if SLO was breached.
