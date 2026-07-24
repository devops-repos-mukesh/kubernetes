# Kubernetes Hybrid Cluster Setup
## Control Plane on Local Laptop + Worker Node on AWS EC2

This guide demonstrates how to create a hybrid Kubernetes cluster where:

- **Control Plane (Master Node):** Local Ubuntu Laptop
- **Worker Node:** AWS EC2 Instance (Ubuntu)
- **Container Runtime:** containerd
- **Bootstrap Tool:** kubeadm
- **CNI:** Calico

---

# Architecture

```text
                        Internet
                            │
                     Public IP / VPN
                            │
                    +----------------+
                    |   AWS EC2      |
                    | Ubuntu         |
                    | Worker Node    |
                    +----------------+
                            ▲
                            │
                       kubeadm join
                            │
────────────────────────────┼────────────────────────────
                            │
                    Home / Office Network
                            │
                    +----------------+
                    |   Local Laptop |
                    | Ubuntu         |
                    | Control Plane  |
                    | API Server     |
                    | etcd           |
                    | Scheduler      |
                    +----------------+
```

---

# Prerequisites

## Local Laptop

- Ubuntu 22.04 / 24.04
- Minimum 4 vCPUs
- Minimum 8 GB RAM
- Stable Internet
- Public IP or VPN (Recommended)

---

## AWS EC2

- Ubuntu 22.04 / 24.04
- t3.medium or higher
- Security Group configured
- Public IP

---

# Step 1: Update System

Run on **both machines**

```bash
sudo apt update
sudo apt upgrade -y
```

---

# Step 2: Disable Swap

Run on **both machines**

```bash
sudo swapoff -a

sudo sed -i '/swap/d' /etc/fstab
```

Verify

```bash
free -h
```

---

# Step 3: Enable Required Kernel Modules

Run on **both machines**

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

Load modules

```bash
sudo modprobe overlay

sudo modprobe br_netfilter
```

Verify

```bash
lsmod | grep br_netfilter

lsmod | grep overlay
```

---

# Step 4: Configure Sysctl

Run on **both machines**

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

Verify

```bash
sysctl net.ipv4.ip_forward
```

Output

```
net.ipv4.ip_forward = 1
```

---

# Step 5: Install Containerd

Run on **both machines**

```bash
sudo apt install -y containerd
```

Create configuration

```bash
sudo mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml
```

---

# Step 6: Configure containerd

Edit

```bash
sudo nano /etc/containerd/config.toml
```

Find

```
SystemdCgroup = false
```

Replace with

```
SystemdCgroup = true
```

Restart

```bash
sudo systemctl restart containerd

sudo systemctl enable containerd
```

Verify

```bash
systemctl status containerd
```

---

# Step 7: Install Kubernetes Components

Run on **both machines**

Install dependencies

```bash
sudo apt install -y apt-transport-https ca-certificates curl gpg
```

Create keyring directory

```bash
sudo mkdir -p /etc/apt/keyrings
```

Add Kubernetes repository

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key \
| sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add repository

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install Kubernetes

```bash
sudo apt update

sudo apt install -y kubelet kubeadm kubectl

sudo apt-mark hold kubelet kubeadm kubectl
```

Verify

```bash
kubectl version --client

kubeadm version

kubelet --version
```

---

# Step 8: Configure Firewall

## Laptop

Allow

```
6443
2379-2380
10250
10257
10259
```

Example (UFW)

```bash
sudo ufw allow 6443/tcp

sudo ufw allow 2379:2380/tcp

sudo ufw allow 10250/tcp

sudo ufw allow 10257/tcp

sudo ufw allow 10259/tcp
```

---

## AWS Security Group

Allow inbound

| Port | Source |
|--------|------------|
|22|Your Public IP|
|6443|Your Public IP|
|10250|Your Public IP|
|30000-32767|Optional|

---

# Step 9: Get Laptop Public IP

```bash
curl ifconfig.me
```

Example

```
49.xx.xx.xx
```

---

# Step 10: Initialize Control Plane

Run only on **Laptop**

```bash
sudo kubeadm init \
--control-plane-endpoint=<YOUR_PUBLIC_IP>:6443 \
--pod-network-cidr=192.168.0.0/16
```

Example

```bash
sudo kubeadm init \
--control-plane-endpoint=49.xx.xx.xx:6443 \
--pod-network-cidr=192.168.0.0/16
```

Wait until initialization completes.

---

# Step 11: Configure kubectl

Run only on **Laptop**

```bash
mkdir -p $HOME/.kube

sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify

```bash
kubectl get nodes
```

Output

```
control-plane    NotReady
```

---

# Step 12: Install Calico

Run only on **Laptop**

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/master/manifests/calico.yaml
```

Check pods

```bash
kubectl get pods -n kube-system
```

---

# Step 13: Get Join Command

If kubeadm init just completed, it will display something like

```bash
kubeadm join <CONTROL_PLANE_IP>:6443 \
--token <TOKEN> \
--discovery-token-ca-cert-hash sha256:<HASH>
```

Copy this command.

---

## If Join Command is Lost

Generate a new token

```bash
kubeadm token create --print-join-command
```

---

# Step 14: Join EC2 Worker

Run on **EC2**

Example

```bash
sudo kubeadm join 49.xx.xx.xx:6443 \
--token abcdef.0123456789abcdef \
--discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Wait for successful join.

---

# Step 15: Verify Cluster

Run on Laptop

```bash
kubectl get nodes
```

Output

```
NAME            STATUS   ROLES           AGE

laptop          Ready    control-plane   10m

ec2-worker      Ready    <none>          2m
```

---

# Useful Commands

## List Nodes

```bash
kubectl get nodes
```

---

## List All Pods

```bash
kubectl get pods -A
```

---

## Cluster Information

```bash
kubectl cluster-info
```

---

## Check Node Details

```bash
kubectl describe node <node-name>
```

Example

```bash
kubectl describe node ip-172-31-22-10
```

---

## View Worker Status

```bash
kubectl get nodes -o wide
```

---

## Check kubelet

```bash
sudo systemctl status kubelet
```

---

## Check containerd

```bash
sudo systemctl status containerd
```

---

## Restart kubelet

```bash
sudo systemctl restart kubelet
```

---

## Restart containerd

```bash
sudo systemctl restart containerd
```

---

## Check Kubernetes System Pods

```bash
kubectl get pods -n kube-system
```

---

## Watch Cluster

```bash
watch kubectl get nodes
```

---

## Drain Worker

```bash
kubectl drain <node-name> --ignore-daemonsets
```

---

## Uncordon Worker

```bash
kubectl uncordon <node-name>
```

---

## Delete Worker

```bash
kubectl delete node <node-name>
```

---

# Reset Cluster

## Control Plane

```bash
sudo kubeadm reset -f

sudo rm -rf ~/.kube

sudo rm -rf /etc/cni/net.d
```

---

## Worker

```bash
sudo kubeadm reset -f

sudo rm -rf /etc/cni/net.d
```

---

# Troubleshooting

## Worker Cannot Join

Check connectivity

```bash
nc -zv <CONTROL_PLANE_IP> 6443
```

---

## Check API Server

```bash
kubectl get pods -n kube-system
```

---

## Check kubelet Logs

```bash
journalctl -u kubelet -f
```

---

## Check containerd Logs

```bash
journalctl -u containerd -f
```

---

## Check Node Status

```bash
kubectl describe node <node-name>
```

---

# Notes

- The control plane runs on your local laptop.
- The EC2 instance acts as a worker node.
- The control plane must remain powered on for the cluster to function.
- If using a public IP, ensure TCP port **6443** is reachable from the EC2 instance.
- For better security and reliability, consider using a VPN solution like **Tailscale** or **WireGuard** instead of exposing the API server directly to the internet.
