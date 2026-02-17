---
title: 07 – Argo CD (GitOps)
nav_order: 1
parent: Platform Management
---

# Argo CD (GitOps)

This cluster uses **Argo CD** to deploy and continuously reconcile Kubernetes applications from Git (GitOps).

Argo CD is installed from the **vendor** submodule, and then bootstraps the rest of the cluster via an **app-of-apps** chart (**argo-cd-resources**).

This chapter explains:

- Vendor vs overrides model
- How Argo CD is installed in this repository
- How `argo-cd-resources` works (Projects + Applications)
- How to handle **private Git repositories** using GitHub credentials (PAT)
- Operational commands using the `Taskfile.yml`

---

## Vendor vs Overrides

### Vendor (`vendor/`)
`vendor/` is a git submodule pointing to a shared Helm deployments repository:

```ini
[submodule "vendor"]
  path = vendor
  url = https://github.com/AarhusAI/helm-deployments
```

Vendor contains upstream Helm charts and templates, including:

- `vendor/applications/argo-cd`
- `vendor/applications/argo-cd-resources`

### Overrides (`overrides/`)
`overrides/` contains cluster-specific configuration such as:

- Ingress, domain, TLS
- OIDC and RBAC configuration
- Value overrides for apps
- Additional templates (for example SealedSecrets)

This separation makes it easy to:

- update upstream versions via vendor tags
- keep municipality-specific changes isolated in overrides

---

## Components

### 1) Argo CD (vendor chart)

Argo CD is defined as a vendor chart:

- `vendor/applications/argo-cd/Chart.yaml`

Cluster-specific settings are applied via:

- `overrides/argo-cd/values.yaml`
- `overrides/argo-cd/templates/*` (optional, e.g. sealed OIDC secret)

Typical override examples:

- OIDC configuration (issuer, clientID, secret reference)
- RBAC policies
- Ingress and domain
- Custom Argo CD image (optional)

---

### 2) Argo CD Resources (app-of-apps)

Argo CD Resources is the bootstrap chart that creates:

- **AppProjects** (security boundaries)
- **Applications** (one per app)

Vendor location:

- `vendor/applications/argo-cd-resources`

The chart generates `Application` objects from a list in `values.yaml`:

- `templates/applications.yaml` loops over `Values.apps`
- `templates/projects/*.yaml` define AppProjects

This enables a scalable **app-of-apps** setup: one root chart creates all app definitions, and Argo CD reconciles them.

---

## How this repo bootstraps GitOps

This repository uses `Taskfile.yml` and two scripts:

- `scripts/k8s/argo-cd-upgrade.sh`
- `scripts/k8s/argo-cd-resources-apply.sh`

### Install/upgrade Argo CD

```bash
task gitops:argo-cd
```

What it does (high-level):

1. Adds the Argo Helm repo and builds chart dependencies
2. Creates the `argo-cd` namespace if missing
3. Renders vendor chart + applies overrides:
   - `vendor/applications/argo-cd`
   - `overrides/argo-cd/values.yaml`
4. Port-forwards the Argo CD server temporarily
5. Logs into the Argo CD CLI
6. Adds/updates the Git repository credentials in Argo CD (see “Private repo access”)

### Apply Argo CD Resources (root app-of-apps)

```bash
task gitops:argo-cd-resources
```

What it does:

- Renders and applies:
  - `vendor/applications/argo-cd-resources`
  - `overrides/argo-cd-resources/values.yaml`

This creates the AppProjects and Applications, which then causes Argo CD to deploy everything else.

### Full GitOps bootstrap

```bash
task gitops:bootstrap
```

---

## Private repository access (GitHub PAT)

If the GitOps repository is **private**, Argo CD must authenticate to fetch manifests and Helm definitions.

This repo solves that by reading credentials from `cluster.env` and registering the repo with Argo CD:

- `ARGO_CD_REPO_USERNAME`
- `ARGO_CD_REPO_PASSWORD` (commonly a GitHub PAT, e.g. `github_pat_...`)

The `argo-cd-upgrade.sh` script registers the repository using the Argo CD CLI:

```bash
argocd repo add https://github.com/<org>/<repo>.git \
  --upsert \
  --username ${ARGO_CD_REPO_USERNAME} \
  --password ${ARGO_CD_REPO_PASSWORD}
```

### Recommended credential approach

Use a dedicated GitHub user or service account and create a PAT with **read access** to the private repo.

> Never commit credentials into Git.  
> Keep them only in `cluster.env` (which should not be committed).

---

## Cluster-specific repoUrl and overrides

In vendor, `argo-cd-resources` usually points to the vendor repository.

In this cluster, we override it so Argo CD reads from the **cluster repository** and uses paths that include vendor + overrides.

Typical pattern in `overrides/argo-cd-resources/values.yaml`:

- `repoUrl` points to the GitOps repository (this repository)
- each app uses a `path`:
  - vendor application path (via vendor submodule)
  - override templates path (to apply extra sealed secrets / config)
- each app references override values via `valueFiles`

This is why Argo CD must be able to read the GitOps repository.

---

## Verify GitOps

### Check Argo CD pods

```bash
kubectl -n argo-cd get pods
```

### Check Applications

```bash
kubectl -n argo-cd get applications
```

### Port-forward the UI locally

```bash
task gitops:port-forward
```

Then open:

- http://localhost:8080

> Ingress domain, TLS, and external URL are cluster-specific and configured in `overrides/argo-cd/values.yaml`.

---

## Troubleshooting

### Repo authentication errors

Symptoms:

- Applications stuck in “Missing” / “Unknown”
- UI shows repository access errors

Fix checklist:

- Confirm `ARGO_CD_REPO_USERNAME` and `ARGO_CD_REPO_PASSWORD` are set in `cluster.env`
- Confirm the PAT has access to the private repo
- Re-run:

```bash
task gitops:argo-cd
```

### Sync issues

If an Application is OutOfSync:

- Inspect differences in Argo CD UI (or `kubectl describe`)
- Confirm CRDs exist before CRs (operators often require ordering)
- Check namespace creation (CreateNamespace=true is enabled, but RBAC/PSA can still block)
