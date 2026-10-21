# HTTPS Plan: Production

**Status:** Out of scope; no production implementation is planned in this document
**Last updated:** 2026-10-22

## Scope

Production deployment is separate from the private home-lab staging setup. Do not treat the Raspberry Pi, home-lab reverse proxy, or Tailscale Serve configuration as production hosting.

When production work is authorized, plan for:

- Public DNS for production hostnames.
- Reliable hosting with managed or automatically renewed public TLS certificates.
- Private networking for backend services and databases; expose only the required ingress.
- Production-specific Auth0 applications, callback URLs, logout URLs, web origins, and secrets.
- Backups, monitoring, and a tested rollback procedure.
- A security review of firewall rules, credentials, updates, and data handling.

Tailscale Funnel is not the assumed production ingress. The home-lab proxy must not be exposed publicly as-is.

## Relationship to staging

Private staging hostnames and their DNS/TLS work are documented in [2026-10-07-https-staging.md](2026-10-07-https-staging.md). Production planning and implementation remain out of scope until requested.
