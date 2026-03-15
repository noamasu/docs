# Running Storage Checkups on GCP

This guide covers two storage checkups available in the OpenShift Console for validating storage on a GCP cluster with Hyperdisk Balanced.

Both checkups are found at: **Virtualization > Checkups**

---

## 1. Storage Checkup

A quick checkup that validates storage is working correctly for VirtualMachines using the kiagnose engine.

### Step 1: Open the Storage Checkup

In the OpenShift Console, navigate to:

**Virtualization > Checkups**

Click the **Storage** tab.

### Step 2: Install Permissions

Make sure you are on the correct project (namespace) where you want to run the test.

Click **Install permissions** and wait for it to complete.

### Step 3: Run the Checkup

Click **Run checkup** (top left). Configure the following:

| Field | Value |
|-------|-------|
| **Name** | Leave the auto-generated name or enter your own |
| **Timeout (minutes)** | `10` (default, recommended) |

Click **Run**.

---

## 2. Storage Self-Validation Checkup

A comprehensive test suite that exercises storage features in depth.

> **Warning:** This checkup may put the cluster under stress and can take up to 3 hours to complete. Do not run it in production environments as it may impact cluster performance.

### Step 1: Open the Self-Validation Checkup

In the OpenShift Console, navigate to:

**Virtualization > Checkups**

Click the **Self validation** tab.

### Step 2: Install Permissions

Make sure you are on the correct project (namespace) where you want to run the test.

Click **Install permissions** and wait for it to complete.

### Step 3: Configure and Run the Checkup

Click **Run checkup** (top left). Configure the following fields:

| Field | Value |
|-------|-------|
| **Name** | Leave the auto-generated name or enter your own |
| **Test suites** | Select **Storage** only (uncheck all others) |

Expand the **Advanced settings** section and configure:

| Field | Value | Notes |
|-------|-------|-------|
| **StorageClass** | Your Hyperdisk StorageClass (e.g. `sp-balanced-storage`) | |
| **Test skips** | Leave empty | Pipe-separated list of test IDs to skip, if needed |
| **PVC size** | **4 GiB** | Change from the default 2 GiB to meet the [Hyperdisk Balanced minimum disk size](gcp-known-errors-and-limits.md#1-hyperdisk-balanced-minimum-disk-size-4-gb) |
| **Dry run** | **Off** | Must be off to actually run the tests |

Under **Storage capabilities**, select all options **except** `Storage RWX FileSystem`:

- [x] Storage Class RHEL
- [x] Storage Class Windows
- [x] Storage RWX Block
- [ ] Storage RWX FileSystem
- [x] Storage RWO FileSystem
- [x] Storage RWO Block
- [x] Storage Class CSI
- [x] Storage Snapshot
- [x] Online Resize
- [x] WFFC

`Storage RWX FileSystem` is excluded because the GCP PD CSI driver does not support RWX in Filesystem mode (see [No Native RWX Filesystem Support](gcp-known-errors-and-limits.md#6-no-native-rwx-filesystem-support)).

### Step 4: Run

Click **Run** and wait for the checkup to complete (up to 3 hours).
