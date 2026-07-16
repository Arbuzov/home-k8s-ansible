# relay

Deploys a **native systemd** [socat](http://www.dest-unreach.org/socat/) L4 TCP forwarder (no
Docker) on a Selectel RU VDS — **Variant A** of the anti-DPI cascade. It blindly pipes bytes
from `:<listen>` to a GCP endpoint, plus a minimal host firewall.

## Why this exists — the cascade (Variant A)

The Rostelecom TSPU applies a volumetric "TCP 16-20" freeze: it stalls any TCP flow to a tagged
**foreign** destination prefix after ~16–20 KB. It is **not** applied to Russian prefixes. Our
GCP VPN endpoint (`vpn.example.com` = `<GCP_IP>`) measures frozen from home. So we cascade
through a RU hop:

```
Keenetic --WireGuard--> home k8s pod (xray VLESS/Reality client)
   │  TLS :2053 (Reality, SNI example.com, flow xtls-rprx-vision)
   ▼
[Selectel VDS: socat :2053 -> GCP:2053]   ← THIS ROLE — TCP to a RUSSIAN IP, TCP16-20 not applied
   │  TLS :2053 (same stream, byte-for-byte)
   ▼
[GCP: existing VLESS/Reality inbound]     ← UNTOUCHED
```

`:2053` is the **actual VLESS/Reality inbound** on GCP (verified: `openssl s_client` to GCP:2053
with SNI `example.com` returns the borrowed `example.com` cert — the Reality camouflage).
GCP `:443` is only the x-ui panel (a plain `vpn.example.com` cert), **not** the VPN — don't
forward it.

The relay is **transparent**: socat copies bytes without touching TLS, so the Reality handshake
(SNI `example.com`, the server's x25519 key) reaches GCP unchanged and authenticates exactly
as on a direct connection. The client only needs its **server address** repointed to the relay;
Reality does not care about the TCP destination IP.

## Variant A vs Variant B

- **A (this role):** Selectel is a dumb L4 forwarder; the origin server on GCP is reused as-is.
  **GCP is not touched.** The whole DPI-exposed path stays TCP — the transport measured clean.
- **B ([roles/wstunnel](../wstunnel/README.md)):** Selectel terminates a wstunnel WebSocket and
  re-emits WireGuard UDP to GCP; that requires a WireGuard/wstunnel **server on GCP** (GCP *is*
  touched) and its last leg is untested UDP. Kept as a fallback.

The two are **mutually exclusive** on the box — this role stops `wstunnel.service` if present.

## What it does

- Installs `socat` + `nftables`, creates an unprivileged `relay` system user.
- Runs `socat TCP4-LISTEN:2053,fork,reuseaddr TCP4:<GCP>:2053` as a systemd service.
- Runs fully **unprivileged**: the listen port is non-privileged (≥ 1024), so socat needs no
  capabilities — the unit empties `CapabilityBoundingSet=` and sets `NoNewPrivileges=true`, plus
  safe sandboxing (`ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, …). (For a *privileged*
  listen port you would add `AmbientCapabilities=CAP_NET_BIND_SERVICE`.)
- Installs a minimal host firewall (nftables): default-drop input, allow only loopback,
  established/related, ICMP, `:22` (SSH) and the relay port; forward drop; output open. Persisted
  via `nftables.service`. The box is single-purpose, so the role owns the firewall.

## Source of truth

The relay is **stateless** — its whole config is the systemd `ExecStart` line, rendered from
variables. The only per-deployment fact, `relay_target` (the GCP endpoint), lives in the
gitignored `inventory-vpn.yml`, not the repo.

## GCP side — nothing to do

Variant A reuses the **existing** GCP VLESS/Reality inbound on `:2053`. Do **not** touch GCP. The
only requirement is that GCP `:2053` is reachable from the Selectel IP (it already is — it served
the home client directly before).

## Home client change (the one thing you do change)

Repoint the home **xray VLESS client** at the relay: in the `wg-vless-gateway` config (the
out-of-band `config.json` Secret, `ns vpn`), set the VLESS outbound **address** from the GCP IP to
the **Selectel IP**. Keep everything else identical — **port `2053`**, SNI (`example.com`),
flow (`xtls-rprx-vision`), UUID, shortId, publicKey — because Reality still terminates on the same
GCP server. Then `kubectl -n vpn rollout restart deploy/wg-vless-gateway`.

## Usage

```bash
cp inventory-vpn.yml.example inventory-vpn.yml   # fill in the relay host + relay_target
ansible-playbook -i inventory-vpn.yml playbooks/deploy-relay.yml
```

## Known risk / fallback

If the Selectel→GCP `:2053` leg is ever throttled at the RU border, the transport is still plain
TCP (measured clean), so the likely fixes stay in-band (retry, alternate port). Variant B
(WireGuard-over-wstunnel) remains available but requires standing up a server on GCP.

## Variables

See [defaults/main.yml](defaults/main.yml). Per-host, in the gitignored `inventory-vpn.yml`:
`relay_target` (required, `<GCP_IP>:2053`); optionally `relay_listen_port` (≥ 1024),
`relay_ssh_port`.
