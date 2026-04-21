# Runbook: Data Pipeline Backlog

**Severity:** SEV-2 (batch pipeline delayed) / SEV-3 (within SLA buffer)
**On-Call Team:** Data Platform SRE
**Escalation:** Data Platform Engineering Lead, affected domain team lead

---

## Overview

A pipeline backlog occurs when a scheduled Spark batch job fails to complete within its expected window, causing downstream Iceberg tables to fall behind their freshness SLA. This affects analytical dashboards, ML training schedules, and offline feature store partitions.

---

## Detection

- Automated alert: Spark job has not completed within 150% of its expected duration
- SLA monitoring alert: Iceberg table partition has not been updated within the declared freshness SLA
- Downstream team reports stale data in dashboards or a failed ML training run
- Manual observation: a job is still running when the next scheduled run should begin

---

## Initial Assessment (< 10 minutes)

1. Open the **Pipeline Monitoring Dashboard** and identify:
   - Which Spark job is delayed?
   - What stage of the job is it stuck in?
   - How long has it been running relative to its expected duration?

2. Check the Spark job history server for the job's stage breakdown and identify the slowest stages.

3. Determine whether this is a first-time failure or a recurring issue on the same job.

4. Check for resource contention: is the Spark cluster underprovisioned due to multiple concurrent jobs?

---

## Mitigation Steps

### Job Is Stuck in a Long-Running Stage

1. Check for data skew: a single partition with significantly more data than others will slow a stage to that partition's pace:
   - Inspect the Spark UI for tasks within the stuck stage; look for outlier task durations.
   - If skew is confirmed, apply salting or repartitioning to the input dataset.

2. Check for insufficient executor memory causing spill to disk:
   - Review GC overhead metrics in the Spark UI.
   - Increase executor memory in the job configuration and retry.

### Job Has Failed

3. Review the Spark driver logs for the error:
   ```
   kubectl logs <spark-driver-pod> -n data-platform
   ```

4. If the failure is due to a transient resource issue (OOM, node failure), retry the job from the beginning:
   - Iceberg's transactional writes ensure an incomplete previous run does not corrupt the output table.
   ```
   spark-submit --class <MainClass> <job-jar> --date <processing-date>
   ```

5. If the failure is due to a data quality issue in the input (corrupt records, schema mismatch):
   - Quarantine the offending input partition.
   - Notify the upstream team via their data quality SLA contact.
   - Retry the job with the quarantined partition excluded; document the exclusion.

### Resource Contention

6. If the cluster is overloaded due to concurrent jobs:
   - Pause lower-priority jobs (non-SLA-bound training jobs) to free resources.
   - Increase cluster capacity by scaling the Spark node pool:
     ```
     kubectl scale nodepool spark-workers --node-count=<target> -n data-platform
     ```
   - After the SLA-critical job completes, resume lower-priority jobs.

---

## Communicating Delays to Downstream Teams

If a freshness SLA breach is confirmed, notify the affected teams proactively:

1. Identify all datasets with a declared dependency on the delayed output (from the data catalog lineage view).
2. Post to `#data-platform-incidents` with the affected dataset, expected delay, and current ETA.
3. If ML training is blocked, coordinate with the MLOps team to assess whether a training window can be deferred.

---

## Validation

1. Confirm the Spark job has completed successfully in the job history server.
2. Confirm the Iceberg table partition has been updated with the expected timestamp.
3. Confirm freshness SLA monitoring returns to green on the pipeline health dashboard.
4. Notify downstream teams that data is current.

---

## Post-Incident

1. Document the root cause and the processing delay with precise timestamps.
2. Identify whether the job's resource configuration is appropriately sized for current data volume.
3. Review whether job scheduling can be adjusted to reduce peak concurrency.
4. If data skew caused the issue, file a ticket to apply permanent input repartitioning.
5. Update pipeline freshness SLA if the job's expected duration has grown with data volume growth.
