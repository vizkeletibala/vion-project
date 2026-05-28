# Routing

Traefik is the entrypoint for HTTP routing in this stack.

## Current Setup

Traefik runs with the Docker provider enabled and `exposedbydefault=false`, which means only explicitly labeled containers are routed.

Active entrypoints and ports:

- port `80`: application traffic
- port `8080`: Traefik dashboard in insecure dev mode

Current routed hostnames:

- `traefik.vion.test`
- `jenkins.vion.test`
- `grafana.vion.test`
- `cadvisor.vion.test`
- `registry.vion.test`

The registry is routed through Traefik and should use the same LAN endpoint for browser-style HTTP access and Docker CLI operations: `registry.vion.test:80`.

## Label Pattern

Each routed service follows this shape:

```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.app.rule=Host(`app.vion.test`)
  - traefik.http.routers.app.entrypoints=web
  - traefik.http.services.app.loadbalancer.server.port=8080
```

The last line must point to the port exposed inside the container, not the host port.

## Network Expectations

Routed services should join:

- `edge` so Traefik can reach them
- `internal` if they also need internal platform access

Purely internal components such as Loki can skip `edge`.

## Reusing This In A Game Repo

For a game project, Traefik can front:

- the main backend API
- matchmaking or session services
- admin tools
- an internal build dashboard
- asset patch or download services

Recommended hostname pattern:

- `api.game.test`
- `admin.game.test`
- `grafana.game.test`
- `jenkins.game.test`
- `registry.game.test`

## DNS Options

There are two local-development approaches in this repo:

1. Use `dnsmasq` with the optional `dns` profile for wildcard `*.vion.test` resolution.
2. Skip wildcard DNS and add manual host entries for the few services you need.

## Registry Routing

The registry deliberately uses the same wildcard DNS pattern as the other services. Docker clients need the exact HTTP endpoint in their insecure registry configuration, for example `registry.vion.test:80`.

## Minimal Example For A New App

```yaml
services:
  game-api:
    image: registry.game.test:80/game-api:dev
    labels:
      - traefik.enable=true
      - traefik.http.routers.game-api.rule=Host(`api.game.test`)
      - traefik.http.routers.game-api.entrypoints=web
      - traefik.http.services.game-api.loadbalancer.server.port=8080
      - logging.enabled=true
      - logging.stack=game
      - logging.service=game-api
      - prometheus.scrape=true
      - prometheus.port=9091
      - prometheus.path=/metrics
    networks:
      - edge
      - internal
```

This gives the service routing, logs, and metrics using the same conventions as the Vion platform.
