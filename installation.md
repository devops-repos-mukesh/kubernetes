# Hybrid Kubernetes Cluster Setup (Laptop as Control Plane + AWS EC2 as Worker)

This guide explains how to create a hybrid Kubernetes cluster where:

- **Control Plane:** Local Ubuntu Laptop
- **Worker Node:** AWS EC2 Ubuntu Instance
- **Container Runtime:** containerd
- **Networking:** Tailscale VPN
- **Bootstrap Tool:** kubeadm
- **CNI Plugin:** Calico

---

## Architecture

<img width="1176" height="740" alt="image" src="https://github.com/user-attachments/assets/040c1b9c-15d3-4c5d-8d34-9b87da4cfb1a" />


---

# Prerequisites

## Laptop

- Ubuntu 24.04
- 4 CPU
- 8 GB RAM
- Internet Connection

## EC2

- Ubuntu 24.04
- t3.medium or larger
- Security Group allowing SSH

---

# 1. Update System

Run on **both machines**

```bash
sudo apt update
sudo apt upgrade -y
```

---

# 2. Disable Swap

```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

Verify

```bash
free -h
```

---

# 3. Load Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

---

# 4. Configure Sysctl

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
```

Apply

```bash
sudo sysctl --system
```

---

# 5. Install Required Packages

```bash
sudo apt install -y \
containerd \
conntrack \
socat \
ebtables \
ethtool \
ipset \
apt-transport-https \
ca-certificates \
curl \
gpg
```

---

# 6. Configure containerd

Generate config

```bash
sudo mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml
```

Edit

```bash
sudo nano /etc/containerd/config.toml
```

Change

```
SystemdCgroup = false
```

to

```
SystemdCgroup = true
```

Restart

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

# 7. Install Kubernetes

```bash
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key \
| sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update

sudo apt install -y kubelet kubeadm kubectl

sudo apt-mark hold kubelet kubeadm kubectl
```

---

# 8. Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Start

```bash
sudo tailscale up
```

Login using the URL displayed.

Verify

```bash
tailscale status
```

Example

```
100.x.x.x laptop
100.x.x.x ec2
```

---

# 9. Initialize Control Plane

Run **only on Laptop**

Find Tailscale IP

```bash
tailscale ip -4
```

Example

```
100.114.208.15
```

Initialize

```bash
sudo kubeadm init \
--apiserver-advertise-address=100.114.208.15 \
--control-plane-endpoint=100.114.208.15:6443 \
--pod-network-cidr=192.168.0.0/16
```

---

# 10. Configure kubectl

```bash
mkdir -p ~/.kube

sudo cp /etc/kubernetes/admin.conf ~/.kube/config

sudo chown $(id -u):$(id -g) ~/.kube/config
```

---

# 11. Install Calico

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/calico.yaml
```

Wait until pods become Running

```bash
kubectl get pods -n kube-system
```

---

# 12. Get Join Command

```bash
kubeadm token create --print-join-command
```

Example

```bash
sudo kubeadm join 100.114.208.15:6443 \
--token <TOKEN> \
--discovery-token-ca-cert-hash sha256:<HASH>
```

---

# 13. Join Worker

Run on EC2

```bash
sudo kubeadm join 100.114.208.15:6443 \
--token <TOKEN> \
--discovery-token-ca-cert-hash sha256:<HASH>
```

---

# 14. Verify Cluster

```bash
kubectl get nodes -o wide
```

Expected

```
NAME              STATUS   ROLES
mukesharma        Ready    control-plane
ip-172-31-90-92   Ready    <none>
```

---

# Useful Commands

View Nodes

```bash
kubectl get nodes
```

Watch Nodes

```bash
kubectl get nodes -w
```

View Pods

```bash
kubectl get pods -A
```

View System Pods

```bash
kubectl get pods -n kube-system
```

Describe Node

```bash
kubectl describe node <node-name>
```

Cluster Info

```bash
kubectl cluster-info
```

---

# Troubleshooting

### conntrack missing

```bash
sudo apt install conntrack
```

---

### API Server Timeout

Check

```bash
curl -k https://<TAILSCALE_IP>:6443/healthz
```

Should return

```
ok
```

---

### Verify Tailscale

```bash
tailscale status
```

Both machines must appear.

---

### Verify API Server

```bash
sudo ss -tlnp | grep 6443
```

---

### Check kubelet

```bash
sudo systemctl status kubelet
```

---

### Check containerd

```bash
sudo systemctl status containerd
```

---

### Reset Cluster

Control Plane

```bash
sudo kubeadm reset -f

sudo rm -rf ~/.kube

sudo rm -rf /etc/cni/net.d
```

Worker

```bash
sudo kubeadm reset -f

sudo rm -rf /etc/cni/net.d
```

---

# Final Result

<img width="1176" height="740" alt="image" src="https://github.com/user-attachments/assets/7695ac4c-abf4-4fbf-8615-7728716f8a11" />




---



# Kubernetes Hybrid Cluster SOP
**Control plane:** Local Ubuntu (`mukesharma`) · Tailscale `100.78.23.85` · K8s `v1.31.14`  
**Worker:** AWS EC2 `ip-172-31-82-100` · PodCIDR `192.168.1.0/24`  
**Connectivity:** Tailscale VPN · API on TCP `6443`
---
## Architecture
```
Laptop (Ubuntu) ── Tailscale ──► Control Plane (6443) ── Tailscale ──► EC2 Worker
   100.78.23.85                    etcd, scheduler,                    192.168.1.0/24
                                   controller-manager                 PodCIDR
                                   Calico
```
---
## Prerequisites
| Component | Control Plane | Worker (EC2) |
|-----------|---------------|--------------|
| OS | Ubuntu 22.04+ | Ubuntu 22.04+ |
| RAM | ≥ 2 GB | ≥ 2 GB |
| Swap | Disabled | Disabled |
| Runtime | containerd | containerd |
| Network | Tailscale installed | Tailscale installed |
| Ports | 6443, 2379-2380, 10250, 10259, 10257 | 10250, 30000-32767 |
```bash
# Both nodes
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo modprobe overlay br_netfilter
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```
---
## Step 1: Install Tailscale (Both Nodes)
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale ip -4    # note IPs for both nodes
```
Ensure both nodes appear in the same Tailscale tailnet and can ping each other.
---
## Step 2: Install Kubernetes v1.31.14 (Both Nodes)
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet=1.31.14-* kubeadm=1.31.14-* kubectl=1.31.14-*
sudo apt-mark hold kubelet kubeadm kubectl
```
Install containerd on both nodes and enable systemd cgroup:
```bash
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable --now kubelet
```
---
## Step 3: Initialize Control Plane (Laptop)
Use the **Tailscale IP** as the API advertise address so the remote worker can reach it.
```bash
TS_IP=$(tailscale ip -4)
sudo kubeadm init \
  --apiserver-advertise-address=$TS_IP \
  --pod-network-cidr=192.168.0.0/16 \
  --kubernetes-version=v1.31.14
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Allow workloads on the control-plane node (single-node dev) if needed:
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```
---
## Step 4: Install Calico (Control Plane)
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
kubectl get pods -n calico-system -w
```
---
## Step 5: Join EC2 Worker
On the **control plane**, generate a join command:
```bash
kubeadm token create --print-join-command
```
On **EC2 worker**, run the join command with the Tailscale IP of the control plane:
```bash
sudo kubeadm join 100.78.23.85:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```
Verify from control plane:
```bash
kubectl get nodes -o wide
```
Expected: both nodes `Ready`, worker shows EC2 internal IP and assigned PodCIDR.
---
## Step 6: Configure kubectl on Laptop
Point kubeconfig to the Tailscale API endpoint:
```bash
# In ~/.kube/config, set server to:
# server: https://100.78.23.85:6443
kubectl cluster-info
kubectl get nodes
```
---
## Verification Checklist
```bash
kubectl get nodes                          # both Ready
kubectl get pods -A                        # all Running
kubectl run test --image=nginx --restart=Never
kubectl get pod -o wide                    # scheduled on worker
kubectl delete pod test
```
From laptop, confirm API reachability:
```bash
curl -k https://100.78.23.85:6443/version
tailscale ping <ec2-tailscale-ip>
```
---
## Troubleshooting
### Worker stuck `NotReady`
```bash
kubectl describe node ip-172-31-82-100
kubectl get pods -n calico-system -o wide
sudo journalctl -u kubelet -f          # on worker
```
**Fix:** Ensure Calico pods are Running; verify `net.ipv4.ip_forward=1`; restart kubelet.
---
### Worker cannot join cluster
| Symptom | Fix |
|---------|-----|
| `connection refused` on 6443 | Open TCP 6443 on control plane; confirm `--apiserver-advertise-address` is Tailscale IP |
| `x509 certificate` error | Re-join with fresh token; ensure join uses Tailscale IP, not localhost |
| `token expired` | Run `kubeadm token create --print-join-command` again |
