# Monitoring Stack for EKS

This Helm chart deploys the monitoring stack for the Amazon EKS project, consisting of:

- Prometheus Server (via kube-prometheus-stack)
- Alertmanager
- Grafana
- MySQL Exporter
- Elasticsearch Exporter
- Kube-State-Metrics
- Node Exporter

## Installation Prerequisites

The chart requires the following Helm repositories to be added:

```bash
# Add the Prometheus Community Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

## Important Deployment Instructions

### Pre-deployment Setup
Before deploying the monitoring stack, you must manually install the Prometheus CRDs due to an issue with large annotations in the CRDs:

```bash
# Install Prometheus CRDs manually
kubectl create -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.68.0/example/prometheus-operator-crd/monitoring.coreos.com_prometheuses.yaml
```

### Slack Webhook Configuration

The monitoring stack is configured to send alerts to Slack. To set this up:

1. Create a Kubernetes secret for the Slack webhook:

```bash
kubectl create secret generic alertmanager-slack-webhook -n monitoring \
  --from-literal=slack_webhook_url="https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
```

2. The AlertmanagerConfig in the chart will automatically use this secret.

### Manual Helm Installation (for testing)

```bash
# Create namespace
kubectl create namespace monitoring

# Install the monitoring stack
cd helm-chart/monitoring
helm dependency update
helm install prometheus . -n monitoring
```

### ArgoCD Deployment (recommended)

The monitoring stack is automatically deployed via ArgoCD using the ApplicationSet defined in `/argocd/monitoring-applicationset.yaml`.

## Storage Configuration

This chart uses gp2 storage class for persistent volumes. Ensure this storage class exists in your EKS cluster.

## Dashboards

The monitoring stack includes pre-configured dashboards for:

- Kubernetes Cluster
- MySQL
- Elasticsearch
- Node Exporter

## Alert Rules

Custom alert rules are defined for:

- MySQL connection count
- Elasticsearch cluster health
- Application error rates and response times
- Karpenter failures and pending pods
- Node resource utilization

## Access

### DNS Configuration

The monitoring stack automatically configures DNS for Grafana using External DNS:

1. The Grafana ingress is created with annotations for automatic DNS registration
2. External DNS deployment is included in the chart and will create/update the required DNS records
3. Grafana will be accessible at `grafana.<region>.blizzard.co.il` (e.g., `grafana.us-east-1.blizzard.co.il`)

Prerequisites for DNS automation:
- Route53 hosted zone for `blizzard.co.il` domain must exist in the AWS account
- The IAM role `eks-blizzard-<region>-route53-dns-manager-irsa` must be created with proper permissions
  - This role is created automatically by the eks-addons Terraform module when `create_route53_dns_manager_irsa = true`
  - The service account `route53-dns-manager` in the `monitoring` namespace must have the proper IAM role annotation
- The OIDC provider must be configured for the EKS cluster

## Troubleshooting

If you encounter issues during deployment:

1. **CRD Installation Issues**:
   - Check if Prometheus CRDs are installed: `kubectl get crd | grep prometheus`
   - If missing, manually install them with the command in the Pre-deployment Setup section
   - Make sure `prometheusOperator.createCustomResource: false` in values.yaml

2. **Slack Webhook Issues**:
   - Verify the `alertmanager-slack-webhook` secret exists: `kubectl get secret -n monitoring alertmanager-slack-webhook`
   - Check that the AlertmanagerConfig is created: `kubectl get alertmanagerconfig -n monitoring`

3. **Grafana Issues**:
   - If multiple datasources are competing for default, check values.yaml to ensure only one has `isDefault: true`

4. **Storage Issues**:
   - Check that the storage class exists: `kubectl get sc`
   - All PVCs should be using gp2 storage class (not gp3)
   - Verify PVCs are bound: `kubectl get pvc -n monitoring`

5. **Complete Cleanup (if needed)**:
   A cleanup script is provided in the repository that preserves important configuration:
   ```bash
   ./cleanup-monitoring-force.sh
   ```

## Dependency Versions

The chart depends on the following Prometheus Community charts:

- kube-prometheus-stack: 51.4.0
- prometheus-mysql-exporter: 1.14.0
- prometheus-elasticsearch-exporter: 5.2.0