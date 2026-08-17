# Kubernetes Pod Pending Due to Unbound PersistentVolumeClaim

## Problem

A Kubernetes Pod may remain in the `Pending` state and fail to get
scheduled.

Example error from:

``` bash
kubectl describe pod nginx-pod -n custom-namespace
```

``` text
Warning  FailedScheduling
0/2 nodes are available: pod has unbound immediate PersistentVolumeClaims.
```

This means the Pod is using a PersistentVolumeClaim (PVC), but the PVC
is not currently bound to a PersistentVolume (PV).

The scheduler cannot assign the Pod to a node until the required PVC is
successfully bound.

------------------------------------------------------------------------

## Architecture

``` text
Pod
 |
 | uses
 v
PersistentVolumeClaim (PVC)
 |
 | binds to
 v
PersistentVolume (PV)
 |
 | uses
 v
hostPath directory on worker node
```

In this setup:

``` text
Control Plane
     |
     | schedules
     v
Worker Node
ip-172-31-82-100
     |
     +-- /home/mukesh/data/nginx-pv
             |
             +-- PersistentVolume
                    |
                    +-- PersistentVolumeClaim
                           |
                           +-- nginx-pod
```

------------------------------------------------------------------------

# Root Cause

The original Pod had the following scheduling error:

``` text
pod has unbound immediate PersistentVolumeClaims
```

The nodes themselves were healthy:

``` text
NAME               STATUS   ROLES
ip-172-31-82-100   Ready    <none>
mukesharma         Ready    control-plane
```

The actual problem was found by checking the PV:

``` bash
kubectl get pv
```

The PV was:

``` text
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS
nginx-pv   5Gi        RWO            Retain           Released
```

The PVC was:

``` text
NAME        STATUS
nginx-pvc   Pending
```

The important status was:

``` text
Released
```

------------------------------------------------------------------------

# What Does `Released` Mean?

A PV becomes `Released` when:

1.  The PV was previously bound to a PVC.
2.  That PVC was deleted.
3.  The PV has a `Retain` reclaim policy.
4.  Kubernetes keeps the PV and its previous claim information.

For example:

``` text
PV nginx-pv
    |
    +-- Previously bound to:
        custom-namespace/nginx-pvc

PVC deleted
    |
    v
PV becomes Released
```

A newly created PVC with the same name does not automatically make the
old `Released` PV available again.

------------------------------------------------------------------------

# Why the Host Directory Still Exists

The PV used:

``` yaml
hostPath:
  path: /home/mukesh/data/nginx-pv
  type: DirectoryOrCreate
```

`hostPath` points to a normal directory on the Kubernetes node.

Deleting the Kubernetes PV or PVC does not automatically delete this
directory.

Therefore:

``` text
kubectl delete pv nginx-pv
```

deletes the Kubernetes PV object, but the following directory can
remain:

``` text
/home/mukesh/data/nginx-pv
```

This is expected behavior.

The directory can be reused by a newly created PV.

------------------------------------------------------------------------

# Correct PV Configuration

For a static `hostPath` PV:

``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /home/mukesh/data/nginx-pv
    type: DirectoryOrCreate
```

Important points:

-   Capacity is `5Gi`.
-   Access mode is `ReadWriteOnce`.
-   `storageClassName` is explicitly empty.
-   Reclaim policy is `Retain`.
-   The storage is a directory on the node.
-   `DirectoryOrCreate` creates the directory if it does not exist.

------------------------------------------------------------------------

# Correct PVC Configuration

For the static PV above:

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
  namespace: custom-namespace
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  volumeName: nginx-pv
  resources:
    requests:
      storage: 5Gi
```

Important fields:

``` yaml
storageClassName: ""
```

This prevents the PVC from trying to use a StorageClass for dynamic
provisioning.

``` yaml
volumeName: nginx-pv
```

This explicitly tells Kubernetes that this PVC should use the PV named
`nginx-pv`.

------------------------------------------------------------------------

# Complete Pod Configuration

The Pod can use the PVC as follows:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: custom-namespace
spec:
  nodeSelector:
    operate: cloud

  volumes:
    - name: shared-storage
      persistentVolumeClaim:
        claimName: nginx-pvc

  containers:

    - name: nginx-container-1
      image: nginx:latest
      ports:
        - containerPort: 80
      volumeMounts:
        - name: shared-storage
          mountPath: /usr/share/nginx/html

    - name: busybox-container-1
      image: busybox:latest
      command:
        - sh
        - -c
        - |
          while true; do
            echo "hello world"
            sleep 10
          done
      volumeMounts:
        - name: shared-storage
          mountPath: /shared

    - name: redis-container
      image: redis:latest
      ports:
        - containerPort: 6379
      volumeMounts:
        - name: shared-storage
          mountPath: /data
```

------------------------------------------------------------------------

# Resolution

## Step 1: Check the Pod

``` bash
kubectl get pods -n custom-namespace
```

If the Pod is `Pending`, inspect it:

``` bash
kubectl describe pod nginx-pod -n custom-namespace
```

If you see:

``` text
pod has unbound immediate PersistentVolumeClaims
```

move to PVC/PV troubleshooting.

------------------------------------------------------------------------

## Step 2: Check the PVC

``` bash
kubectl get pvc -n custom-namespace
```

If you see:

``` text
nginx-pvc   Pending
```

the PVC is not bound.

Inspect it:

``` bash
kubectl describe pvc nginx-pvc -n custom-namespace
```

------------------------------------------------------------------------

## Step 3: Check the PV

``` bash
kubectl get pv
```

Possible states include:

  -----------------------------------------------------------------------
  PV Status                           Meaning
  ----------------------------------- -----------------------------------
  `Available`                         PV can be claimed

  `Bound`                             PV is successfully bound to a PVC

  `Released`                          Previous PVC was deleted; PV
                                      retains old claim information

  `Failed`                            PV encountered a problem
  -----------------------------------------------------------------------

For this issue, the important state was:

``` text
Released
```

------------------------------------------------------------------------

## Step 4: Remove the Old PVC and PV

If the PV is `Released` and this is a test environment:

``` bash
kubectl delete pod nginx-pod -n custom-namespace
kubectl delete pvc nginx-pvc -n custom-namespace
kubectl delete pv nginx-pv
```

The host directory is normally retained because it is a `hostPath`.

Verify:

``` bash
kubectl get pv
kubectl get pvc -n custom-namespace
```

It is normal to see:

``` text
No resources found
```

at this point.

------------------------------------------------------------------------

## Step 5: Recreate the Resources

Apply the YAML:

``` bash
kubectl apply -f pod-volume.yaml
```

Expected output:

``` text
namespace/custom-namespace unchanged
persistentvolume/nginx-pv created
persistentvolumeclaim/nginx-pvc created
pod/nginx-pod created
```

------------------------------------------------------------------------

# Verification

## Check PV

``` bash
kubectl get pv
```

Expected:

``` text
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS
nginx-pv   5Gi        RWO            Retain           Bound
```

## Check PVC

``` bash
kubectl get pvc -n custom-namespace
```

Expected:

``` text
NAME        STATUS   VOLUME     CAPACITY
nginx-pvc   Bound    nginx-pv   5Gi
```

The important state is:

``` text
PVC = Bound
```

## Check Pod

``` bash
kubectl get pod -n custom-namespace -o wide
```

Expected:

``` text
NAME        READY   STATUS    NODE
nginx-pod   3/3     Running   ip-172-31-82-100
```

------------------------------------------------------------------------

# Check the Node Label

The Pod has:

``` yaml
nodeSelector:
  operate: cloud
```

Therefore the worker node must have the label:

``` text
operate=cloud
```

Check:

``` bash
kubectl get nodes --show-labels
```

If necessary:

``` bash
kubectl label node ip-172-31-82-100 operate=cloud
```

Then check:

``` bash
kubectl get pods -n custom-namespace -o wide
```

Note: The original scheduling error was caused by the unbound PVC, not
by the node selector, because both nodes were already in the `Ready`
state.

------------------------------------------------------------------------

# Check the HostPath Directory

Because the PV uses:

``` yaml
hostPath:
  path: /home/mukesh/data/nginx-pv
```

the directory is on the node where the Pod uses the volume.

Check the worker node:

``` bash
ls -ld /home/mukesh/data/nginx-pv
```

The directory is not automatically removed when you delete:

-   Pod
-   PVC
-   PV
-   Namespace

If the directory must be removed manually:

``` bash
sudo rm -rf /home/mukesh/data/nginx-pv
```

Use `rm -rf` only after confirming the path is correct.

------------------------------------------------------------------------

# Testing the Containers

## BusyBox Logs

The BusyBox container continuously writes:

``` text
hello world
```

Check:

``` bash
kubectl logs nginx-pod \
  -n custom-namespace \
  -c busybox-container-1
```

Expected:

``` text
hello world
hello world
hello world
```

## Redis

Test Redis:

``` bash
kubectl exec -it nginx-pod \
  -n custom-namespace \
  -c redis-container \
  -- redis-cli ping
```

Expected:

``` text
PONG
```

## Nginx

Check the Nginx container:

``` bash
kubectl logs nginx-pod \
  -n custom-namespace \
  -c nginx-container-1
```

Nginx may not produce continuous logs, so an empty result does not
necessarily indicate a problem.

------------------------------------------------------------------------

# Important: `Defaulted Container`

If a Pod has multiple containers and you run:

``` bash
kubectl logs nginx-pod -n custom-namespace
```

Kubernetes may show:

``` text
Defaulted container "nginx-container-1" out of:
nginx-container-1, busybox-container-1, redis-container
```

This is not an error.

It means Kubernetes selected one container because no container was
specified.

To select a container explicitly:

``` bash
kubectl logs nginx-pod \
  -n custom-namespace \
  -c redis-container
```

or:

``` bash
kubectl logs nginx-pod \
  -n custom-namespace \
  -c busybox-container-1
```

------------------------------------------------------------------------

# Troubleshooting Cheat Sheet

## Pod is Pending

Run:

``` bash
kubectl describe pod nginx-pod -n custom-namespace
```

If you see:

``` text
unbound immediate PersistentVolumeClaims
```

check:

``` bash
kubectl get pvc -n custom-namespace
kubectl get pv
```

------------------------------------------------------------------------

## PVC is Pending

Run:

``` bash
kubectl describe pvc nginx-pvc -n custom-namespace
```

Check:

-   PV status
-   StorageClass
-   Access mode
-   Requested capacity
-   `volumeName`
-   PVC events

------------------------------------------------------------------------

## PV is Released

If using a test/static PV:

``` bash
kubectl delete pvc nginx-pvc -n custom-namespace
kubectl delete pv nginx-pv
```

Then recreate the PV and PVC.

------------------------------------------------------------------------

## PVC is Bound but Pod is Pending

Check:

``` bash
kubectl describe pod nginx-pod -n custom-namespace
```

Then verify the node selector:

``` bash
kubectl get nodes --show-labels
```

The required label is:

``` text
operate=cloud
```

------------------------------------------------------------------------

## Pod is Running but `kubectl logs` is Empty

This can be normal.

Check the correct container:

``` bash
kubectl logs nginx-pod \
  -n custom-namespace \
  -c <container-name>
```

For BusyBox in this example:

``` bash
kubectl logs nginx-pod \
  -n custom-namespace \
  -c busybox-container-1
```

------------------------------------------------------------------------

# Quick Troubleshooting Commands

When this problem happens again, run these commands in order:

``` bash
kubectl get pods -n custom-namespace
```

``` bash
kubectl describe pod nginx-pod -n custom-namespace
```

``` bash
kubectl get pvc -n custom-namespace
```

``` bash
kubectl get pv
```

``` bash
kubectl describe pvc nginx-pvc -n custom-namespace
```

``` bash
kubectl describe pv nginx-pv
```

Then verify:

``` text
PV       → Bound
PVC      → Bound
Pod      → Running
Node     → Ready
```

------------------------------------------------------------------------

# Key Lesson

The most important troubleshooting chain from this incident is:

``` text
Pod Pending
     |
     v
FailedScheduling
     |
     v
"pod has unbound immediate PersistentVolumeClaims"
     |
     v
Check PVC
     |
     v
PVC Pending
     |
     v
Check PV
     |
     v
PV Released
     |
     v
Old PVC was deleted
     |
     v
PV retained old claim information
     |
     v
Delete/recreate PV and PVC
     |
     v
PV Bound
     |
     v
PVC Bound
     |
     v
Pod scheduled
     |
     v
Pod Running
```

## Final Working Configuration

For this static `hostPath` setup, the critical PV/PVC relationship is:

``` yaml
# PV
storageClassName: ""
```

and:

``` yaml
# PVC
storageClassName: ""
volumeName: nginx-pv
```

with matching:

``` text
Capacity:    5Gi
AccessMode:  ReadWriteOnce
```

Once:

``` text
nginx-pv  → Bound
nginx-pvc → Bound
```

the scheduler can schedule the Pod.
