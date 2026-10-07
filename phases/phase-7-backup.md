# Phase 7: backup and restore with Velero

**Status:** ✅ Done
**Goal:** back up the cluster to storage that isn't on the NAS, and prove it by actually restoring something, since a backup that has never been restored doesn't count.

## Off the NAS, onto Cloudflare R2

The NAS has no RAID, so a backup that sits on the same disks doesn't protect against the thing most likely to go wrong. I picked Cloudflare R2 for the target. It speaks the same API as Amazon S3, which is what Velero expects, it has a 10 GB free tier with no download fees, and I already had the Cloudflare account from Phase 5.

The bucket is called `homelab-velero`, and it only holds Velero's backups. If I back up the media stack's config to R2 later, that gets its own bucket and its own token, so a leaked key for one can't read the other. The token itself is limited to Object Read & Write on that one bucket.

## Deciding what's worth backing up

Most of the cluster is already in Git, and Argo CD can rebuild it from there. What Git can't give back is data. So the backup covers the volumes for Prometheus, Grafana, Alertmanager, Jenkins and pgAdmin, plus the Kubernetes objects in those namespaces.

The media stack is mostly left out. The library itself is an rclone mount of content that lives with my debrid and Usenet providers, so there's nothing on the NAS to copy and no point trying. The parts that would hurt to lose, zurg's and Jellyfin's config, sit in hostPath folders, and Velero's file-level backup can't read those. That's a separate job for later.

## The master key stays out, and so do Secrets

My first instinct was to put the Sealed Secrets master key in the bucket too, since nobody can reach my NAS without being on my tailnet, but that misses how the key is used. My SealedSecrets are already public in the `homelab-config` repo, so anyone with the master key can clone the repo and decrypt every one of them on their own laptop without ever touching my network. The key stays in my password manager and iCloud, and nowhere else.

The same thinking applies to the backups themselves. Velero backs up Kubernetes objects by asking the API for them, and the API hands Secrets back in their decrypted form. Those go into a compressed archive in the bucket without any extra encryption, so a backup of the `monitoring` namespace would have put my Grafana password and Slack webhook in R2 in readable form. I exclude Secrets from every backup instead. The ones that come from SealedSecrets rebuild themselves from Git plus the master key, and the handful I created by hand are in my password manager.

## Getting the credentials in

Velero's AWS plugin wants its credentials as a whole AWS-style credentials file stored under a single key called `cloud`, rather than as separate values. `--from-file` handles that, and the key name goes in front of the path:

```bash
kubectl create secret generic velero-r2-credentials \
  --from-file=cloud=r2.text \
  --namespace=velero \
  --dry-run=client -o yaml | \
kubeseal --controller-namespace sealed-secrets --controller-name sealed-secrets \
  --format yaml > /volume1/Storage/homelab-config/manifests/velero/sealed-secret.yaml
```

I kept the plaintext file outside the repo and deleted it straight after, since `git add -A` would have happily committed it to a public repo. Once it decrypted in the cluster, `describe` showed the `cloud` key at 148 bytes, which is exactly what a correct file comes to with a 32 character key ID and a 64 character secret.

Like Phase 6, this is two Argo CD apps: `velero-secrets` for the namespace and the SealedSecret, and `velero` for the chart, so a sealing mistake can't get tangled up with the install.

## An R2 bug in the plugin

When I checked versions, I found that velero-plugin-for-aws 1.14.0 through 1.14.2 send an empty tagging header that R2 rejects, so every backup fails with `501 NotImplemented: Header 'x-amz-tagging'`. The fix shipped in 1.14.3, and I'm on 1.14.4. The chart's own example still points at 1.13.1, so it's worth checking rather than copying.

```yaml
initContainers:
  - name: velero-plugin-for-aws
    image: velero/velero-plugin-for-aws:v1.14.4
configuration:
  backupStorageLocation:
    - name: default
      provider: aws
      bucket: homelab-velero
      default: true
      config:
        region: auto
        s3ForcePathStyle: "true"
        s3Url: https://<account-id>.r2.cloudflarestorage.com
snapshotsEnabled: false
deployNodeAgent: true
```

The image tag also needs the `v` (`v1.14.4` exists, `1.14.4` doesn't), and `s3Url` is the account endpoint without the bucket on the end, even though Cloudflare's dashboard shows it with the bucket attached. The real test that it all worked was the backup location showing `Available`, which means Velero reached R2 and was let in with the sealed credentials.

## File System Backup, not snapshots

There are two ways Velero can back up a volume. A CSI snapshot asks the storage system to freeze a copy of the whole volume at one moment, but my cluster doesn't have the snapshot controller or CRDs installed, and a Longhorn snapshot stays on my own disks anyway. File System Backup runs a small agent on each node that reads the files out of each volume and uploads them through Kopia, which also deduplicates them, so unchanged data is only stored once.

I made it opt-in rather than backing up every volume automatically. Each pod gets an annotation naming the volume to back up:

```yaml
backup.velero.io/backup-volumes: prometheus-monitoring-kube-prometheus-prometheus-db
```

These pods also have plenty of scratch volumes (plugin caches, temp folders, generated config), and backing up everything would have uploaded all of them for nothing.

## Bringing Jenkins into Git first

Adding Jenkins' annotation turned up a gap: Jenkins was installed with the `helm` command back in Phase 3 and was never in Git, so there was no file to put the annotation in, and nothing would reinstall it after a rebuild. I moved it into Argo CD as its own Application, using the same release name, chart version and values so that Argo would take over the running install instead of creating a second one.

The chart reuses Jenkins' admin password by looking up the existing Secret with Helm's `lookup` function, and only generates a new random one if it finds nothing. Argo renders charts without access to the cluster, so the lookup always finds nothing, and every render would have produced a new password and overwritten the old one without any warning. Pointing the chart at a SealedSecret holding the current password avoids it:

```yaml
controller:
  nodeSelector:
    workload-strength: cpu-heavy
  podAnnotations:
    backup.velero.io/backup-volumes: jenkins-home
  admin:
    existingSecret: jenkins-credentials
```

Before pushing, I rendered the chart and ran `kubectl diff` against the live Jenkins. The only changes were the backup annotation and the Secret name, so Argo took it over without recreating anything and my login kept working. Afterwards I deleted Helm's leftover Secret and its release record with `kubectl`, and not with `helm uninstall`, which would have deleted every object the chart created, including the volume with all the jobs and build history.

## First backup, then a schedule

```bash
velero backup create first-manual --include-namespaces monitoring,jenkins,pgadmin --exclude-resources secrets --wait
```

It finished in under three minutes with 296 objects and no errors. Prometheus came to 2.55 GB, about half of what Longhorn's volume size suggested, because Longhorn counts space that was used and later freed.

The schedule lives in the chart values in Git, not as a one-off CLI command. It runs every day at 3am Eastern, which is `0 7 * * *` because Velero schedules run on UTC, and Velero's time zone option has a history of bugs. Each backup is kept for 14 days. I also left `useOwnerReferencesInBackup` off, because with it on, every backup is "owned" by the Schedule, and if Argo ever removed the Schedule, Kubernetes would delete every backup along with it.

Backups now report into Prometheus from Phase 6, with two alerts: one for a scheduled backup that fails, and one for no successful backup in 25 hours, which catches the quieter case where the schedule just stops running. Both needed the `release: monitoring` label, the same lesson as the cloudflared ServiceMonitor.

## The restore test

Deleting something and restoring it doesn't work well in a GitOps cluster. Velero skips any object that already exists, and Argo CD recreates deleted apps within seconds, so it would put back an empty pgAdmin before Velero got there, Velero would skip it, and the restore would still report success with none of my data in it.

So I restored into a new namespace instead. I added a server group called `restore-test-oct7` in pgAdmin so there was something specific to look for, took a fresh backup, and restored it as a copy next to the real one:

```bash
velero restore create pgadmin-restore-test --from-backup pgadmin-restore-test \
  --namespace-mappings pgadmin:pgadmin-restore \
  --exclude-resources ingresses,sealedsecrets.bitnami.com --wait
```

The SealedSecret was sealed for the `pgadmin` namespace and won't decrypt anywhere else, so I left it out and created a temporary plain Secret in the test namespace for the copy to start with. I left out the Ingress too, because it would have claimed the same hostname as my real pgAdmin, and reached the copy through a port-forward instead. I logged into the restored copy, `restore-test-oct7` was there, and then I deleted the test namespace.

## A detour through pgAdmin

Getting to that test took a while, because I locked myself out of pgAdmin first.

pgAdmin only uses the password from its Secret once, when it first creates its database, and after that it only trusts the password stored in its own database file. When I rotated the password in Phase 4, I changed the Secret and never changed pgAdmin, so the two had been different ever since, and nobody noticed until I tried to log in.

Fixing it meant running pgAdmin's own admin tool inside the pod, and that got killed for running out of memory at the 256Mi limit, the second time pgAdmin has run out of memory in this project, so the limit is 512Mi now. A container's memory limit has to cover everything that might run in it, admin commands included, not just the app on a normal day. The tool's password reset also crashes with `KeyError: 'role'` unless you pass `--role`.

## What's left

Three things are still installed with Helm outside of Git: Argo CD itself, Longhorn and the Sealed Secrets controller. A full rebuild would depend on reinstalling them by hand with the right settings, so that goes on the list for Phase 8, along with a real in-place restore with Argo paused.

The cluster's data is now set to leave the NAS every night, I know it comes back because I've restored it, and I'll hear about it in Slack if it stops.
