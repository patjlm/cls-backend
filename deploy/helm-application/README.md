# CLS Backend Application Helm Chart

This Helm chart deploys the CLS Backend application to Kubernetes.

## Prerequisites

- Kubernetes cluster (GKE with Workload Identity)
- Google Cloud resources deployed using the `helm-cloud-resources` chart
- Container image available in a registry

## Resources Created

1. **Namespace** - Application namespace (`cls-system`)
2. **ServiceAccount** - Kubernetes service account with Workload Identity annotations
3. **ConfigMap** - Application configuration including DATABASE_URL and GOOGLE_CLOUD_PROJECT
4. **Deployment** - CLS Backend application pods (3 replicas by default) with Cloud SQL Proxy sidecar
5. **Service** - ClusterIP service for internal access
6. **Migration Job** - Helm pre-install/pre-upgrade hook for database schema migrations

## Authentication Architecture

**Zero stored passwords.** The application uses GCP IAM for all authentication:

| Component | Auth Method | Details |
|-----------|------------|---------|
| **Application** | IAM (passwordless) | Cloud SQL Proxy `--auto-iam-authn` + Workload Identity |
| **Migration job** | Ephemeral postgres password | `gcloud sql users set-password` via Admin API, password in tmpfs, forgotten on pod exit |

### How the migration job works

1. **copy-migrations** init container: copies SQL files from the app image
2. **set-postgres-password** init container: uses `gcloud sql users set-password` to set a random postgres password via Cloud SQL Admin API (authenticates via Workload Identity — no stored credentials)
3. **cloud-sql-proxy** sidecar: provides encrypted connection to Cloud SQL
4. **migration** container: reads ephemeral password from tmpfs, connects as postgres, runs DDL + GRANTs for the IAM user
5. Pod exits, tmpfs is gone, password forgotten

## Installation

### 1. Configure Values

```yaml
gcp:
  project: "your-gcp-project-id"

image:
  repository: "quay.io/your-org/cls-backend"
  tag: "latest"

database:
  instanceName: "cls-backend-db"  # Must match cloud-resources chart
```

### 2. Install the Chart

```bash
helm install cls-backend-app ./deploy/helm-application \
  --values values.yaml \
  --create-namespace
```

## Values Reference

### Required Values

| Parameter | Description | Default |
|-----------|-------------|---------|
| `gcp.project` | **Required** GCP Project ID | `""` |

### Database Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `database.type` | Connection type (`cloud-sql` or `external`) | `"cloud-sql"` |
| `database.instanceName` | Cloud SQL instance name | `""` |
| `database.databaseName` | Database name | `""` (defaults to `cls_backend`) |
| `database.cloudSql.enableProxy` | Enable Cloud SQL Proxy sidecar | `true` |
| `database.cloudSql.proxyImage` | Proxy image | `"gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.8.0"` |
| `database.cloudSql.proxyPort` | Proxy port | `5432` |

### Application Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `namespace.name` | Kubernetes namespace | `"cls-system"` |
| `image.repository` | Container image repository | `"quay.io/apahim/cls-backend"` |
| `image.tag` | Container image tag | `"latest"` |
| `deployment.replicas` | Number of replicas | `3` |
| `config.port` | HTTP server port | `8080` |
| `config.environment` | Environment name | `"production"` |
| `config.logLevel` | Log level | `"info"` |
| `config.disableAuth` | Disable authentication | `false` |
| `config.metricsEnabled` | Enable Prometheus metrics | `true` |
| `config.metricsPort` | Metrics port | `8081` |

### Service Account Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `serviceAccount.name` | Kubernetes service account name | `"cls-backend"` |
| `serviceAccount.gcpServiceAccountName` | GCP service account name | `"cls-backend"` |
| `serviceAccount.workloadIdentity` | Enable Workload Identity | `true` |

## Configuration Consistency

Values in this chart must match those used in the `helm-cloud-resources` chart:

- `database.instanceName` → Cloud SQL instance name
- `database.databaseName` → Cloud SQL database name
- `pubsub.clusterEventsTopic` → Pub/Sub topic name
- `serviceAccount.gcpServiceAccountName` → GCP service account name

## Post-Installation

```bash
# Check pods
kubectl get pods -n cls-system

# Check logs
kubectl logs -f deployment/cls-backend-app -n cls-system

# Port forward and test
kubectl port-forward service/cls-backend-app 8080:80 -n cls-system
curl http://localhost:8080/health
curl http://localhost:8080/api/v1/clusters
```

## Monitoring

The application exposes Prometheus metrics on port 8081:

```bash
kubectl port-forward service/cls-backend-app 8081:8081 -n cls-system
curl http://localhost:8081/metrics
```

## Troubleshooting

### Database Connection Issues

```bash
# Check Cloud SQL Proxy sidecar logs
kubectl logs deployment/cls-backend-app -c cloud-sql-proxy -n cls-system

# Check IAM user exists in Cloud SQL
gcloud sql users list --instance=cls-backend-db --project=YOUR_PROJECT_ID
```

### Workload Identity Issues

```bash
# Verify Workload Identity binding
gcloud iam service-accounts get-iam-policy \
  cls-backend@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

## Uninstallation

```bash
helm uninstall cls-backend-app
```

Note: This only removes the Kubernetes resources. Google Cloud resources (database, Pub/Sub, etc.) are managed by the cloud resources chart.
