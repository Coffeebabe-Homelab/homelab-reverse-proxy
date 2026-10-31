# HTTPS: Private Staging

**Status:** Tailscale Services working; confirmed by the operator
**Last updated:** 2026-10-28

Private staging uses one Tailscale Service per application on `mecha-rat`, with stable names and automatically provisioned HTTPS certificates. See [README.md](README.md) for URLs and local upstreams, and [TAILSCALE.md](TAILSCALE.md) for host configuration and access policy. [Production remains out of scope](https-production.md).

The following settings and checks are the operating reference for future changes. The working status reflects operator confirmation, rather than automated checks from this repository.

## Application settings

### Nextcloud

Add `nextcloud.tail028aa9.ts.net` to trusted domains. Review trusted proxies against the actual source Nextcloud sees from Serve; trust only the required proxy addresses. Set HTTPS detection and canonical/CLI URL settings as required so redirects and generated links use `https://nextcloud.tail028aa9.ts.net`.

Verify required upload sizes through Serve and review Nextcloud and its own web-server limits.

### Jenkins

Set Jenkins' external URL to `https://jenkins.tail028aa9.ts.net/jenkins`. Confirm Jenkins itself runs under the `/jenkins` context path; the Service proxy does not add that prefix. Review webhook, agent, and integration URLs and any separate connectivity they require. The Service exposes the HTTP application over 443, not a separate Jenkins agent TCP listener.

### Coding Tycoon staging

Keep the frontend on loopback port `5173` and retain its application server. Verify that `/tycoon` is a supported frontend base path and that API calls reach `/api/` on the same Service origin.

Update Auth0 Allowed Callback URLs and Allowed Logout URLs with the actual application paths under `https://coding-tycoon-stg.tail028aa9.ts.net`. Set Allowed Web Origins to that HTTPS origin without a path. Review related origin/CORS settings and update embedded `VITE_*` configuration, then rebuild and redeploy the frontend image. Remove obsolete allowed URLs after all clients have moved.

## Acceptance checks

From an allowed client connected to Tailscale, verify MagicDNS resolution and valid HTTPS without bypassing certificate verification:

```sh
curl -I https://nextcloud.tail028aa9.ts.net/
curl -I https://jenkins.tail028aa9.ts.net/jenkins/
curl -I https://coding-tycoon-stg.tail028aa9.ts.net/tycoon
```

Inspect redirect targets as well as status codes. Application login redirects can be expected; old hostnames, HTTP redirects, certificate errors, and proxy failures need investigation.

- Nextcloud: browser login, WebDAV/sync, upload and download including the required large-file size, and generated share links.
- Jenkins: login, assets, redirects, job execution, and required agents/integrations.
- Coding Tycoon: direct page load, refresh under `/tycoon`, assets, Auth0 login/callback/logout, API requests, and any WebSocket connections.
- Networking: repeat on home Wi-Fi and remotely with Tailscale connected; confirm a restricted identity cannot access the Services. Check that direct application ports do not bypass intended restrictions.
- Service health: check `sudo tailscale serve status --json` and the admin console for active Service hosts.

## Recovery

Keep private snapshots of Serve configuration, access policy, application external URLs, and Auth0 settings before changes. Restore the affected settings or upstream application if a change fails; rebuild the frontend when reverting embedded `VITE_*` values.

Inspect `sudo tailscale serve status --json` and the Service host's approval state. Restore individual endpoint mappings using the commands in [TAILSCALE.md](TAILSCALE.md). Resume drained Services with `tailscale serve advertise svc:<name>` after verifying their configuration. Avoid a global Serve reset.
