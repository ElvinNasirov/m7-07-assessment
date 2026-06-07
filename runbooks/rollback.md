# Runbook — Roll back the recommendation model

Target: restore the **previous successful production model version**. Rollback redeploys the previous good `ghcr.io/<repo>:<git-sha>` image; the model is baked in, so this is a single, atomic change.

## When to roll back

Roll back when any of these fire (matches [`monitoring/alerts.yaml`](../monitoring/alerts.yaml)):

- [ ] **Availability burn page** — `AvailabilityBurnFast` (fast error-budget burn on 99.7% SLO).
- [ ] **p95 > 250 ms for 15 min** — `LatencyP99High`.
- [ ] **Model version mismatch** — `ModelVersionMismatch` (served `X-Model-Version` ≠ registry production version).
- [ ] **CTR/conversion guardrail regression** — `ABTestMetricRegression` (treatment >1pp below control).
- [ ] **Feature freshness breach** — `FeatureFreshnessStale` (p99 snapshot age > 15 min) **if** it is caused by the new release rather than an upstream pipeline outage.

## How to roll back

```bash
# Option A: direct script
./scripts/rollback.sh production

# Option B: via CI/CD workflow
gh workflow run deploy-model.yml -f action=rollback -f environment=production
```

## What to verify

- [ ] Served `X-Model-Version` now equals the previous good version on all replicas.
- [ ] `ModelVersionMismatch` and the triggering alert have cleared.
- [ ] Sync p95 back under 120 ms; p99 under 250 ms.
- [ ] 5xx rate back under 0.3%; availability burn stopped.
- [ ] Fallback rate back to baseline (not masking a dependency outage).

## Who to notify

- [ ] On-call (page acknowledged) and **ML Lead**.
- [ ] **Product Owner** if an active A/B experiment was affected (pause/annotate the experiment).
- [ ] **Platform Owner** if the cause is infra (stores, autoscaling, capacity).
- [ ] Post a short incident note in the team channel with the alert, action taken, and current state.

## What not to do

- [ ] Do not push a brand-new "fix-forward" model straight to production during an active incident.
- [ ] Do not edit the catalog/candidate store or feature store to mask a model regression.
- [ ] Do not skip the canary on the next deploy or bypass the registry promotion gates.
- [ ] Do not change SLO thresholds to silence the alert.

## When to roll forward

- [ ] Root cause is understood and fixed, and the fix has passed **staging smoke + canary verify**.
- [ ] Triggering metrics have been stable on the rolled-back version for a full evaluation window.
- [ ] A new version is promoted through the normal gates (offline quality, latency sim ≤ 120 ms, cold-start guardrail, schema, canary) with ML Lead + Product Owner + Platform Owner approval.
