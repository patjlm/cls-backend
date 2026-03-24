# CLS Backend Cloud Resources Helm Chart

This Helm chart deploys the Google Cloud resources required for the CLS Backend using Google Config Connector.

## Prerequisites

- Google Kubernetes Engine (GKE) cluster with [Config Connector](https://cloud.google.com/config-connector/docs/overview) installed
- Appropriate IAM permissions to create Cloud SQL instances, Pub/Sub topics, and service accounts

## Resources Created

This chart creates the following Google Cloud resources:

1. **GCP Service Enablement** - Automatically enables required GCP APIs
2. **Cloud SQL PostgreSQL Instance** - Database with IAM authentication enabled (`cloudsql.iam_authentication=on`)
3. **Cloud SQL Database** - `cls_backend` database within the instance
4. **Cloud SQL IAM User** - IAM service account user (`type: CLOUD_IAM_SERVICE_ACCOUNT`) for passwordless application auth
5. **Pub/Sub Topics** - `cluster-events` and `nodepool-events` topics
6. **IAM Service Account** - Service account for the CLS Backend application
7. **IAM Policy Bindings** - Permissions for Cloud SQL, Pub/Sub, and Workload Identity

## Authentication Architecture

This chart uses **GCP IAM database authentication** — no stored passwords:

- **Application**: Authenticates via IAM service account through Cloud SQL Proxy with `--auto-iam-authn`
- **Migration job**: Uses ephemeral postgres password set via Cloud SQL Admin API (`gcloud sql users set-password`), forgotten after job completes

### IAM Roles Granted

| Role | Purpose |
|------|---------|
| `roles/cloudsql.admin` | Migration job: set ephemeral postgres password via Admin API |
| `roles/cloudsql.client` | Cloud SQL Proxy connectivity |
| `roles/cloudsql.instanceUser` | IAM database authentication |
| `roles/pubsub.editor` | Pub/Sub topic publishing |

## Installation

### 1. Configure Values

Create a `values.yaml` file with your project configuration:

```yaml
gcp:
  project: "your-gcp-project-id"
  region: "us-central1"

# Optional: Customize database configuration
database:
  instance:
    name: "cls-backend-db"
    tier: "db-custom-2-4096"  # 2 vCPU, 4GB RAM
    diskSize: "50"            # 50GB
```

### 2. Install the Chart

```bash
helm install cls-backend-cloud-resources ./deploy/helm-cloud-resources \
  --values values.yaml \
  --namespace config-connector \
  --create-namespace
```

## Values Reference

| Parameter | Description | Default |
|-----------|-------------|---------|
| `gcp.project` | **Required** GCP Project ID | `""` |
| `gcp.region` | GCP region for resources | `"us-central1"` |
| `database.instance.name` | Cloud SQL instance name | `"cls-backend-db"` |
| `database.instance.tier` | Cloud SQL instance tier | `"db-custom-1-3840"` |
| `database.instance.diskSize` | Disk size in GB | `"20"` |
| `database.instance.diskType` | Disk type | `"PD_SSD"` |
| `database.instance.version` | PostgreSQL version | `"POSTGRES_15"` |
| `database.database.name` | Database name | `"cls_backend"` |
| `pubsub.clusterEventsTopic` | Cluster events Pub/Sub topic | `"cluster-events"` |
| `pubsub.nodepoolEventsTopic` | NodePool events Pub/Sub topic | `"nodepool-events"` |
| `serviceAccount.name` | Service account name | `"cls-backend"` |
| `services.enabled` | Enable GCP APIs via Config Connector | `true` |

## Post-Installation

```bash
# Check Cloud SQL instance
gcloud sql instances list --project=YOUR_PROJECT_ID

# Check Pub/Sub topic
gcloud pubsub topics list --project=YOUR_PROJECT_ID

# Check service account
gcloud iam service-accounts list --project=YOUR_PROJECT_ID

# Check Config Connector resource status
kubectl get sqlinstance,sqldatabase,sqluser,pubsubtopic,iamserviceaccount -n config-connector
```

## Connection Information

After deployment, the application chart uses:

- **Database Instance**: Value from `database.instance.name`
- **Database Name**: `cls_backend`
- **Database User**: IAM service account (`serviceAccount.name@gcp.project.iam`) — no password
- **Pub/Sub Topics**: `cluster-events`, `nodepool-events`
- **Service Account**: `{serviceAccount.name}@{gcp.project}.iam.gserviceaccount.com`

## Important Notes

⚠️ **Security**: Review `authorizedNetworks` and `requireSsl` before deploying to production. Defaults allow all IPs without SSL (suitable for development only).

⚠️ **Cost**: Cloud SQL instances incur costs even when not in use. Enable `deletionProtectionEnabled` in production.

## Troubleshooting

### Config Connector Issues

```bash
# Check Config Connector status
kubectl get pods -n cnrm-system

# Check resource status
kubectl get sqlinstance,sqldatabase,sqluser,pubsubtopic,iamserviceaccount -n config-connector
```

## Uninstallation

```bash
helm uninstall cls-backend-cloud-resources --namespace cls-system
```

⚠️ **Warning**: This will permanently delete your Cloud SQL instance and all data. Backup your data before uninstalling.
