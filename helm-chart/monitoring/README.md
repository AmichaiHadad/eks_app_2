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

## Deployment Methods

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

The monitoring stack is automatically deployed via ArgoCD using the ApplicationSet:

```yaml
# Key configurations in ApplicationSet:
helm:
  repositories:
  - name: prometheus-community
    url: https://prometheus-community.github.io/helm-charts
```

## Slack Alerting Configuration

The monitoring stack is configured to send alerts to Slack. To set this up:

1. Create a secret in AWS Secrets Manager with the following details:

```bash
# Create the secret with AWS CLI
aws secretsmanager create-secret \
    --name "eks-blizzard/slack-webhook" \
    --description "Slack webhook URL for alerting" \
    --secret-string "{\"slack_webhook_url\":\"https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK\"}"
```

2. The ExternalSecret will automatically fetch this secret and configure Alertmanager.

3. The secret should have the following structure:
```json
{
  "slack_webhook_url": "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
}
```

## Dependency Versions

The chart depends on the following Prometheus Community charts:

- kube-prometheus-stack: 51.4.0
- prometheus-mysql-exporter: 1.14.0
- prometheus-elasticsearch-exporter: 5.2.0

## Storage Configuration

This chart uses GP3 storage class for persistent volumes. Ensure this storage class exists in your EKS cluster.

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

Grafana is accessible via the ingress at the URL specified in the values.yaml file. The admin password is managed by External Secrets.

## Troubleshooting

If you encounter dependency issues:

1. Verify that the Prometheus Community repository is added to ArgoCD
2. Check that the versions specified in Chart.yaml are available in the repository
3. Try manually updating dependencies:
   ```bash
   helm dependency update
   ```
4. Check ArgoCD logs for more detailed error messages