---
layout: page
title: Kubernetes & OpenShift Deployment
permalink: /ops/kubernetes/
parent: Operations Guide
---

# Kubernetes & OpenShift Deployment

![Kubernetes](https://img.shields.io/badge/Kubernetes-1.27+-326CE5?style=flat-square&logo=kubernetes&logoColor=white)
![OpenShift](https://img.shields.io/badge/OpenShift-4.x-EE0000?style=flat-square&logo=redhatopenshift&logoColor=white)
![Helm](https://img.shields.io/badge/approach-plain%20manifests-gray?style=flat-square)

This page documents the Kubernetes (and OpenShift) translation of the Docker Compose stacks. The manifests in `k8s/` and `openshift/` are structured equivalents of `docker-compose_2024.yml` and `docker-compose_2024-monitoring.yml`.

---

## Manifest Structure

```
k8s/
├── base/
│   ├── namespace.yaml              # dspace-cris namespace
│   ├── secrets.yaml                # DB credentials, Django secret key
│   ├── configmaps.yaml             # local.cfg, discovery.xml, db-init SQL
│   ├── persistentvolumeclaims.yaml # pgdata, assetstore, solr-data
│   ├── postgres.yaml               # Deployment + Service
│   ├── solr.yaml                   # Deployment + Service
│   ├── dspace.yaml                 # Deployment + Service
│   ├── django.yaml                 # Deployment + Service
│   └── frontend.yaml               # frontend + django-frontend Deployments,
│                                   # Services, and Ingress
└── monitoring/
    └── monitoring.yaml             # Loki, Prometheus, Alertmanager, Grafana
                                    # + ServiceAccount + RBAC + Ingress

openshift/
├── scc.yaml                        # SecurityContextConstraints + binding
├── routes.yaml                     # Routes replacing Ingress
└── networkpolicy.yaml              # Namespace isolation policies
```

---

## Docker Compose → Kubernetes Translation

The table below maps each Compose concept to its Kubernetes equivalent.

| Docker Compose | Kubernetes | Notes |
|---|---|---|
| `service:` | `Deployment` + `Service` | Each compose service becomes a Deployment; the DNS name becomes a `ClusterIP` Service |
| `depends_on: service_healthy` | `initContainers` | Poll the dependency with `pg_isready` or `curl` until ready |
| `healthcheck:` | `readinessProbe` + `livenessProbe` + `startupProbe` | Three separate probe types replace the single compose healthcheck |
| `volumes: pgdata:` | `PersistentVolumeClaim` | Named volumes → PVCs bound to the cluster's storage class |
| `ports: "5432:5432"` | `Service.spec.ports` | Services handle port routing; no host port binding needed |
| `networks: dspacenet` | Namespace + `ClusterIP` Services | All pods in the same namespace can reach each other by service name |
| `environment:` | `env:` with `valueFrom` | Sensitive values reference `Secret`, non-sensitive reference `ConfigMap` |
| `labels: service: dspace` | Pod annotations | Used by Fluent Bit for log enrichment |
| Host port `4000:80` | `Ingress` or `Route` | External access via Ingress controller (K8s) or Router (OpenShift) |
| `strategy: Recreate` | `strategy.type: Recreate` | Required for `ReadWriteOnce` PVCs — ensures old pod stops before new one starts |

### `depends_on` → `initContainers`

Kubernetes has no built-in equivalent of Compose's `depends_on: condition: service_healthy`. The pattern used throughout these manifests is an `initContainer` that polls the dependency:

```yaml
initContainers:
  - name: wait-for-postgres
    image: postgres:15-alpine
    command:
      - sh
      - -c
      - until pg_isready -h dspacedb -U dspace -d dspace; do sleep 5; done
```

The main container does not start until all `initContainers` complete successfully.

### Startup probe for DSpace

DSpace takes up to 3 minutes to start (Maven build, database migration, Solr warmup). The `startupProbe` prevents the `livenessProbe` from killing the pod during startup:

```yaml
startupProbe:
  httpGet:
    path: /server/api
    port: 8080
  failureThreshold: 30
  periodSeconds: 10   # up to 300s allowed for initial startup
livenessProbe:
  httpGet:
    path: /server/api
    port: 8080
  initialDelaySeconds: 180
  periodSeconds: 30
  failureThreshold: 5
```

### Trusted proxy IP range

In Docker Compose, the trusted proxy range is hardcoded to `172.23.0.0/16` (the Docker network subnet). In Kubernetes, pods receive IPs from the cluster's pod CIDR. Find your cluster's CIDR and set it in the `dspace-config` ConfigMap and in the DSpace deployment:

```bash
# Find your pod CIDR
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'
# Example output: 10.244.0.0/24 10.244.1.0/24

# Then set in local.cfg ConfigMap:
proxies.trusted.ipranges = 10.244.0, 10.244.1
```

---

## Building and Pushing Images

The Kubernetes manifests reference images that must be built from the DSpace source and pushed to a container registry accessible to your cluster. The compose files build images at deploy time — Kubernetes requires pre-built images.

```bash
# 1. Clone DSpace source
git clone --branch dspace-cris-2024.02.04 --depth 1 \
    https://github.com/4Science/DSpace.git dspace-src-2024

# 2. Build the Solr image
docker build \
  -f dspace-src-2024/dspace/src/main/docker/dspace-solr/Dockerfile \
  --build-arg SOLR_VERSION=8.11.4 \
  -t your-registry/dspace-solr:2024.02.04 \
  dspace-src-2024

# 3. Build the DSpace (Spring Boot) image
docker build \
  -f dspace-src-2024/Dockerfile.test \
  -t your-registry/dspace:2024.02.04 \
  dspace-src-2024

# 4. Build the Django sidecar
docker build \
  -f django/Dockerfile \
  -t your-registry/dspace-django:latest \
  django/

# 5. Build the frontend
docker build \
  -f frontend/Dockerfile \
  -t your-registry/dspace-frontend:latest \
  frontend/

# 6. Build the Config Cockpit
docker build \
  -f django-frontend/Dockerfile \
  -t your-registry/dspace-django-frontend:latest \
  django-frontend/

# 7. Push all images
for img in dspace-solr:2024.02.04 dspace:2024.02.04 dspace-django:latest \
           dspace-frontend:latest dspace-django-frontend:latest; do
  docker push your-registry/$img
done
```

Update all `image:` fields in the manifests with your registry path before applying.

---

## Deploying to Kubernetes

```bash
# 1. Create namespace and apply base stack
kubectl apply -f k8s/base/namespace.yaml
kubectl apply -f k8s/base/secrets.yaml
kubectl apply -f k8s/base/configmaps.yaml
kubectl apply -f k8s/base/persistentvolumeclaims.yaml

# Apply services in dependency order
kubectl apply -f k8s/base/postgres.yaml
kubectl apply -f k8s/base/solr.yaml
kubectl apply -f k8s/base/dspace.yaml
kubectl apply -f k8s/base/django.yaml
kubectl apply -f k8s/base/frontend.yaml

# 2. Watch startup progress
kubectl -n dspace-cris get pods -w

# 3. Check DSpace logs
kubectl -n dspace-cris logs -f deployment/dspace

# 4. Verify all services are ready
kubectl -n dspace-cris get pods
kubectl -n dspace-cris get services
kubectl -n dspace-cris get ingress

# 5. (Optional) Apply monitoring stack
kubectl apply -f k8s/monitoring/monitoring.yaml
```

### Apply all at once (after order dependencies are satisfied)

```bash
kubectl apply -f k8s/base/ && kubectl apply -f k8s/monitoring/
```

---

## Deploying to OpenShift

```bash
# 1. Login and create project
oc login https://your-openshift-cluster:6443
oc new-project dspace-cris

# 2. Grant anyuid SCC (required for postgres uid=999 and solr uid=8983)
oc adm policy add-scc-to-serviceaccount anyuid -z default -n dspace-cris
oc adm policy add-scc-to-serviceaccount anyuid -z default -n dspace-monitoring

# 3. Apply SCC manifests (alternative to above — creates a scoped SCC)
oc apply -f openshift/scc.yaml

# 4. Apply base Kubernetes manifests (OpenShift is Kubernetes-compatible)
oc apply -f k8s/base/

# 5. Apply Routes instead of Ingress
# (skip or remove the Ingress in k8s/base/frontend.yaml before applying)
oc apply -f openshift/routes.yaml

# 6. Apply NetworkPolicies
oc apply -f openshift/networkpolicy.yaml

# 7. Apply monitoring stack
oc new-project dspace-monitoring
oc adm policy add-scc-to-serviceaccount anyuid -z default -n dspace-monitoring
oc apply -f k8s/monitoring/monitoring.yaml
oc apply -f openshift/routes.yaml   # includes the Grafana route in dspace-monitoring ns

# 8. Watch startup
oc -n dspace-cris get pods -w
```

---

## OpenShift-Specific Considerations

### Security Context Constraints (SCC)

OpenShift's SCC system is stricter than Kubernetes pod security policies. Three containers require special handling:

| Container | UID | Required SCC | Why |
|---|---|---|---|
| `postgres:15-alpine` | 999 | `anyuid` or `dspace-cris-scc` | PostgreSQL requires a fixed UID for data directory ownership |
| Solr | 8983 | `anyuid` or `dspace-cris-scc` | Solr uses uid 8983 by convention |
| `solr-init-permissions` initContainer | 0 (root) | `anyuid` | `chown` on the data volume requires root |
| All other containers | Random (OCP assigns) | `restricted` (default) | nginx, Django, DSpace, Grafana work fine with random UIDs |

The quickest approach for a development cluster:

```bash
oc adm policy add-scc-to-serviceaccount anyuid -z default -n dspace-cris
```

For production, use the custom `dspace-cris-scc` manifest (`openshift/scc.yaml`) which grants only the minimum necessary permissions.

### Routes vs Ingress

OpenShift Routes are the native external access mechanism. They are equivalent to Kubernetes Ingress but configured differently:

| Feature | Kubernetes Ingress | OpenShift Route |
|---|---|---|
| TLS termination | Annotation on Ingress | `spec.tls.termination: edge/passthrough/reencrypt` |
| IP whitelist | Controller-specific annotation | `haproxy.router.openshift.io/ip_whitelist` |
| Path routing | `spec.rules[].http.paths` | `spec.path` (one path per Route) |
| Hostname | `spec.rules[].host` | `spec.host` |
| HTTP → HTTPS redirect | Controller annotation | `spec.tls.insecureEdgeTerminationPolicy: Redirect` |

The `openshift/routes.yaml` file creates one Route per service path. The Config Cockpit and Grafana routes include the `ip_whitelist` annotation to restrict access to internal IP ranges.

### ImageStreams (optional)

For environments using an internal OpenShift registry, create ImageStreams instead of referencing external registries directly:

```bash
# Create ImageStreams for all custom images
oc import-image dspace:2024.02.04 \
  --from=your-registry/dspace:2024.02.04 \
  --confirm -n dspace-cris

# Reference in Deployment as:
# image: image-registry.openshift-image-registry.svc:5000/dspace-cris/dspace:2024.02.04
```

---

## Secrets Management

The `secrets.yaml` in these manifests contains base64-encoded default values. **Do not commit plain Secret manifests with real credentials to version control.**

### Recommended approaches by platform

| Platform | Tool | Notes |
|---|---|---|
| Any Kubernetes | [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) | Encrypt secrets with a cluster public key; safe to commit to git |
| Any Kubernetes | [External Secrets Operator](https://external-secrets.io/) | Pull secrets from Vault, AWS SSM, Azure Key Vault, GCP Secret Manager |
| OpenShift | [OpenShift Secrets Store CSI Driver](https://docs.openshift.com/container-platform/4.14/nodes/pods/nodes-pods-secrets-store.html) | Mount secrets from HashiCorp Vault or cloud KMS directly into pods |
| OpenShift | `oc create secret` | Imperative creation — never stored in the cluster manifest |

```bash
# Create secrets imperatively (no YAML committed to git)
kubectl create secret generic dspace-db-credentials \
  --from-literal=POSTGRES_USER=dspace \
  --from-literal=POSTGRES_PASSWORD=<real-password> \
  --from-literal=POSTGRES_DB=dspace \
  --from-literal=DB_NAME=django_config \
  --from-literal=DB_USER=dspace \
  --from-literal=DB_PASSWORD=<real-password> \
  -n dspace-cris

kubectl create secret generic django-secret \
  --from-literal=DJANGO_SECRET_KEY=$(python3 -c "import secrets; print(secrets.token_urlsafe(50))") \
  -n dspace-cris
```

---

## Resource Requests and Limits

The manifests include conservative starting values. Adjust based on your expected load:

| Service | CPU request | CPU limit | Memory request | Memory limit |
|---|---|---|---|---|
| dspacedb | 250m | 1000m | 256Mi | 1Gi |
| dspacesolr | 500m | 2000m | 1Gi | 3Gi |
| dspace | 1000m | 4000m | 2Gi | 6Gi |
| django | 100m | 500m | 256Mi | 512Mi |
| frontend | 50m | 200m | 64Mi | 256Mi |
| django-frontend | 50m | 200m | 64Mi | 256Mi |

DSpace is the most memory-hungry service. The JVM is configured with `-XX:MaxRAMPercentage=75`, meaning it will use up to 75% of its container memory limit. With a 6Gi limit, the heap ceiling is ~4.5Gi.

---

## DSpace 2025 / Solr 9 Adjustments

For the 2025 stack (`docker-compose_2025.yml`), one additional change is required in `solr.yaml`:

```yaml
# In the dspacesolr Deployment, add SOLR_OPTS:
containers:
  - name: solr
    image: your-registry/dspace-solr:2025.x
    env:
      - name: SOLR_OPTS
        value: "-Dsolr.config.lib.enabled=true"
        # On macOS dev clusters, also add:
        # value: "-Dsolr.config.lib.enabled=true -Djava.security.manager=allow"
```

All other manifests are identical between the 2024 and 2025 stacks. See the [Solr 9 Upgrade]({{ '/ops/solr9-upgrade/' | relative_url }}) page for the full list of changes.

---

## Maintenance Commands

### DSpace

```bash
# Open a shell in the DSpace pod
kubectl -n dspace-cris exec -it deployment/dspace -- bash

# Full Solr reindex
kubectl -n dspace-cris exec deployment/dspace -- \
  /dspace/bin/dspace index-discovery -f

# Database migrate
kubectl -n dspace-cris exec deployment/dspace -- \
  /dspace/bin/dspace database migrate
```

### Django

```bash
# Open Django shell
kubectl -n dspace-cris exec -it deployment/django -- \
  python manage.py shell

# Re-import submission forms after DSpace config changes
kubectl -n dspace-cris exec deployment/django -- \
  python manage.py import_plain_config --config-dir /app/frontend-config

# Import CRIS layout
kubectl -n dspace-cris exec deployment/django -- \
  python manage.py import_cris_layout /path/to/cris-layout.xls

# Create a Config Cockpit superuser
kubectl -n dspace-cris exec -it deployment/django -- \
  python manage.py createsuperuser
```

### PostgreSQL

```bash
# Connect to psql
kubectl -n dspace-cris exec -it deployment/dspacedb -- \
  psql -U dspace -d dspace

# Dump the dspace database
kubectl -n dspace-cris exec deployment/dspacedb -- \
  pg_dump -U dspace dspace > dspace_backup_$(date +%Y%m%d).sql

# Dump the django_config database
kubectl -n dspace-cris exec deployment/dspacedb -- \
  pg_dump -U dspace django_config > django_config_backup_$(date +%Y%m%d).sql
```

---

## Reference

| Resource | Link |
|---|---|
| Kubernetes Deployments | [kubernetes.io/docs/concepts/workloads/controllers/deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) |
| initContainers | [kubernetes.io/docs/concepts/workloads/pods/init-containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) |
| Startup / Liveness / Readiness probes | [kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) |
| PersistentVolumeClaims | [kubernetes.io/docs/concepts/storage/persistent-volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) |
| NetworkPolicy | [kubernetes.io/docs/concepts/services-networking/network-policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) |
| OpenShift SCC | [docs.openshift.com/container-platform/latest/authentication/managing-security-context-constraints](https://docs.openshift.com/container-platform/latest/authentication/managing-security-context-constraints.html) |
| OpenShift Routes | [docs.openshift.com/container-platform/latest/networking/routes/route-configuration](https://docs.openshift.com/container-platform/latest/networking/routes/route-configuration.html) |
| Sealed Secrets | [github.com/bitnami-labs/sealed-secrets](https://github.com/bitnami-labs/sealed-secrets) |
| External Secrets Operator | [external-secrets.io](https://external-secrets.io/) |

<div class="page-nav">
  <a href="{{ '/ops/solr9-features/' | relative_url }}">← Solr 9 New Features</a>
  <a href="{{ '/' | relative_url }}">↑ Home</a>
</div>
