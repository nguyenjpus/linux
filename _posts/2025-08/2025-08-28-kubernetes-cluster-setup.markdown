# Setting Up a Kubernetes Cluster: A Theater Analogy

This document summarizes the process of setting up a Kubernetes cluster with two nodes (`mrgenjplinux` as the control plane on amd64, `usv` as a worker on arm64) to host an nginx website, accessible via a single IP (`10.0.0.200`). The analogy of a theater company makes the concepts memorable, followed by technical steps, commands, and explanations addressing all questions asked during the process.

## Analogy: The Kubernetes Theater

- **Theater**:
  You prepared the stages by installing essential tools (like lighting and sound equipment, aka socat, apt-transport-https) and adding a playbook (Kubernetes repository) to each stage’s script library (/etc/apt/sources.list.d/kubernetes.list).
  You confirmed the playbook versions (kubeadm, kubectl, kubelet) to ensure both stages could perform the same show (Kubernetes v1.30).

- **Hiring the Crew**:

On the main stage (mrgenjplinux), you hired a director (control plane) using kubeadm init, setting up a backstage communication network (--pod-network-cidr=192.168.0.0/16) for actors to talk across stages.
You fixed missing crew members (installed socat) and opened communication channels (enabled IP forwarding) to ensure the director could coordinate the show.
The secondary stage (usv) joined the production with kubeadm join, becoming a worker stage under the director’s guidance.

- **Wiring the Stages**:

You installed a communication system (Calico) to connect actors (pods) across stages, ensuring they could share lines (network traffic) seamlessly.
You confirmed the crew (system pods like calico-node, kube-proxy) was present on both stages, with usv hosting its share, even if their name tags didn’t say “usv” (random pod names like calico-node-znxzl).

- **Casting the Actors:**:

You cast two actors (nginx pods) for the performance (nginx deployment) and scaled to two performers (--replicas=2).
Initially, both actors crowded onto the secondary stage (usv) because the main stage had a “VIP only” sign (control-plane taint). You removed this restriction, spreading actors across both stages for a balanced show.

- **Opening the Ticket Booth::**:

Initially, you opened specific doors on each stage (NodePort at 10.0.0.45:31569 and 10.0.0.46:31569), but the audience had to know which door to use.
To simplify, you hired a ticket booth operator (MetalLB) to provide a single entry point (10.0.0.200). You gave them a stack of tickets (IP pool 10.0.0.200-10.0.0.250) and a megaphone (L2Advertisement via ARP) to announce the booth’s address to the audience (your home network).
You upgraded the performance’s access to a LoadBalancer service, so the audience could enter through http://10.0.0.200 and enjoy the show (nginx welcome page) without knowing which stage the actors were on.

- **Theater**: The Kubernetes cluster, with two stages: `mrgenjplinux` (main stage, control plane) and `usv` (secondary stage, worker node).

- **Performance**: An nginx website hosted by actors (nginx pods).

- **Goal**: Allow the audience (clients like your Mac or `usv`) to access the show through a single ticket booth (IP `10.0.0.200`), hiding which stage hosts the actors.

- **Crew**: System components (containerd for containers, Calico for networking, MetalLB for load balancing, `kube-proxy` for routing) that coordinate the show.

- **Challenges**: Preparing stages, spreading actors, and providing a unified entry point, mimicking a real-world website.

## Technical Steps and Questions Answered

### Step 1: Prepare the Stage Floor (Disable Swap and Install Container Runtime)

**Goal**: Clear the stage floor (disable swap) and hire a stage crew (containerd) to manage actors (containers) on both nodes.

**Commands** (on both `mrgenjplinux` and `usv`):

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo apt update
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

**Explanation**:

- Disabled swap (`swapoff -a` and commented out swap lines in `/etc/fstab`) because Kubernetes requires swap to be off for predictable memory management.
- Installed `containerd`, the container runtime, to manage containers (actors) that Kubernetes runs as pods.
- Configured `containerd` with a default config file and enabled its service to start automatically.
- **Question Addressed**: You noted this step was missing. It’s now included as the foundational setup for both nodes, ensuring Kubernetes can run containers.

### Step 2: Equip the Stages (Install Kubernetes Dependencies)

**Goal**: Set up both nodes with Kubernetes tools and repository, like equipping stages with lighting and sound.

**Commands** (on both nodes):

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
```

**Explanation**:

- Installed dependencies (`apt-transport-https`, `ca-certificates`, `curl`, `gnupg`) for secure package downloads.
- Added the Kubernetes apt repository for version 1.30 to access `kubeadm`, `kubectl`, and `kubelet`.
- **Question (Are we good to run `apt-cache policy kubeadm`?)**: You asked if the repository setup was correct. I confirmed it was, as `apt update` succeeded, and suggested `apt-cache policy kubeadm` to verify package versions (e.g., `1.30.5-00`).

### Step 3: Install Kubernetes Components

**Goal**: Install the actors’ scripts (`kubeadm`, `kubectl`, `kubelet`) and start the stagehand service (`kubelet`).

**Commands** (on both nodes):

```bash
sudo apt install -y kubelet=1.30.5-00 kubeadm=1.30.5-00 kubectl=1.30.5-00
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

**Explanation**:

- Installed Kubernetes components at version 1.30.5 for compatibility.
- `apt-mark hold` prevented version mismatches during updates.
- Enabled `kubelet`, the stagehand that runs pods on each node.

### Step 4: Hire the Director (Set Up Control Plane)

**Goal**: Set up the director (control plane) on `mrgenjplinux`, fixing crew and communication issues.

**Question**: You hit errors during `kubeadm init`:

- `socat not found`
- `/proc/sys/net/ipv4/ip_forward contents are not set to 1`

**Commands** (on both nodes):

```bash
sudo apt update
sudo apt install -y socat
sudo sysctl -w net.ipv4.ip_forward=1
sudo sh -c 'echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf'
sudo sysctl -p
```

**Commands** (on `mrgenjplinux`):

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Explanation**:

- **Answer to Question**: Installed `socat` (a networking tool) and enabled IP forwarding for pod communication, made permanent via `/etc/sysctl.conf`.
- Initialized the control plane with `kubeadm init`, setting a pod network CIDR (`192.168.0.0/16`) for Calico.
- Configured `kubectl` to communicate with the control plane using `admin.conf`.

### Step 5: Add the Secondary Stage (Join Worker Node)

**Goal**: Have `usv` join the theater as a worker stage.

**Question**: After `kubeadm join` on `usv`, `kubectl get nodes` failed with a `connection refused` error (`localhost:8080`).

**Commands** (on `mrgenjplinux`):

```bash
sudo cp /etc/kubernetes/admin.conf /home/mrgenjp/admin.conf
sudo chown mrgenjp:mrgenjp /home/mrgenjp/admin.conf
scp /home/mrgenjp/admin.conf root@10.0.0.46:/root/admin.conf
```

**Commands** (on `usv`):

```bash
sudo kubeadm join 10.0.0.45:6443 --token gj2l41.btv1yw6ufzfs7kmc --discovery-token-ca-cert-hash sha256:d2df85a631e5cc3c279a22d9b7f12e82275c337c3d7e7dcc96653373d6a555ca
mkdir -p /root/.kube
mv /root/admin.conf /root/.kube/config
chown root:root /root/.kube/config
```

**Explanation**:

- **Answer to Question**: The error occurred because `kubectl` on `usv` wasn’t configured to reach the control plane (`10.0.0.45:6443`). Copied `admin.conf` to `usv` as `/root/.kube/config` to fix this.
- `kubeadm join` added `usv` to the cluster, confirmed by `kubectl get nodes` on `mrgenjplinux` showing both nodes as `Ready`.

### Step 6: Wire the Stages (Install Calico)

**Goal**: Install Calico to connect actors (pods) across stages.

**Question**: You asked why `kubectl get pods -n kube-system` didn’t show `usv` in pod names, wondering if it was weird.

**Commands** (on `mrgenjplinux`):

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.4/manifests/calico.yaml
kubectl get pods -n kube-system -o wide
```

**Explanation**:

- **Answer to Question**: Pods like `calico-node-znxzl` and `kube-proxy-z6bfd` on `usv` had random names because DaemonSets generate them, not node-specific names like control-plane pods (`etcd-mrgenjplinux`). The `-o wide` output confirmed these pods ran on `usv` (IP `10.0.0.46`).
- Calico enabled pod networking across the `192.168.0.0/16` CIDR.

### Step 7: Cast the Actors (Deploy nginx)

**Goal**: Deploy the nginx performance and make it accessible.

**Questions**:

- How to access nginx using `curl http://10.0.0.45:<external-port>` from `usv` or Mac?
- Why were both nginx pods on `usv`?

**Commands** (on `mrgenjplinux`):

```bash
kubectl create deployment nginx --image=nginx
kubectl scale deployment nginx --replicas=2
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get services
curl http://10.0.0.45:31569
curl http://10.0.0.46:31569
```

**Commands** (on `usv`):

```bash
kubectl delete pod -l app=nginx
kubectl get pods -o wide
curl http://10.0.0.45:31569
curl http://10.0.0.46:31569
```

**Explanation**:

- **Answer to Access Question**: The `NodePort` service exposed nginx on port 31569 on both nodes (`10.0.0.45:31569`, `10.0.0.46:31569`). `curl` from `usv` or Mac reached the nginx pods via `kube-proxy`.
- **Answer to Pod Distribution Question**: Both pods were on `usv` due to a control-plane taint on `mrgenjplinux`. You deleted pods to reschedule them, and since the taint was absent (confirmed by `taint not found` error), one pod landed on each node.

### Step 8: Set Up the Ticket Booth (MetalLB)

**Goal**: Provide a single entry point (`10.0.0.200`) for the audience.

**Questions**:

- How to combine node IPs into one address for clients (like a 4-node chassis website)?
- Why did MetalLB installation fail with 404 errors?
- What are IP address pool and L2Advertisement, and how to create `metallb-config.yaml`?

**Commands** (on `mrgenjplinux`):

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
cat <<EOF > metallb-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - 10.0.0.200-10.0.0.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default
EOF
kubectl apply -f metallb-config.yaml
kubectl delete service nginx
kubectl expose deployment nginx --port=80 --type=LoadBalancer
kubectl get services
curl http://10.0.0.200
```

**Commands** (on `usv`):

```bash
curl http://10.0.0.200
```

**Explanation**:

- **Answer to Unified IP Question**: MetalLB’s `LoadBalancer` service provided a single IP (`10.0.0.200`), routing traffic to nginx pods, mimicking a production website’s single entry point.
- **Answer to 404 Error**: Initial URLs were outdated; we used the correct `metallb-native.yaml` to deploy MetalLB’s controller and speaker.
- **Answer to IP Pool/L2Advertisement Question**: The IP pool (`10.0.0.200-10.0.0.250`) is a range MetalLB assigns to services. L2Advertisement uses ARP to announce the IP on your home network. You created `metallb-config.yaml` with `cat` to define this.
- Converted nginx to a `LoadBalancer` service, accessible via `http://10.0.0.200`.

## Final State

- **Cluster**: Two nodes (`mrgenjplinux` control plane, `usv` worker) running Kubernetes 1.30.5.
- **Application**: Two nginx pods, one per node, serving a website.
- **Access**: Single IP `10.0.0.200` via MetalLB, accessible from both nodes.
- **Networking**: Containerd for containers, Calico for pod networking, MetalLB for load balancing.

## Next Steps

- **Test from Mac**: `curl http://10.0.0.200` to verify external access.
- **Ingress**: Add an nginx-ingress controller for domain-based access (e.g., `http://nginx.local`).
- **Complex Apps**: Deploy a database with persistent storage.
- **Monitoring**: Add Prometheus for cluster health tracking.
