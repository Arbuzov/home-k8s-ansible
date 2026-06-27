# Worker Role

## Description
The `worker` role joins a worker node to an existing Kubernetes cluster.

## Tasks
- Check cluster join status
- Copy the join command from the local machine
- Join the cluster using `kubeadm join`
- Verify that the join succeeded
- **Automatically label the node with the worker role**

## Variables
The role requires no additional variables — it uses the join command from a file.

## Key Functions

### Status Check
Checks for the existence of `/etc/kubernetes/kubelet.conf` to determine whether the node has already joined the cluster.

### Join Command
Uses the command from the file `./kubeadm-join-command`, which contains:
```bash
kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### Automatic Labeling
Applies the worker role label to the node:
```bash
kubectl label node <hostname> node-role.kubernetes.io/worker= --overwrite
```

## Task Tags
- `worker-label`: apply the role label only

## Dependencies
- Role `common`
- Role `containerd`
- Role `kubernetes`
- An initialized master cluster
- The file `./kubeadm-join-command` containing a valid token

## Verifying Operation
```bash
# Check on the master
kubectl get nodes

# Check on the worker node
systemctl status kubelet
journalctl -u kubelet
```

## Join Process
1. Verify system readiness (cgroups, swap, containerd)
2. Run kubeadm preflight checks
3. Generate kubelet configuration
4. Start kubelet
5. Join the cluster
6. Apply the role label

## Known Issues
- **Expired token**: generate a new one on the master
- **Network issues**: verify that the master is reachable
- **cgroups**: ensure cgroup memory is configured correctly
- **swap**: must be completely disabled

## Notes
- The role must be executed after the master is fully configured
- The join token has a limited lifetime
- After joining, the node is automatically labeled with the worker role
- kubelet starts automatically and registers with the cluster
