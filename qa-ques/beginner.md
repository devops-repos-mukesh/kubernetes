<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# A developer deploys an application using a Deployment, but no Pods appear. How would you investigate the issue?

A Pod remains in Pending state after deployment. What possible causes would you check, and which commands would you use?
A container repeatedly restarts and shows CrashLoopBackOff. How would you identify whether the issue is in the application, its configuration, or its dependencies?
An application works locally but fails in Kubernetes because an environment variable is missing. How would you inject and manage configuration values?
A Pod needs database credentials, but the team does not want to store them in the container image or plain YAML files. How would you handle this?
A service is running with three Pods, but users cannot access it through the Service. What would you verify from Pod labels through Service endpoints?
A new version of an application is deployed, but traffic is still reaching old Pods. What Kubernetes objects and labels would you inspect?
A Pod is deleted manually, but it is recreated immediately. Why does this happen, and how would you stop it from being recreated?
A Deployment has five replicas, but only three Pods are running. How would you diagnose the missing replicas?
A container must write temporary files during runtime, but data does not need to survive a Pod restart. Which volume type would you use and why?
A developer exposes an application with NodePort, but it is not reachable externally. What networking, firewall, and cluster-level checks would you make?
One Pod must run on a specific node because it requires a local GPU. How would you control Pod placement?
short answers with commands required

```
Use this as a concise interview-ready troubleshooting flow. Replace placeholders such as `<ns>`, `<deploy>`, and `<pod>` with your actual resource names.
```


## Deployment and scheduling

### 1. Deployment exists but no Pods appear

Check whether the Deployment created a ReplicaSet, then inspect events for admission, image, quota, or scheduling errors.

```bash
kubectl get deploy,rs,pods -n <ns>
kubectl describe deployment <deploy> -n <ns>
kubectl get events -n <ns> --sort-by='.lastTimestamp'
kubectl get deployment <deploy> -n <ns> -o yaml
```

A Deployment normally creates a ReplicaSet, which then creates Pods. If no ReplicaSet exists, inspect the Deployment spec and events; if the ReplicaSet exists but has no Pods, inspect ReplicaSet events.[^1]

### 2. Pod stays `Pending`

Start with Pod events. Common causes are insufficient CPU/memory, node selectors or affinity with no matching node, taints without tolerations, unbound PVCs, resource quotas, or unavailable nodes.

```bash
kubectl describe pod <pod> -n <ns>
kubectl get events -n <ns> --sort-by='.lastTimestamp'
kubectl get nodes
kubectl describe node <node>
kubectl get pvc -n <ns>
kubectl get resourcequota,limitrange -n <ns>
```

Look specifically for events such as `FailedScheduling`, `Insufficient cpu`, `Insufficient memory`, `didn't match node selector`, or `unbound immediate PersistentVolumeClaims`.[^1]

### 3. Pod is in `CrashLoopBackOff`

Check current and previous container logs, then inspect exit code, events, environment variables, mounted volumes, probes, and resource limits.

```bash
kubectl logs <pod> -n <ns> -c <container>
kubectl logs <pod> -n <ns> -c <container> --previous
kubectl describe pod <pod> -n <ns>
kubectl get pod <pod> -n <ns> -o yaml
kubectl exec -it <pod> -n <ns> -c <container> -- printenv
```

- **Application issue:** application logs show exceptions, bad startup arguments, or the process exits unexpectedly.
- **Configuration issue:** missing environment variables, ConfigMaps, Secrets, files, or incorrect commands.
- **Dependency issue:** failed database, DNS, API, or message-queue connection.
- **Resource issue:** look for `OOMKilled` and increase memory limits only after validating actual usage.

Kubernetes recommends checking logs, Pod events, configuration, external dependencies, and resource limits when diagnosing `CrashLoopBackOff`.[^2]

### 4. Missing environment variable

Use a ConfigMap for non-sensitive settings and inject it with `env`, `envFrom`, or `valueFrom`. Do not bake environment-specific configuration into the image.

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  -n <ns>

kubectl get configmap app-config -n <ns> -o yaml
kubectl rollout restart deployment/<deploy> -n <ns>
```

Example Deployment fragment:

```yaml
envFrom:
  - configMapRef:
      name: app-config
```

For one value:

```yaml
env:
  - name: APP_ENV
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_ENV
```


### 5. Database credentials securely

Use a Kubernetes Secret for runtime injection; for production, source it from an external secret manager such as AWS Secrets Manager, HashiCorp Vault, or Azure Key Vault through an approved operator/CSI driver. Never put real secret values in image layers or plain Git manifests.

```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=appuser \
  --from-literal=DB_PASSWORD='<password>' \
  -n <ns>

kubectl get secret db-credentials -n <ns>
kubectl describe secret db-credentials -n <ns>
```

Inject it:

```yaml
envFrom:
  - secretRef:
      name: db-credentials
```

Remember: normal Kubernetes Secrets are base64-encoded, not inherently encrypted unless encryption at rest is configured.

### 6. Service has Pods but is unreachable

Verify the Service selector matches Pod labels, confirm EndpointSlices contain ready Pod IPs, and validate `port` and `targetPort`.

```bash
kubectl get svc <service> -n <ns> -o yaml
kubectl get pods -n <ns> --show-labels
kubectl get pods -n <ns> -l app=<label>
kubectl get endpointslices -n <ns> \
  -l kubernetes.io/service-name=<service>
kubectl describe svc <service> -n <ns>
```

Then test from inside the cluster:

```bash
kubectl run nettest --rm -it --restart=Never -n <ns> \
  --image=busybox:1.36 -- sh

wget -qO- http://<service>.<ns>.svc.cluster.local:<port>
```

A Service only routes to Pods selected by its labels and represented as ready endpoints; also confirm that the Service `targetPort` maps to the actual application port.[^3][^1]

## Rollouts and replicas

### 7. Traffic still reaches old Pods after deployment

Inspect rollout status, old ReplicaSets, Service selectors, EndpointSlices, readiness, and any external load balancer or Ingress cache.

```bash
kubectl rollout status deployment/<deploy> -n <ns>
kubectl rollout history deployment/<deploy> -n <ns>
kubectl get rs -n <ns>
kubectl get pods -n <ns> -l app=<app> -o wide
kubectl get endpointslices -n <ns> \
  -l kubernetes.io/service-name=<service>
kubectl describe deployment <deploy> -n <ns>
```

Old Pods may still receive traffic if they are still Ready, the rollout is incomplete, the Service selector matches both versions, or the old ReplicaSet has not scaled down.

### 8. Deleted Pod comes back

The Pod is managed by a controller such as a Deployment, ReplicaSet, StatefulSet, DaemonSet, or Job. The controller continuously reconciles actual replicas to the desired state.

```bash
kubectl get pod <pod> -n <ns> -o yaml
kubectl get deploy,rs,sts,ds,job -n <ns>
kubectl describe pod <pod> -n <ns>
```

To stop recreation, scale or remove its owner controller instead of deleting the Pod:

```bash
kubectl scale deployment/<deploy> --replicas=0 -n <ns>
# Or, if appropriate:
kubectl delete deployment/<deploy> -n <ns>
```


### 9. Deployment wants five replicas but only three run

Compare desired, current, updated, available, and unavailable replicas. Then inspect non-running Pods and scheduling events.

```bash
kubectl get deployment <deploy> -n <ns>
kubectl describe deployment <deploy> -n <ns>
kubectl get rs -n <ns>
kubectl get pods -n <ns> -l app=<app> -o wide
kubectl get events -n <ns> --sort-by='.lastTimestamp'
```

Typical causes include Pods stuck in `Pending`, `ImagePullBackOff`, `CrashLoopBackOff`, failed readiness probes, quota limits, insufficient capacity, or a rollout strategy limiting temporary extra Pods.

## Storage and networking

### 10. Temporary writable storage

Use `emptyDir`. It is created empty when the Pod starts and is removed when the Pod is removed; it is appropriate for caches, scratch files, and temporary processing data.

```yaml
volumes:
  - name: tmp-data
    emptyDir: {}

containers:
  - name: app
    volumeMounts:
      - name: tmp-data
        mountPath: /tmp/app
```

For memory-backed temporary files:

```yaml
emptyDir:
  medium: Memory
```

`emptyDir` is node-local ephemeral storage managed by kubelet.[^4]

### 11. NodePort is unreachable externally

First verify the Service, its endpoints, the node IP, and whether the application is reachable internally. Then check firewall/security-group rules and kube-proxy/CNI health.

```bash
kubectl get svc <service> -n <ns>
kubectl describe svc <service> -n <ns>
kubectl get endpointslices -n <ns> \
  -l kubernetes.io/service-name=<service>
kubectl get nodes -o wide
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl get pods -n <ns> -l app=<app> -o wide
```

Test from a node or another system:

```bash
curl http://<node-ip>:<node-port>
nc -vz <node-ip> <node-port>
```

Check:

- The Service has a valid `nodePort`, `port`, and `targetPort`
- EndpointSlices contain ready Pods
- The application listens on the expected container port
- The node firewall, cloud security group, and network ACL allow the NodePort
- `kube-proxy` and the CNI plugin are healthy
- You are using a reachable node IP, not a private-only address

NodePort exposes the same allocated port on each node, normally from the default 30000–32767 range.[^5][^3]

### 12. Schedule a GPU workload to a specific node

Label the GPU node and use `nodeSelector` for simple placement. Also request the GPU resource so Kubernetes schedules only where it is available.

```bash
kubectl label node <gpu-node> accelerator=nvidia
kubectl describe node <gpu-node>
```

Example Pod or Deployment fragment:

```yaml
spec:
  nodeSelector:
    accelerator: nvidia
  containers:
    - name: gpu-app
      image: <image>
      resources:
        limits:
          nvidia.com/gpu: 1
```

For more flexible placement rules, use node affinity:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: accelerator
              operator: In
              values:
                - nvidia
```

Verify scheduling and GPU availability:

```bash
kubectl get nodes -L accelerator
kubectl describe node <gpu-node> | grep -A5 -i gpu
kubectl describe pod <pod> -n <ns>
```
