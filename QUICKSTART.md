# Quick start — Kubernetes on Raspberry Pi

## Prerequisites

- Raspberry Pi 4 (4 GB+) for the control plane, Pi 3/4 for workers
- Raspberry Pi OS 64-bit on every node
- SSH access to all nodes (password or key) with sudo
- `ansible-core` 2.16+ on your control machine (install the collections with
  `ansible-galaxy collection install -r requirements.yml`)

> The `common` role renames hosts to `kube-master`, `kube-worker-1`, `kube-worker-2`, …
> based on the inventory, so don't be surprised when hostnames change.

## 1. Install Ansible collections

```bash
ansible-galaxy collection install -r requirements.yml
```

## 2. Credentials

```bash
cp credentials.json.example credentials.json
$EDITOR credentials.json
```

```json
{
  "ansible_user": "pi",
  "ansible_password": "your-password",
  "ansible_ssh_private_key_file": "~/.ssh/id_rsa",
  "kubernetes_cluster_name": "raspberry-k8s",
  "kubernetes_pod_subnet": "10.244.0.0/16",
  "kubernetes_service_subnet": "10.96.0.0/12",
  "kubernetes_api_server_advertise_address": "192.168.1.100"
}
```

Use either a password **or** an SSH key — keep only the fields you need.
`credentials.json` is git-ignored.

## 3. Inventory

Edit `inventory-home.yml` with your node IPs and the Kubernetes version per node:

```yaml
all:
  children:
    kubernetes_cluster:
      children:
        masters:
          hosts:
            kube-master:
              ansible_host: 192.168.1.100
              node_role: master
              node_taints: []            # master also runs workloads
              kubernetes_version: "1.35.5"
              kubernetes_major_minor: "1.35"
        workers:
          hosts:
            kube-worker-1:
              ansible_host: 192.168.1.101
              node_role: worker
              kubernetes_version: "1.35.5"
              kubernetes_major_minor: "1.35"
            kube-worker-2:
              ansible_host: 192.168.1.102
              node_role: worker
              kubernetes_version: "1.35.5"
              kubernetes_major_minor: "1.35"
  vars:
    ansible_python_interpreter: /usr/bin/python3
    ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
```

Versions are set **per node**. Workers may trail the control plane by at most one
minor version (Kubernetes version-skew policy).

## 4. Install the cluster

```bash
# Check connectivity first
ansible all -m ping -e @credentials.json

# Build the cluster (add -e ansible_become_pass=... if sudo needs a password)
ansible-playbook site.yml -e @credentials.json
```

## 5. Verify

```bash
ssh pi@192.168.1.100
kubectl get nodes -o wide
# NAME            STATUS   ROLES                  AGE   VERSION
# kube-master     Ready    control-plane,worker   5m    v1.35.5
# kube-worker-1   Ready    worker                 3m    v1.35.5
# kube-worker-2   Ready    worker                 2m    v1.35.5
```

## Add a worker later

```bash
ansible-playbook playbooks/install-worker-with-join.yml \
  --limit new-node,kube-master -e @credentials.json
```

## Upgrade

Bump `kubernetes_version` / `kubernetes_major_minor` in `inventory-home.yml`, then upgrade
**the control plane first, workers afterwards, one minor version at a time**:

```bash
ansible-playbook playbooks/update-master-version.yml -e @credentials.json
ansible-playbook playbooks/update-worker-version.yml -e target_node=kube-worker-1 -e @credentials.json
# repeat per worker
```

Always test an upgrade on a single worker first. Take an etcd snapshot before
touching the control plane.

## Notes

- Use Raspberry Pi OS **64-bit** only.
- Don't ignore kubeadm preflight errors — fix the underlying issue.
- Swap must be fully off (the `common` role handles `dphys-swapfile` and zram).
- The `cgroup_enable=...` kernel parameters are mandatory on Raspberry Pi.

More: [README.md](README.md) · [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
