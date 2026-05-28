# Local Development

This file describes how the current platform is meant to be used on a developer or team machine.

## Start The Platform

Main stack:

```bash
docker compose up -d
```

Optional wildcard DNS helper:

```bash
docker compose --profile dns up -d dnsmasq
```

## Main URLs

- Traefik dashboard: `http://localhost:8080` or `http://traefik.vion.test`
- Jenkins: `http://localhost:8081` or `http://jenkins.vion.test`
- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3000` or `http://grafana.vion.test`
- cAdvisor: `http://localhost:8083` or `http://cadvisor.vion.test`
- Registry HTTP route: `http://registry.vion.test/v2/`
- Registry Docker endpoint: `registry.vion.test:80`

## DNS Strategy

The README describes a one-step team DNS flow using a wildcard `*.vion.test` domain through `dnsmasq`. Docker clients that push or pull images also need `registry.vion.test:80` configured as an insecure registry.

For single-machine testing before wildcard DNS is configured, add a local hosts entry such as:

```text
127.0.0.1 registry.vion.test
127.0.0.1 arena.vion.test
127.0.0.1 api.arena.vion.test
```

Then add the insecure registry entry to `/etc/docker/daemon.json` and restart Docker:

```json
{
  "insecure-registries": ["registry.vion.test:80"]
}
```

That is convenient for:

- shared LAN environments
- demo machines
- small teams using the same stack host

If you do not want local DNS changes, you can still use direct host ports such as `3000`, `8081`, and `9090`.

## Default Credentials

Grafana defaults:

```text
admin / admin
```

If Grafana has already been initialized and the password was changed, the live login comes from the persistent `grafana-data` volume rather than the compose defaults.

Jenkins first-run setup is still enabled, so expect manual bootstrap in a fresh environment.

## What Another Codex Session Should Know

If this platform is copied into a game repo, the local-dev experience depends on a few conventions:

- services should log to stdout or stderr
- services should expose metrics when possible
- routed apps need explicit Traefik labels
- observable apps need explicit logging and metrics labels

Without those labels, the stack will not automatically discover everything.

## Good First Checks

Useful commands after startup:

```bash
docker compose ps
docker compose logs --tail=100 promtail
docker compose logs --tail=100 traefik
curl -sS http://localhost:9090/-/ready
curl -sS http://localhost:3000/api/health
docker exec -u root vion-project-grafana-1 grafana cli admin reset-admin-password NEW_PASSWORD
```

## Recommended Adaptation For A Game Repo

- keep the same platform stack in a separate compose file, such as `docker-compose.platform.yaml`
- keep game services in their own compose file
- connect both files through shared networks and naming conventions
- keep observability opt-in through labels so local experiments stay lightweight

This keeps platform concerns separate from game runtime services while still letting them work together during development and deployment.
