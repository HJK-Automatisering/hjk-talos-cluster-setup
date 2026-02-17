---
layout: default
title: vLLM
nav_order: 1
parent: Applications
---

# vLLM (Inference & Embeddings)

This chapter documents how we deploy **vLLM** using the **vLLM Production Stack Helm chart** to run:

- a large **instruct model**
- an **embeddings model**

The deployment assumes:

- a GPU-enabled Talos Kubernetes node
- NVIDIA drivers and device plugin are working
- a Kubernetes `RuntimeClass` named **`nvidia`** exists (managed via GitOps)

---

## Vendor vs overrides (GitOps model)

This documentation follows the repository structure used by the cluster IaC repository:

- **Vendor** contains the upstream application Helm chart (as a submodule)
- **Overrides** contains municipality / cluster-specific configuration (values + templates)

For vLLM this means:

- Vendor chart: `vendor/applications/vllm`
- Overrides: `overrides/vllm`

> The vLLM application is deployed via **Argo CD** (app-of-apps) and should be treated as GitOps-managed.  
> Manual `helm install` is not recommended for day-to-day operation.

---

## Project structure

### Vendor chart (upstream)

`vendor/applications/vllm/`

```
vendor/applications/vllm/
├── Chart.yaml
├── gpu-test.yml
├── templates
│   └── metrics-service.yml
└── values.yaml
```

### Overrides (cluster-specific)

`overrides/vllm/`

```
overrides/vllm/
├── templates
│   ├── sealed-hf-secret.yaml
│   └── sealed-vllm-secret.yaml
└── values.yaml
```

---

## Prerequisites

### NVIDIA GPU availability

Before deploying vLLM, ensure that GPU scheduling works.

A test pod is provided by the vendor chart:

`vendor/applications/vllm/gpu-test.yml`

```bash
kubectl apply -f vendor/applications/vllm/gpu-test.yml
kubectl logs -f pod/gpu-test
```

Expected output must include `nvidia-smi` information.

---

### RuntimeClass `nvidia`

The vLLM deployment uses:

```yaml
runtimeClassName: "nvidia"
```

Therefore, Kubernetes must provide a matching `RuntimeClass`.

This cluster defines it as a manifest (managed by GitOps):

`k8s/manifests/runtimeclass-nvidia.yaml`

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia
```

Verify:

```bash
kubectl get runtimeclass nvidia
```

> This RuntimeClass must exist **before** vLLM pods are scheduled.

---

## Helm chart (vendor)

The vLLM application is defined as a Helm chart with a dependency on the upstream vLLM Production Stack:

`vendor/applications/vllm/Chart.yaml`

```yaml
apiVersion: v2
name: vllm
version: 0.1.6
dependencies:
  - name: vllm-stack
    alias: vllm
    version: 0.1.8
    repository: https://vllm-project.github.io/production-stack
```

---

## Secrets (Hugging Face + vLLM API key)

vLLM requires two secrets:

1. **Hugging Face token** (for downloading models)
2. **vLLM API key** (for OpenAI-compatible access)

In the GitOps model, secrets are committed as **SealedSecrets**:

- `overrides/vllm/templates/sealed-hf-secret.yaml`
- `overrides/vllm/templates/sealed-vllm-secret.yaml`

> Never commit plaintext tokens into Git.  
> Use Sealed Secrets (kubeseal) and keep plaintext values local only.

---

## Configuration overview (overrides)

The main cluster-specific configuration is stored in:

`overrides/vllm/values.yaml`

The override file typically configures:

- `runtimeClassName: nvidia`
- model images and tags
- model URLs (Hugging Face / HF Hub identifiers)
- resources (CPU / memory / GPU requests)
- persistent storage for model cache
- node selection strategy (vGPU vs physical GPU)

---

### Deployment strategy (single GPU)

In a single-GPU environment, it is common to use a deployment strategy that avoids GPU contention.  
Depending on the upstream chart, this can be implemented as:

- `strategy: Recreate`, or
- `replicaCount: 1` with careful rollout configuration

If you see GPU scheduling conflicts during rollouts, prefer **Recreate-like behavior** so the old pod is terminated before a new one starts.

---

### RuntimeClass

All vLLM engine pods should use:

```yaml
runtimeClassName: "nvidia"
```

This ensures pods run using the NVIDIA container runtime.

---

### Node selection (important)

Both the instruct and embeddings models can define `nodeSelectorTerms`.

Example pattern:

```yaml
nodeSelectorTerms:
  - matchExpressions:
      # - key: nvidia.com/vgpu.present
      #   operator: "In"
      #   values:
      #     - "true"
      - key: kubernetes.io/hostname
        operator: "In"
        values:
          - NODE
```

Replace `NODE` with the hostname of your GPU node(s).

---

## Deployment via Argo CD

High-level deployment order:

1. GPU enablement completed  
2. RuntimeClass `nvidia` applied  
3. Sealed secrets committed  
4. Argo CD application synced  

---

## Verification

```bash
kubectl get pods -n vllm
kubectl get svc -n vllm
kubectl get secret -n vllm
```

---

## Summary

vLLM is deployed using the **vLLM Production Stack** and managed via **GitOps**.
