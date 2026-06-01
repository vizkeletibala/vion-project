# vion-project onboarding notes

Last inspected: 2026-05-29

## Repository status

`vion-project` is currently a seed repository. The tracked source tree contains only:

- `README.md` — short product statement for a package that helps small teams dockerize native-run applications with Compose templates, monitoring, and reverse proxying.
- `ONBOARDING.md` — this onboarding map.

There are no committed application modules, package manifests, Dockerfiles, Compose files, tests, scripts, or CI definitions in this repository yet. The nearest related reference material on this machine is the sibling repository `/home/vion/src/git/vion-arena`, especially its `docs/platform/` directory, which documents a local Docker platform that appears intended to be reused here.

## Product intent

The project aims to become a packaged developer-platform starter kit for small teams. The likely deliverable is not a single app service, but a reusable set of templates and automation around:

- Dockerizing existing applications that currently run natively.
- Providing Docker Compose files and service templates.
- Adding reverse proxy routing.
- Adding monitoring, logging, and dashboard defaults.
- Giving teams safe local or LAN deployment patterns.

## Current architecture map

Because the implementation is not committed yet, this is the proposed architecture implied by the README and the sibling platform docs.

### 1. Template/package layer

Purpose: reusable project files that users copy or generate into their own app repositories.

Likely future contents:

- `templates/compose/` for baseline `docker-compose` service templates.
- `templates/proxy/` for Traefik labels and routing snippets.
- `templates/observability/` for Prometheus, Grafana, Loki, and Promtail defaults.
- `templates/ci/` for Jenkins or other CI pipeline examples.
- `examples/` for sample native apps converted into Dockerized services.

### 2. Platform services layer

Reference services from `/home/vion/src/git/vion-arena/docs/platform/docker-compose.platform.yaml`:

| Service | Role | Typical access |
| --- | --- | --- |
| Traefik | HTTP reverse proxy and Docker-label router | `http://localhost:8080`, `http://traefik.vion.test` |
| Jenkins | CI/CD runner with Docker socket access | `http://localhost:8081`, `http://jenkins.vion.test` |
| Docker Registry | Local image storage | `http://registry.localhost` |
| Prometheus | Metrics collection | `http://localhost:9090` |
| Grafana | Dashboards and log exploration | `http://localhost:3000`, `http://grafana.vion.test` |
| Loki | Log storage backend | internal `http://loki:3100` |
| Promtail | Docker log collector | internal `http://promtail:9080` |
| node-exporter | Host metrics | internal `node-exporter:9100` |
| cAdvisor | Container metrics | `http://localhost:8083`, `http://cadvisor.vion.test` |
| dnsmasq | Optional wildcard DNS for `*.vion.test` | Docker Compose profile `dns` |

### 3. Application integration layer

Applications using this package should expose enough metadata for the platform to discover them:

- Traefik labels for routing.
- Prometheus labels for metrics scraping.
- Logging labels for Promtail/Loki collection.
- Docker networks named or equivalent to `edge` and `internal`.

Reference label contracts from `vion-arena`:

```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.<name>.rule=Host(`<host>`)
  - traefik.http.routers.<name>.entrypoints=web
  - traefik.http.services.<name>.loadbalancer.server.port=<container-port>
  - prometheus.scrape=true
  - prometheus.port=<metrics-port>
  - prometheus.path=/metrics
  - logging.enabled=true
  - logging.stack=vion
  - logging.service=<service-name>
```

## Development commands available today

The repository itself has no runnable code yet. The only safe current commands are repository inspection commands:

```bash
git status
git log --oneline --decorate --max-count=10
```

No `npm`, `python`, `docker compose`, `pytest`, or build commands are defined in `vion-project` today because there are no manifests or Compose files.

## Reference commands from the sibling platform docs

These commands are not runnable from `vion-project` yet, but they represent the likely target workflow once platform files are added:

```bash
# start a platform compose stack
docker compose up -d

# optional wildcard DNS helper
docker compose --profile dns up -d dnsmasq

# inspect stack health
docker compose ps
docker compose logs --tail=100 traefik
docker compose logs --tail=100 promtail
curl -sS http://localhost:9090/-/ready
curl -sS http://localhost:3000/api/health

# build and push an app image to the local registry
docker build -t registry.localhost/my-team/my-app:dev .
docker push registry.localhost/my-team/my-app:dev
```

## Safest first automation opportunities

Start with low-risk, repository-local automation before introducing privileged Docker orchestration.

1. Add a `docs/architecture.md` or expand this file into a stable architecture contract.
   - Risk: very low.
   - Why first: establishes names, folder layout, supported use cases, and non-goals before templates sprawl.

2. Add a skeleton template tree without executable side effects.
   - Suggested paths: `templates/compose/`, `templates/traefik/`, `templates/observability/`, `examples/`.
   - Risk: low.
   - Verification: static file review and YAML validation.

3. Add static validation for YAML, Markdown, and shell snippets.
   - Suggested checks: `yamllint`, `markdownlint`, `shellcheck` once files exist.
   - Risk: low.
   - Why: catches template mistakes before they reach users.

4. Add a dry-run generator or copier.
   - Example: a script that renders templates into a temporary output directory, never overwriting user files unless explicitly requested.
   - Risk: medium-low.
   - Safety guard: default to `--dry-run` and print a file plan.

5. Add a Compose smoke test only after templates exist.
   - Example: `docker compose config` for syntax validation before `docker compose up`.
   - Risk: medium.
   - Safety guard: do not mount `/var/run/docker.sock` or start privileged services in first-pass CI.

6. Defer Jenkins automation and Docker socket mounting.
   - Risk: high because Jenkins with Docker socket access is effectively host-privileged.
   - Recommended prerequisite: document threat model, credentials, network exposure, and local-only assumptions.

## Security and operations notes

The sibling platform docs are local-development friendly, not production hardened. Before copying them directly into this repository, decide how to handle:

- Registry authentication and TLS.
- Grafana default credentials.
- Traefik dashboard exposure.
- Jenkins first-run setup, authentication, and Docker socket access.
- Whether `*.vion.test` and `registry.localhost` should be normalized to one hostname scheme.
- Data retention and backup expectations for Prometheus, Loki, Grafana, Jenkins, and registry volumes.

## Suggested next repository changes

A practical next commit could add:

```text
README.md
ONBOARDING.md
docs/architecture.md
templates/compose/docker-compose.platform.yaml
templates/app/docker-compose.service.yaml
templates/observability/prometheus.yaml
templates/observability/promtail-config.yaml
examples/minimal-http-service/
scripts/validate-templates.sh
```

The first implementation milestone should be static and reversible: template files plus validation, not a script that modifies a user application in place.
