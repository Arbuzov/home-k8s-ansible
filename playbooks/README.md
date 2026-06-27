# Install Worker with Join Playbook

## Overview

The `install-worker-with-join.yml` playbook automates the complete process of installing and joining a worker node to an existing Kubernetes cluster.

## Features

- **Full installation** of all components on the worker node
- **Token generation** on the master without updating the master
- **Swap disable** and cgroups configuration
- **Cluster join** without ignoring preflight checks
- **Automatic label** `node-role.kubernetes.io/worker` applied to the node

## Usage

### Basic usage

```bash
ansible-playbook playbooks/install-worker-with-join.yml \
  --limit kube-worker-3,kube-master \
  -e @credentials.json
```

### With specific nodes

```bash
ansible-playbook playbooks/install-worker-with-join.yml \
  --limit kube-worker-2,kube-master \
  -e @credentials.json
```

### Run only specific stages (tags)

```bash
# Component installation only
ansible-playbook playbooks/install-worker-with-join.yml \
  --tags worker-setup \
  --limit kube-worker-2 \
  -e @credentials.json

# Token generation only
ansible-playbook playbooks/install-worker-with-join.yml \
  --tags join-token \
  --limit kube-master \
  -e @credentials.json

# Join only
ansible-playbook playbooks/install-worker-with-join.yml \
  --tags worker-join \
  --limit kube-worker-2,kube-master \
  -e @credentials.json

# Label only
ansible-playbook playbooks/install-worker-with-join.yml \
  --tags label \
  --limit kube-worker-2,kube-master \
  -e @credentials.json
```

## Execution stages

### 1. Component installation (worker-setup)

- Run the `common` role (cgroups, swap, system settings)
- Run the `containerd` role (container runtime)
- Run the `kubernetes` role (kubelet, kubeadm, kubectl)

### 2. Token generation (join-token)

- Verify cluster initialisation on the master
- Generate a new join token
- Save the token on the master and locally

### 3. Cluster join (worker-join)

- Disable swap
- Read the join command from the local file
- Execute `kubeadm join` without ignoring preflight checks
- Verify kubelet status

### 4. Label application (label)

- Apply the label `node-role.kubernetes.io/worker=` to the worker node

## Requirements

### Worker node

- Raspberry Pi OS 64-bit (Bookworm)
- SSH access with the provided credentials
- Internet connectivity

### Master node

- An initialised Kubernetes cluster
- Accessible `/etc/kubernetes/admin.conf`
- Ability to run `kubeadm token create`

### General

- Network reachability between the master and worker node
- Correct credentials in `credentials.json`

## Variables

The playbook uses the standard variables from `group_vars/all.yml`:

- `kubernetes_version`: Kubernetes version to install
- `kubernetes_major_minor`: major.minor version

## Verifying the result

After a successful run:

```bash
# On the master
kubectl get nodes

# Expected output:
# NAME            STATUS   ROLES                  AGE   VERSION
# kube-master      Ready    control-plane,worker   5d    v1.33.1
# kube-worker-1    Ready    worker                 5d    v1.33.1
# kube-worker-3    Ready    worker                 1m    v1.33.1
```

## Troubleshooting

### Preflight errors

The playbook **does not ignore** preflight checks. If errors occur:

- Check cgroups: `cat /proc/cmdline`
- Check swap: `free -h`
- Check containerd: `systemctl status containerd`

### Token expired

If the token has expired, re-run only the token generation stage:

```bash
ansible-playbook playbooks/install-worker-with-join.yml \
  --tags join-token,worker-join \
  --limit kube-worker-3,kube-master \
  -e @credentials.json
```

### Node does not join

1. Check network reachability between nodes
2. Ensure the master is reachable on port 6443
3. Check kubelet logs: `journalctl -u kubelet -f`

## Notes

- The playbook is idempotent — it can be run multiple times safely
- Join tokens are valid for a limited time (24 hours by default)
- After joining, the node automatically receives the worker role label
- For security, use SSH keys instead of passwords in production
