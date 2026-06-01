# Home Kubernetes on Raspberry Pi — Ansible

Production-style, self-hosted Kubernetes cluster running on Raspberry Pi 4, fully
provisioned, joined and upgraded with **Ansible** and **kubeadm**. `containerd`
runtime, Flannel CNI, ARM64, per-node version pinning and zero-downtime rolling
upgrades.

[![CI](https://github.com/Arbuzov/home-k8s-ansible/actions/workflows/ci.yml/badge.svg)](https://github.com/Arbuzov/home-k8s-ansible/actions/workflows/ci.yml)
![Kubernetes](https://img.shields.io/badge/Kubernetes-1.35-blue?logo=kubernetes&logoColor=white)
![containerd](https://img.shields.io/badge/runtime-containerd-575757?logo=containerd&logoColor=white)
![Ansible](https://img.shields.io/badge/Ansible-2.9%2B-EE0000?logo=ansible&logoColor=white)
![Arch](https://img.shields.io/badge/arch-arm64-FF6600)
![License](https://img.shields.io/badge/license-MIT-green)

---

## What this project demonstrates

- **Idempotent Ansible roles** that take a bare Raspberry Pi OS install to a working
  cluster member: `common` → `containerd` → `kubernetes` → `master` / `worker`.
- **kubeadm cluster bootstrap** with automatic Flannel CNI install and **automated
  worker join** (token generated on the control plane, no manual copy/paste).
- **Per-node version pinning** in inventory, aware of the Kubernetes
  [version-skew policy](https://kubernetes.io/releases/version-skew-policy/).
- **Zero-downtime rolling minor upgrades** — `drain → kubeadm upgrade → uncordon`,
  one minor at a time, control plane first. This repo's cluster was carried
  `1.33 → 1.34 → 1.35` live (see [Engineering notes](#real-world-engineering-notes)).
- **Heterogeneous fleet**: nodes on Debian 11/12/**13 (Trixie)** and
  containerd **1.7 and 2.x** are managed by the same roles.
- **Operated on a constrained network** where `registry.k8s.io` (AWS CloudFront)
  is unreachable — solved with package/image mirrors instead of disabling checks.

This started as my actual home lab and grew into a reusable template. The
defaults reflect a real, running cluster (Kubernetes 1.35.5).

---

## Architecture

```
                 ┌──────────────────────────┐
                 │  kube-master (RPi 4)      │  control-plane + worker
                 │  etcd · apiserver · CM ·  │
                 │  scheduler · kubelet      │
                 └────────────┬─────────────┘
                              │  Flannel (VXLAN) 10.244.0.0/16
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
 ┌──────┴──────┐       ┌──────┴──────┐       ┌──────┴──────┐
 │ kube-worker-1│       │ kube-worker-2│       │ kube-worker-3│
 │  RPi (kubelet│       │  RPi (kubelet│       │  RPi (kubelet│
 │  + kube-proxy)│      │  + kube-proxy)│      │  + kube-proxy)│
 └─────────────┘       └─────────────┘       └─────────────┘
```

| Component        | Choice                                             |
|------------------|----------------------------------------------------|
| Orchestrator     | Kubernetes 1.35 (kubeadm), 1.27+ supported         |
| Container runtime| containerd (1.7 / 2.x), SystemdCgroup              |
| CNI              | Flannel (VXLAN)                                    |
| OS               | Raspberry Pi OS 64-bit (Bullseye / Bookworm / Trixie) |
| Architecture     | ARM64                                              |
| Config mgmt      | Ansible                                            |

---

## Requirements

- Raspberry Pi 4 (4 GB+ recommended for the control plane), Pi 3/4 for workers
- Raspberry Pi OS 64-bit on every node
- SSH access (password or key) with sudo
- Ansible 2.9+ on the control machine and the collections in `requirements.yml`

> The `common` role sets each node's hostname from the inventory
> (`kube-master`, `kube-worker-1`, …), so the names below are what nodes end up with.

---

## Quick start

```bash
# 1. Install Ansible collections
ansible-galaxy collection install -r requirements.yml

# 2. Credentials (git-ignored)
cp credentials.json.example credentials.json
$EDITOR credentials.json

# 3. Inventory — set your node IPs and versions
cp inventory.yml.example inventory.yml   # or edit inventory.yml directly
$EDITOR inventory.yml

# 4. Check connectivity
ansible all -m ping -e @credentials.json

# 5. Build the whole cluster
ansible-playbook site.yml -e @credentials.json
```

See **[QUICKSTART.md](QUICKSTART.md)** for the step-by-step version and
**[TROUBLESHOOTING.md](TROUBLESHOOTING.md)** when something on a Pi misbehaves.

---

## Repository layout

```text
.
├── site.yml                  # Full cluster build (all roles, all hosts)
├── inventory.yml(.example)   # Hosts + per-node Kubernetes version
├── credentials.json.example  # SSH / cluster variables template (real one git-ignored)
├── requirements.yml          # Ansible collection requirements
├── ansible.cfg
├── group_vars/               # all.yml, masters.yml, workers.yml
├── host_vars/
├── roles/
│   ├── common/               # base OS prep: hostname, cgroups, swap off, sysctl, modules
│   ├── containerd/           # containerd install + CRI/SystemdCgroup config
│   ├── kubernetes/           # kubelet/kubeadm/kubectl install (pkgs.k8s.io / mirror)
│   ├── master/               # kubeadm init, Flannel, join-token generation
│   └── worker/               # kubeadm join, role labels
├── playbooks/                # focused operations (see below)
└── tests/                    # Molecule + kind-based tests
```

---

## Ansible roles

| Role          | Responsibility |
|---------------|----------------|
| **common**    | Packages, timezone, hostname from inventory, `/etc/hosts`, **cgroup enablement** in `cmdline.txt` (idempotent), **full swap off** (incl. zram), kernel modules (`br_netfilter`, `overlay`), Kubernetes sysctls, DNS. |
| **containerd**| Installs containerd from the Docker repo (keyring + `signed-by`), enables the CRI plugin and `SystemdCgroup`. |
| **kubernetes**| Installs `kubelet`/`kubeadm`/`kubectl` from the official `pkgs.k8s.io` repo (or a mirror, see below), pins and holds versions. |
| **master**    | `kubeadm init`, installs Flannel, generates join tokens. |
| **worker**    | `kubeadm join`, applies the `node-role.kubernetes.io/worker` label. |

---

## Operations

```bash
# Build / converge the whole cluster
ansible-playbook site.yml -e @credentials.json

# Add a new worker to an existing cluster (install + join in one shot)
ansible-playbook playbooks/install-worker-with-join.yml \
  --limit kube-worker-3,kube-master -e @credentials.json

# Rolling upgrade — control plane first, then workers one by one.
# Bump kubernetes_version / kubernetes_major_minor in inventory.yml, then:
ansible-playbook playbooks/update-master-version.yml -e @credentials.json
ansible-playbook playbooks/update-worker-version.yml -e target_node=kube-worker-1 -e @credentials.json
# ... repeat per worker. NEVER skip a minor version (kubeadm forbids it).

# Roll a single node back to a previous version
ansible-playbook playbooks/rollback-node.yml -e target_host=kube-worker-1 -e rollback_version=1.34

# Maintenance / health / cleanup
ansible-playbook playbooks/maintenance.yml --tags check
ansible-playbook playbooks/maintenance.yml --tags cleanup
```

Kubernetes versions are pinned **per node** in `inventory.yml`:

```yaml
kube-master:
  kubernetes_version: "1.35.5"     # exact package version
  kubernetes_major_minor: "1.35"   # used for the apt repo URL
```

---

## Configuration

`credentials.json` (git-ignored — copy from the example):

```json
{
  "ansible_user": "pi",
  "ansible_password": "YOUR_PASSWORD",
  "ansible_ssh_private_key_file": "~/.ssh/id_rsa",
  "kubernetes_cluster_name": "raspberry-k8s",
  "kubernetes_pod_subnet": "10.244.0.0/16",
  "kubernetes_service_subnet": "10.96.0.0/12",
  "kubernetes_api_server_advertise_address": "192.168.1.100"
}
```

Use either a password or an SSH key — keep only what you need. Sudo that needs a
password is supplied with `-e ansible_become_pass=...` (or configure passwordless sudo).

---

## Real-world engineering notes

A few problems this repo actually solved — kept here because they are the
interesting part of running Kubernetes on bare Raspberry Pis on a home network.

- **Blocked `registry.k8s.io` (AWS CloudFront).** On this network, CloudFront/S3
  endpoints time out, so both the Kubernetes package CDN (`pkgs.k8s.io`) and image
  blob downloads stall. Fixes, without weakening the cluster:
  - **Packages** → the apt repo is pointed at the TUNA mirror via the
    `kubernetes_repo_base` variable in `group_vars/all.yml` (set it back to
    `https://pkgs.k8s.io` when the network allows).
  - **Images** → the cluster's `imageRepository` is set to
    `registry.aliyuncs.com/google_containers` (non-CloudFront, reachable). New
    nodes / upgrades pull from there.
- **Debian 13 (Trixie) + containerd 2.x.** `apt-key` is gone (keyring + `signed-by`
  instead), `software-properties-common` no longer exists, the `cmdline.txt` cgroup
  edit was made idempotent (it used to append every run and trigger reboots), sysctls
  moved to `/etc/sysctl.d/99-kubernetes.conf`, and `systemd-zram-generator` swap is
  disabled so kubelet starts.
- **Rolling upgrade 1.33 → 1.34 → 1.35**, one minor at a time, etcd snapshot first,
  control plane before workers, with `kubeadm upgrade apply` / `kubeadm upgrade node`.
- **Disk hygiene.** Every `kubeadm upgrade apply` leaves an ~900 MB etcd backup under
  `/etc/kubernetes/tmp/kubeadm-backup-etcd-*`; these accumulate and can trigger
  `DiskPressure` on a small SD card — prune the old ones after upgrades.

---

## Testing

```bash
# Syntax + lint
ansible-playbook --syntax-check site.yml
ansible-lint playbooks/ roles/

# Dry run against the inventory
ansible-playbook site.yml --check --diff -e @credentials.json

# Molecule (kind-based) tests
cd tests && molecule test
```

CI runs lint and syntax checks on every push — see
[`.github/workflows/ci.yml`](.github/workflows/ci.yml).

---

## Contributing

Issues and PRs welcome. Please keep roles idempotent, run `ansible-lint` before
opening a PR, and never commit `credentials.json` or a real `inventory.yml` with
secrets (both patterns are git-ignored by default).

## License

[MIT](LICENSE) © Arbuzov Sergey
