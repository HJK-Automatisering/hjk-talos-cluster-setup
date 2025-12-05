---
layout: default
title: Installation af Rook Ceph (single-node)
nav_order: 5
---

# Installation af Rook Ceph (single-node)

Rook Ceph bruges som storage-backend til Kubernetes.  
I en **single-node** konfiguration fungerer Ceph **kun som test-/udviklingsmiljø**, da der ikke er redundans eller høj tilgængelighed.

Denne dokumentation beskriver en **ren, professionel og reproducerbar installation**, baseret på jeres projektstruktur:

```
Talos/Bash/charts/rook-ceph/
Talos/Bash/charts/rook-ceph-cluster/
```

---

# 1. Forudsætninger

Før installationen:

- Talos-klyngen er installeret og bootstrap’et
- Cilium kører stabilt
- Node har én eller flere rå diske til Ceph OSD’er (ikke formaterede)
- `kubectl`, `helm` og `talosctl` fungerer i WSL2

Verificér:

```bash
kubectl get nodes
kubectl -n kube-system get pods
```

---

# 2. Opsætning af Helm repository

```bash
helm repo add rook-release https://charts.rook.io/release
helm repo update
```

Tjek tilgængelige charts:

```bash
helm search repo rook
```

---

# 3. Generér operatorens values.yaml

Navigér til operatorens chart-mappe:

```bash
cd Talos/Bash/charts/rook-ceph
```

Generér Helm-chart værdier:

```bash
helm show values rook-release/rook-ceph --version v1.18.8 > values.yaml
```

> Fastlås gerne versionen i projektet og commit filen.

---

# 4. Installér Rook Ceph operator

```bash
helm upgrade --install rook-ceph ./charts/rook-ceph --namespace rook-ceph --create-namespace --values ./charts/rook-ceph/values.yaml
```

Tjek pods:

```bash
kubectl -n rook-ceph get pods
```

---

# 5. Generér clusterens values.yaml

Navigér til cluster chart-mappen:

```bash
cd Talos/Bash/charts/rook-ceph-cluster
```

Generér standardværdier:

```bash
helm show values rook-release/rook-ceph-cluster --version v1.18.8 > values.yaml
```

---

# 6. Generér manifest.yaml via template.sh

``template.sh`` indholder:
```bash
helm repo add rook-release https://charts.rook.io/release

helm template rook-ceph-cluster rook-release/rook-ceph-cluster \
  --namespace rook-ceph \
  --version v1.18.8 \
  -f values.yaml >> manifest.yaml
```

```bash
bash template.sh
```

Dette genererer:

```
manifest.yaml
```

---

# 7. Installér Ceph clusteret

```bash
kubectl apply -f ./charts/rook-ceph-cluster/manifest.yaml
```

Tjek clusterets tilstand:

```bash
kubectl -n rook-ceph get cephcluster
kubectl -n rook-ceph get pods
```

---

# 8. StorageClass

```bash
kubectl get storageclass
```

Hvis `rook-ceph-block` ikke er default:

```bash
kubectl patch storageclass rook-ceph-block   -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

# 9. Giv Talos de nødvendige undtagelser

```yaml
# rook-ceph-security-exemption.yaml
cluster:
  apiServer:
    admissionControl:
    - name: PodSecurity # Name is the name of the admission controller.
      # Configuration is an embedded configuration object to be used as the plugin's
      configuration:
        apiVersion: pod-security.admission.config.k8s.io/v1alpha1
        exemptions:
          namespaces:
          - rook-ceph
```

```bash
kubectl apply -f ./Bash/rook-ceph-security-exemption.yaml
```

---

# 10. Opgradering af Rook (upgrade.sh)

``rook-ceph/upgrade.sh`` indholder:

```bash
helm repo add rook-release https://charts.rook.io/release

helm upgrade rook-ceph rook-release/rook-ceph \
  --install \
  --create-namespace \
  --namespace rook-ceph \
  --version v1.18.8 \
  -f values.yaml
```

``rook-ceph-cluster/upgrade.sh`` indholder:

```bash
helm repo add rook-release https://charts.rook.io/release

helm upgrade rook-ceph-cluster rook-release/rook-ceph-cluster \
  --install \
  --create-namespace \
  --namespace rook-ceph \
  --version v1.18.8 \
  -f values.yaml
```

```bash
bash ./charts/rook-ceph/upgrade.sh
bash ./charts/rook-ceph-cluster/upgrade.sh
```

---

# 11. Test PVC-provisionering

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

```bash
kubectl apply -f pvc-test.yaml
kubectl get pvc
```

---

# 12. Kendte begrænsninger for single-node Ceph

| Begrænsning | Forklaring |
|------------|------------|
| Ingen replikering | Pools kører med `size: 1` |
| Ingen HA | Mon/Mgr kører kun på én node |
| Diskfejl = datatab | Ingen redundans |
| Ikke til produktion | Kun test/PoC |

