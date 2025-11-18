# MBO Application Stack

Consolidated Kubernetes manifests for the MBO (Market By Order) application with Prometheus monitoring and Grafana dashboards.

## 📁 File Structure

```
mbo/
├── mbo-stack.yaml        # All deployments, services, and routes
├── mbo-config.yaml       # All ConfigMaps (nginx, prometheus, grafana)
└── external-secret.yaml  # Bitwarden secrets integration
```

## 🚀 Quick Start

Deploy the entire stack:

```bash
# Apply in order
kubectl apply -f external-secret.yaml
kubectl apply -f mbo-config.yaml
kubectl apply -f mbo-stack.yaml
```

## 📦 Components

### Application Stack
- **mbo-backend** - Rust API server with Prometheus metrics
- **mbo-frontend** - SvelteKit web interface
- **mbo-reverse-proxy** - Nginx reverse proxy

### Monitoring Stack
- **mbo-prometheus** - Metrics collection and storage
- **mbo-grafana** - Dashboard and visualization

## 🔗 Endpoints

- **Application**: https://mbo.hiibolt.com
- **Grafana**: https://mbo-grafana.hiibolt.com
- **Metrics**: https://mbo.hiibolt.com/metrics
- **Health**: https://mbo.hiibolt.com/health
- **OpenAPI**: https://mbo.hiibolt.com/openapi.json

## 🎯 Metrics

The backend exposes comprehensive Prometheus metrics:

- `mbo_active_connections` - Number of active SSE connections
- `mbo_http_requests_total` - Total HTTP requests
- `mbo_db_operations_total` - Database operations
- `mbo_db_operation_duration_seconds` - DB query latency (histogram)
- `mbo_http_request_duration_seconds` - HTTP latency (histogram)
- `mbo_messages_processed_total` - MBO messages processed
- `mbo_messages_errors_total` - Processing errors
- `mbo_order_book_depth` - Current order book depth
- `mbo_order_book_updates_total` - Order book updates
- `mbo_order_book_apply_duration_seconds` - Order book update latency (histogram)

## 🔄 Updates

Update container images:

```bash
# Restart to pull latest :latest tags
kubectl rollout restart deployment/mbo-backend
kubectl rollout restart deployment/mbo-frontend
kubectl rollout restart deployment/mbo-reverse-proxy

# Check status
kubectl rollout status deployment/mbo-backend
kubectl get pods -l app=mbo-backend
```

## 🗑️ Cleanup Old Files

The following files have been consolidated and can be safely deleted:

```bash
# Old deployment files (now in mbo-stack.yaml)
rm deployment.yaml
rm service.yaml
rm http-route.yaml
rm monitoring-deployment.yaml
rm monitoring-service.yaml
rm monitoring-http-route.yaml
rm observability-deployment.yaml
rm observability-service.yaml
rm observability-httproute.yaml

# Old config files (now in mbo-config.yaml)
rm configmap.yaml
rm prometheus-config.yaml
rm grafana-datasources.yaml
rm grafana-dashboards-config.yaml
rm grafana-dashboards-provider.yaml
rm grafana-dashboard.yaml
```

Or use this one-liner:

```bash
cd nuclearbomb/default/mbo
rm deployment.yaml service.yaml http-route.yaml \
   monitoring-deployment.yaml monitoring-service.yaml monitoring-http-route.yaml \
   observability-deployment.yaml observability-service.yaml observability-httproute.yaml \
   configmap.yaml prometheus-config.yaml grafana-datasources.yaml \
   grafana-dashboards-config.yaml grafana-dashboards-provider.yaml grafana-dashboard.yaml
```

## 📊 Grafana Dashboard

A pre-configured dashboard "MBO Order Book Metrics" is automatically provisioned with:

- **Stats Panels**: Messages processed, active connections, order book depth, errors
- **Time Series**: Message processing rate, connection trends
- **Latency Graphs**: Order book apply latency (p50/p95/p99), HTTP request latency (p50/p95/p99)

Access it at: https://mbo-grafana.hiibolt.com

## 🔐 Secrets

Secrets are managed via External Secrets Operator:

- `dbn-key` - Databento API key for MBO data
- `grafana-password` - Grafana admin password

Both pulled from Bitwarden via ClusterSecretStore `bitwarden-login`.

## 💾 Storage

NFS storage at `192.168.1.215:/meow/mbo/`:

- `/data` - SQLite database (mbo-backend)
- `/prometheus` - Prometheus TSDB
- `/grafana` - Grafana dashboards and config

## 🎨 Labels

All resources use consistent labels:

- `app: <service-name>` - Service identifier
- `component: backend|frontend|proxy|monitoring` - Component type

## ✨ Features

- **Auto-scaling SSE**: Backend supports long-running SSE connections for real-time order book updates
- **Comprehensive metrics**: Full instrumentation of backend operations
- **Pre-configured dashboards**: Grafana dashboards auto-provisioned
- **Health probes**: Liveness and readiness checks on all services
- **Resource limits**: CPU/memory limits set for all containers
