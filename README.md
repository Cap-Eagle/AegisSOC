# AegisSOC

AegisSOC is a context-aware, AI-assisted Security Operations Center platform. It is designed to turn telemetry into detections and evidence-backed investigations while keeping analysts in control.

**Current status: documentation foundation awaiting review.** No backend, ML pipeline, frontend, database schema, or Docker stack is implemented. The contracts and configuration below describe intended behavior; they are not runnable services.

## Phase 1

```text
CSV dataset → validation → preprocessing → feature engineering
            → binary classifier → risk score → REST API
            → PostgreSQL → SOC dashboard
```

The first target is `0 = BENIGN`, `1 = MALICIOUS`. Start with Logistic Regression, compare Random Forest, and consider XGBoost/LightGBM only when measured results justify the added complexity. Use one modular backend and one frontend; keep inference and training batch-based initially.

## Core rules

- **Event:** raw or normalized telemetry. **Alert:** a detection from one or more events. **Incident:** a correlated case containing related alerts and events.
- LLMs never serve as the primary threat detector. AI explanations cite structured evidence and identify uncertainty.
- All telemetry, uploaded datasets, enrichment, and model inputs are untrusted.
- Automated response actions require analyst approval by default. Never execute malware on the development host.
- Threat Severity and Evidence Confidence are separate; a classifier score is neither by itself.
- Prevent train/test, duplicate-flow, target, and preprocessing leakage. Report precision, recall, F1, PR-AUC, ROC-AUC, false-positive rate, false-negative rate, and the confusion matrix.
- Design batch boundaries for later streaming, and record every major architectural change in an ADR.

## Planned stack

| Area | Initial technologies |
| --- | --- |
| Backend | Python, FastAPI, Pydantic, SQLAlchemy |
| ML | NumPy, Pandas or Polars, scikit-learn |
| Database | PostgreSQL |
| Frontend | React, TypeScript, Vite |
| Infrastructure | Docker, Docker Compose |

Future candidates: XGBoost/LightGBM, Kafka, Redis, OpenSearch, Neo4j, MLflow, SHAP, Zeek, and Suricata. None is a Phase 1 dependency. The dataframe library choice and concrete dependency versions remain implementation decisions.

## Documentation

| Document | Purpose |
| --- | --- |
| [Agent instructions](AGENTS.md) | Required reading, architecture rules, tool usage, and Beads workflow |
| [Architecture](docs/architecture.md) | Phase 1 boundaries and long-term capability map |
| [Event schema](docs/event-schema.md) | Event, Alert, Incident, provenance, and relationships |
| [Threat model](docs/threat-model.md) | Trust boundaries, abuse cases, and required controls |
| [ML pipeline](docs/ml-pipeline.md) | Dataset validation, leakage controls, training, and evaluation |
| [Risk engine](docs/risk-engine.md) | Threat Severity, Evidence Confidence, and explainable scoring |
| [API contracts](docs/api-contracts.md) | Proposed REST resources, errors, and authorization |
| [Deployment](docs/deployment.md) | Planned local topology, configuration, and operational controls |
| [ADR 001](docs/decisions/001-event-alert-incident-model.md) | Why telemetry, detections, and cases stay separate |

## Working in this repository

Read [AGENTS.md](AGENTS.md) before making changes. Use `bd prime` for Beads context and `bd ready` to find tracked work. This foundation does not authorize application implementation; review and approval are the next step. Do not commit, push, or sync remotely without authorization.

[.env.example](.env.example) contains non-secret placeholders for future configuration. Keep real `.env` files, telemetry, datasets, trained artifacts, credentials, and local database volumes out of Git. There are no installation, build, test, or startup commands yet because no application has been created.
