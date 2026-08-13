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
