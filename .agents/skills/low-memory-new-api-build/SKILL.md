---
name: low-memory-new-api-build
description: Build and deploy new-api on low-memory servers where the full Dockerfile frontend stages may OOM. Use when Docker builds fail with exit 137, SIGKILL, JavaScript heap out of memory, web/classic Vite build failures, or when deploying local frontend changes on 4GB RAM plus swap servers.
---

# Low-Memory new-api Build

Use this workflow when a normal `docker build -t new-api:... .` fails during frontend builds, especially `web/classic`.

## Failure Signals

- Docker build exits with `exit code: 137`, `SIGKILL`, or `Killed`.
- `web/classic` build reaches `✓ 18030 modules transformed` then dies.
- Host build exits with `JavaScript heap out of memory`.
- The server has around 4GB RAM, even with swap enabled.

## Key Lessons

- Running the service is light enough; full image builds are the expensive part.
- The repository Dockerfile builds both frontends in Docker:
  - `web/default` with Rsbuild
  - `web/classic` with Vite
- `web/classic` may exceed Node's default heap limit. Retry host builds with:

```bash
NODE_OPTIONS='--max-old-space-size=4096' VITE_REACT_APP_VERSION=$(cat /opt/new-api/VERSION) bun run build
```

- If `VERSION` is empty, do not rely on `$(cat VERSION)` for `-ldflags`; inject an explicit local version label.
- Ensure Docker build context excludes frontend dependencies:

```dockerignore
/web/default/node_modules
/web/classic/node_modules
```

## Recommended Deployment Workflow

### 1. Verify current runtime config

Capture existing container settings before replacing it:

```bash
docker ps --filter name='^/new-api$'
docker inspect new-api
```

Preserve:

- database DSN
- Redis connection string
- data/log volumes
- `SESSION_SECRET`
- Docker network
- port binding

### 2. Build frontends on the host

Default frontend:

```bash
cd /opt/new-api/web/default
bun install
VITE_REACT_APP_VERSION=$(cat /opt/new-api/VERSION) bun run build
```

Classic frontend:

```bash
cd /opt/new-api/web/classic
bun install
NODE_OPTIONS='--max-old-space-size=4096' VITE_REACT_APP_VERSION=$(cat /opt/new-api/VERSION) bun run build
```

If `bun install` changes a lockfile only due to Bun metadata churn, inspect the diff and revert the unrelated lockfile change.

### 3. Build a Go-only Docker image

Use prebuilt `web/default/dist` and `web/classic/dist` instead of rebuilding frontends inside Docker:

```bash
docker build --build-arg APP_VERSION='local-build' -t new-api:local-build -f - /opt/new-api <<'EOF'
FROM golang:1.26.1-alpine@sha256:2389ebfa5b7f43eeafbd6be0c3700cc46690ef842ad962f6c5bd6be49ed82039 AS builder
ENV GO111MODULE=on CGO_ENABLED=0
ARG TARGETOS
ARG TARGETARCH
ARG APP_VERSION=dev
ENV GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH:-amd64}
ENV GOEXPERIMENT=greenteagc
WORKDIR /build
ADD go.mod go.sum ./
RUN go mod download
COPY . .
RUN test -f web/default/dist/index.html && test -f web/classic/dist/index.html
RUN go build -ldflags "-s -w -X github.com/QuantumNous/new-api/common.Version=${APP_VERSION}" -o new-api

FROM debian:bookworm-slim@sha256:f06537653ac770703bc45b4b113475bd402f451e85223f0f2837acbf89ab020a
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates tzdata libasan8 wget \
    && rm -rf /var/lib/apt/lists/* \
    && update-ca-certificates
COPY --from=builder /build/new-api /
EXPOSE 3000
WORKDIR /data
ENTRYPOINT ["/new-api"]
EOF
```

### 4. Smoke-test with a candidate container

Run the new image on `127.0.0.1:3001` first, using the same DB/Redis/volumes:

```bash
docker run -d --name new-api-candidate \
  --network <network> \
  -p 127.0.0.1:3001:3000 \
  -v newapi-data:/data \
  -v newapi-logs:/app/logs \
  -e SQL_DSN='<postgres-dsn>' \
  -e REDIS_CONN_STRING='<redis-url>' \
  -e TZ='Asia/Shanghai' \
  -e ERROR_LOG_ENABLED='true' \
  -e BATCH_UPDATE_ENABLED='true' \
  -e NODE_NAME='new-api-node-1' \
  -e SESSION_SECRET='<existing-session-secret>' \
  new-api:local-build --log-dir /app/logs
```

Validate:

```bash
curl -fsS http://127.0.0.1:3001/api/status
docker logs --since 30s new-api-candidate
```

Confirm `success: true`, Redis enabled, PostgreSQL selected, and a non-empty version label.

### 5. Replace production container

Prefer binding the app only to localhost when OpenResty or another reverse proxy handles public traffic:

```bash
docker stop new-api
docker rm new-api
docker run -d --name new-api --restart always \
  --network <network> \
  -p 127.0.0.1:3000:3000 \
  -v newapi-data:/data \
  -v newapi-logs:/app/logs \
  -e SQL_DSN='<postgres-dsn>' \
  -e REDIS_CONN_STRING='<redis-url>' \
  -e TZ='Asia/Shanghai' \
  -e ERROR_LOG_ENABLED='true' \
  -e BATCH_UPDATE_ENABLED='true' \
  -e NODE_NAME='new-api-node-1' \
  -e SESSION_SECRET='<existing-session-secret>' \
  new-api:local-build --log-dir /app/logs
```

Then remove the candidate:

```bash
docker rm -f new-api-candidate
```

## Final Verification

Check direct local service:

```bash
curl -fsS http://127.0.0.1:3000/api/status
```

Check reverse proxy domains with `Host` headers when testing locally:

```bash
curl -k -fsS -H 'Host: redeveloz.com' https://127.0.0.1/api/status
curl -k -fsS -H 'Host: www.redeveloz.com' https://127.0.0.1/api/status
```

Report:

- image tag deployed
- container port binding
- status endpoint result
- any workaround used for memory limits
