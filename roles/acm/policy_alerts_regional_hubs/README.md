# policy_alerts_regional_hubs

Deploys observability alert rules and Alertmanager routing to all **Regional Hub** (Tier 2) clusters via ACM Policy. The policy is applied from the Global Hub and targets clusters in the `regional-hubs` ManagedClusterSet. Each Regional Hub's Thanos Ruler evaluates rules against metrics federated from its managed clusters.

## Alert flow

```
Managed Cluster PrometheusAgent → Regional Hub Thanos Receive → Thanos Ruler evaluates rules → Alertmanager → Email / Webhook
```

## Key files

| File | Purpose |
|------|---------|
| `templates/custom_rules.yaml.j2` | All alert rule definitions (PromQL expressions, labels, annotations) |
| `templates/alertmanager.yaml.j2` | MCO Alertmanager config: notification channels (email, webhook) and routing rules |
| `templates/ocp-alertmanager-config.yaml.j2` | OCP platform Alertmanager config: notification receivers and silence routes |
| `templates/alerts-policy.yaml.j2` | ACM Policy that enforces the above as ConfigMap/Secret on each Regional Hub |
| `templates/alerts-placement.yaml.j2` | Placement targeting the `regional-hubs` ManagedClusterSet |
| `defaults/main.yaml` | Threshold defaults, SMTP variable mapping, webhook settings, and cluster label selector |

## How to add or modify an alert rule

Edit `templates/custom_rules.yaml.j2`. Each rule follows standard Prometheus syntax:

```yaml
- alert: MyNewAlert
  expr: some_metric{label="value"} > threshold
  for: 5m
  labels:
    severity: warning
    team: platform-ops
  annotations:
    summary: "Something happened on {% raw %}{{ $labels.cluster }}{% endraw %}"
    description: "Detail: {% raw %}{{ $value }}{% endraw %}"
```

**Important:** Any `{{ }}` in annotations or expressions must be wrapped in `{% raw %}...{% endraw %}` to prevent Jinja2 from interpreting them before the template is rendered into the ConfigMap.

After editing, redeploy from the Global Hub:

```bash
ansible-playbook playbooks/acm/acm-global-hub-alerts-install.yaml -i inventory.yaml
```

Or as part of a full Tier 1 run:

```bash
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml --tags tier1
```

## Notification channels

Email and webhook notifications are independently toggled. Both can be active at the same time — Alertmanager will fire all configured notifiers in a receiver.

### Email (default: enabled)

Controlled by `alerting_email_enabled` (default `true`). SMTP credentials come from the inventory `smtp.*` variables.

To disable email and use only webhook:

```yaml
# inventory.yaml
all:
  vars:
    alerts:
      email:
        enabled: false
```

### Webhook / ServiceNow (default: disabled)

Controlled by `alerts.webhook.enabled` (default `false`) and `alerts.webhook.url`. Set both in the inventory:

```yaml
# inventory.yaml
all:
  vars:
    alerts:
      webhook:
        enabled: true
        url: "https://your-instance.service-now.com/api/global/em/jsonv2"
```

### Critical receiver (default: disabled)

A separate `Critical` receiver can be enabled for `severity=critical` alerts. It uses per-receiver
SMTP configuration, which allows a different mail relay, no-auth open relay, or different TLS
settings than the `Default` receiver's global SMTP. Critical alerts use tighter repeat and group
intervals than the defaults.

The `from` address automatically includes the cluster name via Alertmanager's
`{{ $externalLabels.cluster }}` label, so notifications identify the originating cluster without
requiring separate configs per cluster.

```yaml
# inventory.yaml
all:
  vars:
    alerts:
      critical:
        enabled: true
        recipients:
          - "itsm-eng@example.com"
          - "itsm-ops@example.com"
        smtp_host: "internal-relay.example.com:25"
        smtp_from_prefix: "no-reply-critical-"   # cluster name inserted here
        smtp_from_suffix: "@example.com"          # → no-reply-critical-<cluster>@example.com
        smtp_require_tls: false
        group_interval: "10m"
        repeat_interval: "24h"
```

### Adding a routing rule

Edit `templates/alertmanager.yaml.j2`. Add a matcher under `route.routes` and a corresponding receiver. Include whichever notifier blocks apply:

```yaml
routes:
  - matchers:
      - team="my-team"
    receiver: 'my-team'

receivers:
  - name: 'my-team'
    email_configs:
      - to: 'my-team@example.com'
        send_resolved: true
    webhook_configs:
      - url: 'https://hooks.example.com/my-team'
        send_resolved: true
```

## Silencing OCP platform alerts

This role manages the OCP platform Alertmanager (`openshift-monitoring/alertmanager-main`) on each Regional Hub in addition to the MCO Alertmanager. Any alert that fires through OCP platform monitoring — including ACM's own PrometheusRules — can be silenced by adding it to one of three inventory lists. Silenced alerts are routed to a null receiver and never reach your notification channels.

```yaml
# inventory.yaml
all:
  vars:
    # OCP built-in alerts to silence
    ocp_silences:
      - name: Watchdog
      - name: AlertmanagerReceiversNotConfigured
      - name: CPUThrottlingHigh

    # ACM-specific alerts to silence
    acm_silences:
      - name: MultiClusterObservabilityAddonDegraded
      - name: SearchPVCNotPresent

    # Any other alerts to silence (environment-specific)
    custom_silences:
      - name: NodeFilesystemSpaceFillingUp
        extra_matchers:
          - 'severity="warning"'
```

Each entry requires `name` (the exact `alertname` label value). The optional `extra_matchers` list accepts additional Alertmanager matcher strings to narrow which instances of that alert are silenced.

Redeploy after changing silence lists:

```bash
ansible-playbook playbooks/acm/acm-global-hub-alerts-install.yaml -i inventory.yaml
```

## Thresholds

Defaults are in `defaults/main.yaml` and can be overridden in the inventory:

| Variable | Default | Effect |
|----------|---------|--------|
| `spoke_offline_threshold_minutes` | `5m` | `for:` duration on `ManagedClusterOffline` |
| `storage_critical_threshold_percent` | `90` | Storage fullness threshold |
| `alerting_email_enabled` | `true` | Send alerts via email |
| `alerting_webhook_enabled` | `false` | Send alerts via webhook |
| `alerting_webhook_url` | `webhook.url` from inventory | Webhook endpoint URL |
| `regional_hub_cluster_label_key` | `global-hub.open-cluster-management.io/managed-hub` | Label used by the Placement to select Regional Hubs |
| `regional_hub_cluster_label_value` | `true` | Expected value for the label above |
| `ocp_silences` | `[]` | OCP platform alerts to silence |
| `acm_silences` | `[]` | ACM-specific alerts to silence |
| `custom_silences` | `[]` | Additional environment-specific alerts to silence |
| `alerting_critical_email_enabled` | `false` | Enable the Critical receiver for severity=critical alerts |
| `alerting_critical_email_recipients` | `[]` | List of email addresses for the Critical receiver |
| `alerting_critical_smtp_host` | `""` | SMTP host for Critical receiver (empty = use global SMTP) |
| `alerting_critical_smtp_from_prefix` | `"no-reply-"` | From address prefix (cluster name appended automatically) |
| `alerting_critical_smtp_from_suffix` | `"@example.com"` | From address domain suffix |
| `alerting_critical_smtp_require_tls` | `false` | Require TLS for Critical receiver SMTP |
| `alerting_critical_group_interval` | `"10m"` | How long to wait before re-sending grouped critical alerts |
| `alerting_critical_repeat_interval` | `"24h"` | How long before re-notifying on a firing critical alert |

## Adding metrics that rules depend on

The Regional Hub's Thanos gets its data from the managed cluster PrometheusAgents via remote_write. The metrics available are controlled by MCOA `ScrapeConfig` objects in the `open-cluster-management-agent-addon` namespace on each managed cluster.

The `policy_alerts_managed_clusters` role deploys a `custom-kpi-metrics` ScrapeConfig to managed clusters that covers the metrics this role's rules depend on. If you add a rule here that uses a metric not currently federated, add it to the `match[]` list in `roles/acm/policy_alerts_managed_clusters/templates/alerts-policy.yaml.j2` under the `custom-kpi-metrics` ScrapeConfig payload, then redeploy that role as well.

To check which metrics are currently available in Thanos on a Regional Hub:

```bash
oc --kubeconfig=<regional-hub-kubeconfig> get --raw \
  '/api/v1/namespaces/open-cluster-management-observability/services/http:observability-thanos-query:9090/proxy/api/v1/query?query=<metric_name>'
```
