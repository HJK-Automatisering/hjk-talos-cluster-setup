---
layout: default
title: Installation af Cilium
nav_order: 4
---

# Installation af Cilium

Cilium fungerer som CNI (Container Network Interface) i klyngen og erstatter den indbyggede Talos CNI.  
Dette kapitel beskriver installationen af **Cilium** ved hjælp af de Helm charts og scripts, der findes i projektmappen `Talos/Bash/charts/cilium`.

Installationen består af tre trin:

1. Forberedelse af værdier (values.yaml)  
2. Installation via Helm  
3. Validering af netværk og konnektivitet  

---

# 1. Cilium Helm values (values.yaml)

Projektet inkluderer en komplet Cilium konfiguration i:

```
Talos/Bash/charts/cilium/values.yaml
```

Her sættes bl.a.:

- IPAM (kube-router)
- Hubble aktivering
- BPF indstillinger
- Operator settings
- Native-routing mode
- ENCAP/DirectRouting

Eksempeludsnit:

```yaml
hubble:
  ui:
    enabled: true

k8sServiceHost: 127.0.0.1
k8sServicePort: 7445
```

Tilpas kun values-filen, hvis jeres netværksmiljø kræver det.

---

# 2. Installation af Cilium via Helm

Fra jeres WSL2-arbejdsstation:

## Tilføj Cilium repo

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
```

## Installér Cilium chartet fra projektmappen

```bash
helm upgrade --install cilium ./charts/cilium   --namespace kube-system   --values ./charts/cilium/values.yaml
```

> Bemærk: Vi bruger `upgrade --install` så kommandoen både kan installere og opdatere.

Talos/Bash indeholder også et script til opgradering:

```bash
./Bash/charts/cilium/upgrade.sh
```

---

# 3. Validering af installation

## Tjek pods

```bash
kubectl -n kube-system get pods -l k8s-app=cilium
kubectl -n kube-system get pods -l k8s-app=cilium-operator
```

Alle pods skal blive **Ready**.

## Tjek Cilium status

```bash
cilium status
```

Du bør se:

- CNI OK  
- Hubble Relay OK  
- Datapath Healthy  

## Test netværkskonnektivitet

```bash
cilium connectivity test
```

Dette tester bl.a.:

- DNS
- Pod-to-pod trafik  
- Pod-to-service trafik  
- Node routing  

---

# 4. Aktivering af Hubble UI (valgfrit)

Hvis Hubble er slået til i values.yaml, kan UI’en eksponeres lokalt:

```bash
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
```

Åbn i browser:

```
http://localhost:12000
```
