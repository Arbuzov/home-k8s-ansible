# Common Role

## Description

The `common` role performs baseline system configuration required to run Kubernetes on Raspberry Pi.

## Tasks

- Update the package cache and install system packages
- **Set the hostname** from `inventory_hostname`
- **Update `/etc/hosts`** with the new hostname
- Configure the timezone
- Configure the GPU memory split for Raspberry Pi
- **Enable cgroup memory and cpuset** for Kubernetes
- **Disable swap** (Kubernetes requirement)
- Load required kernel modules (`br_netfilter`, `overlay`)
- Apply sysctl parameters for Kubernetes
- Configure DNS

## Variables

- `system_packages`: list of system packages to install
- `timezone`: timezone (default: `"Europe/Moscow"`)
- `arm_memory_split`: GPU memory on ARM (default: `16`)
- `cgroup_enabled`: enable cgroup settings (default: `true`)
- `swap_disabled`: disable swap (default: `true`)
- `inventory_hostname`: used to set the system hostname

## Critical Changes

### cgroup Settings

The role appends the following critical parameters to `/boot/firmware/cmdline.txt`:

```
cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```

**Parameter order matters!** A reboot is required after this change.

### Disabling Swap

Swap is disabled completely:

- Immediate: `swapoff -a`
- Permanent: comment out the swap entry in `/etc/fstab`

## Handlers

- `reboot system`: reboots the system when critical parameters change

## Compatibility

- Raspberry Pi OS (64-bit)
- Kubernetes 1.27+
- containerd as the container runtime

## Notes

- The role must run with root privileges (`become: true`)
- A reboot may be required after the role completes
- Changes to `cmdline.txt` take effect only after a reboot
