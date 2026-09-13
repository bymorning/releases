# ByMorning Gateway

Self-hosted ByMorning: the web UI and API in one container. Postgres-compatible
data is persisted with PGlite inside `/data`.

Image: `ghcr.io/bymorning/gateway`

This repository is the public install. You do not need the private source tree. You need Docker Compose. After it is up, open http://localhost:3210 and select **Continue locally**.

The MIT license in this repo covers these install files only. The image is not MIT.

## Requirements

- Docker Engine with Compose v2
- Port `3210` free on the host

## Install

```sh
git clone https://github.com/bymorning/releases.git
cd releases
docker compose up -d --wait
```

Compose pulls `ghcr.io/bymorning/gateway` and publishes the app at `127.0.0.1:3210`.

Open http://localhost:3210 — use `localhost`, not `127.0.0.1`, so it matches `APP_ORIGIN`. Select **Continue locally**.

Stop:

```sh
docker compose down
```

Data stays in the named volume. Add `--volumes` only if you intend to wipe `/data`.

## What you get

| Service | Role |
| --- | --- |
| `app` | UI + API on port `3210`, with PGlite under `/data/pglite` |

The gateway process runs migrations on start, writes a cookie key, PGlite files, and filesystem storage under `/data`, and listens on `0.0.0.0` inside the container.

## Configuration

Defaults in `compose.yml` are enough for a laptop install. Override with a Compose `environment` block or an `.env` file next to `compose.yml`.

| Variable | Default | Meaning |
| --- | --- | --- |
| `APP_ORIGIN` | `http://localhost:3210` | Browser origin. Must match the URL you open. Local Login only works for `http://localhost`, `http://127.0.0.1`, or `http://[::1]`. |
| `PORT` | `3210` | Listen port inside the container. If you publish a different host port, keep this at `3210` unless you also change the container port mapping and `APP_ORIGIN`. |
| `HOST` | `0.0.0.0` | Listen address inside the container. Leave this set so Docker can reach the process. |
| `BYMORNING_DATA_DIR` | `/data` | Cookie key, PGlite directory, object storage, and cache. Persist this volume. |
| `DATABASE_URL` | unset | Optional PostgreSQL URL instead of PGlite. Do not set this and a PGlite directory together. |

To use another host port (example `8080`):

```yaml
ports:
  - "127.0.0.1:8080:3210"
environment:
  APP_ORIGIN: http://localhost:8080
```

Open http://localhost:8080.

## Data

| Volume | Contents |
| --- | --- |
| `bymorning-gateway-data` | `/data` — cookie key, PGlite, files, cache |

Losing the cookie-key file signs everyone out. Losing the volume drops workspaces and configuration. Existing Compose Postgres volumes are not imported.

## Update

```sh
docker compose pull
docker compose up -d --wait
```

## Limits

- Local Login on loopback HTTP only. This compose file is for a machine you open at `http://localhost`.
- No code interpreter (no sandbox image, no Docker socket).
- No S3, no AWS, no SST.
- Public HTTPS and SSO are a different deployment.
