# Phase 0: one manual deployment, by hand

**Status:** ✅ Done
**Goal:** deploy pgAdmin as a plain `Deployment` + `PersistentVolumeClaim` + `Service`, written from scratch. No Helm, no generator, no copy-pasted manifest.

## Why start here

Every later phase in this roadmap automates deployment in some way: GitOps, Helm charts, operators. None of that is worth doing until the thing being automated is actually understood by hand. Phase 0 is that baseline, three core Kubernetes objects, built and debugged manually, on a real cluster.

## What got built

A `pgadmin` namespace, one namespace per application (the convention this project follows throughout, not per environment, since there's only one cluster).

A `PersistentVolumeClaim` for a 500Mi `ReadWriteOnce` volume. No `PersistentVolume` was hand-written; the cluster's default `StorageClass` (`local-path`) provisions and binds a matching volume dynamically, the moment a pod that references the claim actually gets scheduled.

A `Deployment` running a single pgAdmin replica, with explicit CPU and memory requests and limits (non-negotiable from this project's Phase 1 forward; see the [incident below](../incidents/oom-killed-pgadmin.md) for why that number mattered) and a `volumeMount` that redirects `/var/lib/pgadmin` onto the PVC, so pgAdmin's saved connections survive a pod replacement instead of living in the container's disposable writable layer.

A `ClusterIP` `Service`, selecting on the pod's label, which decouples the Service's own port from the container's actual listening port.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgadmin-deployment
  namespace: pgadmin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pgadmin
  template:
    metadata:
      labels:
        app: pgadmin
    spec:
      containers:
        - name: pgadmin-container
          image: dpage/pgadmin4:latest
          ports:
            - containerPort: 80
          env:
            - name: PGADMIN_DEFAULT_EMAIL
              value: "<redacted>"
            - name: PGADMIN_DEFAULT_PASSWORD
              value: "<redacted — plaintext here is a known, deliberate gap; Phase 4 replaces this with Sealed Secrets>"
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
          volumeMounts:
            - name: data
              mountPath: /var/lib/pgadmin
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: pgadmin-pvc
```

## Verification, not assumption

"It applied without errors" isn't the same as "it works." Confirmed end to end before calling this done:

- `kubectl get pvc` → `Bound`, not just created
- `kubectl get pods` → `1/1 Running`, `0` restarts (after fixing the incident below)
- `kubectl get endpoints pgadmin-service` → showed the pod's actual IP:80, proving the Service's label selector genuinely found the pod, not just that both objects exist independently
- A direct `curl` to the Service's ClusterIP returned `HTTP 302`, pgAdmin's real login-page redirect. An actual response came back through Service, pod, and container, not a connection that merely didn't error.

## What this phase actually taught

- The container-vs-pod field boundary in a manifest (`env`, `resources`, `volumeMounts` belong to one container; `volumes` is a pod-level sibling), and `kubectl explain <path>` as the tool for checking that against the live API schema instead of trusting a blog post.
- `PersistentVolume` vs `PersistentVolumeClaim`, and why dynamic provisioning means the PV is never hand-written here.
- Why a Service's `port` and `targetPort` are fully decoupled.
- That Kubernetes tells you when a resource limit is wrong, loudly, with an exit code, rather than making you guess it right the first time.

Full technical notes for this phase live in the private working notes; this is the portfolio-facing summary.
