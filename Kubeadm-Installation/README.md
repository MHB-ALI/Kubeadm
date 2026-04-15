# Kubeadm Deployment on Azure

A step-by-step guide for deploying a Kubernetes cluster using `kubeadm` on Azure virtual machines (tested on Rocky Linux).

> **Note:** This guide sets up a single-node cluster where the control-plane node also runs workloads. For a production multi-node setup, skip step 17.

---

## Prerequisites

- An Azure account with sufficient quota
- SSH access to your VM(s)
- Basic familiarity with Linux and Kubernetes concepts

---

## Steps

### 1. Create a Resource Group

Create a Resource Group in the closest and most cost-effective region to minimize latency and cost.

---

### 2. Create a Virtual Network (VNet)

Set up a VNet to provide network isolation for your resources.

---

### 3. Create a Network Security Group (NSG)

- Internet access is handled via Azure's default outbound NAT — no NAT Gateway needed for a test lab.
- Assign a **Public IP** to each VM only if you need to SSH in from your local machine.
  - Basic SKU Public IPs are free while the VM is deallocated.

---

### 4. Configure NSG Inbound Rules

Add your local machine's IP address to the SSH inbound rule to restrict access.

---

### 5. Create the Virtual Machine(s)

Provision your VM(s) and attach them to the VNet and NSG created above.

---

### 6. Disable SELinux

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

---

### 7. Disable Swap

```bash
sudo swapoff -a
```

Then open `/etc/fstab` and comment out the swap line to ensure it stays disabled after reboots:

```
# /dev/mapper/rl-swap    swap    swap    defaults    0 0
```

> If the line is not present, swap is already disabled by default.

---

### 8. Disable Firewall

```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

---

### 9. Enable IP Forwarding

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply without reboot
sudo sysctl --system
```

---

### 10. Load Required Kernel Modules

```bash
# Register modules to load at boot time
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Load modules immediately
sudo modprobe overlay
sudo modprobe br_netfilter
```

---

### 11. Install containerd

```bash
# Install yum-utils
sudo yum install -y yum-utils

# Add the Docker repo (contains containerd)
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Install containerd
sudo yum install -y containerd.io

# Enable and start the service
sudo systemctl enable containerd --now
```

---

### 12. Configure the systemd Cgroup Driver for containerd

```bash
# Generate default config
containerd config default | sudo tee /etc/containerd/config.toml
```

Edit `/etc/containerd/config.toml` and set `SystemdCgroup = true`:

```toml
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc]
  ...
  [plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
    SystemdCgroup = true
```

```bash
sudo systemctl restart containerd
```

---

### 13. Add the Kubernetes Repository and Install Components

> The `exclude` parameter prevents `yum update` from inadvertently upgrading Kubernetes packages. Kubernetes upgrades require a dedicated procedure.

```bash
# Add Kubernetes repo (adjust version as needed)
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

# Install kubelet, kubeadm, and kubectl
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

---

### 14. Enable the kubelet Service

```bash
sudo systemctl enable --now kubelet
```

---

### 15. Set Up kubectl Access

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

### 16. Install a CNI Plugin

Apply your CNI manifest. The example below uses Calico:

```bash
kubectl apply -f <your-cni-manifest>.yaml
```

Popular CNI options: [Calico](https://docs.tigera.io/calico/latest/getting-started/kubernetes/), [Flannel](https://github.com/flannel-io/flannel), [Cilium](https://cilium.io/).

---

### 17. Allow Pods on the Control-Plane Node *(single-node only)*

By default, Kubernetes taints the control-plane node to prevent workloads from running on it. For a single-node setup, remove the taint:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

> For multi-node clusters, skip this step. It is recommended to read about [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) to understand the concept.

---

### 18. Verify the Cluster

```bash
kubectl get nodes
kubectl get pods --all-namespaces
```

---

## Automation

The steps above are a lengthy manual process. For a faster setup, an Ansible playbook is available that automates the `kubeadm` deployment:

**[ansible-vanila-k8-playbook by Eng. Hamid Mahdi](https://github.com/HamidM92/ansible-vanila-k8-playbook)**

> It is recommended to integrate this as part of your own Ansible playbook for maximum flexibility and time savings.

---

## References

- [Kubernetes Official Docs — kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [containerd Getting Started](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)
- [Calico CNI](https://docs.tigera.io/calico/latest/getting-started/kubernetes/)
