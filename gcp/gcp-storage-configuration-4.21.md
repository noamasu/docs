# KubeVirt Storage Configuration for GCP (4.21.x)

**Applicable to:** OpenShift Virtualization (KubeVirt) **4.21.x**.

- For versions **before 4.21.1**, see [gcp-workaround-pre-4.21.1.md](gcp-workaround-pre-4.21.1.md) (full manual configuration).
- For **4.22 and above**, see [gcp-storage-configuration-4.22.md](gcp-storage-configuration-4.22.md) (automatic VolumeSnapshotClass provisioning).

---

## Why This Configuration is Needed

On GCP, standard snapshots (using `pd.csi.storage.gke.io`) are limited to **6 restores per hour per snapshot**. Using a VolumeSnapshotClass with `snapshot-type: images` enables unlimited restores. Those image-type snapshots cannot be created from ReadWriteMany (RWX) PVCs—they must use RWO sources—but they can be restored to RWX volumes (e.g. for Live Migration). More details are available in the [GCP documentation](https://cloud.google.com/compute/docs/disks/storage-pools).

In KubeVirt this leads to two VolumeSnapshotClasses:

- **VSC with `snapshot-type: images`** — for DataImportCron (golden) images.
- **VSC without `snapshot-type: images`** — for regular VM snapshots.

In **4.21.1 and above**, CDI adds StorageProfile annotations so that DataImportCron automatically uses RWO and the images VSC (see note after Step 2). You still need to create the VolumeSnapshotClass with `snapshot-type: images` in 4.21 (it is not yet provisioned by the GCP PD CSI driver operator).

## Prerequisites

- OpenShift Virtualization **4.21.x** deployed on GCP
- GCP PD CSI driver installed and configured
- Cluster admin access to create a StorageClass and apply a VolumeSnapshotClass

## Configuration Steps

### Step 1: Create a Hyperdisk StorageClass (prerequisite)

If you do not already have a default Hyperdisk StorageClass, create it first. This will be used for regular VM snapshots and for VM disks.

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

### Step 1.1: Remove default from standard-csi StorageClass

GCP clusters come with a `standard-csi` StorageClass set as default. After creating the Hyperdisk StorageClass as the new default, remove the default annotation from `standard-csi` to avoid having two default StorageClasses:

```bash
oc annotate storageclass standard-csi storageclass.kubernetes.io/is-default-class-
```

Verify only your Hyperdisk StorageClass is marked as default:

```bash
oc get storageclass
```

### Step 2: Create the VolumeSnapshotClass for images

The GCP PD CSI driver operator does not yet provision this class in 4.21. Create it manually by applying the official asset:

```bash
oc apply -f https://raw.githubusercontent.com/openshift/gcp-pd-csi-driver-operator/main/assets/volumesnapshotclass_images.yaml
```

This creates a VolumeSnapshotClass named `csi-gce-pd-vsc-images` with `snapshot-type: images`. It is not set as the default, so regular VM snapshots continue to use your default VolumeSnapshotClass.

**Note (4.21.1 and above):** From 4.21.1 onward, CDI adds two annotations to the StorageProfile so that golden images are automatically created on the correct VSC—no user configuration is required. The StorageProfile will use:

- **`cdi.kubevirt.io/useReadWriteOnceForDataImportCron`** — RWO access mode for DataImportCron PVCs when not explicitly configured.
- **`cdi.kubevirt.io/snapshotClassForDataImportCron`** — The VolumeSnapshotClass name for DataImportCron (CDI can auto-discover a matching VSC with the required parameters).

For implementation details, see [containerized-data-importer PR #3991](https://github.com/kubevirt/containerized-data-importer/pull/3991).

## Verification

- Confirm the VolumeSnapshotClass exists:
  ```bash
  oc get volumesnapshotclass csi-gce-pd-vsc-images -o yaml | grep snapshot-type
  ```
  You should see `snapshot-type: images`.

- After golden images are imported, check that snapshots in the golden images namespace (e.g. `openshift-virtualization-os-images`) use `csi-gce-pd-vsc-images`:
  ```bash
  oc get volumesnapshot -n openshift-virtualization-os-images -o yaml | grep csi-gce-pd-vsc-images
  ```

## Summary

| Version   | What you do |
|----------|-------------|
| **Pre-4.21.1** | Full manual setup: dedicated StorageClass for images, manual VSC, StorageProfile spec, and HyperConverged `dataImportCronTemplates`. See [gcp-workaround-pre-4.21.1.md](gcp-workaround-pre-4.21.1.md). |
| **4.21.x**      | Create default Hyperdisk StorageClass (if needed) + apply [volumesnapshotclass_images.yaml](https://raw.githubusercontent.com/openshift/gcp-pd-csi-driver-operator/main/assets/volumesnapshotclass_images.yaml). From 4.21.1, StorageProfile annotations steer golden images to the correct VSC automatically. |
| **4.22+**       | Create default Hyperdisk StorageClass (if needed); VSC with `snapshot-type: images` is provisioned automatically by the GCP PD CSI driver operator. See [gcp-storage-configuration-4.22.md](gcp-storage-configuration-4.22.md). |
