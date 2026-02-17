---
layout: default
title: LiteLLM
nav_order: 2
parent: Applications
---

# LiteLLM (Proxy + Database)

This chapter documents how we deploy **LiteLLM** as an OpenAI-compatible proxy in front of model backends (for example **vLLM**), including a **CloudNativePG** Postgres database for persistence.

LiteLLM is deployed via Helm and managed via **Argo CD** (GitOps).

---

## Vendor vs overrides (GitOps model)

This documentation follows the repository structure used by the cluster IaC repository:

- **Vendor** contains the upstream application Helm chart (as a submodule)
- **Overrides** contains municipality / cluster-specific configuration (values + templates)

For LiteLLM this means:

- Vendor chart: `vendor/applications/litellm`
- Overrides: `overrides/litellm`

> The LiteLLM application is deployed via **Argo CD** (app-of-apps) and should be treated as GitOps-managed.
> Manual `helm install` is not recommended for day-to-day operation.

---

## Project structure

### Vendor chart (upstream)

`vendor/applications/litellm/`

```text
vendor/applications/litellm/
├── Chart.yaml
├── cloudnative-pg-values.yaml
├── litellm-values.yaml
└── templates
    ├── database.yaml
    ├── message-trimming-config.yaml
    └── migration-job.yaml
```

### Overrides (cluster-specific)

`overrides/litellm/`

```text
overrides/litellm/
├── local-secrets
│   └── cloudnative-pg-cluster-litellm.yaml
├── templates
│   ├── sealed-cloudnative-pg-secret.yaml
│   └── sealed-litellm-secrets.yaml
└── values.yaml
```

---

## Prerequisites

- Kubernetes cluster reachable and healthy
- Namespace `litellm` exists (or is created by Argo CD sync)
- **Sealed Secrets** controller installed (see **09 – Sealed Secrets**)
- **vLLM** deployed if used as a backend (see **10 – vLLM**)
- CloudNativePG operator installed in the cluster

---

## Helm chart overview (vendor)

LiteLLM is deployed using the vendor Helm chart, which includes:

- a LiteLLM deployment (database-enabled configuration)
- CloudNativePG resources (database cluster + migrations)
- optional guardrails configuration (for example message trimming)

The database-enabled image is required because LiteLLM performs Prisma migrations on startup.

---

## Cluster configuration (overrides)

The main cluster-specific configuration is stored in:

`overrides/litellm/values.yaml`

Typical configuration includes:

- Ingress annotations (Traefik + TLS entrypoints)
- `proxy_config.model_list` defining available models and backends
- Guardrails enabled per model (for example `message_trimming`)
- Database connection settings referencing a Kubernetes Secret

Example pattern:

```yaml
litellm:
  proxy_config:
    model_list:
      - model_name: <MODEL_ALIAS>
        litellm_params:
          model: openai/<provider>/<model>
          api_key: os.environ/<ENV_VAR_NAME>
          api_base: http://<service>.<namespace>.svc.cluster.local/v1
        guardrails:
          - message_trimming

  db:
    endpoint: cloudnative-pg-cluster-rw
    database: litellm
    url: postgresql://$(DATABASE_USERNAME):$(DATABASE_PASSWORD)@$(DATABASE_HOST)/$(DATABASE_NAME)
    secret:
      name: cloudnative-pg-cluster-litellm
      usernameKey: username
      passwordKey: password
```

---

## Secrets

LiteLLM relies on Kubernetes secrets that are created locally, sealed, and committed.

### 1) Proxy secrets (LiteLLM runtime configuration)

The proxy secrets are committed as a SealedSecret:

- Sealed: `overrides/litellm/templates/sealed-litellm-secrets.yaml`

This typically includes values such as:

- `CA_VLLM_LOCAL_API_KEY` (used to authenticate to vLLM)
- `PROXY_MASTER_KEY` (LiteLLM master key)

> Never commit plaintext tokens into Git.
> Use Sealed Secrets (kubeseal) and keep plaintext values local only.

#### Important: vLLM API key coupling

`CA_VLLM_LOCAL_API_KEY` **must match** the API key configured for vLLM.

If the keys differ, LiteLLM will fail authentication when forwarding requests to vLLM.

Whenever the vLLM API key is rotated, the LiteLLM secret must be updated and re-sealed.

---

### 2) Database credentials (CloudNativePG)

Database credentials are provided via a sealed secret:

- Local (plaintext, not committed): `overrides/litellm/local-secrets/cloudnative-pg-cluster-litellm.yaml`
- Sealed (committed): `overrides/litellm/templates/sealed-cloudnative-pg-secret.yaml`

The LiteLLM values reference the created Secret under `litellm.db.secret`.

---

## Guardrails: message trimming

LiteLLM can use a message trimming guardrail to avoid oversized prompts.

The guardrail script is mounted from a ConfigMap provided by the vendor chart:

- ConfigMap template: `vendor/applications/litellm/templates/message-trimming-config.yaml`
- Mount path example: `/app/custom_guardrails/message_overflow.py`

Enabled per model under `proxy_config.model_list.guardrails`.

---

## Deployment via Argo CD

Recommended order:

1. Deploy Sealed Secrets controller
2. Deploy vLLM (if used as a backend)
3. Create and seal LiteLLM secrets (proxy + database credentials)
4. Commit overrides (`values.yaml` + templates)
5. Sync the Argo CD application

---

## Verification

```bash
kubectl get pods -n litellm
kubectl get svc -n litellm
kubectl get secret -n litellm
```

If the database migration job is used, also check:

```bash
kubectl get jobs -n litellm
kubectl logs -n litellm job/<migration-job-name>
```

---

## Summary

LiteLLM provides a centralized proxy layer for model access.

Key requirements:

- Sealed Secrets for credentials
- Matching API keys between LiteLLM and vLLM (when vLLM is a backend)
- CloudNativePG operator installed and healthy
- Argo CD as the single source of truth
