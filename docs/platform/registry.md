# Registry

The stack includes a local Docker Registry service for storing built images close to the deployment environment.

## Current Service Definition

- image: `registry:2`
- routed hostname: `registry.vion.test`
- Docker endpoint: `registry.vion.test:80`
- internal app port: `5000`
- debug and metrics port: `5001`
- persistent volume: `registry-data`

Environment settings:

```text
REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/var/lib/registry
REGISTRY_HTTP_DEBUG_ADDR=0.0.0.0:5001
REGISTRY_HTTP_DEBUG_PROMETHEUS_ENABLED=true
REGISTRY_HTTP_DEBUG_PROMETHEUS_PATH=/metrics
```

## What This Enables

- local image pushes from developers or Jenkins
- deployment without depending on a third-party registry
- Prometheus scraping for registry metrics

## Current Access Pattern

The service is exposed through Traefik with:

```yaml
- traefik.http.routers.registry.rule=Host(`registry.vion.test`)
- traefik.http.services.registry.loadbalancer.server.port=5000
```

The base platform does not publish the registry directly on a host-only port. Docker clients should push and pull through the LAN route:

```text
registry.vion.test:80
```

If direct port access is needed for troubleshooting or migration, start the stack with `docker-compose.registry-direct.yaml`.

Every Docker daemon that pushes or pulls images from this HTTP registry must include the exact endpoint in its insecure registry list:

```json
{
  "insecure-registries": ["registry.vion.test:80"]
}
```

The registry hostname must also resolve on the Docker host before Compose can pull application images:

```bash
getent hosts registry.vion.test
sh scripts/validate-config.sh --strict
```

The registry is also labeled for:

- logging through Promtail
- metrics scraping through Prometheus on port `5001`

## Example Use

Build and tag:

```bash
docker build -t registry.vion.test:80/my-team/game-api:dev .
```

Push:

```bash
docker push registry.vion.test:80/my-team/game-api:dev
```

Use in Compose:

```yaml
image: registry.vion.test:80/my-team/game-api:dev
```

## Reusing This In A Game Repo

This is useful when:

- the game backend is deployed from Docker images
- CI should publish artifacts into a local environment
- staging or LAN environments should keep running without cloud dependencies

Suggested hostname pattern for a new repo:

- `registry.game.test`

Use a single LAN DNS name everywhere: Jenkins, developer machines, deployment compose files, and app docs.

## Important Security Note

This registry setup is local-development friendly, not production ready.

Current limitations:

- no TLS
- no authentication
- no image signing or policy enforcement

If customers want image names without `:80` and without Docker insecure-registry setup, add HTTPS on `443` with certificates trusted by every Docker client. If the stack will deploy outside a trusted LAN, add auth and TLS or use a managed registry.
