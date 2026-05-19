# 🚢 K8s Cluster on RedHat Distros (SelfHosted)

*A complete step-by-step guide to deploy a 3-node Kubernetes cluster.*

> **Important Notes Before Starting:**
> - ✅ Kubernetes v1.36.1
> - ✅ All commands run as root or with sudo
> - ✅ Ensure nodes can reach the internet for package downloads

---

## 📦 Technology Stack

| Component | Version | Purpose |
|-----------|---------|---------|
| 🧠 Kubernetes | v1.36.1 | Container orchestration platform |
| 🐳 Container Runtime | containerd v1.7.x | Low-level runtime to run containers |
| 🌐 Network (CNI) | Calico v3.32.0 | Pod networking & network policies |
| ⚖️ Load Balancer | MetalLB v0.15.3 | Bare-metal LoadBalancer services |
| 🚦 Ingress | NGINX Ingress v1.15.1 | HTTP/HTTPS routing & TLS termination |

---

## 🗂️ Cluster Specification

| Role | Hostname | IP Address | Requirements |
|------|----------|------------|-------------|
| 🎛️ Control Plane | k8s-master | master-ip | 2+ CPU, 2GB+ RAM |
| 👷 Worker Node | k8s-node1 | node1-ip | 2+ CPU, 2GB+ RAM |
| 👷 Worker Node | k8s-node2 | node2-ip | 2+ CPU, 2GB+ RAM |

> 💡 Replace master-ip, node1-ip, node2-ip with your actual IPs

---

## ✅ Pre-Installation Checklist (Run on All Nodes)

> **What this does:** Prepares the OS foundation for Kubernetes by disabling swap (required by kubelet), updating packages, and setting up time synchronization.  
> **Why install:** Kubernetes components expect a clean, synchronized, and swap-free environment to schedule pods reliably.  
> **If skipped:** Kubelet may fail to start, pods could be evicted unexpectedly, or time-sensitive operations (like TLS/certs) may break.

```bash
# Disable swap permanently (kubelet requires this)
sudo swapoff -a
sudo sed -i '/^\([^#].*swap.*\)/s/^/#/' /etc/fstab

# Update system packages to latest stable versions
sudo dnf update -y
sudo dnf install -y kernel-modules dnf-plugins-core chrony

# Enable and start chronyd for time synchronization
sudo systemctl enable --now chronyd
sudo systemctl start chronyd

# Set timezone to UTC for consistent cluster logging
sudo timedatectl set-timezone UTC
```

---

## 🖥️ 1. Hostname & Network Configuration (Run on All Nodes)

### 1.1 🏷️ Set Hostnames

> **What this does:** Assigns a unique, resolvable hostname to each node.  
> **Why install:** Kubernetes uses hostnames for node identity, certificate validation, and API communication.  
> **If skipped:** Nodes may fail to join the cluster, or certificates may not validate, causing authentication errors.

```bash
# Set hostname on control plane node
sudo hostnamectl set-hostname k8s-master
```
```
# Set hostname on first worker node
sudo hostnamectl set-hostname k8s-node1
```
```
# Set hostname on second worker node
sudo hostnamectl set-hostname k8s-node2
```
```
# Reload shell to apply hostname changes immediately
exec bash
```

### 1.2 🌐 Configure Static IP

> **What this does:** Assigns a fixed IP address to each node's network interface.  
> **Why install:** Kubernetes components communicate via stable IPs; dynamic IPs can break cluster connectivity.  
> **If skipped:** Nodes may lose connectivity after reboot, causing pod evictions or control plane unreachable errors.

```bash
# List available network interfaces to identify the correct one
nmcli device status

# Configure static IP, gateway, and DNS (replace placeholders)
sudo nmcli connection modify <interface-name> \
  ipv4.method manual \
  ipv4.addresses <your-node-ip>/24 \
  ipv4.gateway <your-gateway-ip> \
  ipv4.dns "8.8.8.8,1.1.1.1" \
  connection.autoconnect yes

# Restart network connection to apply new settings
sudo nmcli connection down <interface-name> && sudo nmcli connection up <interface-name>
```

### 1.3 📝 Update /etc/hosts (All Nodes)

> **What this does:** Creates local hostname-to-IP mappings for cluster nodes.  
> **Why install:** Ensures nodes can resolve each other by hostname without external DNS.  
> **If skipped:** kubeadm join may fail with "node not found" or certificate hostname mismatch errors.

```bash
# Append cluster node mappings to hosts file on every node
sudo tee -a /etc/hosts <<EOF

# Kubernetes Cluster Nodes
<master-ip>   k8s-master
<node1-ip>    k8s-node1
<node2-ip>    k8s-node2
EOF
```

### 1.4 🔄 Reboot All Nodes

> **What this does:** Applies all hostname, network, and kernel changes.  
> **Why install:** Ensures a clean state before installing Kubernetes components.  
> **If skipped:** Some settings (like kernel modules or hostname) may not take effect, causing subtle failures later.

```bash
# Reboot to apply all system changes
sudo reboot
```

---

## ⚙️ 2. System Preparation (Run on All Nodes)

### 2.1 🔌 Load Kernel Modules

> **What this does:** Loads `overlay` and `br_netfilter` kernel modules required for pod networking.  
> **Why install:** Kubernetes CNI plugins and kube-proxy rely on these for container isolation and iptables rules.  
> **If skipped:** Pods may not communicate, network policies won't work, or kube-proxy fails to program rules.

```bash
# Load bridge networking module immediately
sudo modprobe bridge

# Install extra kernel modules for container networking
sudo dnf install -y kernel-modules-extra

# Persist required kernel modules across reboots
sudo tee /etc/modules-load.d/k8s.conf > /dev/null <<EOF
overlay
br_netfilter
EOF

# Load modules now without reboot
sudo modprobe overlay
sudo modprobe br_netfilter
```

### 2.2 🎚️ Configure Sysctl Parameters

> **What this does:** Enables IP forwarding and bridge traffic inspection via iptables.  
> **Why install:** Required for pod-to-pod communication across nodes and proper network policy enforcement.  
> **If skipped:** Pods on different nodes cannot communicate; Calico network policies will not function.

```bash
# Write required sysctl settings for Kubernetes networking
sudo tee /etc/sysctl.d/k8s.conf > /dev/null <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl settings immediately and persist them
sudo sysctl --system
```

### 2.3 🛡️ Configure SELinux

> **What this does:** Installs SELinux policies that allow container runtimes to function under enforcing mode.  
> **Why install:** Maintains system security while permitting Kubernetes operations.  
> **If skipped:** Container processes may be blocked by SELinux, causing pod startup failures or permission denied errors.

```bash
# Install SELinux policies optimized for container workloads
sudo dnf install -y container-selinux

# Keep SELinux in enforcing mode for production security
# For testing only: temporarily set to permissive if troubleshooting
# sudo setenforce 0
# sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

### 2.4 🔥 Configure Firewall Rules

> **What this does:** Opens required ports for Kubernetes API, kubelet, and node services.  
> **Why install:** Ensures control plane and worker nodes can communicate securely.  
> **If skipped:** kubeadm init/join will timeout; nodes appear "NotReady"; services become unreachable.


#### Open ports required (All Nodes)
> **Port:10250** kubelet API: pod management, exec, logs, health checks.
> **Port:30000-32767** NodePort services: external access to services.
> **NAT masquerading:** enables pod egress to internet.
```bash

sudo firewall-cmd --permanent --add-port=10250/tcp       
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --permanent --add-masquerade           
```
```
# Reload firewall to apply all new rules
sudo firewall-cmd --reload
```

#### Open control-plane-specific ports (K8s-Master)
> **Port:6443** kube-apiserver: main Kubernetes API endpoint.
> **Port:2379-2380** etcd: cluster state storage (client and peer ports).
> **Port:10259** kube-scheduler: metrics and health endpoint.
> **Port:10257** kube-controller-manager: metrics and health endpoint.

```bash
sudo firewall-cmd --permanent --add-port=6443/tcp        
sudo firewall-cmd --permanent --add-port=2379-2380/tcp   
sudo firewall-cmd --permanent --add-port=10259/tcp       
sudo firewall-cmd --permanent --add-port=10257/tcp      
```
```
# Reload firewall to apply all new rules
sudo firewall-cmd --reload
```

---

## 🐳 3. Install containerd (Run on All Nodes)

> **What this does:** Installs and configures containerd, the CRI-compatible container runtime.  
> **Why install:** Kubernetes needs a runtime to pull and run container images; containerd is lightweight and CNCF-graduated.  
> **If skipped:** kubelet cannot start pods; you'll see "container runtime not ready" errors.

```bash
# Add Docker's official repo to access containerd packages
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

# Install containerd runtime package
sudo dnf install -y containerd.io

# Create config directory and generate default configuration
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# Enable systemd cgroup driver for kubelet compatibility
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Set pause image version matching Kubernetes requirements
sudo sed -i 's|sandbox_image = ".*"|sandbox_image = "registry.k8s.io/pause:3.10"|' /etc/containerd/config.toml

# Enable and start containerd service immediately
sudo systemctl enable --now containerd
```

---

## 🧱 4. Install Kubernetes Components (Run on All Nodes)

> **What this does:** Installs kubeadm, kubelet, and kubectl from the official Kubernetes RPM repo.  
> **Why install:** These are the core binaries to bootstrap, run, and manage the cluster.  
> **If skipped:** You cannot initialize or join nodes to a Kubernetes cluster.

```bash
# Add official Kubernetes v1.36 RPM repository with GPG verification
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

# Install core Kubernetes binaries, excluding auto-upgrades
sudo dnf install -y kubelet kubeadm kubectl cri-tools --disableexcludes=kubernetes

# Lock package versions to prevent accidental cluster-breaking upgrades
sudo dnf versionlock add kubelet kubeadm kubectl

# Enable kubelet to start on boot (it waits for kubeadm init)
sudo systemctl enable --now kubelet
```

---

## 🎛️ 5. Initialize Control Plane (Only on k8s-master)

> **What this does:** Bootstraps the Kubernetes control plane components (API server, scheduler, controller-manager).  
> **Why install:** Creates the cluster's brain that manages state, schedules pods, and exposes the API.  
> **If skipped:** No cluster exists; worker nodes have nothing to join.

```bash
# Pre-pull required control plane images to avoid timeout during init
sudo kubeadm config images pull

# Initialize cluster with Calico CIDR and advertise master IP
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<master-ip> \
  --control-plane-endpoint=<master-ip> \
  --upload-certs

# Setup kubectl config for current user to manage the cluster
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Generate and display join command for worker nodes
kubeadm token create --print-join-command
```

---

## 👷 6. Join Worker Nodes (On k8s-node1 & k8s-node2)

> **What this does:** Registers worker nodes with the control plane using a secure token.  
> **Why install:** Workers execute pods scheduled by the control plane; without them, no workloads run.  
> **If skipped:** Cluster has no compute capacity; pods remain pending.

```bash
# Run the join command generated on master (includes token and CA hash)
sudo kubeadm join <master-ip>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

## 🌐 7. Install Calico CNI (On Master)

> **What this does:** Installs Calico, a CNI plugin providing pod networking and network policies.  
> **Why install:** Kubernetes needs a CNI for pod-to-pod communication; Calico adds robust policy enforcement.  
> **If skipped:** Pods cannot communicate across nodes; network policies won't work; cluster stays "NotReady".

### Legacy Install Method:
```bash
# Apply Calico manifest for basic installation
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### Operator Install Method (Recommended):
```bash
# Prevent NetworkManager from managing Calico interfaces (RHEL-specific fix)
sudo mkdir -p /etc/NetworkManager/conf.d/
sudo tee /etc/NetworkManager/conf.d/calico.conf > /dev/null <<EOF
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico
EOF
sudo systemctl reload NetworkManager

# Install Calico operator for lifecycle management
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/tigera-operator.yaml

# Apply custom resources to activate Calico with default settings
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/custom-resources.yaml
```

---

## ⚖️ 8. Install MetalLB (On Master)

> **What this does:** Deploys MetalLB to provide LoadBalancer-type services on bare-metal clusters.  
> **Why install:** Cloud LoadBalancers aren't available on-prem; MetalLB assigns real IPs to services.  
> **If skipped:** Services of type LoadBalancer stay in "Pending" state with no external IP.

```bash
# Deploy MetalLB native manifest with controller and speaker pods
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml

# Wait for MetalLB controller to be ready before configuring
kubectl wait --namespace=metallb-system \
  --for=condition=ready pod \
  --selector=component=controller \
  --timeout=90s
```

### Configure IPAddressPool
```bash
# Create MetalLB configuration with IP pool and L2 advertisement
cat <<EOF > metallb-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - <your-external-ip-range>

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
EOF

# Apply the MetalLB configuration to activate IP assignment
kubectl apply -f metallb-config.yaml
```

> 💡 IP range must be in same L2 subnet as nodes, not in DHCP range, and not overlapping with node IPs. Example: `10.0.0.200-10.0.0.220`

---

## 🚦 9. Install NGINX Ingress Controller (On Master)

> **What this does:** Deploys NGINX Ingress to route external HTTP/HTTPS traffic to cluster services.  
> **Why install:** Provides a single entry point with TLS termination, path-based routing, and host rules.  
> **If skipped:** No easy way to expose web apps; you'd need NodePorts or manual reverse proxies.

```bash
# Deploy NGINX Ingress Controller for bare-metal environments
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml

# Wait for ingress controller pod to be ready before testing
kubectl wait --namespace=ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

---

## ✅ 10. Verification

> **What this does:** Confirms all components are running and the cluster is healthy.  
> **Why install:** Validates your deployment before deploying applications.  
> **If skipped:** You might deploy apps to a broken cluster and waste debugging time.

```bash
# Verify all nodes are in Ready status
kubectl get nodes

# Check that system pods (kube-system, calico, metallb) are running
kubectl get pods --all-namespaces

# Confirm ingress-nginx service received an external IP from MetalLB
kubectl get svc -n ingress-nginx
```

---

## 🔄 Cluster Reset / Clean State

### Reset Worker Nodes (k8s-node1 & k8s-node2)

> **What this does:** Removes Kubernetes state, network config, and runtime data to allow clean re-install.  
> **Why install:** Essential for testing, re-provisioning, or recovering from misconfiguration.  
> **If skipped:** Subsequent kubeadm init/join may fail with stale certificate or network errors.

```bash
# Reset kubeadm state and remove cluster configuration
sudo kubeadm reset -f

# Remove CNI network configuration files
sudo rm -rf /etc/cni/net.d

# Remove kubelet data and registered pods
sudo rm -rf /var/lib/kubelet

# Remove etcd data (only relevant if node was control plane)
sudo rm -rf /var/lib/etcd

# Flush all iptables rules to clear stale network policies
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X

# Restart container runtime to clear container state
sudo systemctl restart containerd

# Restart kubelet to pick up clean configuration
sudo systemctl restart kubelet

# Remove user kubectl config
sudo rm -rf $HOME/.kube
```

### Reset Master Node (k8s-master)

> **What this does:** Fully wipes control plane state including etcd, certs, and manifests.  
> **Why install:** Required when rebuilding the cluster from scratch or fixing critical control plane issues.  
> **If skipped:** New cluster may inherit old certificates, tokens, or etcd corruption.

```bash
# Reset kubeadm and remove all control plane manifests
sudo kubeadm reset -f

# Remove Kubernetes configuration directory
sudo rm -rf /etc/kubernetes

# Remove kubelet working directory and pod state
sudo rm -rf /var/lib/kubelet

# Remove etcd data directory (critical for clean state)
sudo rm -rf /var/lib/etcd

# Remove CNI configuration to avoid network conflicts
sudo rm -rf /etc/cni/net.d

# Flush all firewall and NAT rules
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X

# Restart container runtime and kubelet services
sudo systemctl restart containerd
sudo systemctl restart kubelet

# Remove local kubectl configuration
sudo rm -rf $HOME/.kube
```

### Reboot All Nodes
```bash
# Final reboot to ensure clean kernel and network state
sudo reboot
```
---

## 📚 Official Documentation

- 🧠 Kubernetes: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/
- 🌐 Calico: https://docs.tigera.io/calico/latest/
- ⚖️ MetalLB: https://metallb.universe.tf/
- 🚦 NGINX Ingress: https://kubernetes.github.io/ingress-nginx/

---

🎉 **Your Kubernetes cluster is now ready for deployment!**  
Run `kubectl get nodes` to confirm all nodes are Ready, then start deploying your applications. 🚀
