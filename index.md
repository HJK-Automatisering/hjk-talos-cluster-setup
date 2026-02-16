---
layout: default
title: Introduction
nav_order: 1
---

# HJK Talos Cluster Setup

Welcome to the documentation for the setup and operation of a **single-node Kubernetes cluster** based on:

- **Talos Linux**
- **Cilium** (CNI)
- **Rook Ceph** (single-node storage for test and development)
- **OS2AI / AarhusAI**
- Access from **Ubuntu WSL2** on developer workstations

The purpose of this documentation is to provide a **consistent, reproducible, and transparent** approach to installing, operating, and troubleshooting the cluster, both for current and future team members.

This documentation describes the *how* and *why* of the platform.  
The actual **infrastructure-as-code (IaC)** implementation lives in a separate repository.

---

## Documentation structure

Below is an overview of the main documentation sections and their purpose.

1. [01 – Environment](docs/01-environment)
2. [02 – Bootstrap](docs/02-bootstrap)
3. [03 – Cilium](docs/03-cilium)
4. [04 – Rook Ceph](docs/04-rook-ceph)
5. [05 – WSL Access](docs/05-wsl-access)
6. [06 – NVIDIA GPU](docs/06-nvidia-gpu)
7. [07 – Argo CD (GitOps)](docs/07-argocd)
8. [08 – Observability](docs/08-observability)
9. [09 – Sealed Secrets](docs/09-sealed-secrets)
10. [10 – vLLM](docs/10-vllm)
11. [11 – LiteLLM](docs/11-litellm)
12. [12 – Open WebUI](docs/12-openwebui)
13. [13 – Upgrades](docs/13-upgrades)
14. [90 – Troubleshooting](docs/90-troubleshooting)

---

### 1. Environment preparation
Hardware requirements, networking, static IP addressing, WSL2 setup, and required tooling.

**File:** `docs/01-environment.md`

---

### 2. Talos installation and cluster bootstrap
Booting from ISO, generating cluster configuration, applying patches, and bootstrapping the control plane.

**File:** `docs/02-bootstrap.md`

---

### 3. Cilium installation
Installing Cilium as the Kubernetes CNI using Helm, validating networking, and troubleshooting datapath issues.

**File:** `docs/03-cilium.md`

---

### 4. Rook Ceph installation
Deploying Rook Ceph in a **single-node configuration**, configuring StorageClasses, and understanding limitations.

**File:** `docs/04-rook-ceph.md`

---

### 5. Cluster access from WSL2
Accessing Talos and Kubernetes from Windows via Ubuntu WSL2, including kubeconfig handling and networking considerations.

**File:** `docs/05-wsl-access.md`

---

### 6. NVIDIA GPU enablement
Enabling NVIDIA GPUs on Talos, including drivers, container runtime, RuntimeClass, and device plugin.

**File:** `docs/06-nvidia-gpu.md`
---

### 7. Argo CD (GitOps)
Bootstrapping GitOps with Argo CD and the app-of-apps pattern (Argo CD Resources), including authentication to private Git repositories.

**File:** `docs/07-argocd.md`
---

### 8. Observability (Prometheus, Grafana, Loki, Tempo)
Deploying and operating the cluster observability stack, including Grafana authentication and datasource wiring for logs and traces.

**File:** `docs/08-observability.md`

---

### 10. Sealed Secrets
Managing application secrets securely using Sealed Secrets in a GitOps workflow.

**File:** `docs/09-sealed-secrets.md`

---

### 11. vLLM
Deploying GPU-accelerated inference and embedding workloads using vLLM.

**File:** `docs/10-vllm.md`

---

### 12. LiteLLM
Deploying LiteLLM as an OpenAI-compatible proxy in front of model backends, including database persistence and guardrails.

**File:** `docs/11-litellm.md`

---

### 13. Open WebUI
Deploying Open WebUI as the user-facing chat interface, integrated with LiteLLM, vLLM, RAG, and persistence services.

**File:** `docs/12-openwebui.md`

---

### 14. Upgrading applications
How to upgrade platform components and applications using the vendor submodule and local overrides.

**File:** `docs/13-upgrades.md`

---

### 15. Troubleshooting and FAQ
Common failure scenarios related to Talos, Kubernetes, Cilium, Rook Ceph, WSL2, and recovery procedures.

**File:** `docs/90-troubleshooting.md`

---

## Scope and intent

This documentation supports:

- Operation and maintenance of a **local single-node Kubernetes cluster** for AI workloads
- Internal OS2 projects including **OS2AI**
- Reproducible infrastructure based on scripted workflows and declarative configuration
- A clear separation between:
  - **Test / development environments** (single-node)
  - **Future production-grade platforms** (multi-node, HA)

---

## Target audience

This documentation is intended for:

- IT operations staff
- System administrators
- Developers responsible for OS2AI and related platforms
- Future municipal technical operations teams

---

## Prerequisites

Readers are expected to have basic knowledge of:

- Linux and WSL2
- Kubernetes fundamentals
- YAML configuration files
- Basic networking concepts
- Helm-based application deployment

Talos-specific concepts and workflows are explained where relevant throughout the documentation.
