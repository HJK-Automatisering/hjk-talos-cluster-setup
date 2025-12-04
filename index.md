---
layout: default
title: Introduktion
nav_order: 1
---

# HJK Talos Cluster Setup

Velkommen til dokumentationen for opsætning og drift af en **single-node Kubernetes-klynge** baseret på:

- **Talos Linux**  
- **Cilium** (CNI)  
- **Rook Ceph** (single-node storage til test og udvikling)  
- **OS2AI**  
- **Memoctopus MVP**  
- Adgang via **Ubuntu WSL2** på udviklermaskiner  

Formålet med denne dokumentation er at skabe en ensartet, reproducerbar og gennemsigtig proces for installation, drift og fejlfinding af klyngen, både for nuværende og fremtidige medarbejdere.

---

# Dokumentationsstruktur

Nedenfor følger en oversigt over dokumentets hovedsektioner.

## 1. Miljøforberedelse  
Hardware, netværk, statisk IP, WSL2-installation og værktøjer:  
➡ *docs/01-environment.md*

## 2. Installation og bootstrap af Talos  
ISO-boot, clusterconfig, patches, apply-scripts og bootstrap af control-plane:  
➡ *docs/02-bootstrap.md*

## 3. Installation af Cilium  
Installation via Helm, brug af projektets values.yaml og netværksvalidering:  
➡ *docs/03-cilium.md*

## 4. Installation af Rook Ceph  
Single-node Ceph-deployment, storageclass og kendte begrænsninger:  
➡ *docs/04-rook-ceph.md*

## 5. Adgang til cluster fra WSL2  
kubeconfig, talosctl og netværksadgang fra udviklermaskiner:  
➡ *docs/05-wsl-access.md*

## 6. Fejlfinding og FAQ  
Talos, Kubernetes, Cilium, Ceph, WSL2 og konfigurationsrelaterede problemer:  
➡ *docs/90-troubleshooting.md*

---

# Formål og omfang

Dokumentationen understøtter:

- Drift og vedligehold af lokal single-node Kubernetes til AI-formål  
- Interne OS2-projekter, herunder OS2AI og Memoctopus  
- Reproducerbar installation baseret på scripts i projektets *Talos/Bash/*-mappe  
- Klar afgrænsning mellem testmiljø (single-node) og produktion  

---

# Målgruppe

Dokumentationen er målrettet:

- IT-drift  
- Systemadministratorer  
- Udviklere med ansvar for OS2AI og beslægtede systemer  
- Fremtidig kommunal teknisk drift  

---

# Forudsætninger

Brugeren bør have kendskab til:

- Linux / WSL2  
- Kubernetes grundbegreber  
- YAML-konfigurationer  
- Grundlæggende netværksforståelse  
- Helm-baserede deployments  

Alle Talos-relaterede begreber forklares undervejs.

---

# Kom i gang

Start med:

➡ **01 – Miljøforberedelse**  
for at sikre, at arbejdsstation, netværk og server er klar til installationen.

