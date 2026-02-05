# helm-charts/keycloak

[Keycloak](https://www.keycloak.org/) is an open source Identity and Access Management solution aimed at modern applications and services. It makes it easy to secure applications and services with little to no code.

## Introduction

This Helm chart deploys Keycloak on Kubernetes using the official Keycloak container image from quay.io/keycloak/keycloak.

### Database Options

The chart supports two database configuration methods:

1. **MariaDB Operator** (Recommended): Uses the official [mariadb-operator](https://github.com/mariadb-operator/mariadb-operator) to manage MariaDB instances
2. **External Database**: Connect to an existing database outside of this chart

**Important**: The MariaDB Operator must be pre-installed on your Kubernetes cluster if you choose option 1.

In short: 
```bash
helm repo add mariadb-operator https://helm.mariadb.com/mariadb-operator
helm install mariadb-operator-crds mariadb-operator/mariadb-operator-crds
helm install mariadb-operator mariadb-operator/mariadb-operator
```

### Key Features

- Official Keycloak image from Quay.io
- Automatic admin credentials management via Kubernetes secrets
- Supports both new MariaDB instances and existing MariaDB deployments managed by the operator
- StatefulSet deployment for clustering and data consistency
- Infinispan clustering support with JGroups for multi-replica deployments
- Optional Ingress configuration with proxy header support
- Health probes for high availability
- Headless service for clustering communication

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
helm install keycloak syntax3rror404/keycloak -n keycloak --create-namespace
```

**Note**: A simple install uses default values like `admin/admin` for credentials. **You must create your own values file for production use.**

### Installation with Custom Values

```bash
helm install keycloak syntax3rror404/keycloak -n keycloak --create-namespace -f myvalues.yaml
```

## Upgrade

```bash
helm upgrade keycloak syntax3rror404/keycloak -n keycloak
```

**Note**: The admin credentials are stored as secrets and will be preserved during upgrades.

## Uninstall

```bash
# Remove the Helm release
helm uninstall keycloak -n keycloak

# Delete the namespace (optional - this will remove all resources including PVCs)
kubectl delete ns keycloak
```

## Configuration

### Core Configuration Values

| Parameter | Default | Description |
|-----------|---------|-------------|
| `replicaCount` | `1` | Number of Keycloak replicas (use >1 for clustering) |
| `image.repository` | `quay.io/keycloak/keycloak` | Official Keycloak container image |
| `image.tag` | `26.4.7` | Keycloak version |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy |
| `admin.username` | `admin` | Keycloak admin username |
| `admin.password` | `admin` | Keycloak admin password (change in production!) |
| `updateStrategy` | `RollingUpdate` | StatefulSet update strategy |

### Keycloak Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `keycloak.proxy.headers` | `xforwarded` | Proxy headers mode (xforwarded, forwarded) |
| `keycloak.hostname.strict` | `false` | Enable strict hostname verification |
| `keycloak.cache.type` | `ispn` | Cache type (ispn for clustering, local for single node) |
| `keycloak.args` | `["start"]` | Keycloak args list |
| `keycloak.httpPath` | `"/"` | Keycloak used http path |

### Database Configuration - MariaDB Operator

| Parameter | Default | Description |
|-----------|---------|-------------|
| `mariadbOperator.enabled` | `false` | Enable MariaDB Operator integration |
| `mariadbOperator.database` | `keycloak` | Database name to create |
| `mariadbOperator.username` | `keycloak` | Database username to create |
| `mariadbOperator.password` | `keycloakpass` | Database password |
| `mariadbOperator.storageClass` | `longhorn` | Storage class for MariaDB |
| `mariadbOperator.useExisting.enabled` | `false` | Use existing MariaDB instance |
| `mariadbOperator.useExisting.namespace` | - | Namespace of existing MariaDB |
| `mariadbOperator.useExisting.mariaDbRef` | - | Name of existing MariaDB CR |

### Database Configuration - External Database

| Parameter | Default | Description |
|-----------|---------|-------------|
| `database.type` | `mariadb` | Database type (mariadb, postgres) |
| `database.name` | `keycloak` | Database name |
| `database.username` | `keycloak` | Database username |
| `database.password` | `keycloakpass` | Database password |
| `database.host` | `""` | Database host (auto-set with operator) |
| `database.port` | `3306` | Database port |
| `database.dbUrl` | `3306` | Database db url |

### Service Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `service.type` | `ClusterIP` | Service type |
| `service.port` | `8080` | Service port |

### Ingress Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `ingress.enabled` | `false` | Enable Ingress |
| `ingress.className` | `""` | Ingress class name (e.g., nginx) |
| `ingress.annotations` | `{}` | Ingress annotations |
| `ingress.hosts` | See values.yaml | Ingress hosts configuration |
| `ingress.tls` | `[]` | TLS configuration |

### Health Probes

| Parameter | Default | Description |
|-----------|---------|-------------|
| `livenessProbe.httpGet.path` | `/health/live` | Liveness probe path |
| `livenessProbe.httpGet.port` | `9000` | Liveness probe port |
| `livenessProbe.periodSeconds` | `10` | Probe check interval |
| `readinessProbe.httpGet.path` | `/health/ready` | Readiness probe path |
| `readinessProbe.httpGet.port` | `9000` | Readiness probe port |
| `readinessProbe.periodSeconds` | `10` | Probe check interval |

## Configuration Examples

### Example 1: Single Instance with New MariaDB

```yaml
# myvalues.yaml
mariadbOperator:
  enabled: true
  database: keycloak
  username: keycloak
  password: "SecurePassword123!"
  storageClass: longhorn

admin:
  username: admin
  password: "ChangeMe123!"

keycloak:
  proxy:
    headers: xforwarded
  cache:
    type: local  # Single instance doesn't need clustering
  hostname:
    strict: false

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: auth.mycompany.com
      paths:
        - path: /
          pathType: Prefix
  tls: 
    - secretName: keycloak-tls
      hosts:
        - auth.mycompany.com
```

### Example 2: Clustered Setup with Existing MariaDB

```yaml
# myvalues.yaml
replicaCount: 3  # For high availability

mariadbOperator:
  enabled: true
  database: keycloak
  username: keycloak
  password: "SecurePassword123!"
  useExisting:
    enabled: true
    namespace: database
    mariaDbRef: mariadb-shared

admin:
  username: admin
  password: "SuperSecure456!"

keycloak:
  proxy:
    headers: xforwarded
  cache:
    type: ispn  # Infinispan for clustering
  hostname:
    strict: true

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "route"
  hosts:
    - host: sso.mycompany.com
      paths:
        - path: /
          pathType: Prefix
  tls: 
    - secretName: keycloak-tls
      hosts:
        - sso.mycompany.com

resources:
  limits:
    cpu: 2000m
    memory: 2Gi
  requests:
    cpu: 500m
    memory: 1Gi
```

### Example 3: External PostgreSQL Database

```yaml
# myvalues.yaml
mariadbOperator:
  enabled: false

database:
  type: postgres
  name: keycloak_prod
  username: keycloak_user
  password: "ExternalDbPassword!"
  host: postgres.database.svc.cluster.local
  port: 5432

admin:
  username: admin
  password: "AdminPassword123!"

keycloak:
  proxy:
    headers: xforwarded
  cache:
    type: ispn

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: keycloak.example.com
      paths:
        - path: /
          pathType: Prefix
```

### Example 4: Development Setup

```yaml
# dev-values.yaml
replicaCount: 1

mariadbOperator:
  enabled: true
  database: keycloak_dev
  username: keycloak
  password: "devpass"
  storageClass: local-path

admin:
  username: admin
  password: admin

keycloak:
  proxy:
    headers: xforwarded
  cache:
    type: local
  hostname:
    strict: false

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: keycloak.local
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 256Mi
```

## Architecture

### StatefulSet Deployment

The chart uses a StatefulSet to ensure:
- Stable network identity for each pod
- Ordered deployment and scaling
- Consistent pod naming for clustering

### Clustering with Infinispan

When `replicaCount > 1` and `keycloak.cache.type: ispn`:
- JGroups is used for cluster communication
- Headless service enables pod-to-pod communication
- Ports 7800 (jgroups) and 57800 (jgroups-fd) are exposed
- Each pod discovers others using Kubernetes DNS

### Services

Two services are created:
1. **Regular Service**: External access to Keycloak
2. **Headless Service**: Internal cluster communication

### Admin Credentials Management

Admin credentials are stored as Kubernetes secrets:
- Secret name: `<release-name>-admin`
- Contains: `username` and `password` keys
- Preserved during upgrades
- Can be rotated by deleting the secret and upgrading

### Database Connection

**With MariaDB Operator**:
- Automatic service discovery: `mariadb-<release-name>-internal.<namespace>.svc:3306`
- Database, User, and Grant resources are created automatically

**With External Database**:
- Configure connection via `database.*` values
- Connection string built from provided values

## Troubleshooting

### Database Connection Issues

Check the database connectivity:

```bash
# Check pod logs
kubectl logs -n keycloak -l app.kubernetes.io/name=keycloak

# Verify MariaDB Operator resources
kubectl get mariadb,database,user,grant -n keycloak

# Test database connection from pod
kubectl exec -n keycloak keycloak-0 -- sh -c "nc -zv <db-host> 3306"
```

### Clustering Issues

Verify cluster formation:

```bash
# Check if all pods are ready
kubectl get pods -n keycloak

# Check JGroups logs
kubectl logs -n keycloak keycloak-0 | grep -i jgroups

# Verify headless service
kubectl get svc -n keycloak
kubectl get endpoints -n keycloak
```

### Admin Password Reset

To reset the admin password:

```bash
# Delete the admin secret
kubectl delete secret keycloak-admin -n keycloak

# Update your values file with new password and upgrade
helm upgrade keycloak syntax3rror404/keycloak -n keycloak -f myvalues.yaml
```

### Proxy/Ingress Issues

If Keycloak shows incorrect URLs:

1. Verify `keycloak.proxy.headers` matches your ingress configuration
2. Check ingress annotations are correct
3. Ensure TLS is properly configured if using HTTPS

```bash
# Check ingress configuration
kubectl describe ingress -n keycloak

# Test from inside cluster
kubectl run -n keycloak test --rm -it --image=curlimages/curl -- sh
curl -v http://keycloak:8080
```

### Health Check Failures

If pods are not becoming ready:

```bash
# Check health endpoint manually
kubectl exec -n keycloak keycloak-0 -- curl http://localhost:9000/health/ready

# Check startup logs
kubectl logs -n keycloak keycloak-0 --tail=100

# Verify database migrations completed
kubectl logs -n keycloak keycloak-0 | grep -i migration
```

## Production Recommendations

### Security

1. **Change default passwords** in your values file
2. **Use Kubernetes secrets** for sensitive data
3. **Enable TLS** via Ingress with cert-manager
4. **Configure pod security contexts**:
   ```yaml
   podSecurityContext:
     fsGroup: 1000
     runAsNonRoot: true
   
   securityContext:
     allowPrivilegeEscalation: false
     runAsUser: 1000
     capabilities:
       drop:
         - ALL
   ```

### High Availability

1. **Run multiple replicas** (`replicaCount: 3`)
2. **Use Infinispan clustering** (`keycloak.cache.type: ispn`)
3. **Configure pod anti-affinity**:
   ```yaml
   affinity:
     podAntiAffinity:
       preferredDuringSchedulingIgnoredDuringExecution:
         - weight: 100
           podAffinityTerm:
             labelSelector:
               matchExpressions:
                 - key: app.kubernetes.io/name
                   operator: In
                   values:
                     - keycloak
             topologyKey: kubernetes.io/hostname
   ```

### Performance

1. **Set appropriate resource limits**:
   ```yaml
   resources:
     limits:
       cpu: 2000m
       memory: 2Gi
     requests:
       cpu: 500m
       memory: 1Gi
   ```

2. **Configure database connection pooling** (if using external DB)

3. **Use persistent storage** for themes and extensions (add volumeMounts)

### Monitoring

1. Enable metrics endpoint (requires additional configuration)
2. Use Prometheus ServiceMonitor (if using Prometheus Operator)
3. Monitor health endpoints via your monitoring solution

## Migration from Other Deployments

### From Docker Compose

1. Export your database
2. Install MariaDB Operator on Kubernetes
3. Deploy this chart with `mariadbOperator.enabled: true`
4. Import your database to the new MariaDB instance
5. Update your DNS/URLs to point to the new Ingress

### From Bitnami Keycloak Chart

1. Backup your existing database
2. Note your current configuration and secrets
3. Install MariaDB Operator
4. Deploy this chart with equivalent configuration
5. Restore database if needed

## Advanced Configuration

### Custom Themes

To add custom themes, create a PVC and mount it:

```yaml
volumeMounts:
  - name: keycloak-themes
    mountPath: /opt/keycloak/themes/custom

volumes:
  - name: keycloak-themes
    persistentVolumeClaim:
      claimName: keycloak-themes-pvc
```

### Custom Providers

Similar to themes, mount your provider JARs:

```yaml
volumeMounts:
  - name: keycloak-providers
    mountPath: /opt/keycloak/providers

volumes:
  - name: keycloak-providers
    persistentVolumeClaim:
      claimName: keycloak-providers-pvc
```

### Environment Variables

Add custom environment variables via values:

```yaml
# Note: This requires extending the StatefulSet template
env:
  - name: KC_LOG_LEVEL
    value: INFO
  - name: KC_FEATURES
    value: "preview"
```

## Support and Contributing

- **Issues**: https://github.com/Syntax3rror404/helm-charts/issues
- **Source Code**: https://github.com/Syntax3rror404/helm-charts
- **Keycloak Documentation**: https://www.keycloak.org/documentation
- **MariaDB Operator**: https://github.com/mariadb-operator/mariadb-operator
