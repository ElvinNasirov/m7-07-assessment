# ADR 0002: Bake the ONNX ranker into the image, keep catalog/candidate data outside

## Context

The serving image ships a ranking model and must run at 800 RPS across 9–11 replicas with fast, repeatable deploys, canaries, and rollbacks. We have two kinds of data: a small, slowly-changing **ranking model** (~180 MB ONNX) and a large, frequently-changing **product catalog / candidate index**. We need clear versioning and the ability to roll back to a known-good model.

## Decision

**Bake** the ONNX ranker into the image at `/app/models/ranker.onnx`. The image is the unit of deploy and tagged `ghcr.io/<repo>:<git-sha>`; the model version is surfaced in the `X-Model-Version` response header and registry metadata. **Do not bake** the product catalog or candidate index — retrieve it at request time from the online **candidate store/cache**, and read user features from the online feature store.

## Alternatives rejected

- **Mount/download the model at runtime:** image stays smaller, but introduces a startup dependency and a way for the running model to drift from the image tag, weakening immutable deploys and rollback guarantees. Rejected.
- **Bake the catalog/candidate data into the image too:** would make images huge, force a full redeploy on every catalog change, and couple data freshness to release cadence. Rejected.

## Consequences

- Immutable, self-contained serving image → a git-sha tag fully determines the served model, making rollback to the previous successful version trivial.
- Image size grows by ~180 MB (estimated total 350–450 MB) — acceptable; see [`container/README.md`](../../container/README.md).
- Catalog/candidate freshness is decoupled from releases and owned by the offline pipeline + store.
- A new model requires a new image build + deploy (intended — that's the audit/rollback boundary).

## Revisit if

- The model grows large enough that baking bloats images or slows pulls/cold starts significantly.
- We need to hot-swap models without redeploying (e.g. very frequent model updates), which would favor a mounted-artifact pattern with its own versioning.
