# Manifests

Kubernetes manifests that install the Resource Review mixin into a cluster that already runs Prometheus Operator and Grafana.

| File | Kind | Purpose |
|---|---|---|
| `prometheus-rule.yaml` | `PrometheusRule` | 22 recording rules (`resource_review:*`), evaluated every minute. No alerts. |
| `dashboard-configmap.yaml` | `ConfigMap` | The "Kubernetes Resource Review" Grafana dashboard, loaded by the Grafana dashboard sidecar. |

Both files are generated from the Jsonnet sources. Do not edit them by hand: change `config.libsonnet`, then run `make generate && make validate`.

## Requirements

- **Prometheus Operator**, for example through `kube-prometheus-stack`. The `PrometheusRule` CRD must exist:
  ```bash
  kubectl get crd prometheusrules.monitoring.coreos.com
  ```
- **kube-state-metrics**, scraped with `job="kube-state-metrics"`.
- **cAdvisor through the kubelet**, scraped with `job="kubelet", metrics_path="/metrics/cadvisor"`. Its series must carry `container` and `image` labels. Nodes that use containerd or CRI-O do. Nodes that use Docker through cri-dockerd do not.
- **Grafana 12 or later**, with the dashboard sidecar enabled. kube-prometheus-stack enables it by default.
- Optional:
  - `container_cpu_cfs_throttled_seconds_total`. kube-prometheus-stack drops it by default; see `../kube-prometheus-stack-values.yaml` to keep it.
  - PSI metrics. Without them the pressure panels show no data.

Check that the source metrics exist:

```promql
count by (job) (kube_pod_info)
count by (job, metrics_path) (container_cpu_usage_seconds_total{container!="", image!=""})
```

## Match your installation

The defaults assume a Helm release named `kube-prometheus-stack` in the `monitoring` namespace. Check what your cluster selects:

```bash
# Labels that Prometheus requires on PrometheusRule objects
kubectl get prometheus -A -o jsonpath='{range .items[*]}{.spec.ruleSelector}{"  "}{.spec.ruleNamespaceSelector}{"\n"}{end}'

# Label that the Grafana sidecar watches for dashboards
kubectl get deploy -A -l app.kubernetes.io/name=grafana -o json \
  | jq -r '.items[].spec.template.spec.containers[] | select(.name | test("sc-dashboard")) | .env[] | select(.name | test("LABEL|NAMESPACE")) | "\(.name)=\(.value)"'
```

If the selectors differ, update these settings in `../config.libsonnet` and regenerate:

| Setting | Default |
|---|---|
| `prometheusRuleLabels` | `{ release: 'kube-prometheus-stack' }` |
| `dashboardConfigMapLabels` | `{ grafana_dashboard: '1' }` |
| `namespace` | `monitoring` |
| `kubeStateMetricsSelector` | `job="kube-state-metrics"` |
| `cadvisorSelector` | `job="kubelet", metrics_path="/metrics/cadvisor"` |

```bash
make generate && make validate
make cluster-check   # server-side dry run against the current kubectl context
```

## Install

Install the rules first, then the dashboard:

```bash
kubectl apply -f manifests/prometheus-rule.yaml
kubectl apply -f manifests/dashboard-configmap.yaml
```

## Verify

```promql
count(resource_review:workload_revision_pod_container_info)   # > 0 within about 1 minute
count(resource_review:container_cpu_usage_cores:rate5m)       # > 0 within about 5–10 minutes
```

Then open Grafana: **Dashboards → Kubernetes → Kubernetes Resource Review**.

The rules record data only from the moment they are loaded. Findings about low usage need at least 7 days of history.

## Other setups

- **Grafana runs outside the cluster:** do not apply `dashboard-configmap.yaml`. Import `../generated/resource-review.json` instead, through **Dashboards → New → Import** or the Grafana API.
- **Prometheus runs without the Operator:** do not apply `prometheus-rule.yaml`. Load `../generated/prometheus-rules.yaml` through `rule_files` in `prometheus.yml`, then reload Prometheus.

## Uninstall

```bash
kubectl delete -f manifests/dashboard-configmap.yaml
kubectl delete -f manifests/prometheus-rule.yaml
```

Recorded `resource_review:*` series expire with Prometheus retention.
