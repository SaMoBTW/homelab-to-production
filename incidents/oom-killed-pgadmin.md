# Incident: pgAdmin OOMKilled on first deploy

**Date:** 2026-08-30
**Phase:** [0 — Manual deployment](../phases/phase-0-manual-deploy.md)
**Severity:** Low (single-node, no external users). Documented for the debugging process, not the stakes.

## Summary

The first applied version of pgAdmin's `Deployment` set `resources.limits.memory: 128Mi`. The pod was killed by the kernel's OOM killer within roughly 30 to 60 seconds of every start, and kept crash-looping.

## Impact

pgAdmin never reached a usable state on the first deploy. Storage (`PersistentVolumeClaim`) and networking (`Service`) were both working correctly the entire time, so this was purely a compute resource problem, isolated to one container.

## Timeline

- Deployment applied with `requests.memory: 64Mi` / `limits.memory: 128Mi`.
- Pod scheduled successfully to `g14-server`, image pulled, container started.
- Container repeatedly terminated within under a minute of starting.
- `kubectl get pods` showed `STATUS: OOMKilled`, `RESTARTS` climbing on every check.

## Root cause

```
State:          Running
  Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
Limits:
  memory:  128Mi
Requests:
  memory:  64Mi
```

Exit code `137` (128 + 9) means the process was killed by `SIGKILL` after the container's cgroup hit its memory limit. `dpage/pgadmin4`'s first boot runs database migrations and starts a Python/Flask + gunicorn stack, which needs meaningfully more headroom than 128Mi to get through startup. 64Mi/128Mi was simply too small a budget for this specific image, not a configuration mistake elsewhere in the manifest.

## Resolution

Raised the container's memory budget:

```diff
  resources:
    requests:
      cpu: "100m"
-     memory: "64Mi"
+     memory: "128Mi"
    limits:
      cpu: "250m"
-     memory: "128Mi"
+     memory: "256Mi"
```

Reapplied. The new pod came up and stayed up: `0` restarts, confirmed stable for 17+ hours afterward.

## Lessons learned

- **A resource limit is a hypothesis, not a guess you're expected to get right blind.** There's no universal correct number. The honest way to find it is to under-provision slightly, watch it fail with a real signal (`OOMKilled` / exit `137`), and adjust from evidence.
- **`kubectl describe pod`, specifically the `Last State` and `Exit Code` fields, is the first place to look**, not the `Events` list, when a pod is crash-looping. The exit code alone (`137` for OOM, `1` for an application error, `143` for a graceful `SIGTERM`) narrows the search immediately.
- **Storage and networking working correctly didn't matter yet.** A broken compute budget on the one container that has to actually run made the rest of the stack irrelevant until it was fixed. Worth checking the failing layer first rather than re-verifying the layers already confirmed working.
