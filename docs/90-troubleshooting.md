---
layout: default
title: Troubleshooting and FAQ
nav_order: 90
---

# Troubleshooting and FAQ

This section collects the most common failure scenarios when operating a **single-node Talos Kubernetes cluster**, along with proven solutions based on project scripts, configuration patterns, and known platform limitations.

The content is based on hands-on experience during installation and operation.

---

## 1. General troubleshooting checklist

Always start with the basics:

```bash
talosctl health
kubectl get nodes
kubectl get pods -A
```

If Talos does not respond:

- The server may still be running from the Talos ISO (maintenance mode)
- Static IP configuration in `network.yaml` may be incorrect
- Firewall or routing between WSL2 and the server may block traffic

---

## 2. Talos-related issues

### 2.1 `connection refused` or timeout from talosctl

Common causes:

- Talos endpoint or node not configured:
  ```bash
  talosctl config endpoint <NODE_IP>
  talosctl config node <NODE_IP>
  ```
- Node is still running from ISO
- Static IP was not applied correctly

Validate:

```bash
ping <NODE_IP>
talosctl version
```

---

### 2.2 Talos health reports errors

```bash
talosctl health --nodes <NODE_IP>
```

Typical problems:

**Kubernetes API unavailable**
- Cilium not running or misconfigured
- Certificate issues during bootstrap
- Invalid controlplane configuration

**etcd issues**
- Bootstrap not executed:
  ```bash
  talosctl bootstrap --nodes <NODE_IP>
  ```
- System time out of sync (check NTP)

---

## 3. Kubernetes API errors (`context deadline exceeded`)

```bash
kubectl get pods -A
```

Possible causes:

- Cilium not installed or failing
- kube-apiserver in restart loop
- kubeconfig not generated yet

Inspect logs:

```bash
talosctl logs kube-apiserver
talosctl logs kubelet
```

---

## 4. Cilium issues

### 4.1 Cilium pods not ready

```bash
cilium status
kubectl -n kube-system get pods
```

Common causes:

- Wrong `k8sServiceHost` / `k8sServicePort`
- Talos default CNI not disabled
- Kernel or BPF limitations

### 4.2 Recovery

```bash
helm upgrade --install cilium ./k8s/charts/cilium   --namespace kube-system   --values ./k8s/charts/cilium/values.yaml
```

Connectivity test:

```bash
cilium connectivity test
```

---

## 5. Rook Ceph issues (single-node)

```bash
kubectl -n rook-ceph get cephcluster
kubectl -n rook-ceph get pods
```

### 5.1 OSDs not starting
- Disks already formatted or mounted
- Wrong device filters
- Permission issues

### 5.2 MON / MGR instability
- Insufficient resources
- Expected warnings in single-node setups

### 5.3 StorageClass missing

```bash
kubectl get storageclass
```

Set default if needed:

```bash
kubectl patch storageclass rook-ceph-block   -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## 6. kubeconfig problems

### 6.1 API server unreachable

```bash
talosctl kubeconfig --nodes <NODE_IP> --force
```

Verify:

```bash
kubectl config view
kubectl get nodes
```

### 6.2 WSL2 cannot reach node

Common causes:

- Firewall blocking WSL2 subnet (often 172.x.x.x)
- WSL2 IP changed after reboot
- Missing routing between VLANs

```bash
ip route
ping <NODE_IP>
```

---

## 7. Pods cannot schedule on single-node cluster

Cause:

```yaml
allowSchedulingOnControlPlanes: false
```

Fix by enabling scheduling:

```yaml
cluster:
  allowSchedulingOnControlPlanes: true
```

---

## 8. NVIDIA GPU issues

This section covers common problems related to NVIDIA GPU enablement on Talos-based Kubernetes clusters.

GPU support spans **both Talos OS and Kubernetes**, so issues may appear at either layer.

---

### 8.1 NVIDIA device plugin DaemonSet shows `DESIRED = 0`

Check:

```bash
kubectl get ds -n kube-system | grep -i nvidia
```

If `DESIRED` is `0`, the DaemonSet is not matching any nodes.

**Common cause**

The NVIDIA device plugin chart may include a `nodeAffinity` that depends on
Node Feature Discovery (NFD) labels, for example:

- `feature.node.kubernetes.io/pci-10de.present=true`
- `nvidia.com/gpu.present=true`

In this setup, **NFD is not installed**, so the affinity prevents scheduling.

**Fix**

Disable affinity in the Helm values:

```yaml
affinity: null
```

Then upgrade the release:

```bash
helm upgrade --install nvidia-device-plugin nvdp/nvidia-device-plugin   -n kube-system   -f k8s/charts/nvidia-device-plugin/values.yaml
```

Verify:

```bash
kubectl get ds -n kube-system | grep -i nvidia
kubectl get pods -n kube-system | grep -i nvidia
```

---

### 8.2 GPUs not visible on the node (`nvidia.com/gpu` missing)

Check node resources:

```bash
kubectl describe node <NODE_NAME> | grep -A10 -i nvidia.com/gpu
```

If no GPU resources are shown:

- Verify the NVIDIA device plugin pod is running
- Verify Talos has loaded NVIDIA kernel modules

Check on Talos:

```bash
talosctl read /proc/driver/nvidia/version
talosctl read /proc/modules | grep nvidia
```

If modules are missing, reapply the NVIDIA kernel module patch and reboot.

---

### 8.3 `nvidia-smi` fails inside a pod

Example test:

```bash
kubectl run nvidia-test   --restart=Never   -ti --rm   --image nvcr.io/nvidia/cuda:12.5.0-base-ubuntu22.04   --overrides '{"spec": {"runtimeClassName": "nvidia"}}'   nvidia-smi
```

**Common causes**

- `runtimeClassName: nvidia` missing
- NVIDIA container toolkit extension not installed in Talos
- Device plugin not running

**Notes**

- A PodSecurity warning may appear for this test pod
- The warning does **not** prevent GPU access

---

### 8.4 NVIDIA driver missing after Talos upgrade

After upgrading Talos, GPUs may stop working if the installer image does not
include the required NVIDIA extensions.

Verify extensions:

```bash
talosctl get extensions
```

If NVIDIA extensions are missing, upgrade Talos using the correct Image Factory installer image:

```bash
talosctl upgrade   --image factory.talos.dev/installer/<SCHEMATIC_ID>:<TALOS_VERSION>
```

---

### 8.5 Quick GPU health checklist

```bash
talosctl read /proc/driver/nvidia/version
kubectl get ds -n kube-system | grep -i nvidia
kubectl describe node <NODE_NAME> | grep -A10 -i nvidia.com/gpu
```

If all three checks succeed, GPU support is operational.

---

## 9. Helm-related issues

### 9.1 Chart version conflicts

```bash
helm repo update
```

### 9.2 Values not applied

```bash
helm get values <release>
helm upgrade --install <release> <chart> --values values.yaml
```

---

## 10. Talos logging and diagnostics

Talos does not support SSH.

```bash
talosctl logs <service>
talosctl dmesg
talosctl service list
```

Examples:

```bash
talosctl logs containerd
talosctl logs kubelet
```

---

## 11. Emergency reboot

```bash
talosctl reboot --nodes <NODE_IP>
```

---

## 12. Known limitations

- Single-node Ceph is **not highly available**
- Control-plane and workloads share resources
- Cilium requires kernel BPF support
- Correct networking is critical during first apply

---

## 13. Full recovery procedure

If the cluster becomes unrecoverable:

```bash
task talos:build
task talos:apply-insecure
task talos:bootstrap
task talos:kubeconfig
```

---

## 14. Further reading

- Talos CLI reference: https://docs.siderolabs.com/talos/reference/cli/
- Cilium troubleshooting: https://docs.cilium.io/en/stable/
- Rook Ceph troubleshooting: https://rook.io/docs/rook/latest/Troubleshooting/
