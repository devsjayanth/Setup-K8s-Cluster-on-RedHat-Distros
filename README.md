# 🚢3-Node Kubernetes Cluster on RedHat Family 🚀

A complete step-by-step guide to deploy a 3-node Kubernetes cluster.

---

## 🌟 Cluster Specification

| Role          | Hostname      | IP Address          |
|---------------|---------------|---------------------|
| Control Plane | k8s-master    | `<master-ip>`       |
| Worker Node   | k8s-node1     | `<node1-ip>`        |
| Worker Node   | k8s-node2     | `<node2-ip>`        |

**Technology Stack (Latest Stable - May 2026)**

- **Kubernetes**: v1.36
- **Container Runtime**: containerd
- **CNI**: Calico v3.32.0
- **Load Balancer**: MetalLB v0.15.3
- **Ingress**: NGINX Ingress Controller v1.15.1

---

## 1. 🔧 Hostname & Network Configuration (Run on All Nodes)

### 1.1 Set Hostnames
```bash
sudo hostnamectl set-hostname k8s-master
sudo hostnamectl set-hostname k8s-node1
sudo hostnamectl set-hostname k8s-node2

exec bash
```

### 1.2 Configure Static IP
```bash
nmcli device status

sudo nmcli connection modify <interface-name> \
  ipv4.method manual \
  ipv4.addresses <your-node-ip>/24 \
  ipv4.gateway <your-gateway-ip> \
  ipv4.dns "8.8.8.8,1.1.1.1" \
  connection.autoconnect yes

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

**Reboot all nodes**:
```bash
sudo reboot
```

---

## 2. 🛠 System Preparation (Run on All Nodes)

**🔸 What is this step?**  
Basic OS-level setup and tuning.

**❓ Why do we do it?**  
Kubernetes has strict requirements on the operating system.

**🎯 Purpose:**  
Prepares the system for stable Kubernetes operation.

**⚠️ What happens if skipped?**  
Cluster initialization may fail or pods may not communicate properly.

```bash
sudo dnf update -y

# Disable Swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl settings
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# SELinux permissive
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

**Firewall (Recommended)**
```bash
sudo firewall-cmd --permanent --add-port={6443,2379-2380,10250,10259,10257}/tcp
sudo firewall-cmd --reload
```

---

## 3. 📦 Install containerd (Run on All Nodes)

**🔸 What is containerd?**  
A lightweight and reliable container runtime.

**❓ Why do we install it?**  
Kubernetes needs a runtime to create and manage containers.

**🎯 Purpose:**  
Runs your application containers (pods).

**⚠️ What happens if not installed?**  
`kubeadm init` will fail — you cannot create the cluster.

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y containerd.io

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl enable --now containerd
```

---

## 4. ☸️ Install Kubernetes Components (Run on All Nodes)

**🔸 What are these tools?**  
kubeadm, kubelet, and kubectl.

**❓ Why install them?**  
They are the official tools to build and manage Kubernetes.

**🎯 Purpose:**  
- `kubeadm`: Bootstrap the cluster  
- `kubelet`: Runs on every node  
- `kubectl`: Command-line interface

**⚠️ What happens if skipped?**  
You cannot create or interact with the cluster.

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

---

## 5. 🚀 Initialize Control Plane (Only on k8s-master)

```bash
sudo kubeadm config images pull

sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<master-ip> \
  --control-plane-endpoint=k8s-master
```

**Configure kubeconfig**
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> Save the `kubeadm join` command.

---

## 6. 🔗 Join Worker Nodes (On k8s-node1 & k8s-node2)

Run the `kubeadm join ...` command (from master) on both worker nodes.

---

## 7. 🌐 Install Calico CNI (On Master)

**🔸 What is Calico?**  
A popular networking plugin (CNI).

**❓ Why install it?**  
Kubernetes needs networking between pods.

**🎯 Purpose:**  
Enables pod-to-pod communication and network policies.

**⚠️ What happens if skipped?**  
Pods cannot talk to each other → cluster is broken.

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/custom-resources.yaml

watch kubectl get tigerastatus
```

---

## 8. ⚖️ Install MetalLB (On Master)

**🔸 What is MetalLB?**  
Bare-metal load balancer for Kubernetes.

**❓ Why install it?**  
To get external IPs for `LoadBalancer` services on-prem.

**🎯 Purpose:**  
Assigns real external IPs to services.

**⚠️ What happens if skipped?**  
LoadBalancer services stay in `Pending` state.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml
```

**Create `metallb-config.yaml`** (edit IP range first):
```yaml
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
```

```bash
kubectl apply -f metallb-config.yaml
```

---

## 9. 🚪 Install NGINX Ingress Controller (On Master)

**🔸 What is it?**  
NGINX-based Ingress Controller.

**❓ Why install it?**  
To expose multiple applications via HTTP/HTTPS.

**🎯 Purpose:**  
Handles routing based on domain name and URL path.

**⚠️ What happens if skipped?**  
You cannot use `Ingress` resources for clean external access.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml
```

---

## 10. ✅ Verification & Testing

```bash
kubectl get nodes
kubectl get pods --all-namespaces
kubectl get svc -n ingress-nginx
```

---

## 🧹 Cluster Reset / Clean State (If Something Goes Wrong)

**This guide starts from a clean state.**  
If you mess up any step, use the following commands to reset everything:

### Full Cluster Reset

**On Worker Nodes (node1 & node2) first:**
```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/kubelet
sudo rm -rf /var/lib/etcd
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
sudo systemctl restart containerd
```

**On Master Node:**
```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes
sudo rm -rf /var/lib/kubelet
sudo rm -rf /var/lib/etcd
sudo rm -rf /etc/cni/net.d
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
sudo systemctl restart containerd
```

**Then delete leftover files on all nodes:**
```bash
sudo rm -rf $HOME/.kube
```

After reset, you can start again from **Step 1**.

---

## 📚 Official Documentation

- [Kubernetes + kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [Calico](https://docs.tigera.io/calico/latest/)
- [MetalLB](https://metallb.universe.tf/)
- [NGINX Ingress](https://github.com/kubernetes/ingress-nginx)

---

**🎉 Congratulations! Your Kubernetes cluster is now ready.**
