# Known GCP Errors and Limitations for KubeVirt Storage

This document catalogues known errors, quotas, and platform limitations encountered when running KubeVirt (OpenShift Virtualization) on GCP with the `pd.csi.storage.gke.io` CSI driver and Hyperdisk Balanced storage. These issues originate on the GCP side and may surface during normal KubeVirt operations or testing.

---

## 1. Hyperdisk Balanced Minimum Disk Size (4 GB)

**Error:**

```
googleapi: Error 400: Disk size cannot be smaller than 4 GB for disk type hyperdisk-balanced., badRequest
```

**When it occurs:** Any PVC requesting less than 4 GB with a Hyperdisk Balanced StorageClass.

**Impact:** PVC stays in `Pending` state indefinitely. VMs that depend on the PVC will not start.

**Mitigation:** Always request at least 4 Gi, or use the `cdi.kubevirt.io/applyStorageProfile` label on PVCs so that CDI's StorageProfile `minimumSupportedPvcSize: 4Gi` auto-adjusts the size.

---

## 2. Volume Attachment Limit Per Node (15 volumes)

**Error:**

```
FailedScheduling: 0/N nodes are available: 1 node(s) exceed max volume count
```

**When it occurs:** Attaching more than ~15 Hyperdisk Balanced volumes to a single node. The CSI driver reports `allocatable.count: 15` on worker nodes.

**Impact:** Pods requiring additional volume attachments beyond the limit cannot be scheduled. Hotplug operations stall at `AttachedToNode` and never transition to `Ready`.

**Details:** This limit is reported by the GCP PD CSI driver in the `CSINode` object:

```bash
oc get csinode <node-name> -o jsonpath='{.spec.drivers[?(@.name=="pd.csi.storage.gke.io")].allocatable.count}'
```

**Mitigation:** Spread volumes across multiple nodes. For workloads requiring many volumes per node, investigate increasing the attachment limit via node labels (see [volume attachment limit override](https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/gce-pd-csi-driver#volume_limit)).

---

## 3. Storage Pool IOPS Overprovisioning Limit

**Error:**

```
googleapi: Error 400: Adding/updating the disk brings the storage pool's used iops to 52000,
which exceeds the overprovisioning limit of 50000. Please increase the storage pool's
provisioned iops., badRequest
```

**When it occurs:** Creating a new disk when the storage pool's total provisioned IOPS across all disks would exceed the pool's IOPS overprovisioning limit. Each disk in a storage pool is assigned a share of the pool's provisioned IOPS based on its size and type. The aggregate across all disks cannot exceed the pool's limit.

**Impact:** PVC provisioning fails. The disk is not created and the PVC stays in `Pending`.

**Details:** Hyperdisk Balanced IOPS scale at 6 IOPS per GiB with a baseline of 3,000 IOPS per instance. Per-disk and per-instance IOPS limits depend on disk size, machine type, and vCPU count (see [Persistent Disk performance overview](https://docs.cloud.google.com/compute/docs/disks/performance)). When using storage pools, the pool's provisioned IOPS acts as an additional aggregate cap.

**Mitigation:** Increase the storage pool's provisioned IOPS, or delete unused disks to free up IOPS headroom. You can check and adjust pool IOPS via the GCP Console or `gcloud compute storage-pools update`. See [Create a Hyperdisk pool](https://docs.cloud.google.com/compute/docs/disks/create-storage-pools) and [Hyperdisk performance and size limits](https://docs.cloud.google.com/compute/docs/disks/hyperdisk-performance).

---

## 4. Storage Pool Throughput Overprovisioning Limit

**Error:**

```
googleapi: Error 400: Adding/updating the disk brings the storage pool's used throughput to
10405 MiB/s, which exceeds the overprovisioning limit of 10240 MiB/s. Please increase the
storage pool's provisioned throughput., badRequest
```

**When it occurs:** Creating a new disk when the storage pool's total provisioned throughput across all disks would exceed the pool's throughput overprovisioning limit. Each disk in a storage pool is assigned a share of the pool's provisioned throughput. The aggregate across all disks cannot exceed the pool's limit.

**Impact:** PVC provisioning fails. The disk is not created and the PVC stays in `Pending`.

**Details:** Hyperdisk Balanced throughput scales at 0.28 MiBps per GiB with a baseline of 140 MiBps per instance. Per-disk and per-instance throughput limits depend on disk size, machine type, and vCPU count (see [Persistent Disk performance overview](https://docs.cloud.google.com/compute/docs/disks/performance)). When using storage pools, the pool's provisioned throughput acts as an additional aggregate cap.

**Mitigation:** Increase the storage pool's provisioned throughput, or delete unused disks to free up throughput headroom. You can check and adjust pool throughput via the GCP Console or `gcloud compute storage-pools update`. See [Create a Hyperdisk pool](https://docs.cloud.google.com/compute/docs/disks/create-storage-pools) and [Hyperdisk performance and size limits](https://docs.cloud.google.com/compute/docs/disks/hyperdisk-performance).

---

## 5. Disk Resize Rate Limiting

**Error:**

```
googleapi: Error 400: Invalid resource usage: 'Disk cannot be resized due to being rate limited.'
```

```
googleapi: Error 400: Invalid resource usage: 'Disk cannot be resized while there is an ongoing mutation.'
```

**When it occurs:** Performing rapid or concurrent PVC resize operations. A single resize works fine; back-to-back or parallel resizes on the same or multiple disks trigger the rate limit.

**Impact:** PVC expansion fails with `ControllerResizeError`. The CSI resizer backs off exponentially, and the resize may not complete within expected timeframes. The guest VM does not see the size change.

**Details observed in testing:**

| Scenario | Result |
|----------|--------|
| Single VM, single resize (e.g. 8 Gi to 16 Gi) | Succeeded |
| 10 VMs, single resize simultaneously | Succeeded (some took ~30 s for node-level FS resize) |
| 10 VMs, double resize (rapid back-to-back) | All 10 PVCs stuck; GCP returned rate-limit error |

**Mitigation:** Avoid rapid sequential resizes on the same disk. When performing bulk expansions, stagger them or allow each resize to complete before initiating the next.

---

## 6. Snapshot Name Conflict (409 Already Exists)

**Error:**

```
googleapi: Error 409: The resource 'projects/<project>/global/snapshots/snapshot-<id>' already exists, alreadyExists
```

**When it occurs:** A snapshot creation is retried (e.g. after a timeout) while the original snapshot is still being created in GCP.

**Impact:** Snapshot creation fails. Downstream operations that depend on the snapshot (restores, clones) are blocked.

**Mitigation:** This is typically a transient issue. Retrying after the in-flight snapshot completes usually resolves it. If persistent, delete the orphaned snapshot and recreate.

---

## 7. No Native RWX Filesystem Support

**Symptom:** Operations requiring ReadWriteMany (RWX) Filesystem PVCs fail because GCP PD only provides ReadWriteOnce (RWO).

**When it occurs:** Any KubeVirt feature that requires RWX Filesystem volumes, such as `vmStateStorageClass` for CBT live migration.

**Impact:** Features like CBT across live migration are not available without an additional RWX-capable storage provider.

**Mitigation:** Deploy a separate RWX Filesystem storage solution (e.g. GCP Filestore CSI, NFS) alongside GCP PD for features that require it.

---

## 8. Size Rounding and Filesystem Overhead

**Symptom:** Actual usable disk size differs from requested size. For example, a 4 Gi Filesystem PVC provides only ~3.2 GiB of usable space.

**When it occurs:** Any Filesystem-mode PVC on Hyperdisk Balanced. GCP PD also rounds up requested sizes to its own size tiers (e.g. 1 Gi to 4 Gi, 6 Gi to 8 Gi).

**Impact:**

- Block-to-Filesystem storage migrations can fail at the 4 Gi boundary: a 4 Gi block device (4,294,967,296 bytes) does not fit in a 4 Gi Filesystem PVC (~3,438,280,704 bytes usable after ext4 overhead).
- Tools like `lsblk` report usable size (~3.2 G), not nominal size (4 G), which can cause confusion.

**Details:**

| Requested | Actual PVC | Usable (Filesystem) | Overhead |
|-----------|-----------|---------------------|----------|
| 1 Gi | 4 Gi (minimum) | ~3.2 GiB | ~19% |
| 4 Gi | 4 Gi | ~3.2 GiB | ~19% |
| 5 Gi | 6 Gi | ~5.6 GiB | ~7% |
| 8 Gi | 8 Gi | ~7.6 GiB | ~5% |

**Mitigation:** Always account for filesystem overhead when sizing PVCs, especially at boundaries near the minimum size. For block-to-filesystem migrations, the destination PVC should be larger than the source block PVC.

---

## 9. API Throttling During Bulk Operations

**Symptom:**

```
"Waited before sending request" delay="N.Ns" reason="client-side throttling, not priority and fairness"
```

**When it occurs:** Bulk volume operations such as creating many DataVolumes or PVCs simultaneously.

**Impact:** API request delays escalate (observed: 1 s, 4.75 s, 8.55 s), slowing down status polling and causing operations to miss their timeouts.

**Mitigation:** Reduce the burst rate of bulk operations. Stagger volume creation rather than submitting all requests simultaneously.

---

## Quick Reference

| Limit / Error | Value | Impact |
|---------------|-------|--------|
| Minimum disk size (Hyperdisk Balanced) | 4 GB | PVC creation fails below this |
| Volume attachment limit per node | ~15 | Pods fail to schedule beyond this |
| Storage pool IOPS overprovisioning | Pool-defined limit | PVC creation fails when pool IOPS exceeded |
| Storage pool throughput overprovisioning | Pool-defined limit | PVC creation fails when pool throughput exceeded |
| Disk resize rate limit | 1 resize at a time per disk | Concurrent/rapid resizes fail |
| RWX Filesystem support | Not available (RWO only) | Features requiring RWX FS need separate storage |
| Filesystem overhead at 4 Gi | ~19% (~3.2 GiB usable) | Block-to-FS migration fails at minimum boundary |
