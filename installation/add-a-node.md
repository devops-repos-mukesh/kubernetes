# Adding a New Node to the Cluster (kubeadm + Tailscale + Calico)

This documents the exact process for this cluster setup:

- **Master**: `mukesharma` (laptop) — Tailscale IP `100.78.23.85`
- **Networking**: Tailscale VPN connects all nodes (no port-forwarding/public IP needed)
- **CNI**: Calico, pinned to the `tailscale0` interface
- **Kubernetes version**: v1.31.14 (pinned/held on all nodes)

Use this same process for any new worker node (EC2, another laptop, VPS, etc.) — just repeat it for each new machine.

---

## Prerequisites on the new node

1. **Join the same Tailscale network (tailnet)**
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up
   ```
   Get its Tailscale IP:
   ```bash
   tailscale ip -4
   ```
   Note this IP down — you'll need it later for the `--node-ip` step.

2. **Install a container runtime (containerd)**
   ```bash
   sudo apt-get update
   sudo apt-get install -y containerd
   sudo systemctl enable --now containerd
   ```

3. **Disable swap** (kubeadm requires this)
   ```bash
   sudo swapoff -a
   sudo sed -i '/ swap / s/^/#/' /etc/fstab
   ```

4. **Install kubeadm, kubelet, kubectl — matching the cluster's version (v1.31.x)**

   ⚠️ **Critical**: newer Ubuntu images often ship a much newer Kubernetes repo by default (e.g. v1.34). If the node's kubeadm version is too far ahead of the control plane, `kubeadm join` fails with:
   ```
   this version of kubeadm only supports deploying clusters with control plane version >= X
   ```
   Always explicitly set up the **v1.31** repo:

   ```bash
   sudo mkdir -p /etc/apt/keyrings
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
     sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
     sudo tee /etc/apt/sources.list.d/kubernetes.list

   sudo apt-get update
   ```

   Check what versions are available and pick the one matching the master (here: `1.31.14-1.1`):
   ```bash
   apt-cache madison kubeadm
   ```

   Install the exact matching version. If the node already has a newer version installed and/or held, force the downgrade:
   ```bash
   sudo apt-mark unhold kubelet kubeadm kubectl 2>/dev/null || true

   sudo apt-get install -y --allow-downgrades --allow-change-held-packages \
     kubelet=1.31.14-1.1 kubeadm=1.31.14-1.1 kubectl=1.31.14-1.1

   sudo apt-mark hold kubelet kubeadm kubectl
   ```

   Verify:
   ```bash
   kubeadm version
   kubelet --version
   ```
   Both should report `v1.31.14`.

---

## Step 1 — Get a fresh join command from the master

Tokens expire after 24 hours, so always generate a fresh one before joining a new node.

On the **master** (`mukesharma`):
```bash
kubeadm token create --print-join-command
```

Copy the full output, e.g.:
```
kubeadm join 100.78.23.85:6443 --token <new-token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

## Step 2 — Join the new node

On the **new node**, run the exact command from Step 1:
```bash
sudo kubeadm join 100.78.23.85:6443 \
  --token <new-token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

## Step 3 — Pin kubelet to the node's Tailscale IP

This ensures the node advertises its Tailscale IP instead of its local/private network IP (e.g. an EC2 private `172.31.x.x` address), which is required for the cluster to route pod traffic correctly over Tailscale.

On the **new node**:
```bash
sudo mkdir -p /etc/systemd/system/kubelet.service.d/
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service.d/20-tailscale.conf
[Service]
Environment="KUBELET_EXTRA_ARGS=--node-ip=<new-node-tailscale-ip>"
EOF

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Replace `<new-node-tailscale-ip>` with the IP you noted in Prerequisites step 1.

---

## Step 4 — Verify from the master

On the **master**:
```bash
kubectl get nodes -o wide
```

Check:
- The new node appears and eventually shows `STATUS: Ready`
  (it starts `NotReady` until Calico's pod finishes starting on it — this can take 30–90 seconds)
- Its `INTERNAL-IP` column shows the **Tailscale IP**, not a private/local network IP

Also confirm Calico deployed correctly onto it:
```bash
kubectl get pods -n calico-system -o wide
```
Look for a `calico-node-xxxxx` pod running on the new node, status `Running`.

---

## Troubleshooting quick reference

| Symptom | Likely cause | Fix |
|---|---|---|
| `kubeadm join` fails: "control plane version >= X" | Worker's kubeadm/kubelet version too new vs master | Reinstall matching `v1.31.x` packages (see Prerequisites step 4) |
| `apt-get install` fails: "Held packages were changed" | Packages are on `apt-mark hold` | Add `--allow-change-held-packages` and `--allow-downgrades` to the install command |
| Join fails: "Port 6443 is in use" / manifests already exist | Leftover state from a previous cluster on that node | Run `sudo kubeadm reset -f`, then manually clean `/etc/kubernetes`, `/var/lib/etcd`, `~/.kube`, `/etc/cni/net.d`, and flush iptables (`sudo iptables -F && sudo iptables -t nat -F`) |
| Node stuck `NotReady` after joining | Calico pod not running / picked wrong network interface | Check `kubectl get pods -n calico-system -o wide`; confirm `custom-resources.yaml` on master has `nodeAddressAutodetectionV4: interface: tailscale0` |
| `INTERNAL-IP` shows the wrong (private/local) IP | kubelet `--node-ip` not set | Repeat Step 3 on that node |
| Token expired | More than 24h since token was created | Generate a new one: `kubeadm token create --print-join-command` |

---

## Reference: current cluster values

- Master Tailscale IP: `100.78.23.85`
- API server port: `6443`
- Pod network CIDR: `192.168.0.0/16`
- Calico encapsulation: `VXLAN`
- Calico MTU: `1350`
- Kubernetes version (all nodes): `v1.31.14`
