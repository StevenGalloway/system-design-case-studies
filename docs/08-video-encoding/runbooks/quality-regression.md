# Runbook: Encoding Quality Regression

**Severity:** SEV-2
**On-Call Team:** Content Engineering SRE, Encoding Quality Team
**Escalation:** Encoding Platform Engineering Lead, VP of Content Engineering

---

## Overview

An encoding quality regression occurs when a change to the encoding pipeline, per-title optimizer, or codec configuration causes a degradation in VMAF scores or visible quality artifacts on published content. Unlike an encoding failure, quality regressions may result in content reaching subscribers with substandard visual quality before detection.

---

## Detection

- VMAF monitoring alert: batch VMAF audit detects average score drop of >= 2 points across a title cohort
- Subscriber complaint spike: complaints referencing picture quality on specific titles
- A/B quality review flag: human quality reviewers flag a title during post-publish audit
- Pipeline change correlation: VMAF score trend shifts shortly after an encoding configuration deployment

---

## Initial Assessment (< 15 minutes)

1. Determine the scope:
   - Are the affected titles all encoded after a specific date or pipeline version?
   - Is the regression isolated to a specific resolution, codec, or content type?
   - Are the VMAF scores just below threshold, or significantly degraded?

2. Correlate with recent changes:
   - Review the encoding pipeline deployment log for changes in the past 7 days.
   - Check per-title optimizer model version history for recent updates.
   - Check codec configuration change log.

3. Pull VMAF score trends from the encoding quality dashboard for the affected title cohort.

---

## Immediate Mitigation

### Prevent Further Degraded Encodes from Publishing

1. If a specific pipeline version is identified as the cause, halt new encoding jobs using that version:
   ```
   encoding-cli halt-queue --pipeline-version <version>
   ```

2. Roll back the encoding pipeline to the last confirmed stable version:
   ```
   encoding-cli deploy --version <stable-version>
   ```

### Re-Encode Affected Titles

3. Identify all titles encoded with the affected pipeline version using the encoding artifact store:
   ```
   SELECT title_id, encode_version FROM encoding_artifacts
   WHERE pipeline_version = '<affected-version>'
   AND published_at > '<regression-start-timestamp>'
   ```

4. Prioritize re-encoding by subscriber impact: titles with high current viewership are encoded first.

5. Queue re-encoding jobs at elevated priority. Until re-encoding completes, affected titles remain available (degraded quality is preferable to unavailability for most cases).

6. If VMAF scores are critically low (below 70 for 1080p), consider temporarily making affected titles unavailable in the highest affected resolution tier while re-encoding completes.

---

## Investigation

1. Run VMAF scoring on a representative sample of affected and unaffected encodes side by side.
2. Identify which component of the quality metric dropped: blocking artifacts, banding, blurriness, or temporal inconsistency.
3. If the per-title optimizer is suspected, compare generated bitrate ladders before and after the regression date.
4. If a codec configuration change is suspected, encode a reference test title with both the old and new configuration and compare VMAF scores directly.

---

## Validation

1. Confirm re-encoded titles pass the VMAF quality gate.
2. Confirm VMAF score trend has returned to baseline on the quality dashboard.
3. Confirm no new complaints correlated with the affected title cohort.
4. Run the full VMAF audit against all titles published in the affected window.

---

## Post-Incident

1. Document which titles were affected, the duration of the regression, and the delta in VMAF scores.
2. Determine whether the quality gate at publish time should have caught this; if not, propose gate threshold tightening.
3. Identify whether the encoding pipeline change went through a quality regression test in staging before production deployment.
4. File a requirement to expand the pre-deployment quality regression test suite to cover the failure scenario.
5. Notify studio partners if their content was affected.
