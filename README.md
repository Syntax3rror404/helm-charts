# helm-charts

A collection of Helm charts for self-hosted applications on Kubernetes.

[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/syntaxerror404)](https://artifacthub.io/packages/search?repo=syntaxerror404)

## Charts

| Chart | Description | Version |
|-------|-------------|---------|
| keycloak | Rootless Keycloak with mariadb-operator and multi-replica support | ![version](https://img.shields.io/badge/dynamic/yaml?url=https://syntax3rror404.github.io/helm-charts/charts/index.yaml&query=$.entries.keycloak[0].version&label=) |
| minecraft | Minecraft Java (vanilla & NeoForge) with auto-update support | ![version](https://img.shields.io/badge/dynamic/yaml?url=https://syntax3rror404.github.io/helm-charts/charts/index.yaml&query=$.entries.minecraft[0].version&label=) |
| mumble | Mumble voice server | ![version](https://img.shields.io/badge/dynamic/yaml?url=https://syntax3rror404.github.io/helm-charts/charts/index.yaml&query=$.entries.mumble[0].version&label=) |
| snipeit | Snipe-IT asset management | ![version](https://img.shields.io/badge/dynamic/yaml?url=https://syntax3rror404.github.io/helm-charts/charts/index.yaml&query=$.entries.snipeit[0].version&label=) |
| teamspeak | TeamSpeak 3 server | ![version](https://img.shields.io/badge/dynamic/yaml?url=https://syntax3rror404.github.io/helm-charts/charts/index.yaml&query=$.entries.teamspeak[0].version&label=) |

## Installation

Both HTTPS and OCI are supported.

### HTTPS

```bash
helm repo add syntax3rror404 https://syntax3rror404.github.io/helm-charts/charts
helm repo update
helm install <release> syntax3rror404/<chart> -f values.yaml
```

### OCI

```bash
helm install <release> oci://ghcr.io/syntax3rror404/helm-charts/<chart> --version <version> -f values.yaml
```
