# Roadmap

The plan this journey follows. Each phase names the tool and the target shape — not the exact command or manifest. The point is building the judgment to write those from the tool's own documentation, not adapting a copy-pasted example blindly.

## Starting point

- `samir-nas` — 8 vCPU, 24GB RAM, control-plane, Debian
- `g14-server` — 16 vCPU, ~14GB RAM, Ubuntu
- `mini-pc-ubuntu` — 12 vCPU, ~11GB RAM, Ubuntu

k3s, freshly stood up. Default install only: CoreDNS, Traefik, metrics-server, `local-path` as the only StorageClass.

## Phase 0 — One manual deployment, by hand

Deploy pgAdmin with a plain Deployment, Service, and PVC written from scratch — no Helm, no generator, no copy-pasted manifest. Delete it, rebuild it from memory. Every later phase automates this process; if "this" isn't understood by hand first, GitOps just becomes YAML running itself, unsupervised.

## Phase 1 — Cluster structure

Namespace per application, not per environment. Resource requests/limits mandatory on every workload from day one. Node affinity based on what each node is actually good at (`g14-server` = CPU, `samir-nas` = RAM) — reserved for workloads where placement genuinely matters, not applied blanket.

## Phase 2 — Storage: Longhorn

Replaces `local-path` as the default StorageClass. Chosen over Rook-Ceph (too much operational overhead for 3 nodes) and NFS (reintroduces a single point of failure). Replica count 2, not 3 — three nodes means a 3-replica volume has zero tolerance for taking one down for maintenance.

## Phase 3 — GitOps: Argo CD

App-of-apps pattern, two repos: `homelab-apps` (application source) and `homelab-config` (Kubernetes manifests only — the only repo Argo CD ever reads). CI can propose a change to Git; only Argo CD applies it to the cluster. Auto-sync and self-heal from day one.

## Phase 4 — Secrets: Sealed Secrets

Correctly-sized for one person's homelab — a single controller, ciphertext safe to commit. Vault is a deliberate future stretch project, not bundled into the first GitOps rollout. The controller's private key gets backed up off-cluster the moment it's generated — the one step in this whole roadmap where skipping it means real, permanent data loss.

## Phase 5 — Exposure: Cloudflare Tunnel

Extends the existing tunnel already used for Plex/Seerr rather than standing up a second one. Plain Kubernetes `Ingress`, not Traefik's `IngressRoute` CRD — portable, and what shows up in real job postings. `cert-manager` skipped for now since Cloudflare already terminates TLS.

## Phase 6 — Observability: kube-prometheus-stack

Deployed via Argo CD — the first proof the GitOps pipeline handles real infrastructure, not just toy apps. At least three real Alertmanager alerts, each deliberately triggered and confirmed before this phase counts as done.

## Phase 7 — Backup and DR: Velero

Backup target has to be physically separate from the NAS itself. The NAS has no RAID redundancy — stacking real data on a cluster with no backup plan, on storage with no redundancy, is the single biggest gap versus an actual production environment. A nightly backup that's never been restored doesn't count.

## Phase 8 — Practicing failure on purpose

Graceful node drain, then PodDisruptionBudgets, then a hard, unannounced node kill — timed and compared against the graceful case. The gap between those two failure modes is worth having actually felt, on hardware that can actually die.
