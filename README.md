# k8s-reliability-lab

Portfolio Kubernetes reliability lab on a local kind cluster: monitoring, deliberate faults, and short postmortems grounded in real public incidents.

本地 kind 集群上的 Kubernetes 可靠性实验：监控、主动注入故障，以及基于公开事故的短复盘。

## Stack

| Piece | What |
|-------|------|
| Cluster | kind `lab` — 1 control-plane + 2 workers (Kubernetes v1.37) |
| Monitoring | kube-prometheus-stack (Helm release `kps`, ns `monitoring`) |
| Workload | [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo) (ns `boutique`) |

## Layout

```
cluster/                 kind config
monitoring/alerts/       PrometheusRule manifests
drills/                  how to reproduce each fault
postmortems/             write-ups + Grafana evidence
```

## Drill 01 — CPU throttling (Buffer)

See [`drills/01-cpu-throttling/`](drills/01-cpu-throttling/) and [`postmortems/01-cpu-throttling.md`](postmortems/01-cpu-throttling.md).

Source: [Making Kubernetes services faster by removing CPU limits](https://erickhun.com/posts/kubernetes-faster-services-no-cpu-limits/).

## Quick recreate

```bash
# on WSL Ubuntu, from a clone of this repo
kind create cluster --name lab --config cluster/kind-config.yaml
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kps prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
kubectl create ns boutique
kubectl apply -n boutique -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/release/kubernetes-manifests.yaml
kubectl apply -f monitoring/alerts/cpu-throttling.yaml
```

Port-forwards (bind to localhost only):

```bash
kubectl -n monitoring port-forward svc/kps-grafana 3000:80 --address 127.0.0.1
kubectl -n boutique port-forward svc/frontend 8080:80 --address 127.0.0.1
```

## Status

- [x] Drill 01 CPU throttling — run + postmortem (2026-11-04)
- [ ] Drill 01 — alert verification rerun
- [ ] Drill 02 — Zalando CoreDNS (planned)
