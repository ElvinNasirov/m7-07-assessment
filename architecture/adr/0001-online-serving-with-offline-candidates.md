# ADR 0001: Online synchronous serving with offline candidate generation

## Context

Recommendations render on every home-screen load at ~800 RPS peak with a 120 ms p95 end-to-end budget. Personalization needs the user's last 30 days of activity and should reflect current session intent. The catalog is large, so scoring everything per request is infeasible inside the budget. Cold-start users must still get reasonable results.

## Decision

Serve recommendations **online and synchronously** through `/v1/recommendations`. Push the expensive work **offline**: batch pipelines generate a bounded candidate set per user/segment (candidate store) and compute last-30-day user features (online feature store/cache). At request time the API only does a feature lookup, a candidate lookup, and a small ONNX ranking pass over a few hundred candidates, then applies business rules. Cold-start and dependency degradation fall back to popularity-by-segment.

## Alternatives rejected

- **Fully precomputed lists (offline batch serving):** cheapest and fastest to serve, but stale, ignores current session, and wastes compute on inactive users. Rejected because freshness and session intent matter on the home screen.
- **Full online retrieval + ranking over the whole catalog per request:** most flexible, but cannot meet 120 ms p95 at 800 RPS without large cost. Rejected on latency/cost.
- **Pure on-device model:** removes server latency but cannot use fresh server-side signals or be A/B tested centrally, and complicates updates. Rejected.

## Consequences

- Predictable, cheap online step → 120 ms p95 is achievable on CPU.
- Two-plane system to operate: online serving plus offline candidate/feature pipelines, each with its own freshness expectations.
- Candidate/feature staleness becomes an SLO (`recommendation_freshness`, snapshots ≤ 15 min).
- Cold-start quality is capped by the fallback; needs its own guardrail metric.

## Revisit if

- p95 budget loosens materially (e.g. > 300 ms) — full online retrieval may become viable.
- Catalog or candidate generation becomes cheap enough to do online.
- Session-intent signals prove far more valuable than 30-day history, pushing more work to request time.
