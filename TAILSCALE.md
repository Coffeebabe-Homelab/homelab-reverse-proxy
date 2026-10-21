# Internal DNS, MagicDNS, and Tailscale Serve

This design keeps Tailscale Serve on the server's HTTPS endpoint and forwards its traffic to NGINX on port 8000. Pi-hole is the home LAN resolver at `192.168.1.254` and also serves DNS to Tailscale clients through the Pi's Tailscale address. Internal DNS resolves custom service names to the proxy server's private address; NGINX routes requests by HTTP hostname.

The private service zone is `homelab.rat`. `192.168.1.254` is the Pi-hole address, not the reverse-proxy host address. The custom DNS records are configured and resolving; their A records point to the reverse-proxy host's LAN IP.

## 1. Create internal DNS records

Reserve a stable LAN address for `mecha-rat`. Create records in Pi-hole's **Local DNS → DNS Records**, pointing service names to the proxy host (not to Pi-hole):

| Hostname | Record | Address |
| --- | --- | --- |
| `nextcloud.homelab.rat` | A | `mecha-rat` LAN IP |
| `coding-tycoon-stg.homelab.rat` | A | `mecha-rat` LAN IP |

Use the resolver's **local DNS**, **DNS records**, or **host overrides** screen. For example:

- **Pi-hole:** **Local DNS → DNS Records → Add**; enter each full hostname and the proxy server's LAN IP.
- **AdGuard Home:** **Filters → DNS rewrites → Add DNS rewrite**; enter each hostname and the server's LAN IP.
- **Router DNS:** add equivalent local host records or DHCP reservations with local DNS names. If the router cannot create arbitrary local records, use Pi-hole/AdGuard Home as the LAN DNS resolver.

Do not create public A/AAAA records for these service names. Do not add an AAAA record unless the proxy host has a stable, reachable private IPv6 address and the firewall allows it.

Set the router's LAN/DHCP DNS server to Pi-hole at `192.168.1.254`. Do not set Pi-hole's upstream DNS to itself or set the router to forward Pi-hole's queries back to Pi-hole. Renew client DHCP leases or reconnect Wi-Fi, then check from a LAN device:

```sh
nslookup nextcloud.homelab.rat
nslookup coding-tycoon-stg.homelab.rat
```

Both answers should be the proxy host's LAN IP. DNS records only resolve names; they do not make the services reachable. Permit TCP port 8000 from the LAN on the server firewall and do not configure Internet port forwarding.

## 2. Serve DNS to Tailscale clients

Pi-hole also answers DNS for tailnet clients while they are connected to Tailscale, including when they are away from home. Tailscale runs on the Pi-hole host, so use that host's Tailscale IPv4 address (`100.x.y.z`), not its LAN address `192.168.1.254`, as the tailnet nameserver. Get the current address on the Pi with `tailscale ip -4` or from the Tailscale Machines page. Tailscale node addresses are stable for the lifetime of the node.

1. Open the Tailscale Admin Console and go to **DNS**.
2. Enable **MagicDNS**. This keeps device names such as `mecha-rat.tail028aa9.ts.net` working.
3. Under **Nameservers**, add Pi-hole's Tailscale IPv4 address as a **global nameserver**. Enable **Override local DNS** (or **Override DNS servers**) if all tailnet DNS queries should pass through Pi-hole. This is the configured design; MagicDNS remains enabled for tailnet device names.
4. Ensure Pi-hole accepts UDP and TCP port 53 from tailnet clients. Keep DNS access limited to the LAN and tailnet; do not expose port 53 to the public Internet. Pi-hole's **Permit all origins** option can create an open resolver, so do not enable it unless the host firewall explicitly limits DNS traffic to trusted LAN and Tailscale interfaces.
5. Ensure tailnet devices accept Tailscale DNS settings (the client option is usually **Use Tailscale DNS settings**).
6. From a connected tailnet client, verify `nslookup nextcloud.homelab.rat <pihole-tailscale-ip>` and `nslookup coding-tycoon-stg.homelab.rat <pihole-tailscale-ip>`, then repeat without specifying a resolver. These records are currently confirmed working. Check Pi-hole's Query Log if troubleshooting; both should resolve to the proxy host's LAN IP.

MagicDNS resolves tailnet device names; it does not create arbitrary service records. Pi-hole owns the custom service records and forwards public-domain queries to its configured upstream DNS provider. If you later prefer to send only the private zone to Pi-hole, replace the global nameserver with a restricted/split-DNS nameserver for `homelab.rat`.

## 3. Configure NGINX and Tailscale Serve

The NGINX server blocks in `nginx/conf.d/homelab.conf` must use exactly the DNS hostnames above. Compose publishes NGINX on `PROXY_BIND_ADDRESS:8000`; the default `0.0.0.0` allows both local Tailscale Serve and LAN traffic. Restrict TCP 8000 with the host firewall to the LAN and intended tailnet traffic. Do not forward it from the Internet.

Deploy and verify the proxy on the server:

```sh
cd /opt/home-lab-reverse-proxy
docker compose up -d
docker compose exec -T nginx nginx -t
curl -I -H 'Host: nextcloud.homelab.rat' http://127.0.0.1:8000/
curl -I -H 'Host: coding-tycoon-stg.homelab.rat' http://127.0.0.1:8000/
```

Then direct Tailscale Serve's existing HTTPS handler to local port 8000. Check the installed syntax first:

```sh
sudo tailscale serve --help
sudo tailscale serve 8000
sudo tailscale serve status
```

This leaves Tailscale owning the HTTPS listener and forwards its requests to NGINX. NGINX then uses the incoming `Host` header to select the upstream. If Serve's traffic arrives with the node's `*.ts.net` hostname, it will match the default server and be rejected until you add a matching server block; it will not magically become a custom hostname.

### Important HTTPS behavior

Tailscale Serve terminates TLS using a certificate for the node's `*.ts.net` hostname. DNS pointing a custom hostname at the server does not change that certificate. Clients opening `https://coding-tycoon-stg.homelab.rat` through Serve will see a certificate-name mismatch unless TLS for that custom hostname is terminated by an ingress with a matching certificate. Thus:

- The Serve-to-NGINX `:8000` path supports HTTPS at the `mecha-rat.tail028aa9.ts.net` URL, but custom hostnames need their own certificate endpoint.
- For custom HTTPS hostnames, either terminate TLS at NGINX on a separate port (with private CA or DNS-01 certificates) and direct custom DNS clients to it, or use an ingress that supports your certificate. Tailscale cannot keep ownership of the same 443 socket while NGINX also terminates custom-domain TLS there.
- Auth0 requires HTTPS for the custom app origin. Do not use the custom hostname over plain HTTP for that flow.

For LAN clients using custom HTTPS names, configure custom-domain TLS separately. For tailnet access, make that TLS endpoint reachable over Tailscale and route the custom DNS names to its reachable address.

## Upstreams and safety

- Nextcloud remains on host port 80; NGINX proxies to it. Tailscale currently owns host port 443.
- Coding Tycoon remains on loopback host port 5173. NGINX proxies to that local upstream.
- Keep port 5173 private to the host. Restrict port 8000 to the intended private clients; never forward it from the Internet.
- Set Nextcloud trusted-proxy and overwrite-protocol settings for this proxy as appropriate.
- Update Auth0 callback, logout, and web origin to the final HTTPS custom hostname, then rebuild the frontend after any `VITE_*` change.
