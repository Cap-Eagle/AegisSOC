# Deployment

Status: planned topology and configuration only. There is no Dockerfile, Compose file, dependency manifest, database migration, or runnable application. Do not run or advertise startup commands until implementation is approved.

## Initial local topology

Use Docker and Docker Compose later to coordinate one modular Python/FastAPI backend, PostgreSQL, and a React/TypeScript/Vite frontend development environment. The backend can run offline batch training/ingestion commands from the same codebase; a broker or separate worker service is not initially required. Keep long-running processing out of request handlers and define durable job handling when implemented.

Bind exposed development ports to loopback. PostgreSQL should be reachable only by the backend within the Compose network by default. Use a named database volume and explicit ignored dataset/artifact/upload directories. Vite's development server is for development; a deployed dashboard needs an appropriate static-serving arrangement and TLS boundary.

## Configuration template

[.env.example](../.env.example) documents proposed variable names. No loader consumes them yet.

| Variables | Intended use |
| --- | --- |
| `AEGIS_ENV`, `AEGIS_LOG_LEVEL` | Environment name and bounded log level |
| `API_HOST`, `API_PORT`, `CORS_ALLOWED_ORIGINS` | Local bind and explicit allowed browser origins |
| `POSTGRES_HOST/PORT/DB/USER/PASSWORD` | Backend database connection fields; password is a placeholder |
| `VITE_API_BASE_URL` | Public browser API location; never a credential channel |
| `DATASET_DIR`, `ARTIFACT_DIR`, `UPLOAD_DIR` | Local storage roots outside version control |
| `AI_EXPLANATIONS_ENABLED` | Default `false`; Phase 1 has deterministic evidence explanations |
| `RESPONSE_APPROVAL_REQUIRED` | Default `true`; server-side authorization remains mandatory |
| `OPENAI_API_KEY` | Empty, optional future server-side secret; no Phase 1 dependency |

Host-run tools use `127.0.0.1` for the database. Inside future Compose, set `POSTGRES_HOST=postgres`; the backend binds `0.0.0.0` inside its container while host publication remains loopback-only. Browser URLs refer to a host-accessible address, not internal Compose service names. Validate configuration at startup and fail on placeholder credentials in deployed environments. Never log connection strings, raw telemetry, or keys.

Store real secrets in an ignored `.env` for local development and an appropriate secret mechanism for deployment. Browser-exposed `VITE_*` values are public. A configuration flag is not a response authorization mechanism; any later relaxation of approval requires explicit reviewed policy and an ADR.

## Planned operational controls

- Run containers as non-root, grant only necessary filesystem permissions, restrict network access, and keep input storage separate from approved artifact storage. Never mount a malware execution capability into the development host.
- Pin/review dependencies and base images when selected; build reproducible artifacts and record versions. Do not load untrusted pickle/joblib artifacts.
- Use a restricted application DB role and a separately controlled migration role. Apply versioned migrations, back up before destructive changes, and test restore procedures.
- Protect telemetry, database volumes, backups, and artifacts with access controls and deployment-appropriate encryption. Define retention and deletion policy before real data is accepted.
- Provide liveness/readiness checks, request/job IDs, safe structured logs, ingestion failure counts, inference latency and model identity. Later add drift and delayed-label performance monitoring per [ML pipeline](ml-pipeline.md).
- Promote only artifacts with evaluation reports and approved manifests. Keep previous artifacts and compatible feature definitions for rollback; write new results with new versions instead of overwriting history.
- Treat retry/restart as normal: resume durable jobs from checkpoints and enforce idempotent writes. Back up evidence and audit records together with relationship integrity.

Authentication, retention periods, quotas, exact storage layout, dependency versions, migration tooling, and production hosting remain review decisions. This development plan does not assert production readiness. Required controls are described in [the threat model](threat-model.md).

## Evolution

Adopt Kafka, Redis, OpenSearch, Neo4j, MLflow, or separate deployment units only after measured need and an ADR. Add streaming adapters without coupling transforms to a broker. Review infrastructure changes for evidence retention, replay, privacy, access scope, model rollback, and operational cost. Dynamic malware analysis, if ever needed, belongs in separately reviewed isolated infrastructure, never on the development host.
