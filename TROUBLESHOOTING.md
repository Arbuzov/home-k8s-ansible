# Troubleshooting

Real issues hit while running this cluster on Raspberry Pis, and how to fix them.

## Node is `NotReady`

```bash
ssh pi@<node>
sudo systemctl status kubelet
sudo journalctl -u kubelet -f
```

Most common root causes on a Pi: swap re-enabled, missing cgroup kernel params,
`ip_forward=0`, or the CNI not initialized (no `/etc/cni/net.d` config yet — wait for
the Flannel pod, or check it can pull its image).

## cgroups not enabled

```bash
cat /proc/cmdline
# must contain: cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```

The `common` role appends these to `/boot/firmware/cmdline.txt` and reboots. The edit
is idempotent — if you ever see the string duplicated many times, collapse it to one
copy (a non-idempotent append is a classic reboot loop).

## Swap keeps coming back

```bash
swapon --show          # expect no output
free -h                # Swap should be 0
```

- Classic swapfile: `sudo swapoff -a && sudo systemctl disable --now dphys-swapfile`.
- **Raspberry Pi OS Trixie uses zram swap** via `systemd-zram-generator`. Disable it
  or kubelet won't start (`failSwapOn`):

```bash
sudo swapoff -a
printf '# disabled for kubernetes\n' | sudo tee /etc/systemd/zram-generator.conf
sudo systemctl mask systemd-zram-setup@zram0.service
```

## `ip_forward` reset after reboot

kubeadm preflight fails with `/proc/sys/net/ipv4/ip_forward contents are not set to 1`.
The role writes the sysctls to `/etc/sysctl.d/99-kubernetes.conf`; apply now with:

```bash
sudo sysctl --system
cat /proc/sys/net/ipv4/ip_forward   # 1
```

## containerd / CRI problems

```bash
sudo systemctl status containerd
sudo crictl version
sudo crictl info | grep -i cgroupDriver   # expect systemd
```

On **containerd 2.x** (Trixie) the config shipped by the Docker package is minimal; the
role regenerates it (`containerd config default`), sets `SystemdCgroup = true` and a
reachable `sandbox`/registry config.

## Images won't pull / `kubeadm upgrade apply` hangs at "Pulling images"

If `registry.k8s.io` blob downloads time out (AWS CloudFront unreachable on some
networks), nothing pulls and upgrades stall.

- **Packages**: point the apt repo at a mirror — set `kubernetes_repo_base` in
  `group_vars/all.yml` (e.g. the TUNA mirror).
- **Images**: set the cluster `imageRepository` to a reachable registry such as
  `registry.aliyuncs.com/google_containers`:

```bash
# on the control plane
kubectl -n kube-system get cm kubeadm-config \
  -o jsonpath='{.data.ClusterConfiguration}' > /tmp/cc.yaml
sed -i 's#imageRepository: registry.k8s.io#imageRepository: registry.aliyuncs.com/google_containers#' /tmp/cc.yaml
sudo kubeadm init phase upload-config kubeadm --config /tmp/cc.yaml
```

- **Offline transfer between nodes** (when only the LAN is fast): export from a node
  that already has the image and import on the target:

```bash
sudo ctr -n k8s.io images export /tmp/img.tar <image-ref>
# copy /tmp/img.tar to the target node, then:
sudo ctr -n k8s.io images import /tmp/img.tar
```

## `kubeadm upgrade node` fails: missing `cri-socket` annotation

```
node <name> doesn't have kubeadm.alpha.kubernetes.io/cri-socket annotation
```

Happens on nodes that were re-imaged/joined without the annotation. Add it and retry:

```bash
kubectl annotate node <name> \
  kubeadm.alpha.kubernetes.io/cri-socket=unix:///var/run/containerd/containerd.sock --overwrite
ssh pi@<node> 'sudo kubeadm upgrade node'
kubectl uncordon <name>
```

## `DiskPressure` on the control plane

```bash
df -h /
sudo du -xsh /etc/kubernetes/tmp/* | sort -rh | head
```

Every `kubeadm upgrade apply` leaves an ~900 MB etcd backup under
`/etc/kubernetes/tmp/kubeadm-backup-etcd-*`. They pile up. Keep the most recent and
remove the rest:

```bash
cd /etc/kubernetes/tmp
ls -dt kubeadm-backup-etcd-* | tail -n +2 | sudo xargs rm -rf
```

Also remove orphaned `Failed`/`ContainerStatusUnknown` pods left after node drains:
`kubectl delete pods -A --field-selector status.phase=Failed`.

## Useful diagnostics

```bash
ansible all -m ping -e @credentials.json
ansible masters -m shell -a "kubectl get nodes -o wide" -b -e @credentials.json
ansible all -m shell -a "free -h && df -h /" -b -e @credentials.json
ansible all -m shell -a "journalctl -u kubelet --no-pager -n 50" -b -e @credentials.json
```

See also [README.md](README.md) and [QUICKSTART.md](QUICKSTART.md).
