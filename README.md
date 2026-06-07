# Retail Recommendations — MLOps Design Dossier

## Executive summary

This repository is the build-ready design for **Scenario X — personalized in-app product recommendations for a B2C retail mobile app**. On every home-screen load the app calls a synchronous `/v1/recommendations` endpoint that ranks a small candidate set with a baked-in ONNX ranker and returns a personalized list within a 120 ms p95 budget at 800 RPS peak. Heavy work is pushed offline: batch pipelines generate candidates and user/product features, an online feature store serves the last-30-day user signals, and a deterministic fallback covers cold-start users and degraded mode. Serving runs on CPU (AWS `c7g.xlarge`) because the ranker is small and CPU is cheaper and simpler to scale than GPU at this latency. Promotion is gated by offline quality, latency simulation, cold-start guardrails, and a canary, with rollback to the previous successful production model version.

## Architecture diagram

See [`architecture/architecture.md`](architecture/architecture.md) for the full Mermaid diagram (online path, offline path, registry/CI/CD boundaries, and the monitoring + A/B feedback loop).

## Key numbers

| Item | Value |
|---|---|
| Scenario | X — personalized in-app recommendations (B2C retail) |
| Peak RPS | 800 |
| p95 latency budget | 120 ms end-to-end |
| Hardware | AWS `c7g.xlarge` CPU (4 vCPU, 8 GiB) |
| Model size estimate | ~180 MB ONNX ranker |
| Replicas | 9 at peak, 11 with warm buffer |
| Monthly compute estimate | ~$1.2k (planning figure, not billing) |
| Availability SLO | 99.7% (internal, rolling 30d) |
| Latency SLO | 95% of successful sync requests < 120 ms |

## Navigation

| Area | Primary artifact |
|---|---|
| Architecture | [`architecture/architecture.md`](architecture/architecture.md), [`architecture/JUSTIFICATION.md`](architecture/JUSTIFICATION.md), [`architecture/adr/`](architecture/adr/) |
| Lifecycle & registry | [`lifecycle/lifecycle.md`](lifecycle/lifecycle.md), [`lifecycle/model-registry.yaml`](lifecycle/model-registry.yaml) |
| Container | [`container/Dockerfile`](container/Dockerfile), [`container/README.md`](container/README.md) |
| API contract | [`api/openapi.yaml`](api/openapi.yaml), [`api/examples/`](api/examples/) |
| Capacity & SLOs | [`serving/capacity-plan.md`](serving/capacity-plan.md), [`serving/slos.yaml`](serving/slos.yaml), [`serving/load-test-plan.md`](serving/load-test-plan.md) |
| CI/CD | [`cicd/.github/workflows/deploy-model.yml`](cicd/.github/workflows/deploy-model.yml) |
| Monitoring | [`monitoring/alerts.yaml`](monitoring/alerts.yaml) |
| Runbook | [`runbooks/rollback.md`](runbooks/rollback.md) |

## Validation

Lint the API contract (no Makefile in this repo):

```bash
npx @redocly/cli lint api/openapi.yaml
```

## Open questions

1. **Candidate store ownership** — who owns the candidate index freshness SLA, and is a 15-minute feature snapshot acceptable for purchase-driven recency, or do we need a streaming top-up for "just bought" signals?
2. **Cold-start signal** — for users with no 30-day history, do we have enough first-session context (device locale, entry campaign) to beat a popularity baseline, or is popularity-by-segment the honest ceiling?
3. **A/B traffic split control** — does the experiment ID come from the app/edge or from the recommendation API, and which system is the source of truth when an assignment conflict happens during a canary overlap?
