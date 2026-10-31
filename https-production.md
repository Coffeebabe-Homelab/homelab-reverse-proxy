# HTTPS Plan: Production

**Status:** Out of scope; no production implementation is planned
**Last updated:** 2026-10-28

## Scope

The Tailscale Services setup covers private home-lab access and staging. The `*.tail028aa9.ts.net` application URLs require tailnet connectivity and access grants. They do not provide public application hosting or public webhook reachability.

Production deployment requires a separate design. When authorized, plan for:

- Public DNS and a public ingress where public access is required.
- Reliable hosting with managed or automatically renewed TLS certificates.
- Private networking for backend services and databases.
- Production-specific Auth0 applications, callback URLs, logout URLs, web origins, and secrets.
- Backups, monitoring, and a tested rollback procedure.
- Firewall rules, credentials, updates, and operational ownership.

Tailscale Services may support private production resources in a future design, but the home-lab setup makes no production availability commitment. Tailscale Funnel is not the assumed public ingress.

## Relationship to staging

See [https-staging.md](https-staging.md) for application changes, acceptance checks, and recovery, and [TAILSCALE.md](TAILSCALE.md) for Service configuration.
