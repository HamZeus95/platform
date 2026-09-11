# Runbook: PersistentVolumeAlmostFull

**Severity:** warning · **Fires when:** < 15% of a PVC remains for 5m

## Why this one matters more than it looks

Storage is `local-path`, which provisions **hostPath** volumes on a single node.
There is no replication and no rescheduling around a full or failed disk. A full
Postgres volume means failed writes; a full Prometheus volume means silent gaps
in the metrics you are trying to debug with.

## Which volume, and on which node

```bash
kubectl --context prod get pvc -A
```

```promql
(1 - kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes)
```

Volumes live under `/var/local-path-provisioner/` on the owning node:

```bash
TALOSCONFIG=~/talos/talosconfig talosctl -n <node-ip> -e 192.168.1.118 \
  list -l /var/local-path-provisioner
```

## By volume

**`default/db-data` (Postgres, 5Gi).** Check real size, then look for bloat or
an unvacuumed table. Back up before any cleanup:

```bash
kubectl --context prod exec -n default deploy/db -- \
  pg_dump -U postgres app > ~/homelab-backups/app-$(date +%F).sql
```

**`monitoring/prometheus-...-db` (10Gi).** Lower `retention` in
`apps/prod/monitoring.yaml` (currently 15d) and let ArgoCD apply it.

**`monitoring/storage-loki-0` (10Gi).** Lower `limits_config.retention_period`
(currently 168h). The compactor enforces retention.

## Growing a volume

local-path does **not** support online expansion. To grow: back up, delete the
PVC and its workload, raise the size in git, let ArgoCD recreate it, restore.
Plan a maintenance window — this is downtime, not a resize.

## Prevention

The durable fix is replicated storage (Longhorn), which also removes the
node-local single point of failure. See [[auth-app-target-down]] for how a lost
node strands an RWO volume.
