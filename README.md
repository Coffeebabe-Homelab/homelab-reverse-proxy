# Home Lab Service Access

Private applications on `mecha-rat` are exposed through Tailscale Services. The setup is confirmed working; this repository documents its configuration and operation.

## Services

| Application | HTTPS URL | Local upstream |
| --- | --- | --- |
| Nextcloud | `https://nextcloud.tail028aa9.ts.net/` | `http://127.0.0.1:80` |
| Jenkins | `https://jenkins.tail028aa9.ts.net/jenkins` | `http://127.0.0.1:8080` |
| Coding Tycoon staging | `https://coding-tycoon-stg.tail028aa9.ts.net/tycoon` | `http://127.0.0.1:5173` |

Tailscale Services provide stable application names, private routing, and automatically provisioned HTTPS certificates. Clients must connect to the tailnet and have access to the relevant Service, including while on home Wi-Fi.

## Configuration and operation

Follow [TAILSCALE.md](TAILSCALE.md) for prerequisites, Service definitions, local upstream commands, approval, access policy, and lifecycle operations. The admin console defines each Service's name and advertised port; configure its local upstream using the CLI on `mecha-rat`.

See [https-staging.md](https-staging.md) for application settings and verification. [Production remains separate](https-production.md).

MagicDNS supplies application names. Pi-hole can continue providing DNS filtering and other local records. This repository contains documentation only; Service configuration lives on the host and in the Tailscale admin console.

Keep credentials, keys, and local configuration backups out of Git. Keep application ports private.
