# Grafana Observability Setup

This guide walks you through setting up the OpenTelemetry Collector to forward Temporal SDK metrics to either:
- **Local Grafana** (runs in Docker, no cloud account needed)
- **Grafana Cloud** (managed service, free tier available)
- **Both** (send metrics to local and cloud simultaneously)

## Prerequisites

- Docker and Docker Compose installed
- Your Temporal SDK Worker running and exposing Prometheus metrics (port 9464)
- (Optional) A GrafanaCloud account for cloud monitoring (https://grafana.com/products/cloud/)

## Quick Start

Choose your deployment option:

### Option 1: Local Grafana (Recommended for Development)

No credentials needed! Just run:

```bash
# Start local Grafana, Prometheus, and OTel Collector
docker-compose -f docker-compose.otel.yaml --profile local up -d

# Start your Temporal SDK Worker
# (command varies by SDK - ensure metrics are exposed on port 9464)

# Access Grafana at http://localhost:4000
# Default credentials: admin/admin
```

### Option 2: Grafana Cloud Only

Requires cloud credentials. See [Grafana Cloud Setup](#grafana-cloud-setup) below.

```bash
# Create .env file with your Grafana Cloud credentials (see below)
# Then start the collector
docker-compose -f docker-compose.otel.yaml --profile cloud up -d

# Start your Temporal SDK Worker
# (command varies by SDK - ensure metrics are exposed on port 9464)
```

### Option 3: Both Local and Cloud

Send metrics to both destinations simultaneously:

```bash
# Create .env file with your Grafana Cloud credentials (see below)
# Start everything
docker-compose -f docker-compose.otel.yaml --profile both up -d

# Start your Temporal SDK Worker
# (command varies by SDK - ensure metrics are exposed on port 9464)

# Access local Grafana at http://localhost:4000
```

---

## Grafana Cloud Setup

**Only required for `cloud` or `both` profiles.**

### Step 1: Get GrafanaCloud Credentials

1. Log into your GrafanaCloud account
2. Navigate to **Connections** → **Add new connection** → **Hosted Prometheus metrics**
3. Or go directly to: https://grafana.com/docs/grafana-cloud/send-data/metrics/metrics-prometheus/
4. You'll need:
   - **Prometheus Remote Write Endpoint**: Usually looks like `https://prometheus-prod-XX-XXX.grafana.net/api/prom/push`
   - **Username**: Your instance ID (e.g., `123456`)
   - **API Key/Token**: Generate a new API token with `metrics:write` permission

### Step 2: Create Environment Variables

Create a `.env` file in the project root (this file is gitignored):

```bash
# .env file
GRAFANA_CLOUD_PROMETHEUS_ENDPOINT=https://prometheus-prod-XX-XXX.grafana.net/api/prom/push
GRAFANA_CLOUD_USERNAME=123456
GRAFANA_CLOUD_PASSWORD=glc_xxxxxxxxxxxxxxxxxxxx
```

---

## Verifying Metrics

### Local Grafana

1. Open http://localhost:4000 (login: admin/admin)
2. Go to **Explore**
3. The Prometheus datasource is pre-configured
4. Query for metrics like:
   - `temporal_workflow_*`
   - `temporal_activity_*`
   - `temporal_worker_*`
5. Filter by label: `service="temporal-worker"`

### Grafana Cloud

1. Log into your GrafanaCloud dashboard
2. Go to **Explore**
3. Select your Prometheus data source
4. Query for metrics (same as above)

## Troubleshooting

### Collector can't reach worker metrics

If you see errors about connecting to `host.docker.internal:9464`:

**On Linux**: Replace `host.docker.internal` with your machine's IP or `172.17.0.1` in `otel-collector-config.yaml`:
```yaml
- targets: ['172.17.0.1:9464']
```

**On macOS/Windows**: `host.docker.internal` should work out of the box.

### No metrics appearing in GrafanaCloud

1. Check collector logs: `docker-compose -f docker-compose.otel.yaml logs otel-collector`
2. Verify worker is exposing metrics: `curl http://localhost:9464/metrics`
3. Check GrafanaCloud API key has `metrics:write` permission
4. Verify the endpoint URL is correct (should end with `/api/prom/push`)

### Authentication errors

If you see 401/403 errors, check your API key format:
- Should be `<username>:<api-token>` or just the token if using Bearer auth
- Make sure the API token is active and has proper permissions

## Customization

### Change scrape interval

Edit `otel-collector-config.yaml`:
```yaml
scrape_interval: 30s  # Change from default 15s
```

### Add additional labels

Edit the `resource` processor in `otel-collector-config.yaml`:
```yaml
processors:
  resource:
    attributes:
      - key: team
        value: platform
        action: upsert
```

### Enable debug logging

Edit `otel-collector-config.yaml`, uncomment the logging exporter:
```yaml
service:
  pipelines:
    metrics:
      exporters: [prometheusremotewrite, logging]
```

## Stopping Services

```bash
# Stop all services (use the same profile you started with)
docker-compose -f docker-compose.otel.yaml --profile local down
# or
docker-compose -f docker-compose.otel.yaml --profile cloud down
# or
docker-compose -f docker-compose.otel.yaml --profile both down
```

## Architecture

### Local Profile
```
Temporal SDK Worker (localhost:9464)
         ↓ (scrape every 15s)
OpenTelemetry Collector (Docker)
         ↓ (batch & forward)
Prometheus (Docker:9090)
         ↓
Grafana (Docker:4000)
```

### Cloud Profile
```
Temporal SDK Worker (localhost:9464)
         ↓ (scrape every 15s)
OpenTelemetry Collector (Docker)
         ↓ (batch & forward)
GrafanaCloud Prometheus
         ↓
GrafanaCloud Dashboards
```

### Both Profile
```
Temporal SDK Worker (localhost:9464)
         ↓ (scrape every 15s)
OpenTelemetry Collector (Docker)
         ↓ (batch & forward to both)
         ├─→ Local Prometheus (Docker:9090) → Grafana (Docker:4000)
         └─→ GrafanaCloud Prometheus → GrafanaCloud Dashboards
```

## Next Steps

- Create Grafana dashboards for your Temporal metrics
- Set up alerts for workflow failures or high activity latency
- Consider adding traces and logs for full observability