# ADR-0001: Shadow Deployment for Model Validation

## Status
Accepted

## Context

Machine learning models have a well-documented gap between offline evaluation metrics and production behavior. A model that achieves the best offline NDCG score on a held-out test set does not always produce the best recommendations in production. Reasons for this gap include:

- **Distribution shift:** The training data distribution may not match the production distribution at the time of deployment
- **Feature pipeline differences:** Offline features computed in a batch training context may differ from online features served by the feature store in production
- **Feedback loops:** Existing production model behavior influences the training data; a new model trained on this data may behave differently when the feedback loop changes
- **Counterfactual evaluation:** Offline metrics measure performance on historical data; they cannot evaluate the model's behavior on traffic it would handle differently than the incumbent

The personalization team was experiencing cases where models with improved offline metrics performed worse on A/B test engagement metrics, wasting multi-week experiment slots on models that were not ready for production.

A validation mechanism was needed that could evaluate new models on real production traffic before any subscriber's experience was affected by the new model's predictions.

## Decision

Implement shadow deployment as the mandatory pre-production validation step for all ML model updates.

In shadow deployment mode, a configurable percentage of production inference requests (typically 5-10%) are duplicated and sent to both the incumbent model and the shadow model. The shadow model's predictions are recorded but not served to users. The incumbent model's predictions continue to be used for all user-facing decisions during the shadow period.

Shadow evaluation metrics are collected for a minimum of 72 hours before a model can be promoted to canary deployment. The metrics evaluated include:
- Prediction distribution drift relative to the incumbent
- Feature distribution at inference time compared to offline training distribution
- Inference latency at p50, p95, and p99
- Any hard errors or invalid prediction outputs

Promotion from shadow to canary is gated on:
- Prediction distribution change within acceptable bounds (configurable per model type)
- No increase in inference latency p99 greater than 20% relative to the incumbent
- Zero hard errors during the shadow period

## Consequences

### Positive
- Catches distribution shift, feature pipeline differences, and latency regressions before any user sees the new model
- Provides production-fidelity evaluation data that offline metrics cannot replicate
- Reduces the cost of failed A/B experiments by preventing clearly underperforming models from consuming experiment traffic slots

### Negative
- Shadow deployment requires duplicating a fraction of inference traffic, increasing inference compute costs proportionally
- Shadow results require careful interpretation: good shadow metrics do not guarantee positive engagement metrics, since the shadow model's predictions are never served
- Minimum shadow period of 72 hours adds to the model deployment lead time; teams that want to move fast must plan for this gate

## Alternatives Considered

**Offline evaluation only:** Fast but insufficient; the gap between offline and production performance is a recurring source of failed experiments.

**Immediate canary deployment without shadow:** Exposes a fraction of users to the new model before its production behavior is understood. Acceptable for low-risk model updates, but creates subscriber experience risk for major model changes.

**Interleaving evaluation:** Exposes both models' predictions to the same user in an interleaved manner and tracks which predictions are engaged with. More statistically efficient than A/B testing but requires changes to the serving infrastructure that are not universally applicable.
