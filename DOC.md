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
