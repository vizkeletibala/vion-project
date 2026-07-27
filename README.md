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
- Registry HTTP route: `http://registry.vion.test/v2/`
- Registry Docker endpoint: `registry.vion.test:80`

## Start The Platform

```bash
cp .env.example .env
sh scripts/validate-config.sh
docker compose up -d
```

Optional team DNS:

```bash
docker compose --profile dns up -d dnsmasq
```

## Team DNS (one-step)
1. Set `LAN_IP` in `.env` to the server IP and run `docker compose --profile dns up -d dnsmasq`, then point client DNS (or the router DHCP DNS) to the same server IP. After that, any `*.vion.test` hostname resolves to the stack.

Notes:
- If you prefer a UI and ad-blocking, swap `dnsmasq` for Pi-hole later.
- Grafana is pre-provisioned to reach Prometheus and Loki using internal service names (`http://prometheus:9090`, `http://loki:3100`).
- Docker pushes and pulls should use `registry.vion.test:80` on every LAN machine.
- Before deploying apps, run `sh scripts/validate-config.sh --strict` on any machine that will push or pull images. It checks that `registry.vion.test` resolves and that Docker has the insecure registry entry.
- Configure each Docker daemon that pushes or pulls images with this insecure registry entry:

```json
{
  "insecure-registries": ["registry.vion.test:80"]
}
```

- If you want image names without `:80` and without insecure-registry setup, add HTTPS on `443` with certificates trusted by every Docker client.

## Jenkins Image

Jenkins runs from the custom image:

```text
registry.vion.test:80/vion/jenkins-docker:lts-jdk17
```

That image is built locally from `platform-stack/jenkins/Dockerfile` during `docker compose up`, so the first platform startup does not need the platform registry to already be running. The image tag still points at the LAN registry endpoint so it can be pushed or reused after the registry is up.

It already includes:

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
- `docs/platform/shipping-checklist.md` contains the LAN product readiness checklist.
- `docs/platform/vitrial-cicd-migration.md` records the Vitrial migration boundary: this repo owns the shared platform stack, while the `vion-arena` app repo keeps its Jenkinsfile, app compose files, Unity validation, and future dedicated-server packaging files.
