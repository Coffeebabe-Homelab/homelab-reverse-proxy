# Tailscale Services

**Status:** Working; confirmed by the operator
**Last updated:** 2026-10-28

Three Tailscale Services are hosted by `mecha-rat`. Each Service has its own stable MagicDNS name and virtual IP (TailVIP). Tailscale Serve terminates HTTPS and forwards directly to the local application.

## Prerequisites

- Use an active tailnet with Owner, Admin, or Network admin permissions to configure Services.
- Run Tailscale 1.86 or later on the host. Prefer current clients; clients 1.94 and later do not need `accept-routes` for Services. Linux clients 1.93 or earlier require `sudo tailscale set --accept-routes`.
- Give the Service host a tag-based identity, such as `tag:homelab`. Define tag ownership and review existing grants and SSH rules before converting a user-owned device to a tagged device.
- Enable MagicDNS and HTTPS certificates in the tailnet. Clients must accept Tailscale DNS settings.
- Confirm each upstream is reachable from the host's Tailscale daemon at the loopback address below. Keep application ports private; publishing a Service does not restrict existing direct LAN access automatically.

Service host identity tags and optional Service identity tags are separate settings. A user-authenticated host cannot advertise a Service until it uses a tag-based identity.

See the official [Services setup guide](https://tailscale.com/docs/features/tailscale-services) and [Serve reference](https://tailscale.com/docs/reference/tailscale-cli/serve).

## 1. Inspect the host configuration

Run on `mecha-rat`:

```sh
tailscale version
sudo tailscale serve status --json
sudo tailscale serve --help
```

Save configuration snapshots privately outside this repository before making changes, including access policy, application external URLs, and Auth0 settings.

## 2. Define Services

In the Tailscale admin console, open **Services**, select **Advertise → Define a Service**, and create:

| Name | Endpoint | Local upstream | Client URL |
| --- | --- | --- | --- |
| `nextcloud` | `tcp:443` | `http://127.0.0.1:80` | `https://nextcloud.tail028aa9.ts.net/` |
| `jenkins` | `tcp:443` | `http://127.0.0.1:8080` | `https://jenkins.tail028aa9.ts.net/jenkins` |
| `coding-tycoon-stg` | `tcp:443` | `http://127.0.0.1:5173` | `https://coding-tycoon-stg.tail028aa9.ts.net/tycoon` |

Confirm the generated names match the tailnet's actual DNS suffix. No Pi-hole records are needed for these names.

## 3. Configure and advertise the host

The admin console defines the Service identity and advertised port. Configure the local upstream separately on `mecha-rat` with these commands when setting up or restoring the endpoints:

```sh
sudo tailscale serve --service=svc:nextcloud --https=443 http://127.0.0.1:80
sudo tailscale serve --service=svc:jenkins --https=443 http://127.0.0.1:8080
sudo tailscale serve --service=svc:coding-tycoon-stg --https=443 http://127.0.0.1:5173
sudo tailscale serve status --json
```

Service mode automatically advertises the endpoints and runs persistently in the background. HTTPS certificates are provisioned for the Service names.

These root handlers forward the whole application. They do not introduce or remove `/jenkins` or `/tycoon`; verify that the upstream applications already support those paths. Whole-application forwarding also allows the staging frontend's `/api/` requests to reach its own application server.

## 4. Approve hosts and configure access

Approve each pending host advertisement on the admin console's Services page, unless an existing auto-approval policy already permits it. Host approval allows advertisement; it does not itself grant users access.

Merge a grant into the existing tailnet policy for the intended users or groups. For example, this fragment permits all tailnet members to reach the three Services on TCP 443; replace the source with a narrower group where appropriate:

```json
{
  "grants": [
    {
      "src": ["autogroup:member"],
      "dst": ["svc:nextcloud", "svc:jenkins", "svc:coding-tycoon-stg"],
      "ip": ["tcp:443"]
    }
  ]
}
```

Preserve existing policy entries and review broader grants that may already permit access. Use separate rules if Jenkins should have fewer users than the other applications. Test access with both allowed and restricted identities.

## DNS, HTTPS, and LAN behavior

Pi-hole runs on `192.168.1.254` for LAN DNS, with its Tailscale IP configured as the tailnet global resolver and Override local DNS enabled. MagicDNS supplies Service names, so these applications do not need Pi-hole records. Remove obsolete `homelab.rat` records once no clients use them.

DNS aliases do not provide matching HTTPS certificates for custom names. The canonical URLs use `*.tail028aa9.ts.net` and Tailscale-managed certificates.

Clients need Tailscale connectivity and Service access even while at home. LAN devices without Tailscale require a separately designed gateway arrangement.

## Service maintenance

Use [https-staging.md](https-staging.md) to verify application behavior after changes. Drain a Service before modifying active endpoints. Avoid a global Serve reset, which could remove unrelated endpoints.

To withdraw an individual Service for maintenance, drain it first:

```sh
sudo tailscale serve drain svc:nextcloud
```

Wait for existing connections to finish, then remove only that Service's configuration if required:

```sh
sudo tailscale serve clear svc:nextcloud
```

Repeat for the other Service names as needed. To resume a drained Service whose configuration remains intact, run `sudo tailscale serve advertise svc:nextcloud`. Refer to the [Service lifecycle documentation](https://tailscale.com/docs/features/tailscale-services) before changing active endpoints.
