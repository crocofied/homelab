# CNPG (Postgres) backups — Garage (S3) on the NAS

CloudNativePG (CNPG) ships base backups and continuous WAL archiving for
every `Cluster` to the same **Garage** S3-compatible instance already used
for Longhorn backups (`http://192.168.3.20:3900`, see
[longhorn-backup-garage.md](longhorn-backup-garage.md) for how Garage itself
is set up on the NAS). This is a separate bucket, dedicated to Postgres.

All four CNPG clusters (`paperless-db`, `tandoor-db`, `n8n-db`, `zipline-db`)
back up into the same bucket under a per-cluster prefix, and each has its own
daily `ScheduledBackup`.

---

## Status

- Manifests updated (`infrastructure/databases/*/`.yaml): every `Cluster`
  now has a `spec.backup.barmanObjectStore` block and an
  `argocd.argoproj.io/sync-options: Prune=false` annotation.
- A `*-db-backup.yaml` `ScheduledBackup` sits next to each `Cluster` manifest
  (daily, staggered 03:00–03:45 CEST).
- **Not done yet**: the Garage bucket/key and the `cnpg-backup` Kubernetes
  secret. CNPG will report the backup as failing (`.status` conditions on
  the `Cluster` and unschedulable `Backup` objects) until these exist. See
  below.

---

## Part 1 — Garage bucket + key (garage-webui, `:3909`)

1. **Buckets** → create `cnpg-backups`.
2. **Keys** → create `cnpg` → copy both the key ID (`GK…`) and the secret
   immediately, it's shown once.
3. Grant the `cnpg` key read + write on `cnpg-backups`.

Same instance as Longhorn's backups, different bucket and a dedicated key
(least privilege — this key can't touch `longhorn-backups`).

---

## Part 2 — The Kubernetes secret

Unlike Longhorn, CNPG's `s3Credentials` block wants the region as a
**secret key reference** too, so the secret carries three keys instead of
two:

| Key | Value |
|-----|-------|
| `AWS_ACCESS_KEY_ID` | Garage key ID (`GK…`) |
| `AWS_SECRET_ACCESS_KEY` | Garage secret |
| `AWS_REGION` | `garage` (must match `s3_region` in `garage.toml`, same as the `@garage` in the Longhorn backup target URL) |

Two `Cluster`s live in `media` (`paperless-db`, `zipline-db`), two live in
`tools` (`tandoor-db`, `n8n-db`) — a `Secret` doesn't cross namespaces, so
the same credentials need to be sealed **twice**, once per namespace, both
named `cnpg-backup`.

```powershell
foreach ($ns in "media", "tools") {
  $sealed = kubectl create secret generic cnpg-backup -n $ns `
    --from-literal=AWS_ACCESS_KEY_ID="GK..." `
    --from-literal=AWS_SECRET_ACCESS_KEY="..." `
    --from-literal=AWS_REGION="garage" `
    --dry-run=client -o yaml |
    kubeseal --format=yaml `
      --controller-name=sealed-secrets `
      --controller-namespace=sealed-secrets |
    Out-String

  [System.IO.File]::WriteAllText(
    "$PWD\infrastructure\secrets\cnpg-backup-$ns.yaml",
    $sealed.TrimEnd() + "`n",
    [System.Text.Encoding]::ASCII)
}
```

Same ASCII/`WriteAllText` handling as every other sealed secret in this
repo — PowerShell 5.1's default `Out-File`/`>` produces UTF-16 with a BOM,
which ArgoCD/kubeseal can't parse.

Each generated file needs `metadata.namespace` set explicitly (`media` /
`tools`) — the `secrets` Application's `destination.namespace: tools` is
only a fallback, same caveat as `garage-backup.yaml`.

Commit both files under `infrastructure/secrets/` and push. ArgoCD picks
them up via the `secrets` Application (sync-wave `2`, same wave as the DB
Applications — the secret and the `Cluster` land close enough together that
CNPG just retries until the secret exists).

---

## Verification

```powershell
# WAL archiving + backup target reachable
kubectl get cluster -n media paperless-db -o jsonpath='{.status.conditions}'

# on-demand test backup (don't wait for 03:00)
kubectl cnpg backup paperless-db -n media

# watch it
kubectl get backups.postgresql.cnpg.io -n media -w
```

A `Backup` with `.status.phase: completed` confirms the round trip. Check
the Garage web UI object browser for `cnpg-backups/paperless-db/...` to see
the actual objects.

Repeat for `tandoor-db` / `n8n-db` (namespace `tools`) and `zipline-db`
(namespace `media`).

---

## Restoring

CNPG restores by **bootstrapping a new cluster from an object store**, not
by editing a live one — see the
[CNPG recovery docs](https://cloudnative-pg.io/documentation/current/recovery/).
Skeleton:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: paperless-db-restored
  namespace: media
spec:
  instances: 1
  storage:
    size: 5Gi
    storageClass: longhorn
  bootstrap:
    recovery:
      source: paperless-db
  externalClusters:
    - name: paperless-db
      barmanObjectStore:
        destinationPath: s3://cnpg-backups/paperless-db
        endpointURL: http://192.168.3.20:3900
        s3Credentials:
          accessKeyId:
            name: cnpg-backup
            key: AWS_ACCESS_KEY_ID
          secretAccessKey:
            name: cnpg-backup
            key: AWS_SECRET_ACCESS_KEY
          region:
            name: cnpg-backup
            key: AWS_REGION
```

Point the app's `PAPERLESS_DBHOST` (etc.) secret at the restored cluster's
`-rw` service once it's healthy, then decommission the old one.

---

## Why `Prune=false` on the `Cluster` resources

This backup setup exists because `paperless-db` got wiped on 2026-08-03: an
ArgoCD-tracked path move (`infrastructure/databases/paperless-db.yaml` →
`infrastructure/databases/paperless/paperless-db.yaml`, done as two
back-to-back commits) briefly left the live `Cluster` unmatched by its
`Application`'s source path. With `prune: true` + `selfHeal: true`, ArgoCD
deleted the `Cluster`; CNPG cascade-deleted its PVC; the next sync created a
fresh, empty one.

`argocd.argoproj.io/sync-options: Prune=false` on the `Cluster` metadata
(now present on all four) makes ArgoCD refuse to auto-delete it even if the
resource momentarily disappears from source — it'll show
`OutOfSync`/orphaned instead of silently pruning, so a future path
reshuffle degrades to a stuck sync rather than a lost database.

This is a second, independent line of defense on top of that annotation —
if the resource is ever deleted anyway (e.g. someone intentionally deletes
the `Cluster`, or `Prune=false` gets removed later), the S3 backups let it
be restored instead of starting over with an empty database.

---

## Known gaps / TODO

- Same single-node Garage caveats as the Longhorn target (see
  [longhorn-backup-garage.md](longhorn-backup-garage.md#known-gaps--todo)):
  plain HTTP, no redundancy inside Garage, no lifecycle policy beyond CNPG's
  own `retentionPolicy: 14d`.
- No alerting if a `ScheduledBackup` silently stops producing successful
  `Backup` objects — worth a Prometheus rule once Garage/CNPG metrics are
  scraped.
