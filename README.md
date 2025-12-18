# Sonatype Nexus Repository Helm Chart

This repository provides a Helm chart for deploying Sonatype Nexus Repository 3 in a highly available configuration backed by PostgreSQL. The chart keeps a secure-by-default StatefulSet deployment, exposes the web UI and optional Docker registry connectors, and now includes HA-friendly defaults for the delta cluster’s `postgres-delta` database.

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

> **Note:** Active/active HA requires Nexus Repository Pro and a shared blob store (for example S3/MinIO). This chart wires Nexus to PostgreSQL and runs multiple pods, but you must move blob stores to shared storage after installation for true HA behaviour.

## High availability on the delta cluster

1. Create the PostgreSQL database and user (runs against the existing `postgres-delta` HA cluster):
   ```bash
   kubectl exec -n postgresql postgres-delta-postgres-ha-0 -- bash -c "PGPASSWORD=PerkinzkSecure42 psql -U perkinzk -h localhost -c \"CREATE DATABASE nexus;\""
   kubectl exec -n postgresql postgres-delta-postgres-ha-0 -- bash -c "PGPASSWORD=PerkinzkSecure42 psql -U perkinzk -h localhost -c \"CREATE USER nexus WITH PASSWORD 'NexusDbSecure42';\""
   kubectl exec -n postgresql postgres-delta-postgres-ha-0 -- bash -c "PGPASSWORD=PerkinzkSecure42 psql -U perkinzk -h localhost -c \"GRANT ALL PRIVILEGES ON DATABASE nexus TO nexus;\""
   ```
2. Create the Nexus namespace and DB credentials secret consumed by the chart:
   ```bash
   kubectl create namespace nexus
   kubectl create secret generic nexus-db -n nexus \
     --from-literal=username=nexus \
     --from-literal=password=NexusDbSecure42
   ```
3. Deploy/upgrade Nexus with the built-in HA defaults:
   ```bash
   helm upgrade --install nexus . \
     --namespace nexus --create-namespace
   ```
4. After startup, configure your blob stores to use shared object storage (S3/MinIO) before putting the cluster under load.

## Common configuration

| Value | Description | Default |
| ----- | ----------- | ------- |
| `image.repository` / `image.tag` | Container image coordinates for Nexus | `sonatype/nexus3:3.71.0` |
| `replicaCount` | Number of Nexus pods (HA) | `3` |
| `database.*` | External PostgreSQL wiring (`host`, `port`, `name`, `user`, `existingSecret` + keys) | Pre-set to `postgres-delta` HA cluster |
| `ha.*` | HA helpers: PDB, pod management policy, rolling update strategy | Enabled with PDB `minAvailable: 2` |
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
