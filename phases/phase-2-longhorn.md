# Phase 2: storage with Longhorn

**Status:** ✅ Done
**Goal:** replace `local-path` with Longhorn as the default StorageClass, so volumes are replicated across nodes instead of living on a single disk.

## The problem with where we started

`local-path` writes a PVC's data to a directory on whichever node the pod landed on, and nowhere else. If that node dies, the data dies with it, and the pod can't be rescheduled anywhere because its storage doesn't exist anywhere else. Fine for scratch space, wrong for anything worth keeping.

Longhorn solves that by presenting a replicated block device. Writes go synchronously to replicas on separate machines before being acknowledged, so a node can disappear and the volume stays readable from a surviving copy.

## Two host-level blockers on the NAS

The install went in cleanly on both Ubuntu workers and failed only on `samir-nas`, the UGREEN NAS acting as control plane. Two separate problems, neither visible from `kubectl`:

**`iscsid` was installed but never started or enabled.** Longhorn attaches volumes over iSCSI, so its CSI plugin cannot function without it. The `iscsi_tcp` module shipped with the UGREEN kernel but had never been loaded. Both workers already had this running, which is why it looked node-specific rather than like a Longhorn problem.

**`/var/lib/longhorn` sat on an overlay filesystem.** UGOS boots with a read-only base plus a writable overlay, so the NAS's root filesystem is type `overlay` rather than a real one. Longhorn's replicas are sparse files that want `O_DIRECT`, which overlayfs handles badly.

The useful discovery was that the NAS had perfectly good ext4 underneath the whole time, both on the partition backing the overlay and on the 7.3TB `/volume1` pool. So rather than excluding the NAS from Longhorn, the fix was a bind mount:

```bash
mount --bind /volume1/longhorn /var/lib/longhorn
echo '/volume1/longhorn /var/lib/longhorn none bind 0 0' >> /etc/fstab
```

That keeps `defaultDataPath` identical across all three nodes while the NAS's bytes land on real ext4. It reports 7.4TB schedulable instead of the 96GB overlay partition, which is how you confirm the mount actually took effect.

## Install and the two settings that mattered

Installed via Helm rather than raw manifests, specifically so there'd be a tracked release to upgrade and roll back rather than a pile of orphaned objects.

Two decisions worth explaining:

**Replica count 2, not 3.** With three nodes, three replicas means every volume occupies every node and there is no spare capacity. Take one node down and the volume sits degraded until that exact machine returns, because there's nowhere to rebuild. With two replicas there's always a third node free, so Longhorn heals onto it immediately. Three replicas buys tolerance of two simultaneous node failures, which matters much less here than being able to take a node down for maintenance.

**Longhorn as the default StorageClass.** Checked first that nothing would silently move: every existing PVC in the media namespace pins `storageClassName: local-path` explicitly, and the media workloads use `hostPath` volumes which ignore StorageClasses entirely. Changing the default only affects new PVCs that omit the field.

A gotcha worth knowing: setting `defaultSettings.defaultReplicaCount=2` is not enough. The chart's StorageClass hardcodes `numberOfReplicas` from a separate value, and a StorageClass parameter overrides the global default. The global setting only governs volumes created without that parameter, such as ones made through the UI. The value you actually want is `persistence.defaultClassReplicaCount`.

## Migrating pgAdmin off local-path

`storageClassName` is immutable on a PVC, so there's no in-place conversion. The sequence is always: create a new PVC, copy the data, repoint the app, delete the old one.

Scaling the deployment to zero first matters. Copying a live database file while something is writing to it produces a corrupt copy.

The copy itself needs a throwaway pod, because it's the only way to have both volumes mounted simultaneously. The deployment can only mount one. A `busybox` pod with both PVCs mounted at `/old` and `/new`, running `cp -a /old/. /new/`, does the job. `-a` rather than `-r` preserves the ownership pgAdmin cares about.

One detail that becomes obvious in hindsight: the old `local-path` PV has node affinity pinning it to the machine it was created on, so the copy pod gets scheduled there automatically. The Longhorn volume can attach anywhere, so it follows along.

## Killing a node on purpose

The bar for this phase was watching a volume survive a node dying, not just watching a test volume get created and deleted.

Setup: pgAdmin's pod and one of its two replicas both lived on `mini-pc-ubuntu`. The other replica was on `g14-server`, and `samir-nas` held none, making it the spare. Killing `mini-pc-ubuntu` tests everything at once, since the pod has to reschedule *and* the volume has to reattach elsewhere.

What happened:

1. Node went `NotReady` within about 40 seconds.
2. The pod was rescheduled to `g14-server`, then sat in `ContainerCreating` and stayed there.
3. The volume was stuck in `attaching`, and stayed stuck.

The reason turned out to be instructive. The original pod was still shown as `Terminating` on the dead node, and Kubernetes will never finish deleting it, because the kubelet that would confirm the container stopped is gone with the machine. Longhorn in turn won't attach the volume to a second node while something still claims it. The unblock is an explicit force delete:

```bash
kubectl delete pod -n pgadmin <pod> --force --grace-period=0
```

That is not a workaround for a bug. Kubernetes deliberately refuses to do this automatically because it cannot distinguish "the node is dead" from "the node is unreachable but still running and writing." Forcing it is an assertion that you know which one it is. Get that wrong and two pods write to one ReadWriteOnce volume, which is how filesystems get corrupted.

After the force delete the volume attached to `g14-server` and pgAdmin came back with its data intact, served from the surviving replica.

## The rebuild that didn't happen, and why

The volume stayed `degraded` well past the ten minute replenishment interval, with a perfectly good spare node available. The cause was that the NAS's bind mount had silently reverted at some point earlier that day: `/var/lib/longhorn` was a plain overlay directory again, `/etc/fstab` no longer had the entry, and Longhorn reported `DiskFilesystemChanged` because the recorded disk UUID no longer matched what it found. A node whose disk isn't Ready can't receive a replica.

Restoring the bind mount and restarting that node's `longhorn-manager` brought the disk back to 6890GB available, and the rebuild onto `samir-nas` completed immediately. Volume went from `degraded` to `healthy` with replicas on two live nodes.

This is the documented risk of putting host configuration on an appliance OS's overlay layer: a vendor update or maintenance job can revert it. Worth knowing that anything configured this way on the NAS is not durable, and is the first thing to check when Longhorn misbehaves there.

## What this phase actually proved

A node was physically killed and a real application's data survived it, moved to a different machine, and then self-healed back to full redundancy without the dead node ever returning. That last part is the entire argument for replica count 2 on a three node cluster, demonstrated rather than assumed.
