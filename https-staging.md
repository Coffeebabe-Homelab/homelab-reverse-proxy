# HTTPS Plan: Private Staging

**Status:** Custom DNS is configured and resolving; custom-domain TLS method remains to be selected
**Last updated:** 2026-10-22

## Goal

Route private home-lab services by custom hostname. LAN DNS resolves those hostnames to the proxy host, Tailscale MagicDNS resolves tailnet device names while Pi-hole serves DNS to tailnet clients, and Tailscale Serve keeps the existing HTTPS listener on 443 and forwards to NGINX on port 8000. Production is tracked separately and remains out of scope.

## Current context

- The home-lab host is `mecha-rat` in tailnet `tail028aa9.ts.net`.
- Tailscale owns host port 443. Its current Serve configuration is `https://mecha-rat.tail028aa9.ts.net/` forwarding to `localhost:80`.
- Nextcloud listens on host port 80. It does not own 443.
- Coding Tycoon Compose publishes its frontend on loopback port 5173. Its own NGINX serves the frontend and proxies `/api/` while preserving the backend `/api` prefix.
- Auth0 `VITE_*` settings are embedded in the frontend image. The app must be reached over HTTPS for browser authentication.

## Private routing design

Pi-hole runs on the Raspberry Pi Zero at reserved LAN address `192.168.1.254`. The router advertises that address as DNS to LAN DHCP clients. Tailscale is installed on the Pi-hole host; its Tailscale IPv4 address is configured as the tailnet global nameserver and Override local DNS is enabled. MagicDNS remains enabled for tailnet device names. Pi-hole forwards public DNS queries to its selected upstream provider.

The following private A records are configured in Pi-hole and confirmed resolving correctly. They point to the reverse-proxy host's LAN IP (not Pi-hole's address). NGINX selects the upstream using the request Host header.

| Name | Private DNS address | NGINX upstream |
| --- | --- | --- |
| `nextcloud.homelab.rat` | `mecha-rat` LAN IP | Host port 80 |
| `coding-tycoon-stg.homelab.rat` | `mecha-rat` LAN IP | Loopback port 5173 |

Detailed Pi-hole, router DNS, MagicDNS, and Tailscale DNS steps are in [TAILSCALE.md](TAILSCALE.md). Do not enable Pi-hole's **Permit all origins** without a firewall rule that limits DNS traffic to trusted LAN and Tailscale clients; never expose DNS to the public Internet.

## Tailscale Serve and TLS behavior

The shared NGINX proxy listens on port 8000. Change Tailscale Serve to forward from its existing HTTPS listener on 443 to `localhost:8000`; this does not displace Nextcloud on port 80. NGINX routes requests whose Host header is one of the configured service names. A server block also accepts `mecha-rat.tail028aa9.ts.net` for Nextcloud through the existing Serve URL.

Tailscale Serve terminates TLS with a certificate for the node's `*.ts.net` hostname. It does **not** provide a certificate for `coding-tycoon-stg.homelab.rat`. Therefore private DNS and NGINX host routing alone make the custom domain resolve and route, but do not make `https://coding-tycoon-stg.homelab.rat` certificate-valid through Serve. For Auth0 at a custom HTTPS origin, terminate TLS with a matching certificate at a separate ingress endpoint (for example NGINX on 443, using a private CA or DNS-01 certificate), which means Tailscale cannot also own that same 443 socket. Alternatively, retain Tailscale Serve on 443 and use its `*.ts.net` hostname for the HTTPS Auth0 URL while using custom DNS hostnames for LAN-only HTTP services.

If custom-domain TLS should also be served to tailnet clients, expose the separate TLS ingress over Tailscale and make its address reachable to those clients. Tailscale DNS can return an address reachable through a subnet router or a Tailscale-enabled resolver/host. Port 8000 must be firewalled to intended private sources and must not be forwarded from the Internet.

## Staging steps

1. Confirm the reserved LAN IP for `mecha-rat` and choose the private DNS zone. Pi-hole's reserved LAN address is `192.168.1.254`.
2. **Completed:** Pi-hole Local DNS has A records for `nextcloud.homelab.rat` and `coding-tycoon-stg.homelab.rat`, and both resolve to `mecha-rat`'s LAN IP. The router's LAN/DHCP DNS is set to `192.168.1.254`. Keep these records private; do not publish them publicly.
3. Tailscale DNS is configured: MagicDNS is enabled, Pi-hole's Tailscale IPv4 address is the global nameserver, and Override local DNS is enabled. Ensure tailnet clients accept Tailscale DNS settings and Pi-hole accepts DNS from tailnet clients without exposing port 53 publicly.
4. Use `nextcloud.homelab.rat` and `coding-tycoon-stg.homelab.rat` in `nginx/conf.d/homelab.conf`. Keep the `mecha-rat.tail028aa9.ts.net` alias if the existing Serve URL should continue routing to Nextcloud.
5. Deploy NGINX on port 8000 and allow it through the host firewall only from the LAN and intended tailnet sources. Keep frontend port 5173 bound to loopback.
6. Verify routing with `curl -H 'Host: <service-hostname>' http://127.0.0.1:8000/` from the server.
7. Run `sudo tailscale serve 8000` after checking the installed CLI help, then verify `sudo tailscale serve status`. This leaves tailscaled owning HTTPS port 443 and changes its local target from port 80 to port 8000.
8. Choose how to provide trusted HTTPS for the custom app hostname. Configure the matching endpoint and update Auth0 callback, logout, and web origin. Rebuild the frontend after changing any `VITE_*` value.
9. Verify DNS on LAN and tailnet clients, intended HTTPS/TLS trust, Auth0 login/callback/logout, API access, Nextcloud, and that upstream ports are not exposed directly.
