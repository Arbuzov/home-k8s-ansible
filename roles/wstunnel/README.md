# wstunnel

Deploys a [wstunnel](https://github.com/erebe/wstunnel) **server** as a native systemd
service (no Docker) — a minimal WebSocket relay that carries WireGuard UDP over a plain
`ws://` connection.

## Why this exists — the cascade

The Rostelecom TSPU applies a volumetric "TCP 16-20" freeze: it stalls any TCP flow to a
**tagged foreign destination prefix** after ~16–20 KB. It's keyed on the destination subnet
and is indifferent to the TLS shape, so switching cloud providers no longer helps (AWS, GCP,
Azure, Hetzner, DO, Vultr, OVH, Oracle, Cloudflare … all measured `DETECTED` on 2026-07-15,
including our own GCP box). It is **not** applied to Russian destination prefixes.

So we cascade through a RU hop:

```
Keenetic (WireGuard client)
   │  UDP :31820
   ▼
[home k8s: wstunnel CLIENT]        ← stays home; wraps WG UDP into WebSocket
   │  ws://<SELECTEL_IP>:80        ← TCP to a RUSSIAN IP: TCP16-20 is not applied
   ▼
[Selectel VDS: wstunnel SERVER]    ← THIS ROLE
   │  UDP :1194
   ▼
[GCP: WireGuard on its public interface]
```

The client **stays at home** on purpose: it is what wraps WG UDP into WebSocket. Move it to
Selectel and the home leg becomes bare WG UDP — exactly what Rostelecom's DPI throttles and
the whole reason wstunnel is in the stack.

## What it does

- Installs / upgrades the `wstunnel` binary from the pinned release tarball (arch auto-detected).
  Idempotency comes from the binary's own `--version` string, so no stamp file.
- Creates an unprivileged `wstunnel` system user.
- Installs a systemd unit, enables and starts it. The service runs:
  ```
  wstunnel server ws://0.0.0.0:80 --websocket-ping-frequency 5s --restrict-to <GCP_IP>:1194
  ```
- Refuses to run without `wstunnel_restrict_to` — a public `:80` relay with no allowlist is an
  open proxy.

## Binding privileged port 80

`:80` is required (a Russian-IP ingress on a well-known port maximises the odds the flow is
treated as ordinary web traffic), and `:80` is privileged. The service does **not** run as
root. Instead the unit grants only `CAP_NET_BIND_SERVICE`:

```ini
User=wstunnel
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
NoNewPrivileges=true
```

systemd (PID 1, root) sets up the ambient capability before dropping to the `wstunnel` user, so
the process can bind `:80` while holding **exactly one** Linux capability and nothing else — the
right trade-off for a byte-shuffler exposed on the public internet in a hostile network. The rest
of the hardening (`ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`) is safe because the relay
writes nothing to disk.

## Source of truth

The relay is **stateless** — its entire configuration is the systemd `ExecStart` line, rendered
from role variables. There is no DB, no keys, no secrets on this host. The only per-deployment
fact is `wstunnel_restrict_to` (the GCP endpoint), set in the gitignored `inventory-vpn.yml`.

## Usage

```bash
cp inventory-vpn.yml.example inventory-vpn.yml   # fill in the wstunnel host + wstunnel_restrict_to
ansible-playbook -i inventory-vpn.yml playbooks/deploy-wstunnel.yml
```

## GCP side (do this by hand — the role does not touch GCP)

For the cascade, the GCP WireGuard endpoint must be reachable from Selectel:

1. **WireGuard listens on the public interface.** In the GCP WG server config set
   `ListenPort = 1194` bound to `0.0.0.0` (not `127.0.0.1`). Previously WG only needed to listen
   on loopback because wstunnel ran on the same host; now the RU relay reaches it over the network.
2. **GCP firewall: allow UDP 1194 only from the Selectel IP.** Add an ingress rule
   `udp:1194` with source range `<SELECTEL_IP>/32`. Do **not** open it to `0.0.0.0/0` — that would
   expose the WG endpoint to the internet.
3. The client-side Helm chart's `wireguard.remoteHost` must be the **GCP IP** (the relay resolves
   that address), and `server.host` must be the **Selectel IP**.

## Known risk — the Selectel→GCP leg is bare WireGuard UDP (fallback: variant A)

The measurement that proved the cascade works (2026-07-15) covered **TCP** egress from Selectel.
The relay's upstream leg to GCP is **UDP** (WireGuard on :1194). If that UDP leg turns out to be
filtered at the Selectel border, the fix is:

**Variant A — dumb L4 forward, GCP untouched.** Instead of terminating WebSocket here and re-emitting
WG UDP, have Selectel do a plain TCP forward `:80 → GCP:80`, where GCP runs the wstunnel *server*.
Then the flow that reaches GCP is exactly the clean TCP stream that was measured `OK`, and the only
DPI-exposed hop (client→Selectel) stays TCP the whole way. This role would be replaced by a
socat/nftables DNAT forward; GCP's existing wstunnel server is reused as-is.

Keep this in your back pocket — variant B (this role) is the lower-latency option, variant A is the
guaranteed-clean-transport fallback.

## Variables

See [defaults/main.yml](defaults/main.yml). Per-host, in the gitignored `inventory-vpn.yml`:
`wstunnel_restrict_to` (required, `<GCP_IP>:1194`); optionally `wstunnel_version`, `wstunnel_listen`,
`wstunnel_ping_frequency`.
