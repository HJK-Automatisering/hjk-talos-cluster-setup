---
title: Observability (Prometheus, Grafana, Loki, Tempo)
nav_order: 2
parent: Platform Management
---

# Observability (Prometheus, Grafana, Loki, Tempo)

This cluster includes a standard observability stack for metrics, logs, traces, dashboards, and alerting.

- **Prometheus Stack** (Prometheus + Alertmanager + Grafana)
- **Loki** (logs)
- **Tempo** (traces)
- **Alloy** (OpenTelemetry / collection pipeline)
- **prometheus-msteams** (bridge from Alertmanager to Microsoft Teams)

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
- `overrides/prometheus-stack/templates/*`
- `overrides/prometheus-msteams/*`

This typically includes:

- Grafana ingress configuration
- OAuth / OIDC authentication settings
- Datasource wiring (Loki + Tempo)
- TLS and domain settings
- Custom dashboards
- Alert rules
- Alert routing to Microsoft Teams
- Any required secrets (OIDC client secret, Teams webhook, etc.)

---

## GitOps deployment (Argo CD)

Observability applications are usually defined in:

- `overrides/argo-cd-resources/values.yaml`

Typical apps:

- `prometheus-stack` (namespace `monitoring`)
- `loki` (namespace `monitoring`)
- `tempo` (namespace `monitoring`)
- `alloy` (namespace `monitoring`)
- `prometheus-msteams` (namespace `monitoring`)

Once Argo CD is bootstrapped, the stack is reconciled automatically by Argo CD.

---

## Prometheus Stack configuration

Cluster settings are defined in:

- `overrides/prometheus-stack/values.yaml`

Common configuration areas:

### Grafana ingress / external access
Ingress settings are cluster-specific and configured in the overrides values file.

Because different municipalities will use different DNS and domains, the documentation stays generic:

- update the Grafana ingress host(s) to match your environment
- ensure TLS secret name matches your cluster (for example wildcard cert)

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
- the secret is provided via a Kubernetes Secret (often via Sealed Secrets)

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
- issuer / authorize / token / userinfo endpoints for your provider

> Important: Keep all municipality-specific URLs (issuer, auth, token, userinfo, and domains) in overrides only.

---

## Custom Grafana image (custom CA certificates)

When Grafana needs to talk to an internal OIDC provider using a certificate signed by an internal CA,
you must ensure Grafana trusts that CA.

This repo uses a custom Grafana image that includes the internal CA certificate:

- `docker/grafana/Dockerfile`

Example pattern:

1. copy the internal CA certificate into the container
2. install / update CA certificates in the image

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

## Custom dashboards

In addition to the vendor-provided dashboards, this cluster includes custom dashboards committed as ConfigMaps under:

- `overrides/prometheus-stack/templates/`

Typical custom dashboards include:

- `openwebui-dashboard.yaml`
- `vllm-dashboard.yaml`
- `litellm-dashboard.yaml`

The current setup also includes GPU monitoring resources under:

- `dcgm-exporter-daemonset.yaml`
- `dcgm-exporter-service.yaml`
- `dcgm-exporter-servicemonitor.yaml`

### Current dashboard coverage

The current dashboard set provides an operational baseline for:

- **DCGM / GPU monitoring**
- **Open WebUI**
- **vLLM**
- **LiteLLM**

These dashboards are intended to provide practical visibility into:

- infrastructure health
- GPU usage and memory pressure
- inference latency and throughput
- proxy latency and spend metrics
- application metrics and logs

> Important: Dashboard queries may need small adjustments if upstream metric names change between application versions.

---

## Alerting

This cluster includes Prometheus alert rules for the main platform components.

Alert rules are committed in:

- `overrides/prometheus-stack/templates/alerts-gpu.yaml`
- `overrides/prometheus-stack/templates/alerts-vllm.yaml`
- `overrides/prometheus-stack/templates/alerts-litellm.yaml`
- `overrides/prometheus-stack/templates/alerts-openwebui.yaml`

The initial alert set is intentionally small and focused on:

- exporter / target availability
- missing metrics
- sustained high latency
- GPU pressure

### Current alert coverage

The current setup includes alerts for:

- **DCGM exporter availability**
- **vLLM engine availability**
- **vLLM latency**
- **LiteLLM availability**
- **LiteLLM latency**
- **Open WebUI metrics missing**

These alerts use labels such as:

```yaml
team: platform
severity: critical
component: vllm
```

This makes it possible to route platform alerts separately from other future alert categories.

---

## Alert routing to Microsoft Teams

Alert delivery is configured as:

- **Prometheus / Alertmanager** evaluates and routes alerts
- **prometheus-msteams** receives Alertmanager webhook notifications
- **Microsoft Teams** receives alerts in a dedicated channel

The Teams bridge is deployed from:

- `overrides/prometheus-msteams/`

and registered via:

- `overrides/argo-cd-resources/values.yaml`

Alertmanager routing is configured in:

- `overrides/prometheus-stack/values.yaml`

### Teams channel

The current setup routes platform alerts to the Teams channel:

- `platform-alerts`

The bridge uses a Teams webhook stored as a Kubernetes Secret / SealedSecret.

> Important: Never commit a plaintext Teams webhook URL to Git.
> Store it only in Kubernetes Secrets / SealedSecrets.

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

- check **Connections → Data sources**
- ensure Prometheus is present
- ensure Loki and Tempo are present (if enabled)

### 3) Dashboards present
In Grafana UI:

- check **Dashboards**
- confirm vendor dashboards exist if those templates are enabled
- confirm custom dashboards for Open WebUI, vLLM, LiteLLM, and GPU monitoring are present

### 4) Loki logs query works
In Grafana Explore:

- select Loki datasource
- query `{namespace="monitoring"}` or another relevant namespace

### 5) Tempo traces visible (if instrumented)
In Grafana Explore:

- select Tempo datasource
- query traces for instrumented services

### 6) Prometheus alert rules present
```bash
kubectl -n monitoring get prometheusrule
```

### 7) Alertmanager running
```bash
kubectl -n monitoring get pods | grep alertmanager
```

### 8) Teams bridge running
```bash
kubectl -n monitoring get pods | grep prometheus-msteams
kubectl -n monitoring get svc | grep prometheus-msteams
kubectl -n monitoring get servicemonitor | grep prometheus-msteams
```

---

## Troubleshooting

### Grafana cannot log in via OIDC
Common causes:

- wrong issuer / auth / token / userinfo URLs
- TLS errors (missing CA in the Grafana image)
- missing `GRAFANA_OAUTH_CLIENT_SECRET` secret
- wrong redirect URL configured in the OIDC provider

Check:

- Grafana pod logs:
  ```bash
  kubectl -n monitoring logs deploy/prometheus-stack-grafana
  ```
- ensure the secret exists:
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
- that Loki / Tempo services exist:
  ```bash
  kubectl -n monitoring get svc | egrep "loki|tempo|grafana|prom"
  ```
- network policies / ingress policies (if any)

### Alert rules are present but no Teams messages arrive
Check:

- that `prometheus-msteams` is running:
  ```bash
  kubectl -n monitoring get pods | grep prometheus-msteams
  ```
- bridge logs:
  ```bash
  kubectl -n monitoring logs deploy/prometheus-msteams --since=10m
  ```
- Alertmanager reload logs:
  ```bash
  kubectl -n monitoring logs alertmanager-prometheus-stack-vendor-alertmanager-0 --since=10m
  ```
- rendered Alertmanager config (generated secret):
  ```bash
  kubectl get secret -n monitoring alertmanager-prometheus-stack-vendor-alertmanager-generated -o json \
  | jq -r '.data["alertmanager.yaml.gz"]' \
  | base64 -d \
  | gunzip -c
  ```

### Open WebUI dashboards show no data
This setup relies on OpenTelemetry metrics flowing through Alloy into Prometheus.

Check:

- Open WebUI environment variables for OTEL are enabled
- Alloy receiver / exporter configuration is correct
- Prometheus can see `http_server_*` metrics for `job="open-webui"`

### vLLM or LiteLLM dashboards show no data
Check:

- the ServiceMonitor exists
- the Prometheus target is `up`
- the metrics endpoint returns data
- the dashboard uses the metric names exposed by the deployed application version

---

## Operational notes

The current observability and alerting setup is intended as a first operational baseline.

It is recommended to review and tune over time:

- latency thresholds
- repeat intervals
- grouping behavior
- severity levels
- Teams notification noise
- dashboard queries after application upgrades

As usage patterns become clearer, additional alerts can be introduced gradually.

---

## Summary

The observability stack now includes:

- metrics via Prometheus
- dashboards via Grafana
- logs via Loki
- traces via Tempo
- OTEL collection / forwarding via Alloy
- platform alerts routed to Microsoft Teams

This provides a practical baseline for monitoring GPU capacity, inference services, proxy behavior, alert delivery, and the Open WebUI application layer in a GitOps-managed cluster.
