# KubeVirt Storage Configuration for GCP (4.22 and above)

**Applicable to:** OpenShift Virtualization (KubeVirt) **4.22 and above**.

- For versions **before 4.21.1**, see [gcp-workaround-pre-4.21.1.md](gcp-workaround-pre-4.21.1.md) (full manual configuration).
- For **4.21.x**, see [gcp-storage-configuration-4.21.md](gcp-storage-configuration-4.21.md) (create VSC manually; CDI StorageProfile annotations).

---

## Overview

Running OpenShift Virtualization on GCP requires a few storage configuration steps that are not set up by default. This guide walks you through all of them.

**1. Hyperdisk StorageClass:** GCP clusters ship with a `standard-csi` StorageClass that uses standard persistent disks. For KubeVirt workloads you need a Hyperdisk Balanced StorageClass backed by a storage pool, set as the cluster default.

**2. Volume attachment limits:** Some GCP machine types report a low volume attachment limit (e.g. 15) to Kubernetes. If you plan to run many VMs per node, you should verify and potentially override this limit.

**3. VolumeSnapshotClass with `snapshot-type: images`:** On GCP, standard snapshots are limited to **6 restores per hour per snapshot**. Using a VolumeSnapshotClass with `snapshot-type: images` removes this limit. Image-type snapshots must be created from RWO sources, but can be restored to RWX volumes (e.g. for Live Migration). In **4.22 and above**, the GCP PD CSI driver operator automatically provisions this VolumeSnapshotClass ([RFE-8550](https://issues.redhat.com/browse/RFE-8550)), so no manual creation is needed. CDI automatically steers DataImportCron to use RWO and the correct VSC via StorageProfile annotations.

**What you will do in this guide:**

1. [Create a default Hyperdisk StorageClass](#step-1-create-a-hyperdisk-storageclass)
2. [Verify volume attachment limits per node](#step-2-verify-volume-attachment-limits)

The VolumeSnapshotClass with `snapshot-type: images` is provisioned automatically by the operator, so no additional steps are required.

## CDI StorageProfile Annotations

From **4.21.1 onward**, CDI adds two annotations to the StorageProfile so that DataImportCron images are automatically created on the correct VSC. No user configuration is required:

- **`cdi.kubevirt.io/useReadWriteOnceForDataImportCron`** — RWO access mode for DataImportCron PVCs when not explicitly configured.
- **`cdi.kubevirt.io/snapshotClassForDataImportCron`** — The VolumeSnapshotClass name for DataImportCron (CDI can auto-discover a matching VSC with the required parameters).

For implementation details, see [containerized-data-importer PR #3991](https://github.com/kubevirt/containerized-data-importer/pull/3991).

## Configuration Steps

### Step 1: Create a Hyperdisk StorageClass

All versions require a default Hyperdisk StorageClass for VM disks and snapshots. If you do not already have one, create it.

Create a file named `hyperdisk-storageclass.yaml` with the following content:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sp-balanced-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
    storageclass.kubevirt.io/is-default-virt-class: "true"
allowVolumeExpansion: true
parameters:
  storage-pools: projects/<project-name>/zones/<location+zone>/storagePools/<pool-name>
  type: hyperdisk-balanced
provisioner: pd.csi.storage.gke.io
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

**Important:**

- Configure this StorageClass as the cluster default and as the KubeVirt default (virt-default).
- Specifying `storage-pools` is highly recommended; it enables thin-provisioning and other benefits described [here](https://docs.cloud.google.com/compute/docs/disks/storage-pools).
- Replace `<project-name>`, `<location+zone>`, and `<pool-name>` with values for your environment. To create a storage pool, see [Create a Hyperdisk pool](https://docs.cloud.google.com/compute/docs/disks/create-storage-pools).

Apply:

```bash
oc apply -f hyperdisk-storageclass.yaml
```

GCP clusters come with a `standard-csi` StorageClass set as default. Remove the default annotation from `standard-csi` to avoid having two default StorageClasses:

```bash
oc annotate storageclass standard-csi storageclass.kubernetes.io/is-default-class-
```

Verify only your Hyperdisk StorageClass is marked as default:

```bash
oc get storageclass
```

### Step 2: Verify volume attachment limits

Check the maximum volume attachment limit reported by each node:

```bash
oc get csinode -o custom-columns="NAME:.metadata.name,MAX-VOLUMES:.spec.drivers[0].allocatable.count"
```

If any node shows a low value (e.g. 15), you may need to apply an override label before running workloads at scale. See [Volume Attachment Limit Per Node](gcp-known-errors-and-limits.md#2-volume-attachment-limit-per-node) for details and override instructions.

In 4.22 and above, the images VolumeSnapshotClass is provisioned automatically by the GCP PD CSI driver operator; no separate YAML apply is needed.

## Reference for documentation (RFE-8550)

For the doc team: the automatic provisioning of the VolumeSnapshotClass with `snapshot-type: images` in 4.22 is delivered via [RFE-8550](https://issues.redhat.com/browse/RFE-8550) (*Provision additional VolumeSnapshotClass with snapshot-type: images in gcp-pd-csi-driver-operator*). The GCP PD CSI driver operator is updated to provision this additional VSC alongside the default one.

## Verification

- List VolumeSnapshotClasses and confirm the one with `snapshot-type: images` exists:
  ```bash
  oc get volumesnapshotclass -o yaml | grep -A2 "snapshot-type: images"
  ```
- After OS images are imported, confirm their snapshots use the images VSC:
  ```bash
  oc get volumesnapshot --all-namespaces -o yaml | grep snapshotClassName
  ```

## References

- [RFE-8550](https://issues.redhat.com/browse/RFE-8550) — Provision additional VolumeSnapshotClass with snapshot-type: images in gcp-pd-csi-driver-operator
- [GCP PD CSI driver operator – volumesnapshotclass_images.yaml](https://github.com/openshift/gcp-pd-csi-driver-operator/blob/main/assets/volumesnapshotclass_images.yaml) (same resource that the operator provisions in 4.22)
- [KubeVirt storage config for pre-4.21.1](gcp-workaround-pre-4.21.1.md) — Full manual workaround for older versions
