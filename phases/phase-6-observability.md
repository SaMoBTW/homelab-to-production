# Phase 6: observability with kube-prometheus-stack

**Status:** ✅ Done
**Goal:** metrics, dashboards and alerting for the whole cluster, deployed through Argo CD like everything else, with at least three real alerts deliberately triggered and confirmed end to end.

## Two Applications, not one

The stack needs two credentials, a Grafana admin login and a Slack incoming webhook URL, and both are sealed into the `monitoring` namespace as `SealedSecret`s the same way I did it in Phase 4.

The real decision was where those SealedSecrets should live. I could have put them in the same Argo CD Application as the chart using a second source, but I split them into two apps instead:

- `monitoring-secrets` points at a folder in my repo with the namespace and the two SealedSecrets.
- `monitoring` points at the upstream Helm chart.

I did this to make problems easier to diagnose. A bad seal doesn't fail the sync: Argo CD applies the SealedSecret without complaint, the controller fails to decrypt it, no Secret ever appears, and Grafana sits waiting for something that doesn't exist. If that happened inside a first-time install of about 50 resources, I'd be stuck figuring out whether the seal or the chart was the problem. With two apps the secrets sync first, and either the Secrets show up or they don't. It also means rotating a credential later only touches the small app.

To check that a secret decrypted without printing its value, `describe` shows the key names and byte sizes and nothing else:

```bash
kubectl -n monitoring get sealedsecrets          # SYNCED should be True
kubectl -n monitoring describe secret grafana-credentials
```

The namespace carries one extra annotation:

```yaml
metadata:
  name: monitoring
  annotations:
    argocd.argoproj.io/sync-options: Prune=false
```

The app has pruning on, so deleting a file from Git deletes the resource in the cluster. That's what I want for everything except the namespace, because deleting a namespace deletes everything in it, including the PVCs the chart creates, and my Longhorn StorageClass has `reclaimPolicy: Delete`. Reverting the commit would bring the manifests back but not the metrics, since Git can roll back configuration but not data.

## An Application that points at a chart

Every Application before this one pointed at a folder of YAML I wrote myself, and this one points at someone else's chart:

```yaml
source:
  chart: kube-prometheus-stack
  repoURL: https://prometheus-community.github.io/helm-charts
  targetRevision: 91.9.0
  helm:
    valuesObject:
      ...
```

There is no `helm install` anywhere. Argo CD renders the chart itself and applies the output like any other manifests, which is why `helm list` shows nothing. The values live inline in the Application, because the source is the chart's repo and not mine, so there's no folder of mine to put a `values.yaml` in.

`valuesObject` took me a few tries to get right. Every key has to match a setting the chart actually has, at exactly the same nesting, and a key the chart doesn't recognize is ignored without any error. The nesting works like an address, so `existingSecret` at the top level and `grafana.admin.existingSecret` are completely different settings. Rendering the chart locally and grepping for the value became the quickest way to check whether a setting had landed:

```bash
helm template monitoring kube-prometheus-stack \
  --repo https://prometheus-community.github.io/helm-charts --version 91.9.0 \
  -n monitoring -f test-values.yaml | grep grafana-credentials
```

The Application name matters too. Argo uses it as the Helm release name and the chart prefixes every resource with it (cut to 26 characters), so renaming later means recreating everything. I picked `monitoring` before the first sync for that reason.

## Two traps handled before the first sync

The first trap is that the CRDs are too big for client-side apply. Client-side apply stores a full copy of each object in the `kubectl.kubernetes.io/last-applied-configuration` annotation so it can diff against it next time, but all the annotations on an object are capped at 262,144 bytes, and the rendered `prometheuses` CRD alone is 857,788 bytes. The sync fails with `metadata.annotations: Too long`. Server-side apply has the API server track which tool owns which field in `managedFields` instead of storing a copy, so the size stops mattering:

```yaml
syncPolicy:
  syncOptions:
    - ServerSideApply=true
```

The second trap is that k3s doesn't have what the chart expects. The chart's defaults assume the scheduler, controller manager and etcd each run as their own pod with their own metrics endpoint. k3s runs them inside one process, and my cluster doesn't run etcd at all, since no node has the etcd role and cluster state lives in SQLite. The chart's Services find no pods, Prometheus gets no targets, and rules like `absent(up{job="kube-scheduler"})` stay true forever, which would mean a constant stream of alerts about things that aren't broken.

Other clusters run into the same thing in different ways: default kubeadm only serves those metrics on localhost, and managed clusters hide the control plane entirely. So checking which targets actually apply to your cluster is part of installing this chart anywhere, not just on k3s.

```yaml
kubeEtcd:
  enabled: false
kubeScheduler:
  enabled: false
kubeControllerManager:
  enabled: false
kubeProxy:
  enabled: false
```

My first attempt at the controller manager used `defaultRules.rules.kubeControllerManager: false`, which only removes the alert rules and leaves the Service and ServiceMonitor scraping nothing. The top-level `enabled: false` removes all of it.

kube-proxy has the same problem, since it also runs inside k3s on every node. I left it on by accident in the first push, it showed up as 0/0 targets, and 15 minutes later `KubeProxyDown` fired a critical alert into Slack. That ended up being the first real proof that alerting worked end to end, so I waited for it to arrive before turning kube-proxy off.

## Values that point at secrets

None of the credentials appear in the values, only the names of the Secrets that hold them.

Grafana reads its admin login from a Secret by name, and the chart's default key names already matched the ones I sealed:

```yaml
grafana:
  admin:
    existingSecret: grafana-credentials
```

Alertmanager needs two parts: the operator mounts the Secret into the pod as files, and the Slack receiver reads the URL from that file.

```yaml
alertmanager:
  alertmanagerSpec:
    secrets: ["alertmanager-slack"]   # mounted at /etc/alertmanager/secrets/alertmanager-slack/
  config:
    route:
      receiver: slack-samir
    receivers:
      - name: 'null'
      - name: slack-samir
        slack_configs:
          - api_url_file: /etc/alertmanager/secrets/alertmanager-slack/webhook-url
            send_resolved: true
```

The `'null'` receiver has to stay. The chart includes an alert called Watchdog that fires all the time on purpose as a heartbeat, and routes it to a receiver with no destination. When Helm combines your values with the chart's, maps get merged but lists get replaced, so setting `receivers` to just the Slack one would have dropped `'null'` and left the Watchdog route pointing at nothing, and Alertmanager would have rejected the whole config. Running `amtool check-config` on the rendered config caught that kind of mistake before it reached the cluster.

## Storage

Prometheus keeps 15 days on a 20Gi Longhorn volume, with `retentionSize: 18GiB` so it deletes old data before the disk fills. Alertmanager gets 1Gi for silences and notification history. Grafana gets 5Gi with `deploymentStrategy: Recreate`, because it's a single-replica Deployment on a ReadWriteOnce volume and the default rolling update deadlocks on `Multi-Attach`, which is the exact problem I hit with pgAdmin in Phase 4.

## Every target up, not every pod running

Pods being Running only means the containers started. The better test is the Target health page in Prometheus, which shows whether it's actually collecting from everything it's supposed to. After the first sync it showed 13 scrape pools and 21 targets, all up. The kubelet shows up as three pools of three targets, one per node, because each kubelet serves its general, cAdvisor and probe metrics separately.

## Scraping something I own

`cloudflared` from Phase 5 already served metrics on port 2000, and I had left out a Service for it back then on purpose. Getting Prometheus to scrape it works as a chain where every link is a label match:

```
Prometheus  --finds ServiceMonitors labelled-->  release: monitoring
ServiceMonitor  --finds Services labelled-->  app: cloudflared-service
Service  --finds pods labelled-->  app: cloudflared   (port 2000)
```

`labels` are tags you put on something, and `selector` and `matchLabels` are searches for things with those tags, so each search has to match the next object's tags exactly.

I broke this chain three different ways before it worked. `targetPort: 8080` sent traffic to a port nothing listens on. Leaving out `release: monitoring` meant Prometheus never picked up the ServiceMonitor at all, which is harder to spot than a DOWN target because nothing shows up anywhere. Then, while renaming labels to make them clearer, I put `release: monitoring` inside the ServiceMonitor's `matchLabels`, where it searched for Services instead of tagging the ServiceMonitor itself. What fixed it each time was going through every selector, asking which object it was searching for, and checking that object's labels.

The Service and ServiceMonitor live in `manifests/cloudflared/` rather than the monitoring folder, so they come and go with cloudflared. The tradeoff is that the cloudflared app now depends on the ServiceMonitor CRD that the monitoring app installs.

## Three alerts, on purpose

### KubePodCrashLooping

I applied a busybox pod running `exit 1` into a throwaway `testing` namespace with `kubectl`. A throwaway test felt like a reasonable exception to GitOps, since nothing manages it and nothing tries to change it back. It fired after 15 minutes along with `KubePodNotReady`, because one real problem usually trips more than one rule.

Then I left it running for two days. `KubePodNotReady` fired and resolved 13 separate times, which added up to more than 20 Slack messages. That's called flapping, and it's how people learn to ignore alerts, so now I clean up as soon as the alert fires. Deleting the namespace cleared it.

I also had the timing wrong. I expected the resolved message to take 15 minutes because firing did, but `for: 15m` only delays firing so that a short blip doesn't page you. Resolving waits for the rule to turn false (this one looks back 5 minutes) plus Alertmanager's 5 minute grouping interval.

### TargetDown

For this one I pointed the cloudflared Service at the wrong port, so the targets still exist but fail. The roadmap said to scale something to zero, which wouldn't work, because zero replicas removes the targets entirely and `TargetDown` only counts targets that exist and fail. It also had to go through Git, because the cloudflared app has `selfHeal` on and Argo would have reverted a `kubectl edit` within seconds.

The error on the Target health page was `dial tcp 10.42.1.57:8080: connect: connection refused`. "Refused" means the pod was reachable but nothing was listening on that port, while a timeout would have pointed at the network or the node, and knowing which one you're looking at cuts down a lot of guessing. It fired after 10 minutes, and a Git revert resolved it.

### KubeNodeNotReady

This meant powering off a node, and the only one I was willing to pull the plug on was the mini PC, since the NAS runs things I don't want to fiddle with and the G14 is a laptop. The problem was that Prometheus was running on the mini PC, and a Prometheus that dies with the node can't alert on it. Its volume also couldn't reattach anywhere else until the dead node let go of it.

The mini PC held none of Longhorn's replicas (they're on the other two nodes), so moving Prometheus off it was safe. I pinned it to the G14 permanently in Git rather than cordoning the node by hand:

```yaml
prometheus:
  prometheusSpec:
    nodeSelector:
      workload-strength: cpu-heavy
```

The data moved with it, with history intact back to the first day.

When I ran the test, the first Slack message to arrive was `TargetDown` for the mini PC's kubelet and node-exporter, which fire sooner. I took that as my alert and powered the node back on a few minutes before `KubeNodeNotReady` was due. It fired anyway, because the node was still booting when the 15 minutes ran out, but that was luck. The first message to arrive is the fastest rule, not necessarily the one that describes the problem, so next time I'll read the alert name before acting.

Two things broke during the outage that I didn't expect. Argo CD stopped completely, because all of it ran on the mini PC. Most of its pods moved to other nodes after about five minutes, but the application controller is a StatefulSet pod, and Kubernetes won't start a replacement while the old one might still be running on a node it can't reach, since two controllers could end up fighting. That meant GitOps was down for the whole outage. pgAdmin couldn't move either: its replacement sat in `ContainerCreating` on the G14, waiting for a volume that was still attached to the dead node.

Both cleared up on their own once the node came back, and both are going on the list for Phase 8.

## What stays private

Grafana, Prometheus and Alertmanager each get an Ingress from chart values, on the same `nip.io` pattern as everything else, so they're only reachable over Tailscale. Turning on the Prometheus and Alertmanager Ingresses also sets their `externalUrl`, so the links inside Slack alerts open the real pages instead of internal pod addresses.

Grafana has a login, but Prometheus and Alertmanager don't, and anyone who can reach Alertmanager can silence alerts. I'm the only member of my tailnet, so I'm treating it as the boundary for now. If that changes, or if any of this ever goes public, they go behind an auth proxy first.

## What changed

A crashing pod, an unreachable endpoint and a dead node now each produce a Slack message within about 10 to 15 minutes, and another one when the problem clears. The whole stack, including a chart I didn't write, is defined in Git and synced by Argo CD, with no credentials in plaintext anywhere.

The node test also showed where the cluster is still fragile. Prometheus, kube-state-metrics and Grafana all run on the G14 now, so losing the G14 takes monitoring down with it. Argo CD sits entirely on one node, and Alertmanager isn't pinned anywhere. Those are the first things I'll be looking at in Phase 8.
