# vpn

Deploys a [3x-ui](https://github.com/MHSanaei/3x-ui) (Xray) VPN gateway as a **native systemd**
service (no Docker), with Let's Encrypt TLS that survives renewals.

## What it does

- Installs / upgrades the `x-ui` binary from the pinned release tarball (arch auto-detected).
- Installs a systemd unit, enables and starts `x-ui`.
- Obtains a Let's Encrypt cert (`certbot` **DNS-01 via Route53** — no inbound `:80`) and installs
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

## TLS / Route53

Cert issuance uses certbot's DNS-01 plugin against Route53, so nothing needs `:80` open in the
Azure NSG. Supply an IAM key (`vpn_route53_access_key` / `vpn_route53_secret_key`) scoped to the
zone: `route53:ChangeResourceRecordSets`, `route53:GetChange`, `route53:ListHostedZones`. The role
writes it to `/root/.aws/credentials` (0600) so the `certbot.timer` renewals authenticate too.

> REALITY inbounds don't need this cert at all (they borrow the donor's TLS) — it's only for the
> panel HTTPS and the httpupgrade reserve inbound. Leave `vpn_domain` empty to skip TLS entirely.

## Variables

See [defaults/main.yml](defaults/main.yml). Per-host, in the gitignored `inventory-vpn.yml`:
`vpn_domain`, `vpn_certbot_email`, `vpn_route53_access_key`, `vpn_route53_secret_key`; optionally
`vpn_db_restore_src` and `vpn_xui_version`.
