# Known GCNV Errors and Limitations for KubeVirt Storage

This document catalogues known errors, quotas, and platform limitations encountered when running KubeVirt (OpenShift Virtualization) on Google Cloud with Google Cloud NetApp Volumes (GCNV) **Flex File** service level, provisioned via the NetApp Trident CSI driver (`csi.trident.netapp.io`). These issues originate on the GCNV/Trident side and may surface during normal KubeVirt operations or testing.

---

## 1. Minimum Volume Size (1 GiB)

**Error:**

```
rpc error: code = OutOfRange desc = unsupported capacity range;
requested volume size (253755392 bytes) is too small;
the minimum volume size is 1073741824 bytes
```

**When it occurs:** Any PVC requesting less than 1 GiB with a GCNV-backed StorageClass.

**Impact:** PVC stays in `Pending` state. Trident repeatedly rejects the provisioning request. VMs that depend on the PVC will not start.

**Details:** GCNV enforces a hard minimum of **1 GiB** (1,073,741,824 bytes) per volume for the Flex service level. See [GCNV volume limits](https://docs.cloud.google.com/netapp/volumes/docs/quotas#volume_limits). This affects small PVC requests common in VM workloads (e.g. memory dumps, small container disk imports).

**Mitigation:** CDI's StorageProfile for the GCNV StorageClass should have `minimumSupportedPvcSize: 1Gi`. The behavior depends on how the volume is created:

- **DataVolume with `spec.storage`:** CDI applies the StorageProfile automatically. Requesting less than 1 Gi will result in a 1 Gi PVC. This is the recommended approach.
- **DataVolume with `spec.pvc`:** CDI does not apply the StorageProfile. Request at least 1 Gi explicitly.
- **Standalone PVC (not created via DataVolume):** Add the label `cdi.kubevirt.io/applyStorageProfile: "true"` to the PVC to enable automatic size adjustment via the CDI mutating webhook.

---

## 2. Storage Pool Volume Limit (50 volumes per Flex pool)

**Error:**

```
rpc error: code = ResourceExhausted desc = pool has reached its maximum volume count
```

**When it occurs:** Creating a new volume when the storage pool already contains 50 volumes.

**Impact:** PVC provisioning fails. The PVC stays in `Pending`.

**Details:** GCNV Flex pools are limited to **50 volumes per pool**. See [GCNV storage pool limits](https://docs.cloud.google.com/netapp/volumes/docs/quotas#storage_pool_limits). This limit is reached quickly when running KubeVirt workloads that create many DVs/PVCs (e.g. test suites, many VMs, snapshots). Unlike GCP Hyperdisk storage pools (which have higher limits), GCNV requires multiple pools to scale.

**Mitigation:** Create multiple storage pools and list all of them in the `TridentBackendConfig`. Trident will distribute volumes across the available pools. For example, 16 pools with 1 TiB each provides capacity for up to 800 volumes:

```yaml
spec:
  storagePools:
    - my-cluster-flex-pool-1
    - my-cluster-flex-pool-2
    - my-cluster-flex-pool-3
    # ... add as many pools as needed
```

See [gcnv-storage-configuration.md](gcnv-storage-configuration.md) for the full setup guide.

---

## 3. No Block Volume Mode Support (NFS-only)

**Symptom:** GCNV volumes are NFS-backed and only support `volumeMode: Filesystem`. Any PVC or DataVolume requesting `volumeMode: Block` will fail to provision.

**When it occurs:** Tests or workloads that explicitly require raw block devices (e.g. CBT/changed block tracking, block-mode hotplug, block-mode volume migration).

**Impact:** DataVolumes requesting block mode stay in `WaitForFirstConsumer` or `ImportScheduled` indefinitely. Pods that depend on block PVCs will fail to schedule.

**Details:** The Trident `google-cloud-netapp-volumes` driver supports NFS protocol only. It provides:

- `ReadWriteOnce` (RWO)
- `ReadWriteMany` (RWX)
- `ReadOnlyMany` (ROX)

All in Filesystem mode. Block access modes (`ReadWriteOncePod` with Block, or any Block mode) are not supported.

**Impact on KubeVirt features:**

| Feature | Status |
|---|---|
| VM disk storage | Works (Filesystem mode) |
| Live migration | Works (RWX Filesystem) |
| Hotplug volumes (Filesystem) | Works |
| Hotplug volumes (Block) | Not supported |
| VM snapshots and restores | Works |
| VM export (non-gzipped) | Works |
| CBT / Incremental backup | Not supported (requires Block or RWX Filesystem with libvirt fixes) |

**Mitigation:** Use Filesystem mode for all volumes. If block-mode features are required, deploy a separate block-capable storage provider alongside GCNV.

---

## 4. Filesystem Overhead - "No Space Left on Device" During Gzipped Export

**Error:**

```
gzip: /data/disk.img.gz: No space left on device
```

**When it occurs:** Exporting a VM disk in gzipped format via `VirtualMachineExport`. The export downloads the raw disk image and runs `gzip` on the same PVC, exceeding the available space after NFS metadata overhead.

**Impact:** Gzipped export tests fail. Non-gzipped exports (RAW, archive, tarred gzipped) work correctly.

**Details:** NFS filesystem metadata consumes part of the provisioned volume capacity. The default CDI filesystem overhead (5.5%) is not enough for GCNV's NFS volumes. When CDI provisions the target PVC with the same size as the source, the actual usable space is reduced by NFS overhead, and operations that need temporary space on the same PVC (like gzip) fail.

The recommended filesystem overhead for GCNV is **10%** (`0.10`). This should be configured as part of the initial setup (see [gcnv-storage-configuration.md](gcnv-storage-configuration.md)).

**Mitigation:** Set the filesystem overhead for the GCNV StorageClass by patching the HyperConverged CR:

```bash
oc patch hyperconverged kubevirt-hyperconverged -n openshift-cnv --type=merge -p '
{
  "spec": {
    "filesystemOverhead": {
      "storageClass": {
        "gcnv-flex": "0.10"
      }
    }
  }
}'
```

Verify CDI picked up the new value:

```bash
oc get cdiconfig -o jsonpath='{.items[0].status.filesystemOverhead}'
```

You should see `gcnv-flex` listed with `0.10`.

---

## 5. Storage Pool Throughput and IOPS Sizing

**Symptom:** VM disk I/O is slower than expected, or multiple VMs on the same pool experience degraded performance under load.

**When it occurs:** The storage pool's throughput or IOPS is insufficient for the number and type of VM workloads running against it.

**Details:** GCNV Flex File pools have throughput and IOPS that are shared across all volumes in the pool. The default performance scales at **16 KiBps per GiB** of provisioned pool capacity. In [supported regions](https://docs.cloud.google.com/netapp/volumes/docs/discover/service-levels#supported_regions_for_flex_file_custom_performance), custom performance pools allow setting explicit throughput (up to 5 GiBps) and IOPS (up to 160,000) per pool.

For VM workloads, the Flex File service level is the most common choice due to its lower minimum pool size (1 TiB) and volume size (1 GiB). However, the default throughput (16 KiBps/GiB) may be insufficient for I/O-intensive VMs. For example, a 1 TiB pool at default performance provides only ~16 MiBps throughput. If running many VMs or I/O-heavy workloads, consider:

- **Custom performance pools** - allows setting explicit throughput (MiBps) and IOPS at pool creation time. Available in [supported regions](https://docs.cloud.google.com/netapp/volumes/docs/discover/service-levels#supported_regions_for_flex_file_custom_performance).
- **Spreading VMs across multiple pools** - distributes I/O load. Each pool has its own throughput/IOPS budget.

**Mitigation:** When creating storage pools, size the throughput and IOPS to match your expected VM workload. For Flex File custom-performance pools, throughput and IOPS are set at creation time.

For the full breakdown of service levels and performance characteristics, see [GCNV service levels](https://docs.cloud.google.com/netapp/volumes/docs/discover/service-levels). For storage pool creation with custom performance, see [Create a storage pool](https://docs.cloud.google.com/netapp/volumes/docs/configure-and-use/storage-pools/create-storage-pool).

---

## Quick Reference

| Limit / Error | Value | Impact |
|---|---|---|
| Minimum volume size (Flex) | 1 GiB | PVC creation fails below this |
| Maximum volumes per Flex pool | 50 | PVC creation fails when pool is full; use multiple pools |
| Block volume mode | Not supported | NFS provides Filesystem mode only |
| NFS filesystem overhead | ~5-10% | Gzipped exports may fail with "No space left" |
| Pool throughput / IOPS | Shared across all volumes in the pool | Size pools to match VM workload; use custom performance or multiple pools |
