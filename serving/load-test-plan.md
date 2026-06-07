# Load test plan — Retail Recommendations (Scenario X)

**Tool:** [k6](https://k6.io/). Run against a staging environment sized like production (11 replicas, `c7g.xlarge`). Goal: confirm the 120 ms p95 budget and replica math hold at 800 RPS, including a canary overlap.

## Phases

| Phase | Duration | Load |
|---|---|---|
| Warmup | 10 min | 100 RPS |
| Ramp | 20 min | 100 → 800 RPS |
| Sustained peak | 60 min | 800 RPS |
| Canary overlap | 20 min | 800 RPS, 10% routed to canary build |
| Soak | 4 h | 500 RPS |
| Cooldown | 10 min | 100 RPS |

## Traffic shape

- **Endpoint mix:** 90% `/v1/recommendations`, 8% `/v1/recommendations-batch`, 2% `/v1/recommendations-async`.
- **User mix:** 80% returning users (have 30-day features), 20% cold-start (exercise the fallback path).
- API key sent in `X-API-Key`; vary `user_id`/`session_id` to avoid cache-only hot paths.

## Pass/fail criteria

| Metric | Threshold |
|---|---|
| p95 latency (sync) | ≤ 120 ms |
| Rolling-hour p95 | ≤ 250 ms |
| Availability | ≥ 99.7% |
| 5xx rate | ≤ 0.3% |
| Feature lookup p95 | ≤ 12 ms |
| Model version mismatch | = 0 (served `X-Model-Version` matches expected) |

A run fails if any threshold is breached during the sustained-peak or canary-overlap phases.

## Bottleneck checklist

- [ ] **Ranker inference** — p95 of the inference stage near/over its 25 ms budget? Check CPU saturation per replica.
- [ ] **Feature store** — feature lookup p95 over 12 ms or error/timeout rate rising? Check cache hit ratio and connection pool.
- [ ] **Candidate store** — candidate retrieval over 10 ms? Check index size and hot keys.
- [ ] **CPU saturation** — replica CPU pegged at peak? Validates the 120 RPS/replica assumption and 1.3 headroom factor.
- [ ] **Autoscaling** — does scale-out reach 9 (and use the 2 buffer) replicas before SLO burn during ramp?
- [ ] **Fallback rate** — fallback share spiking under load (store timeouts)? Indicates degraded-mode pressure, not real cold-start.
- [ ] **Tail / GC** — p99 vs p95 gap widening? Suggests GC/allocation pressure eating headroom.
- [ ] **Canary parity** — canary p95 and error rate within tolerance of stable during the overlap phase.
