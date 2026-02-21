# helm-charts/snipeit

[Snipe-IT](http://www.snipeitapp.com) is open source software. Transparency, security and oversight is at the heart of everything we do. No vendor lock-in again, ever.

## Introduction

This Helm chart deploys Snipe-IT on Kubernetes using the official Snipe-IT container image. It is secured by default with pod / security context.

### Database Options

The chart supports two database configuration methods:

1. **MariaDB Operator** (Recommended): Uses the [mariadb-operator](https://github.com/mariadb-operator/mariadb-operator) to manage MariaDB instances
2. **External Database**: Connect to an existing database outside of this chart

**Important**: The MariaDB Operator must be pre-installed on your Kubernetes cluster if you choose option 1.

In short: 
```bash
helm repo add mariadb-operator https://helm.mariadb.com/mariadb-operator
helm install mariadb-operator-crds mariadb-operator/mariadb-operator-crds
helm install mariadb-operator mariadb-operator/mariadb-operator
```

### Key Features

- Automatic Laravel APP_KEY generation and secure storage as Kubernetes secret
- Supports both new MariaDB instances and existing MariaDB deployments managed by the operator
- Persistent storage for application data, sessions, and backups
- StatefulSet deployment for data consistency
- Optional Ingress configuration
- Health probes for high availability

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- **MariaDB Operator** installed (if using `mariadbOperator.enabled: true`)
  - Installation guide: https://github.com/mariadb-operator/mariadb-operator

## Installation

### Quick Start

```bash
# Add the Helm repository
helm repo add syntax3rror404 https://syntax3rror404.github.io/helm-charts/charts

# Install with default values
helm install snipeit syntax3rror404/snipeit -n snipeit --create-namespace
```

**Note**: A simple install uses default values like `example.com` for URLs and default passwords. **You must create your own values file for production use.**

### Installation with Custom Values

```bash
helm install snipeit syntax3rror404/snipeit -n snipeit --create-namespace -f myvalues.yaml
```

## Upgrade

```bash
helm upgrade snipeit syntax3rror404/snipeit -n snipeit
```

**Note**: The APP_KEY secret is automatically preserved during upgrades and will not be overwritten.

## Uninstall

```bash
# Remove the Helm release
helm uninstall snipeit -n snipeit

# Delete the namespace (optional - this will remove all resources including PVCs)
kubectl delete ns snipeit
```

## Configuration

### Core Configuration Values

| Parameter | Default | Description |
|-----------|---------|-------------|
| `snipeit.config.url` | `http://snipeit.example.com` | Application URL for Snipe-IT |
| `snipeit.config.timezone` | `Europe/Berlin` | Application timezone |
| `snipeit.config.locale` | `en` | Application locale |
| `snipeit.config.env` | `production` | Application environment |
| `snipeit.config.debug` | `false` | Enable debug mode |
| `snipeit.image.repository` | `docker.io/snipe/snipe-it` | Container image repository |
| `snipeit.image.tag` | `v8.3.4-alpine` | Container image tag |
| `snipeit.persistence.size` | `2Gi` | Persistent volume size |
| `snipeit.persistence.storageClass` | `longhorn` | Storage class for PVC |
| `replicaCount` | `1` | Number of replicas |

### Database Configuration - MariaDB Operator

| Parameter | Default | Description |
|-----------|---------|-------------|
| `mariadbOperator.enabled` | `false` | Enable MariaDB Operator integration |
| `mariadbOperator.database` | `snipeit` | Database name to create |
| `mariadbOperator.username` | `snipeit` | Database username to create |
| `mariadbOperator.password` | `snipeitdbsecret` | Database password |
| `mariadbOperator.storageClass` | `longhorn` | Storage class for MariaDB |
| `mariadbOperator.useExisting.enabled` | `false` | Use existing MariaDB instance |
| `mariadbOperator.useExisting.namespace` | - | Namespace of existing MariaDB |
| `mariadbOperator.useExisting.mariaDbRef` | - | Name of existing MariaDB CR |

### Database Configuration - External Database

| Parameter | Default | Description |
|-----------|---------|-------------|
| `externalDB.enabled` | `false` | Use external database |
| `externalDB.username` | - | Database username (supports valueFrom) |
| `externalDB.password` | - | Database password (supports valueFrom) |
| `externalDB.database` | - | Database name (supports valueFrom) |
| `externalDB.host` | - | Database host (supports valueFrom) |

### Ingress Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `ingress.enabled` | `false` | Enable Ingress |
| `ingress.className` | `""` | Ingress class name (e.g., nginx) |
| `ingress.annotations` | `{}` | Ingress annotations |
| `ingress.hosts` | See values.yaml | Ingress hosts configuration |
| `ingress.tls` | `[]` | TLS configuration |

## Configuration Examples

### Example 1: New MariaDB Instance with Ingress

```yaml
# myvalues.yaml
mariadbOperator:
  enabled: true
  database: snipeit
  username: snipeit
  password: "changeMe123!"
  storageClass: longhorn

snipeit:
  config:
    url: https://snipeit.mycompany.com
    timezone: Europe/Berlin
    locale: de
  persistence:
    size: 20Gi
    storageClass: longhorn

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: snipeit.mycompany.com
      paths:
        - path: /
          pathType: Prefix
  tls: 
    - secretName: snipeit-tls
      hosts:
        - snipeit.mycompany.com
```

### Example 2: Using Existing MariaDB Instance

```yaml
# myvalues.yaml
mariadbOperator:
  enabled: true
  database: snipeit
  username: snipeit
  password: "changeMe123!"
  useExisting:
    enabled: true
    namespace: database
    mariaDbRef: mariadb-shared

snipeit:
  config:
    url: https://assets.mycompany.com
  persistence:
    size: 10Gi
    storageClass: ceph-block

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: assets.mycompany.com
      paths:
        - path: /
          pathType: Prefix
```

### Example 3: External Database

```yaml
# myvalues.yaml
externalDB:
  enabled: true
  username:
    value: snipeit_user
  password:
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password
  database:
    value: snipeit_production
  host:
    value: mysql.database.svc.cluster.local

snipeit:
  config:
    url: https://snipeit.example.com
  persistence:
    size: 5Gi
```

## Architecture

### StatefulSet Deployment

The chart uses a StatefulSet to ensure:
- Stable network identity
- Persistent storage attached to each pod
- Ordered deployment and scaling

### Persistent Volumes

Three volume mounts are configured by default:
- `/var/lib/snipeit` - Main application data
- `/var/www/html/storage/framework/sessions` - PHP sessions
- `/var/www/html/storage/app/backups` - Application backups

### APP_KEY Management

The Laravel APP_KEY is automatically generated on first installation:
1. A pre-upgrade Job generates a new key using `php artisan key:generate`
2. The key is stored as a Kubernetes Secret
3. Upgrades check for the existing secret and preserve it
4. Manual intervention is required if you need to regenerate the key

## Troubleshooting

### Database Connection Issues

Check the database connectivity:

```bash
# Check pod logs
kubectl logs -n snipeit -l app.kubernetes.io/name=snipeit

# Verify MariaDB Operator resources
kubectl get mariadb,database,user,grant -n snipeit
```

### APP_KEY Issues

If you need to regenerate the APP_KEY:

```bash
# Delete the existing secret
kubectl delete secret laravel-app-key -n snipeit

# Trigger a Helm upgrade to regenerate
helm upgrade snipeit syntax3rror404/snipeit -n snipeit
```

### Persistent Volume Issues

Check PVC status:

```bash
kubectl get pvc -n snipeit
kubectl describe pvc <pvc-name> -n snipeit
```

## Migration from Bitnami MariaDB

If you're migrating from the previous version using Bitnami MariaDB as a dependency:

1. **Backup your data** before proceeding
2. Export your existing database
3. Install MariaDB Operator on your cluster
4. Update your values file to use `mariadbOperator.enabled: true`
5. Deploy the new chart version
6. Import your database backup

## Support and Contributing

- **Issues**: https://github.com/Syntax3rror404/helm-charts/issues
- **Source Code**: https://github.com/Syntax3rror404/helm-charts
- **Snipe-IT Project**: https://github.com/snipe/snipe-it
