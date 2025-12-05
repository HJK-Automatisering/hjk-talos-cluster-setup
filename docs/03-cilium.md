---
layout: default
title: Installation af Cilium
nav_order: 4
---

# Installation af Cilium

Cilium fungerer som CNI (Container Network Interface) i klyngen og erstatter den indbyggede Talos CNI.  
Dette kapitel beskriver installationen af **Cilium** ved hjælp af de Helm-kommandoer og scripts, der findes i projektmappen `Talos/Bash/charts/cilium`.

Installationen består af tre hovedtrin:

1. Opret og tilpas `values.yaml` til Cilium  
2. Installér Cilium via Helm (eller via `upgrade.sh`)  
3. Valider netværk og konnektivitet  

> **Bemærk:** Talos’ indbyggede CNI og kube-proxy er slået fra via Talos-patchen `Talos/Bash/cilium.yaml`, som anvendes tidligere i Talos-konfigurationen. Det er en forudsætning for, at Cilium kan overtage netværket.

---

# 1. Opret Cilium Helm values (values.yaml)

Hvis filen ikke allerede findes i projektet, kan den genereres fra Ciliums egen chart-definition.

Gå til Cilium chart-mappen i dit projekt:

```bash
cd Talos/Bash/charts/cilium
```

Hent standardværdierne fra den ønskede Cilium-version og gem dem som `values.yaml`:

```bash
helm show values cilium/cilium --version 1.18.4 > values.yaml
```

> **Bemærk:** Versionen (`1.18.4`) er et eksempel. Brug den version, I ønsker at standardisere på.  
> Det anbefales at fastlåse en version og committe `values.yaml` til repository’et, så installationen bliver reproducerbar.

Herefter kan `values.yaml` tilpasses efter jeres miljø (netværk, Hubble, IPAM osv.).

Eksempeludsnit (for illustration):

```yaml
hubble:
  ui:
    enabled: true

k8sServiceHost: 127.0.0.1
k8sServicePort: 7445
```

Når filen er oprettet og tilpasset, vil den blive brugt i de næste trin.

---

# 2. Installation af Cilium via Helm

## 2.1 Tilføj Cilium repo

Dette udføres typisk én gang pr. miljø:

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
```

## 2.2 Installér Cilium med jeres `values.yaml`

Fra mappen `Talos/Bash/charts/cilium`:

```bash
cd Talos/Bash/charts/cilium

helm upgrade cilium cilium/cilium --install   --version 1.18.4 --namespace kube-system -f values.yaml
```

> Vi bruger `upgrade --install`, så kommandoen både kan installere og opdatere samme release.

Projektet indeholder også et script til opgradering og gentagen installation:

```bash
bash upgrade.sh
```

Indholdet af `upgrade.sh` er:

```bash
helm upgrade cilium cilium/cilium --install --version 1.18.4 --namespace kube-system -f values.yaml
```

Du kan derfor nøjes med at vedligeholde version og konfiguration i `values.yaml` og evt. i scriptet, og blot køre:

```bash
bash upgrade.sh
```

ved fremtidige opgraderinger.

---

# 3. Validering af installation

## 3.1 Tjek Cilium pods

```bash
kubectl -n kube-system get pods -l k8s-app=cilium
kubectl -n kube-system get pods -l k8s-app=cilium-operator
```

Alle Cilium-relaterede pods skal blive **Ready**.

## 3.2 Tjek Cilium status

```bash
cilium status
```

Du bør se at:

- CNI er initialiseret korrekt  
- Datapath er **Healthy**  
- Eventuelle Hubble-komponenter er oppe, hvis de er aktiveret  

## 3.3 Test netværkskonnektivitet

```bash
cilium connectivity test
```

Dette tester bl.a.:

- DNS-opslag  
- Pod-til-pod trafik  
- Pod-til-service trafik  
- Node routing  

---

# 4. Aktivering af Hubble UI (valgfrit)

Hvis Hubble UI er aktiveret i `values.yaml`, kan den eksponeres lokalt:

```bash
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
```

Åbn derefter en browser på din egen maskine:

```
http://localhost:12000
```

Her kan du se live netværkstrafik, flows og fejlsøgning.
