# Phase 1: cluster structure

**Status:** ✅ Done
**Goal:** namespace-per-application convention, mandatory resource requests/limits on every workload, and node labeling + `nodeAffinity` so pods land on hardware suited to them.

## Namespaces and resource limits

Both of these were already satisfied coming out of [Phase 0](./phase-0-manual-deploy.md). The `pgadmin` namespace exists to separate pgAdmin's objects from other unrelated apps that will run in the same cluster, an organizational grouping, not network isolation (namespaces don't provide that by default). pgAdmin's Deployment already carries explicit CPU/memory requests and limits, tuned after the [OOMKilled incident](../incidents/oom-killed-pgadmin.md) in Phase 0.

## Labeling the nodes

The interesting part of this phase was deciding a labeling scheme for the three nodes and getting the naming right. First attempt used `nodeAffinity` as the label key itself, which turned out to be a mistake worth learning from: `nodeAffinity` is the name of the Kubernetes mechanism that *reads* labels, not a property of a node. Naming the label after the feature consuming it is like naming a variable `variable`.

Second attempt used `workload-strength=storage-ram-heavy` for `samir-nas`. Closer, but still wrong: the actual justification for treating `samir-nas` differently is RAM capacity alone (24GB, the most of the three nodes). Storage in this cluster is handled separately by a StorageClass and, starting in Phase 2, by Longhorn, spread across all three nodes regardless of any node label. Folding "storage" into a label meant purely for CPU-vs-RAM scheduling decisions would have made it say something the reasoning didn't actually support.

Final scheme: key `workload-strength`, values describing what each node is actually good at.

```bash
kubectl label nodes g14-server workload-strength=cpu-heavy
kubectl label nodes samir-nas workload-strength=ram-heavy
kubectl label nodes mini-pc-ubuntu workload-strength=general-purpose
```

`g14-server` has the most CPU (16 vCPU) in the cluster. `samir-nas` has the most RAM (~24GB) and doubles as the control plane, confirmed already schedulable with no taints (k3s doesn't cordon the control-plane node by default, unlike `kubeadm` clusters). `mini-pc-ubuntu` doesn't have a standout specialty, so it stays general-purpose.

## Understanding `nodeAffinity` before using it

Rather than bolting an affinity rule onto pgAdmin just to have one, the actual mechanism was worth understanding first, since it's genuinely simple once you see the shape of it: label a node, then write a rule on a pod that says "prefer nodes with this label," and the scheduler checks that match once, at the moment it decides where a new pod goes.

Two details matter more than the syntax:

- **`preferred` vs `required`.** `requiredDuringSchedulingIgnoredDuringExecution` is a hard filter; if no node matches, the pod stays `Pending`, full stop. `preferredDuringSchedulingIgnoredDuringExecution` is a weighted hint the scheduler tries to honor but can override. With only three nodes, `required` risks leaving a pod permanently unschedulable over a preference that usually isn't worth that fragility, so `preferred` is the right default here.
- **`IgnoredDuringExecution`.** The rule is checked once, at scheduling time. Relabeling a node after a pod is already running there does nothing to that pod; affinity doesn't continuously re-herd running pods as labels change.

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: workload-strength
                operator: In
                values:
                  - cpu-heavy
```

## What didn't get built, on purpose

No `nodeAffinity` rule got applied to pgAdmin. Its resource footprint is small enough that placement genuinely doesn't matter, and this roadmap is explicit about not hard-pinning every workload just because the mechanism exists. That rule becomes real once a CPU-heavy or memory-heavy stateful workload actually shows up, most likely Nextcloud or a Postgres-backed app in Phase 2 or 3.
