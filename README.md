# README: Highly Available Kubernetes Cluster Setup

## Overview
This document provides a comprehensive, manual guide to setting up a Highly Available Kubernetes cluster optimized for production. The setup emphasizes simplicity, reliability, and performance.

### Key Features
- **etcd Cluster**: Manages cluster state with high reliability.
- **CNI and Data Plane**: Leverages Cilium with eBPF for efficient networking.
- **Kube-Proxy Replacement**: Cilium replaces kube-proxy for better performance.
- **Native Routing**: Operates on L3 routing without overlay networks.

---

## Prerequisites

### 1. Infrastructure Requirements
- **Control Plane Nodes**: 3
- **Worker Nodes**: 2
- **Networking**: A single L2 broadcast domain (LAN) between all nodes.

### 2. Operating System
- Linux kernel version >= 6.0 (required for eBPF and advanced Cilium features).

### 3. Example Environment
- **Cloud Provider**: Hetzner Cloud
- **Private Network**: `172.16.80.0/24`
- **Machine Specifications**:
  - Type: CPX21
  - vCPU: 3
  - RAM: 4GB
  - Disk: 80GB SSD
  - OS: Debian 12 (kernel 6.1)
- **Software Versions**:
  - Container Runtime: containerd
  - Kubernetes: v1.32
  - Cilium: v1.16.5
- **Network Configuration**:
  - TCP Congestion Control: BBR
  - No network overlay; native routing is used.

---

## Kubernetes Provisioning

### 1. Configure Container Runtime Interface (CRI)

#### Install containerd
```bash
apt-get install containerd
```

#### Generate Default Configuration
```bash
containerd config default > /etc/containerd/config.toml
```

#### Update Configuration
Set `systemd` as the cgroup driver:
```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```
Set the sandbox image:
```toml
sandbox_image = "registry.k8s.io/pause:3.10"
```
Restart containerd:
```bash
systemctl restart containerd.service
```

#### Configure crictl
```bash
echo "runtime-endpoint: unix:///run/containerd/containerd.sock" > /etc/crictl.yaml
```

---

### 2. OS Kernel and Networking Configuration

#### Update Kernel Parameters
Append the following settings to `/etc/sysctl.conf`:
```bash
net.core.somaxconn = 65500
net.core.netdev_max_backlog = 20000
net.ipv4.ip_forward = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_sack = 1
net.ipv4.tcp_orphan_retries = 2
net.ipv4.tcp_timestamps = 1
net.ipv4.ip_local_port_range = 2000 65500
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_rfc1337 = 1
net.ipv4.tcp_no_metrics_save = 1
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_congestion_control = bbr
net.core.default_qdisc = fq
fs.file-max = 4194304
```
Apply the changes:
```bash
sysctl -p
```

#### Load BBR Module
Enable and persist TCP BBR:
```bash
modprobe tcp_bbr
echo "tcp_bbr" > /etc/modules-load.d/bbr.conf
```

---

### 3. Kubernetes Control Plane Setup

#### Install Kubernetes Tools
Install the required tools:
```bash
apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

#### Initialize Kubernetes Cluster
Turn off swap:
```bash
swapoff -a
```
Run the `kubeadm init` command:
```bash
kubeadm init \
  --control-plane-endpoint "<API_SERVER_IP>:<API_SERVER_PORT>" \
  --upload-certs \
  --v=5
```

---

### 4. Cilium Installation

#### Install Cilium CLI
Download and install the CLI:
```bash
wget https://github.com/cilium/cilium-cli/releases/download/v0.16.23/cilium-linux-amd64.tar.gz
tar -zxvf cilium-linux-amd64.tar.gz
mv cilium /usr/local/bin/
cilium version
```

#### Install Helm and Add Cilium Repository
Install Helm:
```bash
wget https://get.helm.sh/helm-v3.16.4-linux-amd64.tar.gz
tar -zxvf helm-v3.16.4-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/
helm version
```
Add the Cilium Helm repository:
```bash
helm repo add cilium https://helm.cilium.io/
```

#### Deploy Cilium
Install Cilium using Helm:
```bash
helm install cilium cilium/cilium --version 1.16.5 \
  --namespace kube-system \
  --set k8sServiceHost=${API_SERVER_IP} \
  --set k8sServicePort=${API_SERVER_PORT} \
  --set enableBpfMasquerade=true \
  --set autoDirectNodeRoutes=true \
  --set kubeProxyReplacement=true
```

---

### 5. Validate and Finalize Setup

#### Verify Node and Pod Status
Check the cluster state:
```bash
kubectl get nodes
kubectl get pods --all-namespaces
```

#### Replace kube-proxy with Cilium
Remove kube-proxy:
```bash
kubectl -n kube-system delete ds kube-proxy
kubectl -n kube-system delete cm kube-proxy
iptables-save | grep -v KUBE | iptables-restore
```

#### Validate Cilium Configuration
Confirm that Cilium is properly configured:
```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg status | grep KubeProxyReplacement
kubectl -n kube-system exec ds/cilium -- cilium-dbg status --verbose
```

---

## Additional Resources
- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [Cilium Documentation](https://docs.cilium.io/en/stable/)
- [Kubeadm HA Setup Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)

---

## Conclusion
Your Kubernetes cluster with Cilium as the CNI and proxy replacement is now fully set up and optimized for production use. For further customization or scaling, refer to the official documentation linked above.
