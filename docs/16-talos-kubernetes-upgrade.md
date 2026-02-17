---
title: 16 – Talos & Kubernetes Upgrades
nav_order: 3
parent: Operations
---

# Talos & Kubernetes Upgrades

This chapter describes how to safely upgrade:

- Talos OS (node image)
- `talosctl` client
- Kubernetes control plane version

This documentation assumes:

- Single-node Talos cluster
- NVIDIA GPU enabled via system extensions
- Upgrade performed during a maintenance window

> ⚠️ Always perform upgrades during a planned maintenance window.
> In a single-node cluster, the control plane will be temporarily unavailable.

---

# Upgrade Strategy Overview

Correct upgrade order:

1. Determine new Talos version
2. Generate new Talos factory image (including NVIDIA extensions)
3. Upgrade Talos OS
4. Upgrade `talosctl` client
5. Upgrade Kubernetes version
6. Validate cluster health

---

# 1. Determine Target Versions

Check current Talos version:

```bash
talosctl version
```

Check Kubernetes version:

```bash
kubectl version --short
```

Review compatibility matrix:
https://www.talos.dev

Upgrade one Kubernetes minor version at a time.

---

# 2. Generate Talos Factory Image (with NVIDIA Extensions)

Go to:

https://factory.talos.dev/

Select:

- Machine type: **bare-metal**
- Desired Talos version (e.g. `v1.12.4`)
- Add system extensions:

```yaml
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/nvidia-container-toolkit-production
      - siderolabs/nvidia-open-gpu-kernel-modules-production
```

The factory generates a **schematic ID**.

Example:

```
6698d6f136c5bb37ca8bb8482c9084305084da0a5ead1f4dcae760796f8ab3a2
```

---

# 3. Upgrade Talos OS

```bash
talosctl upgrade \
  --image factory.talos.dev/metal-installer/<SCHEMATIC_ID>:v1.12.4
```

Example:

```bash
talosctl upgrade \
  --image factory.talos.dev/metal-installer/6698d6f136c5bb37ca8bb8482c9084305084da0a5ead1f4dcae760796f8ab3a2:v1.12.4
```

Monitor progress:

```bash
talosctl get members
talosctl health
```

Wait until node is Ready and Kubernetes API responds.

---

# 4. Upgrade talosctl Client

Remove old version:

```bash
sudo rm /usr/local/bin/talosctl
```

Reinstall:

```bash
curl -sL https://talos.dev/install | sh
```

Verify:

```bash
talosctl version
```

Client and node versions should align.

---

# 5. Upgrade Kubernetes

Dry-run first:

```bash
talosctl upgrade-k8s --to 1.35.1 --dry-run
```

Execute upgrade:

```bash
talosctl upgrade-k8s --to 1.35.1
```

---

# 6. Post-Upgrade Validation

Check Kubernetes:

```bash
kubectl version --short
kubectl get nodes
kubectl get pods -A
```

Verify GPU:

```bash
kubectl get pods -n kube-system | grep nvidia
kubectl describe node | grep nvidia
```

Verify platform services (Argo CD, Grafana, applications).

---

# Best Practices

- Upgrade one minor version at a time.
- Test in non-production environment first.
- Record Talos version, schematic ID, Kubernetes version, and date.
- Ensure Ceph and cluster health before upgrade.

---

# Summary

Upgrade order:

1. Generate factory image
2. `talosctl upgrade`
3. Upgrade `talosctl`
4. `talosctl upgrade-k8s`
5. Validate cluster health
