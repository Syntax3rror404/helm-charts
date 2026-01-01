# helm-charts/mumble

Mumble is a free, open-source, low-latency, high-quality voice chat application primarily intended for gaming. Mumble uses a client-server architecture, where users connect to a central server.

## Introduction

This Helm chart deploys a Mumble Server on Kubernetes with persistent storage support.

### Key Features

- Mumble Server with persistent storage
- StatefulSet deployment for data consistency
- Configurable TCP and UDP ports (default: 64738)
- Optional external IP support for voice services
- Automatic superuser password generation
- Resource limits and requests configuration
- Custom configuration via environment variables

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- Persistent Volume provisioner (if using persistent storage)

## Installation

### Quick Start

```bash
# Add the Helm repository
helm repo add syntax3rror404 https://syntax3rror404.github.io/helm-charts/charts

# Install with default values
helm install mumble syntax3rror404/mumble -n mumble --create-namespace
```

**Note**: A simple install uses default values. **You must create your own values file for production use.**

### Installation with Custom Values

```bash
helm install mumble syntax3rror404/mumble -n mumble --create-namespace -f myvalues.yaml
```

## Upgrade

```bash
helm upgrade mumble syntax3rror404/mumble -n mumble
```

## Uninstall

```bash
# Remove the Helm release
helm uninstall mumble -n mumble

# Delete the namespace (optional - this will remove all resources including PVCs)
kubectl delete ns mumble
```

## Configuration

### Core Configuration Values

| Parameter | Default | Description |
|-----------|---------|-------------|
| `image.repository` | `ghcr.io/mumble-voip/mumble-server` | Mumble container image |
| `image.tag` | `v1.5.857-0` | Mumble version tag |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy |
| `service.type` | `ClusterIP` | Kubernetes service type |
| `service.externalIPs` | `[]` | External IPs for the service |
| `headlessService.type` | `ClusterIP` | Headless service type |
| `headlessService.externalIPs` | `[]` | External IPs for headless service |

### Persistence Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `persistence.enabled` | `true` | Enable persistent storage |
| `persistence.size` | `1Gi` | Persistent volume size |
| `persistence.storageClass` | `""` | Storage class for PVC (empty string uses cluster default) |
| `persistence.accessModes` | `[ReadWriteOnce]` | Access modes for PVC |

### Mumble Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `config` | `{}` | Mumble server configuration options |
| `environment.acceptUnknownSettings` | `false` | Accept unknown configuration settings |
| `environment.verbose` | `false` | Enable verbose logging |
| `environment.chownData` | `true` | Change ownership of data directory |
| `environment.customConfigFile` | `""` | Path to custom config file |

### Superuser Password

| Parameter | Default | Description |
|-----------|---------|-------------|
| `superuserPassword.existingSecret` | `""` | Use existing secret for superuser password |
| `superuserPassword.existingSecretKey` | `password` | Key in existing secret |
| `superuserPassword.generate` | `true` | Auto-generate superuser password |
| `superuserPassword.length` | `16` | Length of generated password |

### Resource Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `resources` | `{}` | Resource limits and requests |
| `nodeSelector` | `{}` | Node selector for pod assignment |
| `tolerations` | `[]` | Tolerations for pod assignment |
| `affinity` | `{}` | Affinity rules for pod assignment |

### Custom Labels

| Parameter | Default | Description |
|-----------|---------|-------------|
| `customLabels` | `{}` | Additional labels for all resources |

### Certification Generation

| Parameter | Default | Description |
|-----------|---------|-------------|
| `certificate.generate` | `false` | Generate a certificate with cert-manager |
| `certificate.spec.dnsNames` | `[]` | A list of DNS names to generate certificates for |
| `certificate.spec.issuerRef` | `{}` | A cert-manager reference to the Issuer or ClusterIssuer to use |

## Configuration Examples

### Example 1: Basic Mumble Server with Persistence

```yaml
# myvalues.yaml
persistence:
  enabled: true
  size: 5Gi
  storageClass: longhorn

config:
  welcometext: "Welcome to our Mumble server!"
  registerName: "My Gaming Server"
  bandwidth: "130000"
  users: "50"

resources:
  limits:
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

service:
  type: LoadBalancer
```

### Example 2: Mumble without Persistence (Testing)

```yaml
# myvalues.yaml
persistence:
  enabled: false

config:
  welcometext: "Test Server"
  users: "10"

service:
  type: NodePort
```

### Example 3: Mumble with External IP

```yaml
# myvalues.yaml
persistence:
  enabled: true
  size: 2Gi
  storageClass: ceph-block

config:
  welcometext: "Welcome to the community server"
  users: "100"
  bandwidth: "130000"
  server_password: "MySecretPassword"

service:
  type: ClusterIP
  externalIPs:
    - 192.168.1.100

customLabels:
  environment: production
  team: gaming
```

### Example 4: Using Existing Secret for Superuser Password

```yaml
# myvalues.yaml
superuserPassword:
  generate: false
  existingSecret: "my-mumble-secret"
  existingSecretKey: "superuser-password"

persistence:
  enabled: true
  size: 1Gi

service:
  type: NodePort
```

### Example 5: Using Cert Manager for generating certificates

```yaml
# myvalues.yaml
certificate:
  generate: true
  spec:
    issuerRef:
      group: cert-manager.io
      kind: ClusterIssuer
      name: my-cluster-issuer
    dnsNames:
      - "foo.example.com"
```

### Network Ports

Mumble uses a single port for both TCP and UDP:
- **64738/TCP**: Control channel
- **64738/UDP**: Voice communication (main channel)

### Persistent Storage

Data is stored in `/data` when persistence is enabled, including:
- Server configuration (`mumble-server.ini`)
- Database (SQLite by default)
- Server certificates
- Channel structure
- User registrations

**Without persistence** (`persistence.enabled: false`), data is stored in an emptyDir volume and will be lost when the pod restarts.

## Getting Started

### Retrieving the Superuser Password

If you used auto-generated password (default), retrieve it from the secret:

```bash
# Get the secret name
kubectl get secrets -n mumble

# Retrieve the password
kubectl get secret -n mumble mumble-superuser-password -o jsonpath='{.data.password}' | base64 -d
echo
```

### Connecting to Your Mumble Server

1. Download the Mumble client from https://www.mumble.info/
2. Install and start the client
3. Add a new server with:
   - **Address**: Your server IP or hostname
   - **Port**: 64738 (or your custom port)
   - **Username**: Your desired username
   - **Password**: Leave empty (unless server password is set)

4. After first connection, authenticate as SuperUser:
   - Right-click on your name
   - Select "Register"
   - Use username `SuperUser` with the password from the secret

## Port Forwarding for Local Access

If you're using ClusterIP service type, you can access Mumble locally:

```bash
# Forward both TCP and UDP (you may need two terminals)
kubectl port-forward -n mumble svc/mumble 64738:64738
```

## Common Configuration Options

### Server Identity

```yaml
config:
  registerName: "My Server Name"
  registerHostname: "mumble.example.com"
  registerPassword: "registration_password"
```

### User Limits and Bandwidth

```yaml
config:
  users: "100"              # Maximum users
  bandwidth: "130000"       # Bandwidth per user (bits/sec)
  opusthreshold: "100"      # Opus quality threshold
```

### Security Settings

```yaml
config:
  server_password: "server_access_password"
  allowping: "false"        # Disable server ping
  certrequired: "true"      # Require client certificates
```

### Text Messages

```yaml
config:
  welcometext: "Welcome to our Mumble server!<br/>Rules:<br/>1. Be respectful<br/>2. No spam"
  sendversion: "false"      # Hide server version
```

## Troubleshooting

### Check Pod Status

```bash
# Check if the pod is running
kubectl get pods -n mumble

# Check pod logs
kubectl logs -n mumble -l app.kubernetes.io/name=mumble

# Describe the pod for events
kubectl describe pod -n mumble -l app.kubernetes.io/name=mumble
```

### Persistent Volume Issues

```bash
# Check PVC status
kubectl get pvc -n mumble

# Describe PVC for issues
kubectl describe pvc -n mumble

# Check if PV is bound
kubectl get pv
```

### Connection Issues

```bash
# Check service endpoints
kubectl get svc -n mumble

# Test connectivity from within cluster
kubectl run -it --rm debug --image=busybox --restart=Never -n mumble -- sh
# Inside the pod:
nc -zv mumble.mumble.svc.cluster.local 64738
```

### Configuration Issues

```bash
# Check environment variables in pod
kubectl exec -n mumble -l app.kubernetes.io/name=mumble -- env | grep MUMBLE

# Check mounted volumes
kubectl exec -n mumble -l app.kubernetes.io/name=mumble -- ls -la /data
```

### Password Issues

```bash
# Verify secret exists
kubectl get secret -n mumble

# Check secret content (base64 encoded)
kubectl get secret mumble-superuser-password -n mumble -o yaml

# Decode password
kubectl get secret mumble-superuser-password -n mumble -o jsonpath='{.data.password}' | base64 -d
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

Not recommended for Mumble as it uses UDP for voice traffic. Use LoadBalancer or ExternalIP instead.

## Performance Tuning

### Resource Recommendations

**Small Server (< 25 users)**:
```yaml
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    memory: 128Mi
```

**Medium Server (25-100 users)**:
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    memory: 256Mi
```

**Large Server (> 100 users)**:
```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

### Bandwidth Settings

Adjust based on your needs:
- **Low**: 48000 bits/sec (acceptable quality)
- **Medium**: 72000 bits/sec (good quality)
- **High**: 130000 bits/sec (excellent quality)
- **Ultra**: 200000+ bits/sec (maximum quality)

## Advanced Configuration

### Using Custom Configuration File

1. Create a ConfigMap with your configuration:

```bash
kubectl create configmap mumble-config -n mumble --from-file=mumble-server.ini
```

2. Mount it in your values:

```yaml
environment:
  customConfigFile: "/config/mumble-server.ini"

# You'll need to add volumeMounts manually or extend the chart
```

### SSL/TLS Certificates

Mumble automatically generates self-signed certificates. For custom certificates:

1. Create a secret with your certificates
2. Mount them to `/data` or configure via `config` options

## Backup and Restore

### Backup

```bash
# Get the PVC name
kubectl get pvc -n mumble

# Create a backup pod
kubectl run backup -n mumble --image=busybox --command -- sleep infinity

# Copy data from PVC
kubectl exec -n mumble backup -- tar czf /tmp/mumble-backup.tar.gz -C /data .
kubectl cp mumble/backup:/tmp/mumble-backup.tar.gz ./mumble-backup.tar.gz

# Cleanup
kubectl delete pod backup -n mumble
```

### Restore

```bash
# Create a restore pod
kubectl run restore -n mumble --image=busybox --command -- sleep infinity

# Copy backup to pod
kubectl cp ./mumble-backup.tar.gz mumble/restore:/tmp/mumble-backup.tar.gz

# Extract to PVC
kubectl exec -n mumble restore -- tar xzf /tmp/mumble-backup.tar.gz -C /data

# Cleanup
kubectl delete pod restore -n mumble

# Restart the StatefulSet
kubectl rollout restart statefulset -n mumble
```
## Monitoring

### Check Server Status

```bash
# View logs
kubectl logs -f -n mumble -l app.kubernetes.io/name=mumble

# Check resource usage
kubectl top pod -n mumble

# Get server statistics (if exposed)
kubectl exec -n mumble -l app.kubernetes.io/name=mumble -- murmur-user-wrapper murmurd -version
```
