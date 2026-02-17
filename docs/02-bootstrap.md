---
layout: default
title: Talos Installation and Cluster Bootstrap
nav_order: 2
parent: Cluster Foundation
---

# Talos Installation and Cluster Bootstrap

This section describes the complete installation and bootstrap process for **Talos Linux** on a **single-node Kubernetes cluster**.

The process is based on:
- Bash scripts
- Talos configuration patches
- Taskfile-based automation

All infrastructure code lives in the separate repository

This documentation focuses on **what happens and in which order**, not the exact implementation details.

---

## Overview of the bootstrap process

High-level steps:

1. Boot the server from Talos ISO  
2. Generate Talos configuration (control plane)  
3. Apply configuration to the node  
4. Bootstrap Kubernetes  
5. Retrieve kubeconfig and verify access  

---

## 1. Boot from Talos ISO

1. Mount the Talos ISO via iDRAC (or equivalent out-of-band management)
   - Download from: https://github.com/siderolabs/talos/releases
   - Use the image: `metal-amd64.iso`

2. Boot the server.

3. During the first boot, Talos will obtain a **temporary DHCP address**.

> This DHCP address is only used for initial installation and troubleshooting.

You can discover the node using:

```bash
talosctl discover
```

---

## 2. Generate Talos configuration

Talos configuration is generated using a combination of:
- cluster metadata
- secrets
- configuration patches

This is automated in the infrastructure repository using:

```bash
task talos:build
```

Internally, this runs `talosctl gen config` with:
- static networking
- DNS configuration
- Cilium-related settings
- single-node scheduling enabled

Generated files include (not committed to Git):

- `controlplane.yaml`
- `talosconfig`
- `secrets.yaml`

These files are written to:

```text
talos/generated/
```

---

## 3. Static network configuration

Static networking is defined via a Talos patch.

Example (simplified):

```yaml
machine:
  network:
    interfaces:
      - interface: <INTERFACE_NAME>
        dhcp: false
        addresses:
          - <NODE_IP>/<SUBNET_CIDR>
        routes:
          - network: 0.0.0.0/0
            gateway: <GATEWAY_IP>
```

> All IP addresses shown are placeholders.  
> Actual values are defined in the infrastructure repository and `cluster.env`.

---

## 4. Allow scheduling on the control plane

Since this is a **single-node cluster**, workloads must be allowed to run on the control plane.

This is handled via a patch:

```yaml
cluster:
  allowSchedulingOnControlPlanes: true
```

---

## 5. Apply Talos configuration to the node

Once Talos is running (initially via DHCP), apply the generated configuration.

Recommended approach:

```bash
task talos:apply-insecure
```

This is required **only for the first application**, before certificates are established.

Afterwards, Talos will reboot and come up using the **static IP**.

For subsequent changes, use:

```bash
task talos:apply
```

---

## 6. (Optional) Dry-run validation

Before applying changes, configuration can be validated without modifying the node:

```bash
task talos:apply-dry-run
```

This is strongly recommended when making changes to networking or patches.

---

## 7. Bootstrap Kubernetes

Once Talos is running with the static IP, bootstrap Kubernetes:

```bash
FORCE_BOOTSTRAP=1 task talos:bootstrap
```

Bootstrap:
- initializes etcd
- starts the Kubernetes control plane
- makes the API server available

> Bootstrap must be executed **exactly once** per cluster.

---

## 8. Retrieve kubeconfig

After bootstrap, retrieve the Kubernetes kubeconfig:

```bash
task talos:kubeconfig
```

This writes a kubeconfig file to:

```text
talos/generated/kubeconfig
```

You may export it temporarily:

```bash
export KUBECONFIG=talos/generated/kubeconfig
```

Verify access:

```bash
kubectl get nodes
kubectl get pods -A
```

---

## 9. Talos endpoint configuration (optional)

To avoid specifying node and endpoint on every command:

```bash
talosctl config endpoint <NODE_IP>
talosctl config node <NODE_IP>
```

Verify connectivity:

```bash
talosctl version
talosctl health
```
