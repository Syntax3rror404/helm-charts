# Minecraft Java Server – Helm Chart

A Helm chart to run a Minecraft java server (mods supported) on Kubernetes using Eclipse Temurin (OpenJDK 21) and a StatefulSet with persistent storage.

If the volume doesnt include any server.jar a init container will automaticly download it and will bootrap the minecraft server.

You can upgrade minecraft by set a new minecraft version value and simply delete the server jar. If you restart the pod the new server jar would be automatily downloaded.

If you want to install any mods, you can easyly use kubectl cp.

## Features

| Feature | Details |
|---|---|
| Base image | `eclipse-temurin:21-jre-alpine` (rootless, minimal) |
| Workload | `StatefulSet` – stable network identity & ordered pods |
| Persistence | `PersistentVolumeClaim` via `volumeClaimTemplates` |
| JAR download | Init container (`curlimages/curl`) downloads the server JAR if missing |
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

1. Update `minecraft.version` and `minecraft.jarUrl` in `values.yaml`
2. Delete the existing JAR so the init container re-downloads:
   ```bash
   kubectl exec -it my-minecraft-0 -n minecraft -- rm /data/server.jar
   kubectl rollout restart statefulset/my-minecraft -n minecraft
   ```

## Uninstall

```bash
helm uninstall my-minecraft -n minecraft
# PVC is NOT deleted automatically – delete manually if desired:
kubectl delete pvc data-my-minecraft-0 -n minecraft
```
