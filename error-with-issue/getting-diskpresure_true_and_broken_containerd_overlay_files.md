# Kubernetes Pod Pending and ContainerCreating – Problem & Resolution

## Problem

The Kubernetes Pod `nginx-pod` was initially stuck in the `Pending` state and later in the `ContainerCreating` state.

The first issue was with **disk pressure on the Kubernetes worker node**. The scheduler reported:

```text
0/2 nodes are available:
1 node had untolerated taint {node-role.kubernetes.io/control-plane:}
1 node had untolerated taint {node.kubernetes.io/disk-pressure:}
```

The worker node had very little free disk space, approximately **900 MB**. Kubelet was reporting:

```text
EvictionThresholdMet
Attempting to reclaim ephemeral-storage
```

Disk usage investigation showed that `/var/log`, `/var/lib`, and `/var/cache` were consuming significant space. `/var/lib/containerd` was the largest part of `/var/lib`, using approximately **1.3 GB**, with the overlayfs snapshotter using approximately **991 MB**.

After cleaning unnecessary logs and package cache, disk space increased to approximately **1.7 GB**, and kubelet reported:

```text
NodeHasNoDiskPressure
```

The Pod was then successfully scheduled on the worker node.

However, the Pod subsequently became stuck in `ContainerCreating`.

The second issue was a **broken/inconsistent containerd overlayfs snapshot state**. Kubelet reported:

```text
FailedCreatePodSandBox

failed to stat parent:
stat /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/1/fs:
no such file or directory
```

Containerd was trying to use snapshot `1`, but the corresponding `snapshots/1/fs` directory did not exist. This meant that containerd's metadata and its actual overlayfs snapshot files were inconsistent, preventing Kubernetes from creating the Pod sandbox.

## Resolution

### 1. Resolve DiskPressure

Unnecessary system logs and package cache were cleaned:

```bash
sudo journalctl --vacuum-size=200M
sudo apt clean
```

Disk usage was then checked using:

```bash
df -h
```

The worker recovered from disk pressure and Kubernetes reported:

```text
NodeHasNoDiskPressure
```

### 2. Rebuild the Broken Containerd State

Because this was a disposable Kubernetes lab worker, the local containerd state was rebuilt.

Kubelet and containerd were stopped:

```bash
sudo systemctl stop kubelet
sudo systemctl stop containerd
```

The existing containerd data was preserved by moving it:

```bash
sudo mv /var/lib/containerd /var/lib/containerd.broken
```

A fresh containerd directory was created:

```bash
sudo mkdir -p /var/lib/containerd
sudo chown root:root /var/lib/containerd
```

Then containerd and kubelet were restarted:

```bash
sudo systemctl start containerd
sudo systemctl start kubelet
```

This allowed containerd to create a fresh snapshotter state instead of using the corrupted/missing snapshot reference.

Containerd was verified using:

```bash
sudo crictl info
```

The important results were:

```text
RuntimeReady: true
NetworkReady: true
lastCNILoadStatus: OK
```

This confirmed that the container runtime and Calico CNI were functioning correctly.

## Final Result

The complete failure sequence was:

```text
Low worker disk space
        ↓
DiskPressure
        ↓
Pod Pending
        ↓
Cleaned logs/cache
        ↓
NodeHasNoDiskPressure
        ↓
Pod successfully scheduled
        ↓
ContainerCreating
        ↓
Containerd missing overlayfs snapshot
        ↓
FailedCreatePodSandBox
        ↓
Rebuilt containerd local state
        ↓
RuntimeReady=True
NetworkReady=True
        ↓
Pod can be created normally
```

### Root Causes

There were **two separate problems**:

1. **Insufficient worker-node ephemeral storage**, which caused `DiskPressure` and prevented scheduling.
2. **Inconsistent containerd overlayfs snapshot state**, where containerd referenced a snapshot whose `fs` directory was missing, preventing creation of the Pod sandbox.

### Important Lesson

`Pending` and `ContainerCreating` indicate different stages of failure:

* `Pending` → Kubernetes could not schedule the Pod.
* `ContainerCreating` → Kubernetes scheduled the Pod, but kubelet/containerd could not create the containers.

For long-term stability, the worker node should have sufficient disk capacity (for example, **20–30 GB or more for a lab environment**) to accommodate Kubernetes, container images, logs, and ephemeral storage.
