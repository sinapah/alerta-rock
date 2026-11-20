# Alerta rock

This repository holds a Rockcraft project that builds an OCI image (a "rock") of Alerta 9.1.0 on Ubuntu 24.04 (amd64). The rock is meant to be used by a future Juju Kubernetes charm.

## Contents of the rock

| Component | Version | Source |
| --- | --- | --- |
| `alerta-server` | 9.1.0 | git tag `v9.1.0` of https://github.com/alerta/alerta |
| `alerta` (CLI) | 8.5.3 | PyPI |
| `alerta-webui` | 8.7.1 | GitHub release tarball |
| `nginx` | Ubuntu 24.04 package | apt |
| `uWSGI` | 2.0.31 | PyPI |

The `alerta-contrib` plugins are not bundled.

## Running the rock

Pebble supervises the following services:

| Service | Startup | Purpose |
| --- | --- | --- |
| `uwsgi` | enabled | Alerta API server |
| `nginx` | enabled | Serves the web UI and proxies `/api` to uWSGI |
| `housekeeping` | disabled | Periodic deletion of expired and informational alerts |
| `heartbeats` | disabled | Periodic alerting on stale heartbeats |

- Port `8080` serves the web UI at `/` and the API at `/api`.
- All services run as the `_daemon_` user.
- The readiness check `alerta-ready` calls `GET /api/management/gtg`.
- `/etc/alerta/nginx.conf` and `/etc/alerta/uwsgi.ini` can be replaced by a charm with a Pebble push.
- The web UI configuration is `/srv/alerta/web/config.json`.

### Configuration

The server is configured with environment variables. The ones most relevant to a charm are:

- `DATABASE_URL`
- `DATABASE_NAME`
- `SECRET_KEY`
- `AUTH_REQUIRED`
- `ADMIN_USERS`
- `AUTH_PROVIDER`
- `PLUGINS`
- `CORS_ORIGINS`
- `BASE_URL`

See https://docs.alerta.io/configuration.html for the full reference.

The `housekeeping` and `heartbeats` services use the Alerta CLI, which reads `ALERTA_ENDPOINT` (default `http://localhost:8080/api`) and `ALERTA_API_KEY` (required when `AUTH_REQUIRED` is on). They are tuned with `HK_EXPIRED_DELETE_HRS` (default 2), `HK_INFO_DELETE_HRS` (default 12) and `HEARTBEAT_SEVERITY` (default `major`).

### Example

```bash
docker network create alerta
docker run -d --name alerta-db --network alerta \
  -e POSTGRES_PASSWORD=alerta -e POSTGRES_DB=alerta postgres:16
docker run -d --name alerta --network alerta -p 8080:8080 \
  -e DATABASE_URL=postgresql://postgres:alerta@alerta-db:5432/alerta \
  -e SECRET_KEY=change-me \
  alerta:9.1.0
```

## Repository layout

```
.github/
  ci.yaml
  workflows/
9.1.0/
  rockcraft.yaml
  rootfs/
  spread.yaml
  spread/
Makefile
README.md
SECURITY.md
```

## Building and testing locally

Prerequisites:

```bash
sudo snap install rockcraft --classic
sudo snap install lxd
lxd init --auto
sudo snap install docker  # only needed to run the image
```

Build and test:

```bash
cd 9.1.0
rockcraft pack
rockcraft test
```

`make test-all` from the repository root tests every rock that has a `spread.yaml`.

Load the image into Docker:

```bash
sudo rockcraft.skopeo --insecure-policy copy oci-archive:alerta_9.1.0_amd64.rock docker-daemon:alerta:9.1.0
```

## CI

GitHub Actions build the rock, test it with spread and push it to GHCR on `main`, and a daily job scans the published images for CVEs. The behaviour is controlled by `.github/ci.yaml`; see https://github.com/canonical/rocks-template-actions for the schema.

## Security

See `SECURITY.md`.
