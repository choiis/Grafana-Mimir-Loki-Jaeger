# Grafana Mimir Stack for Kubernetes (Minikube)

This directory contains Kubernetes manifests for deploying the complete Grafana Mimir observability stack on Minikube.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Minikube Cluster                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Grafana   │  │   Jaeger    │  │   Consul    │  │    MinIO    │    │
│  │  (LB:3000)  │  │ (LB:16686)  │  │  (LB:8500)  │  │  (LB:9001)  │    │
│  └──────┬──────┘  └─────────────┘  └─────────────┘  └──────┬──────┘    │
│         │                                                   │           │
│         ▼                                                   │           │
│  ┌─────────────────────────────────────────────────────────┴────────┐  │
│  │                         Mimir Components                          │  │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐ │  │
│  │  │  Distributor  │  │Query Frontend │  │      Ingester x3      │ │  │
│  │  │   (LB:8800)   │  │   (LB:8880)   │  │    (StatefulSet)      │ │  │
│  │  └───────────────┘  └───────────────┘  └───────────────────────┘ │  │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐ │  │
│  │  │   Querier x2  │  │   Compactor   │  │  Store Gateway x2     │ │  │
│  │  └───────────────┘  └───────────────┘  │    (StatefulSet)      │ │  │
│  │  ┌───────────────┐  ┌───────────────┐  └───────────────────────┘ │  │
│  │  │     Ruler     │  │ Alertmanager  │                            │  │
│  │  └───────────────┘  └───────────────┘                            │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│         ▲                                                              │
│         │                                                              │
│  ┌──────┴──────┐  ┌─────────────┐                                     │
│  │ Prometheus  │  │    Loki     │                                     │
│  │    x2       │  │             │                                     │
│  └─────────────┘  └─────────────┘                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

## Components

| Component | Type | Replicas | Description |
|-----------|------|----------|-------------|
| **MinIO** | Deployment | 1 | S3-compatible object storage for blocks |
| **Consul** | Deployment | 1 | Service discovery and configuration |
| **Jaeger** | Deployment | 1 | Distributed tracing |
| **Loki** | Deployment | 1 | Log aggregation |
| **Grafana** | Deployment | 1 | Visualization and dashboards |
| **Prometheus** | Deployment | 2 | Metrics collection |
| **Mimir Distributor** | Deployment | 1 | Receives and distributes metrics |
| **Mimir Ingester** | StatefulSet | 3 | Writes metrics to long-term storage |
| **Mimir Querier** | Deployment | 2 | Executes PromQL queries |
| **Mimir Query Frontend** | Deployment | 1 | Query caching and splitting |
| **Mimir Store Gateway** | StatefulSet | 2 | Serves historical data from storage |
| **Mimir Compactor** | StatefulSet | 3 | Compacts blocks in storage |
| **Mimir Ruler** | Deployment | 1 | Evaluates recording and alerting rules |
| **Mimir Alertmanager** | Deployment | 1 | Handles alerts |

## Prerequisites

- Minikube installed and running
- kubectl configured to use Minikube context
- At least 8GB RAM allocated to Minikube (recommended)

```bash
# Start Minikube with sufficient resources
minikube start --memory=8192 --cpus=4
```

## Deployment

### Deploy the stack

```bash
kubectl apply -f Grafana-Mimir-Loki-Jaeger.yaml
```

### Enable external access (LoadBalancer)

In a separate terminal, run:

```bash
minikube tunnel
```

This enables LoadBalancer services to get external IPs.

### Check deployment status

```bash
# View all pods
kubectl -n mimir get pods

# View all services
kubectl -n mimir get svc

# Watch pods until ready
kubectl -n mimir get pods -w
```

## Accessing Services

After running `minikube tunnel`, the following services are accessible via LoadBalancer:

| Service | Port | URL | Credentials |
|---------|------|-----|-------------|
| Grafana | 3000 | http://\<EXTERNAL-IP\>:3000 | admin / admin |
| Consul UI | 8500 | http://\<EXTERNAL-IP\>:8500 | - |
| Jaeger UI | 16686 | http://\<EXTERNAL-IP\>:16686 | - |
| MinIO Console | 9001 | http://\<EXTERNAL-IP\>:9001 | minioadmin / minioadmin |
| Mimir Distributor | 8800 | http://\<EXTERNAL-IP\>:8800 | - |
| Mimir Query Frontend | 8880 | http://\<EXTERNAL-IP\>:8880 | - |

### Alternative: Use minikube service

```bash
# Open Grafana in browser
minikube service -n mimir grafana

# Open Consul in browser
minikube service -n mimir consul

# Open Jaeger in browser
minikube service -n mimir jaeger
```

## Grafana Datasources

The following datasources are pre-configured in Grafana:

| Name | Type | Description |
|------|------|-------------|
| Mimir-Docker | Prometheus | Mimir metrics (tenant: mimir) |
| Prom-Docker | Prometheus | Prometheus metrics (tenant: prom) |
| Loki | Loki | Log aggregation |

## Sending Metrics to Mimir

### Remote Write Configuration

To send metrics from an external Prometheus to Mimir:

```yaml
remote_write:
  - url: http://<DISTRIBUTOR-IP>:8800/api/v1/push
    headers:
      X-Scope-OrgId: <tenant-id>
```

### Example with curl

```bash
# Check distributor health
curl http://<DISTRIBUTOR-IP>:8800/ready

# Check query frontend health
curl http://<QUERY-FRONTEND-IP>:8880/ready
```

## Storage

The stack uses the following PersistentVolumeClaims:

| PVC Name | Size | Used By |
|----------|------|---------|
| minio-pvc | 10Gi | MinIO |
| compactor-data-mimir-compactor-* | 5Gi each | Mimir Compactors (x3) |
| prometheus1-pvc | 5Gi | Prometheus 1 |
| prometheus2-pvc | 5Gi | Prometheus 2 |
| grafana-pvc | 2Gi | Grafana |
| tsdb-storage-mimir-ingester-* | 5Gi each | Mimir Ingesters (x3) |
| tsdb-storage-mimir-store-gateway-* | 5Gi each | Mimir Store Gateways (x2) |

## Cleanup

### Delete all resources

```bash
kubectl delete -f Grafana-Mimir-Loki-Jaeger.yaml
```

### Delete namespace (removes everything)

```bash
kubectl delete namespace mimir
```

### Delete PVCs (to remove persistent data)

```bash
kubectl -n mimir delete pvc --all
```

## Troubleshooting

### Pods not starting

```bash
# Check pod events
kubectl -n mimir describe pod <pod-name>

# Check pod logs
kubectl -n mimir logs <pod-name>
```

### MinIO buckets not created

The `minio-create-buckets` Job should create buckets automatically. Check its status:

```bash
kubectl -n mimir get jobs
kubectl -n mimir logs job/minio-create-buckets
```

### Ingester not joining memberlist

Ingesters use a headless service for memberlist communication. Ensure all ingester pods are running:

```bash
kubectl -n mimir get pods -l app=mimir-ingester
```

### LoadBalancer pending

If LoadBalancer services show `<pending>` for EXTERNAL-IP:

```bash
# Ensure minikube tunnel is running
minikube tunnel
```

## Configuration

### Mimir Configuration

The Mimir configuration is stored in the `mimir-config` ConfigMap. Key settings:

- **Storage Backend**: MinIO (S3-compatible)
- **Memberlist**: Used for ring communication between components
- **Multi-tenancy**: Enabled (use `X-Scope-OrgId` header)
- **Replication Factor**: 2 for ingesters and store-gateways

### Modifying Configuration

1. Edit the ConfigMap in the YAML file
2. Reapply: `kubectl apply -f Grafana-Mimir-Loki-Jaeger.yaml`
3. Restart affected pods: `kubectl -n mimir rollout restart deployment/<name>`

## References

- [Grafana Mimir Documentation](https://grafana.com/docs/mimir/latest/)
- [Grafana Loki Documentation](https://grafana.com/docs/loki/latest/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [Consul Documentation](https://developer.hashicorp.com/consul/docs)
