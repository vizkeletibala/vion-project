# Shipping Checklist

Use this checklist before handing the LAN/internal compose bundle to a customer.

## Required Configuration

- Copy `.env.example` to `.env`.
- Set `LAN_IP` to the platform host address reachable by customer machines.
- Confirm `BASE_DOMAIN`, `REGISTRY_HOST`, and `REGISTRY_ENDPOINT` match the DNS customers will use.
- Run `sh scripts/validate-config.sh --strict` on the platform host and any build/deploy host, then fix DNS or Docker daemon warnings before shipping.

## Registry

- Confirm `REGISTRY_ENDPOINT` is the same value on the platform host, Jenkins, developer machines, and application repos.
- Confirm `getent hosts registry.vion.test` resolves to the platform host on every machine that will push or pull images.
- Configure every Docker daemon that pushes or pulls images with:

```json
{
  "insecure-registries": ["registry.vion.test:80"]
}
```

- Use `docker-compose.registry-direct.yaml` only for troubleshooting or migration.

## Application Integration

- Application repositories that deploy into this stack should use `registry.vion.test:80`, not `localhost:5000`, in Jenkinsfiles, compose defaults, and docs.
- Application compose files should consume the platform networks as external networks. With the default compose project, those names are `vion-project_edge` and `vion-project_internal`.
- Do not copy the platform stack into application repos. Keep app-specific Jenkinsfiles, Dockerfiles, compose files, and validation scripts app-local while consuming the shared stack through registry, network, routing, and observability conventions.
- For the Vitrial lane specifically, follow `docs/platform/vitrial-cicd-migration.md` before treating the arena-side scaffold as production CI/CD.

## Security Boundary

- Treat this v1 profile as LAN/internal only.
- Change default Grafana credentials before customer handoff.
- Do not expose Jenkins, Grafana, Prometheus, cAdvisor, or the registry directly to the public internet.
- For public internet use, add TLS on `443`, authentication, registry hardening, secret management, and backups first.
