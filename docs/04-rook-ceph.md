---

## docs/04-rook-ceph.md

```markdown
---
layout: default
title: Installation af Rook Ceph
nav_order: 5
---

# Installation af Rook Ceph

Rook Ceph bruges til at levere distribueret storage til Kubernetes.

## Installation

1. Installer Rook operator:
```bash
kubectl create -f common.yaml
kubectl create -f operator.yaml
```

2. Opret Ceph cluster:
```bash
kubectl create -f cluster.yaml
```