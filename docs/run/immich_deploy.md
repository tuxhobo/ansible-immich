# Immich Deployment Runbook

**Version:** 0.2  
**Environment:** Proxmox / TrueNAS SCALE / Docker (Debian Trixie)  
**Last Updated:** 2026-05-26

---

## 1. Purpose

Step-by-step procedure to deploy Immich photo management on the home lab Docker host.
Covers infrastructure preparation, container deployment, and initial user configuration.
Designed to be repeatable and verifiable at each step.

Errors are not expected at any step. Any deviation is a stop condition.

---

## 2. Prerequisites

### 2.1 Infrastructure

| Component | Host | Status Required |
|---|---|---|
| Proxmox hypervisor | Dell R530 | Running |
| TrueNAS SCALE | lala110 (10.0.0.110) | Running, API accessible |
| Docker VM | docker01 (10.0.0.115) | Running, Docker daemon healthy |
| ZFS pool `atl` | lala110 | Mounted, POSIX ACLs confirmed |
| Docker data-root | docker01 | Migrated to `/home/docker-data` |

### 2.2 Ansible Control Node

- `configure_ansible.md` completed in full
- SSH key `~/.ssh/ansible_pbs_key` present on control node
- Ansible Vault password file `~/.ansible/.vault_pass_immich` in place
- Repo checked out at `~/home-lab/ansible-immich`
- `community.docker` collection installed (required for `deploy_immich` roles)

Verify collections:

```bash
ansible-galaxy collection list | grep -E "community\.docker|ansible\.posix"
```

Install if missing:

```bash
ansible-galaxy collection install community.docker
```

### 2.3 Vault Secrets Required

All secrets must be populated before running any playbook. No `CHANGEME` values may remain.

| Vault File | Key | How to Generate |
|---|---|---|
| `group_vars/all/vault.yml` | `vault_truenas_lala110_api_key` | TrueNAS UI → API Keys |
| `group_vars/docker_hosts/vault.yml` | `vault_immich_db_password` | `openssl rand -hex 32` |
| `group_vars/docker_hosts/vault.yml` | `vault_immich_jwt_secret` | `openssl rand -hex 32` |

Check for unset values:

```bash
ansible-vault view inventories/group_vars/all/vault.yml | grep CHANGEME
ansible-vault view inventories/group_vars/docker_hosts/vault.yml | grep CHANGEME
# Expected: empty output
```

---

## 3. Manual Pre-Flight: Provision Ansible User on docker01

This is a one-time manual step. The `ansible` user does not exist on docker01 yet.
All commands run as `admin` over SSH.

**On the control node**, get the Ansible public key:

```bash
cat ~/.ssh/ansible_pbs_key.pub
```

Copy the full output. Then:

```bash
ssh admin@10.0.0.115
```

**On docker01**, run as `admin`:

```bash
sudo useradd -m -s /bin/bash ansible
sudo mkdir -p /home/ansible/.ssh
sudo chmod 700 /home/ansible/.ssh
echo 'PASTE_PUBLIC_KEY_HERE' | sudo tee /home/ansible/.ssh/authorized_keys
sudo chmod 600 /home/ansible/.ssh/authorized_keys
sudo chown -R ansible:ansible /home/ansible/.ssh
sudo usermod -aG sudo ansible
sudo usermod -aG docker ansible
echo 'ansible ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/ansible
sudo chmod 440 /etc/sudoers.d/ansible
```

Verify from the control node:

```bash
ssh -i ~/.ssh/ansible_pbs_key ansible@10.0.0.115 echo OK
# Expected: OK

ssh -i ~/.ssh/ansible_pbs_key ansible@10.0.0.115 docker info | grep "Docker Root Dir"
# Expected: Docker Root Dir: /home/docker-data
```

If Docker Root Dir is not `/home/docker-data`, stop here. The data-root migration must be completed first.

---

## 4. Inventory

The inventory describes two host groups with different connection methods.

```
all
├── truenas
│   └── lala110 (10.0.0.110)   ansible_connection=local  [API-only, uri module]
└── docker_hosts
    └── docker01 (10.0.0.115)  ansible_connection=ssh    [ansible user, key auth]
```

`lala110` uses `ansible_connection: local` because all TrueNAS tasks communicate
via its REST API from the control node. No SSH session is opened to TrueNAS.

`docker01` uses standard SSH with key-based auth. The `ansible` user requires
passwordless sudo, as provisioned in Section 3.

Validate the inventory resolves correctly before running any playbook:

```bash
ansible-inventory --graph
ansible all -m ping
# Expected: lala110 | SUCCESS, docker01 | SUCCESS
```

---

## 5. Playbook 1: `deploy_immich_data`

**Purpose:** Configure TrueNAS storage — creates the `immich` user/group, ZFS datasets,
and NFS exports. Must complete successfully before `deploy_immich` runs.  
**Idempotent:** Yes — safe to re-run.  
**Host group:** `truenas`

### 5.1 Pre-flight

```bash
# TrueNAS API reachable
curl -sk https://10.0.0.110/api/v2.0/system/info \
  -H "Authorization: Bearer $(ansible-vault view inventories/group_vars/all/vault.yml \
  | grep api_key | awk '{print $2}' | tr -d '"')" \
  | python3 -m json.tool | grep hostname
# Expected: "lala110"

# ZFS pool online
ssh admin@10.0.0.110 "zpool status atl | grep state"
# Expected: state: ONLINE

# UID 9123 is free on TrueNAS (no leftover user from previous attempts)
ssh admin@10.0.0.110 "getent passwd 9123 || echo free"
# Expected: free
```

### 5.2 Run: `truenas_user`

Creates the `immich` group (GID 9123) and user (UID 9123) on TrueNAS.

```bash
ansible-playbook playbooks/deploy_immich_data.yml --tags truenas_user
```

Verify:

```bash
ssh admin@10.0.0.110 "getent passwd immich && getent group immich"
# Expected: immich entries with UID/GID 9123
```

### 5.3 Run: `truenas_dataset`

Creates ZFS datasets and sets ownership to `immich:immich`.

```bash
ansible-playbook playbooks/deploy_immich_data.yml --tags truenas_dataset
```

Verify:

```bash
ssh admin@10.0.0.110 "zfs list | grep immich"
# Expected:
# atl/immich        ...  /mnt/atl/immich
# atl/immich/ted    ...  /mnt/atl/immich/ted
# atl/immich/sally  ...  /mnt/atl/immich/sally

ssh admin@10.0.0.110 "stat /mnt/atl/immich/ted | grep Uid"
# Expected: Uid: ( 9123/ immich)   Gid: ( 9123/ immich)
```

### 5.4 Run: `truenas_nfs_share`

Creates NFS exports on TrueNAS and reloads the NFS service.

Exports created:
- `/mnt/atl/immich/ted` — rw, client `10.0.0.115`
- `/mnt/atl/immich/sally` — rw, client `10.0.0.115`
- `/mnt/atl/images` — ro, client `10.0.0.115`

```bash
ansible-playbook playbooks/deploy_immich_data.yml --tags truenas_nfs_share
```

Verify from the Docker host:

```bash
ssh admin@10.0.0.115 "showmount -e 10.0.0.110"
# Expected: all three paths listed
```

---

## 6. Playbook 2: `deploy_immich`

**Purpose:** Deploy the Immich Docker Compose stack on docker01.  
**Idempotent:** Yes — safe to re-run.  
**Host group:** `docker_hosts`

Roles run in order:

| Role | What it does |
|---|---|
| `nfs_client` | Installs `nfs-common`, mounts NFS shares, writes fstab |
| `linux_user` | Creates `immich` user/group on docker01 (UID/GID 9123) |
| `immich_config` | Creates directory layout, templates `.env` and `docker-compose.yml`, starts stack |
| `immich_smoke_test` | Post-deploy assertions — fails fast on any problem |

### 6.1 Pre-flight

```bash
# Ansible user SSH access confirmed
ssh -i ~/.ssh/ansible_pbs_key ansible@10.0.0.115 echo OK
# Expected: OK

# Docker daemon healthy
ssh -i ~/.ssh/ansible_pbs_key ansible@10.0.0.115 "docker info > /dev/null && echo OK"
# Expected: OK

# NFS exports visible from docker01 (prerequisite: Section 5.4 complete)
ssh -i ~/.ssh/ansible_pbs_key ansible@10.0.0.115 "showmount -e 10.0.0.110"
# Expected: /mnt/atl/immich/ted, /mnt/atl/immich/sally, /mnt/atl/images listed
```

### 6.2 Run Playbook

```bash
ansible-playbook playbooks/deploy_immich.yml
```

To run individual roles:

```bash
ansible-playbook playbooks/deploy_immich.yml --tags nfs_client
ansible-playbook playbooks/deploy_immich.yml --tags linux_user
ansible-playbook playbooks/deploy_immich.yml --tags immich_config
ansible-playbook playbooks/deploy_immich.yml --tags smoke_test
```

### 6.3 Directory Layout Created on docker01

```
/mnt/nfs/
  images/              ← lala110:/mnt/atl/images        (ro)
  immich/
    ted/               ← lala110:/mnt/atl/immich/ted     (rw)
    sally/             ← lala110:/mnt/atl/immich/sally   (rw)

/home/docker-data/compose/immich/
  docker-compose.yml
  .env                 (secrets — mode 0640, not in git)
  data/
    db/                (Postgres data — local disk, owned 999:999)
    model-cache/       (ML model cache — owned immich:immich)
```

### 6.4 Smoke Test Results

The `immich_smoke_test` role runs automatically at the end of the playbook.
It asserts:

- All four containers (`immich_server`, `immich_machine_learning`, `immich_postgres`, `immich_redis`) are in `running` state
- Postgres is accepting connections (`pg_isready`)
- Immich HTTP endpoint returns 200 or 302
- `/mnt/nfs/immich/ted` accepts a write
- `/mnt/nfs/images` rejects a write (read-only confirmed)

Any failure here is a stop condition. Do not proceed to Section 7 until smoke tests pass.

---

## 7. Manual Steps: Initial Immich Configuration

These steps are not automated. Execute after the playbook and smoke tests succeed.

### 7.1 First Admin Setup

1. Open browser → `http://10.0.0.115:2283`
2. Complete the onboarding wizard
3. Create the admin account using `ted`'s email from `inventories/group_vars/all/users.yml`

### 7.2 Create Users

Administration → Users → Create User:

| Username | Email | Role |
|---|---|---|
| ted | ted@lalalandus.com | Admin |
| sally | sally@lalalandus.com | User |
| btian | brian@lalalandus.com | User |
| immich-archive | *(service account — choose an internal address)* | User |

### 7.3 Configure External Library (Archive)

1. Log in as `immich-archive`
2. Administration → External Libraries → Create
3. Path: `/usr/src/app/external-library` (container-side path — maps to `/mnt/nfs/images` on host)
4. Run initial scan — first run against the archive will take several hours; elevated CPU is expected
5. Enable partner sharing → share with ted, sally, btian

### 7.4 Verify User Upload Paths

Upload one photo from the iOS app as `ted`. Confirm:

- Photo appears in the Immich UI
- File is present at `/mnt/nfs/immich/ted/<date>/<filename>` on docker01

Repeat for `sally`.

### 7.5 Pin Immich Version

Once the deployment is confirmed stable, pin the image tag. In `inventories/group_vars/docker_hosts/docker_hosts.yml`:

```yaml
immich_version: "v1.132.3"   # replace with current stable tag
```

Re-run the playbook to apply:

```bash
ansible-playbook playbooks/deploy_immich.yml --tags immich_config
```

Commit the version change to the repo. Do not leave `release` in production.

---

## 8. Rollback

```bash
# Stop and remove the stack — NFS data is safe, local DB data is removed
ssh admin@10.0.0.115 \
  "docker compose -f /home/docker-data/compose/immich/docker-compose.yml down"

# Remove compose project directory (includes Postgres data volume)
ssh admin@10.0.0.115 "sudo rm -rf /home/docker-data/compose/immich"

# Unmount NFS shares if needed
ssh admin@10.0.0.115 "sudo umount /mnt/nfs/images /mnt/nfs/immich/ted /mnt/nfs/immich/sally"
```

TrueNAS datasets and NFS exports are not removed automatically.
Manual deletion via the TrueNAS UI is required if a full teardown is needed.

---

## 9. Deferred Items

| Item | Notes |
|---|---|
| SSL/TLS | Reverse proxy with valid cert — not yet in scope |
| Brian external library | `/mnt/atl/home/brian` is `0700` — permissions must be resolved first |
| `pg_dump` backup | Integrate into ZFS replication chain |
| lldap / OIDC integration | Phase 2 |
| Gotify notifications | No native Immich support — requires middleware |
| Graylog logging | Configure GELF log driver post-deploy |
| Brian Android app | Revisit after Phase 1 stable |
| Docker VM flat partition layout | Current split-partition layout is debt — redeploy VM with single root partition managed via Proxmox disk extension |

---

## 10. Known Conditions

- TrueNAS uses a self-signed TLS certificate. All API calls use `validate_certs: false`. This is intentional and documented in `inventories/group_vars/truenas/truenas.yml`.
- `/mnt/atl/images` is owned by `ted:ted (UID 3000)` with mode `0775`. The world-readable bit allows the `immich` container (UID 9123) read access without an ownership change. This is intentional.
- Immich machine learning performs one-time indexing on the first library scan. Expect several hours of elevated CPU against the archive library on first run.
- The `.env` file contains secrets and must never be committed to the repo. It is managed exclusively by the `immich_config` role.
