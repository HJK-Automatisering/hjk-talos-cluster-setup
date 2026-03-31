---
layout: default
title: Troubleshooting and FAQ
nav_order: 2
parent: Operations
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

---

## 15. CloudNativePG volume expansion and instance rebuild

This section covers a common recovery pattern for **CloudNativePG** when a Postgres instance
fails because the PVC is too small or the cluster enters a degraded state after storage pressure.

This procedure was validated during recovery of application databases running on
**Rook Ceph block storage** (`ceph-block`).

---

### 15.1 Symptoms

Typical symptoms:

```bash
kubectl get pods -A
kubectl get cluster -A
```

Examples:

- A CNPG instance is in `CrashLoopBackOff`
- Application pods are running but not ready
- `kubectl get cluster` reports:
  - `Not enough disk space`
  - `Waiting for the instances to become active`
  - `Cluster Is Not Ready`

CNPG logs may show:

```text
Detected low-disk space condition, avoid starting the instance
```

---

### 15.2 Verify the problem

Check cluster state:

```bash
kubectl -n <namespace> get cluster
kubectl -n <namespace> get cluster <cluster-name> -o yaml
kubectl -n <namespace> get pods -o wide
```

Check current PVC sizes:

```bash
kubectl -n <namespace> get pvc
kubectl -n <namespace> get pvc <pvc-name> -o yaml
```

Check whether the StorageClass supports expansion:

```bash
kubectl get storageclass ceph-block -o yaml
```

Expected:

```yaml
allowVolumeExpansion: true
```

If a healthy CNPG instance still exists, inspect filesystem usage:

```bash
kubectl -n <namespace> exec -it <healthy-db-pod> -c postgres -- df -h
kubectl -n <namespace> exec -it <healthy-db-pod> -c postgres -- du -sh /var/lib/postgresql/data/pgdata
kubectl -n <namespace> exec -it <healthy-db-pod> -c postgres -- du -sh /var/lib/postgresql/data/pgdata/pg_wal
```

---

### 15.3 Update desired storage in Git

Increase the CNPG storage size in Git before doing instance recovery.

Recommended pattern:

- keep application values in `overrides/<app>/values.yaml`
- keep CNPG-specific settings in a dedicated file:
  - `overrides/<app>/cloudnative-pg-values.yaml`

Example:

```yaml
cloudnative-pg:
  cluster:
    storage:
      size: 20Gi
```

Then ensure the Argo CD application includes the file as the last values file so it overrides vendor defaults.

Example pattern:

```yaml
valueFiles:
  - cloudnative-pg-values.yaml
  - values.yaml
  - ../../../overrides/<app>/values.yaml
  - ../../../overrides/<app>/cloudnative-pg-values.yaml
```

---

### 15.4 Validate with Helm before merge

Render the chart locally before merging:

```bash
helm dependency build ./vendor/applications/<app>

helm template <release-name> ./vendor/applications/<app> \
  -f ./vendor/applications/<app>/cloudnative-pg-values.yaml \
  -f ./vendor/applications/<app>/values.yaml \
  -f ./overrides/<app>/values.yaml \
  -f ./overrides/<app>/cloudnative-pg-values.yaml \
  > /tmp/<app>-rendered.yaml
```

Inspect the rendered CNPG cluster:

```bash
sed -n '/kind: Cluster/,+160p' /tmp/<app>-rendered.yaml
```

Verify:

```yaml
storage:
  size: 20Gi
```

---

### 15.5 If PVCs do not resize automatically

Even when Argo CD is synced and the CNPG cluster spec shows the new size, the PVCs may still remain at the old size.

Check:

```bash
kubectl -n <namespace> get cluster <cluster-name> -o yaml | grep -A3 'storage:'
kubectl -n <namespace> get pvc
```

If the cluster shows the new desired size but the PVCs still show the old size, patch the PVCs manually:

```bash
kubectl -n <namespace> patch pvc <pvc-1> --type merge -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
kubectl -n <namespace> patch pvc <pvc-2> --type merge -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

Verify:

```bash
kubectl -n <namespace> get pvc
kubectl -n <namespace> get pvc <pvc-1> -o yaml | grep -A6 'resources:'
kubectl -n <namespace> get pvc <pvc-2> -o yaml | grep -A6 'resources:'
```

---

### 15.6 Recover a broken instance by scaling down and back up

If one instance remains broken after storage expansion, recover the cluster by temporarily reducing replicas.

Scale down to one instance:

```bash
kubectl -n <namespace> patch cluster <cluster-name> --type merge -p '{"spec":{"instances":1}}'
```

Verify the spec changed:

```bash
kubectl -n <namespace> get cluster <cluster-name> -o jsonpath='{.spec.instances}{"\n"}'
```

If the broken instance does not disappear automatically, delete the Pod:

```bash
kubectl -n <namespace> delete pod <broken-db-pod>
```

Wait until the cluster is stable with one ready instance:

```bash
kubectl -n <namespace> get pods
kubectl -n <namespace> get cluster <cluster-name> -o yaml | grep -E 'readyInstances|currentPrimary|phase'
```

Expected result:

- only one DB pod remains
- phase becomes healthy
- applications depending on the database may recover

Then restore high availability:

```bash
kubectl -n <namespace> patch cluster <cluster-name> --type merge -p '{"spec":{"instances":2}}'
kubectl -n <namespace> get pods -w
```

CNPG should create a fresh replica automatically.

---

### 15.7 If the old instance still blocks recovery

If scaling down is not enough and the broken instance identity remains stuck, it may be necessary to delete the old PVC **after** the cluster is stable at one instance.

Only do this after confirming that one healthy instance remains.

Example:

```bash
kubectl -n <namespace> delete pvc <broken-instance-pvc>
```

This forces CNPG to rebuild the replica from the remaining healthy instance.

---

### 15.8 Verification after recovery

Check cluster health:

```bash
kubectl -n <namespace> get pods
kubectl -n <namespace> get cluster <cluster-name> -o yaml | grep -E 'readyInstances|currentPrimary|phase'
```

Expected:

- primary is running
- replica is running
- `readyInstances: 2`
- `phase: Cluster in healthy state`

Check application pods:

```bash
kubectl -n <namespace> get pods
```

For Open WebUI specifically, the web pods should return to `1/1 Running` once the database becomes healthy again.

---

### 15.9 Notes

- This recovery pattern was used successfully for both **Authentik** and **Open WebUI**
- The issue may present differently:
  - invalid storage shrink in GitOps
  - PVC too small / low-disk-space protection
  - broken replica stuck in restart loop
- On `ceph-block`, volume expansion is supported, but PVCs may still need manual patching
- Always update the GitOps source of truth first before performing manual recovery actions
