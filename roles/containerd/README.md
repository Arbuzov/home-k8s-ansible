# Containerd Role

## Description
The `containerd` role installs and configures containerd as the container runtime for Kubernetes.

## Tasks
- Install the required packages for working with containerd
- Add the Docker GPG key and repository
- Install the latest version of containerd.io
- Create and configure the containerd configuration file
- **Enable SystemdCgroup** for compatibility with Kubernetes
- **Fix the CRI plugin** (remove it from disabled_plugins)
- Restart and enable the containerd service

## Variables
The role requires no additional variables — it uses default values.

## Critical Settings

### SystemdCgroup
```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```
Required for correct operation with cgroup v2 in Kubernetes.

### CRI Plugin
```toml
disabled_plugins = []  # instead of ["cri"]
```
The CRI plugin must be enabled for communication with kubelet.

## Dependencies
- This role must run after the `common` role
- Requires cgroups configured on the system

## Compatibility
- Kubernetes 1.27+
- cgroup v2
- ARM64 architecture (Raspberry Pi)

## Verifying the Role
After the role completes, you can verify the setup:
```bash
# Service status
systemctl status containerd

# containerd version
containerd --version

# Check the CRI API
crictl version
```

## Notes
- The role installs containerd from the Docker repository
- The configuration is generated automatically using `containerd config default`
- Changes take effect immediately after the service is restarted
