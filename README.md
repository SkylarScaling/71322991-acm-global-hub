# ACM Global Hub Automation

Ansible automation for deploying and configuring a multi-tier **Red Hat Advanced Cluster Management (ACM) Global Hub** architecture on AWS OpenShift, with a focus on centralized Observability and Alerting.

## Architecture

The environment is a three-tier cluster hierarchy:

```
Global Hub (Tier 1)
├── Regional Hub 1 (Tier 2)
│   ├── Managed Cluster 1 (Tier 3)
│   └── Managed Cluster N (Tier 3)
└── Regional Hub 2 (Tier 2)
    ├── Managed Cluster N+1 (Tier 3)
    └── Managed Cluster N+M (Tier 3)
```

| Tier | Role | Key Components |
|------|------|----------------|
| **Global Hub** | Central management plane | ACM Global Hub, MCO, ODF, Grafana KPI dashboards |
| **Regional Hubs** | Regional management + observability aggregation | ACM, MCO, ODF, Thanos, Alertmanager |
| **Managed Clusters** | End-user workload clusters | PrometheusAgent (MCOA), GitOps, ACM addons |

All clusters are provisioned on AWS and configured post-deployment via OpenShift GitOps (ArgoCD).

## Prerequisites

- Python 3+ and `pip3 install ansible`
- OpenShift CLI (`oc`), AWS CLI, OpenShift Installer CLI
- AWS credentials in `~/.aws/`
- Red Hat pull secret
- Ansible Python modules: `kubernetes`, `openshift`, `boto3`

Install CLI prerequisites automatically:
```bash
ansible-playbook roles/ocp/install_ocp_prerequisites/tasks/main.yaml --ask-become-pass
```

## Quick Start

See **[playbooks/README.md](playbooks/README.md)** for the full inventory schema, example values, and common deployment commands.

### Deploy the full environment

```bash
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml
```

### Deploy a single tier

```bash
# Tier 1 only (Global Hub)
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml --tags tier1

# Tier 2 only (Regional Hubs)
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml --tags tier2

# Tier 3 only (Managed Clusters)
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml --tags tier3
```

### IPI variant

If provisioning all tiers via OpenShift Installer (IPI) rather than ACM/Hive:
```bash
ansible-playbook playbooks/full-global-hub-ipi-setup.yaml -i inventory.yaml
```

## Repository Structure

```
playbooks/
  full-global-hub-setup.yaml       # Main entrypoint — all three tiers
  full-global-hub-ipi-setup.yaml   # IPI variant
  acm/
    acm-global-hub-alerts-install.yaml  # Standalone alerts-only deployment
  test/
    trigger-alerts.yaml            # Deploy/remove alert test workloads

roles/
  acm/
    aws_provision_cluster_via_acm/ # Provision clusters via ACM/Hive
    policy_alerts_global_hub/      # Tier 1 alert rules & Alertmanager config
    policy_alerts_regional_hubs/   # Tier 2 alert rules & Alertmanager config
    policy_alerts_managed_clusters/# Tier 3 alert rules & metric federation
    global_hub_custom_dashboards/  # Grafana KPI dashboards
    policy_enable_uwm/             # User Workload Monitoring policy
    policy_install_openshift_gitops_*/  # GitOps installation policies
    ...
  argo/
    apps_install_acm/              # ArgoCD app for ACM operator
    apps_install_mco/              # ArgoCD app for Multicluster Observability
    apps_install_odf/              # ArgoCD app for OpenShift Data Foundation
    apps_install_global_hub/       # ArgoCD app for ACM Global Hub
    ...
  ocp/
    aws_provision_ocp_ipi/         # Provision clusters via IPI
    install_openshift_gitops/      # Install GitOps operator
    aws_create_machineset/         # Create infra/storage node pools
    ...
  aws/
    aws_eip_quota/                 # Request Elastic IP quota increase
```

## Observability & Alerting

Each tier has independent alert rules and Alertmanager routing, managed by separate Ansible roles so they can be updated without affecting the others. Alerts are evaluated by the Thanos Ruler on each Regional Hub and delivered via email through Alertmanager.

| Role | Targets | README |
|------|---------|--------|
| `policy_alerts_global_hub` | Global Hub (`local-cluster`) | [roles/acm/policy_alerts_global_hub/README.md](roles/acm/policy_alerts_global_hub/README.md) |
| `policy_alerts_regional_hubs` | All Regional Hubs | [roles/acm/policy_alerts_regional_hubs/README.md](roles/acm/policy_alerts_regional_hubs/README.md) |
| `policy_alerts_managed_clusters` | All Managed Clusters | [roles/acm/policy_alerts_managed_clusters/README.md](roles/acm/policy_alerts_managed_clusters/README.md) |

SMTP configuration is set once in the inventory under the `smtp:` key and shared across all three roles. See [playbooks/README.md](playbooks/README.md) for the full schema.

### Deploying alerts only

To update alert rules or Alertmanager config without redeploying clusters:

```bash
# Update Global Hub and Regional Hub alert policies
ansible-playbook playbooks/acm/acm-global-hub-alerts-install.yaml -i inventory.yaml

# Update Managed Cluster alert policies (runs as part of Tier 2)
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml --tags tier2
```

### Testing alerts

A test workload playbook deploys pods and jobs that trigger several alert rules:

```bash
# Deploy test workloads
ansible-playbook playbooks/test/trigger-alerts.yaml -i inventory.yaml

# Clean up
ansible-playbook playbooks/test/trigger-alerts.yaml -i inventory.yaml -e state=absent
```
