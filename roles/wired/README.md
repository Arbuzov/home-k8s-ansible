# wired — copper backbone for inter-node traffic

Wi-Fi stays the front door (client access, internet egress, node IPs known to
Kubernetes); a 4-port Ethernet hub interconnects the four Pis as a private
wired segment that carries the heavy traffic — Flannel VXLAN (pod-to-pod) and
SMB (photo/media shares served by kube-master).

Background: all four nodes had `eth0` NO-CARRIER since forever — the cluster
was built Wi-Fi-only. Measured inter-node throughput over the shared 2.4GHz
airtime was 0.1–0.2 MB/s (multi-hop), which made everything media-heavy
(pigallery2, SMB) crawl. Copper fixes exactly that hop; the client leg stays
on Wi-Fi by design (no cable run to the home LAN).

## Address plan

Private subnet `10.10.0.0/24`, **no gateway, no DNS** — the default route
stays on `wlan0`. Last octet mirrors the node's LAN address so the mapping
stays memorable:

| Node | wlan0 (LAN) | eth0 (wired) |
| --- | --- | --- |
| kube-master | 192.168.99.44 | 10.10.0.44 |
| kube-worker-1 | 192.168.99.101 | 10.10.0.101 |
| kube-worker-2 | 192.168.99.102 | 10.10.0.102 |
| kube-worker-3 | 192.168.99.93 | 10.10.0.93 |

The subnet deliberately avoids `10.244.0.0/16` (pod CIDR), `10.96.0.0/12`
(service CIDR — note it covers 10.96–10.111, so e.g. 10.99.x.x would clash)
and `192.168.99.0/24` (LAN).

`wired_ip` is set per host in `inventory-home.yml`.

## What it changes on a node

A node whose `eth0` already carries its `wired_ip` is **left untouched** —
the role assumes an already-addressed interface is persistently configured.
That is true of the live fleet, which was largely hand-configured
(2026-07-17): kube-master and kube-worker-1 via `interface eth0` blocks in
`/etc/dhcpcd.conf`, kube-worker-2 via the NM auto-profile `Wired
connection 1`, kube-worker-3 via its netplan-owned `netplan-eth0` profile
(modified with `nmcli con mod` — on trixie NM persists that back through
netplan). The role exists for rebuilds and future nodes:

- **NetworkManager nodes** (bookworm/trixie): a `wired-backbone` connection
  profile — manual IPv4, `never-default`, IPv6 off, autoconnect. If the
  cable is unplugged at play time the profile simply activates later on
  carrier-up.
- **dhcpcd nodes** (bullseye): a managed block in `/etc/dhcpcd.conf`
  (`interface eth0` / `static ip_address` / `nogateway`) plus a live
  `ip addr replace` so no dhcpcd restart is needed (dhcpcd also manages
  wlan0, and the SSH session rides on it).

Kubernetes itself is untouched by this role: node IPs, kubelet, etcd and the
API server all stay on Wi-Fi addresses (control-plane chatter is light).
Moving the data plane onto copper is done by the Flannel `--iface` patch —
see `roles/master/tasks/flannel-iface.yml` and `flannel_ifaces` in
`group_vars/all.yml`.

## Rollout

```sh
# 1. Plug all four nodes into the hub FIRST — the playbook refuses to touch
#    Flannel until the full wired mesh answers pings, but the dhcpcd nodes
#    assign their eth0 IP even without carrier, and Flannel selects an iface
#    by address presence alone, so patching before cabling would advertise
#    unreachable endpoints.
# 2. Configure interfaces, verify the mesh, then patch Flannel:
ansible-playbook -i inventory-home.yml playbooks/deploy-wired-backbone.yml
```

The mesh-ping play runs BEFORE the Flannel patch; if a node fails it, check
its cable/hub port and re-run. The Flannel step also reconciles endpoints on
re-runs: pods that latched onto `wlan0` (e.g. started during a cable outage)
are rollout-restarted once the node's `flannel.alpha.coreos.com/public-ip`
annotation disagrees with its `wired_ip`.

## Known ceiling

`flannel_ifaces` lists `eth0` first with `wlan0` as fallback: a node whose
cable dies re-registers its VXLAN endpoint on the Wi-Fi address after a
flannel pod restart, but peers on the wired subnet can't reach that address
directly, so treat the fallback as "node survives reboot with hub powered
off", not "seamless degradation". If the hub is retired, remove
`flannel_ifaces` overrides and re-run the master role to return to defaults.
