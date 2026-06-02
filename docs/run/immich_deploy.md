# Immich Deployment Runbook

**Version:** 1.0 (post deployment)  
**Environment:** Proxmox / TrueNAS SCALE / Docker (Debian Trixie)  
**Last Updated:** 2026-05-30

---

## 1. Purpose

Step-by-step procedure to deploy Immich photo management on the home lab Docker host.
Covers infrastructure preparation, container deployment, and initial user configuration. Designed to be repeatable and verifiable at each step.

Errors are not expected at any step. Any deviation is a stop condition.

---

## 2. Prerequisites

### 2.1 Infrastructure

| Component | Host | Status Required |
|---|---|---|
| Proxmox hypervisor | lala100 (10.0.0.100) | Running |
| TrueNAS SCALE | lala110 (10.0.0.110) | Running, API accessible |
| Docker VM | docker (10.0.0.115) | Running, Docker daemon healthy |
| ZFS pool `atl` | lala110 | Mounted, POSIX ACLs confirmed |

Overcome initial deployment problem: root of docker host is also the docker-root. 
The root file system is too small and is near max capacity. The short term solution
moves the docker data-root folder to the data partition in the host file system.
The next test insures that interum step has been taken. Long term, the docker host vm shall have only one partition. Resizing VM disk size is a simple process with Proxmox 
which eliminates the need for a separate root partition. 
| Docker data-root | docker | Migrated to `/home/docker-data` |

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

### 2.3 Accounts & Credentials

- TrueNAS ansible user password disabled, home directory /var/empty
- TrueNAS ansible group with LOCAL_ADMINISTRATOR Privillate
- TrueNAS API key generated for ansible user and stored in vault

- Docker host create `ansible` user.
  **On the ansible control node**, get the Ansible public key:
```bash
cat ~/.ssh/ansible_pbs_key.pub
```
Copy the key string and replace "PASTE_ANSIBLE_PUBLIC_KEY_HERE" in the command below.

- Ansible user has sudo on docker host, no password, ssh key file onlyAll commands run SSH to the docker host and run the following:
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

### 2.4 Vault Secrets Required

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
### 2.5 Verify Ansible Connectivity

```bash
# From control node
ansible all -m ping
# Expected: lala110 | SUCCESS, docker | SUCCESS

ansible-vault view inventory/group_vars/truenas/vault.yml
# Expected: vault_truenas_lala110_api_key
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

Verify connectivity from the ansible control node:

```bash
ssh -i ~/.ssh/ansible_pbs_key ansible@10.0.0.115 echo OK
# Expected: OK
```

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
**Host group:** `truenas_user', 'truenas_dataset", 'truenas_nfs_share'


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

ssh admin@10.0.0.110 "stat /mnt/atl/immich/ted | grep Uid"
# Expected: Uid: ( 9123/ immich)   Gid: ( 9123/ immich)
```

### 5.4 Run: `truenas_nfs_share`

Creates NFS exports on TrueNAS and reloads the NFS service.

Exports created:
- `/mnt/atl/immich` — rw, client `10.0.0.115`
- `/mnt/atl/images` — ro, client `10.0.0.115`

```bash
ansible-playbook playbooks/deploy_immich_data.yml --tags truenas_nfs_share
```

Verify from the Docker host:

```bash
ssh admin@10.0.0.115 "showmount -e 10.0.0.110"
# Expected: all paths listed
```

---

## 6. Playbook 2: `deploy_immich`

**Purpose:** Deploy the Immich Docker Compose stack on the docker host.  
**Idempotent:** Yes — safe to re-run.  
**Host group:** `docker_hosts`

Roles run in order:

| Role | What it does |
|---|---|
| 'apt-packages'| install required system packages |
| `nfs_client` | Installs `nfs-common`, mounts NFS shares, writes fstab |
| `linux_user` | Creates `immich` user/group on docker01 (UID/GID 9123) |
| `immich_config` | Creates directory layout, templates `.env` and `docker-compose.yml`, starts stack |
| `immich_smoke_test` | Post-deploy assertions — fails fast on any problem |

Running without a tag (recomended) will execute all tasks in the above order.

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
# Expected: /mnt/atl/images listed (/mnt/atl.immich if playbook previously run)
```

### 6.2 Run Playbook

Run the entire playbook in sequence:
```bash
ansible-playbook playbooks/deploy_immich.yml
```

Or run roles individually:

```bash
ansible-playbook playbooks/deploy_immich.yml --tags nfs_client
ansible-playbook playbooks/deploy_immich.yml --tags linux_user
ansible-playbook playbooks/deploy_immich.yml --tags immich_config
ansible-playbook playbooks/deploy_immich.yml --tags smoke_test
```

**Verify the immich stack is up:**

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

### 6.3 Directory Layout Created on host docker (10.0.0.115)

```
/mnt/nfs/
  images/              ← lala110:/mnt/atl/images        (ro)
  immich/              ← lala110:/mnt/atl/immich/ted    (rw)

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
- `/mnt/nfs/immich/ted` has created default file structure by immich user
- `/mnt/nfs/images` rejects a write (read-only confirmed)

Any failure here is a stop condition. Do not proceed to Section 7 until smoke tests pass.

---

## 7. Manual Steps: Initial Immich Configuration

These steps are not automated. Execute after the playbook and smoke tests succeed.

### 7.1 First Admin Setup

1. Open browser → `http://10.0.0.115:2283`
2. The onboarding wizard runs first. It will prompt for storage template configuration before user setup — see step 7.1.1 below.
3. Create admin account for `ted` (use email from `inventory/users.yml`)

#### 7.1.1 Storage Template (Wizard Step)

When prompted during onboarding, enable the storage template engine and set the template to:

```
{{y}}/{{MM}}/{{filename}}
```

This produces `2024/06/filename.jpg` — year folder, month subfolder, original filename. Mirrors Apple's iOS photo library organization on disk.

**Note:** There is a known Immich bug where the storage template setting does not persist through the onboarding wizard. After completing setup, verify it is still enabled under Administration → Settings → Storage Template. Re-apply if needed.

The Storage Template only affects new uploads from `ted` and `sally`. The archive external library retains its existing folder structure on TrueNAS regardless of this setting.

---

### 7.2 Create Users

Administration → Users → Create User:

| Username | Email | Role | Storage lable
|---|---|---|
| ted | ted@lalalandus.com | Admin | ted
| sally | sally@lalalandus.com | User | sally
| brian | brian@lalalandus.com | User | brian
| archive | *(service account email)* | User | archive

Use emails from `inventories/group_vars/all/users.yml`.

### 7.3 Configure External Library (Archive)

External library management is an admin function. Do **not** log in as `archive` for this step.

**Prerequisites — verify the container can see the archive path:**

```bash
docker exec -it immich_server bash
ls /usr/src/app/external-library/archive
# Expected: your photo/video folders are visible
exit
```

If the path is empty or does not exist, the volume mount in `docker-compose.yml` is not correctly configured. Do not proceed until this resolves.

**Create the library:**

1. Log in as `ted` (admin)
2. Click avatar (upper right) → Administration → External Libraries
3. Click **Create Library**
4. Select owner: `archive` — this cannot be changed after creation
5. Click **Create**

**Add the import path:**

1. Click into the newly created library
2. Click **Add Folder**
3. Enter: `/usr/src/app/external-library/archive`
4. Click **Add**

The path must be the container-side path, not the host NFS path. No path browser is available — enter it exactly.

**Trigger the initial scan:**

Adding a folder does not start a scan automatically.

1. In the External Libraries list, click the three-dot menu on the library
2. Select **Scan Library Files**
3. Navigate to Administration → Jobs to confirm the scan job is running

The first scan of a large archive will take a long time. Elevated CPU is expected — Immich is building the ML index. This is a one-time cost.

**Enable partner sharing:**

Once the scan completes and photos appear in `archive`'s timeline:

1. Log in as `archive`
2. Avatar → Account Settings → Sharing
3. Share with `ted`, `sally`, `brian`

---

### 7.4 Verify User Upload Paths

For `ted` and `sally`: Immich manages upload paths automatically via the storage template set in 7.1.1. Confirm uploads land correctly by doing a test upload from the iOS app and verifying the file appears at the expected path on the Docker host:

```bash
ssh admin@10.0.0.115 "ls /mnt/nfs/immich/"
# Expected: For each user, year/month folder structure with uploaded file 
```
---

### 7.5 Pin Immich Version

Once the deployment is confirmed stable, pin the image tag. In `inventories/group_vars/docker_hosts/docker_hosts.yml`:

```yaml
immich_version: "2.7.5"   # as of 2026-05-31
```

Re-run the playbook to apply:

```bash
ansible-playbook playbooks/deploy_immich.yml --tags immich_config
```

Commit the version change to the repo. Do not leave `release` in production.

---

## 8. Final Smoke Test

Run this sequence end-to-end after all steps complete:

```bash
# 1. Web UI responds
curl -s -o /dev/null -w "%{http_code}" http://10.0.0.115:2283
# Expected: 200

# 2. All containers healthy
ssh admin@10.0.0.115 "docker ps --filter name=immich --format 'table {{.Names}}\t{{.Status}}'"
# Expected: all Up, no Restarting

# 3. Postgres accessible
ssh admin@10.0.0.115 "docker exec immich_postgres_1 pg_isready"
# Expected: accepting connections

# 4. Archive library visible
# Log into UI as ted → verify archive photos appear via partner share

# 5. iOS upload test
# Upload one photo from iOS app as ted → confirm appears in UI
# Confirm file lands at /mnt/nfs/immich/ted/<date>/filename

# 6. NFS persistence after reboot (optional but recommended)
ssh admin@10.0.0.115 "sudo reboot"
# After reboot:
ssh -S admin@10.0.0.115 "sudo mount | grep nfs && docker ps | grep immich"
# Expected: mounts and containers present
```

---

## 9. Rollback

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

## 10. Deferred Items

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

## 11. Known Conditions

- TrueNAS uses a self-signed TLS certificate. All API calls use `validate_certs: false`. This is intentional and documented in `inventories/group_vars/truenas/truenas.yml`.
- `/mnt/atl/images` is owned by `ted:ted (UID 3000)` with mode `0775`. The world-readable bit allows the `immich` container (UID 9123) read access without an ownership change. This is intentional.
- Immich machine learning performs one-time indexing on the first library scan. Expect several hours of elevated CPU against the archive library on first run.
- The `.env` file contains secrets and must never be committed to the repo. It is managed exclusively by the `immich_config` role.
