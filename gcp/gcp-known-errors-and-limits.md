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

**Mitigation:** Always request at least 4 Gi. CDI's StorageProfile sets `minimumSupportedPvcSize: 4Gi` to handle this automatically, but the behavior depends on how the DataVolume is defined:

- **DataVolume with `storage`:** CDI applies the StorageProfile automatically. Requesting less than 4 Gi (e.g. 1 Gi) will result in a 4 Gi PVC.
- **DataVolume with `pvc`:** CDI does not apply the StorageProfile by default. To enable automatic size adjustment, add the label `cdi.kubevirt.io/applyStorageProfile: "true"` to the PVC.

For full details on Hyperdisk Balanced size and performance limits, see [Hyperdisk Balanced size limits](https://docs.cloud.google.com/compute/docs/disks/hd-types/hyperdisk-balanced#size_limits).

---

## 2. Volume Attachment Limit Per Node

**Error:**

```
FailedScheduling: 0/N nodes are available: 1 node(s) exceed max volume count
```

**When it occurs:** Attaching more volumes than the machine type allows. The GCP PD CSI driver reports `MaxVolumesPerNode` to Kubernetes via the `CSINode` object.

**Impact:** Pods requiring additional volume attachments beyond the limit cannot be scheduled. Hotplug operations stall at `AttachedToNode` and never transition to `Ready`.

**Details:** Most machine types default to **127** volumes per node. Some bare metal types have lower fixed limits, and `c3-metal` in particular defaults to only **15**. To raise `c3-metal` to 128, your GCP project must first be allowlisted by Google (project-level approval). Once allowlisted, apply the override label to all worker nodes (see below). For the full breakdown by machine type, see [GCP C3 disk and capacity limits](https://docs.cloud.google.com/compute/docs/general-purpose-machines#disk_and_capacity_limits_8).

**How to check the current limit on a node:**

```bash
oc get csinode <node-name> -o jsonpath='{.spec.drivers[?(@.name=="pd.csi.storage.gke.io")].allocatable.count}'
```

**Mitigation:** Spread volumes across multiple nodes, or override the limit by labeling worker nodes:

```bash
oc label nodes <worker-1> <worker-2> <worker-3> node-restriction.kubernetes.io/gke-volume-attach-limit-override=127 --overwrite
```

After applying the label, restart the CSI driver node pods for the new limit to take effect:

```bash
oc delete pod -n openshift-cluster-csi-drivers -l app=gcp-pd-csi-driver-node
```

The valid override range is 1-127. For a full step-by-step override procedure (including required RBAC and CSI driver image changes), see [GCP PD CSI Volume Attachment Limit Override](gcp-volume-attachment-limit-override.md).

> **Note:** The node boot disk is considered an attachable disk, so the effective usable limit is one less than the reported value.

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

## 6. No Native RWX Filesystem Support

**Symptom:** GCP PD only provides ReadWriteOnce (RWO) volumes. There is no native RWX Filesystem support.

**When it occurs:** Currently, CBT (Changed Block Tracking) across live migration requires RWX Filesystem for `vmStateStorageClass` due to a libvirt bug ([RHEL-113574](https://issues.redhat.com/browse/RHEL-113574)). This is expected to be resolved in a future libvirt update, after which RWO should be sufficient. CBT without live migration (e.g. restart) already works with RWO Block.

**Impact:** Until the libvirt fix is available, CBT across live migration on GCP requires an additional RWX-capable storage provider.

**Mitigation:** If CBT across live migration is needed before the fix, deploy a separate RWX Filesystem storage solution (e.g. GCP Filestore CSI, NFS) alongside GCP PD.

---

## Quick Reference

| Limit / Error | Value | Impact |
|---------------|-------|--------|
| Minimum disk size (Hyperdisk Balanced) | 4 GB | PVC creation fails below this |
| Volume attachment limit per node | 3-127 (varies by machine type) | Pods fail to schedule beyond this |
| Storage pool IOPS overprovisioning | Pool-defined limit | PVC creation fails when pool IOPS exceeded |
| Storage pool throughput overprovisioning | Pool-defined limit | PVC creation fails when pool throughput exceeded |
| Disk resize rate limit | 1 resize at a time per disk | Concurrent/rapid resizes fail |
| RWX Filesystem support | Not available (RWO only) | CBT across live migration temporarily requires separate RWX storage |
