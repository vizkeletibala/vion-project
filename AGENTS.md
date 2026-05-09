# AGENTS.md

## Purpose

This file is the first-stop handoff for future Codex sessions working in this repository.

This repo contains two related concerns:

- the shared local platform stack in the repo root
- the example application repo in `vion-arena/`, used to exercise CI/CD, routing, metrics, and logs

If you are picking up work here in a new session, read this file first and then open the files listed below in order.

## Read First

Platform:

1. `README.md`
2. `DOC.md`
3. `docs/platform/platform-overview.md`
4. `docs/platform/local-development.md`
5. `docs/platform/observability.md`
6. `docker-compose.yaml`

Application example:

1. `vion-arena/AGENTS.md`
2. `vion-arena/README.md`
3. `vion-arena/Jenkinsfile`
4. `vion-arena/deploy/docker-compose.app.yml`

Grafana and dashboards:

1. `platform-stack/grafana/provisioning/datasources/datasources.yaml`
2. `platform-stack/grafana/provisioning/dashboards/dashboards.yaml`
3. `platform-stack/grafana/provisioning/dashboards/vion-arena/app-health.json`
4. `platform-stack/grafana/provisioning/dashboards/vion-arena/containers-health.json`

## Current Platform State

### Core services

The platform stack currently includes:

- Traefik
- Registry
- Jenkins
- Prometheus
- Grafana
- Loki
- Promtail
- node-exporter
- cAdvisor
- optional `dnsmasq`

Primary compose file:

- `docker-compose.yaml`

### Important current decisions

1. Docker CLI registry endpoint is `localhost:5000`.
   - Use this for `docker build`, `docker push`, and `docker pull`.
   - Example: `localhost:5000/vion/jenkins-docker:lts-jdk17`

2. Traefik still routes the registry at `registry.localhost`.
   - This is fine for routed HTTP access.
   - Do not assume Docker can push to `registry.localhost` on this host.

3. Jenkins runs a custom image, not the stock upstream image.
   - Current image: `localhost:5000/vion/jenkins-docker:lts-jdk17`
   - Dockerfile: `platform-stack/jenkins/Dockerfile`
   - It includes Docker CLI and Docker Compose plugin.

4. Jenkins state must be preserved.
   - Persistent volume: `jenkins-home`
   - Do not replace or wipe it casually.

5. Grafana is file-provisioned for datasources and dashboards.
   - Provisioning root: `platform-stack/grafana/provisioning/`
   - Arena dashboards are intended to persist through Grafana restarts.

6. Grafana compose defaults are `admin / admin`, but the live password may have been changed already.
   - The running instance stores state in the `grafana-data` volume.
   - Do not assume the compose env values still match the real login.

## Current Access Points

- Traefik dashboard: `http://traefik.vion.test` or `http://localhost:8080`
- Jenkins: `http://jenkins.vion.test` or `http://localhost:8081`
- Grafana: `http://grafana.vion.test` or `http://localhost:3000`
- Prometheus: `http://localhost:9090`
- cAdvisor: `http://cadvisor.vion.test` or `http://localhost:8083`
- Registry routed HTTP: `http://registry.localhost`
- Registry Docker endpoint: `localhost:5000`

## Observability Conventions

### Logs

Promtail discovers Docker containers via the Docker socket and only keeps containers with:

- `logging.enabled=true`
- `logging.stack=vion`
- `logging.service=<service-name>`

Main Loki label:

- `service`

### Metrics

Prometheus scrapes Docker-discovered services with:

- `prometheus.scrape=true`
- `prometheus.port=<port>`
- `prometheus.path=/metrics`

Datasource UIDs in Grafana are intentionally stable:

- Prometheus UID: `prometheus`
- Loki UID: `loki`

### Provisioned dashboards

Current persistent dashboards include:

- `Vion Arena App Health`
- `Vion Arena Containers & Platform Health`

These are defined in:

- `platform-stack/grafana/provisioning/dashboards/vion-arena/`

## Vion Arena State

The example app lives in:

- `vion-arena/`

It is used to validate:

- Traefik routing
- Jenkins CI/CD
- local registry push/pull
- Prometheus metrics
- Loki log ingestion
- Grafana dashboards

### Important current decisions for Vion Arena

1. Its images should use `localhost:5000`, not `registry.localhost`.

2. The Jenkins pipeline was fixed to work with a host Docker daemon.
   - Problem: `-v "$PWD":/workspace` broke because Jenkins talks to the host daemon through `/var/run/docker.sock`.
   - Fix: helper containers use `--volumes-from "$HOSTNAME"` and `-w "$PWD/..."`

3. If `npm ci` appears to fail with “no package-lock.json” in Jenkins, check the mount pattern first.
   - The lockfile exists.
   - The usual cause is the wrong Docker mount strategy in Jenkins.

4. Arena deployment compose file:
   - `vion-arena/deploy/docker-compose.app.yml`

5. Arena pipeline:
   - `vion-arena/Jenkinsfile`

## Safe Working Rules

1. Prefer updating docs and source-of-truth config together.
   - If you change `docker-compose.yaml`, check whether `README.md`, `DOC.md`, or `docs/platform/` also need updates.

2. Do not revert unrelated local changes.
   - This workspace may be intentionally dirty.

3. Be careful with credentials and secrets.
   - `.secrets/` may exist locally and should not be treated as public documentation.

4. Avoid destructive Docker or Git operations unless explicitly requested.
   - Especially avoid wiping persistent volumes like `jenkins-home`, `grafana-data`, `registry-data`, `prometheus-data`, `loki-data`.

5. Treat `DOC.md` as the running platform change log.
   - Treat `README.md` as the short operational overview.

## Useful Commands

Start platform:

```bash
docker compose up -d
```

Optional DNS:

```bash
docker compose --profile dns up -d dnsmasq
```

Check platform containers:

```bash
docker compose ps
```

Check Promtail logs:

```bash
docker compose logs --tail=100 promtail
```

Check Grafana:

```bash
curl -sS http://localhost:3000/api/health
```

Check Prometheus:

```bash
curl -sS http://localhost:9090/-/ready
```

Reset Grafana admin password if needed:

```bash
docker exec -u root vion-project-grafana-1 grafana cli admin reset-admin-password NEW_PASSWORD
```

## Known Sharp Edges

1. `registry.localhost` is not the right endpoint for Docker CLI pushes on this machine.
   - Use `localhost:5000`.

2. Grafana API verification may fail if the live admin password differs from the compose default.

3. cAdvisor currently exposes raw cgroup IDs more readily than stable Compose service labels.
   - Per-container dashboards should be designed carefully and may need to lean on logs plus host-level metrics.

4. Arena frontend does not expose Prometheus metrics.
   - Backend has metrics.
   - Frontend visibility mostly comes from logs, Traefik access, and container/platform-level signals.

## When Updating This File

Update this file when any of these change:

- registry endpoint conventions
- Jenkins runtime image or pipeline execution model
- Grafana provisioning structure
- major platform access URLs
- where the “read first” files live

The goal is that a future Codex session can read this file and become productive without needing a long chat-history replay.
