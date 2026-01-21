---
layout: default
title: Rook Ceph Installation (single-node)
nav_order: 5
---

# Rook Ceph Installation (single-node)

Rook Ceph is used as the **storage backend for Kubernetes**.
In a **single-node setup** there is **no redundancy or high availability**.

```
k8s/charts/rook-ceph/
k8s/charts/rook-ceph-cluster/
scripts/k8s/
k8s/patches/
```

---

## 1. Prerequisites

Before proceeding:

- Talos cluster is installed and bootstrapped
- Cilium is installed and healthy
- The node has one or more **raw block devices** available for Ceph OSDs
- `kubectl`, `helm`, and `talosctl` are available (typically from WSL2)
- Kubernetes API is reachable

Verify:

```bash
kubectl get nodes
kubectl -n kube-system get pods
```

---

## 2. Helm repository setup

Add the Rook Helm repository (one-time operation):

```bash
helm repo add rook-release https://charts.rook.io/release
helm repo update
```

Verify available charts:

```bash
helm search repo rook
```

---

## 3. Generate Rook Ceph operator values.yaml

Navigate to the operator chart directory:

```bash
cd k8s/charts/rook-ceph
```

Generate default values for the pinned version:

```bash
helm show values rook-release/rook-ceph --version v1.18.8 > values.yaml
```

> Pinning the version and committing `values.yaml` ensures reproducibility.

Adjust `values.yaml` if needed (resource limits, monitoring, etc.).

---

## 4. Install Rook Ceph operator

Install or upgrade the operator using the provided script:

```bash
bash scripts/k8s/rook-ceph-upgrade.sh
```

This executes:

```bash
helm upgrade rook-ceph rook-release/rook-ceph   --install   --create-namespace   --namespace rook-ceph   --version v1.18.8   -f values.yaml
```

Verify:

```bash
kubectl -n rook-ceph get pods
```

---

## 5. Generate Ceph cluster values.yaml

Navigate to the cluster chart directory:

```bash
cd k8s/charts/rook-ceph-cluster
```

Generate default values:

```bash
helm show values rook-release/rook-ceph-cluster --version v1.18.8 > values.yaml
```

Edit `values.yaml` to ensure:

- Single-node configuration
- `replicated.size: 1`
- Correct device selection

### Single-node adjustments (important)

Because this is a single-node cluster, some upstream defaults must be changed:

- Use `failureDomain: osd` (not `host`) for pools/filesystems in a single-node environment
- Ensure pools are set to `replicated.size: 1`
- Enable CephFS (`cephFileSystems`) if you rely on it (and ensure its pools also use `failureDomain: osd`)

Example:

- `cephBlockPools[].spec.failureDomain: osd`
- `cephFileSystems[].spec.dataPools[].failureDomain: osd`

---

## 6. Render CephCluster manifest

The cluster chart is rendered to a static manifest for transparency and control.

Render using the script:

```bash
bash scripts/k8s/render-rook-ceph-cluster-template.sh
```

The script runs:

```bash
helm template rook-ceph-cluster rook-release/rook-ceph-cluster   --namespace rook-ceph   --version v1.18.8   -f values.yaml > manifest.yaml
```

Result:

```
k8s/charts/rook-ceph-cluster/manifest.yaml
```

---

## 7. Apply Ceph cluster manifest

Apply the rendered manifest:

```bash
kubectl apply -f k8s/charts/rook-ceph-cluster/manifest.yaml
```

Verify:

```bash
kubectl -n rook-ceph get cephcluster
kubectl -n rook-ceph get pods
```

The CephCluster should eventually reach `HEALTH_OK`.

---

## 8. StorageClass

Verify StorageClasses:

```bash
kubectl get storageclass
```

Expected:

```
rook-ceph-block
```

If not default:

```bash
kubectl patch storageclass rook-ceph-block   -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## 9. Talos PodSecurity exemption

Rook Ceph requires elevated privileges.
Talos enforces Pod Security by default.

Apply the exemption:

```bash
kubectl apply -f k8s/patches/rook-ceph-security-exemption.yaml
```

This allows required pods to run in the `rook-ceph` namespace.

---

## 10. Upgrades

Upgrade operator:

```bash
bash scripts/k8s/rook-ceph-upgrade.sh
```

Upgrade Ceph cluster:

```bash
bash scripts/k8s/rook-ceph-cluster-upgrade.sh
```

Always upgrade **operator first**, then **cluster**.

---

## 11. Test PVC provisioning

Create a test PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ceph-pvc-test
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-ceph-block
```

Apply:

```bash
kubectl apply -f pvc-test.yaml
kubectl get pvc
```

The PVC should reach `Bound`.

---

## 12. Known limitations (single-node)

| Limitation | Explanation |
|-----------|-------------|
| No replication | Pool size is `1` |
| No HA | Mon/Mgr/OSD all run on one node |
| Disk failure = data loss | No redundancy |

---

## Summary

Rook Ceph provides **functional persistent storage** for development and PoC workloads
on a single-node Talos cluster.

For production usage:
- Multiple nodes
- Multiple disks
- Replication ≥ 3
- Separate failure domains

are mandatory.
