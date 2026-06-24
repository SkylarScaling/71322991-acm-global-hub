# Alert Architecture

This document describes how OCP and ACM alerts are structured, how they flow through the
three-tier Global Hub hierarchy, and how the automation roles implement that structure.

---

## Two alerting stacks

Every cluster in this architecture runs two independent alerting stacks. They evaluate different
rules against different data sources and deliver notifications through separate Alertmanager
instances. Both are configured by the automation.

| Stack | Prometheus source | Alertmanager | Manages |
|---|---|---|---|
| **OCP platform** | Local cluster Prometheus (`openshift-monitoring`) | `alertmanager-main` in `openshift-monitoring` | OCP built-in rules + ACM operator rules |
| **MCO (observability)** | Thanos Ruler (`open-cluster-management-observability`) | `alertmanager-config` in `open-cluster-management-observability` | Custom rules defined in this automation |

The platform stack fires on OCP's built-in PrometheusRules and any rules that ACM installs into
the cluster. The MCO stack fires on the custom rules defined in the `custom_rules.yaml.j2`
templates, which are evaluated against federated metrics from downstream clusters.

---

## Three-tier alert hierarchy

The three-tier cluster hierarchy determines both where rules are evaluated and what data those
rules can see.

```
┌─────────────────────────────────────────────────────────────────┐
│  Tier 1 — Global Hub                                            │
│                                                                 │
│  OCP Platform Alertmanager          MCO Alertmanager            │
│  (alertmanager-main)                (alertmanager-config)       │
│       │                                    │                    │
│  Local OCP/ACM rules            Thanos Ruler (custom rules)     │
│                                       │                         │
│                              Thanos Receive ←─────────────────┐ │
└─────────────────────────────────────────────────────────────┐ │ │
                                                              │ │ │
┌─────────────────────────────────────────────────────────────┘ │ │
│  Tier 2 — Regional Hubs (one per region)                      │ │
│                                                               │ │
│  OCP Platform Alertmanager          MCO Alertmanager          │ │
│  (alertmanager-main)                (alertmanager-config)     │ │
│       │                                    │                  │ │
│  Local OCP/ACM rules            Thanos Ruler (custom rules)   ┘ │
│                                       │                         │
│                              Thanos Receive ←────────────────┐  │
└──────────────────────────────────────────────────────────────│──┘
                                                               │
┌──────────────────────────────────────────────────────────────┘
│  Tier 3 — Managed Clusters (many per Regional Hub)
│
│  OCP Platform Alertmanager          PrometheusAgent (MCOA)
│  (alertmanager-main)                      │
│       │                          ScrapeConfig federates
│  Local OCP/ACM rules             metrics to Regional Hub
│  (no custom rules here)          (no local Thanos Ruler)
└─────────────────────────────────────────────────────────────────
```

### Alert evaluation per tier

**Tier 1 — Global Hub**

The Global Hub's Thanos Ruler evaluates custom rules against aggregated data from all Regional
Hubs. Rules at this tier are oriented toward the overall health of the hub topology: Regional Hub
connectivity, policy compliance across the fleet, and the health of the MCO/observability
infrastructure itself (Thanos query latency, rule evaluation duration, etc.).

The OCP platform Alertmanager fires on local OCP and ACM operator rules — the same built-in
ruleset present on any OCP cluster, plus ACM-specific PrometheusRules installed by the ACM
operator.

**Tier 2 — Regional Hubs**

Each Regional Hub's Thanos Ruler evaluates custom rules against federated data from its managed
clusters. Rules at this tier cover the full stack: node health, storage, etcd, API server, pod
health, ArgoCD, and managed cluster status. The Regional Hub has visibility into everything its
managed clusters report, which makes it the primary location for infrastructure alerting.

The OCP platform Alertmanager is configured identically to the Global Hub for local OCP/ACM rules.

**Tier 3 — Managed Clusters**

Managed clusters do not run Thanos or a Thanos Ruler. A PrometheusAgent (MCOA architecture)
federates selected metrics from the local Prometheus up to the assigned Regional Hub via the
`custom-kpi-metrics` ScrapeConfig. All custom rule evaluation for managed cluster data happens on
the Regional Hub, not on the managed cluster itself.

The OCP platform Alertmanager is present on managed clusters (it is part of every OCP
installation) but is not configured by this automation beyond OCP's own defaults. Platform alerts
from managed clusters are visible in the OCP console but are not wired into the centralized
notification channels.

---

## MCO alert data flow

### Managed Cluster → Regional Hub

```
Managed Cluster
  local Prometheus (prometheus-k8s.openshift-monitoring.svc:9091)
       │
       │  federate (MCOA ScrapeConfig: custom-kpi-metrics)
       │  scrapes specific {__name__="..."} selectors
       ▼
  PrometheusAgent (open-cluster-management-agent-addon)
       │
       │  remote_write
       ▼
Regional Hub
  Thanos Receive (open-cluster-management-observability)
       │
       │  stored in Thanos object storage
       ▼
  Thanos Ruler evaluates custom_rules.yaml rules
       │
       │  alert fires
       ▼
  MCO Alertmanager (alertmanager-config secret)
       │
       ├── email
       └── webhook
```

### Regional Hub → Global Hub

```
Regional Hub
  Thanos (open-cluster-management-observability)
       │
       │  remote_write (MCO Thanos federation)
       ▼
Global Hub
  Thanos Receive (open-cluster-management-observability)
       │
       │  stored in Thanos object storage
       ▼
  Thanos Ruler evaluates custom_rules.yaml rules
       │
       │  alert fires
       ▼
  MCO Alertmanager (alertmanager-config secret)
       │
       ├── email
       └── webhook
```

### Metric federation: MCOA ScrapeConfig

The MCOA (Multicluster Cluster Observability Addon) architecture uses `ScrapeConfig` objects in the
`open-cluster-management-agent-addon` namespace on each managed cluster. The
`policy_alerts_managed_clusters` role deploys a `custom-kpi-metrics` ScrapeConfig (PAYLOAD 4) that
defines exactly which metrics are federated. Only metrics listed in its `match[]` selectors reach
the Regional Hub's Thanos and are available for rule evaluation.

A legacy `observability-metrics-custom-allowlist` ConfigMap (PAYLOAD 3) is also deployed for
backward compatibility with older MCO versions, but MCOA does not use it.

---

## OCP platform alert data flow

```
Any cluster (Tier 1, 2, or 3)
  local Prometheus (openshift-monitoring)
       │
       │  evaluates OCP built-in PrometheusRules
       │  + ACM operator PrometheusRules
       │
       │  alert fires
       ▼
  OCP platform Alertmanager (alertmanager-main secret, openshift-monitoring)
       │
       ├── email   (configured by automation on Tier 1 and Tier 2)
       └── webhook (configured by automation on Tier 1 and Tier 2)
```

The OCP platform Alertmanager on Tier 1 and Tier 2 clusters is fully configured by this
automation (see [Automation implementation](#automation-implementation) below). Tier 3 managed
clusters retain OCP's default platform alertmanager configuration and are not wired into
centralized notification channels.

---

## Alert rule scope by tier

Each tier's `custom_rules.yaml.j2` is intentionally scoped to what that tier can see.

| Category | Global Hub | Regional Hubs | Managed Clusters |
|---|:---:|:---:|:---:|
| Cluster offline / unreachable | ✓ | ✓ | — |
| ACM policy non-compliance | ✓ | ✓ | — |
| Node health (CPU, memory, disk) | — | ✓ | — |
| etcd health | — | ✓ | — |
| API server latency / errors | — | ✓ | — |
| Pod crash / OOM / not ready | — | ✓ | — |
| Storage (PV fill rate) | — | ✓ | — |
| Kubelet certificate expiry | — | ✓ | — |
| Backup (Velero) | — | ✓ | — |
| Security (ACS vulnerabilities) | — | ✓ | — |
| ArgoCD sync / degraded | — | ✓ | — |
| MCO / Thanos internals | ✓ | ✓ | — |

Managed cluster rules are evaluated on the Regional Hub (using data federated via ScrapeConfig),
so they appear in the "Regional Hubs" column rather than having their own column.

---

## Notification channels

Three notification channels are available for the OCP platform Alertmanager (`alertmanager-main`
on Tier 1 and Tier 2 clusters). The MCO Alertmanager uses the same `email` and `webhook`
settings for its own routing.

### Default receiver (email / webhook)

Catches all alerts not matched by a more specific route. Email and webhook are independently
toggled and can both be active simultaneously.

```yaml
# inventory.yaml
all:
  vars:
    alerts:
      email:
        enabled: true
        host: "smtp.gmail.com"
        port: "587"
        from_address: "alerts@example.com"
        require_tls: "true"
        username: "alerts@example.com"
        app_password: "<app password>"
        default_to: "oncall@example.com"
        esp_to: "platform-team@example.com"
      webhook:
        enabled: false
        url: ""  # e.g. https://your-instance.service-now.com/api/global/em/jsonv2
```

Set `email.enabled: false` to make the Default receiver a dead-letter (catches alerts but sends
no notifications). This is appropriate when only the Critical receiver is needed.

### Critical receiver (OCP platform Alertmanager only)

An optional dedicated receiver for `severity=critical` alerts. It routes to a separate list of
recipients with tighter repeat and group intervals than the Default receiver. It uses per-receiver
SMTP configuration, so it can target a different mail relay (e.g. an unauthenticated internal
relay) independently of the Default receiver's global SMTP.

The `from` address is rendered per-cluster at notification time using Alertmanager's
`{{ $externalLabels.cluster }}` label, so recipients can immediately identify the originating
cluster without needing separate configurations per cluster.

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
        smtp_host: "internal-relay.example.com:25"  # omit to fall back to global SMTP
        smtp_from_prefix: "no-reply-critical-"       # } combined at send time:
        smtp_from_suffix: "@example.com"             # } no-reply-critical-<cluster>@example.com
        smtp_require_tls: false
        group_interval: "10m"
        repeat_interval: "24h"
```

When the Critical receiver is enabled, the routing order in `alertmanager-main` is:

```
silence routes (alertname match → null)   ← evaluated first; silenced alerts stop here
  ↓ no match
severity=critical route → Critical receiver
  ↓ no match
Watchdog → Watchdog receiver (dead-letter)
InfoInhibitor → null receiver
  ↓ no match
Default receiver (email / webhook)
```

Silenced alerts are always suppressed, even if they carry `severity=critical`.

---

## Silencing alerts

OCP platform alerts (from `alertmanager-main`) can be silenced without removing the underlying
PrometheusRule. Silenced alerts remain visible in the OCP console but are routed to a null
receiver and generate no notifications.

Silences are configured in inventory and take effect the next time the relevant `policy_alerts_*`
role is applied:

```yaml
# inventory.yaml
all:
  vars:
    # OCP built-in alerts (Watchdog, InfoInhibitor, CPUThrottlingHigh, etc.)
    ocp_silences:
      - name: Watchdog

    # ACM operator alerts (SearchPVCNotPresent, MultiClusterObservabilityAddonDegraded, etc.)
    acm_silences:
      - name: SearchPVCNotPresent

    # Environment-specific alerts, with optional label narrowing
    custom_silences:
      - name: NodeFilesystemSpaceFillingUp
        extra_matchers:
          - 'severity="warning"'
```

Silence routes are inserted before the notification routes in `alertmanager-main`, so they take
priority regardless of severity. An alert that matches a silence entry is never delivered to email
or webhook.

MCO Alertmanager alerts (evaluated by Thanos Ruler from `custom_rules.yaml.j2`) cannot be silenced
this way — those rules should be modified or removed directly from the template if they are not
needed.

---

## Automation implementation

### Roles

Three roles deploy the alert infrastructure, one per tier:

| Role | Applied from | Targets | What it deploys |
|---|---|---|---|
| `acm/policy_alerts_global_hub` | Global Hub | `local-cluster` only | MCO alertmanager, custom rules, metrics allowlist, OCP platform alertmanager |
| `acm/policy_alerts_regional_hubs` | Global Hub | `regional-hubs` cluster set | MCO alertmanager, custom rules, metrics allowlist, OCP platform alertmanager |
| `acm/policy_alerts_managed_clusters` | Each Regional Hub | `managed-clusters` cluster set | MCO alertmanager, custom rules, metrics allowlist (legacy), MCOA ScrapeConfig |

Each role deploys an ACM `Policy` containing a single `ConfigurationPolicy` with multiple payloads.
The `ConfigurationPolicy` uses `remediationAction: enforce` and `complianceType: musthave`, so ACM
continuously reconciles the target resources back to the declared state.

**Payload structure (Tier 1 and Tier 2):**

```
Policy
└── ConfigurationPolicy (enforce)
    ├── PAYLOAD 1: alertmanager-config Secret     → MCO Alertmanager config (email/webhook)
    ├── PAYLOAD 2: thanos-ruler-custom-rules CM   → Custom alert rules (evaluated by Thanos Ruler)
    ├── PAYLOAD 3: observability-metrics-allowlist → Federated metrics list
    └── PAYLOAD 4: alertmanager-main Secret       → OCP platform Alertmanager config (email/webhook + silences)
```

**Payload structure (Tier 3):**

```
Policy
└── ConfigurationPolicy (enforce)
    ├── PAYLOAD 1: alertmanager-config Secret     → MCO Alertmanager config (email/webhook)
    ├── PAYLOAD 2: thanos-ruler-custom-rules CM   → Custom alert rules
    ├── PAYLOAD 3: observability-metrics-allowlist → Legacy metrics list (backward compat)
    └── PAYLOAD 4: custom-kpi-metrics ScrapeConfig → MCOA metric federation selectors
```

Note that Tier 3 does not include a PAYLOAD 4 for `alertmanager-main` — managed cluster platform
alertmanager is not managed by this automation.

### Templates

Each role renders its payloads from Jinja2 templates at Ansible execution time:

| Template | Used by | Renders |
|---|---|---|
| `alertmanager.yaml.j2` | All three roles | MCO Alertmanager config (SMTP settings, receivers, routes) |
| `ocp-alertmanager-config.yaml.j2` | Tier 1, Tier 2 | OCP platform Alertmanager config (SMTP, silence routes, OCP inhibit rules) |
| `custom_rules.yaml.j2` | All three roles | Prometheus/Thanos Ruler alert rules (PromQL) |

Prometheus template syntax (`{{ $labels.cluster }}`, etc.) inside alert annotations is protected
from Jinja2 interpolation using `{% raw %}...{% endraw %}` blocks.

### Playbook deployment sequence

**Full environment setup** (`full-global-hub-setup.yaml` or `full-global-hub-ipi-setup.yaml`):

```
Tier 1 (hosts: hub_cluster)
  ...cluster provisioning...
  → policy_alerts_global_hub    (kubeconfig: global hub)
  → policy_alerts_regional_hubs (kubeconfig: global hub)
  ...

Tier 2 (hosts: regional_hubs)
  ...cluster provisioning...
  → policy_alerts_managed_clusters (kubeconfig: each regional hub)
  ...
```

`policy_alerts_global_hub` and `policy_alerts_regional_hubs` both run against the Global Hub's
kubeconfig. ACM then propagates the resulting policies to the target clusters via the Placement
rules. `policy_alerts_managed_clusters` runs against each Regional Hub's kubeconfig and ACM
propagates it to that hub's managed clusters.

**Standalone alert update** (without reprovisioning clusters):

```bash
# Update Global Hub + Regional Hub alert config
ansible-playbook playbooks/acm/acm-global-hub-alerts-install.yaml -i inventory.yaml

# Update Managed Cluster alert config (runs Tier 2, which includes policy_alerts_managed_clusters)
ansible-playbook playbooks/full-global-hub-setup.yaml -i inventory.yaml --tags tier2
```

### Configuration flow: inventory → policy → cluster

```
inventory.yaml
  alerts.email.*       ──►  alerting_smtp_* vars (role defaults)  ──►  alertmanager.yaml.j2
  alerts.webhook.*     ──►  alerting_webhook_* vars               ──►  alertmanager.yaml.j2
                                                                   ──►  ocp-alertmanager-config.yaml.j2
  ocp_silences         ──────────────────────────────────────────► ocp-alertmanager-config.yaml.j2
  acm_silences         ──────────────────────────────────────────► ocp-alertmanager-config.yaml.j2
  custom_silences      ──────────────────────────────────────────► ocp-alertmanager-config.yaml.j2
        │
        ▼
  Ansible renders templates → base64 encodes → embeds in ACM Policy YAML
        │
        ▼
  kubernetes.core.k8s applies Policy to hub cluster
        │
        ▼
  ACM propagates ConfigurationPolicy to target clusters (via Placement)
        │
        ▼
  ACM enforces Secret/ConfigMap on each target cluster
        │
        ▼
  Alertmanager / Thanos Ruler reloads config
```
