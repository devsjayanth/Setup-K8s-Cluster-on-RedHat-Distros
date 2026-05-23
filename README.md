# 🚢 Kubernetes Cluster Setup on [ RHEL10/Redhat10 ]
### 🐧 RHEL 10 · Rocky Linux 10 · AlmaLinux 10
### 🕸️ Calico (NFT) · ⚖️ MetalLB · 🚦 F5 NGINX Ingress Controller

This is the **final, fully documented Standard Operating Procedure (SOP)**. It includes plain-English explanations of *why* we are doing each step, inline bash comments, and every battle-tested fix required for RHEL 10's unique security and networking landscape.

---

## 🧠 Architecture & Component Overview

Before typing commands, it helps to understand how traffic flows from the outside world down into your containers.

### 🔄 Traffic Flow Architecture

```text
🌐 External Client (Internet / Local LAN)
      │
      ▼
⚖️ MetalLB (Intercepts ARP, assigns External IP)
      │
      ▼
🚦 NGINX Ingress (Reads HTTP Host Header, routes to Service)
      │
      ▼
🕸️ Calico CNI (Routes traffic across VXLAN Overlay to Pod IP)
      │
      ▼
🚀 App Pods (🐳 containerd on 👷 Worker Nodes)
```

### 📋 Component Glossary

| Component | What it does |
| :--- | :--- |
| ☸️ **Kubernetes (K8s)** | The "operating system" for your cluster. It schedules containers, heals them if they crash, and scales them. |
| 🐳 **Containerd** | The actual engine that downloads images and runs the containers on the Linux kernel. |
| 🕸️ **Calico (CNI)** | The "virtual network switch". It gives every pod its own IP address and creates encrypted tunnels (VXLAN) so pods on different physical servers can talk to each other. |
| ⚖️ **MetalLB** | The "bare-metal cloud provider". Clouds give you Load Balancers automatically; bare-metal doesn't. MetalLB tricks your local network router into thinking your K8s cluster is a physical device by answering ARP requests. |
| 🚦 **NGINX Ingress** | The "smart HTTP router". While MetalLB operates at Layer 4 (TCP/IP), NGINX operates at Layer 7 (HTTP). It looks at the URL (e.g., `app.local`) and routes traffic to the correct internal pod. |

### 🌐 Network Topology

| Setting | Value | Why? |
| :--- | :--- | :--- |
| **Pod CIDR** | `192.168.0.0/16` | The private IP pool Calico will draw from to assign IPs to containers. |
| **Service CIDR** | `10.96.0.0/12` | The internal virtual IPs K8s uses for ClusterIP services (like DNS). |
| **MetalLB Pool** | `10.0.0.201 – 220` | Free IPs on your *physical* LAN that MetalLB is allowed to hand out. |
| **Ingress IP** | `10.0.0.200` | A dedicated, pinned LAN IP exclusively for the NGINX Ingress Controller. |

---

## 🛠️ Step 1 — OS Preparation `[ALL Nodes]`
*Run these steps on **all 3 nodes** (Master and Workers).*

### 1.1 🏷️ Local DNS Resolution
> 🧠 **What this does:** K8s nodes must be able to find each other by name. Since we don't have a local DNS server configured for these static IPs, we hardcode them into the local `/etc/hosts` file.
Set a unique hostname for the machine you are currently on
```bash
sudo hostnamectl set-hostname k8s-master
```
```
sudo hostnamectl set-hostname k8s-node1
```
```
sudo hostnamectl set-hostname k8s-node2
```
Inject the cluster roster into the local hosts file
```
sudo tee /etc/hosts <<'EOF'
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

10.0.0.150  k8s-master
10.0.0.151  k8s-node1
10.0.0.152  k8s-node2
EOF
```

### 1.2 🚫 Disable Swap
> 🧠 **What this does:** K8s guarantees resource limits (QoS). If Linux moves container memory to a slow swap disk, those guarantees break, and the cluster becomes unstable. Kubelet will flat-out refuse to start if swap is on. RHEL 10 uses `zram` (compressed RAM swap) by default, which must also be removed.

```bash
# Turn off swap immediately in memory
sudo swapoff -a
# Comment out swap entries in fstab so it stays off after reboot
sudo sed -i '/\bswap\b/s/^/#/' /etc/fstab
# Disable and remove RHEL 10's zram swap generator
sudo systemctl disable --now swap.target 2>/dev/null || true
sudo dnf remove -y zram-generator-defaults 2>/dev/null || true
```

### 1.3 📦 Install OS Dependencies & Kernel Modules
> 🧠 **What this does:** Installs the tools K8s needs. Crucially, in RHEL 10, the `br_netfilter` module (which allows iptables/nftables to see bridged container traffic) was moved out of the base kernel into `kernel-modules-extra`.

```bash
sudo dnf install -y \
  curl wget git vim bash-completion \
  iproute iproute-tc ipvsadm ipset socat conntrack-tools \
  kernel-modules-extra nftables \
  setools-console policycoreutils-python-utils \
  setroubleshoot-server audit chrony yum-utils
```

### 1.4 ⚠️ MANDATORY REBOOT
> 🧠 **What this does:** Aligns your running kernel with the newly installed `kernel-modules-extra`. If you skip this, `modprobe br_netfilter` will fail, and Calico will never start.

```bash
sudo reboot
```
*(⏳ Log back into all 3 nodes after they reboot)*

### 1.5 🧱 Firewall & Webhook Trust Zones
> 🧠 **What this does:** RHEL's `firewalld` blocks traffic crossing between the physical NIC and virtual CNI bridges. Furthermore, K8s "Webhooks" (used by MetalLB and NGINX to validate configs) require the API server to talk to internal Pod IPs. Adding the Pod and Service CIDRs to the `trusted` zone permanently prevents "no route to host" and "connection refused" webhook errors.

```bash
NODE_HOSTNAME=$(hostname)
sudo systemctl enable --now firewalld

# --- 🤝 TRUST CLUSTER NETWORKS (Fixes Webhooks & API Routing) ---
sudo firewall-cmd --zone=trusted --add-source=10.0.0.150/32 --permanent
sudo firewall-cmd --zone=trusted --add-source=10.0.0.151/32 --permanent
sudo firewall-cmd --zone=trusted --add-source=10.0.0.152/32 --permanent
sudo firewall-cmd --zone=trusted --add-source=192.168.0.0/16 --permanent # Calico Pod CIDR
sudo firewall-cmd --zone=trusted --add-source=10.96.0.0/12 --permanent   # K8s Service CIDR

# --- 🔓 COMMON PORTS (All Nodes) ---
sudo firewall-cmd --permanent --add-service=ntp          # Time sync
sudo firewall-cmd --permanent --add-port=10250/tcp       # Kubelet API
sudo firewall-cmd --permanent --add-port=179/tcp         # Calico BGP
sudo firewall-cmd --permanent --add-port=4789/udp        # Calico VXLAN overlay
sudo firewall-cmd --permanent --add-port=5473/tcp        # Calico Typha
sudo firewall-cmd --permanent --add-port=9099/tcp        # Calico Felix Health
sudo firewall-cmd --permanent --add-port=7946/tcp        # MetalLB Gossip
sudo firewall-cmd --permanent --add-port=7946/udp        # MetalLB Gossip

# --- 🧠 CONTROL PLANE PORTS (Master Only) ---
if [[ "$NODE_HOSTNAME" == *"master"* ]]; then
  sudo firewall-cmd --permanent --add-port=6443/tcp      # K8s API Server
  sudo firewall-cmd --permanent --add-port=2379-2380/tcp # etcd database
  sudo firewall-cmd --permanent --add-port=10257/tcp     # controller-manager
  sudo firewall-cmd --permanent --add-port=10259/tcp     # scheduler
fi

# --- 👷 WORKER PORTS (Workers Only) ---
if [[ "$NODE_HOSTNAME" == *"node"* ]]; then
  sudo firewall-cmd --permanent --add-port=30000-32767/tcp # NodePort TCP
  sudo firewall-cmd --permanent --add-port=30000-32767/udp # NodePort UDP
  sudo firewall-cmd --permanent --add-service=http         # Ingress HTTP
  sudo firewall-cmd --permanent --add-service=https        # Ingress HTTPS
  sudo firewall-cmd --permanent --add-port=8443/tcp        # NGINX Webhook
  sudo firewall-cmd --permanent --add-port=9113/tcp        # NGINX Metrics
fi

# Apply all firewall changes to the running kernel
sudo firewall-cmd --reload
```

### 1.6 ⏱️ Sync System Clock (NTP)
> 🧠 **What this does:** K8s uses TLS certificates for all internal communication. If clocks drift by more than a few minutes between nodes, certificates are rejected as "expired" or "not yet valid", and the cluster breaks.

```bash
sudo systemctl enable --now chronyd
sudo chronyc makestep     # Force immediate time correction
sudo timedatectl set-ntp true
```

### 1.7 🧩 Load Kernel Modules
> 🧠 **What this does:** 
> * `overlay`: The filesystem driver that allows containers to have layered, lightweight filesystems.
> * `br_netfilter`: Allows the host's firewall to inspect traffic passing between containers on a virtual bridge.
> * `ip_vs` / `nf_conntrack`: High-performance kernel load-balancing and connection tracking.

```bash
sudo modprobe overlay br_netfilter ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack

# Persist modules across reboots
sudo tee /etc/modules-load.d/k8s.conf <<'EOF'
overlay
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF
```

### 1.8 ⚙️ Configure Kernel Parameters (sysctl)
> 🧠 **What this does:** 
> * `ip_forward`: Allows the Linux kernel to act as a router (required for pod traffic).
> * `rp_filter`: Reverse Path filtering prevents IP spoofing, but it *breaks* VXLAN overlay networks because the source IP of encapsulated packets looks "asymmetric". We must set it to 0.

```bash
# Replace ens160 with your actual NIC name if different
sudo tee /etc/sysctl.d/k8s.conf <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter         = 0
net.ipv4.conf.default.rp_filter     = 0
net.ipv4.conf.ens160.rp_filter      = 0
net.netfilter.nf_conntrack_max = 524288
fs.inotify.max_user_instances = 8192
fs.inotify.max_user_watches   = 524288
EOF

sudo sysctl --system
sudo sysctl -w net.ipv4.conf.ens160.rp_filter=0
```

### 1.9 🛡️ Configure SELinux
> 🧠 **What this does:** We **never** disable SELinux. Instead, we install the `container-selinux` policies and flip specific "booleans" that grant the container runtime permission to do necessary host-level tasks (like managing cgroups and proxying network traffic).

```bash
sudo dnf install -y container-selinux
sudo semodule -B

sudo setsebool -P container_manage_cgroup 1
sudo setsebool -P container_use_devices 1
sudo setsebool -P nis_enabled 1
sudo setsebool -P httpd_can_network_connect 1
sudo setsebool -P httpd_can_network_relay 1
```

### 1.10 🐳 Install Containerd
> 🧠 **What this does:** Installs the container runtime. We force `SystemdCgroup = true` so containerd and the Kubelet use the same resource management hierarchy (preventing CPU/Memory fighting). We also pin the `pause` image, which K8s uses to hold the network namespace open for pods.

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y containerd.io

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo sed -i 's|sandbox_image = "registry.k8s.io/pause:3.[0-9]*"|sandbox_image = "registry.k8s.io/pause:3.10"|' /etc/containerd/config.toml

sudo systemctl enable --now containerd
```

### 1.11 📥 Install K8s Binaries
> 🧠 **What this does:** 
> * `kubeadm`: The bootstrap tool (used once to build the cluster).
> * `kubelet`: The node agent (runs constantly, ensures pods are alive).
> * `kubectl`: The CLI tool for you to talk to the cluster.

```bash
sudo tee /etc/yum.repos.d/kubernetes.repo <<'EOF'
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable kubelet
```

---

## 🏗️ Step 2 — Initialise the Cluster `[CP]`
*Run ONLY on `k8s-master`.*

### 2.1 📝 Create kubeadm Config File
> 🧠 **What this does:** Defines the cluster blueprint. 
> **⚠️ Crucial Fix:** We set `kube-proxy` to `mode: "iptables"`. In RHEL 10, this transparently uses `iptables-nft` under the hood. This prevents container privilege crashes and ensures internal ClusterIP routing (like Webhooks) works perfectly.

```bash
sudo mkdir -p /etc/kubernetes
sudo tee /etc/kubernetes/kubeadm-config.yaml <<'EOF'
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
  name: k8s-master
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.32.0
controlPlaneEndpoint: "k8s-master:6443"
networking:
  podSubnet: "192.168.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  dnsDomain: cluster.local
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "iptables"
EOF
```

### 2.2 🚀 Bootstrap the Control Plane
```bash
# Initialize the master node and save the output to a log
sudo kubeadm init --config /etc/kubernetes/kubeadm-config.yaml --upload-certs | tee /root/kubeadm-init.log

# Configure kubectl for your local user (DO NOT use sudo for kubectl)
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
*⚠️ **Copy the `kubeadm join` command printed at the bottom of the output.***

### 2.3 🤝 Join Worker Nodes `[W]`
*Run the saved `kubeadm join` command on `k8s-node1` and `k8s-node2`.*
*(💡 Note: If a worker gets an "empty config.yaml" crash loop, run `sudo kubeadm reset -f`, `sudo rm -rf /var/lib/kubelet`, and try the join command again).*

---

## 🕸️ Step 3 — Install Calico CNI `[CP]`
*Run ONLY on `k8s-master`.*

> 🧠 **What this does:** Deploys the Tigera Operator, which acts as a "manager" to install and maintain Calico. We explicitly wait for the Custom Resource Definitions (CRDs) to register before applying the config, avoiding a common race condition.

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/tigera-operator.yaml

# ⏳ Wait for API server to recognize Calico resource types
kubectl wait --for condition=established --timeout=120s crd/installations.operator.tigera.io
kubectl wait --for condition=established --timeout=120s crd/apiservers.operator.tigera.io

# Tell the Operator to install Calico using the NFT dataplane and VXLAN
kubectl apply -f - <<'EOF'
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    linuxDataplane: Nftables
    ipPools:
      - name: default-ipv4-ippool
        blockSize: 26
        cidr: 192.168.0.0/16
        encapsulation: VXLANCrossSubnet
        natOutgoing: Enabled
        nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF
```

### 3.2 🛠️ Install and Configure `calicoctl`
> 🧠 **What this does:** Installs the Calico CLI. Notice the `<<EOF` (without quotes) on the config file generation—this allows `$(whoami)` to dynamically inject your current user's home directory, ensuring `sudo calicoctl` works without permission errors.

```bash
curl -sLO https://github.com/projectcalico/calico/releases/download/v3.32.0/calicoctl-linux-amd64
sudo install -o root -g root -m 0755 calicoctl-linux-amd64 /usr/local/bin/calicoctl
rm -f calicoctl-linux-amd64

sudo mkdir -p /etc/calico
sudo tee /etc/calico/calicoctl.cfg <<EOF
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: kubernetes
  kubeconfig: /home/$(whoami)/.kube/config
EOF

# Disable BGP Full Mesh (we are using VXLAN overlay, not physical BGP routers)
# Configure Felix (the node agent) for optimal logging and health checks
sudo /usr/local/bin/calicoctl apply -f - <<'EOF'
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: false
  asNumber: 64512
---
apiVersion: projectcalico.org/v3
kind: FelixConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Warning
  healthEnabled: true
  healthPort: 9099
  defaultEndpointToHostAction: Accept
  bpfEnabled: false
  wireguardEnabled: false
EOF
```

---

## ⚖️ Step 4 — Install MetalLB `[CP]`

> 🧠 **What this does:** Bare-metal servers don't have AWS/GCP Load Balancers. MetalLB solves this by listening for ARP requests on your local network ("Who has 10.0.0.200?") and answering them, effectively pulling external traffic into the K8s cluster.

```bash
# Install MetalLB in native Layer 2 mode
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.16.0/config/manifests/metallb-native.yaml
kubectl wait --namespace metallb-system --for=condition=Ready pod --selector=app=metallb --timeout=180s

# Define the IP pools and advertise them via Layer 2 ARP
kubectl apply -f - <<'EOF'
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: primary-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.0.0.201-10.0.0.220
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ingress-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.0.0.200/32
  autoAssign: false
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: primary-l2
  namespace: metallb-system
spec:
  ipAddressPools: [primary-pool]
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: ingress-l2
  namespace: metallb-system
spec:
  ipAddressPools: [ingress-pool]
EOF
```

---

## 🚦 Step 5 — Install NGINX Ingress Controller `[CP]`

> 🧠 **What this does:** While MetalLB gets traffic *into* the cluster, it doesn't know HTTP URLs. NGINX Ingress looks at the `Host: app.local` header and routes traffic to the correct internal Service. We use a `find` command to guarantee all F5 Custom Resource Definitions (VirtualServer, Policy) are applied, preventing startup crash loops.

```bash
git clone https://github.com/nginx/kubernetes-ingress.git --branch v5.2.1 --depth 1
cd kubernetes-ingress

# Apply RBAC, ConfigMaps, and IngressClass
kubectl apply -f deployments/common/ns-and-sa.yaml
kubectl apply -f deployments/rbac/rbac.yaml
kubectl apply -f deployments/common/nginx-config.yaml
kubectl apply -f deployments/common/ingress-class.yaml

# 🔍 Bulletproof CRD installation (finds all YAMLs in the crds folder)
find deployments/common/crds -type f -name "*.yaml" -exec kubectl apply -f {} \;

# Deploy the controller and wait for it to be ready
kubectl apply -f deployments/deployment/nginx-ingress.yaml
kubectl rollout status deployment nginx-ingress -n nginx-ingress --timeout=180s

# Expose NGINX to the physical network via MetalLB (Pinned to 10.0.0.200)
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
  namespace: nginx-ingress
  annotations:
    metallb.universe.tf/address-pool: ingress-pool
    metallb.universe.tf/loadBalancerIPs: 10.0.0.200
spec:
  type: LoadBalancer
  selector:
    app: nginx-ingress
  ports:
    - name: http
      port: 80
      targetPort: 80
    - name: https
      port: 443
      targetPort: 443
EOF

# Make NGINX the default ingress class so you don't have to specify it on every rule
kubectl annotate ingressclass nginx ingressclass.kubernetes.io/is-default-class="true"
```

---

## 🧪 Step 6 — The Flawless End-to-End Test `[CP]`

> 🧠 **What this does:** Deploys a tiny web server, creates a K8s Service for it, and creates an Ingress Rule telling NGINX to route `test.local` to it. This validates the entire chain: **User ➡️ ⚖️ MetalLB (L2) ➡️ 🚦 NGINX Pod (L7) ➡️ App Pod**.

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: hashicorp/http-echo:0.2.3
          args:
            - "-text=Hello from AlmaLinux 10 K8s!"
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 5678
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: test.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello
                port:
                  number: 80
EOF

# ⏳ Wait for pods to spin up
kubectl rollout status deployment hello

# 🌐 Test the routing! (Spoofing the Host header so NGINX knows where to send it)
curl -H "Host: test.local" http://10.0.0.200
```

**Expected Output:**
```text
Hello from AlmaLinux 10 K8s!
```

---

## 🧹 Appendix: Safe Node Reset (RHEL 10)
*If a worker node completely breaks and you need to wipe it and rejoin it, use this script. It uses `nft` instead of the deprecated `iptables` command to safely flush the network rules.*

```bash
sudo kubeadm reset --force --cri-socket unix:///run/containerd/containerd.sock
sudo rm -rf /etc/cni/net.d /var/lib/cni /var/lib/kubelet /etc/kubernetes
sudo ip link delete vxlan.calico 2>/dev/null || true
sudo ip link delete cali+ 2>/dev/null || true

# 🧱 RHEL 10 uses nftables natively
sudo nft flush ruleset 2>/dev/null || true

sudo systemctl restart containerd
sudo systemctl restart kubelet
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
