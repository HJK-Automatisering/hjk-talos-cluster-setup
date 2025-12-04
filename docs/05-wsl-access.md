---
layout: default
title: Adgang til klyngen fra WSL2
nav_order: 8
---

# Adgang til klyngen fra WSL2

Denne guide beskriver, hvordan man får fuld adgang til Talos og Kubernetes-klyngen fra en **Windows 11 maskine via Ubuntu WSL2**.

WSL2 bruges til:
- `talosctl` styring af serveren  
- `kubectl` administration af Kubernetes  
- Helm-deployment af Cilium og Rook Ceph 

Forbindelsen sker direkte til serverens statiske IP-adresse (fx `192.168.50.10`).

---

# 1. Forudsætninger

Inden du fortsætter, skal følgende være installeret i WSL2:

```bash
talosctl version
kubectl version --client
helm version
cilium version
```

Også vigtigt:

- Du skal kunne **ping'e serverens IP**  
- Firewalls skal tillade trafik fra din WSL2-adresse  

Test:

```bash
ping 192.168.50.10
```

---

# 2. Talosctl konfiguration

Når Talos er installeret på serveren, skal WSL2 sættes op til at kommunikere med noden.

## 2.1 Sæt Talos endpoint

```bash
talosctl config endpoint 192.168.50.10
```

## 2.2 Sæt node

```bash
talosctl config node 192.168.50.10
```

## 2.3 Test forbindelsen

```bash
talosctl version
talosctl health
```

Hvis dette fejler:

- tjek routing  
- tjek firewall  
- tjek at serveren kører Talos (ikke ISO i maintenance mode)

---

# 3. Hent kubeconfig (Talos → WSL2)

Efter bootstrap:

```bash
talosctl kubeconfig --nodes 192.168.50.10 --force
```

Dette genererer:

```
~/.kube/config
```

Verificér:

```bash
kubectl get nodes
kubectl get pods -A
```

Hvis der returneres data → Kubernetes-adgang virker.

---

# 4. WSL2 netværksbemærkninger

WSL2 fungerer som en **virtuel maskine**, og dens IP ændrer sig ved hvert reboot.

Tjek din WSL2 IP:

```bash
ip addr show eth0
```

Eksempel:

```
inet 172.25.89.14/20
```

### Hvis WSL2 ikke kan nå 192.168.50.10:

1. Sørg for at Windows-maskinen fysisk kan nå serverens subnet  
2. Tjek at router/firewall tillader WSL2-trafik  
3. WSL2 NAT kan kræve statiske ruter i ekstremt låste netværk

---

# 5. Import af kubeconfig til Windows Kubernetes tooling (valgfrit)

Hvis du også ønsker Windows-side tools:

```powershell
mkdir ~/.kube
wsl cp ~/.kube/config ~/.kube/config
```

---

# 6. Brug af Helm fra WSL2

Når kubectl virker, virker Helm automatisk:

```bash
helm ls -A
```

Dette bruges til:

- Installation af Cilium  
- Installation af Rook Ceph  

---

# 7. Fejlfinding

### WSL2 → Talos fejler
```bash
talosctl version
```
Fejl: *connection refused*  
→ Server kører ikke Talos (måske ISO boot).  
→ Firewall blokkerer trafik.

### kubeconfig virker ikke
```bash
kubectl get nodes
```
Fejl: *context deadline exceeded*  
→ kube-apiserver svarer ikke.  
→ Cilium kan være ned (tjek `kubectl -n kube-system get pods`).

### Talos health check
```bash
talosctl health
```
Returnerer detaljer om node-status.

---

# 8. Klar til næste trin

Når WSL2-adgang virker, kan du fortsætte med drift og deployment:

- Installér Cilium  
- Installér Rook Ceph  

Se næste dokumentation for arbejdsprocessen.
