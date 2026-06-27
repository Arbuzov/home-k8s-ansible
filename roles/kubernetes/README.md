# Kubernetes Role

## Description
The `kubernetes` role installs Kubernetes components (kubelet, kubeadm, kubectl) and configures them for operation.

## Tasks
- Clean up legacy Kubernetes keys and repositories
- Install the GPG key from the official Kubernetes repository
- Add the official Kubernetes repository
- Install Kubernetes packages at a pinned version
- Configure kubelet and its systemd service
- Verify the installation

## Variables
- `kubernetes_major_minor`: major.minor K8s version (e.g. `"1.33"`)
- `kubernetes_version`: full K8s version (e.g. `"1.33.1"`)

## Critical Details

### Official Repository
Uses the new official repository:
```
https://pkgs.k8s.io/core:/stable:/v{{ kubernetes_major_minor }}/deb/
```

### GPG Key
The key is downloaded and stored in dearmored format for apt >= 2.4:
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key |
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

### kubelet Configuration
Creates a systemd drop-in override with the correct paths:
- Config: `/var/lib/kubelet/config.yaml` (kubeadm-compatible)
- Bootstrap kubeconfig: `/etc/kubernetes/bootstrap-kubelet.conf`
- Runtime kubeconfig: `/etc/kubernetes/kubelet.conf`

### Version Pinning
All packages are pinned to a specific version:
```bash
apt-mark hold kubelet kubeadm kubectl
```

## Installed Packages
- `kubelet={{ kubernetes_version }}-1.1`
- `kubeadm={{ kubernetes_version }}-1.1`
- `kubectl={{ kubernetes_version }}-1.1`

## Dependencies
- Role `common` (for cgroups and system settings)
- Role `containerd` (for the container runtime)

## Verifying the Installation
```bash
# kubeadm version
kubeadm version

# kubelet status
systemctl status kubelet

# Check connectivity to containerd
kubeadm config images list
```

## Compatibility
- Kubernetes 1.27+
- containerd as the container runtime
- systemd as the cgroup driver
- ARM64 architecture

## Notes
- The role creates a base kubelet configuration
- kubelet will be in an error state until it joins a cluster (this is expected)
- After installation, run `kubeadm init` or `kubeadm join`
