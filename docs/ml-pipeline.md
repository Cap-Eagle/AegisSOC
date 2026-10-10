# ML pipeline

Status: proposed offline workflow; no training or inference code exists. Phase 1 uses a CSV dataset, NumPy, Pandas or Polars, and scikit-learn. The objective is reproducible binary detection, not a claim of production attack coverage.

## Target and data contract

The first target is **`0 = BENIGN`, `1 = MALICIOUS`**, with malicious as the positive class. Use a versioned label mapping; reject or quarantine unknown labels rather than coercing them to benign. Ground truth belongs in training annotations, never the feature vector. Dataset choice, license, provenance, field semantics, and representativeness need approval before training.

Record dataset hash/version, adapter/schema version, row counts, missingness, class distribution, source/capture periods, and label quality. Validate required columns, numeric types, finite values, units, ranges, encoding, and input limits. Preserve rejected-row reasons and source row references; do not silently discard enough data to distort prevalence. Treat CSV strings as untrusted data.

## Workflow and leakage prevention

1. Validate using deterministic schema rules. Retain immutable source evidence and create stable row/Event references.
2. Identify exact duplicates and related flows using label-independent hashes/group keys. Keep capture/session/host groups and near-duplicate flows together according to documented grouping policy.
3. Split into training, validation, and a locked test holdout **before fitting any learned preprocessing**. Prefer chronological and capture/group separation where metadata permits. Stratify only when compatible with those constraints. If grouping metadata is unavailable, report that limitation and do not claim leakage is fully ruled out.
4. Fit imputation, scaling, encoding, feature selection, resampling, and any learned statistics only on training data, inside a pipeline. For cross-validation, fit them again within each training fold. Apply frozen transforms to validation/test.
5. Engineer allowlisted features from information available at inference time. For temporal/window features, use only past information as of the event time. Record feature names/order/types/units/version and missing-value policy.
6. Train candidates, tune using training folds/validation only, and select thresholds against an explicit operational false-positive/recall objective. If needed, calibrate using training folds or a dedicated non-test calibration partition.
7. Freeze the chosen pipeline, model, calibration and threshold; evaluate once on the locked test set. Reusing that test for model/feature decisions makes it validation data and requires a new independent holdout.
8. Register the evaluation report and artifact manifest before making the model eligible for inference. Risk assessment consumes detection results and evidence under a separate policy.

Explicit leakage controls:

| Leakage type | Prevention and verification |
| --- | --- |
| Train/test leakage | Disjoint row IDs, chronological/group-aware partitions, untouched final holdout, fold-local fitting; assert split membership |
| Duplicate-flow leakage | Group exact duplicates, reverse-direction equivalents where appropriate, shared flow/session/capture records and defined near-duplicates; assert no grouping key crosses partitions; grouping never uses target labels |
| Target leakage | Exclude labels, attack/category labels, analyst outcomes, post-detection fields, filenames/source IDs encoding verdicts, and future information; audit features for target proxies |
| Preprocessing leakage | Fit imputers/scalers/encoders/selectors/resamplers only on each training fold; no full-dataset statistics; persist and reuse frozen training transforms |

Detect conflicting duplicate labels during data audit and document resolution. Fit class weights or resampling on training only; preserve natural validation/test prevalence. A random row split alone is insufficient when flows are related.

## Model progression

- **Logistic Regression:** first baseline with fold-local preprocessing and an interpretable coefficient report.
- **Random Forest:** compare on identical splits and operating constraints, including latency, artifact size, and calibration.
- **XGBoost/LightGBM:** future candidates only if repeatable evaluation shows useful improvement that warrants dependencies and operational cost; record the change in an ADR.

Include a trivial majority/constant baseline for context. Rules, anomaly detection, and UEBA later emit distinct detector results and need their own evaluation protocols. LLMs never replace this detector workflow.

## Mandatory evaluation report

For the positive malicious class, report counts `TP`, `FP`, `TN`, `FN` and:

| Metric | Definition/reporting requirement |
| --- | --- |
| Precision | `TP / (TP + FP)` |
| Recall | `TP / (TP + FN)` |
| F1 | Harmonic mean of precision and recall |
| PR-AUC | Area under the precision-recall curve from continuous scores; state the integration method, and distinguish average precision if reported |
| ROC-AUC | Area under the ROC curve from continuous scores |
| False-positive rate | `FP / (FP + TN)` |
| False-negative rate | `FN / (FN + TP)` |
| Confusion matrix | Counts with actual rows/predicted columns and fixed order `[0, 1]`: `[[TN, FP], [FN, TP]]` |

Undefined denominators or single-class evaluation partitions must be reported as undefined with a reason, not silently converted to a reassuring value. Accuracy is optional and never a substitute for these metrics. Include partition sizes, class prevalence, threshold, per-source/time slices where meaningful, uncertainty estimates where feasible, and inference cost. No performance numbers are invented in this foundation.

## Artifact, serving, and monitoring contract

The manifest records dataset hash, split/grouping policy and seed, feature schema/order, preprocessing and model versions, dependency versions, label map, training parameters, threshold, calibration method/status, evaluation metrics, limitations, and artifact checksum. Only reviewed artifacts from trusted origins may be loaded; pickle/joblib files from untrusted sources can execute code.

Serving validates schema/version, applies the identical frozen transform, and records model identity, score semantics, threshold and evidence references. Missing features or incompatible artifacts fail explicitly; never infer benign from an error. A positive class prediction is a model judgment, not confirmed malicious ground truth.

Monitor missingness, invalid inputs, feature and score drift, latency, alert rate, and—when verified labels arrive—precision/recall/FPR/FNR and calibration. Record delayed-label bias and analyst-feedback provenance. Avoid automatic retraining or promotion from unreviewed feedback; compare and approve replacement artifacts, with rollback available.

## Streaming evolution

Keep CSV adapters separate from pure record transforms and model inference. Bound batch memory, version feature definitions, preserve event time, and make result writes idempotent. Later stateful features require explicit windows, late-arrival policy, and online/offline parity tests. A batch feature requiring an entire future capture cannot be reused as an online feature unchanged.
