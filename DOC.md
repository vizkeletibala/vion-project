# Vion Stack Setup Notes

This file summarizes the changes and setup steps applied to the stack during configuration.

## Changes Applied
1. Fixed config mounts to point at the real files under `platform-stack/`:
   - Loki config: `platform-stack/loki/loki-config.yaml` mounted to `/etc/loki/loki-config.yml`
   - Promtail config: `platform-stack/promtail/promtail-config.yaml` mounted to `/etc/promtail/promtail-config.yml`
   - Prometheus config: `platform-stack/prometheus/prometheus.yaml` mounted to `/etc/prometheus/prometheus.yaml`

2. Updated Prometheus to match the new config filename:
   - `--config.file=/etc/prometheus/prometheus.yaml`

3. Removed mount propagation from node-exporter to avoid host mount errors:
   - Changed `/:/host:ro,rslave` to `/:/host:ro`

4. Added hostname-based routing and direct ports for GUI access:
   - Traefik: `traefik.vion.test`
   - Jenkins: `jenkins.vion.test` + `8081:8080`
   - Grafana: `grafana.vion.test` + `3000:3000`
   - cAdvisor: `cadvisor.vion.test` + `8083:8080`

5. Added Grafana provisioning for internal data sources:
   - Prometheus: `http://prometheus:9090`
   - Loki: `http://loki:3100`
   - File: `platform-stack/grafana/provisioning/datasources/datasources.yaml`

6. Fixed Loki retention config by adding:
   - `compactor.delete_request_store: filesystem`

7. Fixed Traefik Docker API mismatch:
   - Set `DOCKER_API_VERSION=1.44` for Traefik

8. Added optional DNS service (dnsmasq) under `--profile dns`:
   - Image: `4km3/dnsmasq:2.90-r3`
   - `NET_ADMIN` capability
   - Wildcard DNS for `*.vion.test` using `VION_LAN_IP`

9. Connected Docker service logs to Loki through Promtail:
   - Promtail uses Docker service discovery through `/var/run/docker.sock`.
   - Promtail only collects containers labeled with `logging.enabled=true`.
   - Logs are labeled with `service`, `stack`, `compose_project`, `compose_service`, and `container`.
   - Promtail stores positions in the `promtail-positions` volume.
   - Grafana can query logs from the provisioned Loki data source.

10. Added logging labels and log-friendly settings for platform services:
   - Traefik emits JSON application logs and JSON access logs.
   - Traefik, Registry, Jenkins, Prometheus, Grafana, Loki, Promtail, node-exporter, cAdvisor, and dnsmasq have `logging.service=<service-name>`.
   - Loki and Promtail expose metrics for Prometheus scraping.
   - Registry exposes debug Prometheus metrics on internal port `5001`.

11. Routed the local registry through the LAN Traefik endpoint:
   - Normal Docker endpoint: `registry.vion.test:80`.
   - Registry route: `registry.vion.test`.
   - Direct `5000:5000` access moved to `docker-compose.registry-direct.yaml` for troubleshooting or migration.

12. Switched Jenkins to a custom image stored in the local registry:
   - Runtime image: `registry.vion.test:80/vion/jenkins-docker:lts-jdk17`
   - Build file: `platform-stack/jenkins/Dockerfile`
   - Compose builds this image locally before starting Jenkins, so initial platform startup does not require a running registry.
   - Based on `jenkins/jenkins:lts-jdk17`
   - Includes Docker CLI and Docker Compose plugin
   - Existing jobs and state remain in the `jenkins-home` volume

13. Added stable Grafana datasource UIDs for provisioned dashboards:
   - Prometheus UID: `prometheus`
   - Loki UID: `loki`
   - File: `platform-stack/grafana/provisioning/datasources/datasources.yaml`

14. Added persistent Grafana dashboard provisioning for the Vion Arena example app:
   - Provider file: `platform-stack/grafana/provisioning/dashboards/dashboards.yaml`
   - Dashboard: `Vion Arena App Health`
   - Dashboard: `Vion Arena Containers & Platform Health`
   - Files live under `platform-stack/grafana/provisioning/dashboards/vion-arena/`
   - Dashboards are loaded from disk and persist through Grafana container restarts

## Log Collection Overview
Logs flow through the stack like this:

```text
Docker containers
  -> Docker JSON log files
  -> Promtail
  -> Loki
  -> Grafana Explore
```

Promtail reads Docker container logs from:

```text
/var/lib/docker/containers/*/*-json.log
```

The main Loki label to use in queries is `service`.

Useful LogQL queries:

```logql
{service="traefik"}
{service="jenkins"}
{service="registry"}
{service="prometheus"}
{service="loki"}
{service="promtail"}
```

Count recent logs:

```logql
count_over_time({service="traefik"}[10m])
count_over_time({service="jenkins"}[10m])
count_over_time({service="registry"}[10m])
count_over_time({service="prometheus"}[10m])
```

## Checking Log Health
Use these checks when validating whether logs are arriving in Loki.

### 1. Check containers are running
```bash
docker compose ps loki promtail grafana prometheus
```

Expected:
- `loki` is `Up`
- `promtail` is `Up`
- `grafana` is `Up`
- `prometheus` is `Up`

### 2. Check Promtail discovered Docker targets
```bash
docker compose logs --tail=100 promtail
```

Good signs:
```text
Starting Promtail
added Docker target
```

If Promtail does not show Docker targets, check:
- The container has `logging.enabled=true`.
- Promtail has `/var/run/docker.sock` mounted.
- Promtail has `/var/lib/docker/containers` mounted read-only.
- The container is writing logs to stdout/stderr.

### 3. Check Promtail readiness
```bash
docker compose exec -T prometheus wget -qO- http://promtail:9080/ready
```

Expected:
```text
Ready
```

### 4. Check Loki readiness
```bash
docker compose exec -T prometheus wget -qO- http://loki:3100/ready
```

Expected:
```text
ready
```

Loki may return `503 Service Unavailable` for a short time immediately after startup. Retry after a few seconds.

### 5. Check which services Loki has received
```bash
docker compose exec -T prometheus wget -qO- \
  'http://loki:3100/loki/api/v1/label/service/values'
```

Expected services include:

```json
["cadvisor","grafana","jenkins","loki","node-exporter","prometheus","promtail","registry","traefik"]
```

If the Vion Arena example app is deployed, you should also expect:

```json
["vion-arena-backend","vion-arena-frontend"]
```

If a service is missing:
- Confirm the service has `logging.enabled=true`.
- Confirm the service has `logging.service=<name>`.
- Restart Promtail or wait for Docker discovery refresh.
- Generate a fresh log line for that service.

### 6. Generate fresh logs
Run a few requests that should produce logs:

```bash
curl -H 'Host: registry.vion.test' http://127.0.0.1/v2/
curl http://127.0.0.1:8081/login
curl http://127.0.0.1:8080/metrics
```

Then query Loki for recent logs:

```bash
docker compose exec -T prometheus wget -qO- \
  'http://loki:3100/loki/api/v1/query?query=count_over_time%28%7Bservice%3D%22traefik%22%7D%5B1m%5D%29'
```

The response should contain:

```json
"status":"success"
```

and a result value greater than `0`.

### 7. Query raw logs through Loki
Traefik:

```bash
docker compose exec -T prometheus wget -qO- \
  'http://loki:3100/loki/api/v1/query?query=%7Bservice%3D%22traefik%22%7D'
```

Jenkins:

```bash
docker compose exec -T prometheus wget -qO- \
  'http://loki:3100/loki/api/v1/query?query=%7Bservice%3D%22jenkins%22%7D'
```

Registry:

```bash
docker compose exec -T prometheus wget -qO- \
  'http://loki:3100/loki/api/v1/query?query=%7Bservice%3D%22registry%22%7D'
```

Prometheus:

```bash
docker compose exec -T prometheus wget -qO- \
  'http://loki:3100/loki/api/v1/query?query=%7Bservice%3D%22prometheus%22%7D'
```

### 8. Check in Grafana
Open Grafana:

```text
http://localhost:3000
```

Default local credentials on first startup:

```text
admin / admin
```

Note:
- Grafana stores state in the `grafana-data` volume.
- If the admin password was changed after first startup, the running instance will use the stored value instead of the compose default.

Go to Explore, select the Loki data source, then run:

```logql
{service="traefik"}
```

Try additional services:

```logql
{service="jenkins"}
{service="registry"}
{service="prometheus"}
```

### 9. Check logging system metrics in Prometheus
Prometheus should scrape Loki and Promtail from the `docker-services` job.

Check active targets:

```bash
curl -sS 'http://127.0.0.1:9090/api/v1/targets?state=active'
```

Expected healthy targets include:

```text
docker-services loki up
docker-services promtail up
docker-services registry up
docker-services traefik up
```

Useful Prometheus UI:

```text
http://localhost:9090/targets
```

Useful Prometheus queries:

```promql
up{job="docker-services",service="loki"}
up{job="docker-services",service="promtail"}
up{job="docker-services",service="traefik"}
up{job="docker-services",service="registry"}
```

Expected value is `1`.

### 10. Common failure checks
- No logs for a service: check `logging.enabled=true` and `logging.service=<name>`.
- Loki label values are empty: check Promtail logs for Docker discovery errors.
- Promtail is not ready: check `docker compose logs promtail`.
- Loki is not ready: check `docker compose logs loki`.
- Grafana has no Loki data source: check `platform-stack/grafana/provisioning/datasources/datasources.yaml`.
- Prometheus target is down: check `http://localhost:9090/targets` and the `prometheus.port` label for that service.
- A Docker push to `registry.vion.test:80` fails: verify DNS resolves to the platform host and Docker is configured with `"insecure-registries": ["registry.vion.test:80"]`.
- Grafana dashboards are missing after restart: check `platform-stack/grafana/provisioning/dashboards/dashboards.yaml` and confirm Grafana logs show `finished to provision dashboards`.

## DNS Setup (Team Usage)
1. Export server IP and start dnsmasq:
   ```bash
   export VION_LAN_IP=<server-ip>
   docker compose --profile dns up -d dnsmasq
   ```
2. Point client DNS (or router DHCP DNS) to `<server-ip>`.
3. Use hostnames such as:
   - `grafana.vion.test`
   - `jenkins.vion.test`
   - `traefik.vion.test`
   - `cadvisor.vion.test`

## Port 53 Conflict (systemd-resolved)
If port 53 is in use (e.g., `systemd-resolved` on `127.0.0.53`), dnsmasq may fail to bind.
Options:
1. Bind dnsmasq to the LAN IP only (recommended):
   - Change ports to `<LAN_IP>:53:53/udp` and `<LAN_IP>:53:53/tcp`
2. Disable or reconfigure systemd-resolved (Linux-only).
3. Move DNS to a dedicated host (router/Pi-hole) instead of the container.

## Jenkins URL Recommendation
Set Jenkins URL to the hostname used by users (example):
```
http://jenkins.vion.test/
```
