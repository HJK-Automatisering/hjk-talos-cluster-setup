---
layout: default
title: Installation af Talos og bootstrap af cluster
nav_order: 3
---

# Installation af Talos og bootstrap af cluster

Dette afsnit beskriver den komplette installationsproces for Talos Linux på en single-node Kubernetes-klynge. Processen baserer sig på de Bash-scripts og YAML-konfigurationer, der findes i projektmappen **Talos/Bash/**.

---

# 1. Boot fra Talos ISO

1. Montér Talos ISO via iDRAC:
   - Download fra: https://github.com/siderolabs/talos/releases
   - Brug `metal-amd64.iso`

2. Boot serveren i *maintenance mode*.

3. Notér den midlertidige DHCP-adresse, som Talos får ved første boot.

> Eksempel: `192.168.50.58` (bruges kun første gang)

---

# 2. Generér clusterkonfiguration via build.sh

Projectet indeholder et script, der samler alle patches og basefiler:

```bash
talosctl gen config --output-types controlplane \
 --with-secrets secrets.yaml \
 --config-patch @nameservers.yaml \
 --config-patch @network.yaml \
 --config-patch @cilium.yaml \
 --config-patch @allow-scheduling-on-control-planes.yaml \
 --config-patch @rook-ceph-security-exemption.yaml \
 cluster2 https://192.168.50.10:6443 --force

```

Dette script genererer:

- `controlplane.yaml`
- `network.yaml`
- `nameservers.yaml`
- Andre nødvendige filer til installation

Patches håndterer bl.a.:

- Statisk IP-konfiguration
- Node-roller
- DNS
- Tillad scheduling på control-plane

---

# 3. Statisk IP konfiguration (network.yaml)

Eksempel fra *network.yaml*:

```yaml
machine:
  network:
    interfaces:
      - interface: eth0
        dhcp: false
        addresses:
          - 192.168.50.10/24
        routes:
          - network: 0.0.0.0/0
            gateway: 192.168.50.1
```

Tilpas IP-adresse, subnet og gateway efter jeres miljø.

---

# 4. Tillad pods på control-plane (allow-scheduling-on-control-planes.yaml)

Single-node klynger kræver scheduling på control-plane:

```yaml
cluster:
  allowSchedulingOnControlPlanes: true
```

---

# 5. Anvend konfiguration på serveren (apply.sh)

Når Talos kører på sin midlertidige DHCP-adresse:

```bash
talosctl --talosconfig talosconfig \
 --endpoints 192.168.50.10 \
 --nodes 192.168.50.10 \
 apply-config -f controlplane.yaml
```

Eller insecure version (nødvendig første gang):

```bash
talosctl --talosconfig talosconfig \
 --endpoints 192.168.50.10\
 --nodes 192.168.50.10\
 apply-config -f controlplane.yaml --insecure
```

Serveren genstarter herefter med den nye statiske IP.

---

# 6. (Valgfrit) Test konfiguration (dry run)

```bash
talosctl --talosconfig talosconfig \
 --endpoints 192.168.50.10 \
 --nodes 192.168.50.10 \
 apply-config -f controlplane.yaml --dry-run

```

---

# 7. Bootstrap Kubernetes (bootstrap.sh)

Når serveren kører Talos med statisk IP:

```bash
talosctl --talosconfig talosconfig \
 --endpoints 192.168.50.10 \
 --nodes 192.168.50.10 \
 bootstrap

```

Når bootstrappen er færdig:

- etcd er oppe
- kube-apiserver kører
- Kubernetes er initialiseret

---

# 8. Hent kubeconfig (kubeconfig.sh)

For at kunne administrere Kubernetes:

```bash
talosctl --talosconfig talosconfig \
 --endpoints 192.168.50.10 \
 --nodes 192.168.50.10 \
 kubeconfig

```

Eller manuelt:

```bash
talosctl kubeconfig --nodes 192.168.50.10 --force
```

Dette skriver kubeconfig til:

```
~/.kube/config
```

Test:

```bash
kubectl get nodes
kubectl get pods -A
```

---

# 9. Talosctl endpoint og node konfiguration

For at undgå at skrive IP-adressen hver gang:

```bash
talosctl config endpoint 192.168.50.10
talosctl config node 192.168.50.10
```

Test:

```bash
talosctl version
talosctl health
```
