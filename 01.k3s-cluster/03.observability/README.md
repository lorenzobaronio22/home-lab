# Observability Components

This folder contains observability infrastructure components deployed to the k3s cluster.

## Components

- **grafana-cloud**: Grafana Cloud observability (metrics, logs, traces, alerting) using the Kubernetes Monitoring Helm chart.

## Deployment Order

Deploy observability after networking and storage are available.

## Free-Tier Guardrails

- Use Grafana Cloud as the only observability backend.
- Keep logs namespace-scoped instead of cluster-wide.
- Keep tracing opt-in for selected applications only.
- Add integrations incrementally and monitor ingestion before widening scope.