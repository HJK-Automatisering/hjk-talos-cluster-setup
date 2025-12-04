---
layout: default
title: Installation af Rook Ceph (single-node)
nav_order: 5
---

# Installation af Rook Ceph (single-node)

Rook Ceph anvendes som distributed storage backend for Kubernetes.  
**I en single-node klynge kan Ceph dog kun bruges til test, udvikling og interne PoC'er**, da Ceph normalt kræver flere noder for replikering og HA.

Dette dokument beskriver installationen baseret på de filer, der ligger i:

```
Talos/Bash/charts/rook-ceph/
Talos/Bash/charts/rook-ceph-cluster/
```

---

# 1. Forudsætninger

Før installation skal følgende være på plads:

- Talos cluster er bootstrap'et og kører stabilt  
- Cilium er installeret og operativ  
- Node har rå blok-enheder tilgængelige til Ceph OSD'er  
- `kubectl` og `helm` virker fra WSL2  

Test:

```bash
kubectl get nodes
kubectl -n kube-system get pods
```

---

# 2. Installér Rook Ceph Operator

Projektet indeholder Helm-opsætning for Rook operatoren.

Værdierne findes her:

```
Talos/Bash/charts/rook-ceph/values.yaml
```

Installer operatoren:

```bash
helm upgrade --install rook-ceph ./charts/rook-ceph   --namespace rook-ceph   --create-namespace   --values ./charts/rook-ceph/values.yaml
```

Tjek pods:

```bash
kubectl -n rook-ceph get pods
```

Der skal komme pods som:

- rook-ceph-operator  
- rook-discover  

---

# 3. Installér Ceph Cluster (single-node manifest)

Cluster-definitionen findes i:

```
Talos/Bash/charts/rook-ceph-cluster/manifest.yaml
```

Denne manifest er allerede forberedt til **single-node drift**.

Installér:

```bash
kubectl apply -f ./charts/rook-ceph-cluster/manifest.yaml
```

Dette opretter:

- CephCluster
- CephBlockPool
- StorageClass
- Manager + Mon + OSD

Vent et par minutter og tjek status:

```bash
kubectl -n rook-ceph get pods
kubectl -n rook-ceph get cephcluster
```

---

# 4. StorageClass

Cluster-manifestet opretter typisk en StorageClass:

```bash
kubectl get storageclass
```

Du bør se:

```
rook-ceph-block (default)
```

Hvis ikke, sæt den som default:

```bash
kubectl patch storageclass rook-ceph-block   -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

# 5. Sikkerhedsmæssige undtagelser (rook-ceph-security-exemption.yaml)

I projektet findes filen:

```
Talos/Bash/rook-ceph-security-exemption.yaml
```

Den bruges til at tillade nødvendige privilegier i Talos-miljøet.

Anvend den:

```bash
kubectl apply -f ./Bash/rook-ceph-security-exemption.yaml
```

---

# 6. Opgraderinger (upgrade.sh)

Projektet indeholder et opgraderingsscript:

```
Talos/Bash/charts/rook-ceph/upgrade.sh
```

Opgradering foretages med:

```bash
./charts/rook-ceph/upgrade.sh
```

---

# 7. Verificér Ceph cluster

### Tjek CephCluster status:

```bash
kubectl -n rook-ceph get cephcluster
```

Forventet:

```
HEALTH_OK
```

### Tjek pods:

```bash
kubectl -n rook-ceph get pods
```

Der skal være:

- rook-ceph-mgr
- rook-ceph-mon
- rook-ceph-osd
- rook-ceph-crashcollector
- rook-ceph-mds (valgfrit)
- rook-ceph-tools (hvis installeret)

---

# 8. Test et PVC claim

Lav en test PersistentVolumeClaim:

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

Gem som `pvc-test.yaml` og kør:

```bash
kubectl apply -f pvc-test.yaml
kubectl get pvc
```

---

# 9. Kendte begrænsninger for single-node Ceph

- **Ingen replikering**  
  Alle Ceph-pools vil køre med `size: 1`

- **Ingen høj tilgængelighed**

- **Diskfejl = tab af data**  
  Der er ingen redundans.

> Anvend kun Ceph i single-node miljøet til test, udvikling og OS2AI PoC'er.
