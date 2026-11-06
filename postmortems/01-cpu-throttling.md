# 演练 01：CPU limit 过低导致的 CFS 节流 / Drill 01: CFS throttling from a too-low CPU limit

参考来源 / Source: Buffer, ["Making Kubernetes services faster by removing CPU limits"](https://erickhun.com/posts/kubernetes-faster-services-no-cpu-limits/)

环境 / Environment: kind 集群 `lab`（1 个控制面 + 2 个 worker，Kubernetes v1.37），kube-prometheus-stack（Helm release `kps`，namespace `monitoring`），Online Boutique（namespace `boutique`）

Environment: kind cluster `lab` (1 control plane + 2 workers, Kubernetes v1.37), kube-prometheus-stack (Helm release `kps`, namespace `monitoring`), Online Boutique (namespace `boutique`)

## 时间线 / Timeline (2026-11-04, Asia/Shanghai)

- 22:05–23:05 基线：`productcatalogservice` 使用默认配置（limit `200m`，request `100m`），节流比例最高约 3%，大部分时间为 0。
  Baseline: `productcatalogservice` on its default settings (limit `200m`, request `100m`); throttled share peaked at about 3% and was 0 most of the time.
- 约 23:08 注入故障：`--limits=cpu=20m --requests=cpu=10m`；节流比例跳到约 40%（峰值约 0.44）。
  Fault injected: `--limits=cpu=20m --requests=cpu=10m`; throttled share jumped to about 40% (peak about 0.44).
- 约 23:19 恢复：改回 `--limits=cpu=200m --requests=cpu=100m`；节流比例回落到 0。
  Recovered: restored `--limits=cpu=200m --requests=cpu=100m`; throttled share fell back to 0.

## 影响 / Impact

loadgenerator 持续加压期间，商品目录服务大约 40% 的 CFS 周期被节流，而商城每个页面都要调用这个服务。本次没有测量页面延迟。

While the loadgenerator kept up the load, about 40% of the product catalog service's CFS periods were throttled, and every shop page calls this service. Page latency was not measured in this drill.

## 根因 / Root cause

kubelet 会把 CPU limit 转换成 CFS 配额：每 100ms 周期内，`20m` 只够用 2ms 的 CPU。一旦某个周期的配额用完，进程就要等到下一个周期才能继续运行。所以即使平均 CPU 用量不高，突发的请求也会被卡住。

The kubelet turns the CPU limit into a CFS quota: a `20m` limit allows just 2ms of CPU in each 100ms period. Once a period's quota runs out, the process has to wait for the next period to keep running. So even when average CPU use is low, bursts of requests get stuck.

## 检测 / Detection

PromQL:

```promql
sum(rate(container_cpu_cfs_throttled_periods_total{namespace="boutique",pod=~"productcatalog.*"}[1m]))
/
sum(rate(container_cpu_cfs_periods_total{namespace="boutique",pod=~"productcatalog.*"}[1m]))
```

证据截图 / Evidence screenshots:

- [grafana-throttle-spike.png](01-cpu-throttling/evidence/grafana-throttle-spike.png) — 故障期间峰值约 40%+
- [grafana-throttle-recovery.png](01-cpu-throttling/evidence/grafana-throttle-recovery.png) — 约 23:19 后回落到 0

## 改进措施 / Action items

1. 为节流比例加一条告警（例如连续 5 分钟大于 25%）。见 [`monitoring/alerts/cpu-throttling.yaml`](../monitoring/alerts/cpu-throttling.yaml)。
   Add an alert on the throttled share (for example, above 25% for 5 minutes). See [`monitoring/alerts/cpu-throttling.yaml`](../monitoring/alerts/cpu-throttling.yaml).
2. 根据实际用量的 P95/P99 来设置 request；对延迟敏感的服务，考虑放宽或去掉 CPU limit（Buffer 的做法）。
   Set requests from the P95/P99 of actual usage; for latency-sensitive services, consider loosening or removing the CPU limit (Buffer's approach).
3. 下次演练加上延迟指标，量化对用户的实际影响。
   Add a latency metric in the next drill to quantify the real impact on users.

## 告警验证复跑 / Alert verification rerun (2026-11-05 → 2026-11-06, Asia/Shanghai)

已应用 [`monitoring/alerts/cpu-throttling.yaml`](../monitoring/alerts/cpu-throttling.yaml)（PrometheusRule `boutique-cpu-throttling`，`HighCPUThrottling`，连续 5 分钟 >25%）。

Applied [`monitoring/alerts/cpu-throttling.yaml`](../monitoring/alerts/cpu-throttling.yaml) (PrometheusRule `boutique-cpu-throttling`, `HighCPUThrottling`, above 25% for 5 minutes).

- 晚间再次注入 `--limits=cpu=20m --requests=cpu=10m`；Explore 节流比例峰值约 0.34～0.42，稳定在约 0.22～0.27。
  Re-injected `--limits=cpu=20m --requests=cpu=10m`; Explore throttled share peaked about 0.34–0.42 and held around 0.22–0.27.
- `HighCPUThrottling` 进入 **Firing**（Grafana Alert rules，`state:firing`）。
  `HighCPUThrottling` entered **Firing** (Grafana Alert rules, `state:firing`).
- 约 00:18 恢复 `--limits=cpu=200m --requests=cpu=100m`；约 00:19 比值回落到 0；随后告警回到 **Normal**。
  Restored `--limits=cpu=200m --requests=cpu=100m` around 00:18; ratio fell to 0 around 00:19; alert returned to **Normal** afterward.

闭环：注入 → 告警 Firing → 恢复 → Normal。
Closed loop: inject → alert Firing → restore → Normal.
