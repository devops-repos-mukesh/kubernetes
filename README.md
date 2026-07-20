# Kubernetes: A Complete Guide

A practical, production-oriented reference covering containers, core Kubernetes objects, operations, security, networking, Helm, observability, troubleshooting, advanced patterns, and EKS specifics.

## Table of Contents

1. [Containers & Docker Basics](#1-containers--docker-basics)
2. [Kubernetes Architecture](#2-kubernetes-architecture)
3. [Pods](#3-pods)
4. [ReplicaSets](#4-replicasets)
5. [Deployments](#5-deployments)
6. [Services](#6-services)
7. [Namespaces](#7-namespaces)
8. [ConfigMaps & Secrets](#8-configmaps--secrets)
9. [Volumes & Persistent Storage](#9-volumes--persistent-storage)
10. [Ingress](#10-ingress)
11. [Scheduling (Affinity, Taints, Tolerations)](#11-scheduling-affinity-taints-tolerations)
12. [Health Checks](#12-health-checks)
13. [Resource Requests & Limits](#13-resource-requests--limits)
14. [Autoscaling](#14-autoscaling)
15. [RBAC & Security](#15-rbac--security)
16. [Networking & CNI](#16-networking--cni)
17. [Helm](#17-helm)
18. [Monitoring & Logging](#18-monitoring--logging)
19. [Troubleshooting](#19-troubleshooting)
20. [Advanced Kubernetes (CRDs, Operators, GitOps, Service Mesh)](#20-advanced-kubernetes-crds-operators-gitops-service-mesh)
21. [EKS-Specific Concepts & Production Best Practices](#21-eks-specific-concepts--production-best-practices)

---

## 1. Containers & Docker Basics

### What is a container?
A container is a lightweight, isolated unit of software that packages application code with all its dependencies (libraries, binaries, config) so it runs consistently across environments. Unlike VMs, containers share the host OS kernel and use Linux primitives:

- **Namespaces** — isolate what a process can *see* (PID, net, mnt, uts, ipc, user).
- **cgroups (control groups)** — limit what a process can *use* (CPU, memory, I/O).
- **Union filesystems (overlayfs)** — layer read-only image layers with a writable container layer.

### Containers vs Virtual Machines
| Aspect | Container | VM |
|---|---|---|
| Isolation | Process-level (shared kernel) | Full OS (hypervisor) |
| Startup time | Seconds or less | Minutes |
| Size | MBs | GBs |
| Density | High | Lower |
| Security boundary | Weaker (shared kernel) | Stronger |

### Docker core concepts
- **Image** — immutable, layered filesystem + metadata built from a `Dockerfile`.
- **Container** — a running instance of an image.
- **Registry** — stores/distributes images (Docker Hub, ECR, GCR, Harbor).
- **Dockerfile** — declarative build recipe.

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
USER node
EXPOSE 3000
ENTRYPOINT ["node", "dist/index.js"]
```

### Key Docker commands
```bash
docker build -t myapp:1.0 .
docker run -d -p 8080:3000 --name myapp myapp:1.0
docker ps -a
docker logs -f myapp
docker exec -it myapp sh
docker images
docker push myrepo/myapp:1.0
docker network create mynet
docker volume create myvol
```

### Best practices
- Use **multi-stage builds** to keep final images small.
- Prefer minimal base images (`alpine`, `distroless`) to shrink attack surface.
- Don't run as root (`USER` directive).
- Pin base image versions/digests for reproducibility.
- Use `.dockerignore` to avoid bloating the build context.
- One process/concern per container; container should be immutable and stateless where possible.
- Layer caching: order Dockerfile instructions from least to most frequently changing.

### Why Kubernetes?
Docker (or another container runtime) runs a *single* container on a *single* host. Kubernetes orchestrates **many containers across many hosts**: scheduling, self-healing, scaling, service discovery, rolling updates, and declarative desired-state management — problems that show up the moment you run containers in production at any scale.

---
## 2. Kubernetes Architecture

Kubernetes is a **declarative, control-loop-based orchestration system**. You declare desired state; controllers continuously reconcile actual state to match it.

### Control Plane components (the "brain")
- **kube-apiserver** — the front door. Exposes the REST API, validates/authenticates/authorizes requests, and is the *only* component that talks to etcd directly. Stateless, horizontally scalable.
- **etcd** — distributed, strongly-consistent key-value store. Holds *all* cluster state (objects, config). The source of truth — back it up religiously.
- **kube-scheduler** — watches for unscheduled Pods and assigns them to Nodes based on resource requests, affinity/anti-affinity, taints/tolerations, and constraints.
- **kube-controller-manager** — runs core control loops (controllers) as a single binary: Node controller, ReplicaSet controller, Endpoints controller, Job controller, ServiceAccount controller, etc. Each controller watches the API server and drives actual state toward desired state.
- **cloud-controller-manager** — integrates with cloud provider APIs (load balancers, node lifecycle, routes) — separates cloud-specific logic from core Kubernetes.

### Node (worker) components
- **kubelet** — the primary "node agent." Registers the node, watches the API server for Pods assigned to it, and ensures containers described in PodSpecs are running and healthy via the container runtime.
- **kube-proxy** — maintains network rules on each node (iptables/IPVS) implementing the Service abstraction (load-balancing traffic to Pods).
- **Container runtime** — actually runs containers, via the **Container Runtime Interface (CRI)**. Modern clusters use **containerd** or **CRI-O** (Docker Engine itself is deprecated as a direct runtime since v1.24 — `dockershim` was removed).

### The reconciliation (control loop) pattern
```
 desired state (etcd, via API)  <---compare--->  observed/actual state
              |                                          |
              +---------- controller reconciles ---------+
```
This "watch, diff, act" loop is the fundamental pattern behind every controller, Deployment, ReplicaSet, and custom Operator in Kubernetes.

### High-level request flow
1. `kubectl apply -f deploy.yaml` → sends REST request to **kube-apiserver**.
2. API server authenticates, authorizes (RBAC), runs admission controllers, validates schema, persists object to **etcd**.
3. **kube-scheduler** notices unscheduled Pods, picks a Node, writes the binding back via the API server.
4. **kubelet** on that Node sees the assignment, pulls images, and asks the **container runtime** (via CRI) to start containers.
5. **kube-proxy** updates networking rules so Services can route to the new Pod.
6. Relevant **controllers** keep reconciling to maintain desired replica count, health, etc.

### Cluster topology
- **Control plane nodes**: typically 3 (or a multiple of 2n+1) for etcd quorum/HA, often not schedulable for regular workloads.
- **Worker nodes**: run application Pods.
- Managed offerings (EKS, GKE, AKS) hide/manage the control plane for you.

---

## 3. Pods

A **Pod** is the smallest deployable unit in Kubernetes — one or more containers that share:
- **Network namespace** — same IP address, same `localhost`, containers talk to each other over `localhost:<port>`.
- **Storage volumes** — can share mounted volumes.
- **Lifecycle** — scheduled together, live and die together.

Pods are **ephemeral and disposable** — you almost never create bare Pods in production; a higher-level controller (Deployment, StatefulSet, Job, DaemonSet) manages them.

### Multi-container pod patterns
- **Sidecar** — helper container augmenting the main container (e.g., log shipper, Envoy proxy in a service mesh).
- **Ambassador** — proxies network connections for the main container.
- **Adapter** — normalizes output (e.g., transforming logs/metrics to a standard format).
- **Init containers** — run to completion *before* app containers start (e.g., wait for a dependency, run migrations, fetch config).

### Basic Pod manifest
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  initContainers:
    - name: init-db-check
      image: busybox:1.36
      command: ["sh", "-c", "until nc -z db 5432; do sleep 2; done"]
  containers:
    - name: myapp
      image: myrepo/myapp:1.0
      ports:
        - containerPort: 3000
      env:
        - name: ENV
          value: "production"
      resources:
        requests: { cpu: "100m", memory: "128Mi" }
        limits: { cpu: "500m", memory: "256Mi" }
```

### Pod lifecycle phases
`Pending` → `Running` → `Succeeded` / `Failed` (plus `Unknown` if the node is unreachable).

Container states within a Pod: `Waiting`, `Running`, `Terminated` — visible via `kubectl describe pod`.

### Useful commands
```bash
kubectl get pods -o wide
kubectl describe pod myapp
kubectl logs myapp -c myapp --previous
kubectl exec -it myapp -- sh
kubectl delete pod myapp
```

### Key facts
- Pod IPs are **not stable** — a new Pod gets a new IP. Never hardcode Pod IPs; use Services.
- Labels + selectors are how controllers/Services find Pods — not names.
- A Pod is a single scheduling unit: all its containers land on the **same Node**.

---

## 4. ReplicaSets

A **ReplicaSet (RS)** ensures a specified number of identical Pod replicas are running at all times. It uses a **label selector** to identify which Pods it manages, and continuously reconciles: too few → create Pods; too many → delete Pods.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myrepo/myapp:1.0
```

### Key points
- You almost **never create a ReplicaSet directly** — Deployments manage ReplicaSets for you and add rolling-update/rollback capability on top.
- The selector is immutable once created.
- If you manually delete a Pod owned by a ReplicaSet, the RS controller notices the mismatch and creates a replacement — this is the self-healing property in action.
- `kubectl get rs`, `kubectl describe rs <name>` to inspect.

---

## 5. Deployments

A **Deployment** is the standard way to manage stateless applications. It manages ReplicaSets, which manage Pods — giving you declarative updates, rollbacks, and scaling.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myrepo/myapp:1.2.0
          ports:
            - containerPort: 3000
          readinessProbe:
            httpGet: { path: /healthz, port: 3000 }
            initialDelaySeconds: 5
```

### Deployment strategies
- **RollingUpdate** (default) — gradually replaces old Pods with new ones, controlled by:
  - `maxUnavailable` — how many Pods can be down during the rollout.
  - `maxSurge` — how many extra Pods can be created above `replicas` during rollout.
- **Recreate** — kills all old Pods before creating new ones (downtime, but avoids two versions running simultaneously — useful for apps that can't run mixed versions).

### Rollouts and rollbacks
```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp
kubectl rollout undo deployment/myapp --to-revision=2
kubectl set image deployment/myapp myapp=myrepo/myapp:1.3.0
kubectl scale deployment myapp --replicas=5
```

### How rollout works internally
1. Changing the Pod template creates a **new ReplicaSet** (old RS scaled down, not deleted — enables rollback).
2. Deployment controller incrementally scales the new RS up and old RS down per the strategy.
3. Old ReplicaSets are kept (up to `revisionHistoryLimit`) purely as rollback history — they run 0 Pods.

### Other workload controllers (for contrast)
- **StatefulSet** — stable network identity (`pod-0`, `pod-1`...) and stable per-Pod persistent storage; for stateful apps like databases, Kafka, etc. Pods are created/deleted in order.
- **DaemonSet** — runs exactly one Pod per (matching) Node; used for node agents like log collectors, CNI plugins, monitoring agents.
- **Job** — runs Pods to completion for a finite task.
- **CronJob** — runs Jobs on a schedule.

---

## 6. Services

Pods are ephemeral and their IPs change. A **Service** provides a stable virtual IP (ClusterIP) and DNS name that load-balances traffic to a dynamic set of Pods selected by labels.

### Service types
| Type | Purpose |
|---|---|
| `ClusterIP` (default) | Internal-only stable IP, reachable within the cluster |
| `NodePort` | Exposes the Service on a static port (30000–32767) on every Node's IP |
| `LoadBalancer` | Provisions an external cloud load balancer (e.g., AWS NLB/ALB via ELB integration) pointing at the Service |
| `ExternalName` | Maps a Service to an external DNS name (CNAME), no proxying |
| `Headless` (`clusterIP: None`) | No load-balancing/virtual IP — DNS returns Pod IPs directly; used by StatefulSets for stable per-Pod addressing |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - port: 80          # Service port
      targetPort: 3000  # container port
      protocol: TCP
```

### How Services work
- **Endpoints/EndpointSlices** — automatically populated with the IPs of Pods matching the selector; updated as Pods come and go.
- **kube-proxy** implements the actual load-balancing on each node using `iptables` or `IPVS` rules (or eBPF with some CNIs like Cilium), intercepting traffic to the Service's virtual IP and DNAT-ing to a backend Pod.
- **DNS**: CoreDNS gives every Service a DNS name: `<service>.<namespace>.svc.cluster.local`. From within the same namespace, just `<service>` works.

### Service discovery example
```bash
# From inside another pod in the same namespace:
curl http://myapp-svc/           # short name
curl http://myapp-svc.prod.svc.cluster.local/   # FQDN
```

### Key facts
- Services select Pods via **label selectors** — decoupling consumers from producers.
- A Service without a `selector` can be manually bound to external endpoints.
- `LoadBalancer` on cloud providers typically also creates a `NodePort` + `ClusterIP` under the hood.
- Session affinity (`sessionAffinity: ClientIP`) can pin a client to the same backend Pod if needed.

---

## 7. Namespaces

**Namespaces** partition a single cluster into multiple virtual clusters — a scoping mechanism for names, RBAC, quotas, and network policies. They do **not** provide compute/node isolation by themselves.

### Built-in namespaces
- `default` — objects with no namespace specified.
- `kube-system` — control plane / system components (CoreDNS, kube-proxy, etc).
- `kube-public` — readable by all, including unauthenticated users.
- `kube-node-lease` — node heartbeat lease objects.

```bash
kubectl create namespace staging
kubectl get pods -n staging
kubectl config set-context --current --namespace=staging   # switch default ns for this context
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    env: staging
```

### What's namespaced vs cluster-scoped
- **Namespaced**: Pods, Deployments, Services, ConfigMaps, Secrets, PVCs, Roles, RoleBindings, Jobs.
- **Cluster-scoped**: Nodes, PersistentVolumes, StorageClasses, ClusterRoles, ClusterRoleBindings, Namespaces themselves, CustomResourceDefinitions.

### Common uses
- Separate environments (`dev`, `staging`, `prod`) or teams within one cluster.
- Apply **ResourceQuotas** and **LimitRanges** per namespace to cap resource consumption.
- Scope **RBAC** (Roles/RoleBindings) and **NetworkPolicies** per team/app.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
```

### Best practices
- Use namespaces to establish blast-radius boundaries, not as a security-only mechanism (they're not a hard security boundary — combine with RBAC + NetworkPolicy).
- Cross-namespace DNS: `service.namespace.svc.cluster.local`.

---

## 8. ConfigMaps & Secrets

Both decouple configuration from container images, following the 12-factor app principle of externalized config.

### ConfigMaps — non-sensitive configuration
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  LOG_LEVEL: "info"
  APP_MODE: "production"
  app.properties: |
    retries=3
    timeout=30s
```

```bash
kubectl create configmap myapp-config --from-literal=LOG_LEVEL=info
kubectl create configmap myapp-config --from-file=app.properties
```

### Secrets — sensitive data (base64-encoded, not encrypted by default)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=   # base64 of "password123"
```

```bash
kubectl create secret generic myapp-secret --from-literal=DB_PASSWORD=password123
kubectl create secret docker-registry regcred --docker-server=... --docker-username=... --docker-password=...
kubectl create secret tls mytls --cert=cert.pem --key=key.pem
```

Secret types include `Opaque` (generic), `kubernetes.io/dockerconfigjson` (registry creds), `kubernetes.io/tls`, `kubernetes.io/service-account-token`.

### Consuming ConfigMaps/Secrets in a Pod

**As environment variables:**
```yaml
envFrom:
  - configMapRef: { name: myapp-config }
  - secretRef: { name: myapp-secret }
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef: { name: myapp-config, key: LOG_LEVEL }
```

**As mounted volumes (preferred for larger config, and required for automatic updates):**
```yaml
volumes:
  - name: config-vol
    configMap: { name: myapp-config }
  - name: secret-vol
    secret: { secretName: myapp-secret }
containers:
  - name: myapp
    volumeMounts:
      - { name: config-vol, mountPath: /etc/config }
      - { name: secret-vol, mountPath: /etc/secrets, readOnly: true }
```

### Important nuances
- **Base64 ≠ encryption.** Secrets are only base64-encoded in etcd by default — anyone with etcd or API read access can decode them. Enable **encryption at rest** (`EncryptionConfiguration`) and restrict RBAC access to Secrets.
- Volume-mounted ConfigMaps/Secrets **auto-update** (with a short propagation delay via kubelet sync) when the source changes; env-var-injected values do **not** update without a Pod restart.
- Prefer external secret managers for production: **AWS Secrets Manager / SSM Parameter Store**, **HashiCorp Vault**, integrated via the **External Secrets Operator** or **CSI Secrets Store driver**, rather than raw Kubernetes Secrets.
- Set `immutable: true` on ConfigMaps/Secrets that shouldn't change, for performance and safety.

---

## 9. Volumes & Persistent Storage

Container filesystems are ephemeral — data is lost on restart. Kubernetes volumes solve data persistence and sharing between containers in a Pod.

### Volume types (selection)
- **emptyDir** — created when Pod starts, deleted when Pod is removed; good for scratch space/caches shared between containers in a Pod.
- **hostPath** — mounts a file/directory from the Node's filesystem (use sparingly — ties Pod to a specific node, security risk).
- **configMap / secret** — projects config as files (see above).
- **persistentVolumeClaim** — durable storage backed by a PersistentVolume (the standard for stateful workloads).
- Cloud-native: `awsElasticBlockStore` (legacy in-tree, now via CSI `ebs.csi.aws.com`), `efs.csi.aws.com`, etc.

### PersistentVolume (PV) & PersistentVolumeClaim (PVC) model
- **PersistentVolume (PV)** — a cluster-scoped piece of actual storage (provisioned statically by an admin, or dynamically by a StorageClass).
- **PersistentVolumeClaim (PVC)** — a namespaced *request* for storage by a user/Pod ("I need 10Gi, ReadWriteOnce, SSD").
- **StorageClass** — defines *how* to dynamically provision PVs (which provisioner, parameters, reclaim policy).

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-data
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: gp3
  resources:
    requests: { storage: 10Gi }
---
apiVersion: v1
kind: Pod
metadata: { name: myapp }
spec:
  containers:
    - name: myapp
      image: myrepo/myapp:1.0
      volumeMounts:
        - { name: data, mountPath: /var/lib/data }
  volumes:
    - name: data
      persistentVolumeClaim: { claimName: myapp-data }
```

### Access modes
- **ReadWriteOnce (RWO)** — one node can mount read-write (most block storage, e.g., EBS).
- **ReadOnlyMany (ROX)** — many nodes, read-only.
- **ReadWriteMany (RWX)** — many nodes, read-write (needs a shared filesystem like EFS, NFS).
- **ReadWriteOncePod** — restricts to a single Pod (newer, stricter guarantee than RWO).

### Reclaim policies
- `Delete` — PV (and often underlying storage) deleted when PVC is deleted (default for dynamically provisioned).
- `Retain` — PV and data kept after PVC deletion, requires manual cleanup — safer for critical data.

### CSI (Container Storage Interface)
Modern Kubernetes storage integrations are all done via CSI drivers (e.g., `ebs.csi.aws.com`, `efs.csi.aws.com`) — a standard plugin interface decoupling Kubernetes core from vendor-specific storage code, supporting dynamic provisioning, snapshots, resizing, and topology-aware scheduling.

### StatefulSets + storage
StatefulSets use `volumeClaimTemplates` to give **each replica its own PVC** (`data-myapp-0`, `data-myapp-1`, ...), preserved across Pod rescheduling — essential for databases.

---

## 10. Ingress

A **Service** of type `LoadBalancer` gives you one external LB per Service — expensive and unwieldy at scale. **Ingress** provides HTTP(S) layer-7 routing (host/path-based) into the cluster through a single entry point.

### Key pieces
- **Ingress resource** — declarative rules (host/path → backend Service).
- **Ingress Controller** — the actual implementation that watches Ingress objects and configures a reverse proxy/load balancer (NGINX Ingress Controller, AWS Load Balancer Controller → ALB, Traefik, HAProxy, Istio Gateway). **An Ingress object does nothing without a controller running in the cluster.**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["app.example.com"]
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port: { number: 80 }
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port: { number: 80 }
```

### Features Ingress typically provides
- Host-based and path-based routing.
- TLS termination (via a referenced `Secret` of type `kubernetes.io/tls`).
- Rewrite rules, redirects, rate limiting, auth (via controller-specific annotations).

### Gateway API (the modern successor)
The **Gateway API** (`gateway.networking.k8s.io`) is a newer, more expressive, role-oriented standard (`GatewayClass`, `Gateway`, `HTTPRoute`) designed to replace Ingress long-term, with better support for multiple teams, protocols beyond HTTP, and traffic splitting — increasingly the recommended approach for new clusters.

### On EKS
The **AWS Load Balancer Controller** watches Ingress objects and provisions an **Application Load Balancer (ALB)** automatically, or a Network Load Balancer for Service type `LoadBalancer` — see the EKS section.

---

## 11. Scheduling (Affinity, Taints, Tolerations)

The **kube-scheduler** decides which Node a Pod runs on in two phases: **filtering** (which nodes are feasible) and **scoring** (rank feasible nodes, pick the best).

### nodeSelector — simplest constraint
```yaml
spec:
  nodeSelector:
    disktype: ssd
```

### Node Affinity — expressive nodeSelector
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - { key: disktype, operator: In, values: ["ssd"] }
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - { key: zone, operator: In, values: ["us-east-1a"] }
```
- `required...` = hard constraint (must match).
- `preferred...` = soft constraint (best-effort, weighted).

### Pod Affinity / Anti-Affinity — relative to other Pods
```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels: { app: myapp }
          topologyKey: "kubernetes.io/hostname"   # spread replicas across nodes
```
Common use: spread replicas across nodes/zones (anti-affinity) for HA, or co-locate latency-sensitive Pods (affinity, e.g., app + cache on same node/zone).

### Taints & Tolerations — repel vs allow
Taints are applied to **Nodes** and repel Pods unless the Pod has a matching **toleration**. This is the inverse of affinity (affinity is Pod choosing Node; taints are Node rejecting Pods).

```bash
kubectl taint nodes node1 dedicated=gpu:NoSchedule
```

```yaml
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
```

Taint effects:
- `NoSchedule` — won't schedule new Pods without matching toleration.
- `PreferNoSchedule` — soft version.
- `NoExecute` — evicts already-running Pods that don't tolerate it (used for node-not-ready/unreachable auto-taints, with `tolerationSeconds` to control grace period).

Common built-in taints: `node.kubernetes.io/not-ready`, `node.kubernetes.io/unreachable`, `node-role.kubernetes.io/control-plane` (keeps workloads off control-plane nodes).

### Topology Spread Constraints — finer-grained even distribution
```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels: { app: myapp }
```
Ensures Pods are evenly spread across zones/nodes rather than just avoiding co-location — more precise than podAntiAffinity for HA across failure domains.

### Other scheduling controls
- **`nodeName`** — bypasses the scheduler entirely, pins directly (rare/debug use).
- **Priority & Preemption (`PriorityClass`)** — higher-priority Pods can preempt (evict) lower-priority ones when resources are scarce.
- **Pod Disruption Budgets (PDB)** — not scheduling per se, but constrains voluntary disruptions (drains/upgrades) to keep a minimum number of Pods available.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: myapp-pdb }
spec:
  minAvailable: 2
  selector:
    matchLabels: { app: myapp }
```

---

## 12. Health Checks

Kubernetes uses **probes** on containers to know when a Pod is truly healthy, ready for traffic, or needs a restart.

### Probe types
- **Liveness probe** — "is this container alive?" Failing → kubelet **restarts** the container. Use for deadlock detection.
- **Readiness probe** — "is this container ready to serve traffic?" Failing → Pod is removed from **Service endpoints** (no restart) — traffic stops flowing until it passes again.
- **Startup probe** — "has this slow-starting container finished starting?" Disables liveness/readiness checks until it succeeds — prevents killing slow-starting apps prematurely.

### Probe mechanisms
- `httpGet` — HTTP GET, success = 2xx/3xx response.
- `tcpSocket` — TCP connection succeeds.
- `exec` — command exits 0.
- `grpc` — gRPC health-checking protocol (native since 1.24+).

```yaml
containers:
  - name: myapp
    image: myrepo/myapp:1.0
    startupProbe:
      httpGet: { path: /healthz, port: 3000 }
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      httpGet: { path: /healthz, port: 3000 }
      initialDelaySeconds: 10
      periodSeconds: 10
      failureThreshold: 3
      timeoutSeconds: 2
    readinessProbe:
      httpGet: { path: /ready, port: 3000 }
      periodSeconds: 5
      failureThreshold: 3
```

### Best practices
- **`/healthz` (liveness)** should check "is the process fundamentally broken" — not downstream dependencies (a DB outage shouldn't cause a liveness-restart storm).
- **`/ready` (readiness)** *should* check critical downstream dependencies (DB connection pool, cache) — it's fine (and correct) for readiness to fail when a dependency is down.
- Always set a **startupProbe** for slow-booting apps (e.g., JVM apps) instead of inflating `initialDelaySeconds` on liveness.
- Tune `periodSeconds`/`failureThreshold` to avoid flapping under transient load spikes.
- Missing readiness probes mean Pods can receive traffic before they're actually ready — a very common cause of 502s during rollouts.

---

## 13. Resource Requests & Limits

Kubernetes uses **requests** and **limits** per container to drive scheduling decisions and enforce runtime boundaries.

- **Requests** — what the scheduler guarantees/reserves. The scheduler only places a Pod on a Node that has ≥ requested resources available. Also used to compute the Pod's **QoS class**.
- **Limits** — the hard ceiling a container cannot exceed at runtime.

```yaml
resources:
  requests:
    cpu: "250m"      # 0.25 vCPU
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### CPU vs Memory enforcement differs
- **CPU is compressible** — exceeding the limit results in **throttling** (CFS quota), not termination. The process slows down but keeps running.
- **Memory is incompressible** — exceeding the limit results in the container being **OOMKilled** (`OOMKilled` status, exit code 137) immediately.

### Quality of Service (QoS) classes
Derived automatically from how requests/limits are set — determines eviction priority under node pressure:
- **Guaranteed** — every container has `requests == limits` for both CPU and memory. Evicted last.
- **Burstable** — at least one request set, but requests ≠ limits. Evicted after BestEffort.
- **BestEffort** — no requests/limits set at all. Evicted first under resource pressure.

### LimitRange — set defaults/bounds per namespace
```yaml
apiVersion: v1
kind: LimitRange
metadata: { name: default-limits }
spec:
  limits:
    - default: { cpu: "500m", memory: "512Mi" }
      defaultRequest: { cpu: "100m", memory: "128Mi" }
      type: Container
```

### ResourceQuota — cap total consumption per namespace
(see Namespaces section above)

### Best practices
- Always set requests — unset requests hurt scheduling quality and bin-packing.
- Set memory limits conservatively (OOMKill is abrupt); consider not setting a CPU limit at all in latency-sensitive services to avoid throttling, relying on requests + node capacity instead (debated but common practice).
- Right-size using historical usage data (VPA recommendations, Prometheus metrics) rather than guessing.
- Requests too low → overcommitted nodes, noisy-neighbor problems, cascading OOM kills/evictions.
- Requests too high → poor bin-packing, wasted cluster capacity, higher cost.

---

## 14. Autoscaling

Kubernetes autoscaling operates at three layers, often used together:

### 1. Horizontal Pod Autoscaler (HPA) — scale *replica count*
Watches metrics (CPU/memory by default, or custom/external metrics via the Metrics API) and adjusts `replicas` on a Deployment/StatefulSet.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: myapp-hpa }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
    - type: Pods
      pods:
        metric: { name: http_requests_per_second }
        target: { type: AverageValue, averageValue: "100" }
```
Requires the **metrics-server** (for CPU/memory) or a **custom/external metrics adapter** (e.g., Prometheus Adapter, KEDA) for custom metrics like queue depth or request rate.

### 2. Vertical Pod Autoscaler (VPA) — scale *resource requests/limits*
Recommends or automatically adjusts a container's CPU/memory requests based on observed usage. Modes: `Off` (recommendations only), `Initial`, `Auto`/`Recreate` (restarts Pods to apply new sizing). **Don't run HPA on CPU/memory and VPA on the same metric simultaneously** — they can fight each other.

### 3. Cluster Autoscaler / Karpenter — scale *Nodes*
- **Cluster Autoscaler** — watches for unschedulable Pods (Pending due to insufficient capacity) and adds Nodes by scaling a cloud Auto Scaling Group / node group; removes underutilized Nodes when Pods could be consolidated elsewhere.
- **Karpenter** (increasingly the default on AWS/EKS) — a more modern, faster just-in-time node provisioner. Doesn't rely on pre-defined node groups; directly launches right-sized EC2 instances based on aggregate Pod requirements (instance type, arch, zone, spot vs on-demand), and consolidates/binpacks more aggressively.

### KEDA (Kubernetes Event-Driven Autoscaling)
Extends HPA to scale on external event sources — SQS queue depth, Kafka lag, cron schedules — including **scale-to-zero**, which vanilla HPA cannot do.

### Putting it together
```
Karpenter/Cluster Autoscaler  →  adds/removes Nodes
            ↑ triggers when
HPA  →  changes replica count based on load
            ↑ informs sizing via
VPA  →  recommends/sets per-Pod CPU/memory requests
```

---

## 15. RBAC & Security

### Authentication vs Authorization
1. **Authentication (AuthN)** — who are you? (client certs, bearer tokens, OIDC/SSO, service account tokens, cloud IAM integration like EKS's `aws-iam-authenticator`/access entries).
2. **Authorization (AuthZ)** — what are you allowed to do? Kubernetes' primary model is **RBAC**.
3. **Admission Control** — mutate/validate requests after AuthN/AuthZ, before persistence (e.g., inject sidecars, enforce policies via OPA Gatekeeper/Kyverno, Pod Security Admission).

### RBAC building blocks
- **Role** — namespaced set of permissions (verbs on resources).
- **ClusterRole** — same, but cluster-scoped (or reusable across namespaces).
- **RoleBinding** — grants a Role (or ClusterRole) to a subject **within a namespace**.
- **ClusterRoleBinding** — grants a ClusterRole **cluster-wide**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { namespace: staging, name: pod-reader }
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: read-pods, namespace: staging }
subjects:
  - kind: User
    name: jane@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl auth can-i delete pods --as jane@example.com -n staging
kubectl create rolebinding ... --clusterrole=view --user=jane --namespace=staging
```

### Service Accounts
Every Pod runs as a **ServiceAccount** (default: `default` in its namespace, with minimal permissions). Bind a dedicated, least-privilege ServiceAccount per workload rather than relying on `default`. On EKS, **IRSA (IAM Roles for Service Accounts)** or **EKS Pod Identity** maps a Kubernetes ServiceAccount to an AWS IAM Role, giving Pods scoped AWS permissions without static credentials.

### Pod / container security hardening
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile: { type: RuntimeDefault }
  containers:
    - name: myapp
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

### Pod Security Admission (PSA)
Replaced the deprecated PodSecurityPolicy. Enforced via namespace labels with three standard levels:
- `privileged` — unrestricted.
- `baseline` — blocks known privilege escalations, minimally restrictive.
- `restricted` — heavily hardened (no root, no privilege escalation, dropped capabilities, seccomp required) — recommended baseline for most workloads.

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

### Network-layer security
- **NetworkPolicy** — firewall rules for Pod-to-Pod traffic (see Networking section).
- **Secrets encryption at rest** in etcd (`EncryptionConfiguration`).
- **Image scanning** (Trivy, ECR scanning, Grype) in CI to catch CVEs before deploy.
- **Admission policy engines** — **OPA Gatekeeper** or **Kyverno** to enforce org policy (e.g., "no `:latest` tags," "must have resource limits," "no privileged containers") as code.
- **Audit logging** on the API server for compliance and forensics.

### Defense-in-depth summary
```
Cloud IAM  →  Cluster AuthN  →  RBAC AuthZ  →  Admission Control (PSA/OPA)
    →  NetworkPolicy  →  Pod SecurityContext  →  Image scanning  →  Runtime security (Falco)
```

---

## 16. Networking & CNI

Kubernetes networking is governed by the **Kubernetes Networking Model**, with four fundamental rules:
1. Every Pod gets its **own unique IP** (no NAT between Pods).
2. Pods on any Node can communicate with **all** Pods on all Nodes without NAT.
3. Agents on a Node (kubelet) can communicate with all Pods on that Node.
4. Containers within a Pod share a network namespace (communicate via `localhost`).

### CNI (Container Network Interface)
Kubernetes itself does **not** implement Pod networking — it delegates to a **CNI plugin**, which sets up the actual networking (assigns IPs, configures routes) when Pods are created. Popular CNIs:
- **AWS VPC CNI** — (default on EKS) assigns Pods real VPC IP addresses directly from ENIs — best AWS integration (security groups for Pods, VPC routing) but IP-address-per-Pod can exhaust subnet capacity at scale.
- **Calico** — BGP or overlay-based, strong NetworkPolicy support, widely used on-prem and in EKS as an alternative/addition.
- **Cilium** — eBPF-based, very high performance, rich L3-L7 network policy, built-in observability (Hubble), can replace kube-proxy entirely.
- **Flannel** — simple overlay network (VXLAN), minimal features, good for simple/dev clusters.

### DNS: CoreDNS
Every Service and (optionally) Pod gets a DNS record via **CoreDNS**, running as a cluster addon. Format: `<service>.<namespace>.svc.cluster.local`.

### NetworkPolicy — Pod-level firewalling
By default, **all Pods can talk to all other Pods** (flat network, fully open). NetworkPolicies (enforced by the CNI, not the API server itself) restrict this.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: api-allow-frontend, namespace: prod }
spec:
  podSelector:
    matchLabels: { app: api }
  policyTypes: ["Ingress", "Egress"]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: frontend } }
      ports:
        - { protocol: TCP, port: 8080 }
  egress:
    - to:
        - podSelector: { matchLabels: { app: db } }
      ports:
        - { protocol: TCP, port: 5432 }
```
Important: NetworkPolicy is **allow-list based** and **additive** — once *any* policy selects a Pod for a direction (ingress/egress), that direction becomes **default-deny** except for what's explicitly allowed; unselected Pods remain fully open.

### kube-proxy modes
- `iptables` (traditional, default historically) — chains of NAT rules, O(n) rule evaluation at scale.
- `IPVS` — kernel load-balancing, better performance at high Service counts.
- eBPF-based dataplanes (e.g. Cilium) can **replace kube-proxy** entirely for better performance/observability.

---

## 17. Helm

**Helm** is the de facto package manager for Kubernetes — templates raw YAML manifests into reusable, parameterized, versioned **charts**.

### Core concepts
- **Chart** — a package of templated Kubernetes manifests + metadata (`Chart.yaml`) + default config (`values.yaml`).
- **Release** — a deployed instance of a chart in a cluster (you can install the same chart multiple times as different releases).
- **Values** — the configuration layer that fills in chart templates (`values.yaml`, `--set`, or `-f custom-values.yaml`).
- **Repository** — an HTTP(S) location hosting packaged charts (like a package registry; also supports OCI registries like ECR).

### Chart structure
```
mychart/
├── Chart.yaml
├── values.yaml
├── charts/            # subcharts/dependencies
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── _helpers.tpl    # reusable template snippets
    └── NOTES.txt        # post-install usage notes
```

### Template example
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels: { app: {{ .Release.Name }} }
  template:
    metadata:
      labels: { app: {{ .Release.Name }} }
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources: {{ toYaml .Values.resources | nindent 12 }}
```

```yaml
# values.yaml
replicaCount: 3
image:
  repository: myrepo/myapp
  tag: "1.2.0"
resources:
  requests: { cpu: 100m, memory: 128Mi }
```

### Common commands
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx

helm install myrelease ./mychart -f values-prod.yaml
helm upgrade myrelease ./mychart --set image.tag=1.3.0
helm rollback myrelease 2
helm uninstall myrelease

helm template myrelease ./mychart      # render locally without installing
helm lint ./mychart                    # validate chart
helm diff upgrade myrelease ./mychart  # (plugin) preview changes before applying
```

### Why Helm over raw YAML/kustomize
- Templating + conditionals/loops (`{{ if }}`, `{{ range }}`) for environment-specific variation.
- Dependency management (subcharts — e.g., an app chart depending on a Redis chart).
- Release lifecycle: install/upgrade/rollback as atomic, versioned operations with history.
- Hooks (`pre-install`, `post-upgrade`, etc.) for migrations and setup tasks.

**Kustomize** (built into `kubectl -k`) is the templating-free alternative — overlays patch base YAML declaratively without a templating language; often preferred for GitOps because output is plain, diffable YAML. Many teams combine Helm (for third-party charts) with Kustomize (for internal overlays), or use `helm template | kustomize` pipelines.

---

## 18. Monitoring & Logging

### The three pillars of observability
1. **Metrics** — numeric time-series (CPU, request rate, latency, error rate).
2. **Logs** — discrete event records.
3. **Traces** — distributed request flow across services.

### Metrics stack (the standard)
- **metrics-server** — lightweight, in-memory CPU/memory metrics used by `kubectl top` and HPA. Not for long-term storage/alerting.
- **Prometheus** — pulls (scrapes) metrics from instrumented apps and Kubernetes components (kubelet, cAdvisor, kube-state-metrics, node-exporter) on an interval; stores as time series; powers alerting via **Alertmanager**.
- **kube-state-metrics** — exposes the *state* of Kubernetes objects as metrics (e.g., `kube_deployment_status_replicas`, `kube_pod_status_phase`) — distinct from cAdvisor's *resource usage* metrics.
- **Grafana** — visualization/dashboards on top of Prometheus (and other data sources).
- **kube-prometheus-stack** Helm chart — the common all-in-one bundle (Prometheus Operator, Grafana, Alertmanager, exporters, dashboards).

```yaml
# Example PrometheusRule (alert)
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata: { name: myapp-alerts }
spec:
  groups:
    - name: myapp
      rules:
        - alert: HighErrorRate
          expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
          for: 10m
          labels: { severity: critical }
          annotations:
            summary: "High 5xx rate for myapp"
```

### Logging stack
Kubernetes doesn't retain logs itself — `kubectl logs` reads current container stdout/stderr from the node, lost on Pod deletion/rotation. Production logging needs a pipeline:
1. **Collection** — a node-level **DaemonSet** agent (Fluent Bit, Fluentd, Vector, or the CloudWatch agent) tails container log files (`/var/log/containers`) on each node.
2. **Shipping/processing** — parse, enrich with Pod/namespace metadata, filter.
3. **Storage & search** — Elasticsearch/OpenSearch (EFK/ELK stack), Loki (pairs with Grafana, indexes only labels — cheaper), or a cloud service (CloudWatch Logs, Datadog).

### Distributed tracing
- **OpenTelemetry** — the vendor-neutral standard for instrumentation (traces, metrics, logs); apps emit spans, an OTel Collector routes them to a backend (Jaeger, Tempo, X-Ray, Datadog, etc).

### Kubernetes-native visibility commands
```bash
kubectl top nodes
kubectl top pods -n prod
kubectl get events --sort-by=.lastTimestamp -n prod
kubectl describe pod myapp   # Events section at bottom is often the fastest debugging signal
```

### What to alert on (SRE golden signals)
**Latency, traffic, errors, saturation** — plus Kubernetes-specific signals: CrashLoopBackOff counts, Pending Pods (scheduling failures), OOMKilled counts, PVC capacity, node disk pressure, HPA at max replicas for extended periods, certificate expiry.

---

## 19. Troubleshooting

A systematic top-down approach, plus a symptom → likely-cause cheat sheet.

### General workflow
```bash
kubectl get pods -n <ns> -o wide          # status, node, restarts at a glance
kubectl describe pod <pod> -n <ns>        # Events section = usually the answer
kubectl logs <pod> -n <ns> -c <container> --previous   # logs from the crashed instance
kubectl get events -n <ns> --sort-by=.lastTimestamp
kubectl exec -it <pod> -n <ns> -- sh      # get inside and poke around
kubectl top pod <pod> -n <ns>             # is it resource-starved?
```

### Common Pod states and causes

| Symptom | Likely cause | How to confirm |
|---|---|---|
| `Pending` | Insufficient cluster resources, unsatisfiable affinity/taint, no matching node, PVC unbound | `kubectl describe pod` → Events: "FailedScheduling" |
| `ImagePullBackOff` / `ErrImagePull` | Wrong image name/tag, private registry auth missing, network egress blocked | `describe pod` Events; check `imagePullSecrets` |
| `CrashLoopBackOff` | App crashes on startup — bad config, missing env var/secret, failing migration, port already in use | `kubectl logs --previous` |
| `OOMKilled` (exit 137) | Memory limit too low, memory leak | `describe pod` shows `Reason: OOMKilled`; check limits vs actual usage |
| `Terminated: Error` (exit 1, etc.) | Application-level failure | check logs |
| Pod `Running` but not `Ready` | Readiness probe failing (dependency down, wrong port/path) | `describe pod` probe failure events |
| Stuck `Terminating` | Finalizer blocking deletion, or graceful shutdown hanging (e.g., not handling SIGTERM) | `kubectl get pod -o yaml` → check `finalizers`; check `terminationGracePeriodSeconds` |
| `Evicted` | Node under disk/memory pressure | `describe pod` shows eviction reason; check node conditions |

### Networking troubleshooting
```bash
kubectl exec -it debug-pod -- nslookup myapp-svc         # DNS resolution
kubectl exec -it debug-pod -- curl -v http://myapp-svc    # connectivity
kubectl get endpoints myapp-svc -n prod                   # are Pods actually registered?
kubectl get networkpolicy -n prod                          # unexpected default-deny?
kubectl run tmp-shell --rm -it --image=nicolaka/netshoot -- bash  # full-featured debug pod
```
- Empty `Endpoints`/`EndpointSlices` for a Service usually means the **selector doesn't match any Pod labels**, or matched Pods aren't `Ready`.

### Deployment / rollout issues
```bash
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl describe deployment myapp     # check Conditions: Progressing/Available
kubectl get rs -l app=myapp           # is the new RS stuck at 0/N ready?
```
A rollout stuck mid-way is almost always: new Pods failing readiness, or `maxUnavailable`/`maxSurge` math + insufficient cluster capacity to schedule the surge Pods.

### Node-level issues
```bash
kubectl get nodes
kubectl describe node <node>          # Conditions: MemoryPressure, DiskPressure, PIDPressure, Ready
kubectl top nodes
```

### Debugging without shell access (distroless/minimal images)
```bash
kubectl debug -it <pod> --image=busybox --target=<container>   # ephemeral debug container, shares process namespace
kubectl debug node/<node> -it --image=busybox                   # debug a node directly
```

### General debugging philosophy
1. **Events first** — `kubectl describe` surfaces scheduler/kubelet-level problems before you even touch logs.
2. **Work top-down**: Cluster → Namespace → Controller (Deployment/RS) → Pod → Container → Application.
3. **Isolate the layer**: is it scheduling, image, config, application logic, network, or storage?
4. **Reproduce minimally**: a throwaway debug Pod in the same namespace tells you a lot about DNS/network reachability without touching the real workload.

---

## 20. Advanced Kubernetes (CRDs, Operators, GitOps, Service Mesh)

### Custom Resource Definitions (CRDs)
CRDs let you extend the Kubernetes API with your own object types, so `kubectl get` / RBAC / etcd storage / watch semantics all work for domain-specific concepts (e.g., `Certificate`, `VirtualService`, `PostgresCluster`).

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata: { name: postgresclusters.db.example.com }
spec:
  group: db.example.com
  scope: Namespaced
  names: { plural: postgresclusters, singular: postgrescluster, kind: PostgresCluster }
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas: { type: integer }
                version: { type: string }
```
A CRD alone is just a schema/API — it does nothing by itself. Pairing it with a controller turns it into an **Operator**.

### Operators
The **Operator pattern** encodes human operational knowledge ("how do I safely upgrade/scale/backup this stateful system?") into a controller that watches a Custom Resource and reconciles real-world state — essentially "an SRE in software." Examples: **Prometheus Operator**, **cert-manager**, **ECK (Elastic Cloud on Kubernetes)**, **Postgres Operators** (Zalando, CloudNativePG), **Strimzi (Kafka)**. Built with frameworks like **Kubebuilder** or **Operator SDK**.

```
CRD (schema)  +  Controller (reconcile loop)  =  Operator
```

### GitOps
Git is the single source of truth for desired cluster state; an in-cluster agent continuously pulls and reconciles — no direct `kubectl apply` from CI pipelines (push-based) into production.

- **Argo CD** — declarative GitOps continuous delivery; watches Git repos, renders manifests (raw YAML/Helm/Kustomize), diffs against live cluster state, syncs (auto or manual), rich UI, drift-detection and auto-heal.
- **Flux** — similar pull-based GitOps operator, more lightweight/composable, tight Helm-controller/Kustomize-controller integration.

Benefits: full audit trail via Git history, easy rollback (`git revert`), consistent environments, drift detection/auto-correction, no long-lived CI credentials with cluster-admin access.

```
Developer → git push → Git repo (desired state)
                              ↓ (pull, poll/webhook)
                        Argo CD / Flux in-cluster
                              ↓ (diff & apply)
                        Live cluster state
```

### Service Mesh
A dedicated infrastructure layer (usually sidecar proxies, increasingly sidecar-less/ambient) handling service-to-service traffic concerns outside application code:
- **mTLS** everywhere (zero-trust networking) automatically.
- Fine-grained **traffic management** — canary/blue-green splits, retries, timeouts, circuit breaking.
- **Observability** — automatic golden-signal metrics and distributed tracing for every service call.
- **Fine-grained authorization policies** between services.

Popular implementations: **Istio** (feature-rich, sidecar or Ambient mesh mode), **Linkerd** (lightweight, simplicity-focused), **AWS App Mesh** (being phased toward native EKS/Cilium mesh options), **Cilium Service Mesh** (eBPF, increasingly sidecar-free).

### Other advanced patterns worth knowing
- **Admission webhooks** (`MutatingWebhookConfiguration` / `ValidatingWebhookConfiguration`) — custom logic hooked into the request pipeline (e.g., Istio's sidecar injector, OPA Gatekeeper's policy enforcement).
- **Multi-cluster** patterns — cluster fleets (Argo CD ApplicationSets, Fleet, Karmada) for scaling GitOps and workload placement across many clusters.
- **Progressive delivery** — Argo Rollouts / Flagger layer canary and blue-green strategies with automated metric-based analysis on top of Deployments.

---

## 21. EKS-Specific Concepts & Production Best Practices

**Amazon EKS (Elastic Kubernetes Service)** is a managed Kubernetes control plane — AWS runs and scales the API server/etcd across multiple AZs; you manage (or let AWS manage) the data plane.

### Control plane
- Fully managed, multi-AZ, auto-scaled by AWS; you don't see or SSH into master nodes.
- Upgrades are explicit and version-skewed one minor version at a time (`aws eks update-cluster-version`) — plan for **kubectl/API deprecations** every release; check `kubectl convert`/deprecation guides before upgrading.
- Control plane logging (API server, audit, authenticator, controller manager, scheduler) ships to **CloudWatch Logs** — enable audit logs for compliance.

### Data plane (node) options
| Option | Description |
|---|---|
| **Managed Node Groups** | AWS-managed EC2 Auto Scaling Groups; handles node provisioning/updates/draining, still your AMI/lifecycle choices |
| **Self-managed nodes** | Full control over AMI, launch template, lifecycle — more ops burden |
| **Fargate** | Serverless — no nodes to manage at all; each Pod runs in its own isolated micro-VM. No DaemonSets, no `hostPath`/hostNetwork, higher per-Pod cost, but zero node patching |
| **Karpenter** | Recommended modern autoscaling data-plane provisioner (see Autoscaling section) — replaces/augments Cluster Autoscaler + managed node groups for faster, more efficient bin-packing |

### Networking on EKS
- **AWS VPC CNI** is the default: Pods get real VPC IP addresses (ENI-backed) — enables Security Groups for Pods and direct VPC routing/peering, but consumes VPC IP space fast; mitigate with **prefix delegation** (assign /28 prefixes instead of individual IPs per ENI) or secondary CIDR ranges.
- **Security Groups for Pods** — attach EC2 security groups directly to specific Pods (via `SecurityGroupPolicy`) for fine-grained network isolation, complementing NetworkPolicy.
- Alternative: switch to **Cilium** or **Calico** as the CNI for advanced NetworkPolicy, eBPF dataplane, or to avoid VPC IP exhaustion (overlay mode).
- **AWS Load Balancer Controller** — the standard add-on that turns:
  - `Ingress` objects → **Application Load Balancer (ALB)**
  - `Service type=LoadBalancer` (with annotation) → **Network Load Balancer (NLB)**

### Identity & access
- **IRSA (IAM Roles for Service Accounts)** — OIDC-federated, per-ServiceAccount IAM role assumption; the traditional/still-widely-used way to give Pods scoped AWS permissions.
- **EKS Pod Identity** — newer, simpler alternative to IRSA (no OIDC provider federation dance, simpler trust policy, supported by the `eks-pod-identity-agent` add-on) — increasingly preferred for new setups.
- **`aws-auth` ConfigMap / EKS Access Entries** — map IAM principals (users/roles) to Kubernetes RBAC identities; **EKS Access Entries** (the newer API-managed approach) is now preferred over hand-editing the `aws-auth` ConfigMap.

### Storage on EKS
- **EBS CSI driver** — default for `ReadWriteOnce` block storage (gp3 recommended over gp2 for cost/performance); must be installed as an add-on (not built-in by default on newer versions).
- **EFS CSI driver** — for `ReadWriteMany` shared storage across AZs/Pods.

### EKS add-ons
AWS-managed lifecycle for core components — `vpc-cni`, `coredns`, `kube-proxy`, `aws-ebs-csi-driver`, `eks-pod-identity-agent`, `metrics-server` — installable/upgradable via `eksctl`/Console/Terraform rather than manual manifests, keeping them in sync with cluster version.

### Production best-practice checklist
- **Multi-AZ everything**: spread node groups and Pods (topology spread constraints) across ≥3 AZs; use PodDisruptionBudgets so AZ-level or upgrade-driven disruption doesn't take down a whole service.
- **Right-size nodes and Pods**: use Karpenter + VPA recommendations instead of static guesses; avoid giant "one-size" node groups.
- **Cost control**: mix Spot (via Karpenter, for fault-tolerant/stateless workloads) and On-Demand; enable **Cluster/Karpenter consolidation**; use **gp3** over gp2; right-size requests to improve bin-packing.
- **Upgrade discipline**: track EKS's ~4-per-year minor releases and end-of-support dates; test against the Kubernetes deprecated-API list before every upgrade; upgrade control plane, then node groups/add-ons, then workloads' API versions.
- **GitOps for everything**: cluster config, add-ons, and application manifests all through Argo CD/Flux + IaC (Terraform/`eksctl`/CDK) — no manual `kubectl apply` to prod.
- **Least privilege**: IRSA/Pod Identity per workload (never broad node-instance-role permissions shared by all Pods); RBAC scoped per team/namespace; Access Entries instead of cluster-admin for humans.
- **Guardrails**: OPA Gatekeeper/Kyverno policies (no `:latest`, mandatory resource limits, no privileged Pods), Pod Security Admission `restricted` by default.
- **Observability**: Container Insights or the kube-prometheus-stack + Grafana; ship logs via Fluent Bit to CloudWatch or OpenSearch; alert on the golden signals plus EKS-specific signals (Karpenter provisioning failures, IRSA/auth errors, add-on health).
- **Backup/DR**: Velero (with the AWS plugin) for cluster resource + PV snapshot backups; test restore, not just backup.
- **Secrets**: prefer AWS Secrets Manager/SSM Parameter Store via the **External Secrets Operator** or **Secrets Store CSI Driver** over raw Kubernetes Secrets for anything sensitive.
- **Private clusters / endpoint access**: restrict the EKS API server's public endpoint (private-only or CIDR-restricted) for production; run CI/CD from within the VPC or over VPN/Direct Connect where possible.

---

## Quick Reference: Essential kubectl Commands

```bash
# Context & config
kubectl config get-contexts
kubectl config use-context <ctx>
kubectl config set-context --current --namespace=<ns>

# Inspecting
kubectl get all -n <ns>
kubectl get <resource> -o yaml
kubectl explain deployment.spec.strategy

# Applying & managing
kubectl apply -f manifest.yaml
kubectl diff -f manifest.yaml
kubectl delete -f manifest.yaml

# Debugging
kubectl describe <resource> <name>
kubectl logs -f <pod> [-c container] [--previous]
kubectl exec -it <pod> -- sh
kubectl port-forward svc/<name> 8080:80
kubectl get events --sort-by=.lastTimestamp

# Scaling & rollouts
kubectl scale deployment <name> --replicas=N
kubectl rollout status/history/undo deployment/<name>
```

---

## Suggested Learning Path

1. Docker basics → build and run a container locally.
2. Pods, Deployments, Services → deploy a simple app on a local cluster (kind/minikube).
3. ConfigMaps/Secrets, Volumes → externalize config and add persistence.
4. Ingress, Namespaces, Resource limits → multi-app, multi-env setup.
5. RBAC, NetworkPolicy, Health checks → harden it.
6. Helm → package it properly.
7. Monitoring/Logging → make it observable.
8. Autoscaling, Scheduling → make it production-scale.
9. GitOps (Argo CD/Flux) → automate delivery.
10. CRDs/Operators, Service Mesh → extend the platform.
11. EKS specifics → run it for real on AWS.
