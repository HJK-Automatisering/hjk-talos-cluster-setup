---

## docs/03-cilium.md

```markdown
---
layout: default
title: Installation af Cilium
nav_order: 4
---

# Installation af Cilium

Cilium bruges som CNI (Container Network Interface) for avanceret netværk og sikkerhed.

## Installation med Helm

1. Tilføj Cilium Helm repo:
```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
```

2. Installer Cilium:
```bash
helm install cilium cilium/cilium --version <version> --namespace kube-system -f cilium-values.yaml
```

3. Tilpas cilium-values.yaml efter jeres behov.

Fejlfinding

- Tjek Cilium pods med:
```bash
kubectl -n kube-system get pods -l k8s-app=cilium
```
