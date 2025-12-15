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
**File:** `docs/06-nvidia-gpu.md`

---

### 7. Troubleshooting and FAQ
Common failure scenarios related to Talos, Kubernetes, Cilium, Rook Ceph, WSL2, and recovery procedures.

**File:** `docs/90-troubleshooting.md`

---

## Scope and intent

This documentation supports:

- Operation and maintenance of a **local single-node Kubernetes cluster** for AI workloads
- Internal OS2 projects, including **OS2AI** and **Memoctopus**
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
