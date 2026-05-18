# K8s Cluster on RedHat Distros (SelfHosted)

A complete step-by-step guide to deploy a 3-node Kubernetes cluster.

**Important Notes Before Starting:**
- This guide is tested on RHEL 8.4+ / Rocky Linux 8+ / AlmaLinux 8+
- Kubernetes v1.36.1 (latest patch as of May 2026)
- All commands run as root or with sudo
- Ensure nodes can reach the internet for package downloads

---

## Technology Stack

| Component | Version | Purpose |
|-----------|---------|---------|
| Kubernetes | v1.36.1 | Container orchestration |
| Container Runtime | containerd v1.7.x | Run containers |
| Network (CNI) | Calico v3.32.0 | Pod networking & policies |
| Load Balancer | MetalLB v0.15.3 | Bare-metal LoadBalancer services |
| Ingress | NGINX Ingress v1.15.1 | HTTP/HTTPS routing |

---

## Cluster Specification

| Role | Hostname | IP Address | Requirements |
|------|----------|------------|-------------|
| Control Plane | k8s-master | <master-ip> | 2+ CPU, 2GB+ RAM |
| Worker Node | k8s-node1 | <node1-ip> | 2+ CPU, 2GB+ RAM |
| Worker Node | k8s-node2 | <node2-ip> | 2+ CPU, 2GB+ RAM |

> Replace <master-ip>, <node1-ip>, <node2-ip> with your actual IPs

---

## Pre-Installation Checklist (Run on All Nodes)

```bash
# Disable swap (required by kubelet)
sudo swapoff -a
sudo sed -i '/^\([^#].*swap.*\)/s/^/#/' /etc/fstab

# Update system
sudo dnf update -y
sudo dnf install -y kernel-modules dnf-plugins-core chrony
sudo systemctl enable --now chronyd
sudo systemctl start chronyd
sudo timedatectl set-timezone UTC
```

---

## 1. Hostname & Network Configuration (Run on All Nodes)

### 1.1 Set Hostnames
```bash
# On k8s-master:
sudo hostnamectl set-hostname k8s-master

# On k8s-node1:
sudo hostnamectl set-hostname k8s-node1

# On k8s-node2:
sudo hostnamectl set-hostname k8s-node2

# Apply changes to current shell
exec bash
```

### 1.2 Configure Static IP
```bash
# Identify network interface
nmcli device status

# Configure static IP (replace values)
sudo nmcli connection modify <interface-name> \
  ipv4.method manual \
  ipv4.addresses <your-node-ip>/24 \
  ipv4.gateway <your-gateway-ip> \
  ipv4.dns "8.8.8.8,1.1.1.1" \
  connection.autoconnect yes

# Restart network
sudo nmcli connection down <interface-name> && sudo nmcli connection up <interface-name>
```

### 1.3 Update /etc/hosts (All Nodes)
```bash
sudo tee -a /etc/hosts <<EOF

# Kubernetes Cluster Nodes
<master-ip>   k8s-master
<node1-ip>    k8s-node1
<node2-ip>    k8s-node2
EOF
```

### 1.4 Reboot All Nodes
```bash
sudo reboot
```

---

## 2. System Preparation (Run on All Nodes)

### 2.1 Load Kernel Modules
```bash
sudo modprobe bridge
sudo dnf install -y kernel-modules-extra

sudo tee /etc/modules-load.d/k8s.conf > /dev/null <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### 2.2 Configure Sysctl Parameters
```bash
sudo tee /etc/sysctl.d/k8s.conf > /dev/null <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 2.3 Configure SELinux
```bash
# Install SELinux policies for containers
sudo dnf install -y container-selinux

# Keep SELinux enforcing (recommended for production)
# If testing only and issues arise, temporarily set to permissive:
# sudo setenforce 0
# sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

### 2.4 Configure Firewall Rules
```bash
# Rules for all nodes (Master + Workers)
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --permanent --add-masquerade

# Rules for control plane node only (k8s-master)
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10259/tcp
sudo firewall-cmd --permanent --add-port=10257/tcp

# Apply all rules
sudo firewall-cmd --reload
```

---

## 3. Install containerd (Run on All Nodes)

```bash
# Add Docker repository and install containerd
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y containerd.io

# Create config directory and generate default config
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# Enable systemd cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Set pause image for Kubernetes
sudo sed -i 's|sandbox_image = ".*"|sandbox_image = "registry.k8s.io/pause:3.10"|' /etc/containerd/config.toml

# Enable and start containerd
sudo systemctl enable --now containerd
```

---

## 4. Install Kubernetes Components (Run on All Nodes)

```bash
# Add Kubernetes repository
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

# Install Kubernetes tools
sudo dnf install -y kubelet kubeadm kubectl cri-tools --disableexcludes=kubernetes

# Prevent accidental upgrades
sudo dnf versionlock add kubelet kubeadm kubectl

# Enable kubelet
sudo systemctl enable --now kubelet
```

---

## 5. Initialize Control Plane (Only on k8s-master)

```bash
# Pull required images
sudo kubeadm config images pull

# Initialize cluster
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<master-ip> \
  --control-plane-endpoint=<master-ip> \
  --upload-certs

# Configure kubectl for current user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Save join command for worker nodes
kubeadm token create --print-join-command
```

---

## 6. Join Worker Nodes (On k8s-node1 & k8s-node2)

```bash
# On each worker node, run the join command from master:
sudo kubeadm join <master-ip>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

## 7. Install Calico CNI (On Master)

Legacy Install Method:
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```
Operator Install Method:
```bash
# Prevent NetworkManager conflicts (RHEL-specific)
sudo mkdir -p /etc/NetworkManager/conf.d/
sudo tee /etc/NetworkManager/conf.d/calico.conf > /dev/null <<EOF
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico
EOF
sudo systemctl reload NetworkManager

# Install Calico
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/custom-resources.yaml
```

---

## 8. Install MetalLB (On Master)

```bash
# Install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml

# Wait for controller to be ready
kubectl wait --namespace=metallb-system \
  --for=condition=ready pod \
  --selector=component=controller \
  --timeout=90s
```

### Configure IPAddressPool
```bash
# Create metallb-config.yaml
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

# Apply configuration
kubectl apply -f metallb-config.yaml
```

> Note: IP range must be in same L2 subnet as nodes, not in DHCP range, and not overlapping with node IPs. Example: 10.0.0.200-10.0.0.220

---

## 9. Install NGINX Ingress Controller (On Master)

```bash
# Install Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml

# Wait for controller to be ready
kubectl wait --namespace=ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

---

## 10. Verification

```bash
# Check node status
kubectl get nodes

# Check system pods
kubectl get pods --all-namespaces

# Check ingress service for external IP
kubectl get svc -n ingress-nginx
```

---

## Cluster Reset / Clean State

### Reset Worker Nodes (k8s-node1 & k8s-node2)
```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/kubelet
sudo rm -rf /var/lib/etcd
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
sudo systemctl restart containerd
sudo systemctl restart kubelet
sudo rm -rf $HOME/.kube
```

### Reset Master Node (k8s-master)
```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes
sudo rm -rf /var/lib/kubelet
sudo rm -rf /var/lib/etcd
sudo rm -rf /etc/cni/net.d
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
sudo systemctl restart containerd
sudo systemctl restart kubelet
sudo rm -rf $HOME/.kube
```

### Reboot All Nodes
```bash
sudo reboot
```

---

## Security Hardening (Optional)

### Enable SELinux Enforcement
```bash
sudo setenforce 1
sudo sed -i 's/^SELINUX=permissive$/SELINUX=enforcing/' /etc/selinux/config
```

### Example Network Policy with Calico
```bash
kubectl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  selector: all()
  types:
  - Ingress
  ingress: []
EOF
```

### Enable Audit Logging
```bash
# Create audit policy
sudo tee /etc/kubernetes/audit-policy.yaml > /dev/null <<EOF
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
EOF

# Edit kube-apiserver manifest: /etc/kubernetes/manifests/kube-apiserver.yaml
# Add to command array:
# - --audit-log-path=/var/log/kubernetes/audit.log
# - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
# - --audit-log-maxage=30
# - --audit-log-maxbackup=10
# - --audit-log-maxsize=100
```

---

## Official Documentation

- Kubernetes: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/
- Calico: https://docs.tigera.io/calico/latest/
- MetalLB: https://metallb.universe.tf/
- NGINX Ingress: https://kubernetes.github.io/ingress-nginx/

---

Your Kubernetes cluster is now ready for deployment.
