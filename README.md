# Observability Monitoring Grafana Mimir MSA & Loki & Jaeger

## Grafana Mimir MSA + Loki + Prometheus + Grafana metrics + Jaeger Trace example

* Grafana Mimir is an open source, horizontally scalable, highly available, multi-tenant, long-term storage for Prometheus.
* For information on Mimir, see the link below https://grafana.com/oss/mimir/

* Jaeger is open source, end-to-end distributed tracing
* For information on Jaeger, see the link below https://www.jaegertracing.io/

* Loki is a log aggregation system designed to store and query logs from all your applications and infrastructure.
* For information on Loki, see the link below https://grafana.com/oss/loki/

---

## Directory Structure

```
.
├── docker-compose/          # Docker Compose deployment files
│   ├── docker-compose.yml   # Main compose file
│   ├── mimir.yml            # Mimir configuration
│   ├── consul/              # Consul configuration
│   ├── datasources.yml      # Grafana datasources (Mimir)
│   ├── datasources2.yml     # Grafana datasources (Prometheus)
│   ├── datasources3.yml     # Grafana datasources (Loki)
│   ├── prometheus1.yml      # Prometheus 1 configuration
│   └── prometheus2.yml      # Prometheus 2 configuration
│
└── kubernetes/              # Kubernetes (Minikube) deployment files
    ├── Grafana-Mimir-Loki-Jaeger.yaml  # All-in-one K8s manifest
    └── README.md            # Kubernetes deployment guide
```

---

## Deployment Options

This repository supports two deployment methods:

| Method | Directory | Use Case |
|--------|-----------|----------|
| **Docker Compose** | `docker-compose/` | Local development, quick testing |
| **Kubernetes** | `kubernetes/` | Production-like environment, Minikube |

---

## Option 1: Docker Compose Deployment

### Prerequisites

* Docker and Docker Compose installed
* Loki Docker driver plugin

### Installation

1. Install Loki docker driver
```bash
docker plugin install grafana/loki-docker-driver:latest --alias loki --grant-all-permissions
```

2. Before running docker compose, open `/docker-compose/consul/mimir.json` file and change all address configurations to your local IP.

3. Execute docker-compose
```bash
cd docker-compose
docker-compose up
```

### Docker Compose Services

| Service | Port | Description |
|---------|------|-------------|
| Grafana | 3000 | Visualization dashboards |
| Consul | 8500 | Service discovery UI |
| Jaeger | 16686 | Distributed tracing UI |
| MinIO | 9001 | S3-compatible storage console |
| Mimir Distributor | 8800 | Receives metrics (admin console) |
| Mimir Query Frontend | 8880 | Query endpoint |
| Prometheus 1 | 9090 | Metrics collection |
| Prometheus 2 | 9080 | Metrics collection |
| Loki | 3100 | Log aggregation |

---

## Option 2: Kubernetes Deployment (Minikube)

### Architecture

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

### Prerequisites

* Minikube installed and running
* kubectl configured to use Minikube context
* At least 8GB RAM allocated to Minikube

```bash
# Start Minikube with sufficient resources
minikube start --memory=8192 --cpus=4
```

### Deployment

1. Deploy the stack
```bash
kubectl apply -f kubernetes/Grafana-Mimir-Loki-Jaeger.yaml
```

2. Enable external access (in a separate terminal)
```bash
minikube tunnel
```

3. Check deployment status
```bash
# View all pods
kubectl -n mimir get pods

# View all services
kubectl -n mimir get svc
```

### Kubernetes Components

| Component | Type | Replicas | Description |
|-----------|------|----------|-------------|
| MinIO | Deployment | 1 | S3-compatible object storage |
| Consul | Deployment | 1 | Service discovery |
| Jaeger | Deployment | 1 | Distributed tracing |
| Loki | Deployment | 1 | Log aggregation |
| Grafana | Deployment | 1 | Visualization |
| Prometheus | Deployment | 2 | Metrics collection |
| Mimir Distributor | Deployment | 1 | Receives metrics |
| Mimir Ingester | StatefulSet | 3 | Writes to storage |
| Mimir Querier | Deployment | 2 | Executes queries |
| Mimir Query Frontend | Deployment | 1 | Query caching |
| Mimir Store Gateway | StatefulSet | 2 | Serves historical data |
| Mimir Compactor | StatefulSet | 3 | Compacts blocks |
| Mimir Ruler | Deployment | 1 | Recording/alerting rules |
| Mimir Alertmanager | Deployment | 1 | Alert handling |

### Cleanup

```bash
# Delete all resources
kubectl delete -f kubernetes/Grafana-Mimir-Loki-Jaeger.yaml

# Delete namespace
kubectl delete namespace mimir
```

> For more detailed Kubernetes deployment instructions, see `kubernetes/README.md`

---

## Accessing Services

### Web UIs

| Service | Docker Compose | Kubernetes | Credentials |
|---------|----------------|------------|-------------|
| Grafana | http://localhost:3000 | http://\<EXTERNAL-IP\>:3000 | admin / admin |
| Consul | http://localhost:8500 | http://\<EXTERNAL-IP\>:8500 | - |
| Jaeger | http://localhost:16686 | http://\<EXTERNAL-IP\>:16686 | - |
| MinIO | http://localhost:9001 | http://\<EXTERNAL-IP\>:9001 | minioadmin / minioadmin |
| Mimir Distributor | http://localhost:8800 | http://\<EXTERNAL-IP\>:8800 | - |
| Mimir Query Frontend | http://localhost:8880 | http://\<EXTERNAL-IP\>:8880 | - |

---

## Consul
* Consul is health checking service components of Grafana Mimir MSA
* You can use Consul UI ai http://localhost:8500/
![Consul UI](http://imageresizer-dev-serverlessdeploymentbucket-xapz1q6q9exe.s3-website-ap-northeast-1.amazonaws.com/gitpng/mimir_consul.PNG)

## Grafana Mimir MSA
* You can access Mimir distributor admin console at http://localhost:8800/
![Mimir Admin UI](http://imageresizer-dev-serverlessdeploymentbucket-xapz1q6q9exe.s3-website-ap-northeast-1.amazonaws.com/gitpng/mimir_console.PNG)

* You can check remote writing metric targets Mimir admin console at http://localhost:8800/distributor/all_user_stats
![Mimir User UI](http://imageresizer-dev-serverlessdeploymentbucket-xapz1q6q9exe.s3-website-ap-northeast-1.amazonaws.com/gitpng/mimir_user_stat_2.PNG)

* You can view the configuration services of Mimir MSA at http://localhost:8800/memberlist
![Mimir member UI](http://imageresizer-dev-serverlessdeploymentbucket-xapz1q6q9exe.s3-website-ap-northeast-1.amazonaws.com/gitpng/mimir_components.PNG)

* Push prometheus metrics using endpoint http://localhost:8800/api/v1/push and X-Scope-OrgId http header
* X-Scope-OrgId http header means tenant id, The tenant id identifies the target

* You can see the metric names that can be checked with curl below
```
curl -i -X GET \
   -H "X-Scope-OrgId:tenant id" \
 'http://localhost:8880/api/prom/api/v1/metadata'
```

* If you know how to use prometheus promql, then You can query metrics data with the http request below.
```
curl -i -X GET \
   -H "X-Scope-OrgId:tenant id" \
 'http://localhost:8880/api/prom/api/v1/{promql}'
```

* http://localhost:8880/api/prom is end point of Mimir query-frontend, add your promql query to it

* You can use minio ui at http://localhost:9001/
* Log in with account, password minioadmin
* mimir-block-docker bucket is storing metric data
* You can see minio S3 buckets status
![Mimir minio UI](http://imageresizer-dev-serverlessdeploymentbucket-xapz1q6q9exe.s3-website-ap-northeast-1.amazonaws.com/gitpng/mimir_minio.PNG)

## Loki
* Loki is collecting logs from docker logging driver and helps grafana visualize
* You can check the loki operation at http://localhost:3100/metrics

## Prometheus
* Prometheus remotely writes collected metrics to Mimir Distributor Metrics data.

## Grafana
* You can use grafana ui at http://localhost:3000/
* Enter account and password admin
* When the grafana container starts up, it creates a prometheus datasource by reading datasources yml files

### Observe the grafana dashboard UI

* You can see spring application metric data sent by Mimir query-frontend and graphs
![Grafana Mimir UI](http://imageresizer-dev-serverlessdeploymentbucket-xapz1q6q9exe.s3-website-ap-northeast-1.amazonaws.com/gitpng/mimir_grafana_metric_2.PNG)

* If you enter docker container name, You can see logs collected by Loki on grafana explore
![Grafana Explore UI](http://imageresizer-dev-serverlessdeploymentbucket-xapz1q6q9exe.s3-website-ap-northeast-1.amazonaws.com/gitpng/mimir_grafana_logs_2.PNG)

* You can see also logs on grafana dashboard
![Grafana Loki UI](http://imageresizer-dev-serverlessdeploymentbucket-xapz1q6q9exe.s3-website-ap-northeast-1.amazonaws.com/gitpng/mimir_grafana_logs_1.PNG)

## Jaeger
* Each Mimir MSA Service send data to Jaeger through Jaeger Agent
* You can search traces between Mimir MSA services at http://localhost:16686

* You can see the structure of calls between Mimir MSA services
![MSA Call](http://imageresizer-dev-serverlessdeploymentbucket-xapz1q6q9exe.s3-website-ap-northeast-1.amazonaws.com/gitpng/mimir_jaeger_topology_2.PNG)

---

## References

- [Grafana Mimir Documentation](https://grafana.com/docs/mimir/latest/)
- [Grafana Loki Documentation](https://grafana.com/docs/loki/latest/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [Consul Documentation](https://developer.hashicorp.com/consul/docs)
