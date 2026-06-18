# policy_alerts_global_hub

Deploys observability alert rules and Alertmanager routing to the **Global Hub** (Tier 1) cluster via ACM Policy. The policy targets only the hub's `local-cluster` managed cluster object, so rules are evaluated by the Global Hub's own Thanos Ruler against data federated from all Regional Hubs below it.

## Alert flow

```
Regional Hub Thanos → Global Hub Thanos Receive → Thanos Ruler evaluates rules → Alertmanager → Email
```

## Key files

| File | Purpose |
|------|---------|
| `templates/custom_rules.yaml.j2` | All alert rule definitions (PromQL expressions, labels, annotations) |
| `templates/alertmanager.yaml.j2` | Alertmanager config: SMTP settings and email routing rules |
| `templates/alerts-policy.yaml.j2` | ACM Policy that enforces the above as ConfigMap/Secret on the hub |
| `templates/alerts-placement.yaml.j2` | Placement targeting `local-cluster` within the `default` ManagedClusterSet |
| `defaults/main.yaml` | Threshold defaults and SMTP variable mapping from inventory |

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

## How to add an email route

Edit `templates/alertmanager.yaml.j2`. Add a matcher under `route.routes` and a corresponding receiver:

```yaml
routes:
  - matchers:
      - team="my-team"
    receiver: 'my-team-email'

receivers:
  - name: 'my-team-email'
    email_configs:
      - to: 'my-team@example.com'
        send_resolved: true
```

SMTP credentials come from the inventory `smtp.*` variables and are never hardcoded in the template.

## Thresholds

Defaults are set in `defaults/main.yaml` and can be overridden in the inventory or at role invocation:

| Variable | Default | Effect |
|----------|---------|--------|
| `spoke_offline_threshold_minutes` | `5m` | `for:` duration on `ManagedClusterOffline` |
| `storage_critical_threshold_percent` | `90` | Storage fullness threshold |

## Metrics available for rules

The Global Hub's Thanos receives federated data from Regional Hubs. Metrics available are whatever the Regional Hub's own MCOA ScrapeConfigs collect, plus any custom ScrapeConfigs deployed to Regional Hubs via the `policy_alerts_regional_hubs` role. If a new alert rule needs a metric not already present, add it to the `policy_alerts_regional_hubs` ScrapeConfig — that data will flow up to the Global Hub automatically.
