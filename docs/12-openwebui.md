---
layout: default
title: Open WebUI
nav_order: 3
parent: Applications
---

# Open WebUI (Chat UI + RAG)

This section documents how we deploy **Open WebUI** as the user-facing chat interface, integrated with:

- **LiteLLM** as the OpenAI-compatible proxy
- **vLLM** for local inference / embeddings
- **CloudNativePG** for persistence
- **Qdrant** as the vector database
- **Redis** for websockets/session middleware (multi-replica)
- **S3-compatible storage** (Rook Ceph RGW) for persistence

Open WebUI is deployed via Helm and managed via **ArgoCD** (GitOps).

---

## Project structure

The Open WebUI application lives under:

`applications/openwebui/`

```text
applications/openwebui/
├── Chart.yaml
├── cloudnative-pg-values.yaml
├── qdrant-values.yaml
├── values.yaml
├── local-secrets/
│   ├── cloudnative-pg-cluster-openwebui-secret.yaml
│   ├── openwebui-secrets.yaml
│   └── openwebui-secrets.yaml.bak
└── templates/
    ├── database.yaml
    ├── s3-claim.yaml
    ├── sealed-cloudnative-pg-cluster-openwebui-secret.yaml
    ├── sealed-openwebui-secrets.yaml
    └── was-middleware.yaml
```

---

## Prerequisites

- Kubernetes cluster reachable and healthy
- Namespace `openwebui` exists
- **Sealed Secrets** controller installed (see **09 – Sealed Secrets**)
- **LiteLLM** deployed (see **11 – LiteLLM**)
- **vLLM** deployed if used for local inference/RAG embeddings (see **10 – vLLM**)
- CloudNativePG operator installed
- Storage:
  - Rook Ceph RGW is available if using S3 persistence
  - A TLS wildcard cert exists if using `wildcard-tls` (or adjust to your environment)
- Ingress controller (Traefik) available

---

## Ingress and host configuration (generic)

This repository must remain municipality-agnostic. Replace host values with your own domain.

In `values.yaml`, keep a generic host placeholder:

- Ingress host: `WEBUI_HOST`
- TLS secret: `wildcard-tls` (or your own secret)

Example:

```yaml
ingress:
  enabled: true
  class: "traefik"
  host: WEBUI_HOST
  tls:
    - hosts:
        - WEBUI_HOST
      secretName: wildcard-tls
```

Also update these env vars to match your domain:

```yaml
- name: WEBUI_URL
  value: "https://WEBUI_HOST"
- name: CORS_ALLOW_ORIGIN
  value: "https://WEBUI_HOST;http://WEBUI_HOST"
```

---

## Core routing: Open WebUI → LiteLLM

Open WebUI is configured to use LiteLLM as its OpenAI API backend:

```yaml
openaiBaseApiUrl: "http://litellm.litellm.svc.cluster.local:4000/v1"
enableOpenaiApi: true
```

### Important: `OPENAI_API_KEY` must match LiteLLM

Open WebUI sends `OPENAI_API_KEY` when calling the OpenAI-compatible endpoint.

In this setup, that endpoint is **LiteLLM**, so the key must match what LiteLLM expects
(typically the LiteLLM `PROXY_MASTER_KEY`, unless you configure LiteLLM differently).

`openwebui-secrets.yaml`:

```yaml
stringData:
  OPENAI_API_KEY: "SECRET"
```

Ensure `OPENAI_API_KEY` aligns with the key configured in **LiteLLM** (see **11 – LiteLLM**).

---

## RAG configuration: embeddings and vector store

Open WebUI uses:

- **Qdrant** as vector database
- **vLLM** for embeddings and/or RAG calls

Configured in `values.yaml`:

```yaml
- name: VECTOR_DB
  value: "qdrant"
- name: QDRANT_URI
  value: "http://openwebui-qdrant:6334"
- name: QDRANT_PREFER_GRPC
  value: "True"
- name: QDRANT_GRPC_PORT
  value: "6334"
```

For embeddings:

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

### Important: `RAG_OPENAI_API_KEY` must match vLLM

`RAG_OPENAI_API_KEY` must match the API key configured for vLLM (the secret backing vLLM’s `vllmApiKey`).
If the keys differ, RAG embedding calls will fail with authentication errors.

---

## Websocket / multi-replica support (Redis)

Open WebUI runs with:

```yaml
replicaCount: 3
```

When replicas > 1, websocket/session coordination is required. This setup enables:

- websocket support
- Redis as websocket manager
- Redis session middleware

Values:

```yaml
websocket:
  enabled: true
  manager: redis
  url: redis://open-webui-redis:6379/0
  redis:
    enabled: true
```

And env vars:

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

## Persistence: S3 (Rook Ceph RGW)

Persistence is enabled and configured for S3-compatible storage:

```yaml
persistence:
  enabled: true
  provider: s3
  s3:
    endpointUrl: "http://rook-ceph-rgw-ceph-objectstore.rook-ceph.svc"
    bucket: "openwebui"
    accessKeyExistingSecret: "s3-openwebui"
    accessKeyExistingAccessKey: "AWS_ACCESS_KEY_ID"
    secretKeyExistingSecret: "s3-openwebui"
    secretKeyExistingSecretKey: "AWS_SECRET_ACCESS_KEY"
```

Ensure:
- the `s3-openwebui` secret exists with the expected keys
- the RGW endpoint is reachable from the `openwebui` namespace
- the bucket exists (or is created by your workflow)

The repo includes `templates/s3-claim.yaml` to support S3 claim provisioning (if you use that workflow).

---

## Database: CloudNativePG

Open WebUI uses Postgres via CloudNativePG and sets `DATABASE_URL` from secrets:

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

The credentials secret is sealed and committed:

- Local: `local-secrets/cloudnative-pg-cluster-openwebui-secret.yaml`
- Sealed: `templates/sealed-cloudnative-pg-cluster-openwebui-secret.yaml`

### Backups (important)

As with LiteLLM, CloudNativePG backups must be configured manually.
Keep backups **disabled** until object storage credentials exist.

In `cloudnative-pg-values.yaml`:

```yaml
backups:
  enabled: false
```

If backups are enabled but `hetzner-s3-backup-credentials` (or equivalent) is missing/invalid,
the operator will continuously emit errors and may fill storage with logs.

---

## Secrets (must be consistent across services)

Open WebUI secrets:

`applications/openwebui/local-secrets/openwebui-secrets.yaml`

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: openwebui-secrets
  namespace: openwebui
stringData:
  WEBUI_SECRET_KEY: "SECRET"
  OPENAI_API_KEY: "SECRET"
  RAG_OPENAI_API_KEY: "SECRET"
  EXTERNAL_DOCUMENT_LOADER_API_KEY: "SECRET"
```

### Key coupling (important)

- `OPENAI_API_KEY` must match what **LiteLLM** expects (typically LiteLLM `PROXY_MASTER_KEY`)
- `RAG_OPENAI_API_KEY` must match the **vLLM** API key
- `EXTERNAL_DOCUMENT_LOADER_API_KEY` must match the **doc-ingestion** service API key

If any of these keys are rotated, update and reseal the Open WebUI secret accordingly.

Sealed version committed to Git:

- `templates/sealed-openwebui-secrets.yaml`

---

## Deployment via ArgoCD

Recommended order:

1. Ensure Sealed Secrets controller exists (**09 – Sealed Secrets**)
2. Ensure dependencies exist:
   - LiteLLM (**11 – LiteLLM**)
   - vLLM (**10 – vLLM**) if used for RAG/embeddings
   - doc-ingestion if content extraction is enabled
3. Create/update local secrets:
   - `openwebui-secrets.yaml`
   - `cloudnative-pg-cluster-openwebui-secret.yaml`
   - S3 secret (if used): `s3-openwebui`
4. Seal secrets and commit to `templates/`
5. Update `values.yaml` with your `WEBUI_HOST`
6. Sync the ArgoCD application for `openwebui`

---

## Verification

### Workloads and ingress

```bash
kubectl get pods -n openwebui
kubectl get svc -n openwebui
kubectl get ingress -n openwebui
```

### Secrets

```bash
kubectl get secret openwebui-secrets -n openwebui
kubectl get secret cloudnative-pg-cluster-openwebui -n openwebui
```

### Common failure modes

- **UI loads but no chat responses**: `OPENAI_API_KEY` does not match LiteLLM expectations or LiteLLM is unreachable
- **RAG embeddings fail**: `RAG_OPENAI_API_KEY` does not match vLLM, or vLLM router service URL is wrong
- **DB healthcheck fails (`/health/db`)**: CloudNativePG not ready or credentials mismatch
- **Persistent storage issues**: S3 secret missing/keys wrong, RGW endpoint unreachable, bucket missing
- **Websocket issues with multiple replicas**: Redis not running or websocket env vars misconfigured

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
- Generic host configuration (`WEBUI_HOST`) for portability across municipalities
- ArgoCD as the single source of truth
