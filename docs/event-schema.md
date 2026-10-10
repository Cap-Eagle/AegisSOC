# Event, Alert, and Incident schema

Status: proposed logical contract, not a database migration or implemented Pydantic model. [ADR 001](decisions/001-event-alert-incident-model.md) explains the entity split.

## Shared conventions

Use opaque UUID identifiers, an explicit `schema_version`, and UTC timestamps with an offset. Preserve `occurred_at` separately from `ingested_at`; if the source lacks a time, leave it unknown and record that limitation instead of inventing one. Missing values are `null`, not zero. Numeric values must be finite and satisfy declared ranges and units. Strings, row counts, nesting, and payload sizes require bounds at ingestion.

All records have a `workspace_id` for access scoping; derive it from authenticated context, not uploaded content. Phase 1 may use one workspace. External identities, IP addresses, timestamps, descriptions, and tags remain untrusted after parsing. Dataset adapters explicitly map supported columns into this envelope; arbitrary CSV columns do not become model features automatically.

## Event: observed evidence

| Field | Meaning |
| --- | --- |
| `event_id`, `schema_version`, `workspace_id` | Identity, contract version, access scope |
| `occurred_at`, `ingested_at` | Nullable source time and required ingestion time |
| `source` | Source type, source ID, optional native record ID |
| `provenance` | Dataset ID/hash, batch ID, row reference, parser/normalization version |
| `deduplication_key` | Stable source-aware replay identity; separate from ML duplicate grouping |
| `kind` | E.g. `network_flow`; future authentication, endpoint, DNS kinds |
| `network` | Optional source/destination IP and port, protocol, duration in seconds, byte/packet counts |
| `subject`, `asset` | Optional scoped user/device references; unknown context remains unknown |
| `raw_ref`, `raw_sha256` | Optional access-controlled raw evidence reference and integrity digest |
| `normalization` | Validation quality, missing fields, transform version, parent raw Event reference if applicable |

An Event is raw or normalized telemetry, not a verdict. Retain immutable source evidence and version normalization rather than overwriting history. Raw evidence can be held outside PostgreSQL using a protected reference; never expose arbitrary file paths or execute attached content. Dataset-only inputs may lack network context; retain supported measurement fields in a versioned typed extension, with units and provenance.

Ground-truth dataset labels belong to a separate training annotation linked by `event_id`/dataset row. Store original label, mapping version, and mapped target (`0 = BENIGN`, `1 = MALICIOUS`). Labels and analyst dispositions must never enter the inference feature vector. Features are derived, versioned records referencing Events, not mutable Event verdict fields.

## Detection result and Alert

Persist a detection result for every scored Event, including below-threshold results: result ID, Event references, detector/version, artifact and feature versions, output class, raw score and its semantics, threshold, evaluation reference, and scoring time. For multi-event detectors, record all input evidence references. An uncalibrated score is not a probability or evidence confidence.

An **Alert** is a detection selected by an explicit alert policy; it requires at least one Event reference.

| Field | Meaning |
| --- | --- |
| `alert_id`, `schema_version`, `workspace_id`, `created_at` | Identity and scope |
| `event_ids`, `detection_result_ids` | Nonempty contributing evidence and originating detection references |
| `detector` | Kind (`rule`, `supervised_ml`, `anomaly`, `ueba`), ID, version, artifact reference where relevant |
| `reason_codes`, `evidence_refs` | Structured reasons and field-level evidence references |
| `assessment` | Separate Threat Severity/risk score and Evidence Confidence per [risk engine](risk-engine.md) |
| `attack_mappings` | Optional MITRE ATT&CK technique/version, mapping source, rationale, evidence references |
| `status` | `new`, `triaged`, or `closed` |
| `disposition` | Separate analyst judgment: `unknown`, `benign`, or `malicious`, with author/time/rationale |

A positive classifier output does not by itself prove a compromise or a MITRE technique. Corroboration, policy, and detector limitations must remain visible. Closing an Alert does not erase its Event evidence or automatically label it benign.

## Incident: correlated security case

| Field | Meaning |
| --- | --- |
| `incident_id`, `schema_version`, `workspace_id`, timestamps | Identity, scope, creation/update times |
| `title`, `summary`, `owner_id` | Analyst case context; summaries cite evidence when AI-assisted |
| `alert_ids`, `event_ids` | Related Alerts and optionally direct supporting Events |
| `correlation` | Rule/manual method, version, time window, rationale, evidence references |
| `assessment` | Case-level severity and confidence with an aggregation policy/version |
| `status` | `open`, `investigating`, `resolved`, or `closed` |
| `resolution` | Analyst disposition, rationale, actor, timestamp |

An Incident must contain at least one Alert or Event; event-only cases require an analyst rationale. Its relationships do not copy or mutate the underlying evidence.

## Relationship and integrity rules

- Events can exist without Alerts or Incidents. An Event may support many Alerts; an Alert may cite many Events.
- Incident-to-Alert and Incident-to-Event links support many-to-many associations with rationale and audit history; prevent duplicate links.
- Enforce same-workspace references, foreign-key integrity, and at least one Event per Alert. Aggregate nonempty constraints require transactional validation.
- Preserve archived evidence references under retention policy; do not silently cascade-delete investigative history. Log authorized edits, case links, and status changes.
- Store explanations as derived records with evidence IDs, generator/version, timestamp, and limitations. Explanations cannot alter detections, approvals, or raw evidence.
