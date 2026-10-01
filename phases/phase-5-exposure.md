# Phase 5: exposure with Cloudflare Tunnel

**Status:** ✅ Done
**Goal:** reach an app in the cluster from the public internet, over HTTPS, with nothing listening on a public IP and no port forwarded.

## Why a tunnel instead of a port forward

The usual way to put a self-hosted app online is to forward a port on the router and point DNS at your home IP. That publishes your address, needs a static IP or dynamic DNS, requires certificate management, and puts an open port on a residential connection for anyone to scan.

A tunnel inverts the direction. `cloudflared` runs inside the network, dials out to Cloudflare, and holds those connections open. Requests arrive on a connection the cluster itself opened. There is nothing inbound to firewall, because there is nothing inbound.

I had already used this pattern once, when my school's Wi-Fi turned out to be blocking the UDP traffic Tailscale relies on. That was a single Docker container. This phase is the same idea done properly: in the cluster, under GitOps, with the config in Git.

## Going against the roadmap

My own roadmap said to extend the existing tunnel, the one already serving Plex and a few other things from the Docker Compose stack on the same box. Adding one ingress rule to it would have taken about two minutes.

I didn't, because that Compose stack is being retired. Anything built on it would need rebuilding within the week. So the new tunnel runs as a Deployment with two replicas under Argo CD, which also gives it redundancy the single container never had.

The argument for the other choice is still worth writing down: a tunnel that lives outside the cluster keeps working when the cluster doesn't, which matters if the tunnel is how you reach things while debugging. In my case Tailscale already covers that, so the tunnel doesn't need to.

## Locally managed, not dashboard managed

There are two ways to configure a tunnel, and the difference matters more than it looks.

Create one in the Cloudflare dashboard and it is remotely managed: you run `cloudflared` with nothing but a token, and the hostname routing lives in a web UI. Create one from the CLI and it is locally managed, taking a config file you supply.

I had just spent two phases moving cluster state into Git and encrypting secrets so they could live there too. Putting the public routing table into a web form would have quietly undone that. The config is five lines of YAML in a repo instead:

```yaml
tunnel: <tunnel-id>
credentials-file: /etc/cloudflared/creds/credentials.json
metrics: 0.0.0.0:2000
no-autoupdate: true
protocol: http2
ingress:
  - hostname: portfolio.samirmahmoud.net
    service: http://traefik.kube-system.svc.cluster.local:80
  - service: http_status:404
```

The tunnel credential is a `SealedSecret`, mounted as a file. The config is a ConfigMap. Both in the repo, nothing sensitive in plaintext.

## The path

```
browser
  -> Cloudflare edge      TLS terminates here
  -> cloudflared pod      outbound only, no listener
  -> traefik ClusterIP    host-based routing
  -> portfolio-service    ClusterIP
  -> portfolio pod
```

TLS starts at the browser and ends at Cloudflare's edge. That is why this phase skips cert-manager entirely: nothing inside the cluster ever handles a certificate, and the hop from `cloudflared` to Traefik is plain HTTP inside the pod network.

Note what the tunnel points at. A tunnel rule can target any address the pod can reach, including `portfolio-service` directly. Pointing it at Traefik instead keeps all host routing in Ingress objects that are already under GitOps, so the tunnel config stays short no matter how many hostnames get added later. Going direct would mean hardcoding a cluster DNS name into the tunnel for every service, with the Ingress layer doing nothing.

## The 404 that was supposed to happen

Traefik routes on the `Host` header. My existing Ingresses only matched the `nip.io` hostnames I use over Tailscale, so forwarding a real hostname to Traefik would hit no rule at all and return 404.

The fix is a second host entry on the same Ingress, same backend, so the public hostname and the Tailscale one both work. Easy to demonstrate once it's in place:

```bash
# same IP, same port, same request shape
curl -o /dev/null -w '%{http_code}\n' \
  -H 'Host: portfolio.samirmahmoud.net' http://100.119.68.88/   # 200
curl -o /dev/null -w '%{http_code}\n' \
  -H 'Host: nope.samirmahmoud.net' http://100.119.68.88/        # 404
```

One header, two outcomes.

## QUIC doesn't survive Tailscale

The tunnel came up and would not register. `cloudflared` retried forever with `timeout: no recent network activity`, and its own startup precheck had already told me why:

```
UDP Connectivity  region1.v2.argotunnel.com  FAIL  QUIC connection failed
TCP Connectivity  region1.v2.argotunnel.com  PASS  HTTP/2 connection successful
```

My first guess was blocked UDP, which was wrong, because DNS resolution passed on the same run and DNS is UDP.

The real tell is `no recent network activity` with nothing returned, not even an ICMP error. Packets left and vanished. That is an MTU black hole. My nodes talk to each other over Tailscale, which runs a 1280 byte MTU, and QUIC's handshake packets are roughly 1200 bytes before encapsulation. DNS queries are small enough to pass, which is exactly what made it confusing.

`protocol: http2` moves the tunnel onto TCP, which negotiates its segment size properly, and it connected immediately. The Docker container never hit this because it uses the host network.

## A config change that wasn't a deploy

Editing a ConfigMap restarts nothing. The file on disk updates, the process never rereads it, and the pod template hasn't changed so Kubernetes has no reason to roll anything. You get a config change that silently isn't applied.

Kustomize's `configMapGenerator` fixes this by appending a content hash to the ConfigMap name. A new name means a new pod template, which means a real rollout. Config changes become deploys.

It failed silently on the first attempt, in a way worth knowing about. A generated ConfigMap has no namespace unless the kustomization sets one, while my Deployment declared its namespace explicitly. Kustomize matches references on full resource identity, namespace included, so it decided the Deployment wasn't referring to that ConfigMap and left the reference alone. No warning. Argo CD then reported Synced and Healthy while the Deployment pointed at a ConfigMap that no longer existed.

```yaml
# kustomization.yaml
namespace: cloudflared        # without this, no rewrite happens
configMapGenerator:
  - name: cloudflared-config
    files:
      - config.yaml
```

The check is to render before pushing and require the hash twice, once on the ConfigMap and once on the Deployment's volume reference:

```bash
kubectl kustomize manifests/cloudflared | grep 'name: cloudflared-config'
```

## Three smaller traps

An Argo CD Application's `metadata.namespace` has to be `argocd`, because that is the only namespace the controller watches. `spec.destination.namespace` is the field that chooses where resources land. I set the first one to the destination, which produced a perfectly valid object that was never reconciled, and a parent app reporting Synced and Healthy while the child did nothing.

`kubectl apply --dry-run=server` can't validate resources in a namespace the same bundle creates. The dry-run namespace is never real, so everything namespaced fails with `NotFound`. Apply the Namespace for real first, then dry-run the rest.

`readOnly` belongs on `volumeMounts`, not on a `secret` volume source. Default field validation is warn rather than reject, so it applies cleanly and is quietly ignored.

A theme across all three: "Synced" and "applied successfully" are weaker statements than they look.

## What stays private

Only the portfolio is public. Argo CD, Jenkins and pgAdmin stay reachable over Tailscale only.

Jenkins is the clear line for me. It's a remote code execution surface by design, so an admin tool on the open internet is one credential away from being someone else's build server. If I expose either of them later it goes behind Cloudflare Access with SSO first, never on its own login page.

## What changed

An app in the cluster is now reachable at a real hostname over HTTPS, in about 130ms, with no port forwarded, no inbound firewall rule, and no public IP anywhere in the path. The routing that makes that work is four lines of YAML in a repo, and adding the next hostname is one Ingress entry and one tunnel rule.

One loose end, on purpose: the old Docker tunnel still serves three hostnames that have to move into this one before the Compose stack comes down.
