# Threat model

Status: required design controls for future implementation; none is claimed to be implemented. Phase 1 handles offline CSV telemetry and model artifacts, a REST API, PostgreSQL, and an analyst dashboard. LLM assistance and response execution are future capabilities.

## Assets and adversaries

Protect raw telemetry, identity/asset context, dataset annotations, analyst notes, evidence integrity, model artifacts, credentials, approvals, and audit history. Telemetry may contain personal information and attacker-chosen text. Adversaries include telemetry producers attempting evasion or poisoning, malicious uploaders, unauthorized API clients, compromised accounts, and compromised dependencies/artifacts. Accidental leakage and misleading evaluations are also security failures.

## Trust boundaries

```text
Untrusted CSV/source → constrained parser → validated Event records
Event records → allowlisted feature transform → approved detection artifact
Authenticated browser → authorized API → scoped database/evidence storage
Selected evidence → redaction/grounding boundary → optional external LLM
Response proposal → analyst approval gate → future restricted executor
```

Validation establishes shape, not truth. Internal enrichment and database content remain untrusted if their source is untrusted. The backend enforces scope, authorization, and approval regardless of frontend controls or environment flags.

## Threats and controls

| Threat | Required controls |
| --- | --- |
| Malformed/oversized CSV, path traversal, parser exhaustion | Bound upload bytes/rows/columns/field sizes; generate server-side storage names; constrain parsing resources; reject unsupported formats; quarantine invalid rows with safe errors |
| Payload/code execution | Treat uploads as data; no `eval`, shell interpolation, or executable deserialization of untrusted content; never execute malware on the development host |
| SQL injection, stored XSS, CSV formula injection | Parameterized database operations; encode text in the dashboard; avoid raw HTML; escape formula-leading values in exports |
| Dataset poisoning, evasion, false ground truth | Record source/license/hash and label provenance; review datasets; isolate evaluation sets; monitor quality/drift; do not claim benchmark accuracy establishes deployment reliability |
| Leakage inflating performance | Apply all [ML leakage controls](ml-pipeline.md); audit splits and feature allowlists; preserve immutable test holdouts |
| Untrusted model artifact execution or replacement | Load only locally produced or explicitly reviewed artifacts; enforce provenance and integrity checks; restrict artifact writes; a checksum alone does not make a hostile artifact safe |
| Unauthorized evidence access or case edits | Authenticate API access, apply role/workspace authorization to every reference, use least-privilege DB accounts, audit mutations; CORS is not authentication |
| Sensitive data disclosure | Minimize retained fields, encrypt deployed traffic/storage, redact logs and AI inputs, define retention/deletion rules, restrict raw evidence access |
| Prompt injection through telemetry/enrichment | Delimit evidence as data; never treat it as instructions; allowlist retrieved fields; disable tool execution; validate structured outputs and evidence references |
| Hallucinated explanation or exaggerated certainty | Require evidence IDs for factual claims, validate references, show uncertainty and abstain when unsupported; LLMs never provide authoritative detection or approval |
| Unapproved response or approval replay | Default to analyst approval; bind approval to exact action, target, evidence and parameters; expiry, authorization, and immutable audit record; executor rejects changed/replayed approval |
| Resource exhaustion and dependency compromise | Bound batch jobs and requests, add quotas/timeouts and monitoring, review/pin dependencies during implementation, protect secrets, scan images/artifacts before deployment |

Do not download, detonate, or execute malware for this foundation. Any future dynamic analysis needs a separate isolated sandbox and reviewed procedures; it must never run on the development host.

## AI and response constraints

Prefer deterministic explanations in Phase 1: reason codes, model/version, relevant features, score semantics, and evidence IDs. If an LLM is later enabled, redact identifiers and secrets, review external data-sharing terms, provide only authorized evidence, and record prompt/template/model versions. Generated text cannot modify raw evidence, detection scores, case state, or response permissions. Feature attribution is not proof of causality.

Response proposals, approvals, and execution results are separate records. Approval requires an authenticated analyst role and explicit action review; a model output, chat message, or uploaded field is never approval. High-impact actions may need additional reviewers. Phase 1 has no executor.

## Verification and residual risks

Before implementation is exposed, verify malformed input handling, duplicate/replay behavior, cross-workspace access rejection, safe dashboard rendering/export, artifact provenance enforcement, leakage assertions, grounded explanation failures, and approval bypass rejection where those capabilities exist. Development uses synthetic or appropriately de-identified data.

Residual risks include noisy labels, unseen attacks, missing asset context, concept drift, compromised analysts, and malicious text surviving validation. Confidence and limitations must remain visible. Exact upload limits, retention periods, auth provider, and deployment policy require review before exposure; do not assume documentation provides these controls.
