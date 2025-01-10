# Highly Available Kubernetes Cluster Setup

This guide describes the steps to manually set up a Highly Available Kubernetes (HAK8s) cluster suitable for production environments. The setup prioritizes simplicity and performance, leveraging modern technologies like Cilium with eBPF.

## Key Features
- **Etcd Cluster:** Integrated as part of Kubernetes for simplicity.
- **Cilium:** Used for CNI and data plane, replacing kube-proxy with native L3 routing.
- **No Overlay Networks:** Native routing on L3 level.

## Prerequisites

Ensure the following requirements are met before proceeding:

### Hardware/VM Requirements
- **Control Plane Machines:** 3
- **Worker Machines:** 2
- **Network:** Single L2 broadcast domain across all machines (LAN).

### Software Requirements
- **OS:** Linux kernel ≥ 6.0 (to fully utilize eBPF and Cilium features).
- **CRI:** `containerd`
- **Kubernetes:** v1.32
- **Cilium:** eBPF with native routing mode and kube-proxy replacement.
- **GitOps Tool:** `FluxCD` for deploying business applications.

### Example Environment
For demonstration purposes, the setup is based on the following configuration:

- **Cloud Provider:** Hetzner Cloud
- **Private Network:** `172.16.80.0/24` attached to all VMs
- **VM Specifications:**
  - Type: `CPX21`
  - vCPU: 3
  - RAM: 4GB
  - Disk SSD: 80GB
  - OS: Debian 12 with Linux kernel 6.1

---

## Step 1: Configure `containerd`

Container runtime interface (CRI) is configured as follows:

### Install `containerd`
```bash
apt-get install containerd
```

### Generate Default Configuration
```bash
containerd config default > /etc/containerd/config.toml
```

### Update Configuration
Ensure the following settings are applied:

1. Use `systemd` as the cgroup driver:
   ```toml
   [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
       SystemdCgroup = true
   ```

2. Update the sandbox image:
   ```toml
   sandbox_image = "registry.k8s.io/pause:3.10"
   ```

### Restart Service
```bash
systemctl restart containerd.service
```

### Configure `crictl`
To avoid deprecated warnings, explicitly set the runtime endpoint:

Create or edit `/etc/crictl.yaml` with the following content:
```yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
```

---

## Step 2: Provision the Kubernetes Control Plane

### Setup the First Control Plane Node
(Additional steps to be detailed here for configuring the control plane and ensuring high availability.)

---

## Data Plane

- **Cilium:**
  - eBPF-based native routing in a single LAN (`172.16.80.0/24`).
  - Replaces kube-proxy for improved performance and simplicity.

---

## GitOps Integration

Deploy business applications using FluxCD for seamless GitOps workflows. 

---

## Notes

This document assumes familiarity with Linux systems and Kubernetes concepts. It is recommended to have basic networking knowledge to handle L2/L3 configurations.

---

### Future Enhancements
- Automating the manual setup process using Ansible or Terraform.
- Adding monitoring and logging tools (e.g., Prometheus, Grafana).
- Enhancing security with Pod Security Policies and Network Policies.

