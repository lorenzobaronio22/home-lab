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

## Datasource UIDs

Rule group JSON files contain real datasource UIDs (no placeholders). Discover UIDs in the Grafana UI under **Connections → Data sources → [datasource] → Settings**.

## Sync model

Alert rules are managed directly in Grafana Cloud. Git Sync does not yet support alert rule groups — this folder is maintained under source control for reference and future readiness. When Grafana adds Git Sync support for alerts, no placeholder substitution will be needed.
