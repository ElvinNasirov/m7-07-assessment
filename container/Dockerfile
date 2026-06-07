# syntax=docker/dockerfile:1

# ---- Builder: compile dependencies into wheels ----
FROM python:3.11-slim AS builder

WORKDIR /build

# Pinned dependencies live in requirements.txt (hash-pinned in CI).
COPY requirements.txt .

# Build wheels so the runtime stage installs offline with no build toolchain.
RUN pip install --no-cache-dir --upgrade pip wheel \
    && pip wheel --no-cache-dir --wheel-dir=/build/wheels -r requirements.txt

# ---- Runtime: slim, non-root, only what we need to serve ----
FROM python:3.11-slim AS runtime

ARG MODEL_VERSION=unknown
LABEL model.source="registry://retail-recs-models/home-screen-ranker" \
      model.framework="onnx" \
      model.version="${MODEL_VERSION}" \
      maintainer="ml-platform@retail.example"

# Create an unprivileged user to run the service.
RUN groupadd --system app && useradd --system --gid app --home /app app

WORKDIR /app

# Install dependencies from prebuilt wheels only (no network, no compilers).
COPY --from=builder /build/wheels /wheels
COPY requirements.txt .
RUN pip install --no-cache-dir --no-index --find-links=/wheels -r requirements.txt \
    && rm -rf /wheels

# Copy application code and the baked ranking model.
COPY app/ /app/app/
COPY models/ranker.onnx /app/models/ranker.onnx

ENV MODEL_PATH=/app/models/ranker.onnx \
    MODEL_VERSION=${MODEL_VERSION} \
    PORT=8080

USER app
EXPOSE 8080

# Liveness/readiness against the app's /health endpoint.
HEALTHCHECK --interval=15s --timeout=3s --start-period=20s --retries=3 \
  CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8080/health').status==200 else 1)"

# App exposes /health and /v1/recommendations.
CMD ["python", "-m", "app.main"]
