# Setting Up a Kubernetes Cluster with kubeadm + Tailscale + Calico (From Scratch)

This documents how to set up a brand-new cluster where nodes can be on completely
different networks (e.g. your laptop as master + a cloud VM as worker) using
Tailscale as the overlay network so no public ports or port-forwarding are needed.

**Example topology used in this doc** (replace with your own):
- Master: a laptop, Tailscale IP `100.78.23.85`
- Worker: an EC2 instance, Tailscale IP `100.94.52.116`
- Kubernetes version: v1.31.14 (pin an exact version across ALL nodes)
- Pod network CIDR: `192.168.0.0/16`
- CNI: Calico (VXLAN encapsulation, bound to `tailscale0`)

You can add more workers later — see the separate `README.md` (Adding a New Node) for that.

---

## 0. Before you start

Decide and write down for your own setup:
- Which machine is the **master** (control plane)
- Which machine(s) are **workers**
- The exact Kubernetes version you'll pin everywhere (don't rely on "latest" — pick one, e.g. `1.31.14-1.1`, and use it on every node)

---

## 1. Install Tailscale on every node

On **every** machine (master and all workers):
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Get each node's Tailscale IP and record it:
```bash
tailscale ip -4
```

Confirm all nodes can see each other:
```bash
tailscale status
```

---

## 2. Install prerequisites on every node

On **every** machine (master and all workers):

### 2a. Disable swap
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### 2b. Load required kernel modules and sysctl settings
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 2c. Install containerd
```bash
sudo apt-get update
sudo apt-get install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable systemd cgroup driver (required for kubelet)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 2d. Install kubeadm, kubelet, kubectl — pin the exact version

⚠️ **Important**: default/newer OS images often carry a very new Kubernetes repo. Always pin explicitly to the version you decided on (`v1.31` shown here) to avoid version-mismatch errors during join.

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
apt-cache madison kubeadm    # pick your exact version from here, e.g. 1.31.14-1.1

sudo apt-get install -y kubelet=1.31.14-1.1 kubeadm=1.31.14-1.1 kubectl=1.31.14-1.1
sudo apt-mark hold kubelet kubeadm kubectl
```

Verify on every node:
```bash
kubeadm version
kubelet --version
```
All nodes should report the same version.

---

## 3. Initialize the control plane (on the master only)

Use the master's **Tailscale IP** for `--apiserver-advertise-address` — this is what makes the cluster reachable across networks via the VPN instead of relying on local/public IPs.

```bash
sudo kubeadm init \
  --apiserver-advertise-address=<master-tailscale-ip> \
  --apiserver-cert-extra-sans=<master-tailscale-ip> \
  --pod-network-cidr=192.168.0.0/16
```

Set up kubectl access:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Save the `kubeadm join ...` command printed at the end** — you'll need it for workers. (It expires in 24h; regenerate anytime with `kubeadm token create --print-join-command`.)

Check the node (will show `NotReady` until CNI is installed — expected):
```bash
kubectl get nodes
```

---

## 4. Install Calico (on the master only)

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
```

Edit `custom-resources.yaml` — the key changes vs the default download:
- Set `cidr` to match your `--pod-network-cidr`
- Add `nodeAddressAutodetectionV4: interface: tailscale0` — **critical**, otherwise Calico picks the wrong network interface (LAN/private IP instead of Tailscale) and cross-node pod traffic breaks
- Set `mtu: 1350` — Tailscale's own WireGuard tunnel already consumes some MTU budget, so Calico's overlay MTU needs to be smaller to avoid fragmentation issues

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
    nodeAddressAutodetectionV4:
      interface: tailscale0
    mtu: 1350
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
```

Apply and wait for it to come up:
```bash
kubectl apply -f custom-resources.yaml
kubectl get pods -n calico-system -w
```
Ctrl+C once everything shows `Running`. Then confirm the master node is `Ready`:
```bash
kubectl get nodes
```

---

## 5. Join each worker node

Do this for every worker (e.g. the EC2 instance). Each worker must already have completed Steps 1–2 above (Tailscale + prerequisites + matching kubeadm version).

### 5a. Get a fresh join token from the master
```bash
kubeadm token create --print-join-command
```

### 5b. Run it on the worker
```bash
sudo kubeadm join <master-tailscale-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### 5c. Pin kubelet to the worker's Tailscale IP
This makes the node advertise its Tailscale IP instead of its cloud-provider private IP (e.g. EC2's `172.31.x.x`), which the cluster needs for correct pod routing over the VPN.
```bash
sudo mkdir -p /etc/systemd/system/kubelet.service.d/
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service.d/20-tailscale.conf
[Service]
Environment="KUBELET_EXTRA_ARGS=--node-ip=<worker-tailscale-ip>"
EOF

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

---

## 6. Firewall / cloud security group notes

- You do **not** need to expose port `6443` (API server) to the public internet — Tailscale handles all cross-node reachability privately.
- On cloud workers (e.g. EC2 Security Group), allow inbound **UDP 41641** for Tailscale's direct/NAT-traversal connections (falls back to Tailscale's DERP relay if blocked, just slower).
- Make sure host-level firewalls (`ufw`, `firewalld`, `iptables`) aren't blocking traffic on the `tailscale0` interface — specifically port `6443` on the master, and Calico's VXLAN port (UDP `4789`) on all nodes.

---

## 7. Final verification (on the master)

```bash
kubectl get nodes -o wide
```
Checklist:
- All nodes show `STATUS: Ready`
- `INTERNAL-IP` for every node is its **Tailscale IP**, not a local/private network IP
- All versions match

```bash
kubectl get pods -n calico-system -o wide
```
Checklist:
- A `calico-node-xxxxx` pod is `Running` on every node (one per node, it's a DaemonSet)

Optional smoke test — deploy a test pod and confirm networking works:
```bash
kubectl run test-nginx --image=nginx
kubectl get pods -o wide -w
```

---

## Troubleshooting quick reference

| Symptom | Likely cause | Fix |
|---|---|---|
| `kubeadm join` fails: "control plane version >= X" | Worker's kubeadm version too new vs master | Reinstall matching pinned version on the worker (Step 2d) |
| `apt-get install` fails: "Held packages were changed" | Packages already on `apt-mark hold` | Add `--allow-change-held-packages --allow-downgrades` to the install command |
| `kubeadm init` fails: "Port 6443 is in use" / manifests already exist | Leftover state from a previous cluster attempt | `sudo kubeadm reset -f`, then manually clean `/etc/kubernetes`, `/var/lib/etcd`, `~/.kube`, `/etc/cni/net.d`, flush iptables (`sudo iptables -F && sudo iptables -t nat -F`) |
| Node stuck `NotReady` | Calico pod not running / wrong interface picked | Check `kubectl get pods -n calico-system -o wide`; confirm `nodeAddressAutodetectionV4: interface: tailscale0` is set |
| `INTERNAL-IP` shows wrong (private/local) IP | kubelet `--node-ip` not set | Repeat Step 5c on that node |
| Join token expired | More than 24h passed | `kubeadm token create --print-join-command` on the master |

---

## Reference values used in this example

- Master Tailscale IP: `100.78.23.85`
- Worker Tailscale IP: `100.94.52.116`
- API server port: `6443`
- Pod network CIDR: `192.168.0.0/16`
- Calico encapsulation: `VXLAN`
- Calico MTU: `1350`
- Kubernetes version (all nodes): `v1.31.14`
