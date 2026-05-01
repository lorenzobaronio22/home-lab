# Observability Components

This folder contains observability infrastructure components deployed to the k3s cluster.

## Components

- **grafana-cloud**: Grafana Cloud observability (metrics, logs, traces, alerting) using the Kubernetes Monitoring Helm chart.
- **grafana-dashboards**: Source-controlled Grafana dashboards and alert artifacts, synced via GitHub Actions.

## Deployment Order

Deploy observability after networking and storage are available.

## Free-Tier Guardrails

- Use Grafana Cloud as the only observability backend.
- Keep logs namespace-scoped instead of cluster-wide.
- Keep tracing opt-in for selected applications only.
- Add integrations incrementally and monitor ingestion before widening scope.

## Dashboards as Code

- Dashboard JSON models live in `01.k3s-cluster/03.observability/grafana-dashboards/dashboards/`.
- Alert artifacts live in `01.k3s-cluster/03.observability/grafana-dashboards/alerts/`.
- CI sync workflow: `.github/workflows/grafana-cloud-dashboards-sync.yml`.