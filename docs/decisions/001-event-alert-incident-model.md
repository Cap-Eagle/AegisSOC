# ADR 001: Separate Event, Alert, and Incident entities

Date: 2026-10-07
Status: proposed foundation awaiting user review. The entity separation is a user-mandated invariant; the detailed relationships are proposed here.

## Context

Raw telemetry, detector judgments, and analyst cases have different provenance, cardinality, and lifecycles. A single flow can support several detections, and a correlated case can combine many Alerts plus supporting Events. The first phase uses a CSV binary classifier; later rules, anomaly detection, UEBA, enrichment, MITRE ATT&CK mappings, and attack graphs need to share evidence without rewriting it.

## Decision

Maintain three separate entities:

- **Event:** raw or normalized telemetry, with source provenance and immutable evidence history. It can exist independently of any detection.
- **Alert:** a detection generated from one or more Events and selected under an explicit alert policy. Store detector/version, contributing evidence, reasons, and a separate assessment.
- **Incident:** a correlated security case containing related Alerts and/or Events, with correlation rationale and analyst workflow state. Event-only cases require analyst rationale.

Use explicit many-to-many relationships between Events and Alerts, Incidents and Alerts, and Incidents and direct supporting Events, constrained to one access scope. Keep per-event detection results even when no Alert is generated. Dataset ground-truth annotations, model predictions, risk assessments, generated explanations, and analyst dispositions are separate derived/context records, not mutable raw telemetry.

Maintain Threat Severity and Evidence Confidence separately. AI explanations cite structured evidence, cannot create authoritative verdicts, and cannot approve actions. Automated response requires analyst approval by default. Incident automation and response execution remain outside Phase 1.

## Alternatives considered

| Alternative | Why it was rejected |
| --- | --- |
| Single telemetry/detection/case table with flags | Conflates observation, prediction, and investigation; loses lifecycle and provenance boundaries |
| Every Event is an Alert | Forces benign/unscored telemetry into an analyst queue and obscures alert policy |
| Rename an Alert to Incident on escalation | Prevents multi-alert correlation and rewrites the meaning of existing identifiers |
| Copy raw Events into each case | Duplicates sensitive evidence and permits conflicting copies |
| Separate microservice per entity immediately | Adds deployment/transaction complexity without Phase 1 need |

## Consequences

Evidence can be reused across detectors and cases; detector reprocessing produces versioned results without mutating observations. Cases can track analyst outcomes without changing ground truth or model outputs. Correlation, graphs, and streaming can evolve behind stable entity boundaries.

The design requires explicit relationship integrity, access-scope checks, idempotency, audit history, and retention rules. It adds some logical structure before full Incident workflows are implemented, while preserving one modular backend and PostgreSQL.

## Validation plan

During later implementation, verify that Events persist without Alerts, every Alert cites at least one Event, multiple Alerts can share evidence, a case can link several Alerts/direct Events, cross-workspace links fail, replay does not duplicate Alerts, and closing a case preserves evidence. Check that labels never enter inference features and that severity, confidence, predictions, and analyst dispositions remain independently represented.

## Related documents and change policy

See [architecture](../architecture.md), [event schema](../event-schema.md), [risk engine](../risk-engine.md), and [threat model](../threat-model.md). Every major architectural change requires a new ADR in this directory with context, decision, alternatives, consequences, status, and validation plan. Supersede decisions explicitly and preserve historical rationale.
