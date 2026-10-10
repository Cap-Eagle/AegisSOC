# Risk engine

Status: proposed assessment contract. Concrete thresholds and weights require validation with analyst feedback; no scoring implementation or calibrated risk model exists yet.

## Separate outputs

**Threat Severity** estimates the potential impact and urgency of the suspected threat. **Evidence Confidence** estimates how well available evidence supports the detection. They answer different questions and must remain separate in storage, API responses, and the dashboard.

| Value | Interpretation |
| --- | --- |
| Classifier score | Detector output; only a calibrated, evaluated score may be described as an estimated malicious probability |
| `risk_score` | Optional `0–100` policy score representing Threat Severity; not a probability |
| `threat_severity` | `unknown`, `low`, `medium`, `high`, or `critical`, using versioned policy bands |
| `evidence_confidence` | Separate optional `0–1` evidence-support index and `unknown/low/medium/high` band; not a calibrated probability unless separately validated |

High severity with low confidence warrants careful validation of a potentially serious threat. High confidence in a low-impact observation does not imply critical severity. Unknown evidence or context is `null`/`unknown`, never silently treated as zero risk or certainty of benign behavior.

## Inputs and assessment record

Inputs include detection results, detector validation/calibration context, supporting Event fields, normalization quality, asset criticality if known, attack/behavior impact mapping if supported, verified enrichment freshness, and corroborating sources. Scores must not treat repeated copies of the same flow as independent corroboration.

An assessment records its subject type/ID, policy/version, assessment time, input evidence IDs, model/detector versions, separate severity and confidence outputs, contribution/reason codes, missing context, and limitations. Reassessment creates a versioned history; keep the assessment originally shown to analysts reproducible.

## Phase 1 policy design

Use a transparent policy and deterministic explanations. Map supported malicious behavior/impact evidence and known asset context into severity. Keep the detector score as a separate triage signal. If the dataset only supplies numeric flow features and binary labels, neither an attack family nor actual business impact is known: show `threat_severity=unknown` and `risk_score=null` unless a reviewed provisional severity policy is adopted. Do not fabricate asset criticality, MITRE mappings, or impact from a binary label.

Evidence Confidence considers provenance, completeness, detector suitability, calibration status, validation coverage, and independent corroboration. A single model result warrants explicit limitations. Phase 1 can use documented confidence bands while numerical weights remain unvalidated; do not manufacture numerical confidence from the classifier score.

Before adopting a numeric formula, define its inputs, scales, missingness behavior, weights/bands, aggregation rules, and validation evidence. Record major policy changes in an ADR. Separate the alert threshold from severity/confidence bands, and display all three decisions independently.

## Incident aggregation and explanations

Future Incident assessment considers the strongest supported impact, affected asset scope, chronology, and independent evidence. Do not sum Alert scores without bounds or average away a critical observation. Deduplicate shared Event evidence, retain conflicting signals, and cite the correlation policy/version. Aggregation is not part of the initial classifier scope.

Each explanation states the assessed subject, observed evidence, detector/threshold, severity rationale, confidence rationale, and missing information. Prefer reason codes and cited structured fields initially. Later LLM wording is allowed only from this assessment and authorized evidence; validate citations, label inference as inference, and abstain when unsupported. Explanations do not change model output, risk policy, or analyst approvals. SHAP or similar attribution may describe feature contribution later but does not prove causation.

Risk can order an analyst queue; it cannot authorize automated response. Any future action requires analyst approval by default under [the threat model](threat-model.md).
