---
layout: default
title: NVIDIA GPU Enablement (H100)
nav_order: 6
parent: Cluster Foundation
---

# NVIDIA GPU Enablement (H100)

This section describes how **NVIDIA GPUs** (specifically **NVIDIA H100 NVL**) are enabled on a **single-node Talos Kubernetes cluster**.

GPU enablement is intentionally split into **two distinct layers**:

1. **Talos OS layer** – NVIDIA drivers and container runtime integration via Talos system extensions  
2. **Kubernetes layer** – RuntimeClass and NVIDIA device plugin to expose GPUs as schedulable resources

This documentation explains **what is required and why**.  
The concrete configuration, patches, and automation live in the separate infrastructure repository.

---

## Scope and assumptions

This guide assumes:

- A **single-node, on-prem Talos cluster**
- Kubernetes already bootstrapped and reachable
- Cilium installed and functional
- NVIDIA GPUs physically present in the node

This setup is intended for:
- AI workloads
- Development and PoC environments
- Internal platforms (OS2AI / AarhusAI)

It is **not** a high-availability production GPU platform.

---

## 1. Prerequisites and quick validation

Before enabling GPUs, verify basic cluster health.

### Talos health
```bash
talosctl health --nodes <NODE_IP>
```

### Kubernetes access
```bash
kubectl get nodes
kubectl get pods -A
```

The node must be **Ready** and the Kubernetes API reachable.

---

## 2. Talos GPU support overview

Talos Linux is immutable and does not support installing drivers via SSH or package managers.

GPU support is provided via **Talos system extensions**, managed through **Image Factory** and applied using a Talos upgrade.

For NVIDIA GPUs, the following extensions are required:

- `siderolabs/nvidia-open-gpu-kernel-modules-production`
- `siderolabs/nvidia-container-toolkit-production`

> **Important**  
> Always keep the NVIDIA kernel modules and container toolkit on the **same version track**  
> (do not mix `lts` and `production`).

---

## 3. Updating Talos with NVIDIA extensions

### 3.1 Generate a new Image Factory schematic

Using Image Factory, generate a new **schematic ID** that includes:

- NVIDIA open GPU kernel modules
- NVIDIA container toolkit

This schematic defines the Talos installer image used for upgrades.

---

### 3.2 Upgrade Talos using the Image Factory installer image

From the infrastructure repository (or anywhere with access to `talos/generated/talosconfig`):

```bash
export TALOSCONFIG="$(pwd)/talos/generated/talosconfig"

talosctl --talosconfig "$TALOSCONFIG" --nodes <NODE_IP> upgrade --image factory.talos.dev/installer/<SCHEMATIC_ID>:v1.11.5
```

During the upgrade:
- Talos reboots the node
- NVIDIA extensions are installed
- Existing configuration is preserved

After the node comes back, verify the extensions:

```bash
talosctl --talosconfig "$TALOSCONFIG" --nodes <NODE_IP> get extensions
```

You should see:
- `nvidia-open-gpu-kernel-modules-*`
- `nvidia-container-toolkit-*`

---

## 4. Loading NVIDIA kernel modules in Talos

Installing the extensions alone is not sufficient.  
The NVIDIA kernel modules must be explicitly loaded via a Talos machine configuration patch.

In the infrastructure repository, this patch is located at:

```
talos/patches/nvidia-oss-modules.yaml
```

### Example patch
```yaml
machine:
  kernel:
    modules:
      - name: nvidia
      - name: nvidia_uvm
      - name: nvidia_drm
      - name: nvidia_modeset
  sysctls:
    net.core.bpf_jit_harden: 1
```

Apply the patch:

```bash
export TALOSCONFIG="$(pwd)/talos/generated/talosconfig"

talosctl --talosconfig "$TALOSCONFIG" --nodes <NODE_IP> patch mc --patch @talos/patches/nvidia-oss-modules.yaml
```

### Verify driver and modules

```bash
talosctl --talosconfig "$TALOSCONFIG" --nodes <NODE_IP> read /proc/driver/nvidia/version

talosctl --talosconfig "$TALOSCONFIG" --nodes <NODE_IP> read /proc/modules | grep -E 'nvidia|nvidia_uvm|nvidia_drm|nvidia_modeset'
```

If the driver version is shown and modules are loaded, Talos GPU support is complete.

---

## 4.1 NVIDIA container runtime (no-cgroups)

Talos uses **cgroups v2**.  
In this setup, the NVIDIA container runtime is configured with `no-cgroups = true`.

Patch location:

```
talos/patches/nvidia-no-cgroups.yaml
```

Example:
```yaml
machine:
  files:
    - op: create
      path: /etc/nvidia-container-runtime/config.toml
      permissions: 0o644
      content: |
        [nvidia-container-cli]
        no-cgroups = true
```

> Without this setting, GPU workloads may fail to start with container runtime errors.

---

## 5. Kubernetes RuntimeClass for NVIDIA

Kubernetes must be instructed to use the NVIDIA runtime when running GPU workloads.

This is done using a `RuntimeClass`.

In the infrastructure repository, the manifest is stored at:

```
k8s/manifests/runtimeclass-nvidia.yaml
```

### RuntimeClass definition
```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia
```

> **GitOps note**  
> In this setup, the RuntimeClass is applied via **ArgoCD**.  
> Avoid creating it manually with `kubectl` except for temporary debugging.

Verify:

```bash
kubectl get runtimeclass nvidia
```

---

## 6. NVIDIA device plugin (Kubernetes)

The NVIDIA device plugin exposes GPUs to Kubernetes as schedulable resources
(e.g. `nvidia.com/gpu`).

### Important note (single-node clusters)

The upstream NVIDIA device plugin chart may include **node affinity rules**
that depend on Node Feature Discovery (NFD).

In this setup:
- NFD is **not installed**
- Affinity must be disabled explicitly

---

### 6.1 Device plugin Helm values

In the infrastructure repository:

```
k8s/nvidia-device-plugin/values.yaml
```

Example values:

```yaml
runtimeClassName: nvidia

# Disable NFD/GFD-based scheduling requirements
affinity: null

nodeSelector:
  kubernetes.io/os: linux

tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
    effect: "NoSchedule"
  - key: "CriticalAddonsOnly"
    operator: "Exists"
```

---

### 6.2 Install or upgrade the device plugin

```bash
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update

helm upgrade --install nvidia-device-plugin nvdp/nvidia-device-plugin -n kube-system -f k8s/nvidia-device-plugin/values.yaml
```

### Verify scheduling

```bash
kubectl get ds -n kube-system | grep -i nvidia
kubectl get pods -n kube-system | grep -i nvidia
```

The DaemonSet should show **DESIRED = 1**.

---

## 7. Verify GPU availability in Kubernetes

Check node resources:

```bash
kubectl describe node <NODE_NAME> | grep -A10 -i nvidia.com/gpu
```

Expected output:
- `nvidia.com/gpu: 2` (for two H100 GPUs)

---

## 8. Test GPU access from a pod

Run a simple CUDA container and execute `nvidia-smi`:

```bash
kubectl run nvidia-test --restart=Never -ti --rm --image nvcr.io/nvidia/cuda:12.5.0-base-ubuntu22.04 --overrides '{"spec": {"runtimeClassName": "nvidia"}}' nvidia-smi
```

If the command prints GPU information, GPU enablement is successful.

> A PodSecurity warning may appear for this test pod.  
> This does **not** affect GPU functionality.

---

## Next steps

Once GPU enablement is complete, the cluster is ready for GPU workloads such as:

- **09 – Sealed Secrets** (secure secret management)
- **10 – vLLM** (GPU-based inference and embeddings, requires `runtimeClassName: nvidia`)

---

## Summary

GPU enablement is complete when:

- Talos reports an NVIDIA driver version
- NVIDIA kernel modules are loaded
- Kubernetes advertises `nvidia.com/gpu`
- `nvidia-smi` runs successfully inside a pod

At this point, the cluster is ready for GPU-accelerated workloads.
