# Kustomize Configuration

This directory contains an optimized Kustomize configuration using the **base/overlays pattern**, organized by application for the k3s cluster running on Raspberry Pi.

## Structure

```
kustomize/
├── base/                           # Base configurations for all apps
│   ├── grafana/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   ├── viktutors/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   └── kustomization.yaml
│   ├── resume/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── emailer/
│       ├── postgres-deployment.yaml
│       ├── postgres-service.yaml
│       ├── emailer-deployment.yaml
│       ├── emailer-service.yaml
│       ├── pvc.yaml
│       ├── secret.yaml
│       └── kustomization.yaml
├── overlays/                       # Namespace/environment-specific overlays
│   ├── grafana/
│   │   └── monitoring/            # Monitoring namespace overlay
│   │       ├── namespace.yaml
│   │       ├── ingress.yaml
│   │       └── kustomization.yaml
│   ├── viktutors/
│   │   └── viktutors/              # Viktutors namespace overlay
│   │       ├── namespace.yaml
│   │       ├── ingress.yaml
│   │       └── kustomization.yaml
│   ├── resume/
│   │   └── default/                # Default namespace overlay
│   │       ├── ingress.yaml
│   │       └── kustomization.yaml
│   └── emailer/
│       └── emailer/                # Emailer namespace overlay
│           ├── namespace.yaml
│           └── kustomization.yaml
├── kustomization.yaml             # Root kustomization
└── README.md
```

## Base/Overlays Pattern

This structure follows Kustomize best practices:

- **base/**: Contains the core application resources (deployments, services, configmaps, etc.) without namespace-specific configuration
- **overlays/**: Contains namespace-specific resources (namespaces, ingress, environment-specific configs) that reference the base

### Benefits

1. **Reusability**: Base can be reused across different namespaces/environments
2. **Separation of Concerns**: Application logic (base) separated from deployment context (overlays)
3. **Scalability**: Easy to add new overlays (e.g., dev, staging, prod)
4. **Maintainability**: Changes to app logic only need to be made in base

## Usage

### Deploy All Applications

```bash
kubectl apply -k kustomize/
```

### Deploy a Specific Application

```bash
# Deploy Grafana to monitoring namespace
kubectl apply -k kustomize/overlays/grafana/monitoring

# Deploy viktutors to viktutors namespace
kubectl apply -k kustomize/overlays/viktutors/viktutors

# Deploy resume to default namespace
kubectl apply -k kustomize/overlays/resume/default

# Deploy emailer to emailer namespace
kubectl apply -k kustomize/overlays/emailer/emailer

# Deploy Prometheus to monitoring namespace
kubectl apply -k kustomize/overlays/prometheus/monitoring
```

### Preview Changes

```bash
# Preview all changes
kubectl kustomize kustomize/

# Preview specific app base
kubectl kustomize kustomize/base/grafana

# Preview specific app overlay
kubectl kustomize kustomize/overlays/grafana/monitoring
```

### Delete Resources

```bash
# Delete all
kubectl delete -k kustomize/

# Delete specific app
kubectl delete -k kustomize/overlays/grafana/monitoring
```

### Validate Configuration

```bash
# Validate all bases
for app in grafana viktutors resume emailer; do
  kubectl kustomize kustomize/base/$app > /dev/null && echo "✓ $app base is valid"
done

# Validate all overlays
kubectl kustomize kustomize/overlays/grafana/monitoring > /dev/null && echo "✓ grafana overlay is valid"
kubectl kustomize kustomize/overlays/viktutors/viktutors > /dev/null && echo "✓ viktutors overlay is valid"
kubectl kustomize kustomize/overlays/resume/default > /dev/null && echo "✓ resume overlay is valid"
kubectl kustomize kustomize/overlays/emailer/emailer > /dev/null && echo "✓ emailer overlay is valid"
kubectl kustomize kustomize/overlays/prometheus/monitoring > /dev/null && echo "✓ prometheus overlay is valid"

# Validate root
kubectl kustomize kustomize/ > /dev/null && echo "✓ Root is valid"
```

## Applications

### Grafana
- **Base**: Deployment, Service
- **Overlay**: Namespace (monitoring), Ingress (mustaci.com)
- **Namespace**: monitoring

### Prometheus
- **Base**: Deployment, Service, ConfigMap, PVC
- **Overlay**: ServiceAccount, ClusterRole/ClusterRoleBinding, Ingress (prometheus.mustaci.com)
- **Namespace**: monitoring
- **Features**:
  - Scrapes Traefik metrics for ingress monitoring
  - Scrapes Kubernetes API server, nodes, pods, and services
  - 30-day retention period
  - 10Gi persistent storage
  - RBAC configured for cluster-wide metrics collection

### Viktutors
- **Base**: Deployment, Service, ConfigMap
- **Overlay**: Namespace (viktutors), Ingress (viktutors.com)
- **Namespace**: viktutors
- **ConfigMap**: References emailer-service via FQDN

### Resume
- **Base**: Deployment, Service
- **Overlay**: Ingress (cv.mustaci.com)
- **Namespace**: default

### Emailer
- **Base**: Postgres Deployment/Service, Emailer Deployment/Service, PVC, Secret
- **Overlay**: Namespace (emailer)
- **Namespace**: emailer
- **Components**: Postgres database + Emailer service

## Cross-Namespace Communication

The viktutors ConfigMap references the emailer service using the FQDN format:
```
http://emailer-service.emailer.svc.cluster.local:80
```

This ensures proper DNS resolution across namespaces in k3s.

## Prometheus Monitoring

Prometheus is configured to monitor:

1. **Traefik Ingress Controller**: Scrapes metrics from Traefik to monitor ingress traffic, request rates, response times, and error rates
2. **Kubernetes Components**: API server, nodes, pods, and services
3. **Application Metrics**: Any pods/services annotated with `prometheus.io/scrape: "true"`

### Accessing Prometheus

- **Web UI**: Access via ingress at `prometheus.mustaci.com` (after deployment)
- **Direct Access**: `kubectl port-forward -n monitoring svc/prometheus 9090:9090`

### Configuring Grafana to Use Prometheus

After deploying both Prometheus and Grafana, configure Grafana:

1. Access Grafana at `mustaci.com`
2. Add Prometheus as a data source:
   - URL: `http://prometheus.monitoring.svc.cluster.local:9090`
   - Access: Server (default)

### Traefik Metrics

Prometheus scrapes Traefik metrics from:
- Service: `traefik.kube-system.svc.cluster.local:8080`
- Metrics endpoint: `/metrics`

Key metrics available:
- `traefik_entrypoint_requests_total` - Total requests per entrypoint
- `traefik_service_requests_total` - Total requests per service
- `traefik_router_requests_total` - Total requests per router
- `traefik_entrypoint_request_duration_seconds` - Request duration

## Best Practices Implemented

### ✅ Base/Overlays Pattern
- Application logic separated from deployment context
- Easy to create new overlays for different environments
- Base resources are reusable

### ✅ Namespace Management
- Namespaces declared in overlays, not hardcoded in base
- Kustomize automatically injects namespaces into all resources
- Single source of truth for namespace configuration

### ✅ Common Labels & Annotations
- Consistent labeling via `commonLabels`:
  - `app`: Application identifier
  - `managed-by: kustomize`: Indicates Kustomize management
  - `part-of`: Namespace/component identifier
- Common annotations track configuration origin

### ✅ Resource Organization
- Resources ordered by dependencies (Secrets/PVCs → Deployments → Services → Ingress)
- Each app is self-contained
- Clear separation between base and overlay concerns

### ✅ k3s Compatibility
- All services use `ClusterIP` (no LoadBalancer support on k3s)
- Cross-namespace service references use FQDN format
- Resource limits appropriate for Raspberry Pi

## Adding New Environments

To add a new environment (e.g., staging), create a new overlay:

```bash
mkdir -p kustomize/overlays/grafana/staging
```

Create `kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: staging

resources:
  - namespace.yaml
  - ../../../base/grafana
  - ingress.yaml
```

Then deploy:
```bash
kubectl apply -k kustomize/overlays/grafana/staging
```

## Notes

- **Secrets**: Currently contain sensitive data in plain text. For production, consider:
  - Sealed Secrets (https://github.com/bitnami-labs/sealed-secrets)
  - External Secrets Operator
  - HashiCorp Vault integration
  
- **Image Tags**: Using SHA256 digests for immutability and reproducibility

- **Resource Limits**: Configured for Raspberry Pi constraints (100m-250m CPU, 128Mi-256Mi memory)

- **Storage**: Postgres uses PVC with 1Gi storage (adjust based on needs)
