# Home Lab Reverse Proxy

NGINX reverse proxy configuration for private home lab services. NGINX routes requests by hostname and forwards them to services running on the `mecha-rat` host.

## Current services

| Hostname | Upstream |
| --- | --- |
| `nextcloud.homelab.rat` | Host port `80` |
| `coding-tycoon-stg.homelab.rat` | Host port `5173` |

Pi-hole provides local DNS for these names. It is hosted on a Raspberry Pi Zero at reserved LAN address `192.168.1.254`. DNS A records point to the reverse proxy host's LAN address. Tailscale is used for private remote access; MagicDNS resolves tailnet device names, and Pi-hole is configured as the tailnet DNS resolver.

## Start the proxy

On the proxy host, from this directory:

```sh
docker compose up -d
docker compose exec -T nginx nginx -t
```

The Compose file publishes NGINX on host port `8000` by default. It proxies Nextcloud to host port `80` and the staging app to host port `5173`. Make sure the upstream services are running and the host firewall limits port `8000` to intended private clients. Do not forward it from the public Internet.

Override the bind address or port by creating a local `.env` file with `PROXY_BIND_ADDRESS` and/or `PROXY_PORT`; `.env` is ignored by Git.

## DNS and Tailscale

The router advertises Pi-hole (`192.168.1.254`) as DNS to LAN clients. Pi-hole's local DNS records resolve the service names above to the proxy host. Tailnet clients reach Pi-hole using its Tailscale IP; MagicDNS can remain enabled for device names.

See [TAILSCALE.md](TAILSCALE.md) for DNS and Tailscale Serve setup, and [https-staging.md](https-staging.md) for the current private staging and HTTPS plan. Production is explicitly out of scope; see [https-production.md](https-production.md).

## Repository contents

- `docker-compose.yml` — NGINX container and port mapping.
- `nginx/conf.d/homelab.conf` — hostname routing and reverse proxy rules.
- `TAILSCALE.md` — Pi-hole DNS, Tailscale DNS, and Tailscale Serve notes.
- `https-staging.md` — private staging plan and remaining HTTPS work.
- `https-production.md` — deferred production plan.

## Safety notes

- Keep credentials, private keys, `.env` files, and generated backups out of Git.
- Pi-hole DNS and the reverse proxy are for private LAN/tailnet access only.
- Do not expose DNS port `53`, NGINX port `8000`, or upstream application ports publicly.
