# Justification — pattern and trade-offs

## Online synchronous serving as the main pattern

The home screen renders recommendations on load, so the result is on the critical path of a user-visible screen. Precomputing a full per-user list nightly would be stale (it ignores the current session and "just browsed" intent) and wasteful for users who never open the app that day. We therefore serve **online and synchronously**: the API assembles fresh features + candidates and ranks them per request. The 120 ms p95 budget is tight but achievable because the expensive parts (candidate generation, feature computation) are precomputed offline; the online step is a lookup + a small ranking model.

## Offline batch candidate generation

Generating candidates from the full catalog at request time is too slow and too variable for a 120 ms budget. Instead a batch pipeline produces a bounded candidate set per user/segment (recently viewed, co-purchased, trending in segment) and writes it to the **candidate store**. The online ranker then only scores a few hundred items, which keeps inference cheap and predictable. User features for the last 30 days are similarly precomputed and served from the **online feature store/cache**.

## Cold-start and fallback

Users with no 30-day history (or any store miss / dependency timeout) must still see something sensible. We keep a deterministic **fallback** — popularity-by-segment using whatever first-session context exists (locale, entry point). The same fallback is the **degraded mode** when the feature store or candidate store is unhealthy, so a dependency outage downgrades quality instead of returning an error. This protects the availability SLO.

## Cloud CPU inference

The ranker is a small ONNX model (~180 MB) scoring a few hundred candidates. That workload is CPU-friendly: per-request compute is low and parallelism comes from horizontal replicas, not from a single large batched matmul. AWS Graviton `c7g.xlarge` gives good price/performance, scales linearly with replicas, and avoids GPU cold-start, scheduling, and cost overhead we don't need. GPU would only pay off with a much larger model or heavy batching, neither of which fits a 120 ms synchronous path. Details in [`serving/capacity-plan.md`](../serving/capacity-plan.md).

## Latency / throughput / cost trade-off

- **Latency** is the binding constraint (120 ms p95). We spend the budget on the steps we can't precompute (feature lookup, candidate retrieval, ranking) and keep ~44 ms headroom for GC, tail variance, and warm buffer.
- **Throughput** comes from stateless replicas: ~120 RPS/replica → 9 replicas at peak with a 1.3 safety factor, 11 with a warm buffer for canary/A-B overlap.
- **Cost** stays modest (~$1.2k/month compute) precisely because we chose CPU and pushed heavy work offline. We accept slightly less per-request modeling power in exchange for predictable tail latency, simple scaling, and low cost. Dynamic batching is deliberately avoided on the sync path; see the capacity plan.
