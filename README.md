# prometheus

# Spinnaker Monitoring with Prometheus and Grafana

This repository contains complete monitoring infrastructure for Spinnaker Clouddriver services using Prometheus Operator and Grafana.

## Repository Structure

```
prometheus/
├── README.md
├── production
│   ├── clouddriver-dashboards.json
│   ├── prometheus-rbac.yaml
│   ├── prometheus-values.yaml
│   └── service-monitor.yaml
└── sandbox
    ├── clouddriver-dashboards.json
    ├── prometheus-rbac.yaml
    ├── prometheus-values.yaml
    └── service-monitor.yaml


```

## Quick Links

- [Sandbox Deployment](#sandbox-deployment)
- [Production Deployment](#production-deployment)
- [Dashboard Configuration](#grafana-dashboard-configuration)
- [Monitoring Configuration](#service-monitor-configuration)
- [Troubleshooting](#troubleshooting)

## Environment Clouddriver Specifications

| Environment | Pods | Data per Pod | Total Data per Scrape | Scrape Interval | Retention |
|-------------|------|--------------|----------------------|-----------------|-----------|
| **Sandbox** | 4    | ~14MB        | ~56MB                | 30s             | 7 days    |
| **Production** | 65 | ~340MB (before filtering) | ~22GB (before filtering) | 30s | 7 days |

## Sandbox Deployment

### Prerequisites
- Kubernetes cluster with RBAC enabled
- Helm 3.x installed
- kubectl configured for your cluster

### 1. Create Namespace
```bash
kubectl create namespace prometheus
```

### 2. Apply RBAC Configuration
```bash
kubectl apply -f sandbox/prometheus-rbac.yaml
```

### 3. Install Prometheus Operator
```bash
# Add Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus stack
helm install adsk-prometheus prometheus-community/kube-prometheus-stack \
  --namespace prometheus \
  --values sandbox/prometheus-values.yaml \
  --version 55.5.0
```

### 4. Deploy Service Monitor
```bash
kubectl apply -f sandbox/servicemonitor.yaml
```

### 5. Deploy Grafana Dashboard
```bash
kubectl apply -f sandbox/grafana-dashboard.yaml
```

## Production Deployment

### Prerequisites
- Kubernetes cluster with RBAC enabled
- Storage class with high IOPS (recommended: gp3 or equivalent)
- Sufficient cluster resources (see [Resource Requirements](#production-resource-requirements))

### 1. Create Namespace
```bash
kubectl create namespace prometheus
```

### 2. Apply RBAC Configuration
```bash
kubectl apply -f prod/prometheus-rbac.yaml
```

### 3. Install Prometheus Operator
```bash
# Add Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus stack with production configuration
helm install adsk-prometheus prometheus-community/kube-prometheus-stack \
  --namespace prometheus \
  --values prod/prometheus-values.yaml \
  --version 55.5.0
```

### 4. Deploy Service Monitor with Metric Filtering
```bash
kubectl apply -f prod/servicemonitor.yaml
```

### 5. Deploy Grafana Dashboard
```bash
kubectl apply -f prod/grafana-dashboard.yaml
```

## Production Resource Requirements

### Compute Resources
- **CPU**: 8-16 cores minimum
- **Memory**: 16-32GB RAM
- **Storage**: 1TB high-performance SSD (gp3 or equivalent)

### Expected Costs
- **AWS**: ~$3,000-5,000/month

## Grafana Dashboard Configuration

### Dashboard Label Requirements
Grafana dashboards must have the correct label to be automatically discovered:

```yaml
metadata:
  labels:
    grafana_dashboard: "1"  # Critical: Must be "1", not "true"
```

### Supported Dashboards
- **Operations and Tasks**: Controller invocations, task execution, operation latencies
- **SQL Caching**: CATS cache operations, relationships, items read/write/delete

### Adding Custom Dashboards
1. Create ConfigMap in `spinnaker` namespace:
```bash
kubectl create configmap grafana-dashboard-custom \
  --from-file=clouddriver-dashboard.json \
  --namespace=spinnaker

kubectl label configmap grafana-dashboard-custom \
  grafana_dashboard=1 \
  --namespace=spinnaker
```

## Service Monitor Configuration

### Metric Filtering (Production)
The production ServiceMonitor includes aggressive metric filtering to reduce data volume by 80-90%:

- **Kept Metrics**: Operations, tasks, SQL caching, essential monitoring metrics
- **Filtered Out**: JVM, HTTP, Redis caching, container metrics, high-cardinality metrics

### Expected Impact
- **Before Filtering**: 340MB per pod × 65 pods = 22GB per scrape
- **After Filtering**: ~30-50MB per pod × 65 pods = 2-3GB per scrape

## Accessing Services

### Prometheus UI
```bash
kubectl port-forward -n prometheus svc/adsk-prometheus-kube-prome-prometheus 9090:9090
# Access: http://localhost:9090
```

### Grafana UI
```bash
kubectl port-forward -n prometheus svc/adsk-prometheus-grafana 3000:80
# Access: http://localhost:3000
# Default credentials: admin 
  kubectl get secrets adsk-prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d
```

## Monitoring Health

### Check Service Monitor Status
```bash
# View targets in Prometheus UI
kubectl port-forward -n prometheus svc/adsk-prometheus-kube-prome-prometheus 9090:9090
# Go to: http://localhost:9090/targets
```

### Verify Metric Filtering
```promql
# Check sample counts (should be much lower in production)
scrape_samples_scraped{job="spin-clouddriver-caching"}

# Verify specific metrics exist
controller_invocations_total
cats_sqlCache_get_itemCount_total
```

## Troubleshooting

### High Memory Usage
- Check scrape sample counts: `scrape_samples_scraped`
- Verify metric filtering is working
- Consider increasing scrape interval from 30s to 60s

### Dashboard Not Loading
- Verify ConfigMap has correct label: `grafana_dashboard: "1"`
- Check ConfigMap is in correct namespace: `spinnaker`
- Restart Grafana sidecar: `kubectl delete pod -n prometheus -l app.kubernetes.io/name=grafana`

### ServiceMonitor Not Working
- Check ServiceMonitor selector matches service labels
- Verify Prometheus can reach target endpoints
- Check for RBAC issues

### Configuration Changes Not Applied
```bash
# Restart Prometheus to reload configuration
kubectl delete pod -n prometheus prometheus-adsk-prometheus-kube-prome-prometheus-0

# Or use configuration reload (if enabled)
curl -X POST http://localhost:9090/-/reload
```

## Upgrade Instructions

### Upgrading Helm Chart
```bash
# Update repository
helm repo update

# Upgrade with same values
helm upgrade adsk-prometheus prometheus-community/kube-prometheus-stack \
  --namespace prometheus \
  --values prod/prometheus-values.yaml \
  --version <NEW_VERSION>
```

### Updating ServiceMonitor
```bash
# Apply updated ServiceMonitor
kubectl apply -f prod/servicemonitor.yaml

# Wait 30-60 seconds for Prometheus to reload
```

## Security Considerations

- RBAC permissions are minimal and scoped to necessary resources
- Grafana admin password should be changed from default
- Consider network policies to restrict access
- Use TLS for external access in production

## Backup and Recovery

### Prometheus Data
- Prometheus data is stored in persistent volumes
- Configure regular snapshots of PVs
- Consider using Thanos for long-term storage

### Grafana Configuration
- Dashboards are stored as ConfigMaps (version controlled)
- Export custom dashboards regularly
- Backup Grafana database if using external datasources

## Support and Maintenance

### Regular Tasks
- Monitor resource usage and adjust as needed
- Update Helm chart versions quarterly
- Review and optimize metric filtering rules
- Check for Grafana security updates

### Alerting (Optional)
Consider adding AlertManager configuration for:
- High Prometheus memory usage
- Scrape failures
- Disk space warnings
- Service unavailability

## Contributing

When adding new monitoring:
1. Test in sandbox environment first
2. Update both sandbox and production configurations
3. Document any new resource requirements
4. Add appropriate metric filtering for production