# Kubernetes Self-Hosting Guide

Deploy Z3rno on Kubernetes for high availability and auto-scaling using the official Helm chart.

## Prerequisites

| Requirement | Version |
|---|---|
| Kubernetes | 1.27+ |
| Helm | 3.x |
| kubectl | Configured for your cluster |

You also need a PostgreSQL database with the `pgvector` and `Apache AGE` extensions. Use a managed service (RDS, Cloud SQL, Neon, AlloyDB) or deploy one in-cluster.

## Install with Helm

### Add the Helm repository

```bash
helm repo add z3rno https://the-ai-project-co.github.io/z3rno-helm
helm repo update
```

### Install with default values

```bash
helm install z3rno z3rno/z3rno \
  --namespace z3rno-system \
  --create-namespace
```

### Or install from source

```bash
git clone https://github.com/the-ai-project-co/z3rno-helm.git
helm install z3rno ./z3rno-helm/charts/z3rno \
  --namespace z3rno-system \
  --create-namespace
```

## Verify deployment

```bash
# Check pods are running
kubectl get pods -n z3rno-system

# Check services
kubectl get svc -n z3rno-system

# Wait for rollout to complete
kubectl rollout status deployment/z3rno-server -n z3rno-system

# Test the health endpoint (via port-forward)
kubectl port-forward svc/z3rno-server 8000:8000 -n z3rno-system &
curl http://localhost:8000/v1/health
```

## Configure values

Override defaults by creating a custom values file:

```bash
helm install z3rno z3rno/z3rno \
  --namespace z3rno-system \
  --create-namespace \
  -f my-values.yaml
```

### Key configuration values

| Key | Default | Description |
|---|---|---|
| `server.replicas` | `2` | Number of API server pods |
| `server.image.repository` | `ghcr.io/the-ai-project-co/z3rno-server` | Server image |
| `server.image.tag` | `latest` | Server image tag |
| `server.resources.requests.memory` | `256Mi` | Memory request per pod |
| `server.resources.limits.memory` | `512Mi` | Memory limit per pod |
| `server.resources.requests.cpu` | `250m` | CPU request per pod |
| `server.resources.limits.cpu` | `500m` | CPU limit per pod |
| `worker.enabled` | `true` | Enable Celery worker deployment |
| `worker.replicas` | `1` | Number of worker pods |
| `valkey.enabled` | `true` | Deploy bundled Valkey |
| `valkey.persistence.size` | `1Gi` | Valkey PVC size |
| `ingress.enabled` | `false` | Enable Ingress resource |
| `autoscaling.enabled` | `false` | Enable HPA for server pods |

See the full [`values.yaml`](https://github.com/the-ai-project-co/z3rno-helm/blob/main/charts/z3rno/values.yaml) for all options.

### External database

For production, point Z3rno at an external PostgreSQL instance. First, create a secret with the database password:

```bash
kubectl create secret generic z3rno-db \
  --from-literal=password='your-db-password' \
  --namespace z3rno-system
```

Then configure your values file:

```yaml
# values-prod.yaml
externalDatabase:
  enabled: true
  host: your-rds-instance.region.rds.amazonaws.com
  port: 5432
  name: z3rno
  user: z3rno
  existingSecret: z3rno-db
  secretKey: password
```

### External Redis

Replace the bundled Valkey with a managed Redis service:

```yaml
valkey:
  enabled: false

externalRedis:
  enabled: true
  host: your-elasticache.region.cache.amazonaws.com
  port: 6379
  existingSecret: z3rno-redis
  secretKey: redis-password
```

### Replicas

Set the number of server and worker replicas:

```yaml
server:
  replicas: 3

worker:
  replicas: 2
```

### Ingress with TLS

Enable Ingress with TLS using cert-manager:

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: z3rno.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: z3rno-tls
      hosts:
        - z3rno.example.com
```

This requires an Ingress controller (nginx, Traefik, etc.) and cert-manager installed in your cluster.

### Secrets

For production, store secrets in a Kubernetes Secret and reference it:

```bash
kubectl create secret generic z3rno-secrets \
  --from-literal=database-url='postgresql+asyncpg://z3rno:pass@host:5432/z3rno' \
  --from-literal=redis-url='redis://host:6379/0' \
  --from-literal=openai-api-key='sk-your-key' \
  --from-literal=z3rno-api-key='z3rno_sk_prod_your-key' \
  --namespace z3rno-system
```

```yaml
secrets:
  existingSecret: z3rno-secrets
```

## Scaling

### Manual scaling

```bash
kubectl scale deployment z3rno-server --replicas=5 -n z3rno-system
```

### Horizontal Pod Autoscaler

Enable automatic scaling based on CPU and memory usage:

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

When `autoscaling.enabled` is `true`, the `server.replicas` value is ignored. The HPA manages replica count.

### Pod Disruption Budget

Ensure availability during voluntary disruptions (node drains, upgrades):

```yaml
podDisruptionBudget:
  enabled: true
  minAvailable: 1
```

## Upgrading

### Upgrade to a new version

```bash
# Update the Helm repo
helm repo update

# Upgrade the release
helm upgrade z3rno z3rno/z3rno \
  --namespace z3rno-system \
  -f my-values.yaml

# Check rollout status
kubectl rollout status deployment/z3rno-server -n z3rno-system
```

### Roll back if needed

```bash
# List release history
helm history z3rno -n z3rno-system

# Roll back to a previous revision
helm rollback z3rno <revision> -n z3rno-system
```

### Upgrade checklist

1. **Review release notes** for breaking changes.
2. **Back up the database** before upgrading.
3. **Run `helm upgrade`** with your values file.
4. **Monitor rollout** -- watch pod status and logs.
5. **Verify health** via the `/v1/health` endpoint.

## Monitoring

Z3rno exposes Prometheus metrics at `/metrics`. If you use the Prometheus Operator, create a `ServiceMonitor`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: z3rno
  namespace: z3rno-system
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: z3rno
      app.kubernetes.io/component: server
  endpoints:
    - port: http
      interval: 15s
      path: /metrics
```

For standalone Prometheus, add a scrape config:

```yaml
scrape_configs:
  - job_name: z3rno
    kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names: [z3rno-system]
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_label_app_kubernetes_io_name]
        regex: z3rno
        action: keep
```

## Troubleshooting

### Pods stuck in CrashLoopBackOff

Check pod logs:

```bash
kubectl logs -n z3rno-system deployment/z3rno-server --tail=100
```

Common causes: missing database connection, invalid API key, or missing extensions in PostgreSQL.

### Cannot connect to external database

Verify the secret exists and the pod can reach the database host:

```bash
kubectl get secret z3rno-db -n z3rno-system
kubectl run pg-test --rm -it --image=postgres:17 -- psql "postgresql://z3rno:pass@host:5432/z3rno"
```

### Ingress not working

Confirm your Ingress controller is running and the Ingress resource was created:

```bash
kubectl get ingress -n z3rno-system
kubectl describe ingress z3rno -n z3rno-system
```
