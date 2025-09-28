---
layout: post
title: "Stopping the Kubernetes Theater"
date: 2025-08-29
tags: [Experiment, Kubernetes]
---

This section extends the Kubernetes cluster setup guide, explaining how to stop the cluster (temporarily or permanently) and addressing errors with `kubectl` and MetalLB deletion, using the theater analogy.

### Analogy: Closing the Theater

The Kubernetes cluster is a theater with stages `MrgenjpLinux` (control plane) and `usv` (worker), hosting an nginx performance via a ticket booth (`10.0.0.200`). Stopping involves dismissing actors (nginx resources), dismantling the ticket booth (MetalLB), and shutting down the crew (`kubelet`, `containerd`). An error occurred when shredding the booth’s paperwork (`metallb-config.yaml`), as the papers were already gone.

### Technical Steps to Stop Kubernetes

#### Step 1: Fix MetalLB Deletion Error

**Goal**: Handle the `NotFound` error when deleting `metallb-config.yaml`.

**Question**: You hit errors running `kubectl delete -f metallb-config.yaml`:

```
Error from server (NotFound): error when deleting "metallb-config.yaml": the server could not find the requested resource
```

**Commands** (on `MrgenjpLinux` as `mrgenjp`):

```bash
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system
# If resources exist, delete explicitly
kubectl delete ipaddresspool default -n metallb-system
kubectl delete l2advertisement default -n metallb-system
```

**Explanation**:

- **Answer to Question**: The `NotFound` error means the `IPAddressPool` and `L2Advertisement` were already deleted or never created. Checking with `kubectl get` confirms their absence, allowing us to proceed.
- **Analogy**: Checks the filing cabinet for ticket booth plans; if empty, the paperwork is already shredded.
- **Verification**: Expect `No resources found`. If resources appear, explicit deletion removes them.

#### Step 2: Dismiss the Actors (Delete nginx Resources)

**Goal**: Remove nginx service and deployment.

**Commands** (on `MrgenjpLinux` as `mrgenjp`):

```bash
kubectl delete service nginx
kubectl delete deployment nginx
```

**Explanation**:

- Deletes the `LoadBalancer` service and deployment, stopping nginx pods.
- **Analogy**: Dismisses actors and closes the ticket booth.
- **Verification**:

  ```bash
  kubectl get pods
  kubectl get services
  ```

  - Expect no nginx pods and only the `kubernetes` service.

#### Step 3: Remove the Ticket Booth

**Goal**: Remove MetalLB components.

**Commands** (on `MrgenjpLinux` as `mrgenjp`):

```bash
kubectl delete -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

**Explanation**:

- Removes MetalLB’s core components (namespace, controller, speaker, CRDs).
- Skips `metallb-config.yaml` deletion due to `NotFound` error.
- **Analogy**: Dismantles the ticket booth structure.
- **Verification**:

  ```bash
  kubectl get pods -n metallb-system
  kubectl get namespace metallb-system
  ```

#### Step 4: Temporarily Stop the Cluster

**Goal**: Pause the cluster by stopping services.

**Commands** (on both `MrgenjpLinux` and `usv`):

```bash
sudo systemctl stop kubelet
sudo systemctl stop containerd
```

**Explanation**:

- Stops `kubelet` (pods) and `containerd` (containers), pausing the cluster.
- **Analogy**: Turns off stage lights and crew management.
- Restart with:

  ```bash
  sudo systemctl start containerd
  sudo systemctl start kubelet
  ```

- **Verification**:

  ```bash
  sudo systemctl status kubelet
  sudo systemctl status containerd
  kubectl get nodes
  ```

#### Step 5: Completely Reset the Cluster (Optional)

**Goal**: Reset the cluster for a fresh start.

**Commands** (on `MrgenjpLinux`, before reset):

```bash
kubectl delete -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.4/manifests/calico.yaml
```

**Commands** (on both `MrgenjpLinux` and `usv`):

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd ~/.kube
sudo systemctl stop kubelet
sudo systemctl stop containerd
```

**Explanation**:

- Deletes Calico, resets Kubernetes, and removes configurations.
- **Analogy**: Tears down the theater.
- **Warning**: Destructive; only use if starting fresh.
- **Verification**:

  ```bash
  ls /etc/kubernetes
  ls ~/.kube
  sudo systemctl status kubelet
  sudo systemctl status containerd
  ```

## Learning Points

- **MetalLB Error**: `NotFound` errors mean resources were already deleted or not created; checking with `kubectl get` confirms this.
- **kubectl Context**: Using `mrgenjp` aligns with the original setup (`~/.kube/config` in `/home/mrgenjp/.kube`).
- **Stopping Kubernetes**: Temporarily stopping preserves setup; resetting clears everything.
- **Your Journey**: Addresses all questions (repository, errors, pod names, access, distribution, MetalLB, swap/runtime, stopping, `kubectl` error, MetalLB deletion error).

## Next Steps

- **Verify Cleanup**: Share outputs of `kubectl get ipaddresspool -n metallb-system` and `kubectl get pods`.
- **Test from Mac**: Try `curl http://10.0.0.200` before stopping.
- **Restart**: Use `systemctl start` to resume.
- **Explore**: Try Ingress, databases, or monitoring.
