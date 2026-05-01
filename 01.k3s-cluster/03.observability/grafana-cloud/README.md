# Grafana Cloud Observability

Reference docs:

- https://grafana.com/docs/grafana-cloud/monitor-infrastructure/kubernetes-monitoring/
- https://github.com/grafana/k8s-monitoring-helm

## Files in this folder

- `Chart.yaml`: pins the Grafana Kubernetes Monitoring chart dependency version.
- `values.yaml`: non-secret default values and free-tier guardrails.

## What this deploys

- Kubernetes and node metrics to Grafana Cloud Prometheus.
- Namespace-scoped pod logs to Grafana Cloud Loki.
- OTLP receivers for app traces to Grafana Cloud Tempo.

## Install or upgrade locally (manual)

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm dependency update ./01.k3s-cluster/03.observability/grafana-cloud

export GRAFANA_CLOUD_API_KEY="<cloud-access-policy-token-with-metrics-logs-traces-write>"
export GRAFANA_CLOUD_METRICS_URL="<https://.../api/prom/push>"
export GRAFANA_CLOUD_METRICS_USERNAME="<prometheus-instance-id>"
export GRAFANA_CLOUD_LOGS_URL="<https://.../loki/api/v1/push>"
export GRAFANA_CLOUD_LOGS_USERNAME="<loki-instance-id>"
export GRAFANA_CLOUD_OTLP_URL="<https://.../otlp>"
export GRAFANA_CLOUD_OTLP_USERNAME="<tempo-instance-id>"

helm upgrade \
  --install \
  grafana-cloud-observability \
  ./01.k3s-cluster/03.observability/grafana-cloud \
  --namespace observability \
  --create-namespace \
  --values ./01.k3s-cluster/03.observability/grafana-cloud/values.yaml \
  --set-string k8s-monitoring.destinations.grafanaCloudMetrics.url="$GRAFANA_CLOUD_METRICS_URL" \
  --set-string k8s-monitoring.destinations.grafanaCloudMetrics.auth.username="$GRAFANA_CLOUD_METRICS_USERNAME" \
  --set-string k8s-monitoring.destinations.grafanaCloudMetrics.auth.password="$GRAFANA_CLOUD_API_KEY" \
  --set-string k8s-monitoring.destinations.grafanaCloudLogs.url="$GRAFANA_CLOUD_LOGS_URL" \
  --set-string k8s-monitoring.destinations.grafanaCloudLogs.auth.username="$GRAFANA_CLOUD_LOGS_USERNAME" \
  --set-string k8s-monitoring.destinations.grafanaCloudLogs.auth.password="$GRAFANA_CLOUD_API_KEY" \
  --set-string k8s-monitoring.destinations.grafanaCloudTraces.url="$GRAFANA_CLOUD_OTLP_URL" \
  --set-string k8s-monitoring.destinations.grafanaCloudTraces.auth.username="$GRAFANA_CLOUD_OTLP_USERNAME" \
  --set-string k8s-monitoring.destinations.grafanaCloudTraces.auth.password="$GRAFANA_CLOUD_API_KEY" \
  --wait
```

## Free-tier defaults in `values.yaml`

- Pod logs are limited to `kube-system`, `longhorn-system`, and `homepage` namespaces.
- High-cost features stay disabled by default: profiling, cost metrics, auto-instrumentation, broad pod-log modes.
- Tracing is enabled only as receiver infrastructure. You decide app-by-app instrumentation.

## Required GitHub repository secrets

- `TS_OAUTH_CLIENT_ID`
- `TS_OAUTH_SECRET`
- `GRAFANA_CLOUD_API_KEY`
- `GRAFANA_CLOUD_METRICS_URL`
- `GRAFANA_CLOUD_METRICS_USERNAME`
- `GRAFANA_CLOUD_LOGS_URL`
- `GRAFANA_CLOUD_LOGS_USERNAME`
- `GRAFANA_CLOUD_OTLP_URL`
- `GRAFANA_CLOUD_OTLP_USERNAME`

## Verify installation

```bash
kubectl -n observability get pods
kubectl -n observability get configmaps
```

Then validate in Grafana Cloud:

1. Metrics in Explore (Prometheus datasource) for cluster and nodes.
2. Logs only from the allowlisted namespaces.
3. Traces appear only for explicitly instrumented apps.
4. Alert rule test notifications reach Telegram.