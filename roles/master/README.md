# Master Role

## Description

The `master` role initializes the Kubernetes cluster master node and configures it for operation.

## Tasks

- Check cluster initialization state
- Initialize the Kubernetes cluster using kubeadm
- Configure kubeconfig for root and the pi user
- Remove the taint from the master node (optional)
- Install Flannel CNI
- **Generate and save the join token**
- Wait for the cluster to become ready

## Variables

- `kubernetes.pod_subnet`: subnet for pods (default: "10.244.0.0/16")
- `kubernetes.service_subnet`: subnet for services (default: "10.96.0.0/12")
- `kubernetes.api_server_advertise_address`: IP address of the API server
- `master_schedulable`: allow pods to be scheduled on the master (default: true)
- `kubernetes_cni`: CNI plugin (default: "flannel")

## Critical Functions

### Cluster Initialization

```bash
kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --apiserver-advertise-address=<IP> \
  --node-name=<hostname>
```

### Join Token Generation

Automatically generates a token for joining worker nodes:

```bash
kubeadm token create --print-join-command
```

The token is saved:

- On the master: `/tmp/kubeadm-join-command`
- Locally: `./kubeadm-join-command`

### CNI Installation

Installs the Flannel CNI for pod-to-pod network communication.

## Task Tags

- `master-init`: cluster initialization only
- `master-join`: join token generation only

## Created Files

- `/etc/kubernetes/admin.conf`: administrator configuration
- `/root/.kube/config`: kubeconfig for root
- `/home/pi/.kube/config`: kubeconfig for the pi user
- `/tmp/kubeadm-join-command`: command for joining worker nodes

## Verification

```bash
# Cluster status
kubectl get nodes

# System pod status
kubectl get pods -n kube-system

# CNI check
kubectl get pods -n kube-flannel
```

## Dependencies

- Role `common`
- Role `containerd`
- Role `kubernetes`

## Notes

- The role runs only once during cluster initialization
- After successful initialization, the presence of `/etc/kubernetes/admin.conf` indicates a ready cluster
- Join tokens are valid for a limited time
- For security, it is recommended to generate new tokens for each join operation
