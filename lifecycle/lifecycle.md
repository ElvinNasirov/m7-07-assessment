# Model lifecycle — home-screen-ranker

End-to-end flow from raw events to production and back into retraining. Artifacts are named on the arrows so lineage is traceable from a served response to the dataset and run that produced it. Steps are marked **[auto]** (pipeline-driven) or **[manual]** (human approval).

```mermaid
flowchart TD
  EV["Event data<br/>impressions, clicks, purchases"]
  FG["Feature + candidate generation [auto]"]
  TRN["Training run [auto]"]
  EVAL["Evaluation [auto]"]
  STG["Registry: staging [auto]"]
  CAN["Canary 10% [auto verify]"]
  APR["Promotion approval [manual]"]
  PROD["Registry: production [manual gate]"]
  MON["Monitoring + A/B [auto]"]
  RETR["Retraining trigger [auto]"]

  EV -->|"dataset_hash"| FG
  FG -->|"feature_schema_version"| TRN
  TRN -->|"run_id, model_uri"| EVAL
  EVAL -->|"evaluation_report"| STG
  STG -->|"model_uri (staging)"| CAN
  CAN -->|"canary result"| APR
  APR -->|"deployed_version"| PROD
  PROD -->|"deployed_version (served)"| MON
  MON -->|"drift_signal"| RETR
  RETR -->|"new dataset_hash"| FG
```

## Notes

- **Automatic:** feature/candidate generation, training, evaluation, staging registration, and canary verification all run from the pipeline on a schedule or on a retraining trigger.
- **Manual:** promotion from `staging` → `production` requires sign-off from ML Lead, Product Owner, and Platform Owner (see [`model-registry.yaml`](model-registry.yaml)).
- **Lineage:** every registered version carries `git_sha`, `dataset_hash`, `feature_schema_version`, `training_image_digest`, `run_id`, `random_seed`, `model_uri`, and `evaluation_report_uri`, so a served `X-Model-Version` traces back to the exact data and code.
- **Feedback:** `drift_signal` (data/score drift) and A/B regressions trigger retraining; a production guardrail breach triggers rollback to the previous successful production version (see [`runbooks/rollback.md`](../runbooks/rollback.md)).
