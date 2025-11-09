# Sonatype Nexus Repository Helm Chart

This repository provides a vanilla Helm chart for deploying the free (OSS) distribution of Sonatype Nexus Repository 3. The chart focuses on a secure-by-default StatefulSet deployment that exposes the web UI as well as optional Docker registry connectors.

## Prerequisites

- Kubernetes 1.24+
- Helm 3.8+
- Persistent storage provisioner (unless you set `persistence.enabled=false`)

## Quick start

```bash
helm dependency build ./
helm install nexus . \
  --namespace nexus --create-namespace
```

Forward the service locally while you work through the onboarding wizard:

```bash
kubectl port-forward svc/nexus 8081:8081 -n nexus
```

The default admin password is written to `/nexus-data/admin.password` the first time Nexus starts. Retrieve it with:

```bash
kubectl exec -n nexus sts/nexus -- cat /nexus-data/admin.password
```

> **Note:** Nexus Repository OSS does not support active/active clustering. The `replicaCount` value is available for completeness, but Sonatype only supports `1` replica.

## Common configuration

| Value | Description | Default |
| ----- | ----------- | ------- |
| `image.repository` / `image.tag` | Container image coordinates for Nexus OSS | `sonatype/nexus3:3.71.0` |
| `service.port` | Primary HTTP service port | `8081` |
| `service.docker.enabled` | Adds an additional Docker-compatible port on the service and container | `true` |
| `ingress.enabled` | Creates an Ingress resource; configure `ingress.hosts`/`ingress.tls` for your cluster | `false` |
| `persistence.size` | Requested storage for the Nexus data volume | `50Gi` |
| `javaOpts` | JVM memory and GC tuning passed via `INSTALL4J_ADD_VM_PARAMS` | `-Xms1200m -Xmx1200m -XX:MaxDirectMemorySize=2g` |
| `nexusProperties` | Optional map or multi-line string rendered to `/nexus-data/etc/<filename>` | `{}` |
| `resources` | CPU / memory requests & limits for the main pod | `requests: {cpu: 500m, memory: 2Gi}` |

Additional fields in `values.yaml` let you control service accounts, probes, topology spread constraints, ingress, extra containers, and arbitrary volumes/mounts.

## Rendering and applying manifests

To inspect what will be applied without touching the cluster, run:

```bash
helm template nexus . --namespace nexus > rendered.yaml
```

To upgrade an existing release after editing `values.yaml`:

```bash
helm upgrade nexus . -n nexus
```

## Contributing

1. Adjust `values.yaml` or add new templates in `templates/`.
2. Run `helm lint .` to verify the chart structure.
3. Use `helm template` to check the rendered manifests before applying them to a cluster.
