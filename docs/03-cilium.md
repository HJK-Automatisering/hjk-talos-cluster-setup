---
layout: default
title: Cilium Installation
nav_order: 4
---

# Cilium Installation

Cilium acts as the **Container Network Interface (CNI)** for the Kubernetes cluster and replaces the built‑in Talos CNI.
This section describes how **Cilium** is installed and managed using Helm and the scripts provided in the
`hjk-talos-cluster` infrastructure repository.

> **Important**  
> Talos’ built‑in CNI and kube‑proxy are disabled earlier via the Talos patch `talos/patches/cilium.yaml`.
> This is a strict prerequisite for running Cilium.

---

## Overview

The installation consists of three main steps:

1. Generate and maintain a `values.yaml` for Cilium  
2. Install or upgrade Cilium using Helm (via script or Taskfile)  
3. Validate networking and connectivity  

---

## 1. Generate Cilium Helm values (`values.yaml`)

Cilium configuration is managed using Helm values stored in:

```
k8s/charts/cilium/values.yaml
```

If the file does not already exist, it can be generated directly from the upstream Cilium Helm chart.

From the infrastructure repository root:

```bash
cd k8s/charts/cilium
```

Generate default values for the desired Cilium version:

```bash
helm show values cilium/cilium --version 1.18.4 > values.yaml
```

> **Note**  
> Version `1.18.4` is an example.  
> Always pin a specific version and commit `values.yaml` to Git to ensure reproducible deployments.

After generation, adjust `values.yaml` according to your environment.

### Example snippet

```yaml
hubble:
  ui:
    enabled: true

k8sServiceHost: 127.0.0.1
k8sServicePort: 7445
```

---

## 2. Install or upgrade Cilium

### 2.1 Add the Cilium Helm repository

This only needs to be done once per environment:

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
```

---

### 2.2 Install / upgrade using script

Cilium is installed using a dedicated script located at:

```
scripts/k8s/cilium-upgrade.sh
```

Run:

```bash
bash scripts/k8s/cilium-upgrade.sh
```

The script internally runs:

```bash
helm upgrade cilium cilium/cilium   --install   --version 1.18.4   --namespace kube-system   -f k8s/charts/cilium/values.yaml
```

This approach ensures:

- Idempotent installs and upgrades
- Version pinning
- Centralised configuration

---

## 3. Validate the installation

### 3.1 Verify Cilium pods

```bash
kubectl -n kube-system get pods -l k8s-app=cilium
kubectl -n kube-system get pods -l k8s-app=cilium-operator
```

All pods should be in **Running** state.

---

### 3.2 Check Cilium status

```bash
cilium status
```

Expected results:

- CNI initialized successfully
- Datapath: **Healthy**
- Hubble components running (if enabled)

---

### 3.3 Run connectivity tests

```bash
cilium connectivity test
```

This validates:

- DNS resolution
- Pod‑to‑pod communication
- Pod‑to‑service traffic
- Node routing

---

## 4. Hubble UI (optional)

If Hubble UI is enabled in `values.yaml`, it can be exposed locally:

```bash
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
```

Open a browser:

```
http://localhost:12000
```

Hubble UI provides live visibility into network flows and is useful for troubleshooting.

---

## Notes & best practices

- Always install Cilium **before** any workload is deployed
- Never mix Talos CNI and Cilium
- Keep `values.yaml` under version control
- Validate networking before proceeding to storage (Rook‑Ceph) or GitOps tools
