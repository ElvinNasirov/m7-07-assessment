# Container plan — recommendation serving image

## Bake vs. mount

| Artifact | Decision | Why |
|---|---|---|
| ONNX ranker (`/app/models/ranker.onnx`, ~180 MB) | **Bake into image** | Immutable, self-contained deploys. The `ghcr.io/<repo>:<git-sha>` tag fully determines the served model, so rollback to the previous successful version is just redeploying the previous tag. The model version is exposed via `X-Model-Version`. |
| Product catalog / candidate index | **Do not bake — retrieve at runtime** | Large and changes far more often than the model. Served from the online **candidate store/cache**. Baking it would bloat images and couple catalog freshness to release cadence. |
| Last-30-day user features | **Do not bake — retrieve at runtime** | Per-user and time-sensitive; read from the online **feature store/cache**. |

This matches [ADR 0002](../architecture/adr/0002-bake-ranker-mount-catalog.md) and the [capacity plan](../serving/capacity-plan.md).

## Base image

`python:3.11-slim`. Multi-stage build: a **builder** stage compiles pinned dependencies into wheels; the **runtime** stage installs from those wheels with no build toolchain, then copies only the app code and the baked model.

## Size estimate

| Layer | Approx. size |
|---|---|
| `python:3.11-slim` base | ~120–150 MB |
| Runtime deps (onnxruntime, web framework, etc.) | ~60–120 MB |
| Baked ONNX ranker | ~180 MB |
| **Total** | **~350–450 MB** |

## Security notes

- **Non-root:** runs as a dedicated `app` user, not root.
- **Slim base:** minimal surface; no compilers or build tools in the runtime stage.
- **Pinned deps:** `requirements.txt` is version-pinned (hash-pinned in CI) and installed offline from prebuilt wheels.
- **Scanned in CI:** Trivy scans the image and **fails on HIGH/CRITICAL** before push (see [`cicd/.github/workflows/deploy-model.yml`](../cicd/.github/workflows/deploy-model.yml)).
- **HEALTHCHECK:** hits `/health` so orchestration can gate traffic on readiness.
