# NetBird routing peer for internal resources

This Docker Compose setup runs a NetBird client in the Linux host's network
namespace so the host can serve as a **routing peer** for access to internal
resources. NetBird calls this use case **VPN-to-Site**. Clients route only the
configured internal destinations through this host; their normal internet
connection remains unchanged.

Starting the container enrolls the peer. **Resources, routing-peer assignment,
access policies, and masquerading are configured centrally in NetBird.**

## Start the peer

Requirements: a Linux Docker host, Docker Compose, `/dev/net/tun`, an existing
NetBird deployment, and a setup key from its dashboard.

```sh
cp example.env .env
# Edit .env with your setup key, management URL, and peer hostname.
docker compose up -d
docker compose exec netbird netbird status -d
```

The Compose file pins `netbirdio/netbird:0.83.0`, uses host networking, and grants
`NET_ADMIN`, `SYS_ADMIN`, and `SYS_RESOURCE`. These let NetBird manage the tunnel,
routes, and firewall in the host's network namespace. See NetBird's
[Docker installation guide](https://docs.netbird.io/get-started/install/docker).

Keep the `netbird-router` volume: it stores the peer identity under
`/var/lib/netbird`. Removing it causes a new enrollment, and the replacement peer
must be assigned as a routing peer again. Do not commit `.env`.

## Configure access to internal resources

After the peer connects:

1. Open **Networking → Network Routing → Networks** and create a Network for
   the internal environment.
2. Assign this host as its **routing peer**.
3. Add resources for the intended internal hosts, CIDRs, or domains. For example,
   `10.20.30.40/32` or `10.20.30.0/24` (replace with your real addresses).
4. Assign resources to resource groups and grant the appropriate client groups
   access to the required resources, protocols, and ports through access policies.
5. Leave **Masquerade** enabled on the routing peer.

Use NetBird's [Networks configuration](https://docs.netbird.io/manage/networks)
for this setup. An **exit node** serves internet traffic using a default route.
Do not add an exit node or publish `0.0.0.0/0` or `::/0` for this deployment.

For domain resources, this host must resolve the internal names. Configure
internal DNS as needed for the routing peer and clients.

## NAT on traffic leaving NetBird

Masquerading works for internal-resource routing and is on by default. Internal
resources see forwarded connections as coming from this host's outgoing interface
address. NetBird manages the source NAT rules; a separate hand-written masquerade
rule is normally unnecessary. Disabling masquerading preserves peer source
addresses and requires return routing for your account's NetBird address range
through this host. See
[NetBird's masquerade documentation](https://docs.netbird.io/manage/networks/masquerade).

NetBird normally enables Linux IP forwarding automatically. If it cannot change
the host setting, enable and persist it on the host. Check IPv4 forwarding with:

```sh
sysctl net.ipv4.ip_forward
```

It must report `1` for kernel forwarding. See
[routing-peer host requirements](https://docs.netbird.io/manage/networks/how-routing-peers-work#ip-forwarding)
for IPv4 and IPv6 settings. Cloud networking and upstream firewalls must also
permit the forwarded traffic.

## Services on this Linux host

Access to resources behind the routing peer does not itself grant access to
services running on the routing peer. Allow those with a separate peer-to-peer
policy targeting this host's peer group and the intended service ports. Check
for broad existing policies that may already grant access.

Use the host's NetBird IP for direct service access. Its LAN IP also needs a
resource covering that address. With this setup's normal Linux kernel forwarding,
`NB_ENABLE_LOCAL_FORWARDING` is unnecessary; it concerns userspace forwarding.
The service must listen on the addressed interface. See
[access to the routing peer itself](https://docs.netbird.io/manage/networks/how-routing-peers-work#reaching-the-routing-peer-itself).

## Environment-variable configuration

NetBird's [CLI reference](https://docs.netbird.io/get-started/cli#environment-variables)
maps flags to environment variables: `--flag-name` becomes `NB_FLAG_NAME`.

This Compose file currently passes these four variables to the container:

| Variable | Purpose |
| --- | --- |
| `NB_SETUP_KEY` | Required enrollment key. |
| `NB_MANAGEMENT_URL` | Required management service URL, such as `https://netbird.example.com`. |
| `NB_HOSTNAME` | Required peer name; also used in Compose project/container names. |
| `NB_LOG_LEVEL` | Log verbosity; defaults to `info`. |

Other relevant flag-derived settings include:

| Variable | Purpose |
| --- | --- |
| `NB_INTERFACE_NAME`, `NB_WIREGUARD_PORT`, `NB_MTU` | Tunnel interface, listening port, and MTU. |
| `NB_DISABLE_DNS`, `NB_DNS_RESOLVER_ADDRESS` | Local DNS integration. |
| `NB_DISABLE_SERVER_ROUTES` | Keep false/unset so this peer can forward assigned routes. |
| `NB_DISABLE_FIREWALL`, `NB_BLOCK_INBOUND` | Keep false/unset for this routing-peer setup. |
| `NB_BLOCK_LAN_ACCESS` | Keep false/unset when serving resources on the attached LAN. |

NetBird also documents advanced daemon variables in its
[client environment-variable reference](https://docs.netbird.io/client/environment-variables).
Examples relevant to Linux routing and troubleshooting:

| Variable | Purpose |
| --- | --- |
| `NB_USE_LEGACY_ROUTING` | Routing compatibility workaround. |
| `NB_SKIP_NFTABLES_CHECK` | Use iptables instead of probing nftables. |
| `NB_WG_KERNEL_DISABLED` | Use userspace WireGuard instead of the kernel module. |
| `NB_FORCE_RELAY` | Use relays instead of direct peer connections. |
| `NB_ENABLE_LOCAL_FORWARDING` | Userspace access to the routing peer's own addresses; not a NAT switch. |

These are selected examples, not an exhaustive list. The references also cover
connectivity timers, retries, logging, and other tuning. Check version support
before applying options from the current documentation to the pinned image.

Adding a variable to `.env` alone does **not** pass it into the container. Add
its mapping under `services.netbird.environment` in `compose.yml`, for example:

```yaml
      NB_MTU: ${NB_MTU:-1280}
```

Then recreate the container with `docker compose up -d`. Advanced variables must
reach the daemon process at startup. For persisted connection settings, consult
the CLI reference when changing an already enrolled peer.

There is no documented client environment variable that creates a Network,
publishes its internal resources, or sets its routing peer's masquerade property.
For automation, use NetBird's management API: the
[Networks API](https://docs.netbird.io/api/resources/networks) manages Networks,
resources, and routing peers. The network router's `masquerade` property controls
source NAT. An enrollment setup key is not a management API token.

## Verify from a client

On an authorized client, run `netbird networks ls` to confirm the Network is
active, then connect to an actual internal resource on an allowed service port.
For example, use `curl https://<internal-service>` for an HTTPS resource. Confirm
the destination sees this routing host's address when masquerading is enabled.
Check that resources outside the granted scope remain inaccessible. Container
health or a connected peer status alone does not verify forwarding and NAT.
