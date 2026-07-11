# vpn

Deploys a [3x-ui](https://github.com/MHSanaei/3x-ui) (Xray) VPN gateway as a **native systemd**
service (no Docker), with Let's Encrypt TLS that survives renewals.

## What it does

- Installs / upgrades the `x-ui` binary from the pinned release tarball (arch auto-detected).
- Installs a systemd unit, enables and starts `x-ui`.
- Obtains a Let's Encrypt cert (`certbot` **standalone HTTP-01** — needs inbound `:80`) and installs
  a **deploy hook** that copies the renewed cert into the panel dir and restarts `x-ui` — fixing
  the classic "certbot renewed but the panel still served the old cert" outage.
- Adds logrotate for `/var/log/x-ui/*.log` (root fs is small).
- Optionally seeds the panel DB on a **fresh** box from a backup — never overwrites a live one.

## Source of truth

The panel DB `/etc/x-ui/x-ui.db` holds all inbounds, UUIDs and REALITY keys. It is **not**
managed here — back it up out-of-band and restore via `vpn_db_restore_src` when bootstrapping
a new box. Configure inbounds/clients through the panel UI.

## Usage

```bash
cp inventory-vpn.yml.example inventory-vpn.yml   # fill in host + vpn_domain + email
ansible-playbook -i inventory-vpn.yml playbooks/deploy-vpn.yml
```

## TLS

Cert issuance uses certbot's **standalone HTTP-01** challenge, so the host needs inbound `:80`
reachable (open it in the cloud firewall) and `vpn_domain` must already resolve to this host before
the first run — Let's Encrypt validates by connecting to `http://vpn_domain`. Renewals run headless
via `certbot.timer` (also standalone, so keep `:80` reachable); the deploy hook copies the fresh
cert into the panel dir and restarts `x-ui`. Set `vpn_certbot_email` for expiry notices, or leave it
empty to register without an email.

> Migrating from an existing box? Seed `/etc/letsencrypt/` (and the panel cert dir) from a backup
> first — the role skips issuance when a cert already exists, so no `:80`/DNS dance on cutover.
>
> REALITY inbounds don't need this cert at all (they borrow the donor's TLS) — it's only for the
> panel HTTPS and the httpupgrade reserve inbound. Leave `vpn_domain` empty to skip TLS entirely.

## Variables

See [defaults/main.yml](defaults/main.yml). Per-host, in the gitignored `inventory-vpn.yml`:
`vpn_domain`, `vpn_certbot_email`; optionally `vpn_db_restore_src` and `vpn_xui_version`.
