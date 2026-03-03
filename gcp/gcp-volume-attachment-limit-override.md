# GCP PD CSI Volume Attachment Limit Override (15 to 127)

By default, the GCP PD CSI driver reports a volume attachment limit of **15 per node**. This document describes how to override that limit to **127** volumes per node.

## Prerequisites

There are 2 images:

1. `quay.io/noamasu/gcp-pd-csi-driver-operator:debug-build`
2. `quay.io/noamasu/gcp-compute-persistent-disk-csi-driver:debug-fix`

Changing the `gcp-pd-csi-driver-operator` image is hard (considering clusterversion permissions and using `oc adm upgrade`).

Therefore need to manually apply both the RBAC and the missing node name passing:

1. Stop operator:

```bash
oc patch clustercsidriver pd.csi.storage.gke.io --type=merge -p '{"spec":{"managementState":"Unmanaged"}}'
```

2. Add the RBAC:

```bash
oc apply -f - <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gcp-pd-node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gcp-pd-node-reader-binding
subjects:
- kind: ServiceAccount
  name: gcp-pd-csi-driver-node-sa
  namespace: openshift-cluster-csi-drivers
roleRef:
  kind: ClusterRole
  name: gcp-pd-node-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

3. Patch the node DaemonSet to use fixed csi-driver image:

```bash
oc set image ds/gcp-pd-csi-driver-node -n openshift-cluster-csi-drivers csi-driver=quay.io/noamasu/gcp-compute-persistent-disk-csi-driver:debug-fix
```

4. Ensure node name is passed:

```bash
oc patch ds/gcp-pd-csi-driver-node -n openshift-cluster-csi-drivers --type='json' -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--node-name=$(KUBE_NODE_NAME)"},{"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{"name":"KUBE_NODE_NAME","valueFrom":{"fieldRef":{"fieldPath":"spec.nodeName"}}}}]'
```

5. Label worker nodes with override:

```bash
oc label nodes -l node-role.kubernetes.io/worker node-restriction.kubernetes.io/gke-volume-attach-limit-override=127 --overwrite
```

6. Restart CSI node pods (or wait for rollout):

```bash
oc delete pod -n openshift-cluster-csi-drivers -l app=gcp-pd-csi-driver-node
```

## Verify

1. Verify fixed driver image is on the node DaemonSet:

```bash
oc get ds gcp-pd-csi-driver-node -n openshift-cluster-csi-drivers -o jsonpath='{.spec.template.spec.containers[?(@.name=="csi-driver")].image}{"\n"}'
```

2. Verify `--node-name` arg is present:

```bash
oc get ds gcp-pd-csi-driver-node -n openshift-cluster-csi-drivers -o jsonpath='{.spec.template.spec.containers[?(@.name=="csi-driver")].args}{"\n"}'
```

3. Verify `KUBE_NODE_NAME` env var is present:

```bash
oc get ds gcp-pd-csi-driver-node -n openshift-cluster-csi-drivers -o jsonpath='{.spec.template.spec.containers[?(@.name=="csi-driver")].env}{"\n"}'
```

4. Verify node override label exists:

```bash
oc get nodes -l node-role.kubernetes.io/worker -o custom-columns=NAME:.metadata.name,OVERRIDE:.metadata.labels.node-restriction\.kubernetes\.io/gke-volume-attach-limit-override
```

5. Verify `CSINode` allocatable count is now `127`:

```bash
oc get csinode <worker-node-name> -o jsonpath='{.spec.drivers[?(@.name=="pd.csi.storage.gke.io")].allocatable.count}{"\n"}'
```
