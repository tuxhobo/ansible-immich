# ansible-immich

Ansible automation to deploy [Immich](https://immich.app/) on a self-hosted Docker host backed by TrueNAS SCALE NFS storage.

---

## 1. Problem Statement

A home lab photo archive spanning many years lives on a TrueNAS NAS. There is no self-hosted solution for browsing, searching, or sharing that archive — or for ingesting new photos from mobile devices. Cloud photo services are not an option: the archive stays on-prem.

The goal is a fully self-hosted photo management platform with:

- Mobile upload from iOS for active users
- ML-powered search and face recognition
- Read-only access to the existing archive without moving or duplicating files
- Repeatable, auditable infrastructure-as-code deployment

---

## 2. Design Goals

- **No manual snowflaking.** Every infrastructure decision is expressed in Ansible. Running the playbooks on a clean host produces a working deployment.
- **NFS UID/GID alignment.** The `immich` service account (UID/GID 9123) is created identically on TrueNAS and the Docker host. NFS ownership resolution works without `no_root_squash` or `anonuid` hacks.
- **Immich does not own the archive.** The historical photo archive is mounted read-only as an external library. Immich indexes it in place; no files are moved or copied.
- **Postgres stays local.** The database runs on VM local disk. NFS does not provide the filesystem semantics Postgres requires.
- **Idempotent playbooks.** All roles are safe to re-run. Re-running against an already-deployed environment is a no-op.
- **Secrets never in plaintext.** All credentials live in Ansible Vault. No secret touches the repo.
- **Incremental scope.** Phase 1 is a working Immich deployment. SSO, OIDC, and reverse proxy are explicitly deferred to Phase 2.

---

## 3. Homelab Server Configuration

| Component | Host | Address | Notes |
|---|---|---|---|
| Proxmox hypervisor | Dell PowerEdge R530 | 10.0.0.100 | Hosts Docker VM |
| TrueNAS SCALE | ZFS pool `atl`, POSIX ACLs |
| Docker VM | Debian Trixie, 8 CPU, 8 GB RAM, 100 GB SSD-backed |
| Ansible control node | WSL2 / Ubuntu 22.04 —  Runs on Windows 11 workstation |

**Docker VM storage layout:**

**Volumes** for docker containers are allocated from a 100G virtual drive thin provisioned from the PVE host SSD.


**Networking:** Tailscale (WireGuard) provides remote access. Local DNS is not yet clean enough to use FQDNs — services are accessed by IP in Phase 1. A reverse proxy with internal TLS is planned as a separate project once DNS is sorted.

---

## 4. Architecture

```
Ansible Control Node (WSL2)
│
├── TrueNAS REST API (api/v2.0)
│   ├── immich group/user  (GID/UID 9123)
│   ├── ZFS dataset:  atl/immich
│   └── NFS exports:
│         /mnt/atl/immich   →  rw
│         /mnt/atl/images   →  ro
└── Docker Host  (SSH)
    ├── immich group/user  (GID/UID 9123) — mirrors TrueNAS
    ├── NFS mounts:
    │     /mnt/nfs/immich/uploads ← atl/immich       (rw)
    │     /mnt/nfs/images         ← atl/images        (ro)
    └── Docker Compose stack (immich)
          immich-server          :2283
          immich-machine-learning
          immich-postgres        (local disk /home/docker-data/compose/immich/data/db)
          immich-redis
```

**Volume mounts inside the immich-server container:**

| Host path | Container path | Access | Purpose |
|---|---|---|---|
| `/mnt/nfs/immich/uploads` | `/usr/src/app/upload` | rw | All mobile uploads |
| `/mnt/nfs/images` | `/usr/src/app/external-library/archive` | ro | Historical archive |

---

## 5. Assumptions & Tradeoffs

**UID/GID 9123 is fixed.** All NFS permission alignment depends on this value matching between TrueNAS and the Docker host. Changing it after deployment requires a chown pass on all NFS data.

**TrueNAS self-signed cert.** All API calls use `validate_certs: false`. This is intentional — the TrueNAS cert is not in the trust store and will not be until the internal PKI project completes.

**No TLS in Phase 1.** Immich is exposed on HTTP at port 2283. Tailscale handles transit encryption for remote access. Local LAN traffic is unencrypted. A Caddy reverse proxy with `tls internal` is planned once DNS is clean.

**Postgres on local disk.** Postgres data lives at `/home/docker-data/compose/immich/data/db`. This is not on NFS — NFS does not provide the fsync and locking semantics Postgres requires. Local disk means Postgres data does not survive a VM rebuild without a restore from backup (see Section 8).

**The archive is read-only and never duplicated.** `/mnt/atl/images`. The world-readable bit gives the `immich` container (UID 9123) read access without an ownership change. This is intentional and documented.

**GPU acceleration not deployed.** The ingest volume does not justify the complexity. The machine-learning container runs on CPU.

**Storage template set to `{{y}}/{{MM}}/{{filename}}`.** This applies to new mobile uploads only. The archive retains its existing folder structure on TrueNAS.

---

## 6. Datasets and NFS

### TrueNAS ZFS Datasets

| Dataset | Path | Owner | Mode | Purpose |
|---|---|---|---|---|
| `atl/immich` | `/mnt/atl/immich` | immich:immich (9123) | 0750 | All Immich-managed uploads |
| `atl/images` | `/mnt/atl/images` | user:user (3000) | 0775 | Historical archive (pre-existing) |

ZFS pool `atl` is configured with `acltype: posix` and `aclmode: discard`. No NFSv4 ACLs are used.

### NFS Exports

| Export path | Client | Access | Mount point on Docker host |
|---|---|---|---|
| `/mnt/atl/immich` | rw | `/mnt/nfs/immich/uploads` |
| `/mnt/atl/images`  | ro | `/mnt/nfs/images` |

Mount options: `nfsvers=4,hard,intr`. Mounts are persisted in `/etc/fstab` by the `nfs_client` role.

---

## 7. Runbooks

Runbooks are in the repo root. Follow them in order.

### [`configure_ansible.md`](configure_ansible.md)

One-time control node setup. Covers WSL2, Ansible installation, project checkout, collections, Vault, and SSH key generation. Run this first on any new control node.

### [`immich_deploy.md`](immich_deploy.md)

Full deployment procedure. Two playbooks run in sequence:

**Playbook 1 — `deploy_immich_data.yml`** (targets TrueNAS via REST API)

| Tag | What it does |
|---|---|
| `truenas_user` | Creates `immich` group and user (UID/GID 9123) on TrueNAS |
| `truenas_dataset` | Creates `atl/immich` ZFS dataset, sets ownership |
| `truenas_nfs_share` | Creates NFS exports, reloads NFS service |

**Playbook 2 — `deploy_immich.yml`** (targets Docker host via SSH)

| Tag | What it does |
|---|---|
| `apt_packages` | Installs `nfs-common` and any other required packages |
| `nfs_client` | Creates mountpoints, verifies exports, mounts NFS shares, writes fstab |
| `linux_user` | Creates `immich` user/group (UID/GID 9123) on Docker host |
| `immich_config` | Templates `.env` and `docker-compose.yml`, runs `compose up` |
| `smoke_test` | Asserts all containers running, Postgres healthy, HTTP reachable, NFS writes/reads correct |

Both playbooks are idempotent and safe to re-run.

**Upgrades:** Re-run `immich_config` after changing `immich_version` in `docker_hosts.yml`:

```bash
ansible-playbook playbooks/deploy_immich.yml --tags immich_config,smoke_test
```

---

## 8. Backup & Restore Strategy

### What needs backing up

| Data | Location | Backup method |
|---|---|---|
| Photo uploads (NFS) | `/mnt/atl/immich` on TrueNAS | ZFS replication (existing pipeline) |
| Historical archive (NFS) | `/mnt/atl/images` on TrueNAS | ZFS replication (existing pipeline) |
| Postgres database | `/home/docker-data/compose/immich/data/db` on Docker VM | Immich schedule pg_dump to NFS store |

### Postgres backup (deferred — Phase 1 debt)

The database is on VM local disk and is not covered by ZFS replication. Immiich schedules a pg_dump to the backup folder in the immich NFS mount.
The immich restore function reads NFS backup folder for recovery.

### Restore procedure (photo data intact, DB lost)

1. Rebuild Docker VM and re-run both playbooks.
2. Complete first-admin setup and re-create user accounts (see `immich_deploy.md` Section 7).
3. Re-create external libraries and trigger library scans.
4. Mobile upload history is lost (not recoverable without DB). Files on NFS are safe.
