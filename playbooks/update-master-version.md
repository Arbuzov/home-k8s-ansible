# Update Master Version Playbook

## Description

The `update-master-version.yml` playbook safely upgrades the Kubernetes control-plane (master) node.

## Features

- ✅ **Phased upgrade**: kubeadm → control-plane → kubelet/kubectl
- ✅ **Automatic upgrade plan** with a diff of pending changes
- ✅ **Control-plane component upgrade**: etcd, kube-apiserver, kube-controller-manager, kube-scheduler
- ✅ **Health checks** for the API server and cluster components
- ✅ **Hold/unhold** of apt-pinned packages
- ✅ **Status verification** after the upgrade

## Usage

### Standard upgrade

```bash
# Upgrade the master node to the version defined in inventory-home.yml
ansible-playbook playbooks/update-master-version.yml \
  -e target_node=kube-master \
  -e @credentials.json
```

### Pre-upgrade dry run

```bash
# Dry-run to preview changes before applying
ansible-playbook playbooks/update-master-version.yml \
  -e target_node=kube-master \
  -e @credentials.json \
  --check --diff
```

## Upgrade Process

1. **Preflight**: Verify that the target host is the master node
2. **Unhold**: Remove apt hold from Kubernetes packages
3. **kubeadm update**: Upgrade kubeadm to the target version
4. **Upgrade plan**: Display the upgrade plan
5. **Control plane**: Upgrade control-plane components:
   - etcd
   - kube-apiserver
   - kube-controller-manager
   - kube-scheduler
6. **kubelet/kubectl**: Upgrade kubelet and kubectl
7. **Hold**: Pin packages at the new version
8. **Restart**: Restart the kubelet service
9. **Health check**: Verify the API server and cluster components
10. **Verify**: Confirm the final node status

## Components Upgraded

### Control Plane:

- **etcd**: Cluster data store
- **kube-apiserver**: Kubernetes API server
- **kube-controller-manager**: Cluster controllers
- **kube-scheduler**: Pod scheduler

### Node Components:

- **kubelet**: Node agent
- **kubectl**: CLI client
- **kubeadm**: Cluster management tool

## Variables

- `target_node`: Name of the master node (default: `kube-master`)
- `kubernetes_version`: Read from `inventory-home.yml`
- `kubernetes_major_minor`: Read from `inventory-home.yml`
- `node_role`: Must contain `master`

## Requirements

- SSH access to the master node with sudo privileges
- A correctly configured `inventory-home.yml` specifying the target K8s versions
- Stable network connection (the upgrade may take several minutes)

## Example Successful Run

```
TASK [Display final master node status] *****************************************************
ok: [kube-master] => {
    "msg": [
        "NAME          STATUS   ROLES                  AGE     VERSION   ...",
        "kube-master   Ready    control-plane,worker   2y35d   v1.33.4   ..."
    ]
}

TASK [Display component status] *************************************************************
ok: [kube-master] => {
    "msg": [
        "NAME                 STATUS    MESSAGE   ERROR",
        "scheduler            Healthy   ok        ",
        "controller-manager   Healthy   ok        ",
        "etcd-0               Healthy   ok        "
    ]
}
```

## Important Notes

⚠️ **Upgrade order**: Always upgrade the **control plane (master) node FIRST**, before any worker nodes. Kubernetes version-skew policy requires the control plane to be at the target version before workers are upgraded — upgrading workers first is unsupported and may break the cluster. Upgrade workers one at a time, one minor version at a time.

✅ **Safety**: The playbook backs up all manifests to `/etc/kubernetes/tmp/` before making changes.

🔄 **Rollback**: If problems occur, the previous manifests can be restored from that backup directory.
