# Alerts (Grafana Cloud)

This folder contains alerting artifacts for the home-lab operational dashboards.

## Severity model (v1)

- `warning`
- `critical`

## Routing

Primary notification target is Telegram (configured in Grafana Cloud notification policies).

## Rule group files

- `rule-groups/cluster-ops.example.json`: minimal reference example.
- `rule-groups/cluster-ops.v1.json`: production-oriented v1 baseline (warning + critical severities).

## Datasource placeholders

Rule group JSON files keep datasource placeholders:

- `__PROM_DS_UID__`
- `__LOKI_DS_UID__`

The sync workflow replaces them automatically from GitHub secrets.
You can discover datasource UIDs in Grafana UI under data source settings or via Grafana HTTP API.

## Required GitHub secrets for alert sync

The workflow `.github/workflows/grafana-cloud-dashboards-sync.yml` expects:

- `GRAFANA_CLOUD_STACK_URL`
- `GRAFANA_CLOUD_DASHBOARDS_API_TOKEN` (Grafana service-account token with dashboard + alert provisioning write permissions)
- `GRAFANA_CLOUD_FOLDER_UID_HOME_LAB`
- `GRAFANA_CLOUD_PROMETHEUS_DS_UID`
- `GRAFANA_CLOUD_LOKI_DS_UID`

During sync, placeholders in rule group JSON are replaced automatically:

- `__PROM_DS_UID__` -> `GRAFANA_CLOUD_PROMETHEUS_DS_UID`
- `__LOKI_DS_UID__` -> `GRAFANA_CLOUD_LOKI_DS_UID`
