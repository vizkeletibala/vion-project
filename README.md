# vion-project
My first product that aims to deliver a package which helps small teams dockerize their native run applications. This includes docker-compose files and templates, monitoring and reverse proxying solutions.

## Team DNS (one-step)
1. Set `VION_LAN_IP` to the server IP and run `docker compose --profile dns up -d dnsmasq`, then point client DNS (or the router DHCP DNS) to the same server IP. After that, any `*.vion.test` hostname resolves to the stack.

Notes:
- If you prefer a UI and ad-blocking, swap `dnsmasq` for Pi-hole later.
- Grafana is pre-provisioned to reach Prometheus and Loki using internal service names (`http://prometheus:9090`, `http://loki:3100`).
