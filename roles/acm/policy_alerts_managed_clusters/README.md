# policy_alerts_managed_clusters

Deploys observability alert rules, Alertmanager routing, and metric federation config to all **Managed Clusters** (Tier 3) via ACM Policy. The policy is applied from each Regional Hub and targets clusters in the `managed-clusters` ManagedClusterSet. Alert rules are evaluated by the Regional Hub's Thanos Ruler — managed clusters themselves only run a PrometheusAgent that ships metrics upstream.

## Alert flow

```
Managed Cluster local Prometheus ← PrometheusAgent federates via ScrapeConfig
PrometheusAgent → Regional Hub Thanos Receive → Thanos Ruler evaluates rules → Alertmanager → Email
```

## Key files

| File | Purpose |
|------|---------|
| `templates/custom_rules.yaml.j2` | Alert rule definitions evaluated on the Regional Hub's Thanos Ruler |
| `templates/alertmanager.yaml.j2` | Alertmanager config deployed to the Regional Hub |
| `templates/alerts-policy.yaml.j2` | ACM Policy containing all payloads (rules, alertmanager config, ScrapeConfig) |
| `templates/alerts-placement.yaml.j2` | Placement targeting the `managed-clusters` ManagedClusterSet |
| `defaults/main.yaml` | Threshold defaults and SMTP variable mapping |

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

If the new rule depends on a metric not currently federated, see **Adding metrics** below.

After editing, redeploy from the Regional Hub (Tier 2 run applies this role to each Regional Hub):

```bash
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml --tags tier2
```

## How to add an email route

Edit `templates/alertmanager.yaml.j2`. Add a matcher under `route.routes` and a corresponding receiver:

```yaml
routes:
  - matchers:
      - namespace="my-app-namespace"
    receiver: 'my-app-email'

receivers:
  - name: 'my-app-email'
    email_configs:
      - to: 'my-team@example.com'
        send_resolved: true
```

SMTP credentials come from the inventory `smtp.*` variables and are never hardcoded in the template.

## Adding metrics that rules depend on

This role uses the new MCOA (Multicluster Cluster Observability Addon) architecture. Metrics are federated from each managed cluster's local Prometheus via `ScrapeConfig` objects — **not** the legacy `observability-metrics-custom-allowlist` ConfigMap.

The policy deploys two metric sources:

1. **Default MCOA ScrapeConfig** (`platform-metrics`) — populated by the MCO operator. Contains aggregated recording rules and selected raw metrics. Do not modify this directly.

2. **Custom ScrapeConfig** (`custom-kpi-metrics`) — defined in PAYLOAD 4 of `templates/alerts-policy.yaml.j2`. This is where you add any raw metric names that the default ScrapeConfig doesn't include.

To add a metric, find the `custom-kpi-metrics` ScrapeConfig in `alerts-policy.yaml.j2` and add an entry to the `match[]` list:

```yaml
params:
  "match[]":
    - '{__name__="kube_job_status_failed"}'
    - '{__name__="my_new_metric"}'          # add here
    - '{__name__="my_new_metric",job="foo"}' # or with label filter
```

The metric name must exist in the local Prometheus on managed clusters (`prometheus-k8s.openshift-monitoring.svc:9091`). To check what's available on a managed cluster:

```bash
oc --kubeconfig=<managed-cluster-kubeconfig> get --raw \
  '/api/v1/namespaces/openshift-monitoring/services/prometheus-k8s:web/proxy/api/v1/label/__name__/values' \
  | python3 -m json.tool | grep my_metric
```

After adding the metric, redeploy (Tier 2 run) and wait one scrape interval (5 minutes) for data to appear in the Regional Hub's Thanos.

## Thresholds

Defaults are in `defaults/main.yaml` and can be overridden in the inventory:

| Variable | Default | Effect |
|----------|---------|--------|
| `spoke_offline_threshold_minutes` | `5m` | `for:` duration on `ManagedClusterOffline` |
| `storage_critical_threshold_percent` | `90` | Storage fullness threshold |
| `target_cluster_set` | `managed-clusters` | ManagedClusterSet targeted by the Placement |

## Alert scope

This role's rules cover workload and infrastructure concerns relevant to end-user clusters. Rules that require ACM, ArgoCD, Thanos internals, or Regional Hub topology awareness (e.g., `PolicyNonCompliant`, `ArgoCDAppOutOfSync`, `ManagedClusterUnavailable`) are intentionally kept in the `policy_alerts_regional_hubs` role so they can be updated independently.
