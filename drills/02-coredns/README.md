# Drill 02 — CoreDNS memory pressure (Zalando-inspired)

Reference: [Zalando Jan 2019 DNS outage](https://kubernetes-on-aws.readthedocs.io/en/stable/postmortems/jan-2019-dns-outage.html) (ndots:5 amplifier + small CoreDNS memory limit → OOM → cluster DNS outage).

Goal: reproduce a **controlled** CoreDNS stress path on kind `lab`, show metrics + business impact (Boutique lookups / errors), then recover and write a postmortem. This is intentionally heavier than Drill 01 (CPU limit): you touch cluster DNS, kube-system, and multi-signal evidence.

## What “done” means (workload proof)

1. Document CoreDNS Deployment resources **before** the drill (`kubectl -n kube-system get deploy coredns -o yaml` excerpt).
2. Inject: lower CoreDNS memory limit (start conservative on kind, e.g. toward historical small limits) **or** generate amplified DNS load (ndots / external lookups) until QPS or memory rises and pods restart / OOM.
3. Capture **at least three signals**: CoreDNS memory or restarts, DNS QPS / latency if available, Boutique or `nslookup`/`dig` failure rate from a debug pod.
4. Recover: restore memory / stop load; confirm DNS and Boutique recover.
5. Postmortem under `postmortems/02-coredns.md` + evidence PNGs; optional PrometheusRule for CoreDNS restarts or memory.

## Safety

- Bind any port-forward to `127.0.0.1` only.
- Do not leave CoreDNS broken: always have the restore commands ready in the same terminal session.
- On a tiny kind cluster, prefer **load amplify + modest memory cut** over setting memory so low that the node becomes unusable.

## Skeleton commands (fill numbers after baseline)

```bash
# baseline
kubectl -n kube-system get deploy coredns -o wide
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system top pods -l k8s-app=kube-dns  # if metrics-server present

# prepare restore snippet first, then inject (example shape — tune after baseline)
# kubectl -n kube-system patch deploy coredns --type='json' -p='[...]'

# debug DNS from a pod
kubectl -n boutique run dnscheck --rm -it --image=busybox:1.36 --restart=Never -- nslookup kubernetes.kube-system.svc.cluster.local

# Grafana: Explore CoreDNS / kube-dns metrics (exact series depend on kube-prometheus-stack version)
```

## Out of scope for v1

Node-local DNS, real 120 kW power budgets, multi-cluster. Those are later backlog.
