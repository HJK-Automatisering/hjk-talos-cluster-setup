---
layout: default
title: Upgrading Kubernetes Applications
nav_order: 13
---

# Upgrading Kubernetes Applications

This section describes how to upgrade applications and platform components in the Kubernetes cluster.

The cluster is managed using a **GitOps-style** workflow:

- The **`vendor/`** folder is a Git submodule pointing to the upstream Helm deployments repository.
- The **`overrides/`** folder contains cluster-specific Helm values and templates.
- Upgrades are performed by **pinning `vendor/` to a new tag**, then adjusting overrides if needed.

> **Important**  
> Always review upstream release notes and test upgrades in a safe window.  
> Some components (especially storage and networking) can be disruptive.

---

## Concepts

### Vendor submodule

The repository includes `vendor/` as a Git submodule:

```ini
[submodule "vendor"]
  path = vendor
  url = https://github.com/AarhusAI/helm-deployments
```

This allows the cluster to **track upstream chart changes by tag**.

### Overrides

Overrides are placed under:

```text
overrides/<application>/values.yaml
overrides/<application>/templates/*.yaml
```

These files contain **local, environment-specific configuration**, such as:

- hostnames / ingress rules
- storage settings
- OIDC / authentication integration
- resource limits
- sealed secrets templates

When upgrading, you should assume:

- upstream defaults may change
- value keys may be renamed/removed
- new required values may appear

---

## Upgrade workflow (recommended)

### 1) Pick the target version

Determine the target upstream tag you want to upgrade to in the vendor repo.

In general, prefer:

- the latest stable tag
- or a tag that contains a specific required fix

---

### 2) Update the vendor submodule to the new tag

From the root of the cluster repo:

```bash
cd vendor
git fetch --tags
git checkout tags/<TAG>
cd ..
git add vendor
git commit -m "Updated vendor submodule to version <TAG>"
```

> If you are switching between branches/tags often, make sure the submodule is properly initialized:

```bash
git submodule update --init --recursive
```

---

### 3) Review differences between vendor versions

Before applying anything to the cluster, inspect what changed upstream:

```bash
cd vendor
git diff tags/<OLD_TAG> tags/<NEW_TAG>
```

Focus on:

- chart template changes
- `values.yaml` schema changes
- image tag updates
- new CRDs or removed APIs

If the vendor repo includes a changelog, review that as well.

---

### 4) Update overrides (if required)

After updating `vendor/`, validate each overridden chart:

1. Compare your overrides with upstream `values.yaml` changes.
2. Check for renamed keys, removed options, and new required values.
3. If templates are overridden in `overrides/<app>/templates/`, verify that:
   - labels/selectors still match
   - API versions are still supported
   - the upstream chart did not replace the template you override

Common signs you need to adjust overrides:

- Helm render errors
- Argo CD diff loops
- Pods stuck in `CrashLoopBackOff` due to missing env vars
- CRDs out of date

---

### 5) Apply the upgrade

How upgrades are applied depends on whether the component is:

- **managed through Argo CD / applications** (typical for most apps)
- or **applied via scripts** (typical for cluster-level components)

#### A) Argo CD-managed applications

If the component is deployed via Argo CD, the upgrade is typically triggered by:

1. Merging the change (vendor tag + overrides) to the tracked branch
2. Letting Argo CD sync the application

Recommended approach:

- sync one application at a time
- verify it becomes **Healthy** and **Synced**

> If you have auto-sync enabled, still watch the rollout closely during upgrades.

#### B) Script-driven platform components

Some core components are upgraded with scripts under:

```text
scripts/k8s/
```

Examples:

- `scripts/k8s/argo-cd-upgrade.sh`
- `scripts/k8s/cillium-upgrade.sh`
- `scripts/k8s/metallb-upgrade.sh`
- `scripts/k8s/rook-ceph-upgrade.sh`
- `scripts/k8s/rook-ceph-cluster-upgrade.sh`

Run the relevant script from the repo root (example):

```bash
./scripts/k8s/argo-cd-upgrade.sh
```

> The scripts may assume `KUBECONFIG` is set, and that you have cluster access from WSL2.

---

### 6) Verify health

Always verify both **Kubernetes state** and **application behavior**.

Minimum checks:

```bash
kubectl get nodes
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp | tail -n 50
```

For the upgraded namespace (example):

```bash
kubectl -n <NAMESPACE> get pods
kubectl -n <NAMESPACE> describe pod <POD>
kubectl -n <NAMESPACE> logs <POD> --tail=200
```

If Argo CD is used:

- confirm the app is **Healthy** and **Synced**
- review diffs for unexpected drift

---

## Special considerations (by component type)

### Networking (Cilium, MetalLB)

Networking upgrades can interrupt:

- cluster DNS
- pod-to-pod traffic
- external ingress / LoadBalancer IPs

Recommendations:

- upgrade during a maintenance window
- keep a console open to verify cluster access is not lost
- validate:
  - Cilium pods and agents are ready
  - service IP announcements (MetalLB)
  - ingress routes

### Storage (Rook Ceph)

Storage upgrades require extra care.

Recommendations:

- ensure the cluster is healthy before upgrading
- verify OSDs and monitors are OK
- avoid concurrent disruptive changes (disk cleaning, node reboot, etc.)

If the upgrade includes CRD changes, apply them in the order defined by the upgrade scripts and upstream documentation.

### GPU-related components (NVIDIA device plugin)

GPU upgrades can affect:

- node device discovery
- runtime classes
- scheduling of GPU workloads

Validate:

- `RuntimeClass` exists (`runtimeclass-nvidia`)
- the device plugin daemonset is ready
- GPU workloads can still schedule

---

## Troubleshooting upgrades

### Vendor tag updated, but nothing changes in the cluster

Typical causes:

- Argo CD is pinned to a different branch/commit
- application chart path did not change
- auto-sync is disabled and a manual sync is required

### Render / apply errors after upgrade

Typical causes:

- values schema changed upstream
- deprecated API versions removed
- CRDs not upgraded before the controller

Actions:

1. inspect the upstream `values.yaml`
2. compare your overrides for outdated keys
3. re-run the upgrade in the correct order (especially CRDs)

### Rollback

If an upgrade fails, the simplest rollback path is usually:

1. revert the vendor tag commit
2. revert any override changes
3. let Argo CD re-sync, or re-run the relevant upgrade script

Rollback example:

```bash
git revert <COMMIT_SHA>
git push
```

> Storage and database components may require a more careful rollback plan. Avoid rolling back stateful components without reviewing upstream guidance.
