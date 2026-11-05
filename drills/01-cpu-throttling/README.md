# Drill 01 — Buffer-style CPU throttling

## Prerequisites

- kind cluster `lab` Ready
- kube-prometheus-stack (`kps`) in `monitoring`
- Online Boutique in `boutique`
- Grafana Explore open with Prometheus

## Baseline PromQL

```promql
sum(rate(container_cpu_cfs_throttled_periods_total{namespace="boutique",pod=~"productcatalog.*"}[1m]))
/
sum(rate(container_cpu_cfs_periods_total{namespace="boutique",pod=~"productcatalog.*"}[1m]))
```

## Inject fault (WSL / Ubuntu)

```bash
kubectl -n boutique set resources deploy/productcatalogservice --limits=cpu=20m --requests=cpu=10m
```

Watch Grafana until the ratio climbs (expect ~40% under loadgenerator traffic).

## Confirm alert (after applying `monitoring/alerts/cpu-throttling.yaml`)

```bash
kubectl -n monitoring get prometheusrule boutique-cpu-throttling
# In Grafana → Alerting → Alert rules, or Prometheus UI → Alerts
# HighCPUThrottling should fire after ~5 minutes above 25%.
```

## Restore

```bash
kubectl -n boutique set resources deploy/productcatalogservice --limits=cpu=200m --requests=cpu=100m
```

Expect the ratio to fall back to ~0 and the alert to resolve.
