---
layout: default
title: Environment Preparation
nav_order: 2
---

# Environment Preparation

This documentation describes the prerequisites and environment preparation for running a **single-node, on‑prem Kubernetes cluster** based on:

- **Talos Linux** (minimal, immutable Kubernetes OS)
- **Cilium** (Container Network Interface)
- **Rook Ceph** (storage backend)
- **NVIDIA GPU support** (H100) for AI workloads
- Access and operations from **Ubuntu WSL2**

> **Important**  
> This documentation explains *what* is required and *why*.  
> The actual infrastructure, configuration files, and automation live in a separate repository

---

## Target Architecture (High-level)

- Single physical server (on‑prem)
- Single Kubernetes control plane node
- Static IP networking
- Kubernetes accessed remotely from WSL2
- Infrastructure managed as code (IaC)

This setup is primarily intended for:
- internal platforms
- development
- proof‑of‑concept workloads  
(not high‑availability production)

---

## Hardware Overview

**Dell PowerEdge R760XA**

| Component | Specification |
|---------|---------------|
| CPU | 2 × Intel Xeon Gold 6526Y |
| Memory | 16 × 64 GB RDIMM |
| Storage | 6 × 1.92 TB SSD |
| GPU | 2 × NVIDIA H100 NVL |
| Out‑of‑band | iDRAC |

---

## Networking Requirements

The Talos node must be reachable via a **static IP address**.

| Parameter | Example value |
|---------|---------------|
| Node IP | `<NODE_IP>` |
| Subnet | `<SUBNET_CIDR>` |
| Gateway | `<GATEWAY_IP>` |
| DNS | `<DNS_IP>` |

> These values are **examples only**.  
> Actual IP addresses are defined in the infrastructure repository via:
> - `cluster.env`
> - Talos network patches

---

## Access Model

- Talos runs directly on bare‑metal
- Kubernetes API is exposed on port `6443`
- Administrators connect from **Ubuntu WSL2**
- No SSH access to Talos nodes (Talos design principle)

---

## Preparing Ubuntu WSL2

Ensure your WSL2 environment is fully updated:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Required Tooling

All cluster management is performed from WSL2.

### Talos CLI

```bash
curl -sL https://talos.dev/install | sh
sudo mv talosctl /usr/local/bin/
```

Verify installation:

```bash
talosctl version
```

---

### kubectl

```bash
sudo snap install kubectl --classic
```

Verify:

```bash
kubectl version --client
```

---

### Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify:

```bash
helm version
```

---

### Cilium CLI (optional but recommended)

```bash
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
tar xvf cilium-linux-amd64.tar.gz
sudo mv cilium /usr/local/bin/
```

Verify:

```bash
cilium version
```

---

## Network Connectivity Test

Verify basic connectivity from WSL2 to the Talos node:

```bash
ping <NODE_IP>
```
