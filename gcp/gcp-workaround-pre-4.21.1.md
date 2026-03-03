# KubeVirt Storage Configuration for GCP (pre-4.21.1)

**Applicable to:** OpenShift Virtualization (KubeVirt) versions **before 4.21.1**.

- For **4.21.x**, see [gcp-storage-configuration-4.21.md](gcp-storage-configuration-4.21.md) (simplified steps; CDI StorageProfile annotations).
- For **4.22 and above**, see [gcp-storage-configuration-4.22.md](gcp-storage-configuration-4.22.md) (automatic VolumeSnapshotClass provisioning).

---

This configuration enables key KubeVirt capabilities like Live Migration and performing more than 6 restores per hour from a single snapshot on Google Cloud Platform (GCP). It uses a VolumeSnapshotClass with the `snapshot-type: images` parameter.

## Why This Configuration is Needed

In KubeVirt, VMs are efficiently created by restoring VM disks from image snapshots.

On GCP, without this configuration you cannot simultaneously:

- Restore image snapshots to ReadWriteMany (RWX) PVCs (required for Live Migration)
- Perform more than 6 restores per hour from a single snapshot

This configuration enables both capabilities.

## Prerequisites

- OpenShift Virtualization deployed on GCP
- GCP PD CSI driver installed and configured
- Hyperdisk StorageClass available (specified as Step 0)
- Cluster admin access to create and modify StorageClass, VolumeSnapshotClass, StorageProfile, and HyperConverged resources

## Configuration Approach

This configuration creates dedicated storage resources for images. Your existing default storage configuration for VM operations remains unchanged.

**After configuration, your cluster will have:**

- A new StorageClass `virt-os-images-rwo` (RWO only) for when importing images
- A new VolumeSnapshotClass `csi-gce-pd-vsc-images` with `snapshot-type: images`
- An automatically created StorageProfile `virt-os-images-rwo` that you will configure with RWO access modes and point to the `csi-gce-pd-vsc-images` VolumeSnapshotClass
- Your original default StorageClass and VolumeSnapshotClass will stay unchanged

## Configuration Steps

### Step 0: Create a Hyperdisk StorageClass (prerequisite)

If you do not already have a default Hyperdisk StorageClass, create your primary Hyperdisk storage that will be used for regular VM snapshots and for the golden-image StorageClass in Step 1.

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

### Step 0.1: Remove default from standard-csi StorageClass

GCP clusters come with a `standard-csi` StorageClass set as default. After creating the Hyperdisk StorageClass as the new default, remove the default annotation from `standard-csi` to avoid having two default StorageClasses:

```bash
oc annotate storageclass standard-csi storageclass.kubernetes.io/is-default-class-
```

Verify only your Hyperdisk StorageClass is marked as default:

```bash
oc get storageclass
```

### Step 1: Create a StorageClass for Hyperdisk Golden Images

Duplicate your existing GCP Hyperdisk StorageClass and create `virt-os-images-rwo`.

1. Get your existing StorageClass:
   ```bash
   oc get storageclass <your-existing-storageclass-name> -o yaml
   ```

2. Create a new file with:
   - `name: virt-os-images-rwo`
   - Ensure `virt-os-images-rwo` is NOT set as the default StorageClass (either omit default class annotations or set them to `"false"`)
   - Remove auto-generated fields like `resourceVersion`, `uid`, `creationTimestamp`, `generation`, and `managedFields`

Example:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: virt-os-images-rwo
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
    storageclass.kubevirt.io/is-default-virt-class: "false"
allowVolumeExpansion: true
parameters:
  storage-pools: projects/<project-name>/zones/<location+zone>/storagePools/<pool-name>
  type: hyperdisk-balanced
provisioner: pd.csi.storage.gke.io
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

Apply:

```bash
oc apply -f <your-sc-file.yaml>
```

### Step 2: Create a VolumeSnapshotClass with Special Parameter

Duplicate your existing VolumeSnapshotClass and create `csi-gce-pd-vsc-images`.

1. Get your existing VolumeSnapshotClass:
   ```bash
   oc get volumesnapshotclass <your-existing-vsc-name> -o yaml
   ```

2. Create a new file with:
   - `name: csi-gce-pd-vsc-images`
   - Add `snapshot-type: images` under `parameters`
   - Ensure `csi-gce-pd-vsc-images` is NOT set as the default VolumeSnapshotClass (either omit default class annotations or set them to `"false"`)
   - Remove auto-generated fields like `resourceVersion`, `uid`, `creationTimestamp`, `generation`, and `managedFields`

Example:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-gce-pd-vsc-images
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "false"
deletionPolicy: Delete
driver: pd.csi.storage.gke.io
parameters:
  snapshot-type: images
```

Apply:

```bash
oc apply -f <your-vsc-file.yaml>
```

### Step 3: Configure StorageProfile

KubeVirt automatically creates a StorageProfile after Step 1. Configure it to use RWO access modes and the `csi-gce-pd-vsc-images` VolumeSnapshotClass:

Edit:

```bash
oc edit storageprofile virt-os-images-rwo
```

In the `spec` section:

- Copy the `claimPropertySets` structure from the `status` section to the `spec` section, but keep only ReadWriteOnce access modes for both Block and Filesystem volume modes, and remove ReadWriteMany (RWX) if present
- Set `snapshotClass: csi-gce-pd-vsc-images`

Example `spec` section:

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: StorageProfile
metadata:
  name: virt-os-images-rwo
spec:
  claimPropertySets:
  - accessModes:
    - ReadWriteOnce
    volumeMode: Block
  - accessModes:
    - ReadWriteOnce
    volumeMode: Filesystem
  snapshotClass: csi-gce-pd-vsc-images
```

### Step 4: Update HyperConverged CR for Golden Images

Configure `dataImportCronTemplates` to use `virt-os-images-rwo` StorageClass.

Edit (replace `<namespace>` with your namespace, typically `kubevirt-hyperconverged` or `openshift-cnv`):

```bash
oc edit hyperconverged kubevirt-hyperconverged -n <namespace>
```

In the `spec` section:

- If `dataImportCronTemplates` are in `status` but not in `spec`, copy them to `spec.dataImportCronTemplates`
- Set `storageClassName: virt-os-images-rwo` for each template in `spec.dataImportCronTemplates`:

```yaml
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
spec:
  dataImportCronTemplates:
  - metadata:
      name: <your-os-image-name>
    spec:
      template:
        spec:
          source:
            registry:
              url: <your-image-url>
          storage:
            storageClassName: virt-os-images-rwo  # Set this field for each template
            resources:
              requests:
                storage: <size>
```

### Step 5: Delete Existing Snapshots to Trigger Re-import

Now that the configuration is complete, delete existing golden image snapshots to trigger re-import with the new `virt-os-images-rwo` StorageClass and `csi-gce-pd-vsc-images` VolumeSnapshotClass:

List snapshots:

```bash
oc get volumesnapshot --all-namespaces --selector cdi.kubevirt.io/dataImportCron
```

Delete:

```bash
oc delete volumesnapshot --all-namespaces --selector cdi.kubevirt.io/dataImportCron
```

DataImportCron will automatically trigger a new import using the new configuration.

### Step 6: Verify the Configuration

After golden images are re-imported, verify:

Snapshots use `csi-gce-pd-vsc-images`:

```bash
oc get volumesnapshot --all-namespaces -o yaml | grep csi-gce-pd-vsc-images
```

StorageProfile is configured:

```bash
oc get storageprofile virt-os-images-rwo -o yaml
```

Confirm:

- Only ReadWriteOnce access modes are present in `spec.claimPropertySets`
- `spec.snapshotClass` is set to `csi-gce-pd-vsc-images`

VM disks use default StorageClass:

```bash
oc get pvc -n <vm-namespace>
```

VM PVCs should show the default StorageClass, not `virt-os-images-rwo`.

## How It Works

- Images are imported as RWO PVCs using the `virt-os-images-rwo` StorageClass
- Snapshots are created with `snapshot-type: images`, enabling:
  - Restoring from RWO snapshots to RWX PVCs (for Live Migration)
  - More than 6 restores per hour from a single snapshot
- When VMs are created, snapshots are restored to RWX volumes using the default StorageClass

## Important Considerations

- Do not set `virt-os-images-rwo` StorageClass or `csi-gce-pd-vsc-images` VolumeSnapshotClass as default classes
- Regular VM operations continue using the original default StorageClass and VolumeSnapshotClass
- VM disks created from images use the default StorageClass (typically RWX), not `virt-os-images-rwo`
- Snapshots of VM disks use the default VolumeSnapshotClass (without `snapshot-type: images`), not `csi-gce-pd-vsc-images`
- The default VolumeSnapshotClass is limited to 6 restores per hour from a single snapshot
- Golden image imports (via DataImportCron) use the `virt-os-images-rwo` StorageClass

## Additional Notes

**Custom Image Uploads:**

- For custom images uploaded (not from golden images), use the `virt-os-images-rwo` StorageClass to ensure snapshots can be restored more than 6 times per hour
- Using `virt-os-images-rwo` StorageClass stores the uploaded image as a RWO PVC, and snapshots created from this PVC will use the `csi-gce-pd-vsc-images` VolumeSnapshotClass (with `snapshot-type: images`)
- These snapshots can be restored to RWX PVCs and support more than 6 restores per hour
- If you upload custom images using a different StorageClass, they will not benefit from these capabilities and will be subject to the 6 restores per hour limitation

**After Setup:** Once golden images are re-imported with the correct configuration, the StorageProfile's explicit `snapshotClass` setting ensures they continue using the `csi-gce-pd-vsc-images` VolumeSnapshotClass.

## Troubleshooting

### Snapshots still using old VolumeSnapshotClass after re-import

**Symptom:** After completing the configuration, snapshots show a different VolumeSnapshotClass than `csi-gce-pd-vsc-images`.

**Solution:**

1. Verify the StorageProfile `virt-os-images-rwo` has `spec.snapshotClass: csi-gce-pd-vsc-images`:
   ```bash
   oc get storageprofile virt-os-images-rwo -o yaml | grep snapshotClass
   ```
2. If missing or incorrect, update the StorageProfile as described in Step 3.
3. Delete existing snapshots again to trigger re-import with the correct configuration.

### Still hitting 6 restores per hour limitation

**Symptom:** Creating multiple VMs from the same golden image fails after 6 restores per hour.

**Solution:**

1. Verify snapshots use `csi-gce-pd-vsc-images` VolumeSnapshotClass:
   ```bash
   oc get volumesnapshot --all-namespaces -o yaml | grep snapshotClassName
   ```
2. Verify the VolumeSnapshotClass has `snapshot-type: images`:
   ```bash
   oc get volumesnapshotclass csi-gce-pd-vsc-images -o yaml | grep snapshot-type
   ```
3. If either is incorrect, review Steps 2 and 3 to ensure proper configuration.

### StorageProfile not automatically created

**Symptom:** After creating the `virt-os-images-rwo` StorageClass, no StorageProfile is created.

**Solution:**

1. Wait a few minutes for KubeVirt to detect the new StorageClass.
2. Check if StorageProfile exists:
   ```bash
   oc get storageprofile virt-os-images-rwo
   ```
3. If it doesn't exist, verify the StorageClass was created correctly and check KubeVirt operator logs.

### Golden images not re-importing after deleting snapshots

**Symptom:** After deleting snapshots in Step 5, golden images don't re-import.

**Solution:**

1. Check DataImportCron status:
   ```bash
   oc get dataimportcron -A
   ```
2. Verify the HyperConverged CR has `storageClassName: virt-os-images-rwo` in `spec.dataImportCronTemplates`.
3. Check DataImportCron events for errors:
   ```bash
   oc describe dataimportcron <name> -n <namespace>
   ```

### VM creation fails with storage errors

**Symptom:** VMs fail to create with errors related to storage or snapshots.

**Solution:**

1. Verify VM disks are using the default StorageClass (not `virt-os-images-rwo`):
   ```bash
   oc get pvc -n <vm-namespace>
   ```
2. Check that the default StorageClass supports RWX access mode (required for Live Migration).
3. Verify the golden image snapshot exists and is ready:
   ```bash
   oc get volumesnapshot --all-namespaces
   ```
