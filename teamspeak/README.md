# helm-charts/teamspeak

[TeamSpeak](https://www.teamspeak.com/) is a voice-over-Internet Protocol (VoIP) application for audio communication between users on a chat channel. TeamSpeak users typically use headphones with a microphone.

## Introduction

This Helm chart deploys a TeamSpeak 3 Server on Kubernetes with MariaDB database backend.

### Database Configuration

The chart uses the [mariadb-operator](https://github.com/mariadb-operator/mariadb-operator) to manage MariaDB instances for TeamSpeak's database backend.

**Important**: The MariaDB Operator must be pre-installed on your Kubernetes cluster.

In short: 
```bash
helm repo add mariadb-operator https://helm.mariadb.com/mariadb-operator
helm install mariadb-operator-crds mariadb-operator/mariadb-operator-crds
helm install mariadb-operator mariadb-operator/mariadb-operator
```

### Key Features

- Secure by default with security context and non root user
- TeamSpeak 3 Server with persistent storage
- MariaDB database backend via MariaDB Operator
- Supports both new MariaDB instances and existing MariaDB deployments
- StatefulSet deployment for data consistency
- Configurable UDP and TCP ports
- Optional external IP support for voice services
- Resource limits and requests configuration

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- **MariaDB Operator** installed
  - Installation guide: https://github.com/mariadb-operator/mariadb-operator
- Persistent Volume provisioner (if using persistent storage)

## Installation

### Quick Start

```bash
# Add the Helm repository
helm repo add syntax3rror404 https://syntax3rror404.github.io/helm-charts/charts

# Install with default values
helm install teamspeak syntax3rror404/teamspeak -n teamspeak --create-namespace
```

**Note**: A simple install uses default values. **You must create your own values file for production use.**

### Installation with Custom Values

```bash
helm install teamspeak syntax3rror404/teamspeak -n teamspeak --create-namespace -f myvalues.yaml
```

## Upgrade

```bash
helm upgrade teamspeak syntax3rror404/teamspeak -n teamspeak
```

## Uninstall

```bash
# Remove the Helm release
helm uninstall teamspeak -n teamspeak

# Delete the namespace (optional - this will remove all resources including PVCs)
kubectl delete ns teamspeak
```

## Configuration

### Core Configuration Values

| Parameter | Default | Description |
|-----------|---------|-------------|
| `teamspeak.image` | `docker.io/teamspeak` | TeamSpeak container image |
| `teamspeak.tag` | `3.13.7` | TeamSpeak version tag |
| `teamspeak.licenseAccept` | `accept` | Accept TeamSpeak license (must be "accept") |
| `teamspeak.ports.udp` | `9987` | Voice communication port (UDP) |
| `teamspeak.ports.tcp` | `[10011, 30033]` | ServerQuery and FileTransfer ports (TCP) |
| `teamspeak.persistence.enabled` | `true` | Enable persistent storage |
| `teamspeak.persistence.size` | `1Gi` | Persistent volume size |
| `teamspeak.persistence.storageClass` | `""` | Storage class for PVC |
| `teamspeak.resources` | `{}` | Resource limits and requests |
| `teamspeak.podSecurityContext` | `{}` | Pod security |
| `teamspeak.containerSecurityContext` | `{}` | Container security |

### Database Configuration - MariaDB Operator

| Parameter | Default | Description |
|-----------|---------|-------------|
| `mariadbOperator.enabled` | `false` | Enable MariaDB Operator integration |
| `mariadbOperator.database` | `teamspeak` | Database name to create |
| `mariadbOperator.username` | `teamspeak` | Database username to create |
| `mariadbOperator.password` | `teamspeaksecret` | Database password |
| `mariadbOperator.storageClass` | `longhorn` | Storage class for MariaDB |
| `mariadbOperator.useExisting.enabled` | `false` | Use existing MariaDB instance |
| `mariadbOperator.useExisting.namespace` | - | Namespace of existing MariaDB |
| `mariadbOperator.useExisting.mariaDbRef` | - | Name of existing MariaDB CR |

### Network Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `service.type` | `ClusterIP` | Kubernetes service type |
| `service.externalIPs` | `[]` | External IPs for the service |
| `headlessService.type` | `ClusterIP` | Headless service type |
| `headlessService.externalIPs` | `[]` | External IPs for headless service |

### Custom Labels

| Parameter | Default | Description |
|-----------|---------|-------------|
| `customLabels` | `{}` | Additional labels for all resources |

## Configuration Examples

### Example 1: Basic TeamSpeak with New MariaDB

```yaml
# myvalues.yaml
mariadbOperator:
  enabled: true
  database: teamspeak
  username: teamspeak
  password: "changeMeSecure123!"
  storageClass: longhorn

teamspeak:
  licenseAccept: "accept"
  persistence:
    enabled: true
    size: 5Gi
    storageClass: longhorn
  resources:
    limits:
      memory: 512Mi
    requests:
      cpu: 200m
      memory: 256Mi

service:
  type: LoadBalancer
```

### Example 2: TeamSpeak with Existing MariaDB

```yaml
# myvalues.yaml
mariadbOperator:
  enabled: true
  database: teamspeak
  username: teamspeak
  password: "changeMeSecure123!"
  useExisting:
    enabled: true
    namespace: database
    mariaDbRef: mariadb-shared

teamspeak:
  tag: "3.13.7"
  persistence:
    enabled: true
    size: 2Gi
    storageClass: ceph-block

service:
  type: NodePort
```

### Example 3: High-Performance Setup with External IP

```yaml
# myvalues.yaml
mariadbOperator:
  enabled: true
  database: teamspeak_prod
  username: ts3_user
  password: "VerySecurePassword456!"
  storageClass: fast-ssd

teamspeak:
  licenseAccept: "accept"
  persistence:
    enabled: true
    size: 10Gi
    storageClass: fast-ssd
  resources:
    limits:
      cpu: 2000m
      memory: 2Gi
    requests:
      cpu: 500m
      memory: 512Mi

service:
  type: ClusterIP
  externalIPs:
    - 192.168.1.100

customLabels:
  environment: production
  team: voip
```

## Architecture

### StatefulSet Deployment

The chart uses a StatefulSet to ensure:
- Stable network identity
- Persistent storage attached to the pod
- Single replica (TeamSpeak doesn't support clustering)

### Network Ports

TeamSpeak uses multiple ports for different functions:
- **9987/UDP**: Voice communication (main port)
- **10011/TCP**: ServerQuery interface (remote administration)
- **30033/TCP**: File transfer port

### Persistent Storage

Data is stored in `/var/ts3server` including:
- Server configuration
- Virtual server data
- Logs and snapshots
- Server identity files

### Database Backend

TeamSpeak 3 supports MariaDB as database backend for improved performance and scalability compared to SQLite.

## Port Forwarding for Local Access

If you're using ClusterIP service type, you can access TeamSpeak locally:

```bash
# Forward voice port
kubectl port-forward -n teamspeak svc/teamspeak 9987:9987

# Forward ServerQuery port
kubectl port-forward -n teamspeak svc/teamspeak 10011:10011
```

## Getting Server Admin Token

After the first installation, retrieve the admin token from the logs:

```bash
kubectl logs -n teamspeak -l app.kubernetes.io/name=teamspeak --tail=100
```

Look for a line containing: `ServerAdmin privilege key created, please use it to gain serveradmin rights`

## Troubleshooting

### Database Connection Issues

Check the database connectivity:

```bash
# Check pod logs
kubectl logs -n teamspeak -l app.kubernetes.io/name=teamspeak

# Verify MariaDB Operator resources
kubectl get mariadb,database,user,grant -n teamspeak

# Check if MariaDB is ready
kubectl get mariadb mariadb-teamspeak -n teamspeak -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
```

### Port Binding Issues

If using hostPort or external IPs:

```bash
# Check service endpoints
kubectl get svc -n teamspeak

# Verify pod is running
kubectl get pods -n teamspeak

# Check for port conflicts
kubectl describe pod -n teamspeak -l app.kubernetes.io/name=teamspeak
```

### Persistent Volume Issues

Check PVC status:

```bash
kubectl get pvc -n teamspeak
kubectl describe pvc data-teamspeak-server-0 -n teamspeak
```

### License Not Accepted

If the server doesn't start, ensure license is accepted:

```yaml
teamspeak:
  licenseAccept: "accept"  # Must be exactly "accept"
```

## Service Exposure Options

### LoadBalancer

Best for cloud environments (AWS, GCP, Azure):

```yaml
service:
  type: LoadBalancer
```

### NodePort

Access via any node IP on a high port:

```yaml
service:
  type: NodePort
```

### External IPs

Direct IP assignment (bare metal):

```yaml
service:
  type: ClusterIP
  externalIPs:
    - 192.168.1.100
```

### Ingress

Not recommended for TeamSpeak as it uses UDP for voice traffic. Use LoadBalancer or ExternalIP instead.

## Performance Tuning

### Resource Recommendations

**Small Server (< 32 slots)**:
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    memory: 256Mi
```

**Medium Server (32-64 slots)**:
```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    memory: 512Mi
```

**Large Server (> 64 slots)**:
```yaml
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2000m
    memory: 2Gi
```

## Migration from Bitnami MariaDB

If you're migrating from the previous version using Bitnami MariaDB as a dependency:

1. **Backup your TeamSpeak data** including database
2. Export your MariaDB database:
   ```bash
   kubectl exec -n teamspeak mariadb-0 -- mysqldump -u teamspeak -p teamspeak > backup.sql
   ```
3. Install MariaDB Operator on your cluster
4. Update your values file to use `mariadbOperator.enabled: true`
5. Deploy the new chart version
6. Import your database backup:
   ```bash
   kubectl exec -n teamspeak mariadb-teamspeak-0 -- mysql -u teamspeak -p teamspeak < backup.sql
   ```

## Security Considerations

- Change default database password before deployment
- Restrict ServerQuery port (10011) access
- Use strong server admin password
- Consider network policies to limit database access
- Enable TLS for ServerQuery if needed

## Known Limitations

- TeamSpeak 3 Server does not support horizontal scaling (single replica only)
- UDP traffic may require special load balancer configuration
- File transfers use a separate TCP port

## Support and Contributing

- **Issues**: https://github.com/Syntax3rror404/helm-charts/issues
- **Source Code**: https://github.com/Syntax3rror404/helm-charts
- **TeamSpeak**: https://www.teamspeak.com/
- **Docker Image**: https://hub.docker.com/_/teamspeak/
