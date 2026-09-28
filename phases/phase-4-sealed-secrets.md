# Phase 4: secrets with Sealed Secrets

**Status:** ✅ Done
**Goal:** no plaintext credential in any repo, while keeping cluster state fully in Git.

## The problem

GitOps says the repo is the source of truth for the whole cluster. Secrets break that, because the obvious approach puts passwords in a file you push to GitHub.

The `homelab-config` repo had been private for exactly this reason: pgAdmin's password sat in plaintext in its Deployment. Private is a workaround, not a solution, and it means the repo can never be shown to anyone.

## How Sealed Secrets works

A controller runs in the cluster holding an RSA keypair. The `kubeseal` CLI encrypts a Secret against the public half, producing a `SealedSecret` that is safe to commit anywhere. Only that controller, holding the private half, can decrypt it, and it does so into an ordinary Kubernetes `Secret` at apply time.

Worth being precise: it does not store secrets. It makes secrets committable. The decrypted result is a normal `Secret`, which is base64 rather than encrypted at rest unless you configure that separately.

Sealed Secrets are also scoped to namespace plus name by default. The same file will not decrypt under a different name or in a different namespace, which stops someone copying a sealed secret into a namespace they control and letting the controller decrypt it for them.

## The step that actually matters

Back up the controller's private key, off the cluster, the moment it is generated.

```bash
kubectl get secret -n sealed-secrets \
  -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > master-key.yaml
```

Lose that key and every SealedSecret ever committed becomes permanently unreadable. The encrypted YAML in Git is worthless without it, and there is no recovery path. This is the one step in the whole roadmap where skipping it is not a learning experience, it is data loss.

It went to a password manager and cloud storage, deliberately not to the NAS, which is the same machine the cluster runs on and has no disk redundancy.

## Rotating rather than rewriting history

The done-when bar was "no plaintext credential in either repo, ever, checkable by grepping your own git history." Sealing a secret going forward does nothing about commits that already contain it.

Two options: rewrite Git history, or rotate the credential so the exposed one is worthless.

Rotating is better. History rewriting is disruptive, easy to get wrong, and does not help if anyone already cloned the repo. A rotated password makes every historical copy dead on arrival, which is the actual goal. It also costs nothing here, since the password was literally named `passiwillchangelater`.

The plaintext only ever existed inside one shell pipe:

```bash
kubectl create secret generic pgadmin-credentials -n pgadmin \
  --from-literal=email='...' --from-literal=password='...' \
  --dry-run=client -o yaml | \
kubeseal --controller-namespace sealed-secrets --controller-name sealed-secrets \
  --format yaml > manifests/pgadmin/sealed-secret.yaml
```

`--dry-run=client` builds the Secret as YAML without sending it anywhere. It goes straight into `kubeseal` and comes out encrypted. Nothing plaintext touches disk or Git.

The Deployment then references the Secret by name rather than holding values:

```yaml
env:
  - name: PGADMIN_DEFAULT_PASSWORD
    valueFrom:
      secretKeyRef:
        name: pgadmin-credentials
        key: password
```

## The multi-attach deadlock

Rolling out that change stalled completely, with the new pod stuck in `ContainerCreating` for twenty minutes and the old one still running.

The event said `Multi-Attach error`. The cause is the default `RollingUpdate` strategy: it starts the new pod before terminating the old one, but the PVC is `ReadWriteOnce` and can only attach to one node at a time. The new pod waits for the volume, the old pod waits for the new one to be ready, and neither moves.

```yaml
spec:
  strategy:
    type: Recreate
```

`Recreate` terminates the old pod fully before starting the new one. A few seconds of downtime instead of a permanent stall, which is the right trade for any single-replica app on an RWO volume. Worth knowing before it happens rather than during.

## Auditing the rest of the stack

Worth checking what else needed sealing rather than assuming pgAdmin was the only offender. Nothing else did, for an interesting reason: every other app keeps its secrets outside Git already.

zurg's debrid token lives in a config file on a hostPath. Jellyfin's config is a hostPath. Sonarr, Radarr, Prowlarr and Seerr keep their API keys in their own databases on PVCs. None of it was ever in the repo.

That is not a clean bill of health though. Those secrets are outside Git, so they are also outside version control and outside any backup story. If the NAS disk dies they go with it. Sealed Secrets solves "secrets in Git"; it does nothing for "secrets nowhere but one disk." That gap belongs to the backup phase.

Three credentials also exist only inside the cluster and in no repo at all: the registry token, Argo CD's deploy key, and the Jenkins admin password. Bootstrap credentials arguably should not live in the repo the deployer reads, but they are equally unrecoverable on a rebuild, so they belong in a password manager alongside the master key.

## What changed

The repo can be public now. The only credential-shaped thing in it is an encrypted blob that is useless to anyone who is not this specific controller, and the one password ever committed in plaintext has been rotated into irrelevance.
