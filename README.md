# vion-project
My first product that aims to deliver a package which helps small teams dockerize their native run applications. This includes docker-compose files and templates, monitoring and reverse proxying solutions.

## What This Stack Includes

- Traefik for local HTTP routing
- Local Docker Registry
- Jenkins with Docker CLI and Docker Compose plugin preinstalled
- Prometheus for metrics
- Grafana for dashboards and logs
- Loki and Promtail for centralized container logging
- node-exporter and cAdvisor for host and container visibility
- Optional `dnsmasq` for wildcard `*.vion.test` resolution

## Main Access Points

- Traefik dashboard: `http://traefik.vion.test` or `http://localhost:8080`
- Jenkins: `http://jenkins.vion.test` or `http://localhost:8081`
- Grafana: `http://grafana.vion.test` or `http://localhost:3000`
- Prometheus: `http://localhost:9090`
- cAdvisor: `http://cadvisor.vion.test` or `http://localhost:8083`
- Registry HTTP route: `http://registry.localhost`
- Registry Docker endpoint: `localhost:5000`

## Start The Platform

```bash
docker compose up -d
```

Optional team DNS:

```bash
docker compose --profile dns up -d dnsmasq
```

## Team DNS (one-step)
1. Set `VION_LAN_IP` to the server IP and run `docker compose --profile dns up -d dnsmasq`, then point client DNS (or the router DHCP DNS) to the same server IP. After that, any `*.vion.test` hostname resolves to the stack.

Notes:
- If you prefer a UI and ad-blocking, swap `dnsmasq` for Pi-hole later.
- Grafana is pre-provisioned to reach Prometheus and Loki using internal service names (`http://prometheus:9090`, `http://loki:3100`).
- Docker pushes and pulls on this host should use `localhost:5000` rather than `registry.localhost`.

## Jenkins Image

Jenkins runs from the custom image:

```text
localhost:5000/vion/jenkins-docker:lts-jdk17
```

That image is built from `platform-stack/jenkins/Dockerfile` and already includes:

- Docker CLI
- Docker Compose plugin

Jenkins state is still persisted in the `jenkins-home` volume, so jobs and configuration survive container recreation.

## Grafana Provisioning

Grafana is provisioned from files under:

- `platform-stack/grafana/provisioning/datasources/`
- `platform-stack/grafana/provisioning/dashboards/`

Current provisioned dashboards include:

- `Vion Arena App Health`
- `Vion Arena Containers & Platform Health`

These dashboards persist through Grafana restarts because they are loaded from the mounted provisioning directory.

## Notes

- `DOC.md` contains the longer stack setup log and troubleshooting notes.
- `docs/platform/` contains reusable platform docs intended for another Codex session or another repo.
