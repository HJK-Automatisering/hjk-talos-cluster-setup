---
layout: default
title: Accessing the Cluster from WSL2
nav_order: 8
---

# Accessing the Cluster from WSL2

This guide explains how to access and operate the Talos and Kubernetes cluster from a **Windows 11 workstation using Ubuntu WSL2**.

WSL2 is used for:
- Managing the server using `talosctl`
- Administering Kubernetes using `kubectl`
- Deploying platform components using Helm (Cilium, Rook Ceph)

Connectivity is established directly to the server’s **static IP address** (for example `192.168.50.10`).

---

## 1. Prerequisites

Before continuing, ensure the following tools are installed inside WSL2:

```bash
talosctl version
kubectl version --client
helm version
cilium version
```

Additionally:
- You must be able to **ping the server IP**
- Firewalls must allow traffic from the WSL2 environment

Test connectivity:

```bash
ping 192.168.50.10
```

---

## 2. Configuring talosctl

Once Talos is installed on the server, WSL2 must be configured to communicate with the node.

### 2.1 Set Talos endpoint

```bash
talosctl config endpoint 192.168.50.10
```

### 2.2 Set Talos node

```bash
talosctl config node 192.168.50.10
```

### 2.3 Verify connectivity

```bash
talosctl version
talosctl health
```

If this fails:
- Verify network routing
- Check firewall rules
- Ensure the server is running Talos (not still booted from the ISO in maintenance mode)

---

## 3. Retrieve kubeconfig (Talos → WSL2)

After the Kubernetes bootstrap has completed:

```bash
talosctl kubeconfig --nodes 192.168.50.10 --force
```

This writes the kubeconfig file to:

```text
~/.kube/config
```

Verify access:

```bash
kubectl get nodes
kubectl get pods -A
```

If resources are returned, Kubernetes access is working correctly.

---

## 4. WSL2 Networking Notes

WSL2 runs as a **lightweight virtual machine**, and its IP address changes on every restart.

Check the WSL2 IP address:

```bash
ip addr show eth0
```

Example output:

```text
inet 172.25.89.14/20
```

### If WSL2 cannot reach the cluster IP

1. Ensure the Windows host itself can reach the server subnet
2. Verify that network firewalls allow traffic from WSL2
3. In very restricted networks, static routes may be required

In most enterprise networks, no additional configuration is needed beyond basic firewall access.

---

## 5. Optional: Use kubeconfig from Windows tools

If you also want to use Kubernetes tools directly on Windows:

```powershell
mkdir ~/.kube
wsl cp ~/.kube/config ~/.kube/config
```

This copies the kubeconfig from WSL2 into the Windows user profile.

---

## 6. Using Helm from WSL2

Once `kubectl` access is working, Helm will work automatically:

```bash
helm ls -A
```

Helm is used to deploy and manage:
- Cilium (CNI)
- Rook Ceph (storage)

---

## 7. Troubleshooting

### WSL2 → Talos connectivity issues

```bash
talosctl version
```

Error: `connection refused`  
→ Talos may not be running (still booted from ISO)  
→ Firewall rules may be blocking access

### kubeconfig does not work

```bash
kubectl get nodes
```

Error: `context deadline exceeded`  
→ Kubernetes API server is not responding  
→ Cilium may not be running (check `kubectl -n kube-system get pods`)

### Talos health check

```bash
talosctl health
```

Returns detailed node health information.

---

## 8. Next Steps

Once WSL2 access is confirmed, you can proceed with platform operations:

- Install Cilium
- Install Rook Ceph

Continue with the next documentation section for platform deployment.
