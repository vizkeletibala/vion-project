# Vitrial CI/CD Migration Into The Version1 Vion Platform

This document is the migration handoff for the Vitrial lane that was first drafted in the arena-side scaffold at commit `9804864`. The checked-out `vion-project` `version1` stack is now the source of truth for shared CI/CD, registry, routing, and observability conventions.

## Decision

Keep the platform stack in this repository and keep application-specific Vitrial files in the `vion-arena` application repository.

Do not copy `docker-compose.yaml`, `platform-stack/`, Jenkins image build logic, Grafana provisioning, Prometheus, Loki, Promtail, Traefik, or the local registry into `vion-arena`. The app repository should consume the already-running platform through documented endpoints, external Docker networks, labels, and image names.

## Boundary

| Concern | Source of truth | Notes |
| --- | --- | --- |
| Platform services | `vion-project/docker-compose.yaml` | Traefik, registry, Jenkins, Prometheus, Grafana, Loki, Promtail, node-exporter, cAdvisor, optional dnsmasq. |
| Jenkins runtime image | `vion-project/platform-stack/jenkins/Dockerfile` | Includes Docker CLI and Docker Compose v2 plugin. Jenkins state remains in `jenkins-home`. |
| Registry endpoint | `vion-project/docs/platform/registry.md` | Docker push/pull endpoint is `registry.vion.test:80`, not `localhost:5000`. |
| Platform networks | `vion-project/docker-compose.yaml` | With the default compose project name, app deploys consume `vion-project_edge` and `vion-project_internal` as external networks. |
| Traefik routing posture | Platform owns Traefik; app owns its own route labels | Vitrial web/API services may publish host rules. Future game server/admin routes remain disabled until auth/TLS/firewall decisions are made. |
| Observability conventions | `vion-project/docs/platform/observability.md` and `platform-stack/` provisioning | App services opt in using Docker labels for Promtail/Loki and Prometheus. Dashboards that should persist across Grafana restarts belong in this repo. |
| Vitrial pipeline and app deploy files | `vion-arena` | `Jenkinsfile`, app Dockerfiles, app compose, Unity validation script, and future dedicated-server runtime image skeleton stay app-local. |

## Drift Reconciled From The Arena Scaffold

The arena-side scaffold from commit `9804864` used several assumptions that must be updated before it is treated as the real Vitrial lane:

1. Registry endpoint
   - Scaffold value: `localhost:5000`.
   - Version1 platform value: `registry.vion.test:80`.
   - Required app-local change: set Jenkins `REGISTRY = "registry.vion.test:80"` and update default images in app compose files to the same endpoint.

2. Docker networks
   - Scaffold app compose defaults used `edge` / `internal` in places.
   - Version1 platform networks are the external networks created by the platform compose project: `vion-project_edge` and `vion-project_internal`.
   - Required app-local change: default app compose external network names to `vion-project_edge` and `vion-project_internal`, while still allowing `EDGE_NETWORK` and `INTERNAL_NETWORK` overrides for non-default compose project names.

3. Jenkins image and Compose plugin
   - Version1 Jenkins already uses the custom image `registry.vion.test:80/vion/jenkins-docker:lts-jdk17`, built from `platform-stack/jenkins/Dockerfile`.
   - Do not add a second Jenkins image or install Docker Compose per job. Pipelines can assume `docker compose` exists inside the platform Jenkins image after this stack is running.
   - Keep the `--volumes-from "$HOSTNAME"` helper-container pattern because Jenkins talks to the host Docker daemon through `/var/run/docker.sock`.

4. Traefik posture
   - Web/API app routes are app-local labels that attach to the platform `edge` network.
   - Future Vitrial dedicated-server service remains internal by default with `traefik.enable=false`; public UDP/game/admin exposure is a separate security decision and should not be smuggled into this migration.

5. Observability labels
   - Promtail keeps only containers with `logging.enabled=true` and `logging.stack=vion`.
   - Use stable `logging.service` values: `vion-arena-frontend`, `vion-arena-backend`, and future `vitrial-server`.
   - Prometheus labels should point at real metrics endpoints only. The Vitrial server can keep `prometheus.scrape=false` until it exposes `/metrics`.

## Required Vion Arena Follow-up Patch

Apply the following app-local migration in `/home/vion/src/git/vion-arena` or the canonical Vitrial application repository.

### Jenkinsfile

Use the version1 registry endpoint and preserve platform network names:

```groovy
environment {
  IMAGE_TAG = "${BUILD_NUMBER}"
  REGISTRY = "registry.vion.test:80"
  FRONTEND_IMAGE = "${REGISTRY}/vion-arena-frontend:${IMAGE_TAG}"
  BACKEND_IMAGE = "${REGISTRY}/vion-arena-backend:${IMAGE_TAG}"
  VITRIAL_SERVER_IMAGE = "${REGISTRY}/vitrial-server:${IMAGE_TAG}"
  EDGE_NETWORK = "vion-project_edge"
  INTERNAL_NETWORK = "vion-project_internal"
}
```

Keep the existing `docker compose -f deploy/docker-compose.app.yml up -d` and gated future server stages, but run them only from the platform Jenkins instance that uses the version1 custom Jenkins image.

### `deploy/docker-compose.app.yml`

Set version1-compatible defaults:

```yaml
services:
  backend:
    image: ${BACKEND_IMAGE:-registry.vion.test:80/vion-arena-backend:dev}
    labels:
      - traefik.docker.network=${EDGE_NETWORK:-vion-project_edge}
      - logging.enabled=true
      - logging.stack=vion
      - logging.service=vion-arena-backend
      - prometheus.scrape=true
      - prometheus.port=8000
      - prometheus.path=/metrics
  frontend:
    image: ${FRONTEND_IMAGE:-registry.vion.test:80/vion-arena-frontend:dev}
    labels:
      - traefik.docker.network=${EDGE_NETWORK:-vion-project_edge}
      - logging.enabled=true
      - logging.stack=vion
      - logging.service=vion-arena-frontend
networks:
  edge:
    external: true
    name: ${EDGE_NETWORK:-vion-project_edge}
  internal:
    external: true
    name: ${INTERNAL_NETWORK:-vion-project_internal}
```

### `deploy/docker-compose.vitrial-server.yml`

Set the future server image default to the version1 registry endpoint and keep it internal:

```yaml
services:
  vitrial-server:
    image: ${VITRIAL_SERVER_IMAGE:-registry.vion.test:80/vitrial-server:dev}
    profiles:
      - vitrial-server
    labels:
      - traefik.enable=false
      - logging.enabled=true
      - logging.stack=vion
      - logging.service=vitrial-server
      - prometheus.scrape=${VITRIAL_PROMETHEUS_SCRAPE:-false}
networks:
  internal:
    external: true
    name: ${INTERNAL_NETWORK:-vion-project_internal}
```

### `docs/vitrial-automation.md`

Update the app documentation so every `localhost:5000` example becomes `registry.vion.test:80`, and note that the platform stack lives in `vion-project` rather than being duplicated in `vion-arena`.

## Verification Commands

Run these from `/home/vion/src/git/vion-project` to verify the platform side of the migration:

```bash
git diff --check
docker compose config >/tmp/vion-project-compose.yaml
docker compose config --services | grep -E '^(traefik|registry|jenkins|prometheus|grafana|loki|promtail|node-exporter|cadvisor)$'
```

If Docker is available and the platform is running, also verify the live endpoints without recreating volumes:

```bash
docker compose ps
curl -sS -H 'Host: registry.vion.test' http://127.0.0.1/v2/
curl -sS http://127.0.0.1:9090/-/ready
curl -sS http://127.0.0.1:3000/api/health
```

Run these from `/home/vion/src/git/vion-arena` after the app-local follow-up patch:

```bash
git diff --check
sh scripts/validate-vitrial-headless.sh
python3 -m json.tool Packages/manifest.json >/dev/null
python3 -m json.tool Assets/Game/Vitrial.Game.asmdef >/dev/null
docker compose -f deploy/docker-compose.app.yml config >/tmp/vion-arena-app-compose.yaml
docker compose -f deploy/docker-compose.vitrial-server.yml --profile vitrial-server config >/tmp/vitrial-server-compose.yaml
grep -R "localhost:5000" Jenkinsfile deploy docs && exit 1 || true
```

Expected invariants after migration:

- `localhost:5000` is absent from Vitrial CI/CD files.
- Image names use `registry.vion.test:80`.
- App deploy compose consumes `vion-project_edge` and `vion-project_internal` as external networks by default.
- Vitrial web/API services are routed through platform Traefik labels.
- Future `vitrial-server` is not Traefik-routed and does not publish public ports by default.
- Platform volumes such as `jenkins-home`, `grafana-data`, `registry-data`, `prometheus-data`, and `loki-data` are preserved.
