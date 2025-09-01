# Learning Kubernetes: A Hands-On Journey to Understand Its Concepts

## Summary: Mastering Kubernetes – A Hands-On Journey

Kubernetes (often called K8s) is a powerful open-source platform for automating the deployment, scaling, and management of containerized applications. Think of it as the head chef and manager of a busy restaurant chain: containers are self-contained kitchens (packaging your app with everything it needs), pods are the smallest units of work (like a table with dishes), nodes are the physical kitchens (your machines), and Kubernetes orchestrates everything—assigning tasks, scaling during rush hours, and fixing issues automatically.

Through this interactive lesson, we set up a single-node Kubernetes cluster on an Ubuntu ARM64 system (likely a Raspberry Pi or similar, running Ubuntu "Plucky" 25.04). We encountered and resolved challenges, learning key concepts:

- **Containers and Runtimes**: Used containerd as the "oven" to run containers, configured with `SystemdCgroup` for resource management (portion control) and the pause image (`sandbox` instead of `sandbox_image` in containerd 2.0.5) as a placemat for pods.
- **Cluster Setup**: Bootstrapped with `kubeadm` (the setup wizard), handling prerequisites like disabling swap (no borrowing slow storage) and enabling packet forwarding (opening communication doors).
- **Networking**: Installed Calico as the "intercom system" for pods, though some Calico pods failed on ARM64 due to resource constraints or image compatibility.
- **Scheduling and Resources**: Removed taints/labels to make the single node versatile (head chef also cooks), and addressed pod evictions (pods thrown out due to low memory/CPU) by setting resource limits.
- **Deployment and Scaling**: Ran a sample Nginx app (a web server "dish"), exposed it via a service (takeout window), and scaled replicas (more cooks for the same recipe).
- **ARM64 Lessons**: Resource constraints caused evictions, requiring careful limits (e.g., 100m CPU, 128Mi memory). Calico needed monitoring for ARM64 compatibility, and we used ARM-friendly images like `nginx:1.25`.

**Key Takeaways**: Kubernetes excels at automation but demands attention to resources, especially on ARM64 hardware. The restaurant analogy simplified complex ideas: Kubernetes manages the chaos of running apps, like a head chef ensuring a smooth kitchen. This guide redoes the setup with updates: Kubernetes v1.33 (latest stable as of August 26, 2025; v1.34 releases tomorrow), Calico v3.30.3, resource-aware deployments, and ARM64 best practices (e.g., monitor with `free -h`, use limits to avoid evictions).

**Prerequisites**: Ubuntu 25.04+ (ARM64), sudo access, 2+ CPUs/2GB+ RAM (check with `lscpu` and `free -h`), internet. Run commands in a terminal. Debug with `journalctl -u kubelet` or `kubectl describe pod <name>` if issues arise.

## Step-by-Step Guide to Set Up a Single-Node Kubernetes Cluster

### Step 1: Prepare Your Ubuntu System (Prerequisites)

This is like cleaning the kitchen before cooking to ensure compatibility.

1. **Update Packages**:

   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

   - **Explanation**: Refreshes the package list and upgrades software for the latest fixes. **Analogy**: Stocking fresh ingredients to avoid spoiled dishes.

2. **Disable Swap**:

   ```bash
   sudo swapoff -a
   sudo sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab
   ```

   - **Explanation**: Disables swap (slow memory borrowing) now and permanently by commenting out swap lines in `/etc/fstab`. **Updated**: Robust `sed` command for ARM64, fixing earlier spacing issue. **Verify**: Run `free -h` (swap should be 0). **Analogy**: No using the slow freezer during a busy service to keep things fast.

3. **Enable Packet Forwarding**:

   ```bash
   cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   net.ipv4.ip_forward = 1
   EOF
   sudo sysctl --system
   ```

   - **Explanation**: Enables network routing for pod communication. **Verify**: `sysctl net.ipv4.ip_forward` should return 1. **Analogy**: Opening doors between kitchen sections for smooth coordination.

4. **Install Dependencies**:
   ```bash
   sudo apt install -y apt-transport-https ca-certificates curl gpg
   ```
   - **Explanation**: Installs tools for secure package downloads. **Analogy**: Setting up locks and keys for safe ingredient delivery.

### Step 2: Install Container Runtime (containerd)

Containerd is the "oven" that runs containers.

1. **Install containerd**:

   ```bash
   sudo apt install -y containerd
   ```

   - **Explanation**: Installs containerd 2.0.5, compatible with ARM64. **Analogy**: Setting up the oven for cooking dishes.

2. **Create Directory and Config**:

   ```bash
   sudo mkdir -p /etc/containerd
   sudo containerd config default | sudo tee /etc/containerd/config.toml
   ```

   - **Explanation**: Creates `/etc/containerd/` and generates the default config file, fixing earlier permission issues with `tee`. **Analogy**: Writing the oven’s manual with proper access.

3. **Edit `config.toml`**:

   ```bash
   sudo nano /etc/containerd/config.toml
   ```

   - Under `[plugins."io.containerd.grpc.v1.cri"]`, ensure or add:
     ```toml
     sandbox = "registry.k8s.io/pause:3.10"
     ```
     - **Updated**: Uses `sandbox` (not `sandbox_image`) per containerd 2.0.5 naming for the pause image (pod’s network placemat). ARM64-compatible.
   - Under `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]`, ensure or add:
     ```toml
     SystemdCgroup = true
     ```
     - **Explanation**: Enables `systemd` for resource management (portion control). **Updated**: Was missing in your config; added explicitly. **Analogy**: Setting portion rules so cooks don’t hog the stove.
   - Save: Ctrl+O, Enter; Exit: Ctrl+X.

4. **Restart containerd**:
   ```bash
   sudo systemctl restart containerd
   sudo systemctl enable containerd
   ```
   - **Explanation**: Applies changes and ensures containerd starts on boot. **Verify**: `sudo systemctl status containerd` (should show “active”). **Analogy**: Turning the oven off and on to use the new manual.

### Step 3: Install Kubernetes Tools

Install `kubeadm` (setup wizard), `kubelet` (node supervisor), and `kubectl` (phone app for cluster commands).

1. **Add Repository Key**:

   ```bash
   sudo mkdir -p -m 755 /etc/apt/keyrings
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   ```

   - **Explanation**: Sets up a secure key for Kubernetes packages. **Analogy**: Getting a trusted passcode for ordering supplies.

2. **Add Repository**:

   ```bash
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```

3. **Install and Hold**:
   ```bash
   sudo apt update
   sudo apt install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   ```
   - **Explanation**: Installs tools and locks versions for stability. **Updated**: Holding prevents version mismatches, as learned. **Analogy**: Hiring the team and locking their playbook to avoid confusion.

### Step 4: Initialize the Cluster

The “grand opening” of the restaurant.

1. **Pull Images**:

   ```bash
   sudo kubeadm config images pull
   ```

   - **Explanation**: Downloads control-plane images. **Analogy**: Pre-ordering ingredients.

2. **Initialize**:
   ```bash
   sudo kubeadm init --pod-network-cidr=192.168.0.0/16
   ```
   - **Explanation**: Sets up the control-plane (API server, etcd, etc.). `--pod-network-cidr` assigns an IP range for pods. **Updated**: Save the `kubeadm join` command from the output (e.g., to `~/kubeadm-join-command.txt`) for future node additions. **Analogy**: Hiring the head chef and setting up the menu board.

### Step 5: Set Up kubeconfig

Your “keycard” to talk to the cluster.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- **Explanation**: Copies the admin config for `kubectl` access. **Verify**: `kubectl get nodes` (shows `ron-ubuntu`). **Analogy**: Issuing a keycard to send orders to the head chef.

### Step 6: Install Networking (Calico)

The “intercom system” for pod communication.

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/custom-resources.yaml
```

- **Explanation**: Deploys Calico for pod networking. **Updated**: Monitor with `watch kubectl get tigerastatus` until `Available: True`. If pods fail (e.g., `Pending`/`Error` on ARM64), reduce replicas:
  ```bash
  kubectl edit deployment calico-apiserver -n calico-apiserver
  ```
  Set `replicas: 1`. Or reinstall Calico if image issues persist. **Analogy**: Wiring phones for kitchens; resource limits can cause “line busy” errors.

### Step 7: Make Node Schedulable

Remove restrictions so the node can run workloads.

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl label nodes --all node.kubernetes.io/exclude-from-external-load-balancers-
```

- **Explanation**: Removes the “NoSchedule” taint and load-balancer exclusion label. **Updated**: Outputs like “untainted”/“unlabeled” confirm success, as seen in your run. **Analogy**: Allowing the head chef to cook, not just manage.

### Step 8: Deploy, Scale, and Clean Up Sample App (Nginx)

Test the restaurant by serving a “dish” (Nginx web server).

1. **Deploy**:
   Create `nginx-deployment.yaml`:

   ```bash
   nano nginx-deployment.yaml
   ```

   Paste:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx
   spec:
     replicas: 1 # Start low for ARM64
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
           - name: nginx
             image: nginx:1.25 # ARM64-compatible
             resources:
               limits:
                 cpu: "100m" # 0.1 CPU
                 memory: "128Mi" # 128MB
               requests:
                 cpu: "50m"
                 memory: "64Mi"
   ```

   Apply:

   ```bash
   kubectl apply -f nginx-deployment.yaml
   ```

   - **Explanation**: Deploys one Nginx pod with resource limits to avoid evictions. **Updated**: Added limits and `nginx:1.25` for ARM64, fixing earlier evictions. **Analogy**: Writing a lightweight burger recipe for one cook.

2. **Expose**:

   ```bash
   kubectl expose deployment nginx --port=80 --type=NodePort
   kubectl get services
   ```

   - **Explanation**: Creates a service (takeout window) on a port (e.g., 31948). **Verify**: Note the port with `kubectl get services`. Test: `curl http://localhost:<port>` or `curl http://192.168.66.3:<port>`. **Analogy**: Opening a window for customers to get burgers.

3. **Scale**:

   ```bash
   kubectl scale deployment nginx --replicas=2
   kubectl get pods
   ```

   - **Explanation**: Increases to two pods if resources allow. **Updated**: Start with one replica to avoid ARM64 resource issues; scale cautiously. **Analogy**: Hiring another cook for the same burger recipe. **Verify**: Check for evictions.

4. **Clean Up**:
   ```bash
   kubectl delete deployment nginx
   kubectl delete service nginx
   ```
   - **Explanation**: Removes the deployment and service. **Analogy**: Clearing the recipe and window for a fresh start.

## Tips for Success

- **ARM64**: Monitor resources (`free -h`, `kubectl describe node ron-ubuntu | grep -A 5 Allocatable`). Use limits to prevent evictions.
- **Debugging**: If pods fail (`Pending`/`Error`/`Evicted`), use `kubectl describe pod <name>` or `journalctl -u kubelet`. Share outputs for help.
- **Learning**: Deploying Nginx shows Kubernetes’ automation, like serving dishes effortlessly. Experiment with `kubectl logs` or scaling.
- **Reset**: Reset with `sudo kubeadm reset` if restarting.
