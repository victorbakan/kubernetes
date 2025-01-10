# High Availability Kubernetes Cluster Setup

This page describes the process for setting up a highly available Kubernetes cluster manually. The configuration is tailored for a production environment, prioritizing simplicity and performance.

---

## Overview

The setup incorporates the following key decisions:

- **etcd Cluster**: Used as part of Kubernetes for simplicity.
- **Networking**:  
  - **CNI & Data Plane**: Cilium with eBPF.
  - **Kube-Proxy Replacement**: Cilium.
  - **Routing**: Native Layer 3 (L3) routing without overlay networks.

---

## Prerequisites

### Hardware/VM Requirements
- **Control Plane Machines**: 3 nodes.
- **Worker Machines**: 2 nodes.
- **Network**: A single L2 broadcast domain connecting all nodes (LAN). Required for native routing with Cilium.
- **Operating System**: Linux kernel version >= 6.0 (for eBPF and Cilium features).

### Sample Environment
- **Provider**: Hetzner Cloud.
- **Private Network**: `172.16.80.0/24` (L2 broadcast domain).
- **Node Specifications**:
  - Type: CPX21
  - vCPU: 3
  - RAM: 4GB
  - Disk: 80GB SSD
  - OS: Debian 12 with Linux kernel 6.1.
- **Networking Configuration**:
  - **Congestion Control**: TCP BBR.
- **Tools**:
  - **CRI**: `containerd`.
  - **Kubernetes**: v1.32.
  - **Cilium**: v1.16.5 with full kube-proxy replacement.
  - **GitOps**: FluxCD for application deployment.

---

## Setup Instructions

### 1. Install and Configure `containerd`

1. Install `containerd`:
   ```bash
   apt-get install containerd
   ```

2. Generate default configuration:
   ```bash
   containerd config default > /etc/containerd/config.toml
   ```

3. Update `config.toml`:
   - Set `SystemdCgroup` to `true`:
     ```toml
     [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
       SystemdCgroup = true
     ```
   - Set sandbox image to the latest version (e.g., `3.10`):
     ```toml
     sandbox_image = "registry.k8s.io/pause:3.10"
     ```

4. Restart the `containerd` service:
   ```bash
   systemctl restart containerd.service
   ```

5. Add explicit runtime endpoint for `crictl`:
   ```bash
   echo "runtime-endpoint: unix:///run/containerd/containerd.sock" > /etc/crictl.yaml
   ```

### 2. Configure OS Kernel Networking (sysctl)

1. Update `/etc/sysctl.conf` with the following configurations:
   ```conf
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

2. Load the TCP BBR module:
   ```bash
   modprobe tcp_bbr
   echo "tcp_bbr" > /etc/modules-load.d/bbr.conf
   ```

3. Apply the sysctl configurations:
   ```bash
   sysctl -p
   ```

### 3. Provision Kubernetes Control Plane Nodes

1. Turn off swap:
   ```bash
   swapoff -a
   ```

2. Install Kubernetes components:
   ```bash
   apt-get install -y apt-transport-https ca-certificates curl gpg
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
   apt-get update
   apt-get install -y kubelet kubeadm kubectl
   apt-mark hold kubelet kubeadm kubectl
   ```

3. Initialize Kubernetes without kube-proxy:
   ```bash
   kubeadm init \
     --skip-phases=addon/kube-proxy \
     --control-plane-endpoint "<LOAD_BALANCER_DNS>:<LOAD_BALANCER_PORT>" \
     --upload-certs \
     --pod-network-cidr "10.248.0.0/16" \
     --v=5
   ```

### 4. Install and Configure Cilium

1. Install `cilium-cli`:
   ```bash
   wget https://github.com/cilium/cilium-cli/releases/download/v0.16.23/cilium-linux-amd64.tar.gz
   tar -zxvf cilium-linux-amd64.tar.gz
   mv cilium /usr/local/bin/
   cilium version
   ```

2. Install Helm and add the Cilium repository:
   ```bash
   wget https://get.helm.sh/helm-v3.16.4-linux-amd64.tar.gz
   tar -zxvf helm-v3.16.4-linux-amd64.tar.gz
   mv linux-amd64/helm /usr/local/bin/
   helm repo add cilium https://helm.cilium.io/
   ```

3. Install Cilium with custom options:
   ```bash
   helm install cilium cilium/cilium --version 1.16.5 -n kube-system --create-namespace \
    --set tunnel=disabled \
    --set enableBpfMasquerade=true \
    --set autoDirectNodeRoutes=true \
    --set kubeProxyReplacement=strict \
    --set cluster.name=k8s-cilium-rnd-cluster \
    --set cluster.id=1
   ```

---

## Additional Information

For detailed guidance, refer to:
- [Kubernetes High Availability Setup](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [Cilium Documentation](https://docs.cilium.io/)

