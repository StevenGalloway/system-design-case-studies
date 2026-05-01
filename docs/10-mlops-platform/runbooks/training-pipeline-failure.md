# Runbook: Training Pipeline Failure

**Severity:** SEV-2 (no new model versions produced; existing model continues serving)
**On-Call Team:** MLOps Platform SRE
**Escalation:** MLOps Platform Engineering Lead, model-owning team DRI

---

## Overview

A training pipeline failure prevents new model versions from being produced. The existing production model continues to serve traffic, so there is no immediate user impact. However, if the pipeline remains failed for an extended period, the production model becomes increasingly stale relative to the current data distribution, which can cause model drift and degrade model performance.

---

## Detection

- Automated alert: scheduled training job has not completed within 150% of its expected duration
- Automated alert: training job is in FAILED state after exhausting retry attempts
- Model Registry alert: no new model artifact has been registered for a model in longer than its declared training cadence
- Manual observation by model owner: expected training run is absent from the Experiment Tracker

---

## Initial Assessment (< 15 minutes)

1. Open the **Training Pipeline Dashboard** and identify:
   - Which model's training pipeline has failed?
   - At what stage did it fail? (data loading, feature join, training computation, evaluation, artifact registration)
   - Is this a one-time failure or a recurring pattern?

2. Review the training job logs for the specific error:
   ```
   mlops logs --job-id <training-job-id>
   ```

3. Determine whether the failure is in the MLOps platform or in the model-specific training code:
   - Platform issues: Spark cluster unavailable, GPU node pool exhausted, feature store unreachable
   - Model-specific issues: training code error, invalid hyperparameter configuration, out-of-memory on training hardware

---

## Mitigation Steps

### Transient Resource Failure

If the failure is due to a transient resource issue (spot instance preemption, temporary GPU node unavailability):

1. Retry the training job. Training jobs are designed to be idempotent; retrying from the beginning is safe:
   ```
   mlops retry --job-id <training-job-id>
   ```

2. If spot instance preemption is a recurring issue for this job, request a reservation for on-demand GPU capacity for the duration of the training run.

### Data Pipeline Failure

If the failure is at the data loading or feature join stage:

3. Check the feature store freshness for the features required by this model (see `07-data-platform/runbooks/pipeline-backlog.md`).

4. If the upstream feature pipeline is delayed, wait for it to recover before retrying the training job. Do not retry training with stale features; this will produce a model trained on a non-representative data snapshot.

5. If the feature pipeline has a structural issue, coordinate with the data platform team to fix the upstream pipeline first.

### Training Code Error

If the failure is in the model-specific training code:

6. Notify the model owner (DRI in the Feature Catalog) immediately; this is a model engineering issue, not a platform infrastructure issue.

7. The model owner should review the error, fix the training code, and submit a corrected training job.

8. The MLOps SRE assists with platform-level debugging (log access, environment inspection) but does not modify model training code.

### Resource Exhaustion (OOM / Insufficient Compute)

9. If the training job fails due to out-of-memory on the training workers, increase the allocated memory in the training job configuration:
   - Request the model owner to update the resource configuration in the training job definition.
   - If it is urgent (production model is critically stale), escalate to the MLOps platform lead to allocate additional GPU capacity.

---

## Assessing Staleness Risk

While the pipeline is being recovered, assess the risk that the production model is becoming stale:

1. Check when the current production model was last trained.
2. Compare the training data distribution at that time against the current data distribution (available in the model monitoring dashboard).
3. If PSI drift scores are already elevated, escalate the incident severity and notify the model owner to consider the staleness risk for their downstream consumers.

---

## Validation

1. Confirm the training job has completed successfully in the Training Pipeline Dashboard.
2. Confirm a new model artifact has been registered in the Model Registry.
3. Confirm the new model version has passed offline evaluation metrics.
4. Confirm the new model has been queued for shadow deployment.

---

## Post-Incident

1. Document the root cause and the duration of the training pipeline outage.
2. Measure how stale the production model became during the outage.
3. Identify whether earlier detection could have shortened the recovery timeline.
4. Review whether training job resource configurations are sized appropriately for current data volume.
5. Evaluate whether training jobs need more aggressive retry logic or resource reservation strategies for reliability.
