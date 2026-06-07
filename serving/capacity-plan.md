# Capacity plan — Retail Recommendations (Scenario X)

Target: **800 RPS peak**, **120 ms p95** end-to-end for the synchronous `/v1/recommendations` path on CPU (`c7g.xlarge`).

## Latency budget (sums to 120 ms)

| Stage | Budget (ms) |
|---|---|
| Network in | 5 |
| Auth + routing | 4 |
| Payload parse + validation | 3 |
| User feature lookup | 12 |
| Candidate retrieval | 10 |
| Ranker inference | 25 |
| Business rules / filtering | 8 |
| Serialization | 4 |
| Network out | 5 |
| Headroom | 44 |
| **Total** | **120** |

The 44 ms headroom absorbs GC pauses, tail variance, cache misses (which trigger the fallback), and per-replica warm-up jitter. The two store lookups (feature 12 ms, candidate 10 ms) and ranker inference (25 ms) are the load-bearing items; everything else is fixed overhead.

## CPU vs GPU decision

**Chosen: CPU on AWS `c7g.xlarge` (Graviton).** The ranker is small (~180 MB ONNX) and scores only a few hundred candidates per request, so per-request compute is low. Throughput scales by adding stateless replicas. GPU would add cold-start, scheduling, and cost overhead and only pays off with a much larger model or heavy batching — neither fits a 120 ms synchronous path. See [`../architecture/JUSTIFICATION.md`](../architecture/JUSTIFICATION.md).

## Cost assumptions

- `c7g.xlarge`: 4 vCPU, 8 GiB memory.
- ~**$0.145/hour** ≈ **$106/month** at 730 hours (on-demand, planning figure — not a billing claim).
- Sources:
  - https://aws.amazon.com/ec2/instance-types/c7g/
  - https://aws.amazon.com/ec2/pricing/on-demand/

## Replica sizing

Per-replica safe throughput: **120 RPS** at the p95 target. Formula: `replicas = ceil(target_rps * 1.3 / per_replica_throughput)` (1.3 = headroom for spikes/imbalance).

| Scenario | RPS | RPS/replica | Replicas | ~Monthly compute |
|---|---|---|---|---|
| Peak | 800 | 120 | `ceil(800 * 1.3 / 120) = 9` | 9 × $106 ≈ **$954** |
| Peak + warm buffer | 800 | 120 | 9 + 2 = **11** | 11 × $106 ≈ **$1,166** |

The 2 warm buffer replicas cover deploy/canary/A-B-test overlap so a canary doesn't eat into peak capacity. Planning estimate ≈ **$1.2k/month** compute.

## Batching decision

- **No dynamic batching on the main home-screen sync path** — the 120 ms p95 budget is too tight to absorb queue/wait time.
- Allow a **small internal microbatch (max size 4, max wait 5 ms) only under high load**, where it improves CPU efficiency without meaningfully hurting tail latency.
- The **batch endpoint** (`/v1/recommendations-batch`) and async endpoint are for partner/internal jobs, **not** the live app path.
