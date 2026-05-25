# Immich Deployment Runbook

**Version:** 0.1-draft  
**Environment:** Proxmox / TrueNAS SCALE / Docker (Debian Trixie)  
**Last Updated:** 2026-05-23

---

## 1. Purpose

Step-by-step procedure to deploy Immich photo management on the home lab Docker host. Covers infrastructure preparation, container deployment, and initial user configuration. Designed to be repeatable and verifiable at each step.

---

## 2. Prerequisites

### 2.1 Infrastructure

| Component | Host | Status Required |
|---|---|---|
| Proxmox hypervisor | Dell R530 | Running |
| TrueNAS SCALE | lala110 (10.0.0.110) | Running, API accessible |
| Docker VM | docker01 (10.0.0.115) | Running, Docker daemon healthy |
| ZFS pool `atl` | lala110 | Mounted, POSIX ACLs confirmed |
| Docker data-root migration | docker01 | **Must be complete** |

### 2.2 Ansible Control Node

- Ansible 2.14+ installed
- SSH key access to `docker01` confirmed
- Ansible Vault password available
- This runbook's repo checked out

### 2.3 Accounts & Credentials

- TrueNAS ansible user password disabled, home directory /var/empty
- TrueNAS ansible group with LOCAL_ADMINISTRATOR Privillate
- TrueNAS API key generated for ansible user and stored in vault
- Ansible user has sudo on docker host, no password, ssh key file only
```bash
  sudo useradd -m -s /bin/bash ansible &&
  sudo mkdir -p /home/ansible/.ssh &&
  sudo chmod 700 /home/ansible/.ssh &&
  sudo echo 'PASTE_ANSIBLE_PUBLIC_KEY_HERE' > /home/ansible/.ssh/authorized_keys &&
  sudo chmod 600 /home/ansible/.ssh/authorized_keys &&
  sudo chown -R ansible:ansible /home/ansible/.ssh &&
  sudo usermod -aG sudo ansible &&
  sudo usermod -aG docker ansible &&
  sudo mkdir -p /etc/sudoers.d &&
  sudo echo 'ansible ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/ansible &&
  sudo chmod 440 /etc/sudoers.d/ansible
```
- TrueNAS `dockeruser` deleted (UID 9123 confirmed free)

### 2.4 Verify Docker Data-Root Migration Complete and ssh access for ansible user

Before proceeding, confirm Docker is running from the correct data root:

```bash
ssh -i ~/.ssh/ansible_pbs_key ansible@10.0.0.115 docker info | grep "Docker Root Dir"
# Expected: /home/docker-data
```

If not correct, **stop here** and complete the data-root migration first.


### 3 Verify Ansible Connectivity

```bash
# From control node
ansible all -m ping
# Expected: lala110 | SUCCESS, docker | SUCCESS

ansible-vault view inventory/group_vars/truenas/vault.yml
# Expected: vault_truenas_lala110_api_key
```

---

## 5. Playbook 1: `deploy_immich_data`

**Purpose:** Prepare storage infrastructure on TrueNAS and mount NFS shares on the Docker host.  
**Idempotent:** Yes — safe to re-run.  
**Tags:** `datasets`, `nfs_shares`, `nfs_mounts`

### 5.1 Pre-flight Checks

```bash
# Verify TrueNAS API is reachable
curl -sk -H "Authorization: Bearer $(ansible-vault view inventories/group_vars/all/vault.yml | grep api_key | awk '{print $2}' | tr -d '"')" \
  https://10.0.0.110/api/v2.0/system/info | python3 -m json.tool | grep hostname
# Expected: "lala110"

# Verify pool is available
ssh admin@10.0.0.110 "sudo -S zpool status atl | grep state"
# Expected: state: ONLINE
```

### 5.2 Run Tag: `datasets`

Creates:
- `atl/immich` parent dataset
- `atl/immich/ted`
- `atl/immich/sally`
- `immich` group (GID 9123) on TrueNAS
- `immich` user (UID 9123) on TrueNAS

```bash
ansible-playbook playbooks/deploy_immich_data.yml --tags datasets
```

**Verify:**

```bash
ssh admin@10.0.0.110 "zfs list | grep immich"
# Expected:
# atl/immich        ...  /mnt/atl/immich
# atl/immich/ted    ...  /mnt/atl/immich/ted
# atl/immich/sally  ...  /mnt/atl/immich/sally

ssh admin@10.0.0.110 "stat /mnt/atl/immich/ted"
# Expected: Uid: (9123/immich) Gid: (9123/immich)

ssh admin@10.0.0.110 "getent passwd immich && getent group immich"
# Expected: immich entries with UID/GID 9123
```

### 5.3 Run Tag: `nfs_shares`

Creates NFS exports on TrueNAS:
- `/mnt/atl/images` — read-only, client `10.0.0.115`
- `/mnt/atl/immich` — read-write, client `10.0.0.115`

```bash
ansible-playbook playbooks/deploy_immich_data.yml --tags nfs_shares
```

**Verify:**

```bash
ssh admin@10.0.0.110 "cat /etc/exports | grep immich"
# Expected: entries for /mnt/atl/images and /mnt/atl/immich

# From Docker host — requires nfs-common
ssh george@10.0.0.115 "showmount -e 10.0.0.110"
# Expected: /mnt/atl/images and /mnt/atl/immich listed
```

### 5.4 Run Tag: `nfs_mounts`

Mounts NFS shares on Docker host:
- `/mnt/nfs/images` ← `lala110:/mnt/atl/images` (ro)
- `/mnt/nfs/immich` ← `lala110:/mnt/atl/immich` (rw)
- Adds entries to `/etc/fstab` for persistence

```bash
ansible-playbook playbooks/deploy_immich_data.yml --tags nfs_mounts
```

**Verify:**

```bash
ssh george@10.0.0.115 "mount | grep nfs"
# Expected: both mounts listed as type nfs4

ssh george@10.0.0.115 "ls /mnt/nfs/images | head -5"
# Expected: existing image folders visible

ssh george@10.0.0.115 "touch /mnt/nfs/immich/test_write && rm /mnt/nfs/immich/test_write"
# Expected: no permission error

ssh george@10.0.0.115 "touch /mnt/nfs/images/test_write"
# Expected: permission denied (read-only mount confirmed)
```

---

## 6. Playbook 2: `deploy_immich`

**Purpose:** Deploy Immich Docker Compose stack on docker01.  
**Idempotent:** Yes.  
**Tags:** None — runs fully or not at all.

### 6.1 Pre-flight Checks

```bash
# NFS mounts must be present
ssh george@10.0.0.115 "mountpoint /mnt/nfs/images && mountpoint /mnt/nfs/immich"
# Expected: both are mountpoints

# Docker running
ssh george@10.0.0.115 "docker info > /dev/null && echo OK"
# Expected: OK

# Compose directory does not already exist (first deploy)
ssh george@10.0.0.115 "ls /home/docker-data/compose/immich 2>/dev/null || echo clear"
# Expected: clear
```

### 6.2 Run Playbook

Creates `immich` user/group on Docker host, deploys Compose file, starts stack.

```bash
ansible-playbook playbooks/deploy_immich.yml
```

**Verify stack is up:**

```bash
ssh george@10.0.0.115 "docker compose -f /home/docker-data/compose/immich/docker-compose.yml ps"
# Expected: immich-server, immich-machine-learning, postgres — all running

ssh george@10.0.0.115 "docker logs immich_immich-server_1 2>&1 | tail -20"
# Expected: no fatal errors, listening on port 2283
```

**Verify web UI reachable:**

```bash
curl -s -o /dev/null -w "%{http_code}" http://10.0.0.115:2283
# Expected: 200 or 302
```

---

## 7. Manual Steps: Initial Immich Configuration

*These steps are not automated. Execute after both playbooks succeed.*

### 7.1 First Admin Setup

1. Open browser → `http://10.0.0.115:2283`
2. Complete the onboarding wizard
3. Create admin account for `ted` (use email from `inventory/users.yml`)

### 7.2 Create Users

In Immich UI → Administration → Users → Create User:

| Username | Email | Role |
|---|---|---|
| ted | ted@yourdomain.com | Admin |
| sally | sally@yourdomain.com | User |
| btian | brian@yourdomain.com | User |
| immich-archive | *(service account email)* | User |

### 7.3 Configure External Library (Archive)

1. Log in as `immich-archive`
2. Administration → External Libraries → Create
3. Path: `/mnt/nfs/images` (as seen inside container)
4. Run initial scan — expect long first run
5. Enable partner sharing → share with ted, sally, btian

### 7.4 Configure User Libraries

For `ted` and `sally`: Immich manages upload paths automatically. Confirm uploads land in `/mnt/nfs/immich/<username>` by doing a test upload from the iOS app.

### 7.5 Pin Immich Version

After confirming deployment is stable, pin the image tag in `docker-compose.yml`:

```yaml
image: ghcr.io/immich-app/immich-server:v1.132.3  # replace with current stable
```

Commit to repo. Do not leave on `release` tag in production.

---

## 8. Final Smoke Test

Run this sequence end-to-end after all steps complete:

```bash
# 1. Web UI responds
curl -s -o /dev/null -w "%{http_code}" http://10.0.0.115:2283
# Expected: 200

# 2. All containers healthy
ssh george@10.0.0.115 "docker ps --filter name=immich --format 'table {{.Names}}\t{{.Status}}'"
# Expected: all Up, no Restarting

# 3. Postgres accessible
ssh george@10.0.0.115 "docker exec immich_postgres_1 pg_isready"
# Expected: accepting connections

# 4. Archive library visible
# Log into UI as ted → verify archive photos appear via partner share

# 5. iOS upload test
# Upload one photo from iOS app as ted → confirm appears in UI
# Confirm file lands at /mnt/nfs/immich/ted/<date>/filename

# 6. NFS persistence after reboot (optional but recommended)
ssh george@10.0.0.115 "sudo reboot"
# After reboot:
ssh george@10.0.0.115 "mount | grep nfs && docker ps | grep immich"
# Expected: mounts and containers present
```

---

## 9. Deferred Items

| Item | Notes |
|---|---|
| Brian external library | `/mnt/atl/home/brian` is `0700` — permissions must be resolved first |
| lldap / OIDC integration | Phase 2 |
| Gotify notifications | No native support — requires middleware |
| Graylog logging | Configure GELF log driver post-deploy |
| `pg_dump` backup | Integrate into ZFS replication chain |
| Immich version pinning | Do immediately after first stable deploy |
| Brian Android app | Revisit after Phase 1 stable |

---

## 10. Rollback

```bash
# Stop and remove stack (data on NFS is safe)
ssh george@10.0.0.115 "docker compose -f /home/docker-data/compose/immich/docker-compose.yml down"

# Remove compose directory
ssh george@10.0.0.115 "sudo rm -rf /home/docker-data/compose/immich"

# Unmount NFS (if needed)
ssh george@10.0.0.115 "sudo umount /mnt/nfs/images /mnt/nfs/immich"

# TrueNAS datasets are NOT deleted automatically — manual action required if needed
```

---

## 11. Known Conditions

- TrueNAS uses a self-signed TLS certificate. All API calls use `validate_certs: false`. This is expected and documented.
- `/mnt/atl/images` is owned by `ted:ted (3000)` with `0775`. World-readable bit allows `immich` (9123) read access without ownership change. This is intentional.
- Immich machine learning container performs one-time indexing on first library scan. Expect elevated CPU for several hours on first run against the archive library.
