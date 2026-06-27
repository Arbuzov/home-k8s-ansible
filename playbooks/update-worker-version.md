# Update Worker Version Playbook

## Description
The `update-worker-version.yml` playbook is designed for quickly upgrading the Kubernetes version on worker nodes.

## Features
- ✅ **Automatically reads versions from inventory-home.yml**
- ✅ **Safe upgrade** with node drain/uncordon
- ✅ **Unhold packages** before upgrading
- ✅ **Hold packages** after upgrading
- ✅ **Update kubelet configuration**
- ✅ **Status verification** after upgrading

## Usage

### Quick upgrade
```bash
# Upgrade kube-worker-2 to the version defined in inventory-home.yml
ansible-playbook playbooks/update-worker-version.yml \
  -l kube-worker-2 \
  -e @credentials.json
```

### Upgrade with parameters
```bash
# Upgrade without drain/uncordon
ansible-playbook playbooks/update-worker-version.yml \
  -l kube-worker-1 \
  -e @credentials.json \
  -e drain_node=false

# Target a specific node
ansible-playbook playbooks/update-worker-version.yml \
  -e target_node=kube-worker-3 \
  -e @credentials.json
```

## Upgrade Process

1. **Prepare**: Display node information
2. **Drain**: Cordon and drain the node (optional)
3. **Unhold**: Unhold Kubernetes packages
4. **Update**: Upgrade kubelet, kubeadm, kubectl
5. **Hold**: Hold packages at the new version
6. **Configure**: Update kubelet configuration
7. **Restart**: Restart the kubelet service
8. **Wait**: Wait for kubelet to become ready
9. **Uncordon**: Return the node to service (optional)
10. **Verify**: Check the final status

## Variables

- `target_node`: Name of the node to upgrade (default: kube-worker-2)
- `drain_node`: Whether to drain the node before upgrading (default: true)
- `kubernetes_version`: Read from inventory-home.yml for each node
- `kubernetes_major_minor`: Read from inventory-home.yml for each node

## Requirements

- Access to the master node for drain/uncordon operations
- SSH access to the target node
- A correctly configured inventory-home.yml with K8s versions

## Example of a Successful Run

```
TASK [Display final node status] ****************************************************
ok: [kube-worker-2] => {
    "msg": [
        "NAME            STATUS   ROLES    AGE   VERSION   ...",
        "kube-worker-2   Ready    worker   65m   v1.33.4   ..."
    ]
}
```
