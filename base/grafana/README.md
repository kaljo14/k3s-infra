# Grafana Base Configuration

This directory contains the base Grafana configuration that is environment-agnostic.

## Resources

### Core Components
- **deployment.yaml**: Grafana deployment with basic configuration
- **service.yaml**: ClusterIP service exposing Grafana on port 3000

### Datasource Configuration
- **datasource.yaml**: ConfigMap that automatically configures Prometheus as a datasource
  - Points to: `http://prometheus.monitoring.svc.cluster.local:9090`
  - Set as default datasource
  - Configured with appropriate timeouts

### Dashboard Provider
- **dashboard-provider.yaml**: ConfigMap that configures Grafana to automatically load dashboards from ConfigMaps
  - Enables dashboard provisioning
  - Looks for dashboards in `/var/lib/grafana/dashboards`

## What Was Added

### Prometheus Integration
The Grafana deployment has been configured to automatically connect to Prometheus:

1. **Datasource Configuration**: Prometheus is automatically configured as the default datasource
   - No manual configuration needed in Grafana UI
   - Automatically available when Grafana starts
   - Connection tested and validated

2. **Dashboard Provisioning**: Grafana is configured to automatically load dashboards from ConfigMaps
   - Dashboards are defined as ConfigMaps with label `grafana_dashboard: "1"`
   - Automatically loaded on startup
   - No manual import needed

### Volume Mounts
The deployment mounts three ConfigMaps:
- `/etc/grafana/provisioning/datasources`: Datasource configuration
- `/etc/grafana/provisioning/dashboards`: Dashboard provider configuration
- `/var/lib/grafana/dashboards`: Dashboard JSON files

## Environment-Specific Configuration

Dashboard configurations (like the Traefik ingress dashboard) are located in the overlays directory:
- `overlays/grafana/monitoring/`: Contains environment-specific dashboards

This separation allows:
- Different dashboards per environment (dev, staging, prod)
- Environment-specific visualizations
- Flexible dashboard management

## Usage

After deployment, Grafana will:
1. Automatically connect to Prometheus (no manual configuration needed)
2. Load all dashboards from the overlay ConfigMaps
3. Be ready to use immediately

Access Grafana via the ingress at `mustaci.com` (or your configured host).

## Customization

To add more datasources or dashboards:
- **Datasources**: Add to `datasource.yaml` (if shared across environments) or overlay
- **Dashboards**: Add to overlay directory as ConfigMaps with label `grafana_dashboard: "1"`

