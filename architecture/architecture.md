# Architecture — Retail Recommendations (Scenario X)

The system has two planes. The **online plane** answers home-screen requests synchronously inside the 120 ms p95 budget. The **offline plane** plus the **registry/CI/CD plane** produce candidates, features, and the ranking model that the online plane consumes. A monitoring and A/B loop closes feedback from production back into the offline plane.

```mermaid
flowchart LR
  subgraph ONLINE["Online plane (synchronous, 120 ms p95 @ 800 RPS)"]
    APP["Mobile app"]
    GW["API gateway<br/>auth + routing"]
    API["Recommendation API<br/>baked ONNX ranker"]
    FS["Online feature store/cache<br/>last-30d user features"]
    CS["Candidate store/cache<br/>product candidates"]
    FB["Fallback<br/>popularity by segment"]
  end

  subgraph OFFLINE["Offline plane (batch)"]
    EV["Event stream"]
    DL["Data lake / warehouse"]
    FP["Feature + candidate pipeline"]
    TR["Training / evaluation"]
  end

  subgraph CTRL["Registry / CI/CD plane"]
    REG["Model registry<br/>retail-recs-models"]
    CD["CI/CD deploy<br/>ghcr.io/&lt;repo&gt;:&lt;git-sha&gt;"]
  end

  MON["Monitoring + A/B analysis"]

  APP -->|"request: user_id, context, top_k"| GW
  GW -->|"validated request + API key"| API
  API -->|"user_id"| FS
  FS -->|"30d feature vector"| API
  API -->|"candidate query"| CS
  CS -->|"candidate item set"| API
  API -.->|"on miss / degraded"| FB
  API -->|"ranked items + X-Model-Version + X-Experiment-ID"| APP

  APP -->|"impressions, clicks, purchases"| EV
  EV -->|"raw events"| DL
  DL -->|"dataset_hash"| FP
  FP -->|"features -> FS / candidates -> CS"| FS
  FP -->|"candidates"| CS
  FP -->|"training set"| TR
  TR -->|"model_uri + evaluation_report"| REG
  REG -->|"approved deployed_version"| CD
  CD -->|"image + ranker.onnx"| API

  API -->|"latency, errors, served model_version"| MON
  APP -->|"CTR / conversion by experiment"| MON
  MON -->|"drift_signal + A/B result"| FP
  MON -.->|"rollback trigger"| CD
```

## Online path (request flow)

Mobile app → API gateway → recommendation API → online feature store → candidate store → ranking model → recommendations response. The ranker is **baked into the image**; candidates and user features are **fetched from stores/caches** (not baked). On a cache miss or store degradation the API serves the **fallback** (popularity-by-segment) so cold-start users and degraded mode still get reasonable results.

## Offline path (production flow)

Event stream → data lake/warehouse → feature + candidate pipeline → training/evaluation → model registry → deployment. The pipeline writes user features to the online feature store and candidate sets to the candidate store; training produces a `model_uri` and `evaluation_report` that land in the registry.

## Boundaries

- **Online plane**: latency-critical, stateless replicas, must stay inside 120 ms p95.
- **Offline plane**: throughput-oriented batch, no latency SLO.
- **Registry/CI/CD plane**: source of truth for the approved model version and the image that ships it.

## Feedback loop

Monitoring collects latency, errors, and the **served `model_version`**; the app reports CTR/conversion tagged by experiment. The A/B analysis and drift detection feed the offline pipeline (retraining) and can fire a **rollback trigger** into CI/CD when guardrails break (see [`monitoring/alerts.yaml`](../monitoring/alerts.yaml) and [`runbooks/rollback.md`](../runbooks/rollback.md)).
