---
layout: default
title: Open WebUI
nav_order: 3
parent: Applications
---

# Open WebUI (Chat UI + RAG)

This chapter documents how we deploy **Open WebUI** as the user-facing chat interface, integrated with:

- **LiteLLM** as the OpenAI-compatible proxy
- **vLLM** for local inference / embeddings
- **CloudNativePG** for persistence
- **Qdrant** as the vector database
- **Redis** for websockets/session middleware (multi-replica)
- **S3-compatible storage** (Rook Ceph RGW) for persistence (optional)

Open WebUI is deployed via Helm and managed via **Argo CD** (GitOps).

---

## Vendor vs overrides (GitOps model)

This documentation follows the repository structure used by the cluster IaC repository:

- **Vendor** contains the upstream application Helm chart (as a submodule)
- **Overrides** contains municipality / cluster-specific configuration (values + templates)

For Open WebUI this means:

- Vendor chart: `vendor/applications/openwebui`
- Overrides: `overrides/openwebui`

> Open WebUI is deployed via **Argo CD** (app-of-apps) and should be treated as GitOps-managed.
> Manual `helm install` is not recommended for day-to-day operation.

---

## Project structure

### Vendor chart (upstream)

`vendor/applications/openwebui/`

```text
vendor/applications/openwebui/
├── Chart.yaml
├── cloudnative-pg-values.yaml
├── qdrant-values.yaml
├── templates
│   ├── database.yaml
│   ├── s3-claim.yaml
│   └── was-middleware.yaml
└── values.yaml
```

### Overrides (cluster-specific)

`overrides/openwebui/`

```text
overrides/openwebui/
├── templates
│   ├── sealed-cloudnative-pg-cluster-openwebui-secret.yaml
│   ├── sealed-openwebui-oidc-secret.yaml
│   └── sealed-openwebui-secrets.yaml
└── values.yaml
```

---

## How values are applied (important)

The Open WebUI Argo CD application typically includes **multiple value files**, for example:

- vendor defaults: `vendor/applications/openwebui/values.yaml`
- vendor component configs:
  - `vendor/applications/openwebui/cloudnative-pg-values.yaml`
  - `vendor/applications/openwebui/qdrant-values.yaml`
- cluster overrides:
  - `overrides/openwebui/values.yaml`

### Helm merge behavior (lists)

Helm merges YAML maps, but **lists are replaced** (not merged).

This matters for Open WebUI because the configuration uses lists like:

- `open-webui.extraEnvVars`

If you override `extraEnvVars`, you typically need to include the full list you want,
not only the new entries.

---

## Prerequisites

- Kubernetes cluster reachable and healthy
- Namespace `openwebui` exists (or is created by Argo CD sync)
- **Sealed Secrets** controller installed (see **09 – Sealed Secrets**)
- **LiteLLM** deployed (see **11 – LiteLLM**)
- **vLLM** deployed if used for local inference/RAG embeddings (see **10 – vLLM**)
- CloudNativePG operator installed
- Ingress controller (Traefik or equivalent) available
- (Optional) S3-compatible object storage if enabling S3 persistence

---

## Ingress and host configuration (portable)

This documentation is intended to be municipality-agnostic.

Replace the following placeholders in `overrides/openwebui/values.yaml`:

- `WEBUI_HOST` (e.g. `ai.example.dk`)
- `TLS_SECRET_NAME` (e.g. `wildcard-tls`, or a municipality-specific secret)

Example pattern:

```yaml
open-webui:
  ingress:
    annotations:
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
    host: WEBUI_HOST
    tls:
      - hosts:
          - WEBUI_HOST
        secretName: TLS_SECRET_NAME
```

Also update environment variables that must match the domain:

```yaml
- name: WEBUI_URL
  value: "https://WEBUI_HOST"
- name: CORS_ALLOW_ORIGIN
  value: "https://WEBUI_HOST;http://WEBUI_HOST"
```

---

## SSO / OIDC

Open WebUI supports OIDC login.

The OIDC client secret is stored in a Kubernetes Secret created from a SealedSecret template:

- `overrides/openwebui/templates/sealed-openwebui-oidc-secret.yaml`

Example configuration pattern:

```yaml
open-webui:
  sso:
    enabled: true
    oidc:
      enabled: true
      clientId: "<OIDC_CLIENT_ID>"
      clientExistingSecret: "openwebui-oidc-secret"
      clientExistingSecretKey: "OPENWEBUI_OAUTH_CLIENT_SECRET"
      providerUrl: "https://<OIDC_PROVIDER>/.well-known/openid-configuration"
      providerName: "<SSO_DISPLAY_NAME>"
      scopes: "openid profile email"
```

> Keep hostnames and issuer URLs generic in committed documentation.
> Use placeholders and document what must be changed per municipality.

---

## Custom image (certificates for internal IdP)

Some environments require Open WebUI to trust a municipal/internal CA (for example when using an internal IdP).
In that case, Open WebUI may be deployed with a **custom image** that includes the required CA certificates.

In `overrides/openwebui/values.yaml`, this typically looks like:

```yaml
open-webui:
  image:
    repository: ghcr.io/<org>/<repo>/open-webui
    tag: "latest"
    pullPolicy: Always
```

---

## Core routing: Open WebUI → LiteLLM

Open WebUI is configured to use LiteLLM as its OpenAI API backend.

In this setup, the `OPENAI_API_KEY` sent by Open WebUI must match what LiteLLM expects
(often the LiteLLM `PROXY_MASTER_KEY`, unless configured differently).

`OPENAI_API_KEY` is typically read from a secret:

```yaml
- name: OPENAI_API_KEY
  valueFrom:
    secretKeyRef:
      name: openwebui-secrets
      key: OPENAI_API_KEY
```

---

## RAG configuration: embeddings and vector store

Open WebUI can use:

- **Qdrant** as vector database
- **vLLM** for embeddings (or another OpenAI-compatible backend)

Qdrant example pattern:

```yaml
- name: VECTOR_DB
  value: "qdrant"
- name: QDRANT_URI
  value: "http://openwebui-qdrant:6334"
- name: QDRANT_PREFER_GRPC
  value: "True"
```

Embeddings via vLLM example pattern:

```yaml
- name: RAG_OPENAI_API_BASE_URL
  value: "http://vllm-router-service.vllm.svc.cluster.local/v1"
- name: RAG_OPENAI_API_KEY
  valueFrom:
    secretKeyRef:
      name: openwebui-secrets
      key: RAG_OPENAI_API_KEY
- name: RAG_EMBEDDING_MODEL
  value: "intfloat/multilingual-e5-large"
```

### Important: API key coupling

- `OPENAI_API_KEY` must match what **LiteLLM** expects
- `RAG_OPENAI_API_KEY` must match the **vLLM** API key (when vLLM is used for embeddings)

If keys differ, you will see authentication errors when Open WebUI calls the upstream services.

---

## Database: CloudNativePG

Open WebUI uses Postgres via CloudNativePG and typically sets `DATABASE_URL` from secrets:

```yaml
- name: POSTGRES_USER
  valueFrom:
    secretKeyRef:
      name: cloudnative-pg-cluster-openwebui
      key: username
- name: POSTGRES_PASSWORD
  valueFrom:
    secretKeyRef:
      name: cloudnative-pg-cluster-openwebui
      key: password
- name: DATABASE_URL
  value: "postgresql://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@cloudnative-pg-cluster-rw:5432/openwebui"
```

The database credentials secret is created from a SealedSecret template:

- `overrides/openwebui/templates/sealed-cloudnative-pg-cluster-openwebui-secret.yaml`

### Backups (important)

CloudNativePG backups should remain **disabled** until object storage credentials exist and are tested.

In many setups this is kept disabled:

```yaml
cloudnative-pg:
  backups:
    enabled: false
```

---

## Websocket / multi-replica support (Redis)

If Open WebUI is deployed with `replicaCount > 1`, websocket/session coordination is required.
This setup often enables:

- websocket support
- Redis as websocket manager
- Redis session middleware

Example env var patterns:

```yaml
- name: ENABLE_WEBSOCKET_SUPPORT
  value: "True"
- name: WEBSOCKET_MANAGER
  value: "redis"
- name: WEBSOCKET_REDIS_URL
  value: "redis://open-webui-redis:6379/0"
- name: ENABLE_STAR_SESSIONS_MIDDLEWARE
  value: "True"
```

---

## Content extraction (doc-ingestion + Tika)

If external document ingestion is enabled, Open WebUI may be configured to use:

- `doc-ingestion` for document loading
- `tika` for content extraction

Example patterns:

```yaml
- name: CONTENT_EXTRACTION_ENGINE
  value: "external"
- name: EXTERNAL_DOCUMENT_LOADER_URL
  value: "http://doc-ingestion.doc-ingestion.svc.cluster.local:8000/api/v1"
- name: EXTERNAL_DOCUMENT_LOADER_API_KEY
  valueFrom:
    secretKeyRef:
      name: openwebui-secrets
      key: EXTERNAL_DOCUMENT_LOADER_API_KEY
- name: TIKA_SERVER_URL
  value: "http://tika.tika.svc.cluster.local:9998"
```

---

## Secrets

Open WebUI relies on Kubernetes secrets that are created locally, sealed, and committed.

Committed SealedSecrets:

- `overrides/openwebui/templates/sealed-openwebui-secrets.yaml`
- `overrides/openwebui/templates/sealed-openwebui-oidc-secret.yaml`
- `overrides/openwebui/templates/sealed-cloudnative-pg-cluster-openwebui-secret.yaml`

### Key coupling (important)

Typical keys that must remain consistent across services:

- `OPENAI_API_KEY` → must match LiteLLM expectations
- `RAG_OPENAI_API_KEY` → must match vLLM API key (if vLLM is used for embeddings)
- `EXTERNAL_DOCUMENT_LOADER_API_KEY` → must match doc-ingestion expectations (if enabled)

If any of these keys are rotated, update and re-seal the Open WebUI secrets accordingly.

---

## Deployment via Argo CD

Recommended order:

1. Ensure Sealed Secrets controller exists (**09 – Sealed Secrets**)
2. Ensure dependencies exist:
   - LiteLLM (**11 – LiteLLM**)
   - vLLM (**10 – vLLM**) if used for RAG/embeddings
   - doc-ingestion and tika if content extraction is enabled
3. Create/update local secrets (plaintext, not committed)
4. Seal secrets and commit to `overrides/openwebui/templates/`
5. Update `overrides/openwebui/values.yaml` with your municipality-specific hostnames
6. Sync the Argo CD application for `openwebui`

---

## Verification

```bash
kubectl get pods -n openwebui
kubectl get svc -n openwebui
kubectl get ingress -n openwebui
```

Secrets:

```bash
kubectl get secret -n openwebui | grep -E "openwebui|cloudnative-pg"
```

---

## Common failure modes

- **UI loads but no chat responses**:
  - `OPENAI_API_KEY` does not match LiteLLM expectations, or LiteLLM is unreachable
- **RAG embeddings fail**:
  - `RAG_OPENAI_API_KEY` does not match vLLM, or vLLM router service URL is wrong
- **DB healthcheck fails**:
  - CloudNativePG not ready or credentials mismatch
- **Websocket issues with multiple replicas**:
  - Redis not running or websocket env vars misconfigured

---

## Summary

Open WebUI provides the user-facing chat interface and integrates with:

- LiteLLM for OpenAI-compatible proxying
- vLLM for local inference/RAG embeddings
- CloudNativePG for persistence (backups disabled until configured)
- Qdrant for vector search
- Redis for websocket/session coordination (multi-replica)

Key requirements:

- Sealed Secrets for credentials
- Correct key wiring across LiteLLM, vLLM, and doc-ingestion
- Generic host configuration (placeholders) for portability across municipalities
- Argo CD as the single source of truth
