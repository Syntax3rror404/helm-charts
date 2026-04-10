# Minecraft vanilla / neoforge java Server 

A Helm chart to run a Minecraft java vanilla and neoforge server on Kubernetes using Eclipse Temurin (OpenJDK 21) and a StatefulSet with persistent storage.

If you set `neoforge.enabled: true` with a version like `neoforge.version: "21.1.219"` a init container will install neoforge for you. If you change the version, it gets automaticly installed.

If the volume doesnt include any server.jar a init container will automaticly download it and will bootrap the minecraft server.

You can upgrade minecraft by set a new minecraft version value.

If you want to install any mods, you can easily use kubectl cp, for example `kubectl cp ./mods.tar.gz  minecraft/minecraft-0:/data/mods.tar.gz` afterwards you can untar the mods tar and restart the server to enable the mods, thats it. 

## Features

| Feature | Details |
|---|---|
| Base image | `docker.io/eclipse-temurin` (rootless, minimal) |
| Workload | `StatefulSet` – stable network identity & ordered pods |
| Persistence | `PersistentVolumeClaim` via `volumeClaimTemplates` |
| JAR download | Init container (`curlimages/curl`) downloads the server JAR if missing or upgrades|
| Config | Server properties managed via `ConfigMap` |
| Security | `runAsNonRoot`, dropped capabilities, `seccompProfile`, `readOnlyRootFilesystem` on init |
| Networking | Headless + external Service, optional NetworkPolicy |
| RCON | Optional remote-console with Secret-backed password |

## Quick Start

```bash
# Add & install (from local chart)
helm install my-minecraft ./minecraft \
  --namespace minecraft \
  --create-namespace \
  --set minecraft.eulaAccepted=true

# Watch the pod start
kubectl get pods -n minecraft -w

# Get the external IP (LoadBalancer)
kubectl get svc my-minecraft-minecraft -n minecraft
```

## Configuration

See [`values.yaml`](values.yaml) for all options. Key settings:

Neoforge Example: 
```yaml
minecraft:
  jarUrl: "https://..."   # override to use a specific jar
  eulaAccepted: true      # MUST be true
  jvmOpts: "-Xms1G -Xmx2G ..."

persistence:
  enabled: true
  size: 10Gi

resources:
  limits:
    cpu: "2000m"
    memory: "2.5Gi"

neoforge:
  enabled: true

rcon:
  enabled: false
  password: "changeme"   # use a Secret in production!
```

Vanilla Example: 
```yaml
minecraft:
  jarUrl: "https://..."   # override to use a specific jar
  eulaAccepted: true      # MUST be true
  jvmOpts: "-Xms1G -Xmx2G ..."

persistence:
  enabled: true
  size: 10Gi

resources:
  limits:
    cpu: "2000m"
    memory: "2.5Gi"

rcon:
  enabled: false
  password: "changeme"   # use a Secret in production!
```

## Upgrade / Changing Minecraft Version

Update `minecraft.jarUrl` or if modded `neoforge.version` in `values.yaml` and perform a helm upgrade. A init container download and install the updates for you.

## Uninstall

```bash
helm uninstall my-minecraft -n minecraft
# PVC is NOT deleted automatically – delete manually if desired:
kubectl delete pvc data-my-minecraft-0 -n minecraft
```
