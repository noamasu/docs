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

**Mitigation:** Always request at least 4 Gi. CDI's StorageProfile sets `minimumSupportedPvcSize: 4Gi` to handle this automatically, but the behavior depends on how the volume is created:

- **DataVolume with `spec.storage`:** CDI applies the StorageProfile automatically. Requesting less than 4 Gi (e.g. 1 Gi) will result in a 4 Gi PVC. This is the recommended approach.
- **DataVolume with `spec.pvc`:** CDI does not apply the StorageProfile. Request at least 4 Gi explicitly.
- **Standalone PVC (not created via DataVolume):** Add the label `cdi.kubevirt.io/applyStorageProfile: "true"` to the PVC to enable automatic size adjustment via the CDI mutating webhook.

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

**How to check the current limit per node:**

```bash
oc get csinode -o custom-columns="NAME:.metadata.name,MAX-VOLUMES:.spec.drivers[0].allocatable.count"
```

**Mitigation:** Spread volumes across multiple nodes, or override the limit by labeling worker nodes:

```bash
oc label nodes <worker-1> <worker-2> <worker-3> node-restriction.kubernetes.io/gke-volume-attach-limit-override=127 --overwrite
```

After applying the label, restart the CSI driver node pods for the new limit to take effect, then re-run the check command above to confirm the updated values:

```bash
oc delete pod -n openshift-cluster-csi-drivers -l app=gcp-pd-csi-driver-node
```

The valid override range is 1-127. The values reported by the CSI driver already account for the node boot disk (GCP limits are reduced by one before being reported to Kubernetes).

> **Important:** OCP version **4.21.5** or later is required for the volume attachment limit override to work correctly, as it contains the necessary fixes in the GCP PD CSI driver.

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

## 5. Online Resize Not Supported in RWX (Multi-Writer) Mode

**Error:**

```
googleapi: Error 400: Size of disks of type hyperdisk-balanced in READ_WRITE_MANY mode cannot be updated when they are attached., badRequest
```

**When it occurs:** Attempting to resize (expand) a PVC while it is attached to a running VM and the underlying Hyperdisk Balanced volume is in `READ_WRITE_MANY` (RWX) mode. Resizing attached volumes in `READ_WRITE_SINGLE` (RWO) mode works as expected.

**Impact:** PVC expansion fails with `ControllerResizeError`. The VM does not see the size change.

**Mitigation:** To resize a Hyperdisk Balanced volume in RWX mode, detach it from all VMs first (stop the VM), perform the resize, then reattach. For details, see [Modify a Hyperdisk volume](https://docs.google.com/compute/docs/disks/modify-hyperdisks#capacity_changes).

For reference: [CNV-75698](https://issues.redhat.com/browse/CNV-75698)

---

## 6. No Native RWX Filesystem Support

**Symptom:** The GCP PD CSI driver supports RWX in Block mode (multi-writer, up to 8 instances) but does not support RWX in Filesystem mode.

**When it occurs:** CBT (Changed Block Tracking) across live migration currently requires RWX Filesystem for `vmStateStorageClass`. This requirement is caused by two libvirt bugs:

- [RHEL-113574](https://issues.redhat.com/browse/RHEL-113574): Migration fails when the qcow2 overlay with a data-file is not on shared storage. An upstream fix exists but has not been officially released.
- [RHEL-145769](https://issues.redhat.com/browse/RHEL-145769): Even with the above fix applied, dirty bitmaps (used by CBT) are not transferred during live migration. After migration, the next backup falls back to a full backup since no bitmaps exist on the target. A fix is in progress but has not been officially released.

Once both fixes are released and shipped as part of the CNV bundle (in a libvirt RPM containing the fixes), RWO storage should be sufficient for CBT across live migration, removing the need for a separate RWX provider on GCP.

CBT without live migration (e.g. restart) already works with RWO Block and is not affected by these issues.

**Impact:** Until both libvirt fixes are officially released and shipped, CBT across live migration on GCP requires an additional RWX-capable storage provider.

**Mitigation:** If CBT across live migration is needed before the fixes are available, deploy a separate RWX Filesystem storage solution (e.g. Google Cloud NetApp Volumes, GCP Filestore CSI, NFS) alongside GCP PD.

---

## Quick Reference

| Limit / Error | Value | Impact |
|---------------|-------|--------|
| Minimum disk size (Hyperdisk Balanced) | 4 GB | PVC creation fails below this |
| Volume attachment limit per node | 3-127 (varies by machine type) | Pods fail to schedule beyond this |
| Storage pool IOPS overprovisioning | Pool-defined limit | PVC creation fails when pool IOPS exceeded |
| Storage pool throughput overprovisioning | Pool-defined limit | PVC creation fails when pool throughput exceeded |
| Online resize in RWX mode | Not supported | Must detach volume from all VMs before resizing |
| RWX Filesystem support | Block only (no Filesystem) | CBT across live migration requires separate RWX Filesystem storage until libvirt fixes are shipped |
