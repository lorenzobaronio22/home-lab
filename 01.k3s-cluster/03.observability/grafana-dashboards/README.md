# Grafana Dashboards as Code

This folder stores Grafana Cloud dashboards and alerting artifacts under source control.

## Goals

- Fast home-lab triage (5-minute operational view).
- Low cardinality and low ingestion pressure for Grafana Cloud free tier.
- Reproducible dashboard and alert changes through Git history.

## Structure

- `dashboards/*.json`: Grafana dashboard JSON models (one file per dashboard).
- `alerts/`: Alert rule artifacts and runbook notes.

## Dashboard conventions

- Stable UID naming prefix: `homelab-...`.
- Time range default: 6h.
- Standard variables:
  - `datasource_prometheus` for Prometheus-backed panels.
  - `datasource_loki` for logs panels.
  - `namespace` for reusable namespace filtering.
- Keep query label sets bounded. Avoid pod UID/container ID dimensions unless required for debugging.

## Deployment model

Dashboards are synced by CI through Grafana HTTP API using `overwrite=true` and stable UIDs.

## Required GitHub secrets for CI sync

- `TS_OAUTH_CLIENT_ID`
- `TS_OAUTH_SECRET`
- `GRAFANA_CLOUD_STACK_URL` (example: `https://<stack>.grafana.net`)
- `GRAFANA_CLOUD_DASHBOARDS_API_TOKEN` (Grafana service account token)
- `GRAFANA_CLOUD_FOLDER_UID_HOME_LAB` (target Grafana folder UID)

## Local JSON validation quick check

Use `find` + `jq` to ensure every dashboard file is valid JSON:

```bash
find ./01.k3s-cluster/03.observability/grafana-dashboards/dashboards -name '*.json' -type f -print0 \
  | xargs -0 -I{} sh -c 'jq empty "$1" && echo "OK $1"' sh {}
```

## Live query verification against Grafana Cloud

Run this script after exporting the same values you use in CI:

```bash
export GRAFANA_CLOUD_API_KEY="<token>"
export GRAFANA_CLOUD_METRICS_URL="<https://.../api/prom/push>"
export GRAFANA_CLOUD_METRICS_USERNAME="<prometheus-instance-id>"
export GRAFANA_CLOUD_LOGS_URL="<https://.../loki/api/v1/push>"
export GRAFANA_CLOUD_LOGS_USERNAME="<loki-instance-id>"
export GRAFANA_CLOUD_QUERY_API_KEY="<token-with-metrics-read-and-logs-read-scopes>"

chmod +x ./01.k3s-cluster/03.observability/grafana-dashboards/tools/verify-live-queries.sh
./01.k3s-cluster/03.observability/grafana-dashboards/tools/verify-live-queries.sh
```

Important token note:

- The live query verifier needs a token with read scopes for data-plane APIs.
- Use a Grafana Cloud access policy token with at least `metrics:read` and `logs:read`.
- The script prefers `GRAFANA_CLOUD_QUERY_API_KEY`; if missing, it falls back to `GRAFANA_CLOUD_API_KEY`.

Quick auth preflight checks:

```bash
PROM_QUERY_URL="${GRAFANA_CLOUD_METRICS_URL%/api/prom/push}/api/prom/api/v1/query"
LOKI_QUERY_URL="${GRAFANA_CLOUD_LOGS_URL%/loki/api/v1/push}/loki/api/v1/query"

curl -sS --get -u "$GRAFANA_CLOUD_METRICS_USERNAME:$GRAFANA_CLOUD_QUERY_API_KEY" \
  --data-urlencode 'query=up' "$PROM_QUERY_URL" | jq -r '.status, .error // ""'

curl -sS --get -u "$GRAFANA_CLOUD_LOGS_USERNAME:$GRAFANA_CLOUD_QUERY_API_KEY" \
  --data-urlencode 'query=sum(rate({namespace=~".+"}[5m]))' "$LOKI_QUERY_URL" | jq -r '.status, .error // ""'
```

The script:

- Extracts all panel queries from dashboard JSON files.
- Executes PromQL and LogQL against live Grafana Cloud APIs.
- Produces a TSV report in `01.k3s-cluster/03.observability/grafana-dashboards/reports/`.

Exit codes:

- `0`: all queries returned non-empty results.
- `2`: one or more queries returned API/query errors.
- `3`: one or more queries returned empty results.

To harden queries quickly, share:

- Script summary output lines.
- Report file path.
- Rows where `status` is `EMPTY` or `ERR`.

## Rollback

Revert the dashboard commit and re-run the sync workflow. The API upsert restores previous JSON by UID.
