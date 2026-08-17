# Longhorn backup target — Garage (S3) on the NAS

Longhorn stores volume backups in an S3-compatible bucket served by
**Garage** running in Docker on the NAS (`192.168.3.20`). MinIO is deliberately
not used: the community edition was archived in February 2026 (read-only, no
maintained images), so it is not a viable dependency for this cluster.

This is bulk backup storage only. Longhorn's *replicas* stay on the nodes'
local disks (`/var/lib/longhorn`) — S3 is object storage and cannot back block
volumes.

---

## Table of contents

1. [Where things live](#where-things-live)
2. [Part 1 — Garage on the NAS](#part-1--garage-on-the-nas)
3. [Part 2 — The Kubernetes secret](#part-2--the-kubernetes-secret)
4. [Part 3 — Longhorn configuration](#part-3--longhorn-configuration)
5. [Verification](#verification)
6. [Troubleshooting](#troubleshooting)
7. [Known gaps / TODO](#known-gaps--todo)

---

## Where things live

| Thing | Location |
|-------|----------|
| Garage container | NAS, UGOS Pro Docker → Projects → `garage` |
| Garage config | `/volume1/docker/garage/config/garage.toml` (NAS) |
| Garage metadata | `/volume1/docker/garage/meta` (NAS) |
| Garage object data | `/volume1/docker/garage/data` (NAS) |
| S3 endpoint | `http://192.168.3.20:3900` |
| Admin API | `http://192.168.3.20:3903` (token in `garage.toml`) |
| Web UI | `http://192.168.3.20:3909` (garage-webui) |
| Bucket | `longhorn-backups` |
| Region | `garage` |
| K8s secret | `infrastructure/secrets/garage-backup.yaml` → `longhorn-system` |
| Longhorn settings | inline `valuesObject` in `apps/longhorn.yaml` |

> The NAS has **no SSH access** in this setup. Everything on the NAS side is
> done through the UGOS Pro web UI plus the Garage web UI. That constraint is
> why the config file is uploaded rather than written in place.

---

## Part 1 — Garage on the NAS

### 1.1 Folders

UGOS Pro can only bind-mount paths under `/volume1/<share>`. There is no
user-writable location outside a share, so a share is mandatory — but it needs
no protocols enabled. Create the share `docker` with SMB/NFS **off**, access
limited to the admin user, then create inside it:

```
docker/garage/config/
docker/garage/meta/
docker/garage/data/
```

### 1.2 `garage.toml`

Written locally and uploaded to `docker/garage/config/` via File Manager.
Secrets are **not** in this repo — they live in the password manager.

```toml
metadata_dir = "/var/lib/garage/meta"
data_dir     = "/var/lib/garage/data"
db_engine    = "lmdb"

replication_factor = 1

rpc_bind_addr   = "[::]:3901"
rpc_public_addr = "127.0.0.1:3901"
rpc_secret      = "<32-byte hex, password manager>"

[s3_api]
s3_region     = "garage"
api_bind_addr = "[::]:3900"
root_domain   = ".s3.garage.local"

[s3_web]
bind_addr   = "[::]:3902"
root_domain = ".web.garage.local"
index       = "index.html"

[admin]
api_bind_addr = "[::]:3903"
admin_token   = "<password manager>"
metrics_token = "<password manager>"
```

> `replication_factor = 1` is mandatory on a single node — Garage refuses to
> start otherwise.

> When editing on Windows, save as "All files" with UTF-8. Notepad's default
> produces `garage.toml.txt` and the container dies with *config file not
> found*. Confirm the filename in File Manager after upload.

### 1.3 Compose project

UGOS Pro Docker → **Projects** → Create → paste:

```yaml
services:
  garage:
    image: dxflrs/garage:v2.1.0
    container_name: garage
    restart: unless-stopped
    ports:
      - "3900:3900"   # S3 API
      - "3903:3903"   # Admin API
    volumes:
      - /volume1/docker/garage/config/garage.toml:/etc/garage.toml:ro
      - /volume1/docker/garage/meta:/var/lib/garage/meta
      - /volume1/docker/garage/data:/var/lib/garage/data

  garage-webui:
    image: khairul169/garage-webui:latest
    container_name: garage-webui
    restart: unless-stopped
    depends_on: [garage]
    ports:
      - "3909:3909"
    volumes:
      - /volume1/docker/garage/config/garage.toml:/etc/garage.toml:ro
    environment:
      API_BASE_URL: "http://garage:3903"
      S3_ENDPOINT_URL: "http://garage:3900"
```

RPC port 3901 is intentionally not published — a single node talks to itself
over `rpc_public_addr` inside the container.

`API_BASE_URL` points at **3903** (admin API), not 3900. Pointing it at the S3
port yields a web UI that loads but shows no data.

### 1.4 Layout, key, bucket (web UI, `:3909`)

1. **Cluster / Layout** → assign the node a zone (`nas`) and a capacity
   (`2T`) → **Apply**.
2. **Keys** → create `longhorn` → copy both values immediately, the secret is
   shown once.
3. **Buckets** → create `longhorn-backups` → grant the `longhorn` key
   read + write.

> ⚠️ **The layout apply is not optional.** Until a layout version is applied,
> Garage accepts no writes and every S3 call fails with a service-unavailable
> style error. This is the single most common reason a fresh Garage looks
> healthy but rejects backups.

---

## Part 2 — The Kubernetes secret

One SealedSecret, namespace `longhorn-system`, name `garage-backup`. Four keys
— Longhorn reads these names exactly:

| Key | Value |
|-----|-------|
| `AWS_ACCESS_KEY_ID` | Garage key ID (`GK…`) |
| `AWS_SECRET_ACCESS_KEY` | Garage secret |
| `AWS_ENDPOINTS` | `http://192.168.3.20:3900` |
| `VIRTUAL_HOSTED_STYLE` | `false` |

Path-style addressing is required: there is no wildcard DNS for
`*.s3.garage.local`, so virtual-hosted-style requests cannot resolve.

Created per `infrastructure/secrets/README.md` — same ASCII/`WriteAllText`
handling, since PowerShell 5.1 would otherwise write UTF-16 with a BOM that
ArgoCD cannot parse:

```powershell
$sealed = kubectl create secret generic garage-backup -n longhorn-system `
  --from-literal=AWS_ACCESS_KEY_ID="GK..." `
  --from-literal=AWS_SECRET_ACCESS_KEY="..." `
  --from-literal=AWS_ENDPOINTS="http://192.168.3.20:3900" `
  --from-literal=VIRTUAL_HOSTED_STYLE="false" `
  --dry-run=client -o yaml |
  kubeseal --format=yaml `
    --controller-name=sealed-secrets `
    --controller-namespace=sealed-secrets |
  Out-String

[System.IO.File]::WriteAllText(
  "$PWD\infrastructure\secrets\garage-backup.yaml",
  $sealed.TrimEnd() + "`n",
  [System.Text.Encoding]::ASCII)
```

The `secrets` Application (`apps/secrets.yaml`) has
`destination.namespace: tools`, which is only a fallback — this file **must**
carry `metadata.namespace: longhorn-system` or the secret lands in the wrong
namespace and Longhorn never sees it.

---

## Part 3 — Longhorn configuration

In `apps/longhorn.yaml`, under `spec.source.helm.valuesObject.defaultSettings`:

```yaml
        defaultSettings:
          defaultDataPath: /var/lib/longhorn
          backupTarget: s3://longhorn-backups@garage/
          backupTargetCredentialSecret: garage-backup
```

Target string anatomy: `s3://<bucket>@<region>/`. The part after `@` is the
**region**, not a host — it must match `s3_region` in `garage.toml`. A mismatch
surfaces as a signature error, which reads like a wrong key and sends you
looking in the wrong place.

The trailing slash matters. Without it some Longhorn versions treat the target
as malformed.

---

## Verification

```powershell
kubectl -n longhorn-system get backuptarget default `
  -o jsonpath='{.status.available}{"\n"}'
```

`true` means Longhorn reached the bucket. If not:

```powershell
kubectl -n longhorn-system get backuptarget default -o yaml
```

`.status.conditions` carries the actual S3 error.

End-to-end check: take a manual backup of a small volume in the Longhorn UI,
then confirm the objects appear under `longhorn-backups` in the Garage web UI
object browser (`backupstore/volumes/…`).

---

## Troubleshooting

**`.status.available: false`, connection refused / timeout**
Nodes cannot reach `192.168.3.20:3900`. Check the Garage container is running
in the UGOS Docker UI and that port 3900 is actually published in the Compose
project.

**`SignatureDoesNotMatch` / `InvalidRegion`**
`@garage` in the backup target does not match `s3_region` in `garage.toml`.
This is far more often the cause than a mistyped secret.

**`AccessDenied` / 403**
The key exists but has no permission on the bucket. In garage-webui →
Buckets → `longhorn-backups` → grant the key read **and** write. Read-only
lets Longhorn list the target and then fail on upload, which looks like an
intermittent fault.

**Service unavailable, no nodes available**
Layout was never applied (Part 1.4, step 1).

**Backup target reverts or ignores repo changes**
`defaultSettings.backupTarget` only *seeds* the `BackupTarget` custom
resource. Once the target is edited in the Longhorn UI, the CR wins and Helm
stops overriding it — the repo then silently drifts from the cluster. Fix by
deleting the CR and letting Longhorn re-seed:

```powershell
kubectl -n longhorn-system delete backuptarget default
```

Then hard-refresh the ArgoCD app. Rule: change the target in Git only.

**Web UI at `:3909` loads but shows nothing**
Admin API not reachable. Verify the `[admin]` block exists in `garage.toml`
and `API_BASE_URL` uses port 3903.

**Restic/backing-image style timeouts on large volumes**
Metadata I/O is Garage's bottleneck. If `meta` sits on a spinning volume,
move that bind mount to an SSD volume and restart the project.

---

## Known gaps / TODO

- **Plain HTTP.** The S3 endpoint is unencrypted on the LAN. Fronting Garage
  with Traefik for TLS would additionally require `AWS_CERT` in the secret
  when using a private CA.
- **Single node, `replication_factor = 1`.** No redundancy inside Garage. A
  NAS disk failure loses the backups; the NAS's own RAID is the only
  protection. This is a backup *target*, not a second copy — the 3-2-1 rule is
  not satisfied by this alone.
- **No lifecycle policy.** Old backups are not expired automatically. Longhorn
  recurring-job retention is the only pruning in place.
- **No monitoring.** Garage exposes Prometheus metrics on the admin port
  (`metrics_token`); not yet scraped by kube-prometheus-stack.
- **Not in GitOps.** The Garage deployment itself is clicked into the NAS
  Docker UI. The Compose file above is the only record — consider committing
  it under `nas/` so a rebuild is reproducible.
