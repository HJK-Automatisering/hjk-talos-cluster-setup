---
layout: default
title: vLLM
nav_order: 10
---

# 10 – vLLM (Inference & Embeddings)

This section documents how we deploy **vLLM** using the **vLLM Production Stack Helm chart** to run:
- a large **instruct model**
- an **embeddings model**

The deployment assumes:
- a GPU-enabled Talos Kubernetes node
- NVIDIA drivers and device plugin are working
- a Kubernetes `RuntimeClass` named **`nvidia`** exists and is managed via **ArgoCD**

---

## Project structure

The vLLM application lives under:

`applications/vllm/`

```text
applications/vllm/
├── Chart.yaml
├── gpu-test.yml
├── local-secrets/
│   ├── hf-secret.yaml
│   └── vllm-secret.yaml
├── templates/
│   ├── metrics-service.yml
│   ├── sealed-hf-secret.yaml
│   └── sealed-vllm-secret.yaml
└── values.yaml
```

---

## Prerequisites

### NVIDIA GPU availability

Before deploying vLLM, ensure that GPU scheduling works.

A test pod is provided:

`applications/vllm/gpu-test.yml`

```bash
kubectl apply -f applications/vllm/gpu-test.yml
kubectl logs -f pod/gpu-test
```

Expected output must include `nvidia-smi` information.

---

### RuntimeClass `nvidia`

The vLLM chart explicitly uses:

```yaml
runtimeClassName: "nvidia"
```

Therefore, Kubernetes must provide a matching `RuntimeClass`.

This repository defines it as a manifest managed by **ArgoCD**:

`k8s/manifests/runtimeclass-nvidia.yaml`

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia
```

> This RuntimeClass must exist **before** vLLM pods are scheduled.

Verify:

```bash
kubectl get runtimeclass nvidia
```

---

## Helm chart

`applications/vllm/Chart.yaml`:

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

## Secrets

vLLM requires two secrets:

1. **Hugging Face token** (model downloads)
2. **vLLM API key** (OpenAI-compatible endpoint)

Plaintext secrets are stored locally and sealed before committing.

---

## Configuration overview (`values.yaml`)

### Deployment strategy

Single-GPU environment requires:

```yaml
strategy:
  type: Recreate
```

This avoids GPU contention by ensuring the old pod is terminated before a new one is scheduled.

---

### RuntimeClass

All vLLM engine pods use:

```yaml
runtimeClassName: "nvidia"
```

This ensures pods run using the NVIDIA container runtime.

---

### Node selection (important)

Both the **inference model** and **embeddings model** explicitly define `nodeSelectorTerms`.

Example from `values.yaml`:

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

#### Why this matters

- In **AarhusAI**, GPU workloads typically run on **virtual GPUs (vGPU)**.
  - Nodes are selected using the label:
    ```yaml
    nvidia.com/vgpu.present: "true"
    ```
  - Multiple GPU nodes may exist, and scheduling is more dynamic.

- In **this cluster**, we run on **bare metal with physical GPUs**.
  - No vGPU layer is present
  - Therefore, the `nvidia.com/vgpu.present` selector is **commented out**
  - Pods are pinned directly to a specific node using:
    ```yaml
    kubernetes.io/hostname: NODE
    ```

Replace `NODE` with the actual hostname of your GPU node.

> If additional GPU nodes are added in the future, this logic should be revisited and generalized.

---

### Models

Two models are configured:

#### Instruct model

- Large instruct model (e.g. Mistral)
- GPU: 1
- Memory: ~56Gi
- Persistent storage for model cache
- Chunked prefill and prefix caching enabled

#### Embeddings model

- Smaller embedding model
- GPU: 1
- Memory: ~16Gi
- CUDA graph explicitly disabled to support embedding model loading

---

### API key configuration

```yaml
vllmApiKey:
  secretName: "vllm-secret"
  secretKey: "KEY"
```

---

### Tolerations

GPU nodes are typically tainted. vLLM pods tolerate GPU taints:

```yaml
tolerations:
  - key: "node-role.kubernetes.io/gpu"
    operator: "Exists"
```

---

## Deployment via ArgoCD

Recommended order:

1. GPU enablement completed
2. RuntimeClass `nvidia` applied via ArgoCD
3. Sealed secrets committed
4. ArgoCD application synced

---

## Verification

```bash
kubectl get pods -n vllm
kubectl get svc -n vllm
kubectl get secret -n vllm
```

---

## Summary

vLLM is deployed using the **vLLM Production Stack** and managed entirely via **GitOps**.

Key requirements:
- Functional NVIDIA GPU support
- RuntimeClass `nvidia` managed by ArgoCD
- Correct node selection strategy (vGPU vs physical GPU)
- Sealed Secrets for credentials
- ArgoCD as the single source of truth
