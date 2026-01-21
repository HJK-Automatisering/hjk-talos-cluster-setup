# 09 – Sealed Secrets (kubeseal)

This section describes how application secrets are handled in Kubernetes in a
secure and GitOps-friendly way using **Sealed Secrets**.

The goals are:
- avoid committing plaintext secrets to Git
- allow secrets to be safely committed
- ensure ArgoCD is the single source of truth

---

## What are Sealed Secrets?

Sealed Secrets is a Kubernetes controller that:
- decrypts **SealedSecret** resources
- creates standard Kubernetes **Secret** objects
- ensures only the *current cluster* can decrypt the secrets

Workflow:
1. A Secret is created locally (plaintext)
2. The Secret is encrypted using `kubeseal`
3. Only the encrypted version is committed to Git
4. The controller automatically decrypts it in the cluster

---

## Prerequisites

This section assumes:
- A running Kubernetes cluster
- ArgoCD installed
- Sealed Secrets controller installed
- `kubectl` and `kubeseal` installed locally

Verify that the controller is running:

```bash
kubectl -n kube-system get pods | grep sealed
kubectl -n kube-system get svc  | grep sealed
```

---

## Directory structure and Git rules

We use the following structure per application:

```text
applications/<app>/
├── local-secrets/        # plaintext (gitignored)
└── templates/
    └── sealed-*.yaml     # encrypted secrets (committed)
```

Plaintext secrets must **only** live in `local-secrets/` and must never be
committed to Git.

---

## Example: LiteLLM

### Secrets used

LiteLLM uses the following secrets:

| Secret name | Purpose |
|------------|---------|
| `litellm-secrets` | Runtime keys for LiteLLM |
| `cloudnative-pg-cluster-litellm` | PostgreSQL credentials |

---

### 1. Create plaintext secrets locally

```bash
mkdir -p applications/litellm/local-secrets
```

#### LiteLLM runtime secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: litellm-secrets
  namespace: litellm
type: Opaque
stringData:
  CA_VLLM_LOCAL_API_KEY: "<generated-value>"
  PROXY_MASTER_KEY: "<generated-value>"
```

#### PostgreSQL credentials

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudnative-pg-cluster-litellm
  namespace: litellm
type: Opaque
stringData:
  username: litellm
  password: "<generated-password>"
```

---

### 2. Seal the secrets using kubeseal

```bash
kubectl create -f applications/litellm/local-secrets/litellm-secrets.yaml \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
  --controller-name sealed-secrets \
  --controller-namespace kube-system \
> applications/litellm/templates/sealed-litellm-secrets.yaml
```

```bash
kubectl create -f applications/litellm/local-secrets/cloudnative-pg-cluster-litellm.yaml \
  --dry-run=client -o yaml \
| kubeseal --format yaml \
  --controller-name sealed-secrets \
  --controller-namespace kube-system \
> applications/litellm/templates/sealed-cloudnative-pg-secret.yaml
```

Only the **sealed** files are committed.

---

### 3. Commit and GitOps flow

```bash
git add applications/litellm/templates/sealed-*.yaml
git commit -m "Add sealed secrets for LiteLLM"
git push
```

After merging, resync the application in ArgoCD:

```bash
task gitops:port-forward
```

Open `http://localhost:8080` and resync the `litellm` application.

---

## Verification

```bash
kubectl -n litellm get sealedsecrets
kubectl -n litellm get secrets
kubectl -n litellm get pods
```

The application should be **Synced** and **Healthy** in ArgoCD.

---

## Common issue: missing ConfigMap

If a pod fails with:

```text
FailedMount: configmap "<name>" not found
```

This usually means:
- The ConfigMap was not applied by ArgoCD
- A Job or Pod started before the ConfigMap existed

Resolution:
1. Ensure the ConfigMap exists in the chart `templates/` directory
2. Commit the change
3. Resync the application in ArgoCD

---

## Summary

- Plaintext secrets are never committed to Git
- Sealed Secrets are cluster-specific
- ArgoCD is always the source of truth
- One PostgreSQL cluster per application
- The same pattern is used for all applications (LiteLLM, OpenWebUI, Authentik)
