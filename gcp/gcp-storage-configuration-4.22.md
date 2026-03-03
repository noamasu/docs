# KubeVirt Storage Configuration for GCP (4.22 and above)

**Applicable to:** OpenShift Virtualization (KubeVirt) **4.22 and above**.

- For versions **before 4.21.1**, see [gcp-workaround-pre-4.21.1.md](gcp-workaround-pre-4.21.1.md) (full manual configuration).
- For **4.21.x**, see [gcp-storage-configuration-4.21.md](gcp-storage-configuration-4.21.md) (create VSC manually; CDI StorageProfile annotations).

---

## What Changed in 4.22

Starting with **OpenShift Virtualization 4.22**, the [GCP PD CSI driver operator](https://github.com/openshift/gcp-pd-csi-driver-operator) automatically provisions an additional VolumeSnapshotClass with the `snapshot-type: images` parameter, as requested in [RFE-8550](https://issues.redhat.com/browse/RFE-8550): *Provision additional VolumeSnapshotClass with snapshot-type: images in gcp-pd-csi-driver-operator*.

## Why This Matters

On GCP, standard snapshots are limited to **6 restores per hour per snapshot**. A VolumeSnapshotClass with `snapshot-type: images` allows unlimited restores per hour, which is required for KubeVirt use cases such as creating many VMs from a single golden image (template) snapshot. Image-type snapshots must be created from RWO sources but can be restored to RWX volumes (e.g. for Live Migration).

With 4.22 and the GCP PD CSI driver operator updated for RFE-8550:

- **Two VolumeSnapshotClasses** are used in KubeVirt:
  - **VSC with `snapshot-type: images`** — used automatically for DataImportCron (golden) images.
  - **Default VSC (without `snapshot-type: images`)** — used for regular VM snapshots.
- **No manual creation** of the images VolumeSnapshotClass is required; the operator provisions it (typically named `csi-gce-pd-vsc-images`).

From **4.21.1 onward**, the StorageProfile has CDI annotations (`cdi.kubevirt.io/useReadWriteOnceForDataImportCron`, `cdi.kubevirt.io/snapshotClassForDataImportCron`) so that golden images are automatically created on the correct VSC. No user configuration is required.

## Configuration Steps

### Step 1: Create a Hyperdisk StorageClass (prerequisite)

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

### Step 1.1: Remove default from standard-csi StorageClass

GCP clusters come with a `standard-csi` StorageClass set as default. After creating the Hyperdisk StorageClass as the new default, remove the default annotation from `standard-csi` to avoid having two default StorageClasses:

```bash
oc annotate storageclass standard-csi storageclass.kubernetes.io/is-default-class-
```

Verify only your Hyperdisk StorageClass is marked as default:

```bash
oc get storageclass
```

In 4.22 and above, the images VolumeSnapshotClass is provisioned automatically by the GCP PD CSI driver operator; no separate YAML apply is needed.

## Reference for documentation (RFE-8550)

For the doc team: the automatic provisioning of the VolumeSnapshotClass with `snapshot-type: images` in 4.22 is delivered via [RFE-8550](https://issues.redhat.com/browse/RFE-8550) (*Provision additional VolumeSnapshotClass with snapshot-type: images in gcp-pd-csi-driver-operator*). The GCP PD CSI driver operator is updated to provision this additional VSC alongside the default one.

## Verification

- List VolumeSnapshotClasses and confirm the one with `snapshot-type: images` exists:
  ```bash
  oc get volumesnapshotclass -o yaml | grep -A2 "snapshot-type: images"
  ```
- After golden images are imported, confirm their snapshots use the images VSC:
  ```bash
  oc get volumesnapshot --all-namespaces -o yaml | grep snapshotClassName
  ```

## References

- [RFE-8550](https://issues.redhat.com/browse/RFE-8550) — Provision additional VolumeSnapshotClass with snapshot-type: images in gcp-pd-csi-driver-operator
- [GCP PD CSI driver operator – volumesnapshotclass_images.yaml](https://github.com/openshift/gcp-pd-csi-driver-operator/blob/main/assets/volumesnapshotclass_images.yaml) (same resource that the operator provisions in 4.22)
- [KubeVirt storage config for pre-4.21.1](gcp-workaround-pre-4.21.1.md) — Full manual workaround for older versions
