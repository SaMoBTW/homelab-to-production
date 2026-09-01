# Homelab to Production

A deliberate, self-directed path from a bare 3-node [k3s](https://k3s.io/) cluster to a production-shaped homelab — GitOps, observability, backup/disaster recovery, and the real debugging stories along the way.

This isn't a tutorial-clone. Each phase follows a [roadmap](./ROADMAP.md) that names the tool and the target shape, but not the exact manifest — the point is building real judgment about *why* a given tool or pattern fits, not copy-pasting one.

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![k3s](https://img.shields.io/badge/k3s-FFC61C?style=flat&logo=k3s&logoColor=black)
![ArgoCD](https://img.shields.io/badge/Argo%20CD-EF7B4D?style=flat&logo=argo&logoColor=white)
![Longhorn](https://img.shields.io/badge/Longhorn-1A56DB?style=flat)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat&logo=prometheus&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare%20Tunnel-F38020?style=flat&logo=cloudflare&logoColor=white)

## The cluster

Three bare nodes, no cloud, no managed control plane:

```mermaid
graph TB
    subgraph cluster["k3s cluster"]
        A["samir-nas<br/>8 vCPU · 24GB RAM<br/>control-plane"]
        B["g14-server<br/>16 vCPU · ~14GB RAM<br/>worker"]
        C["mini-pc-ubuntu<br/>12 vCPU · ~11GB RAM<br/>worker"]
    end
```

`samir-nas` also runs an unrelated, pre-existing Plex/Sonarr/Radarr Docker Compose stack on the same physical box — outside this cluster, but a real constraint the exposure phase has to account for.

## Progress

| Phase | What | Status |
|---|---|---|
| 0 | [Manual deployment, by hand](./phases/phase-0-manual-deploy.md) — pgAdmin as Deployment + PVC + Service, no Helm, no generator | ✅ Done |
| 1 | Cluster structure — namespaces, mandatory resource limits, node affinity | 🚧 In progress |
| 2 | Storage — Longhorn replacing `local-path` | ⏳ Planned |
| 3 | GitOps — Argo CD, app-of-apps, two repos | ⏳ Planned |
| 4 | Secrets — Sealed Secrets | ⏳ Planned |
| 5 | Exposure — Cloudflare Tunnel + Ingress | ⏳ Planned |
| 6 | Observability — kube-prometheus-stack | ⏳ Planned |
| 7 | Backup/DR — Velero, off-box target | ⏳ Planned |
| 8 | Practicing failure on purpose | ⏳ Planned |

## Incidents

Real problems hit and fixed along the way — not staged, not skipped over:

- [pgAdmin OOMKilled during Phase 0](./incidents/oom-killed-pgadmin.md)

## Why document this

Most homelab repos show the final YAML. This one tries to show the reasoning that got there — the tradeoffs considered, the things that broke, and why the fix was the fix. That's the part that doesn't show up in a "deployed X to Kubernetes" resume line.
