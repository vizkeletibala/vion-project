# Observability

This stack uses Prometheus, Grafana, Loki, Promtail, node-exporter, and cAdvisor.

## Components

### Prometheus

- image: `prom/prometheus:v2.54.1`
- config file: `platform-stack/prometheus/prometheus.yaml`
- host port: `9090`
- retention: `7d`

Prometheus scrapes:

- itself
- `node-exporter`
- `cadvisor`
- Docker-discovered services with `prometheus.scrape=true`

The Docker discovery flow builds the scrape target from:

- container network IP
- `prometheus.port`
- optional `prometheus.path`

### Grafana

- image: `grafana/grafana:11.1.4`
- host port: `3000`
- route: `grafana.vion.test`
- default credentials: `admin / admin`

Provisioned datasources:

- Prometheus: `http://prometheus:9090`
- Loki: `http://loki:3100`

Stable datasource UIDs:

- Prometheus: `prometheus`
- Loki: `loki`

Datasource file:

- `platform-stack/grafana/provisioning/datasources/datasources.yaml`

Dashboard provisioning files:

- `platform-stack/grafana/provisioning/dashboards/dashboards.yaml`
- `platform-stack/grafana/provisioning/dashboards/vion-arena/app-health.json`
- `platform-stack/grafana/provisioning/dashboards/vion-arena/containers-health.json`

Provisioned dashboards currently included:

- `Vion Arena App Health`
- `Vion Arena Containers & Platform Health`

These dashboards are loaded from the mounted provisioning directory, so they persist through Grafana container restarts.

### Loki

- image: `grafana/loki:3.1.1`
- internal URL: `http://loki:3100`
- config file: `platform-stack/loki/loki-config.yaml`
- retention: `7d`

The Loki config uses filesystem-backed local storage and a single-node in-memory ring.

### Promtail

- image: `grafana/promtail:3.1.1`
- internal URL: `http://promtail:9080`
- config file: `platform-stack/promtail/promtail-config.yaml`

Promtail:

- discovers containers through Docker socket discovery
- reads Docker JSON log files under `/var/lib/docker/containers`
- only keeps containers labeled with `logging.enabled=true`
- pushes logs to `http://loki:3100/loki/api/v1/push`
- persists positions in `/var/lib/promtail/positions.yaml`

### node-exporter

- image: `prom/node-exporter:v1.8.2`
- internal target: `node-exporter:9100`
- mounted against host rootfs for machine metrics

### cAdvisor

- image: `gcr.io/cadvisor/cadvisor:v0.49.2`
- host port: `8083`
- route: `cadvisor.vion.test`
- internal target: `cadvisor:8080`

## Label Contracts

### For metrics

Add these labels to any service that exposes Prometheus metrics:

```yaml
labels:
  - prometheus.scrape=true
  - prometheus.port=8080
  - prometheus.path=/metrics
```

### For logs

Add these labels to any service whose logs should appear in Loki:

```yaml
labels:
  - logging.enabled=true
  - logging.stack=vion
  - logging.service=my-service
```

Promtail turns these into Loki labels such as:

- `stack`
- `service`
- `compose_project`
- `compose_service`
- `container`
- `image`

## Useful Queries

### Prometheus targets

Use Prometheus UI or API to verify active targets for:

- `docker-services`
- `prometheus`
- `node`
- `cadvisor`

### LogQL

Common log queries:

```logql
{service="traefik"}
{service="jenkins"}
{service="registry"}
{service="prometheus"}
```

Count logs over time:

```logql
count_over_time({service="traefik"}[10m])
count_over_time({service="jenkins"}[10m])
```

## Reusing This In A Game Repo

This setup is a strong fit if the game services run in Docker and expose metrics or write to stdout.

Recommended instrumentation targets:

- game backend API metrics
- matchmaking queue depth
- worker and job durations
- patch or content-delivery service metrics
- build pipeline logs
- dedicated server process logs

If the game eventually uses Kubernetes or a hosted logging platform, these docs still help as a local and staging baseline.

## Security And Ops Notes

- Grafana defaults are fine for local use only.
- The compose defaults are `admin / admin`, but the live password may differ once Grafana has written state into the persistent `grafana-data` volume.
- Loki storage is local filesystem, not object storage.
- Prometheus retention is short and intended for a small stack.
- Promtail depends on Docker log files, so non-Docker processes will need a different scrape method.
