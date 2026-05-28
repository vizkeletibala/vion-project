# Platform Overview

This folder captures the deployment and developer-platform stack currently used in this repo so another Codex session can reuse the same patterns in a different project, including a future game repository.

## Purpose

The platform stack provides:

- Traefik for HTTP routing and local host-based access
- Jenkins for CI/CD orchestration
- Docker Registry for local image storage
- Prometheus for metrics collection
- Grafana for dashboards and log exploration
- Loki and Promtail for centralized container logging
- node-exporter and cAdvisor for host and container metrics
- dnsmasq as an optional wildcard DNS helper for local team environments

## Source Of Truth

The active runtime config lives in:

- `docker-compose.yaml`
- `platform-stack/jenkins/Dockerfile`
- `platform-stack/prometheus/prometheus.yaml`
- `platform-stack/promtail/promtail-config.yaml`
- `platform-stack/loki/loki-config.yaml`
- `platform-stack/grafana/provisioning/datasources/datasources.yaml`
- `platform-stack/grafana/provisioning/dashboards/dashboards.yaml`

This docs folder mirrors those decisions in a format that is easier for another session to understand and transplant.

## Service Map

| Service | Role | Main Access |
| --- | --- | --- |
| Traefik | Reverse proxy and router | `http://traefik.vion.test` and `http://localhost:8080` |
| Registry | Docker image registry | `http://registry.vion.test/v2/` and `registry.vion.test:80` |
| Jenkins | CI/CD server | `http://jenkins.vion.test` and `http://localhost:8081` |
| Prometheus | Metrics backend | `http://localhost:9090` |
| Grafana | Dashboards and logs | `http://grafana.vion.test` and `http://localhost:3000` |
| Loki | Log backend | internal service `http://loki:3100` |
| Promtail | Log collector | internal service `http://promtail:9080` |
| node-exporter | Host metrics | internal service `node-exporter:9100` |
| cAdvisor | Container metrics | `http://cadvisor.vion.test` and `http://localhost:8083` |
| dnsmasq | Optional wildcard DNS | UDP/TCP 53 when profile `dns` is enabled |

## Networks

- `edge`: services that need direct routing or external-style access
- `internal`: services that only need stack-internal communication

Most routed applications join both networks. Internal-only observability services can stay on `internal`.

## Platform Conventions

### Routing labels

Traefik discovers services from Docker labels:

- `traefik.enable=true`
- `traefik.http.routers.<name>.rule=Host(\`example.vion.test\`)`
- `traefik.http.routers.<name>.entrypoints=web`
- `traefik.http.services.<name>.loadbalancer.server.port=<container-port>`

### Metrics labels

Prometheus uses Docker service discovery and only scrapes containers with:

- `prometheus.scrape=true`
- `prometheus.port=<metrics-port>`
- `prometheus.path=/metrics`

### Logging labels

Promtail only ships logs for containers with:

- `logging.enabled=true`
- `logging.stack=vion`
- `logging.service=<service-name>`

The `service` label is the primary Loki query dimension.

## Reuse In Another Repo

For a game project, this stack can stay mostly unchanged. The main work is to:

1. Copy `docs/platform/docker-compose.platform.yaml` into the new repo and adapt service names.
2. Reuse the observability configs if the game services also run in Docker.
3. Add Traefik labels to each game backend, launcher API, asset server, or admin tool.
4. Decide whether Jenkins remains the CI runner or whether another pipeline system should replace it.
5. Replace default credentials and review security before exposing anything outside a local network.

## Important Notes

- Grafana is pre-provisioned to talk to Prometheus and Loki through internal Docker DNS.
- Grafana also provisions persistent dashboards from files under `platform-stack/grafana/provisioning/dashboards/`.
- Jenkins has access to `/var/run/docker.sock`, so it can build and run Docker workloads on the host.
- Jenkins uses a custom image based on `jenkins/jenkins:lts-jdk17` with Docker CLI and Docker Compose plugin preinstalled.
- When Jenkins launches helper containers against the host Docker daemon, container-internal paths should not be bind-mounted with `-v "$PWD":...`; shared-volume approaches such as `--volumes-from` are safer.
- The registry currently has no auth or TLS in this LAN/internal stack.
- Docker clients that push or pull from the LAN registry must configure `registry.vion.test:80` as an insecure registry.
