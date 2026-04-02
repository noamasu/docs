# OpenShift Virtualization on Google Cloud with NetApp Volumes (GCNV)

OpenShift Virtualization with Google Cloud NetApp Volumes (GCNV) storage is now generally available. GCNV provides NFS-based shared storage with ReadWriteMany (RWX) support in Filesystem mode, provisioned through the NetApp Trident CSI driver. For more information, see the following guides:

- [Storage configuration for OpenShift Virtualization with GCNV](gcnv-storage-configuration.md)
- [OpenShift Virtualization with GCNV: Known errors and limitations](gcnv-known-errors-and-limits.md)

**Important**

- Running OpenShift Virtualization with GCNV storage requires OpenShift Container Platform 4.21+ and OpenShift Virtualization 4.21.2+, or later versions.
- Only the **Flex File** service level is supported in this release. When creating storage pools, select the **File** storage type. Flex Unified (which adds iSCSI/block support) is not covered.
- Flex File volumes are NFS-only and support `volumeMode: Filesystem` exclusively. `volumeMode: Block` is not available with Flex File.
- GCNV Flex pools are limited to **50 volumes per pool**. To support larger deployments, create multiple storage pools and list them all in the `TridentBackendConfig`. For more information, see [GCNV storage pool limits](https://docs.cloud.google.com/netapp/volumes/docs/quotas#storage_pool_limits).
- Flex File pools can be **zonal** or **regional**. Regional pools replicate volumes across zones but only support default performance (not custom). For more information on service levels and performance, see [GCNV service levels](https://docs.cloud.google.com/netapp/volumes/docs/discover/service-levels).
