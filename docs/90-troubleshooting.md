---
layout: default
title: Fejlfinding og FAQ
nav_order: 90
---

# Fejlfinding og FAQ

Denne sektion samler de mest almindelige fejlscenarier ved drift af en single-node Talos Kubernetes-klynge, samt løsninger baseret på projektets scripts, konfigurationer og kendte begrænsninger.

Indholdet bygger direkte på praktiske erfaringer samt `history.txt` og `history2.txt` fra projektmappen.

---

# 1. Generelle fejlfindingstrin

Start altid med følgende kontroller:

```bash
talosctl health
kubectl get nodes
kubectl get pods -A
```

### Hvis Talos ikke svarer:
- Serveren kører muligvis stadig på ISO (maintenance mode)
- Forkert IP-konfiguration i `network.yaml`
- Firewall/NAT mellem WSL2 og serveren blokerer trafik

---

# 2. Talos relaterede fejl

## 2.1 “connection refused” eller timeout ved talosctl-kommandoer

Mulige årsager:

- Endpoint er ikke sat:
  ```bash
  talosctl config endpoint 192.168.50.10
  talosctl config node 192.168.50.10
  ```
- Serveren kører ikke Talos endnu (stadig i ISO)
- Statisk IP er ikke blevet anvendt korrekt via `apply.sh`

Test:

```bash
ping 192.168.50.10
talosctl version
```

---

## 2.2 Talos health viser fejl

Kør:

```bash
talosctl health --nodes 192.168.50.10
```

Typiske problemer:

### Kubernetes API ned
- Cilium er ikke startet korrekt  
- Certifikatfejl ved bootstrap  
- ukorrekt controlplane.yaml

### etcd fejl
- Bootstrap ikke udført:
  ```bash
  talosctl bootstrap --nodes 192.168.50.10
  ```
- Nodens tid/dato er forkert (tjek NTP)

---

# 3. Kubernetes API fejler (“context deadline exceeded”)

Kør:

```bash
kubectl get pods -A
```

Hvis API ikke svarer:

Mulige årsager:
- Cilium er ikke installeret eller fejler
- kube-apiserver pod restart loop
- Ingen kubeconfig genereret endnu (`kubeconfig.sh`)

Tjek logs:

```bash
talosctl logs kube-apiserver
talosctl logs kubelet
```

---

# 4. Fejl i Cilium installation

Hvis pods i kube-system viser CrashLoopBackOff eller Pending:

## 4.1 Tjek status

```bash
cilium status
kubectl -n kube-system get pods
```

Typiske fejl:
- Forkert kubeServiceHost / kubeServicePort i values.yaml
- Manglende rettigheder på Talos (Cilium kræver BPF support)
- Konflikt mellem Talos default CNI og Cilium

## 4.2 Løsning
- Geninstaller Cilium:
  ```bash
  helm upgrade --install cilium ./charts/cilium     --namespace kube-system     --values ./charts/cilium/values.yaml
  ```

- Kør connectivity test:
  ```bash
  cilium connectivity test
  ```

---

# 5. Fejl ved Rook Ceph installation

Da dette er en **single-node klynge**, kan visse Ceph-komponenter fejle under opstart.

### Tjek cephcluster status:

```bash
kubectl -n rook-ceph get cephcluster
kubectl -n rook-ceph get pods
```

Typiske problemer:

## 5.1 OSD starter ikke
- Ingen tilgang til rå blokdevices
- DS-mapping fejl
- Forkerte storage paths i manifest.yaml

## 5.2 MON eller MGR fejler
- Ressourcebegrænsninger (Ceph kræver minimum 4GB RAM)
- PGS krav ikke opfyldt i single-node drift (forvent advarsler)

## 5.3 StorageClass eksisterer ikke
Kør:

```bash
kubectl get storageclass
```

Hvis `rook-ceph-block` ikke er default, sæt den:

```bash
kubectl patch storageclass rook-ceph-block   -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

# 6. Problemer med kubeconfig

## 6.1 “The connection to the server was refused”

Løsning:

```bash
talosctl kubeconfig --nodes 192.168.50.10 --force
```

Kontrollér:

```bash
kubectl config view
kubectl get nodes
```

## 6.2 WSL2 kan ikke nå serveren

Mulige årsager:
- Firewall blokerer WSL2-subnet (typisk 172.x.x.x)
- WSL2’s IP har ændret sig efter reboot
- Ingen routing mellem Windows-PC og serverens VLAN

Test:

```bash
ip route
ping 192.168.50.10
```

---

# 7. Pod kan ikke schedule på single-node cluster

Denne fejl forekommer hvis:

```
allowSchedulingOnControlPlanes: false
```

Løsning:

Anvend filen:

```
allow-scheduling-on-control-planes.yaml
```

Eller opdater cluster med:

```yaml
cluster:
  allowSchedulingOnControlPlanes: true
```

---

# 8. Fejl i Helm-installationer

## 8.1 Chart versionskonflikt

Løsning:

```bash
helm repo update
```

## 8.2 Values.yaml ikke indlæst

Tjek:

```bash
helm get values <release>
```

Geninstaller:

```bash
helm upgrade --install <release> <path> --values <path>/values.yaml
```

---

# 9. Talos logindsamling

Talos anvender ikke SSH — brug:

```bash
talosctl logs <service>
talosctl dmesg
talosctl service list
```

Eksempel:

```bash
talosctl logs containerd
talosctl logs kubelet
```

---

# 10. Nød-reboot

Hvis systemet hænger fuldstændig:

```bash
talosctl reboot --nodes 192.168.50.10
```

---

# 11. Kendte begrænsninger

- **Single-node Ceph er ikke HA**  
- **Control-plane og worker deler ressourcer**  
- **Cilium kræver BPF support**  
- **Talos kræver korrekt IP-konfiguration ved første apply**

---

# 12. Hvis alt andet fejler

Genopbyg Talos-konfigurationen:

```bash
./Bash/build.sh
./Bash/apply-insecure.sh
```

Og bootstrap igen:

```bash
./Bash/bootstrap.sh
./Bash/kubeconfig.sh
```

---

# 13. Links til videre læsning

- Talos fejlfinding: https://docs.siderolabs.com/talos/v1.8/reference/cli
- Cilium debugging: https://docs.cilium.io/en/stable/
- Rook Ceph troubleshoot: https://rook.io/docs/rook/latest/Troubleshooting/kubectl-plugin/
