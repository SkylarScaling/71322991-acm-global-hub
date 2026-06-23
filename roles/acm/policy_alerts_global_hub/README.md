# policy_alerts_global_hub

Deploys observability alert rules and Alertmanager routing to the **Global Hub** (Tier 1) cluster via ACM Policy. The policy targets only the hub's `local-cluster` managed cluster object, so rules are evaluated by the Global Hub's own Thanos Ruler against data federated from all Regional Hubs below it.

## Alert flow

```
Regional Hub Thanos → Global Hub Thanos Receive → Thanos Ruler evaluates rules → Alertmanager → Email / Webhook
```

## Key files

| File | Purpose |
|------|---------|
| `templates/custom_rules.yaml.j2` | All alert rule definitions (PromQL expressions, labels, annotations) |
| `templates/alertmanager.yaml.j2` | MCO Alertmanager config: notification channels (email, webhook) and routing rules |
| `templates/ocp-alertmanager-config.yaml.j2` | OCP platform Alertmanager config: notification receivers and silence routes |
| `templates/alerts-policy.yaml.j2` | ACM Policy that enforces the above as ConfigMap/Secret on the hub |
| `templates/alerts-placement.yaml.j2` | Placement targeting `local-cluster` within the `default` ManagedClusterSet |
| `defaults/main.yaml` | Threshold defaults, SMTP variable mapping, and webhook settings from inventory |

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

After editing, redeploy with:

```bash
ansible-playbook playbooks/acm/acm-global-hub-alerts-install.yaml -i inventory.yaml
```

Or during full environment setup, Tier 1 handles this automatically:

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

This role manages the OCP platform Alertmanager (`openshift-monitoring/alertmanager-main`) in addition to the MCO Alertmanager. Any alert that fires through OCP platform monitoring — including ACM's own PrometheusRules — can be silenced by adding it to one of three inventory lists. Silenced alerts are routed to a null receiver and never reach your notification channels.

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

Defaults are set in `defaults/main.yaml` and can be overridden in the inventory or at role invocation:

| Variable | Default | Effect |
|----------|---------|--------|
| `spoke_offline_threshold_minutes` | `5m` | `for:` duration on `ManagedClusterOffline` |
| `storage_critical_threshold_percent` | `90` | Storage fullness threshold |
| `alerting_email_enabled` | `true` | Send alerts via email |
| `alerting_webhook_enabled` | `false` | Send alerts via webhook |
| `alerting_webhook_url` | `webhook.url` from inventory | Webhook endpoint URL |
| `ocp_silences` | `[]` | OCP platform alerts to silence |
| `acm_silences` | `[]` | ACM-specific alerts to silence |
| `custom_silences` | `[]` | Additional environment-specific alerts to silence |

## Metrics available for rules

The Global Hub's Thanos receives federated data from Regional Hubs. Metrics available are whatever the Regional Hub's own MCOA ScrapeConfigs collect, plus any custom ScrapeConfigs deployed to Regional Hubs via the `policy_alerts_regional_hubs` role. If a new alert rule needs a metric not already present, add it to the `policy_alerts_regional_hubs` ScrapeConfig — that data will flow up to the Global Hub automatically.
