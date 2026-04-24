# Runbook: Encoding Job Failure

**Severity:** SEV-2 (single title affected) / SEV-1 (encoding cluster-wide failure)
**On-Call Team:** Content Engineering SRE
**Escalation:** Encoding Platform Engineering Lead

---

## Overview

Encoding job failures delay content availability. Single-job failures are expected to self-recover through automatic retry. Cluster-wide failures that block an entire release window are SEV-1 events.

---

## Detection

- Automated alert: encoding job in FAILED state after exhausting retry attempts
- Dashboard alert: encoding queue depth growing without jobs completing
- Release manager reports a title is not available at scheduled launch time
- Encoding orchestrator reports worker pool health degraded below 80%

---

## Initial Assessment (< 10 minutes)

1. Open the **Encoding Orchestrator Dashboard** and identify:
   - How many jobs are in FAILED state?
   - Is this a single title or multiple titles?
   - At what stage did the failure occur? (complexity analysis, encoding, VMAF validation, packaging)

2. Check the encoding worker pool health:
   - How many workers are active and accepting jobs?
   - Are any workers in an error state?

3. Review the failed job's error log in the orchestrator UI.

---

## Mitigation: Single Job Failure

### Transient Worker Failure

1. If the job failed due to a worker crash or pre-emption, retry the job. Encoding jobs are idempotent:
   ```
   encoding-cli retry --job-id <job-id>
   ```

2. Monitor the retry in the orchestrator dashboard.

### Source File Issue

3. If the error indicates a problem reading the mezzanine file (checksum mismatch, corrupt container):
   - Quarantine the mezzanine file.
   - Notify the content supply chain team to investigate the ingest source.
   - Do not retry until the source file is re-validated.

### Per-Title Optimizer Failure

4. If the job failed at the bitrate ladder generation step:
   - Override the optimizer and use the default bitrate ladder:
     ```
     encoding-cli retry --job-id <job-id> --use-default-ladder
     ```
   - File a bug against the per-title optimizer with the title identifier for post-incident investigation.

### VMAF Quality Gate Rejection

5. If the job completed but was rejected by the VMAF quality gate:
   - Review the VMAF score and compare against the threshold.
   - Inspect the encode for visible quality artifacts using the encoding quality review tool.
   - If the mezzanine source has quality issues, escalate to the studio relationship team.
   - Manual override is available with dual senior engineer approval; document the justification.

---

## Mitigation: Cluster-Wide Failure

6. If multiple jobs are failing across different titles, the issue is likely in the encoding cluster, not the content:
   - Check worker node health: look for failed nodes, resource exhaustion, or network partition.
   - Check the encoding orchestrator service health; if it is unavailable, jobs will queue without processing.

7. If the orchestrator is unavailable, restart it:
   ```
   kubectl rollout restart deployment/encoding-orchestrator -n encoding
   ```

8. Scale up the worker pool to recover the queue backlog faster:
   ```
   kubectl scale deployment encoding-workers --replicas=<target> -n encoding
   ```

9. If a release deadline is at risk, prioritize the affected title's encoding jobs:
   ```
   encoding-cli prioritize --job-id <job-id> --priority critical
   ```

---

## Validation

1. Confirm the failed job has completed successfully in the orchestrator dashboard.
2. Confirm the VMAF quality gate passed for the completed encode.
3. Confirm the content artifact has been written to the artifact store.
4. Confirm the title is available for playback in the staging environment before release.

---

## Post-Incident

1. Document the root cause and the delay duration for the affected title.
2. If the per-title optimizer failed, review the optimizer logs and file an improvement ticket.
3. If worker pre-emption caused the failure, review the worker compute allocation and spot instance strategy.
4. Update encoding job SLA monitoring if thresholds need adjustment.
