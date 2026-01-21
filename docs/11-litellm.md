---
layout: default
title: LiteLLM
nav_order: 11
---

# LiteLLM (Proxy + Database)

This section documents how we deploy **LiteLLM** as an OpenAI-compatible proxy in front of our model backends (for example **vLLM**), including a **CloudNativePG** Postgres database for persistence.

LiteLLM is deployed via Helm and managed via **ArgoCD** (GitOps).

---

## Project structure

The LiteLLM application lives under:

`applications/litellm/`

```text
applications/litellm/
├── Chart.yaml
├── cloudnative-pg-values.yaml
├── litellm-values.yaml
├── local-secrets/
│   ├── cloudnative-pg-cluster-litellm.yaml
│   ├── litellm-secret.yaml
│   └── litellm-secrets.yaml
└── templates/
    ├── database.yaml
    ├── message-trimming-config.yaml
    ├── sealed-cloudnative-pg-secret.yaml
    └── sealed-litellm-secrets.yaml
```

---

## Prerequisites

- Kubernetes cluster reachable and healthy
- Namespace `litellm` exists
- **Sealed Secrets** controller installed (see **09 – Sealed Secrets**)
- **vLLM** deployed if used as a backend (see **10 – vLLM**)
- CloudNativePG operator installed in the cluster

---

## Helm chart overview

LiteLLM is deployed using the `litellm` chart with the database-enabled image:

- Image: `ghcr.io/berriai/litellm-database`
- Tag example: `main-v1.80.0-stable`

The database-enabled image is required because LiteLLM performs Prisma migrations on startup.

---

## Secrets

LiteLLM relies on Kubernetes secrets that are created locally, sealed, and committed.

### Proxy secrets (`litellm-secrets`)

Local plaintext secret:

`applications/litellm/local-secrets/litellm-secrets.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: litellm-secrets
  namespace: litellm
type: Opaque
stringData:
  CA_VLLM_LOCAL_API_KEY: "SECRET"
  PROXY_MASTER_KEY: "SECRET"
```

### Important: vLLM API key coupling

`CA_VLLM_LOCAL_API_KEY` **must match** the API key configured for vLLM.

If the keys differ, LiteLLM will fail authentication when forwarding requests to vLLM.

Whenever the vLLM API key is rotated, the LiteLLM secret must be updated and resealed.

---

## Database configuration (CloudNativePG)

Database credentials are provided via a sealed secret:

- Local: `local-secrets/cloudnative-pg-cluster-litellm.yaml`
- Sealed: `templates/sealed-cloudnative-pg-secret.yaml`

Referenced in `litellm-values.yaml`:

```yaml
db:
  endpoint: cloudnative-pg-cluster-rw
  database: litellm
  url: postgresql://$(DATABASE_USERNAME):$(DATABASE_PASSWORD)@$(DATABASE_HOST)/$(DATABASE_NAME)
```

---

## Guardrails: message trimming

LiteLLM uses a message trimming guardrail to avoid oversized prompts.

The guardrail script is mounted from a ConfigMap:

- ConfigMap: `templates/message-trimming-config.yaml`
- Mount path: `/app/custom_guardrails/message_overflow.py`

Enabled per model under `proxy_config.model_list`.

---

## CloudNativePG backups (important)

Backups are **disabled by default**:

```yaml
backups:
  enabled: false
```

Backups must be explicitly configured before enabling them.  
If backups are enabled without valid credentials (for example missing `hetzner-s3-backup-credentials`), the operator will continuously log errors and may fill available storage.

---

## Deployment via ArgoCD

Recommended order:

1. Deploy Sealed Secrets controller
2. Deploy vLLM (if used)
3. Create and seal LiteLLM secrets
4. Commit values and templates
5. Sync the ArgoCD application

---

## Verification

```bash
kubectl get pods -n litellm
kubectl get svc -n litellm
kubectl get secret -n litellm
```

---

## Summary

LiteLLM provides a centralized proxy layer for model access.

Key requirements:
- Sealed Secrets for credentials
- Matching API keys between LiteLLM and vLLM
- CloudNativePG backups disabled unless explicitly configured
- ArgoCD as the single source of truth
