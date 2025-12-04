---
layout: default
title: Miljøforberedelse
nav_order: 2
---

# Miljøforberedelse

Denne dokumentation beskriver opsætningen af en **single-node Kubernetes-klynge** baseret på:

- **Talos Linux**
- **Cilium**
- **Rook Ceph**
- Adgang fra **Ubuntu WSL2**

# Hardwareoversigt

**Dell PowerEdge R760XA**

| Komponent | Specifikation |
|----------|---------------|
| CPU | 2 × Intel Xeon Gold 6526Y |
| RAM | 16 × 64GB RDIMM |
| Storage | 6 × 1.92TB SSD |
| GPU'er | 2 × NVIDIA H100 NVL |
| OOB | iDRAC |

# Netværkskrav

Statisk IP-adresse anvendes.

| Parameter | Værdi |
|----------|-------|
| IP-adresse | `192.168.50.10` |
| Subnet | `255.255.255.0` |
| Gateway | `192.168.50.1` |
| DNS | Intern eller ekstern |

> Tilpas værdierne efter jeres miljø.

# Forberedelse af WSL2

```bash
sudo apt update && sudo apt upgrade -y
```

# Installation af værktøjer

## Talos CLI
```bash
curl -sL https://talos.dev/install | sh
sudo mv talosctl /usr/local/bin/
```

## kubectl
```bash
sudo snap install kubectl --classic
```

## Helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Cilium CLI
```bash
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
tar xvf cilium-linux-amd64.tar.gz
sudo mv cilium /usr/local/bin/
```

# Netværksadgang fra WSL2

```bash
ping 192.168.50.10
```

# Arbejdsstruktur

```
~/Talos/
  ├── Bash/
  ├── templates/
  ├── secrets/
  ├── clusterconfig/
```
