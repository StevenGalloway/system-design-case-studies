# Runbook: Model Drift Response

**Severity:** SEV-2 (single model drifting) / SEV-1 (drift affecting core personalization or fraud detection)
**On-Call Team:** MLOps Platform SRE, Model-Owning Team
**Escalation:** MLOps Platform Engineering Lead, Personalization Engineering Lead (if applicable)

---

## Overview

Model drift occurs when the statistical relationship between model inputs and outputs changes over time in ways that degrade model performance. Drift can manifest as:

- **Feature drift:** The distribution of input features at inference time diverges from the training distribution
- **Prediction drift:** The distribution of model outputs changes without a corresponding change in expected outcomes
- **Business metric impact:** Downstream engagement, satisfaction, or conversion metrics decline in a pattern correlated with a deployed model

---

## Detection

- Automated alert: feature distribution drift score (Population Stability Index) exceeds threshold (PSI > 0.2) for a key feature
- Automated alert: prediction drift metric exceeds threshold for 30 consecutive minutes
- Downstream business metric alert: a metric tied to the model's output (e.g., homepage click-through rate) drops by >= 5% relative to the 7-day baseline
- Model monitoring dashboard shows divergence between shadow model predictions and incumbent predictions

---

## Initial Assessment (< 15 minutes)

1. Open the **Model Monitoring Dashboard** for the affected model and identify:
   - Which features are drifting? (check the feature drift report)
   - Is prediction drift present independent of feature drift?
   - When did the drift begin? (correlate with data pipeline changes and model deployments)

2. Determine whether the drift is caused by a data pipeline issue or a genuine distribution shift in the underlying data:
   - Has the feature computation pipeline changed recently?
   - Has the upstream data source changed (schema change, new data source, removed data source)?
   - Is the drift seasonal or correlated with a known external event?

3. Check whether downstream business metrics are measurably impacted.

---

## Mitigation Steps

### Rollback the Model

If drift is confirmed and business metrics are impacted:

1. Identify the previous stable model version in the Model Registry.

2. Initiate a model rollback via the MLOps deployment controller:
   ```
   mlops rollback --model-id <model-id> --target-version <stable-version>
   ```

3. Monitor the rollback: confirm traffic is routing to the stable version within 5 minutes.

4. Verify that business metrics begin recovering toward baseline after the rollback.

### Address Feature Drift Without Rollback

If feature drift is confirmed to be caused by a data pipeline issue rather than a model quality issue:

5. Identify and fix the data pipeline issue (see `07-data-platform/runbooks/pipeline-backlog.md` if applicable).

6. Backfill the online feature store with corrected feature values once the pipeline is fixed.

7. Monitor model performance for 24 hours after the feature pipeline is corrected before deciding whether retraining is needed.

### Schedule Retraining

After rollback or pipeline fix:

8. Trigger a new training run with a refreshed training dataset that reflects the current data distribution:
   ```
   mlops train --model-id <model-id> --training-config <config-file>
   ```

9. The new training run goes through the standard shadow deployment validation before promotion.

---

## Communication

- **At detection:** Notify the model owner (DRI listed in the Feature Catalog) via PagerDuty and `#mlops-incidents`.
- **If business metrics are impacted:** Escalate to the relevant product engineering lead.
- **At rollback:** Post confirmation in `#mlops-incidents` with the rolled-back version.
- **At resolution:** Post root cause and retraining plan.

---

## Validation

1. Confirm the rolled-back model is receiving 100% of production traffic.
2. Confirm feature drift scores return to within acceptable bounds on the model monitoring dashboard.
3. Confirm prediction distribution returns to the baseline established by the stable model.
4. Confirm downstream business metrics show recovery trend over 24-48 hours.

---

## Post-Incident

1. Document the nature of the drift: feature, prediction, or business metric impact.
2. Identify whether the drift should have been caught earlier by more sensitive monitoring thresholds.
3. Determine whether the feature pipeline change that caused the drift went through review; if not, propose a gate.
4. File a requirement to add a drift scenario test to the model deployment pipeline.
5. Review the PSI thresholds for the affected model; recalibrate if they were too insensitive to catch the drift sooner.
