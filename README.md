# Homelab

A three-node bare-metal Kubernetes cluster on **Talos Linux**, fully managed by
**ArgoCD** using the App-of-Apps pattern.

Everything after the initial bootstrap is declarative: two `kubectl apply`
commands and two `kubectl create secret` commands are the only manual steps in
the entire cluster's life. Everything else is a git commit.

---

## Table of contents

1. [Architecture](#architecture)
2. [Hardware](#hardware)
3. [Network plan](#network-plan)
4. [External dependencies](#external-dependencies)
5. [Repository layout](#repository-layout)
6. [Part 1 — Talos install](#part-1--talos-install)
7. [Part 2 — ArgoCD bootstrap](#part-2--argocd-bootstrap)
8. [Part 3 — Secrets](#part-3--secrets)
9. [Part 4 — Sync order](#part-4--sync-order)
10. [Adding a new app](#adding-a-new-app)
11. [Troubleshooting](#troubleshooting)
12. [Known gaps / TODO](#known-gaps--todo)

---

## Architecture

- **3 control planes, no dedicated workers.** `allowSchedulingOnControlPlanes: true`
  means workloads run on all three nodes.
- **Floating API VIP** at `192.168.3.130`, managed by Talos itself (no keepalived).
- **MetalLB** in L2 mode hands out LoadBalancer IPs from `192.168.3.140–149`.
- **Traefik** is the single ingress, reachable at `192.168.3.141`.
- **cert-manager** issues Let's Encrypt certs via Cloudflare DNS-01 for
  `*.damians.cloud`.
- **Longhorn** provides replicated block storage; the NAS provides bulk media
  over NFS.
- **ArgoCD** watches `apps/` and reconciles everything below it.

---

## Hardware

| Node  | CPU               | RAM  | Disk      | Role          | IP              | iGPU  |
|-------|-------------------|------|-----------|---------------|-----------------|-------|
| node1 | Intel N97         | 16GB | 512GB SSD | control plane | `192.168.3.133` | Intel |
| node2 | AMD Ryzen 3 4500U | 16GB | 512GB SSD | control plane | `192.168.3.182` | —     |
| node3 | Intel i5-10500T   | 32GB | 256GB SSD | control plane | `192.168.3.139` | Intel |

All three IPs are **static DHCP leases** on the router. They must stay outside
the MetalLB range.

---

## Network plan

| Purpose                 | Address                           |
|-------------------------|-----------------------------------|
| Kubernetes API VIP      | `192.168.3.130`                   |
| Node IPs                | `.133`, `.182`, `.139`            |
| MetalLB pool            | `192.168.3.140` – `192.168.3.149` |
| Traefik LoadBalancer    | `192.168.3.141`                   |
| NAS (NFS)               | `192.168.3.20`                    |

> ⚠️ The MetalLB pool **must** sit outside the router's DHCP range, otherwise the
> router will eventually hand a pool address to a random device and the ingress
> silently breaks.

---

## External dependencies

The cluster does not run standalone. These have to exist first:

**NAS — `192.168.3.20`**
Exports `/volume1/media` over **NFSv3**. Consumed by `pv-nas-media.yaml` with
`mountOptions: [nfsvers=3, nolock]`. NFSv4 and locking must stay off — Talos has
no rpc.statd, so `nolock` is mandatory or mounts hang.
The export needs read/write for the cluster subnet with no root squash issues
for the Jellyfin/Paperless UIDs.

**DNS — Technitium (split-horizon)**
A wildcard record `*.damians.cloud → 192.168.3.141` on the internal resolver, so
internal clients reach Traefik directly instead of going out to the public IP.

**Cloudflare**
DNS for `damians.cloud`. An API token with **Zone → DNS → Edit** on that zone is
used by cert-manager for DNS-01 challenges.

---

## Repository layout

```
apps/                  ArgoCD Application objects — the "table of contents"
                       the root app watches. Flat directory, one file per app.
apps-manifests/        Helm values and raw manifests for applications
infrastructure/        Cluster-level config (namespaces, MetalLB pool,
                       ClusterIssuer, CNPG cluster)
bootstrap/root-app.yaml  The App-of-Apps entry point
talos/                 Machine config patches + schematic definition
docs/                  Secret creation and other runbooks
```

> ⚠️ **The root app does not recurse into subdirectories.** Everything in `apps/`
> must be a flat `*.yaml`. A nested folder there is silently ignored.

---

## Part 1 — Talos install

### 1.1 Build the ISO with system extensions

Go to <https://factory.talos.dev>, choose **bare-metal**, pick the Talos version,
and select these extensions:

| Extension                    | Why                                      |
|------------------------------|------------------------------------------|
| `siderolabs/iscsi-tools`     | **Required by Longhorn** — volumes fail to attach without it |
| `siderolabs/util-linux-tools`| **Required by Longhorn**                 |
| `siderolabs/i915`            | Intel iGPU driver (Jellyfin QuickSync)   |
| `siderolabs/intel-ucode`     | Intel microcode                          |

This matches `talos/schematic.yaml`, schematic ID:

```
249d9135de54962744e917cfe654117000cba369f9152fbab9d055a00aa3664f
```

The factory is deterministic — the same definition always yields the same ID.
Verify rather than trust:

```powershell
curl.exe -X POST --data-binary (Get-Content talos\schematic.yaml -Raw) https://factory.talos.dev/schematics
```

Image URLs derived from the schematic:

```
# ISO
https://factory.talos.dev/image/249d9135de54962744e917cfe654117000cba369f9152fbab9d055a00aa3664f/<TALOS_VERSION>/metal-amd64.iso

# Installer (used by talosctl upgrade)
factory.talos.dev/installer/249d9135de54962744e917cfe654117000cba369f9152fbab9d055a00aa3664f:<TALOS_VERSION>
```

> **Pin the version.** Without `<TALOS_VERSION>` written down here, a rebuild in a
> year produces a different image than the cluster this repo describes.
> Last documented upgrade: `v1.13.7`. Confirm with `talosctl version` before
> relying on it.

### 1.2 Boot the nodes

Flash the ISO, boot all three nodes → they come up in **maintenance mode** with
a DHCP address shown on the console. Assign static DHCP leases now
(`.133`, `.182`, `.139`) so the addresses below stay correct.

> ⚠️ **If running on Proxmox instead of bare metal:** after installing, detach the
> ISO and set boot order to disk-first. Otherwise every reboot boots the ISO
> again — the node looks healthy but runs without your extensions or upgrades.
> Symptom: `talosctl get extensions` comes back empty. Also add
> `siderolabs/qemu-guest-agent` to the schematic in that case.

### 1.3 Generate the config

```powershell
cd C:\Users\damia\homelab\talos
talosctl gen config homelab https://192.168.3.130:6443
```

Produces `controlplane.yaml`, `worker.yaml` (unused — all nodes are control
planes), `talosconfig` and `secrets.yaml`. All of them are gitignored.

> ⚠️ **The `:6443` is mandatory.** Leave it out and the endpoint lands portless in
> both the cluster config and the kubeconfig, and you spend an evening chasing
> `dial tcp :443: connection refused`.

> 🔑 **Back up `secrets.yaml` and `talosconfig` to the password manager right now.**
> Without them the cluster can never be administered again. They must never enter
> the repo.

### 1.4 Patch matrix — which patch goes on which node

| Patch                        | node1 (.133) | node2 (.182) | node3 (.139) | Contents |
|------------------------------|:------------:|:------------:|:------------:|----------|
| `talos/patch.yaml`           | ✅ | ✅ | ✅ | API VIP `.130`, Longhorn bind mount, scheduling on control planes |
| `talos/intel-gpu-nodes.yaml` | ✅ | ❌ | ✅ | Node label `intel.feature.node.kubernetes.io/gpu=true` |
| `talos/disk-patch.yaml`      | ❌ | ❌ | ✅ | `install.disk: /dev/nvme0n1` |

**Why the GPU label must not go on node2:** the Intel device plugin uses that
label as its node selector. Labelling the Ryzen node makes Jellyfin schedulable
onto a machine with no `/dev/dri`, and transcoding silently falls back to CPU
instead of failing loudly.

**Why only node3 gets the disk patch:** node1 and node2 auto-detect their install
disk correctly; node3 needs it pinned to `/dev/nvme0n1`. Verify before applying:

```powershell
talosctl get disks --insecure -n 192.168.3.139
```

### 1.5 Apply the configs

```powershell
# node1 — Intel N97, GPU
talosctl apply-config --insecure -n 192.168.3.133 --file controlplane.yaml `
  --config-patch "@patch.yaml" `
  --config-patch "@intel-gpu-nodes.yaml"

# node2 — Ryzen, no GPU
talosctl apply-config --insecure -n 192.168.3.182 --file controlplane.yaml `
  --config-patch "@patch.yaml"

# node3 — i5-10500T, GPU + pinned NVMe install disk
talosctl apply-config --insecure -n 192.168.3.139 --file controlplane.yaml `
  --config-patch "@patch.yaml" `
  --config-patch "@intel-gpu-nodes.yaml" `
  --config-patch "@disk-patch.yaml"
```

`--insecure` is correct here: the nodes are still in maintenance mode and have no
PKI yet. Each node installs to disk and reboots.

### 1.6 Point talosctl at the cluster

```powershell
# Persistent — a plain $env: variable does not survive a new shell
[Environment]::SetEnvironmentVariable("TALOSCONFIG", "C:\Users\damia\homelab\talos\talosconfig", "User")
# open a new shell, then:

talosctl config endpoint 192.168.3.133 192.168.3.182 192.168.3.139
talosctl config node 192.168.3.133
```

Use the real node IPs as endpoints, not the VIP — the VIP only becomes active
once etcd has elected a leader, which happens in the next step.

### 1.7 Bootstrap etcd — once, on one node

```powershell
talosctl bootstrap
talosctl health          # 2–5 minutes; warnings before etcd comes up are normal
```

> ⚠️ **Exactly once per cluster lifetime, on exactly one node.** Running it a
> second time corrupts etcd.

### 1.8 Get the kubeconfig

```powershell
mkdir $HOME\.kube -Force
talosctl kubeconfig $HOME\.kube\config
kubectl get nodes -o wide     # → 3 nodes Ready, all control-plane
```

> Mnemonic for the classic mix-up: **talosconfig ↔ talosctl** (OS level),
> **kubeconfig ↔ kubectl** (cluster level). Two tools, two files, two variables.

### 1.9 Verify before continuing

```powershell
# Extensions present — if this is empty, go back to 1.1/1.2
talosctl get extensions -n 192.168.3.133

# GPU label on node1 and node3 only
kubectl get nodes -L intel.feature.node.kubernetes.io/gpu

# Longhorn bind mount on every node
talosctl -n 192.168.3.133,192.168.3.182,192.168.3.139 read /proc/mounts | Select-String longhorn

# VIP responds
kubectl cluster-info
```

If extensions are missing, fix without reinstalling:

```powershell
talosctl upgrade -n <node-ip> --image factory.talos.dev/installer/249d9135de54962744e917cfe654117000cba369f9152fbab9d055a00aa3664f:<TALOS_VERSION>
```

---

## Part 2 — ArgoCD bootstrap

ArgoCD needs neither MetalLB nor persistent storage, so it goes in first and is
reached by port-forward until Traefik exists. From here on, nothing is installed
by hand.

### 2.1 Install ArgoCD (manual step 1 of 2)

```powershell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w        # wait for all Running

# Initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | %{ [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }

kubectl port-forward svc/argocd-server -n argocd 8443:443
# → https://localhost:8443 · user: admin
```

ArgoCD then takes over managing itself via `apps/argocd.yaml`.

### 2.2 Connect the repository

ArgoCD UI → **Settings → Repositories → Connect Repo**: HTTPS, the repo URL,
GitHub username, and a **fine-grained PAT** (this repo only, read-only) as the
password.

### 2.3 Apply the root app (manual step 2 of 2)

```powershell
kubectl apply -f bootstrap\root-app.yaml
```

The root app watches `apps/` with `prune: true` and `selfHeal: true` — deleting a
file from git deletes the resource from the cluster, and manual `kubectl edit`
changes get reverted.

---

## Part 3 — Secrets

**Create these before the first sync**, or cert-manager and Paperless will crash-loop.
See `docs/secrets.md`.

```powershell
# Cloudflare token for cert-manager DNS-01 (Zone → DNS → Edit on damians.cloud)
kubectl create secret generic cloudflare-api-token `
  -n cert-manager `
  --from-literal=api-token="<TOKEN>"

# Paperless
kubectl create secret generic paperless-secret `
  -n media `
  --from-literal=PAPERLESS_SECRET_KEY="<RANDOM>"
```

Newer secrets (e.g. Karakeep) aren't created manually anymore — they're encrypted
with [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) and committed
as ciphertext. See `docs/secrets.md`.

The `cert-manager` and `media` namespaces come from `infrastructure/namespaces/`,
so let the namespaces sync first, then create the secrets, then let the rest sync.

---

## Part 4 — Sync order

Namespaces carry the Pod Security labels Talos requires. Talos enforces the
`baseline` standard by default; MetalLB speakers (`NET_RAW`, hostNetwork),
Longhorn (privileged containers) and the Intel device plugins need
`pod-security.kubernetes.io/enforce: privileged`. This is declared in
`infrastructure/namespaces/namespaces.yaml` rather than patched by hand later.

> Symptom if this is missing: DaemonSets stuck at `DESIRED 3 / CURRENT 0` with
> `FailedCreate ... violates PodSecurity`.

Sync waves handle operator → config ordering:

| Wave | Resources |
|------|-----------|
| 0 (default) | namespaces, MetalLB, Longhorn, Traefik, cert-manager, CNPG operator, Intel plugins operator (wave 1) |
| 1 | `intel-device-plugins-operator` |
| 2 | `cert-manager-config` (ClusterIssuer), `intel-device-plugins-gpu`, `paperless-db` (CNPG cluster) |

CRD-owning operators must be healthy before the resources that use their CRDs.

---

## Adding a new app

1. Put Helm values or raw manifests in `apps-manifests/<category>/<app>/`
2. Add a flat `apps/<app>.yaml` Application pointing at that path and namespace
3. Commit and push
4. Watch it sync in the UI

The Application files are ~90% copy-paste; only `name`, `source.path` and
`destination.namespace` change. Pin every chart version — Renovate opens PRs for
updates.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `talosctl get extensions` empty | ISO still booting instead of the installed disk, or the wrong schematic. Re-run `talosctl upgrade` with the installer image. |
| `dial tcp :443: connection refused` | `:6443` was omitted in `talosctl gen config`. Regenerate. |
| DaemonSet `DESIRED 3 / CURRENT 0`, `violates PodSecurity` | Missing `pod-security.kubernetes.io/enforce: privileged` on the namespace. |
| NFS mount hangs | Missing `nfsvers=3,nolock`. Talos has no rpc.statd. |
| cert-manager DNS-01 never validates | Split-horizon DNS returns the internal record, so the self-check fails. Set `dns01RecursiveNameservers: "1.1.1.1:53,8.8.8.8:53"` with `dns01RecursiveNameserversOnly: true` in the cert-manager Helm values. |
| PVC won't delete | ArgoCD auto-sync recreates it. Pause auto-sync on the app, delete, resume. |
| `wsarecv: connection forcibly closed` on GitHub/ghcr downloads | Something inspects the traffic — typically UniFi IDS/IPS or AV web filtering. Workaround: download in a browser and apply locally. Fix properly, or it will resurface as `ImagePullBackOff` inside the cluster. |
| Longhorn volumes won't attach | `iscsi-tools` / `util-linux-tools` extension missing. |

---

## Known gaps / TODO

- [ ] **No backups.** Longhorn has no backup target configured (the NAS is the
      obvious candidate) and the CNPG `paperless-db` has no backup section.
      The cluster structure is reproducible; the data is not.
- [ ] **Dashy config is not GitOps.** `apps-manifests/tools/dashy/data/conf.yaml`
      sits in the repo but is never referenced — the deployment mounts a Longhorn
      PVC at `/app/user-data`. The real dashboard config only exists in-cluster
      and would be lost on rebuild. Mount it as a ConfigMap or delete the file so
      it stops pretending to be documentation.
- [ ] Document the NAS export configuration (permissions, squash settings).