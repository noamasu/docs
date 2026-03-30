# KubeVirt Storage Configuration for Google Cloud NetApp Volumes (GCNV)

**Applicable to:** OpenShift Virtualization **4.21.2+** on **Red Hat OpenShift Container Platform 4.21+** (GCP) with NetApp Trident CSI driver.

---

## Overview

Google Cloud NetApp Volumes (GCNV) provides NFS-based shared storage for OpenShift Virtualization workloads on Google Cloud. GCNV supports **ReadWriteMany (RWX) in Filesystem mode**, which enables VM live migration, and is provisioned through the **NetApp Trident CSI driver**.

This guide walks you through Trident installation and configuration on an OpenShift cluster that already has GCNV resources provisioned on the Google Cloud side.

**What you will do in this guide:**

1. [Install Trident](#step-1-install-trident)
2. [Create a TridentBackendConfig for GCNV](#step-2-create-a-tridentbackendconfig)
3. [Create the StorageClass](#step-3-create-the-storageclass)
4. [Create a VolumeSnapshotClass](#step-4-create-the-volumesnapshotclass)
5. [Set the filesystem overhead](#step-5-set-the-filesystem-overhead)
6. [Verify the setup](#verification)

## Prerequisites

- **Red Hat OpenShift Container Platform 4.21+** deployed on Google Cloud (including OpenShift Dedicated)
- **OpenShift Virtualization 4.21.2+** installed
- Cluster admin access (`oc login` as `kubeadmin` or equivalent)
- A **GCP project** with the [Google Cloud NetApp Volumes API enabled](https://cloud.google.com/netapp/volumes/docs/get-started/configure-access/workflow#before_you_begin)
- A **GCP service account** with the `roles/netapp.admin` role and a **JSON key file** (referred to as `trident-admin.json` throughout this guide). To create one:

  ```bash
  gcloud iam service-accounts keys create trident-admin.json \
    --iam-account="<your-service-account>@<your-project-id>.iam.gserviceaccount.com"
  ```

  For details on creating the service account itself and assigning the required roles, see [Prepare to configure a GCNV backend](https://docs.netapp.com/us-en/trident/trident-use/gcnv-prep.html) and [Create a service account key](https://cloud.google.com/iam/docs/keys-create-delete#creating).
- **Private Service Access (PSA)** configured on the cluster's VPC network - see [Set up access to Google Cloud NetApp Volumes](https://cloud.google.com/netapp/volumes/docs/get-started/configure-access/workflow#before_you_begin)
- **One or more GCNV storage pools** created in the same zone as your worker nodes - see [Create a storage pool](https://docs.cloud.google.com/netapp/volumes/docs/configure-and-use/storage-pools/create-storage-pool)

> **Storage pool sizing:** GCNV Flex pools are limited to **50 volumes per pool** ([GCNV storage pool limits](https://docs.cloud.google.com/netapp/volumes/docs/quotas#storage_pool_limits)). Each Flex pool has a minimum capacity of 1 TiB. Create multiple pools and list them all in the TridentBackendConfig - Trident distributes volumes across pools automatically. For example, 4 pools support ~200 volumes, 16 pools support ~800 volumes. Flex pools are **zonal** and must be in the same zone as your worker nodes.

For the full GCNV preparation checklist (service account, PSA, storage pools, API key), see [Prepare to configure a GCNV backend](https://docs.netapp.com/us-en/trident/trident-use/gcnv-prep.html).

---

## Configuration Steps

### Step 1: Install Trident

Choose one of the two options below.

#### Option A: Via the OpenShift Web Console (operator)

1. Navigate to **Ecosystem > Software Catalog** and search for **"Trident"**
2. Install the **NetApp Trident** operator (version 26.02.0 or later)
3. Once installed, go to **Installed Operators** and open **Trident**
4. Click **Create TridentOrchestrator**
5. Before clicking **Create**, add `enableConcurrency: true` to the spec:

```yaml
apiVersion: trident.netapp.io/v1
kind: TridentOrchestrator
metadata:
  name: trident
spec:
  namespace: trident
  enableConcurrency: true
```

6. Click **Create**

For full details, see [Manually deploy the Trident operator](https://docs.netapp.com/us-en/trident/trident-get-started/kubernetes-deploy-operator.html).

#### Option B: Via `tridentctl` (CLI)

Download the Trident installer and run `tridentctl install`:

```bash
export TRIDENT_VERSION="26.02.0"

wget "https://github.com/NetApp/trident/releases/download/v${TRIDENT_VERSION}/trident-installer-${TRIDENT_VERSION}.tar.gz"
tar -xf "trident-installer-${TRIDENT_VERSION}.tar.gz"
cd trident-installer

./tridentctl install --enable-concurrency -n trident --kubeconfig "${KUBECONFIG}"
```

For full details, see [Install using tridentctl](https://docs.netapp.com/us-en/trident/trident-get-started/kubernetes-deploy-tridentctl.html).

#### Verify

Regardless of which option you chose, verify Trident is running:

```bash
oc get torc trident -o jsonpath='{.status.status}'
```

Expected output: `Installed`. Verify the pods:

```bash
oc get pods -n trident
```

You should see a `trident-controller`, `trident-node-linux` DaemonSet pods, and the `trident-operator` pod all in `Running` state.

---

### Step 2: Create the Credentials Secret and TridentBackendConfig

This step creates the Secret and TridentBackendConfig together, following the pattern from the [official NetApp GCNV examples](https://docs.netapp.com/us-en/trident/trident-use/gcnv-examples.html#example-configurations). The Secret holds the sensitive credential fields (`private_key` and `private_key_id`) from your `trident-admin.json` key file, and the TridentBackendConfig references it via `credentials.name`.

List **all** of your storage pools in the `storagePools` array - Trident will distribute volumes across them, which is critical given the [50-volume-per-pool limit](gcnv-known-errors-and-limits.md#2-storage-pool-volume-limit-50-volumes-per-flex-pool). For the full list of backend configuration parameters, see [GCNV backend configuration options](https://docs.netapp.com/us-en/trident/trident-use/gcnv-examples.html#backend-configuration-options).

Get your GCP project number:

```bash
gcloud projects describe "${GCP_PROJECT_ID}" --format="value(projectNumber)"
```

Create a file named `gcv-backend-flex.yaml` with the following content. Replace all `<placeholder>` values:

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: trident-admin
  namespace: trident
type: Opaque
stringData:
  private_key_id: "<private_key_id from trident-admin.json>"
  private_key: |
    -----BEGIN PRIVATE KEY-----
    <private_key from trident-admin.json>
    -----END PRIVATE KEY-----
---
apiVersion: trident.netapp.io/v1
kind: TridentBackendConfig
metadata:
  name: gcv-backend-flex
  namespace: trident
spec:
  version: 1
  storageDriverName: google-cloud-netapp-volumes
  backendName: gcnv-flex
  projectNumber: "<your-project-number>"
  location: "<worker-node-zone>"
  storage:
    - labels:
        performance: flex
      serviceLevel: flex
  storagePools:
    - <pool-prefix>-flex-pool-1
    - <pool-prefix>-flex-pool-2
    - <pool-prefix>-flex-pool-3
    - <pool-prefix>-flex-pool-4
  apiKey:
    type: service_account
    project_id: "<your-gcp-project-id>"
    client_email: "trident-admin@<your-gcp-project-id>.iam.gserviceaccount.com"
    client_id: "<client-id-from-json>"
    auth_uri: https://accounts.google.com/o/oauth2/auth
    token_uri: https://oauth2.googleapis.com/token
    auth_provider_x509_cert_url: https://www.googleapis.com/oauth2/v1/certs
    client_x509_cert_url: "https://www.googleapis.com/robot/v1/metadata/x509/trident-admin%40<your-gcp-project-id>.iam.gserviceaccount.com"
  credentials:
    name: trident-admin
```

> **Tip:** All the `apiKey` fields and the Secret `stringData` values come from your `trident-admin.json` file. You can extract the non-sensitive `apiKey` portion with:
> ```bash
> jq 'del(.private_key_id, .private_key)' trident-admin.json
> ```

> **Important:** For Flex service level, `location` must be the **zone** (e.g. `us-central1-a`), not the region. For Standard/Premium/Extreme, use the **region** (e.g. `us-central1`).

Apply both resources:

```bash
oc apply -f gcv-backend-flex.yaml
```

Wait for the backend to become ready:

```bash
oc get tridentbackendconfig gcv-backend-flex -n trident -o jsonpath='{.status.lastOperationStatus}'
```

Expected output: `Success`. If it shows `Failed`, check the backend details:

```bash
oc describe tridentbackendconfig gcv-backend-flex -n trident
```

For more examples (virtual pools, SMB, topology), see [GCNV backend configuration examples](https://docs.netapp.com/us-en/trident/trident-use/gcnv-examples.html#example-configurations).

#### Adding More Pools Later

When you need more volume capacity, [create additional storage pools](https://docs.cloud.google.com/netapp/volumes/docs/configure-and-use/storage-pools/create-storage-pool) in GCP and add their names to the `storagePools` list in the TridentBackendConfig:

```bash
oc edit tridentbackendconfig gcv-backend-flex -n trident
```

Add the new pool name to `spec.storagePools` and save. Trident picks up the change automatically.

---

### Step 3: Create the StorageClass

Create a StorageClass that uses the Trident backend. This is what users and CDI will reference when creating PVCs.

Create a file named `gcnv-flex-storageclass.yaml` with the following content:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gcnv-flex
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
    storageclass.kubevirt.io/is-default-virt-class: "true"
provisioner: csi.trident.netapp.io
parameters:
  backendType: "google-cloud-netapp-volumes"
  selector: "performance=flex"
allowVolumeExpansion: true
```

Apply:

```bash
oc apply -f gcnv-flex-storageclass.yaml
```

If the cluster already has a default StorageClass (e.g. `standard-csi`), remove the default annotation:

```bash
oc annotate storageclass standard-csi storageclass.kubernetes.io/is-default-class-
```

Verify only `gcnv-flex` is the default:

```bash
oc get storageclass
```

You should see `gcnv-flex (default)` in the output.

---

### Step 4: Create the VolumeSnapshotClass

A `VolumeSnapshotClass` is required for VM snapshot and restore operations.

Create a file named `gcnv-csi-snapclass.yaml` with the following content:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: gcnv-csi-snapclass
driver: csi.trident.netapp.io
deletionPolicy: Delete
```

Apply:

```bash
oc apply -f gcnv-csi-snapclass.yaml
```

---

### Step 5: Set the Filesystem Overhead

NFS volumes consume part of their provisioned capacity for filesystem metadata. The recommended filesystem overhead for GCNV is **10%**. Without this, CDI may provision PVCs that are too small for operations that need temporary space (e.g. gzipped exports). See [ONTAP Volume Space Reporting](gcnv-known-errors-and-limits.md#4-ontap-volume-space-reporting---no-space-left-on-device-near-full-capacity) for details.

Patch the HyperConverged CR to set the overhead for the `gcnv-flex` StorageClass:

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

## Verification

Run through these checks to confirm everything is working:

**1. Trident pods are running:**

```bash
oc get pods -n trident
```

All pods should be `Running`.

**2. Backend is healthy:**

```bash
oc get tridentbackendconfig -n trident
```

Status should show `Bound` and `Success`.

**3. StorageClass exists and is default:**

```bash
oc get storageclass gcnv-flex
```

**4. VolumeSnapshotClass exists:**

```bash
oc get volumesnapshotclass gcnv-csi-snapclass
```

**5. Create a test PVC:**

Create a file named `gcnv-test-pvc.yaml` with the following content:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gcnv-test-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: gcnv-flex
```

Apply:

```bash
oc apply -f gcnv-test-pvc.yaml
```

Wait for it to bind:

```bash
oc get pvc gcnv-test-pvc -n default
```

Status should be `Bound`. Clean up:

```bash
oc delete pvc gcnv-test-pvc -n default
```

---

## Quick Reference

| Resource | Name | Purpose |
|---|---|---|
| Namespace | `trident` | Trident operator and controller pods |
| Secret | `trident-admin` | GCP service account credentials |
| TridentBackendConfig | `gcv-backend-flex` | Connects Trident to GCNV storage pools |
| StorageClass | `gcnv-flex` | Default StorageClass for VM disks |
| VolumeSnapshotClass | `gcnv-csi-snapclass` | Enables VM snapshot/restore |
| HyperConverged patch | `filesystemOverhead: gcnv-flex: "0.10"` | 10% NFS filesystem overhead for CDI |

## Key Limitations

For known errors, quotas, and platform-level constraints, see [gcnv-known-errors-and-limits.md](gcnv-known-errors-and-limits.md).

| Limit | Value | Notes |
|---|---|---|
| Minimum volume size | 1 GiB | Use `minimumSupportedPvcSize` in StorageProfile |
| Volumes per Flex pool | 50 | Use multiple pools in TridentBackendConfig |
| Volume mode | Filesystem only | No block mode support (NFS backend) |
| Access modes | RWO, RWX, ROX | All in Filesystem mode |
| fstrim / discard | Not supported | NFS does not support TRIM |

## External References

- [NetApp Trident documentation](https://docs.netapp.com/us-en/trident/index.html)
- [Deploy Trident with the operator](https://docs.netapp.com/us-en/trident/trident-get-started/kubernetes-deploy-operator.html)
- [GCNV backend preparation](https://docs.netapp.com/us-en/trident/trident-use/gcnv-prep.html)
- [GCNV backend configuration options and examples](https://docs.netapp.com/us-en/trident/trident-use/gcnv-examples.html)
- [GCNV quotas and limits](https://docs.cloud.google.com/netapp/volumes/docs/quotas)
- [Create a GCNV storage pool](https://docs.cloud.google.com/netapp/volumes/docs/configure-and-use/storage-pools/create-storage-pool)
