# API contracts

Status: proposed REST contract for review. No endpoints, authentication, background jobs, schemas, or database tables are implemented. Phase 1 uses FastAPI and Pydantic; implementation will publish versioned OpenAPI after approval.

## Conventions

Base path: `/api/v1`. Use JSON except bounded multipart CSV uploads. IDs are opaque UUIDs; timestamps are UTC with offsets. Input schemas reject unexpected fields on writes and constrain sizes/types; output schemas exclude secrets, raw filesystem paths, and private artifact locations. `null` denotes unknown, including unavailable risk/confidence values.

Authenticate every evidence endpoint and enforce role/workspace authorization server-side. Viewers read scoped records; analysts may ingest and triage; administrators manage approved configuration/artifacts. The auth provider is undecided. Derive workspace and actor from authenticated context; never accept privilege-bearing fields from clients. Health responses reveal no secrets. Plain local development uses loopback only; deployed traffic requires TLS.

List responses use bounded `limit` and opaque `cursor`, a stable timestamp/ID ordering, and `{ "items": [], "next_cursor": null }`. Specify maximum/default limits during implementation. Filtering supports explicit allowlisted fields; no arbitrary query expressions.

## Phase 1 resources

| Method and path | Intended behavior |
| --- | --- |
| `GET /health/live` | Process liveness; `200` if live |
| `GET /health/ready` | `200` only when required storage and the approved inference artifact are ready; otherwise `503` |
| `POST /datasets` | Analyst multipart upload with `file` and an allowlisted `adapter_id`; `202` with dataset ID, job ID and status link after durable acceptance |
| `GET /jobs/{job_id}` | Job status `queued/running/succeeded/failed`, stage, row counts, safe validation summary, error codes, and resulting dataset/model references |
| `GET /events` | Filter/list normalized Events by source, dataset or time interval |
| `GET /events/{event_id}` | Event and permitted provenance; restricted raw content requires separate authorization |
| `GET /detection-results` | List all persisted inference results, including below-threshold results, by Event/dataset/model |
| `GET /alerts` | Filter/list Alerts by status, severity, confidence band, or time |
| `GET /alerts/{alert_id}` | Alert, Event references, detector identity and separate assessment fields |
| `PATCH /alerts/{alert_id}` | Analyst changes status/disposition with rationale and expected record version; audit actor/time |
| `GET /models/{model_id}/evaluation` | Approved model manifest and complete evaluation metrics/limitations, without executable artifacts |

Dataset ingestion follows the documented validation → frozen preprocessing/features → approved classifier → risk assessment path. It does not train a model in an HTTP request. Offline training and artifact review precede inference; reject inference submissions with `503` if no compatible approved artifact exists. Uploaded target labels are isolated as dataset annotations and cannot affect inference features or be served as predictions. Training/evaluation datasets retain their declared partition role; scoring does not authorize reuse for training.

Track stage failures explicitly. A failed job may have earlier committed valid records; report committed/rejected/scored counts and checkpoint state. Commit each Event/result/Alert unit transactionally and resume idempotently. Only mark the job succeeded when every accepted record has a final processing outcome.

## Example Alert representation

Illustrative values are not evaluation results or a claim of implemented scoring:

```json
{
  "alert_id": "b0d66ef1-3bfd-4bb9-ab3f-c7599f8f6926",
  "schema_version": "1",
  "workspace_id": "79d8e0fd-ea90-4b16-857e-98af82c4f4b9",
  "created_at": "2026-10-07T09:00:00Z",
  "event_ids": ["f1ee408c-55d7-485d-8b3d-386f796f40aa"],
  "detection_result_ids": ["67f87cf5-4e4f-4d89-a63e-94a7d0ce7519"],
  "detector": {"kind": "supervised_ml", "id": "flow-lr", "version": "1"},
  "reason_codes": ["MODEL_THRESHOLD_EXCEEDED"],
  "evidence_refs": ["67f87cf5-4e4f-4d89-a63e-94a7d0ce7519"],
  "assessment": {
    "policy_version": "draft-1",
    "risk_score": null,
    "threat_severity": "unknown",
    "evidence_confidence": {"score": null, "band": "low"},
    "limitations": ["Single detector; asset impact and calibration are unknown"]
  },
  "attack_mappings": [],
  "status": "new",
  "disposition": "unknown",
  "record_version": 1
}
```

Detection results include `predicted_class`, `score`, `score_kind`, `threshold`, model/artifact and feature versions, and scoring time; use the same score scale for score and threshold. Risk/confidence unknowns are valid outcomes. Explanations cite these results and Event fields, not uploaded instructions.

## Errors, replay, and consistency

Errors use `{ "error": { "code": "VALIDATION_ERROR", "message": "Safe summary", "request_id": "opaque-id", "details": [] } }`. Do not echo sensitive rows, stack traces, credentials, or internal paths. Validation details use field/row references and reason codes.

| Status | Meaning |
| --- | --- |
| `400` / `422` | Malformed request / schema validation failure |
| `401` / `403` | Missing/invalid authentication / insufficient authorization |
| `404` | Missing record or resource outside the caller's scope |
| `409` | Idempotency-key payload conflict or stale record version |
| `413` / `415` | Upload too large / unsupported format |
| `429` | Rate/queue limit exceeded; supply retry guidance |
| `503` | Required storage, artifact, or processing capacity unavailable |

`POST /datasets` requires an `Idempotency-Key` scoped to workspace and operation. Persist its request digest and original response; retries with the same content return the original job reference, while different content returns `409`. Define retention before implementation and rely on stable source/row deduplication beyond that period. Reprocessing with a new artifact creates versioned results rather than overwriting old evidence. Alert updates require `record_version`; stale writes return `409`.

## Future resources

Incident creation, case linking, correlation, grounded AI explanations, and response proposals/approvals/execution will receive contracts and ADR review when implemented. Preserve separate `/incidents` resources rather than renaming Alerts as Incidents. Every relationship must validate same-workspace evidence references. Response approval is an explicit authenticated analyst action bound to exact parameters; no executor exists in Phase 1. LLM output cannot invoke a mutation or approve a response.
