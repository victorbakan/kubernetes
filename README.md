# README: Highly Available Kubernetes Cluster Setup

## Overview
This document describes the manual setup process for a Highly Available Kubernetes cluster suitable for production environments. The setup emphasizes simplicity and performance.

### Key Features:
- **etcd cluster**: Simplifies Kubernetes cluster management.
- **CNI and Data Plane**: Utilizes Cilium with eBPF.
- **Kube-Proxy Replacement**: Replaced with Cilium.
- **Native Routing**: Operates on L3 level without overlay.

---

## Prerequisites

1. **Hardware/VM Requirements:**
   - **Control Plane Machines**: 3
   - **Worker Machines**: 2
   - Single L2 broadcast domain (LAN) between all machines.
2. **Operating System:**
   - Linux kernel version >= 6.0 to enable eBPF and Cilium features.
3. **Example Environment:**
   - Cloud Provider: Hetzner Cloud.
   - Private Network: `172.16.80.0/24`.
   - Machine Specifications:
     - Type: CPX21
     - vCPU: 3
     - RAM: 4GB
     - Disk: 80GB SSD
     - OS: Debian 12 (kernel 6.1)
     - CRI: containerd
     - Kubernetes v1.32
     - Cilium v1.16.5
   - Network Configuration:
     - TCP Congestion Control: BBR
     - No network overlay; native routing.

---

## Kubernetes Provisioning

### 1. Configure Container Runtime Interface (CRI)

#### Install containerd:
```bash
apt-get install containerd
```

#### Generate Default Configuration:
```bash
containerd config default > /etc/containerd/config.toml
```

#### Update Configuration:
Ensure `systemd` is set as the cgroup driver:
```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```
Set sandbox image:
```toml
sandbox_image = "registry.k8s.io/pause:3.10"
```
Restart containerd:
```bash
systemctl restart containerd.service
```

#### Configure crictl:
```bash
echo "runtime-endpoint: unix:///run/containerd/containerd.sock" > /etc/crictl.yaml
```

---

### 2. OS Kernel and Networking Configuration

#### Update `sysctl` Settings:
Add the following to `/etc/sysctl.conf`:
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
Apply changes:
```bash
sysctl -p
```

#### Load BBR Module:
```bash
modprobe tcp_bbr
echo "tcp_bbr" > /etc/modules-load.d/bbr.conf
```

---

### 3. Kubernetes Control Plane Setup

#### Install Kubernetes Tools:
Follow the [official Kubernetes documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/):
```bash
apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

#### Initialize Kubernetes:
Turn off swap:
```bash
swapoff -a
```
Run `kubeadm init`:
```bash
kubeadm init \
  --pod-network-cidr "10.248.0.0/16" \
  --control-plane-endpoint "k8s-lb-etcd-apiserver.bluecuttlefish.com:6443" \
  --upload-certs \
  --v=5
```

---

### 4. Cilium Installation

#### Install Cilium CLI:
```bash
wget https://github.com/cilium/cilium-cli/releases/download/v0.16.23/cilium-linux-amd64.tar.gz
tar -zxvf cilium-linux-amd64.tar.gz
mv cilium /usr/local/bin/
cilium version
```

#### Install Helm and Add Cilium Repository:
```bash
wget https://get.helm.sh/helm-v3.16.4-linux-amd64.tar.gz
tar -zxvf helm-v3.16.4-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/
helm version
helm repo add cilium https://helm.cilium.io/
```

#### Install Cilium:
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

#### Check Node and Pod Status:
```bash
kubectl get pods --all-namespaces
```

#### Replace kube-proxy with Cilium:
```bash
kubectl -n kube-system delete ds kube-proxy
kubectl -n kube-system delete cm kube-proxy
iptables-save | grep -v KUBE | iptables-restore
```

#### Update Cilium Configuration:
```bash
helm upgrade cilium cilium/cilium \
  --version 1.16.5 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=${API_SERVER_IP} \
  --set k8sServicePort=${API_SERVER_PORT}
```

Verify configuration:
```bash
kubectl -n kube-system exec ds/cilium -- cilium-dbg status | grep KubeProxyReplacement
kubectl -n kube-system exec ds/cilium -- cilium-dbg status --verbose
```

---

## Conclusion
Your Kubernetes cluster with Cilium as the CNI and proxy replacement is now set up and optimized for production use. For additional configurations and scaling, refer to the official documentation.

