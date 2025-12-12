# HJK Talos Cluster Setup

This repository contains the **technical documentation** for setting up and operating a **single-node Kubernetes cluster** based on:

- **Talos Linux**
- **Cilium** (CNI)
- **Rook Ceph** (single-node, non-HA storage)
- **OS2AI**
- Access from **Ubuntu WSL2** on developer machines

The goal is to ensure a **consistent, reproducible, and well-documented** installation and operational process for Kubernetes-based AI workloads in **Hjørring Municipality**.

> This repository contains **documentation only**.  
> Infrastructure-as-Code (IaC), scripts, and cluster configuration live in a **separate repository**.

---

## Documentation

All documentation is located in the **`docs/`** directory and is published via **GitHub Pages**.

**Live documentation:**  
https://hjk-automatisering.github.io/hjk-talos-cluster-setup/

---

## Documentation Structure

| File | Description |
|-----|------------|
| `index.md` | Introduction and overview |
| `01-environment.md` | Hardware, network, WSL2, and tooling preparation |
| `02-bootstrap.md` | Talos installation, patches, and Kubernetes bootstrap |
| `03-cilium.md` | Cilium installation and validation |
| `04-rook-ceph.md` | Rook Ceph installation (single-node) |
| `05-wsl-access.md` | Cluster access from WSL2 |
| `90-troubleshooting.md` | Troubleshooting and FAQ |

---

## Related Repositories

This documentation is designed to be used together with:

- **Infrastructure repository (IaC & scripts):**  
  https://github.com/HJK-Automatisering/hjk-talos-cluster

The IaC repository contains:
- Talos patches and generated configs
- Bash scripts and Taskfile automation
- Helm values for Cilium and Rook Ceph
- Cluster-specific configuration (not public)

---

## Prerequisites

Before following this documentation, you should have:

- Windows 11 with **WSL2 (Ubuntu)**
- `talosctl`
- `kubectl`
- `helm`
- `cilium` CLI
- Network access to the server’s **static IP**
- iDRAC access to mount the Talos ISO
- Basic knowledge of:
  - Linux
  - Kubernetes fundamentals
  - YAML and Helm

---

## Quick Start

1. Read **01 – Environment preparation**
2. Follow **02 – Talos installation and bootstrap**
3. Install:
   - Cilium
   - Rook Ceph
4. Deploy **OS2AI**
5. Deploy **Memoctopus MVP**

Once the cluster is running, verify access:

```bash
talosctl version
kubectl get nodes
helm ls -A
```

---

## Operations & Troubleshooting

Talos diagnostics:

```bash
talosctl health
talosctl logs <service>
talosctl dmesg
```

Kubernetes diagnostics:

```bash
kubectl get pods -A
kubectl describe pod <pod>
```

For common issues and solutions, see:

**`90-troubleshooting.md`**

---

## Ownership

Maintained by:

**Hjørring Municipality – IT & Digitalisation**  
Responsible for Kubernetes and AI platform infrastructure.
