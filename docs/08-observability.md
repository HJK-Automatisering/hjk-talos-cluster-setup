---
title: 08 – Observability (Prometheus, Grafana, Loki, Tempo)
nav_order: 2
parent: Platform Management
---

# Observability (Prometheus, Grafana, Loki, Tempo)

This cluster includes a standard observability stack for metrics, logs, and traces.

- **Prometheus Stack** (Prometheus + Alerting components + Grafana)
- **Loki** (logs)
- **Tempo** (traces)
- (Optional) **Alloy** (agent/collector depending on how it is configured in your vendor setup)

The stack is deployed via **Argo CD** using the **vendor + overrides** model.

---

## Architecture: vendor vs overrides

### Vendor
Vendor charts live under:

- `vendor/applications/prometheus-stack`
- `vendor/applications/loki`
- `vendor/applications/tempo`
- `vendor/applications/alloy`

The vendor `prometheus-stack` chart also ships example dashboards as templates:

- `templates/DCGM-dashboard.yaml` (GPU dashboard)
- `templates/fastapi-dashboard.yaml`
- `templates/webui-dashboard.yaml`

### Overrides
Cluster-specific changes live under:

- `overrides/prometheus-stack/values.yaml`
- `overrides/prometheus-stack/templates/*` (e.g. sealed secrets)

This typically includes:

- Grafana ingress configuration
- OAuth/OIDC authentication settings
- Datasource wiring (Loki + Tempo)
- TLS and domain settings
- Any required secrets (OIDC client secret, etc.)

---

## GitOps deployment (Argo CD)

Observability applications are usually defined in:

- `overrides/argo-cd-resources/values.yaml`

Typical apps:

- `prometheus-stack` (namespace `monitoring`)
- `loki` (namespace `monitoring`)
- `tempo` (namespace `monitoring`)
- `alloy` (namespace `monitoring`)

Once Argo CD is bootstrapped, the stack is reconciled automatically by Argo CD.

---

## Prometheus Stack configuration

Cluster settings are defined in:

- `overrides/prometheus-stack/values.yaml`

Common configuration areas:

### Grafana Ingress / external access
Ingress settings are cluster-specific and configured in the overrides values file.

Because different municipalities will use different DNS and domains, the documentation stays generic:

- Update the Grafana ingress host(s) to match your environment
- Ensure TLS secret name matches your cluster (e.g. wildcard cert)

Example (conceptual):

```yaml
prometheus-stack:
  grafana:
    ingress:
      enabled: true
      ingressClassName: traefik
      hosts:
        - <grafana-domain>
      tls:
        - secretName: <tls-secret>
          hosts:
            - <grafana-domain>
```

If you do not use ingress, you can port-forward Grafana locally instead (see below).

---

## Grafana authentication (OIDC / OAuth)

Grafana can be configured to authenticate using an OIDC provider (for example Authentik).

In this setup:

- Grafana reads the OIDC client secret from an environment variable
- The secret is provided via a Kubernetes Secret (often via Sealed Secrets)

### Sealed secret
The override includes a template like:

- `overrides/prometheus-stack/templates/sealed-grafana-oidc-secret.yaml`

This secret typically provides:

- `GRAFANA_OAUTH_CLIENT_SECRET`

### Grafana ini configuration
OIDC settings are configured in:

- `overrides/prometheus-stack/values.yaml` under `grafana.grafana.ini`

Example concepts:

- `auth.generic_oauth.enabled = true`
- `auth.generic_oauth.client_id = ...`
- `auth.generic_oauth.client_secret = $__env{GRAFANA_OAUTH_CLIENT_SECRET}`
- issuer/authorize/token/userinfo endpoints for your provider

> Important: Keep all municipality-specific URLs (issuer/auth/token endpoints and domains) in overrides only.

---

## Custom Grafana image (custom CA certificates)

When Grafana needs to talk to an internal OIDC provider using a certificate signed by an internal CA,
you must ensure Grafana trusts that CA.

This repo uses a custom Grafana image that includes the internal CA certificate:

- `docker/grafana/Dockerfile`

Example pattern:

1. Copy the internal CA certificate into the container
2. Install/update CA certificates in the image

This is necessary when:

- your OIDC provider uses internal TLS certificates
- Grafana fails with TLS verification errors when contacting auth endpoints

---

## Datasources (Loki + Tempo)

Grafana is configured to include additional datasources:

- **Loki** for logs
- **Tempo** for traces

These are commonly configured in:

- `overrides/prometheus-stack/values.yaml` under `grafana.additionalDataSources`

Conceptual example:

```yaml
grafana:
  additionalDataSources:
    - name: Loki
      type: loki
      url: http://loki-headless:3100

    - name: Tempo
      type: tempo
      url: http://tempo:3200
      jsonData:
        tracesToLogs:
          datasourceUid: 'Loki'
        serviceMap:
          datasourceUid: 'prometheus'
```

This enables cross-navigation:

- traces → related logs
- traces → service map
- traces → metrics panels (if configured)

---

## Access Grafana

### Option A: Port-forward (recommended for initial validation)

```bash
kubectl -n monitoring port-forward svc/prometheus-stack-grafana 3000:80
```

Then open:

- http://localhost:3000

### Option B: Ingress
If ingress is enabled, use your configured DNS name (from overrides).

---

## Validation checklist

### 1) Pods running
```bash
kubectl -n monitoring get pods
```

### 2) Grafana datasources present
In Grafana UI:

- Check **Connections → Data sources**
- Ensure Prometheus is present
- Ensure Loki and Tempo are present (if enabled)

### 3) Dashboards present
In Grafana UI:

- Check **Dashboards**
- Confirm vendor dashboards (GPU / FastAPI / WebUI) exist if those templates are enabled

### 4) Loki logs query works
In Grafana Explore:

- Select Loki datasource
- Query `{namespace="monitoring"}` or another relevant namespace

### 5) Tempo traces visible (if instrumented)
In Grafana Explore:

- Select Tempo datasource
- Query traces for instrumented services

---

## Troubleshooting

### Grafana cannot log in via OIDC
Common causes:

- wrong issuer/auth/token/userinfo URLs
- TLS errors (missing CA in the Grafana image)
- missing `GRAFANA_OAUTH_CLIENT_SECRET` secret
- wrong redirect URL configured in the OIDC provider

Check:

- Grafana pod logs:
  ```bash
  kubectl -n monitoring logs deploy/prometheus-stack-grafana
  ```
- Ensure the secret exists:
  ```bash
  kubectl -n monitoring get secret grafana-oidc-secret -o yaml
  ```

### TLS verification errors to OIDC provider
If logs mention certificate verification issues:

- confirm Grafana uses the custom image that includes internal CA certs
- confirm the CA certificate is correct and updated
- confirm `update-ca-certificates` runs successfully in the image build

### Loki / Tempo datasource not working
Check:

- service names and URLs in `additionalDataSources`
- that Loki/Tempo services exist:
  ```bash
  kubectl -n monitoring get svc | egrep "loki|tempo|grafana|prom"
  ```
- network policies / ingress policies (if any)

---
