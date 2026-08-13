# Kubernetes Scenario-Based Questions & Answers

A practical reference for common Kubernetes troubleshooting and operational scenarios commonly asked in DevOps/SRE interviews.

---

## Table of Contents

1. [Pod in CrashLoopBackOff](#1-pod-in-crashloopbackoff)
2. [Pod Stuck in ImagePullBackOff](#2-pod-stuck-in-imagepullbackoff)
3. [Pod Stuck in Pending State](#3-pod-stuck-in-pending-state)
4. [Application Not Reachable via Service](#4-application-not-reachable-via-service)
5. [Ingress Returns 502/503/504](#5-ingress-returns-502503504)
6. [Pod OOMKilled (Out of Memory)](#6-pod-oomkilled-out-of-memory)
7. [Pod Evicted Due to Disk Pressure](#7-pod-evicted-due-to-disk-pressure)
8. [Deployment Rollout Stuck or Failed](#8-deployment-rollout-stuck-or-failed)
9. [ConfigMap/Secret Change Not Reflected in App](#9-configmapsecret-change-not-reflected-in-app)
10. [PVC Stuck in Pending](#10-pvc-stuck-in-pending)
11. [DNS Resolution Failing Inside Pod](#11-dns-resolution-failing-inside-pod)
12. [Node Not Ready / Node Failure](#12-node-not-ready--node-failure)
13. [Too Many Pods on a Node (Resource Exhaustion)](#13-too-many-pods-on-a-node-resource-exhaustion)
14. [RBAC: User/ServiceAccount Cannot Access Resources](#14-rbac-userserviceaccount-cannot-access-resources)
15. [Horizontal Pod Autoscaler Not Scaling](#15-horizontal-pod-autoscaler-not-scaling)
16. [Pod Anti-Affinity Causing Scheduling Failure](#16-pod-anti-affinity-causing-scheduling-failure)
17. [Init Container Failing](#17-init-container-failing)
18. [Liveness vs Readiness Probe Failures](#18-liveness-vs-readiness-probe-failures)
19. [Certificate Expired on Ingress/API Server](#19-certificate-expired-on-ingressapi-server)
20. [Multi-Container Pod: Sidecar Not Working](#20-multi-container-pod-sidecar-not-working)
21. [StatefulSet Pod Identity / Ordering Issues](#21-statefulset-pod-identity--ordering-issues)
22. [NetworkPolicy Blocking Traffic](#22-networkpolicy-blocking-traffic)
23. [Cluster Upgrade Causing Workload Disruption](#23-cluster-upgrade-causing-workload-disruption)
24. [Pod Logs Not Appearing](#24-pod-logs-not-appearing)
25. [Resource Quota Preventing Pod Creation](#25-resource-quota-preventing-pod-creation)

---

## 1. Pod in CrashLoopBackOff

**Scenario:** A pod keeps restarting. `kubectl get pods` shows `CrashLoopBackOff`.

**Answer:**

```bash
# Step 1: Inspect pod events and status
kubectl describe pod <pod-name> -n <namespace>

# Step 2: Check current and previous container logs
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous

# Step 3: Verify the container command, env vars, and mounted volumes
kubectl get pod <pod-name> -n <namespace> -o yaml
```

**Common root causes:**
- Application crash on startup (bad config, missing env var, DB unreachable)
- Wrong entrypoint/command in the container image
- Missing ConfigMap/Secret volume mount
- Port already in use inside the container
- Liveness probe too aggressive, killing the pod before it starts

**Fix:** Resolve the underlying application error. Adjust probes (`initialDelaySeconds`, `failureThreshold`). Fix env/config. Ensure dependencies (DB, cache) are reachable.

---

## 2. Pod Stuck in ImagePullBackOff

**Scenario:** Pod status shows `ImagePullBackOff` or `ErrImagePull`.

**Answer:**

```bash
kubectl describe pod <pod-name> -n <namespace>
# Look for: "Failed to pull image", "unauthorized", "not found"
```

**Common root causes:**
- Wrong image name or tag (typo, deleted tag)
- Private registry without `imagePullSecrets`
- Registry credentials expired
- Rate limiting from Docker Hub

**Fix:**

```bash
# Create pull secret for private registry
kubectl create secret docker-registry regcred \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<pass> \
  -n <namespace>

# Reference in pod/deployment spec:
# spec.imagePullSecrets:
#   - name: regcred
```

Verify the image exists: `docker pull <image>:<tag>` or use `crane manifest <image>`.

---

## 3. Pod Stuck in Pending State

**Scenario:** Pod remains `Pending` and never schedules to a node.

**Answer:**

```bash
kubectl describe pod <pod-name> -n <namespace>
# Check Events section for scheduling failures
```

**Common root causes:**
- Insufficient CPU/memory on nodes
- No node matches nodeSelector/affinity rules
- Taints on nodes without matching tolerations
- PVC not bound (volume cannot be provisioned)
- Pod exceeds namespace ResourceQuota
- Cluster autoscaler not provisioning nodes fast enough

**Fix:**
- Add nodes or reduce resource requests
- Relax affinity/tolerations
- Fix PVC/storage class
- Reduce requests or increase quota

```bash
kubectl get nodes
kubectl top nodes
kubectl get pvc -n <namespace>
kubectl describe quota -n <namespace>
```

---

## 4. Application Not Reachable via Service

**Scenario:** Service exists but `curl http://<service>` from another pod fails.

**Answer:**

```bash
# Verify service endpoints
kubectl get svc <service-name> -n <namespace>
kubectl get endpoints <service-name> -n <namespace>

# Check if pods have correct labels matching service selector
kubectl get pods -n <namespace> --show-labels
kubectl get svc <service-name> -n <namespace> -o yaml  # check spec.selector

# Test from a debug pod
kubectl run debug --rm -it --image=busybox -- sh
wget -qO- http://<service-name>.<namespace>.svc.cluster.local:<port>
```

**Common root causes:**
- Service selector doesn't match pod labels
- Pods not ready (readiness probe failing → no endpoints)
- Wrong targetPort (service port vs container port mismatch)
- NetworkPolicy blocking traffic
- Pods listening on wrong port

**Fix:** Align labels/selectors, fix readiness probes, correct `targetPort`, review NetworkPolicies.

---

## 5. Ingress Returns 502/503/504

**Scenario:** External traffic via Ingress returns gateway errors.

**Answer:**

```bash
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Verify backend service has endpoints
kubectl get endpoints -n <namespace>
```

**Common root causes:**
- **502/503:** No healthy backend pods (readiness failing, 0 endpoints)
- **504:** Backend too slow; ingress timeout too low
- Wrong `path` or `host` rule in Ingress
- TLS secret missing or invalid
- Ingress class annotation missing/wrong

**Fix:** Fix backend health, increase timeout annotations, correct paths/hosts, ensure TLS secret exists.

---

## 6. Pod OOMKilled (Out of Memory)

**Scenario:** Pod restarts with reason `OOMKilled`. `kubectl describe pod` shows `Last State: Terminated, Reason: OOMKilled`.

**Answer:**

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl top pod <pod-name> -n <namespace>   # requires metrics-server
```

**Common root causes:**
- Memory limit too low for the workload
- Memory leak in application
- No memory limit set → node-level OOM killer targets the pod

**Fix:**

```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

Profile the app, increase limits appropriately, fix memory leaks. Set requests close to actual usage for better scheduling.

---

## 7. Pod Evicted Due to Disk Pressure

**Scenario:** Pods evicted with reason `Evicted`. Node shows `DiskPressure=True`.

**Answer:**

```bash
kubectl describe node <node-name>
# Conditions: DiskPressure=True

kubectl get pods -A --field-selector status.phase=Failed
```

**Common root causes:**
- Container logs filling disk (`/var/log/pods`)
- Unused images/containers consuming space
- EmptyDir volumes growing unbounded
- Small root volume on worker nodes

**Fix:**
- Configure log rotation on nodes
- Run image garbage collection: `crictl rmi --prune`
- Set log size limits in container runtime
- Increase node disk size
- Use `emptyDir.sizeLimit` for ephemeral storage

---

## 8. Deployment Rollout Stuck or Failed

**Scenario:** `kubectl rollout status deployment/<name>` hangs or reports failure.

**Answer:**

```bash
kubectl rollout status deployment/<name> -n <namespace>
kubectl rollout history deployment/<name> -n <namespace>
kubectl get rs -n <namespace>
kubectl describe deployment <name> -n <namespace>
```

**Common root causes:**
- New pods failing readiness checks
- `maxUnavailable` / `maxSurge` constraints blocking progress
- Image pull failures on new ReplicaSet
- Insufficient cluster resources for surge pods

**Fix:**

```bash
# Rollback to previous version
kubectl rollout undo deployment/<name> -n <namespace>

# Rollback to specific revision
kubectl rollout undo deployment/<name> --to-revision=2 -n <namespace>
```

Fix the failing new version, then redeploy.

---

## 9. ConfigMap/Secret Change Not Reflected in App

**Scenario:** You updated a ConfigMap but the running app still uses old values.

**Answer:**

**Key fact:** Updating a ConfigMap/Secret mounted as a volume is reflected eventually (kubelet syncs periodically, ~60s). Environment variables from ConfigMap/Secret are **not** updated after pod creation.

**Fix:**

```bash
# Force pod restart to pick up changes
kubectl rollout restart deployment/<name> -n <namespace>

# Or delete pods (Deployment recreates them)
kubectl delete pod -l app=<label> -n <namespace>
```

**Best practice:** Mount ConfigMaps as files (not env vars) for hot-reload support, or use tools like Reloader/Stakater to auto-restart on config change.

---

## 10. PVC Stuck in Pending

**Scenario:** `kubectl get pvc` shows status `Pending`.

**Answer:**

```bash
kubectl describe pvc <pvc-name> -n <namespace>
kubectl get storageclass
kubectl get pv
```

**Common root causes:**
- No StorageClass configured or wrong `storageClassName`
- No available PV matching size/access mode
- Cloud provider quota exceeded (EBS, GCE PD limits)
- Zone mismatch (PV in `us-east-1a`, pod scheduled to `us-east-1b`)

**Fix:**
- Define a default StorageClass with a dynamic provisioner
- Ensure access mode is supported (`ReadWriteOnce` vs `ReadWriteMany`)
- For local/static PVs, match node affinity and size

---

## 11. DNS Resolution Failing Inside Pod

**Scenario:** App cannot resolve `service.namespace.svc.cluster.local`.

**Answer:**

```bash
# From inside a pod
nslookup kubernetes.default
nslookup <service>.<namespace>.svc.cluster.local

# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

**Common root causes:**
- CoreDNS pods down or crash-looping
- `dnsPolicy` misconfigured on pod
- NetworkPolicy blocking UDP/TCP port 53
- Custom `dnsConfig` with wrong nameservers
- `ndots` issue — short names not resolving as expected

**Fix:** Restore CoreDNS, allow DNS traffic in NetworkPolicies, use FQDN (`service.namespace.svc.cluster.local`), set `dnsPolicy: ClusterFirst`.

---

## 12. Node Not Ready / Node Failure

**Scenario:** `kubectl get nodes` shows a node as `NotReady`.

**Answer:**

```bash
kubectl describe node <node-name>
kubectl get pods -A -o wide | grep <node-name>

# On the node (SSH)
systemctl status kubelet
journalctl -u kubelet -f
df -h    # disk full?
free -m  # memory pressure?
```

**Common root causes:**
- Kubelet stopped or crashed
- Node out of disk/memory (pressure taints applied)
- Network partition between node and control plane
- Container runtime (containerd/docker) down
- Certificate expired on kubelet

**Fix:** Restart kubelet, free disk space, fix networking, renew certificates. Pods on failed node are rescheduled after `pod-eviction-timeout` (default ~5 min) if workloads have multiple replicas.

**Manual drain before maintenance:**

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node-name>   # after maintenance
```

---

## 13. Too Many Pods on a Node (Resource Exhaustion)

**Scenario:** New pods can't schedule; existing pods are slow or being evicted.

**Answer:**

```bash
kubectl top nodes
kubectl describe node <node-name>   # Allocated resources section
kubectl get pods -A -o wide --field-selector spec.nodeName=<node-name>
```

**Common root causes:**
- Pods without resource requests → scheduler overcommits node
- DaemonSets consuming baseline resources
- No cluster autoscaler or max node limit reached

**Fix:**
- Set appropriate `requests` and `limits` on all workloads
- Use PodDisruptionBudgets for critical apps
- Enable cluster autoscaler
- Spread workloads with pod anti-affinity
- Use LimitRanges and ResourceQuotas at namespace level

---

## 14. RBAC: User/ServiceAccount Cannot Access Resources

**Scenario:** `kubectl` or in-cluster API calls return `Forbidden` errors.

**Answer:**

```bash
# Check what the user/SA can do
kubectl auth can-i create pods --as=system:serviceaccount:<ns>:<sa-name> -n <namespace>
kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa-name> -n <namespace>

# Inspect bindings
kubectl get role,rolebinding,clusterrole,clusterrolebinding -n <namespace>
```

**Common root causes:**
- Missing Role/ClusterRole or RoleBinding/ClusterRoleBinding
- Role grants access in wrong namespace
- ServiceAccount not specified in pod spec
- Aggregated ClusterRoles missing expected rules

**Fix:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: my-app
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: my-app
subjects:
  - kind: ServiceAccount
    name: my-sa
    namespace: my-app
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 15. Horizontal Pod Autoscaler Not Scaling

**Scenario:** HPA exists but replica count never changes under load.

**Answer:**

```bash
kubectl get hpa -n <namespace>
kubectl describe hpa <hpa-name> -n <namespace>
kubectl top pods -n <namespace>   # metrics-server required
```

**Common root causes:**
- Metrics server not installed or not working
- CPU requests not set on pods (HPA uses % of request by default)
- HPA `minReplicas` equals `maxReplicas`
- Custom metrics adapter missing (for non-CPU/memory metrics)
- Cooldown period not elapsed

**Fix:** Install metrics-server, set resource requests, configure appropriate min/max replicas and target utilization.

```yaml
resources:
  requests:
    cpu: "250m"
```

---

## 16. Pod Anti-Affinity Causing Scheduling Failure

**Scenario:** Pods stay Pending; events mention affinity/anti-affinity rules not satisfied.

**Answer:**

```bash
kubectl describe pod <pod-name> -n <namespace>
# "0/3 nodes are available: 3 node(s) didn't match pod anti-affinity rules"
```

**Common root causes:**
- `requiredDuringSchedulingIgnoredDuringExecution` anti-affinity with too few nodes
- Trying to spread more replicas than available nodes
- Hard anti-affinity on a single-node cluster

**Fix:**
- Change to `preferredDuringSchedulingIgnoredDuringExecution` (soft rule)
- Add more nodes
- Reduce replica count
- Review topology key (`kubernetes.io/hostname` vs zone)

---

## 17. Init Container Failing

**Scenario:** Pod stuck in `Init:Error` or `Init:CrashLoopBackOff`.

**Answer:**

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -c <init-container-name> -n <namespace>
```

**Common root causes:**
- Init container waiting for dependency that isn't ready (DB migration, config download)
- Wrong command or missing permissions
- NetworkPolicy blocking init container from reaching external service

**Fix:** Fix init container script/image, add retry logic, ensure dependencies are available, check RBAC if init container talks to K8s API.

---

## 18. Liveness vs Readiness Probe Failures

**Scenario:** Pods restart frequently OR receive traffic before ready.

**Answer:**

| Probe | Purpose | Failure Action |
|-------|---------|----------------|
| **Liveness** | Is the container alive? | Kubelet **restarts** the container |
| **Readiness** | Is the container ready for traffic? | Removed from Service **endpoints** (no restart) |
| **Startup** | Has the app finished starting? | Disables liveness/readiness until success |

**Common mistakes:**
- Using the same endpoint for liveness and readiness
- Liveness probe hitting a slow endpoint → unnecessary restarts
- Readiness probe not checking actual dependencies (DB connection)
- `initialDelaySeconds` too low for slow-starting apps

**Fix:**

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 30
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

---

## 19. Certificate Expired on Ingress/API Server

**Scenario:** HTTPS connections fail with certificate errors; API calls fail with x509 errors.

**Answer:**

```bash
# Check API server cert (from admin machine)
openssl s_client -connect <api-server>:6443 </dev/null 2>/dev/null | openssl x509 -noout -dates

# Check ingress TLS secret
kubectl get secret <tls-secret> -n <namespace> -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates

# kubeadm clusters — renew certs
sudo kubeadm certs check-expiration
sudo kubeadm certs renew all
```

**Fix:**
- Renew certs with kubeadm or your installer (kops, EKS handles control plane)
- For Ingress, update TLS secret or use cert-manager for auto-renewal
- Restart affected components after renewal

---

## 20. Multi-Container Pod: Sidecar Not Working

**Scenario:** Main app container runs but sidecar (logging proxy, service mesh) fails or doesn't receive traffic.

**Answer:**

```bash
kubectl logs <pod-name> -c <sidecar-container> -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

**Common root causes:**
- Sidecar starts after main container needs it (ordering issue — use init container or K8s 1.28+ native sidecar containers with `restartPolicy: Always` in initContainers)
- Shared volume permissions (one container writes, other can't read)
- localhost communication: containers in same pod share network namespace — use `127.0.0.1:<port>`

**Fix:** Ensure shared volumes use correct `emptyDir` or `fsGroup`. Use init containers for ordering. For service mesh, verify injection annotations.

---

## 21. StatefulSet Pod Identity / Ordering Issues

**Scenario:** StatefulSet pods fail to start in order, or apps can't find peer by stable DNS name.

**Answer:**

```bash
kubectl get sts <name> -n <namespace>
kubectl get pods -l app=<label> -n <namespace>
# Pods named: <sts-name>-0, <sts-name>-1, ...
# DNS: <pod-name>.<headless-service>.<namespace>.svc.cluster.local
```

**Common root causes:**
- Pod `-0` not ready → subsequent pods wait (OrderedReady policy)
- Headless Service missing or wrong selector
- PVC from previous pod not released after delete (retained policy)
- Application expects all peers up before joining cluster

**Fix:** Ensure headless Service exists, fix pod-0 first, check PVC binding per pod, consider `podManagementPolicy: Parallel` if order isn't required.

---

## 22. NetworkPolicy Blocking Traffic

**Scenario:** Pods in different namespaces or same namespace can't communicate despite Services being correct.

**Answer:**

```bash
kubectl get networkpolicy -n <namespace>
kubectl describe networkpolicy <name> -n <namespace>
```

**Key behavior:** If any NetworkPolicy exists in a namespace, pods in that namespace are **deny-by-default** for ingress (and egress if policies specify egress rules).

**Fix:**
- Add explicit allow rules for required traffic (DNS on port 53, app ports)
- Label pods correctly for policy selectors
- Test with a temporary permissive policy to confirm

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
```

---

## 23. Cluster Upgrade Causing Workload Disruption

**Scenario:** After upgrading Kubernetes version, workloads behave unexpectedly.

**Answer:**

**Pre-upgrade checklist:**
1. Read release notes for deprecated/removed APIs
2. Run `kubectl deprecations` or pluto to find deprecated resources
3. Upgrade control plane first, then worker nodes one at a time
4. Drain nodes before upgrade

```bash
# Check deprecated APIs in use
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis

# Safe node upgrade
kubectl drain <node> --ignore-daemonsets
# upgrade node
kubectl uncordon <node>
```

**Common post-upgrade issues:**
- Removed beta APIs (e.g., old Ingress, CronJob versions)
- CNI plugin incompatibility with new K8s version
- Changed default admission behavior

---

## 24. Pod Logs Not Appearing

**Scenario:** `kubectl logs <pod>` returns empty or "Error from server".

**Answer:**

```bash
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -c <container> -n <namespace> --previous
kubectl describe pod <pod-name> -n <namespace>
```

**Common root causes:**
- App logs to file instead of stdout/stderr (K8s only captures stdout/stderr)
- Container restarted and `--previous` needed
- Pod already deleted (check centralized logging: ELK, Loki, CloudWatch)
- Insufficient RBAC to read logs

**Fix:** Configure app to log to stdout. Deploy a log aggregation stack (Fluent Bit → Loki/Elasticsearch). Grant `pods/log` RBAC permission.

---

## 25. Resource Quota Preventing Pod Creation

**Scenario:** Pod creation fails with `exceeded quota` error.

**Answer:**

```bash
kubectl describe quota -n <namespace>
kubectl describe limitrange -n <namespace>
```

**Common root causes:**
- Namespace ResourceQuota maxed out (pods, CPU, memory, PVC count)
- LimitRange requires resources but pod spec omits them
- Too many objects (services, secrets, configmaps) hitting quota

**Fix:**
- Increase quota or clean up unused resources
- Add resource requests/limits to pod spec
- Move workloads to a different namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
```

---

## Quick Reference: Essential Debug Commands

```bash
# Cluster health
kubectl get nodes
kubectl get componentstatuses          # deprecated but still useful on some clusters
kubectl get pods -A | grep -v Running

# Pod troubleshooting
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --all-containers
kubectl logs <pod> -n <ns> --previous
kubectl exec -it <pod> -n <ns> -- sh

# Events (cluster-wide, sorted by time)
kubectl get events -A --sort-by='.lastTimestamp'

# Resource usage
kubectl top nodes
kubectl top pods -n <ns>

# Networking
kubectl get svc,endpoints -n <ns>
kubectl run tmp --rm -it --image=nicolaka/netshoot -- bash

# Rollout
kubectl rollout status deployment/<name> -n <ns>
kubectl rollout undo deployment/<name> -n <ns>
```

---

## General Troubleshooting Framework

For any Kubernetes scenario, follow this order:

1. **Observe** — `kubectl get`, `kubectl describe`, check Events
2. **Logs** — current and `--previous` container logs
3. **Configuration** — compare Deployment/Pod YAML with working state
4. **Dependencies** — DNS, Service endpoints, ConfigMaps, Secrets, PVCs
5. **Infrastructure** — node health, network, storage, quotas
6. **Recent changes** — deployments, config updates, cluster upgrades
7. **Fix & verify** — apply fix, confirm with `kubectl get`, test connectivity
