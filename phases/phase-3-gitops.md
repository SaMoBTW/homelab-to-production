# Phase 3: GitOps with Argo CD and Jenkins

**Status:** ✅ Done
**Goal:** a push to the portfolio repo ends with new code running in the cluster, and nobody types `kubectl`.

## The shape of it

```
git push (Portfolio)
  -> Jenkins builds the image, tags it with the commit SHA
  -> pushes to ghcr.io
  -> commits the new tag into homelab-config
  -> Argo CD notices Git changed and applies it
```

Five steps, and the interesting part is which component does which.

## Two repos, and why it matters

`Portfolio` holds application source. `homelab-config` holds desired cluster state, nothing else. Argo CD only ever reads the second one.

That split is what makes the last step possible. Argo CD watches one repo and does not care how changes arrive there, so a human editing YAML and a CI job bumping an image tag are indistinguishable to it. Both are just commits.

## The boundary that actually matters

Jenkins has no cluster credentials. None. It holds a registry token and a Git deploy key, and that is the entire set.

So CI cannot deploy. It can only *propose* a change by writing to Git, and Argo CD is the only thing that touches the cluster. A compromised CI system can push bad YAML, which is visible in a diff, reviewable, and revertable. It cannot reach into the cluster and do something invisible.

This is the security property GitOps is actually selling, and it is easy to accidentally give away by handing your CI system a kubeconfig because it was faster.

## App-of-apps

The root Argo CD Application points at a directory of other Applications rather than at manifests. Adding an app becomes: write a file in `apps/`, commit, push. Argo CD creates the Application, which then syncs its own manifests.

The root app has to be applied by hand exactly once, to bootstrap the loop. That is the only manual `kubectl` in the whole system. Everything after it arrives through Git, including changes to the root app itself, since `apps/` contains the root's own definition and it therefore manages itself.

## Two deploy keys, scoped differently

Argo CD gets a read-only deploy key on `homelab-config`. It only ever reads.

Jenkins gets a separate key with write access, because bumping the image tag is a push.

Two keys rather than one, because a deploy key is scoped to a single repository and a single permission level. If Argo CD is compromised it cannot write to the repo it reads from, which closes an obvious loop where a compromised deployer rewrites its own desired state.

## Tagging with the commit SHA, not `latest`

Every image is tagged with the git commit that produced it.

Using `latest` would break the whole mechanism, not just be untidy. The Deployment spec would be byte-identical on every build, Git would show no diff, and Argo CD would have nothing to sync. Nothing would ever deploy.

The SHA also answers "what exactly is running right now" by pointing at a specific commit, which most setups cannot do.

## Building images inside the cluster

There is no Docker daemon in a Kubernetes pod, so the build needs a tool that can produce an image without one.

The first attempt used Kaniko, which failed in a way worth recording. The application build itself succeeded every time, `npm ci` and `vite build` both completing normally, and then Kaniko died immediately afterwards with `error building stage: failed to execute command: permission denied` and nothing else, even at debug verbosity. Running a probe pod confirmed the container had root and a writable filesystem, so the obvious explanation was wrong.

Rather than keep guessing, the better answer was to stop using Kaniko. Google archived it, and rough edges like this no longer get fixed. Swapping to rootless BuildKit worked on the first attempt with no other changes. Same Dockerfile, same pod, same registry credentials.

The lesson is less about either tool than about recognising when the thing you are debugging is unmaintained, and that continuing to debug it is the expensive option.

## Smaller things that cost time

**Architecture mismatch.** The first image was built on an Apple Silicon Mac and the cluster is all amd64, which surfaces as `no match for platform in manifest` rather than anything mentioning architecture. Building on the cluster makes this disappear permanently, which is an underrated argument for CI over local builds.

**Host key verification.** SSH inside a build container has no `known_hosts` and no interactive prompt to accept one, so a clone just fails. Writing `known_hosts` did not survive the container's non-writable home directory. Setting `GIT_SSH_COMMAND` with `StrictHostKeyChecking=no` sidesteps it, which is acceptable when the deploy key is what actually authenticates.

**GitHub password authentication is gone.** A cloned-over-HTTPS repo prompts for a password and then rejects the account password, because only a personal access token works there now. Over SSH the question never comes up.

## Proving self-heal

Scaling the deployment to 3 replicas by hand and watching it return to 1 without intervention is the real test, since Git says 1 and `selfHeal: true` means Git wins.

It is a small demo that makes a large point: the cluster is no longer something you change. It is something that converges on what the repo says, and drift is corrected rather than accumulated.

## What this phase proved

A commit to application source produced a new container image, a new commit in a separate config repo, and new pods running that exact code, with no human touching the cluster at any point. The running pods can be traced back to the commit that built them, and a manual change to the cluster is silently undone.
