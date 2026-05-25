# Run Book: configure-ansible.md

## Purpose

Install and configure Ansible on the control host in a repeatable, auditable manner.
This is a one-time setup. Once complete, the control host is ready to run all project playbooks.

This document is written for a future operator with no prior context.
Follow the steps exactly and in order.

Errors are not expected at any step.
Any deviation is a stop condition.

---

## Scope

This run book covers:

- WSL2 and Ubuntu setup on Windows 11
- Ansible installation
- Project directory and configuration
- Inventory structure and validation
- Ansible collections installation
- Ansible Vault setup for secrets
- SSH key generation

It intentionally stops before application deployment runbooks.

---

## Preconditions

- Windows 11 with internet access
- Homelab intranet access

---

## Step 1: Install WSL2 and Ubuntu

From PowerShell running as Administrator:

```powershell
wsl --install -d Ubuntu-22.04
```

Reboot when prompted. When the Ubuntu terminal opens, set a username and password.

All remaining steps run inside the WSL2 Ubuntu shell.

---

## Step 2: Install Ansible

```bash
sudo apt update && sudo apt install -y ansible
```

Verify:

```bash
ansible --version
```

Expected output (versions may differ):

```
ansible [core 2.17.14]
  config file = /home/ted/home-lab/ansible-immich/ansible.cfg
  configured module search path = ['/home/ted/.ansible/plugins/modules', ...]
  python version = 3.10.12
  jinja version = 3.0.3
  libyaml = True
```

---

## Step 3: Create the Project Directory

If the project is already in a git repo:

```bash
git clone https://github.com/tuxhobo/ansible-immich ~/home-lab/ansible-immich
cd ~/home-lab/ansible-immich
```

All subsequent steps assume the working directory is `~/home-lab/ansible-immich`.

---

## Step 4: Verify ansible.cfg

Confirm `ansible.cfg` in the project root matches the following exactly:

```ini
[defaults]
remote_user         = ansible
private_key_file    = /home/ted/.ssh/ansible_pbs_key
inventory           = inventories
roles_path          = roles
collections_path    = ./collections:~/.ansible/collections:/usr/share/ansible/collections:/usr/lib/python3/dist-packages/ansible_collections
log_path            = /tmp/ansible.log
vault_password_file = ~/.ansible/.vault_pass_immich

host_key_checking   = True
retry_files_enabled = False
deprecation_warnings = False
interpreter_python  = auto_silent

stdout_callback     = default
result_format       = yaml
bin_ansible_callbacks = True
timeout             = 30
forks               = 10

[inventory]
enable_plugins = host_list, script, auto, yaml, ini

[privilege_escalation]
become = False
become_method = sudo
become_ask_pass = True
become_flags = -H -S -E

[ssh_connection]
pipelining = true
ssh_args   = -o ControlMaster=auto -o ControlPersist=60s
```

Note: `vault_password_file` is commented out in the repo. Uncomment it after completing Step 6.

---

## Step 5: Install Ansible Collections

Collections are defined in `./collections/requirements.yml`:

```yaml
collections:
  - name: community.general
    version: ">=9.0.0"
  - name: ansible.posix
    version: ">=1.5.0"
  - name: ansible.utils
    version: ">=4.1.0"
```

Install:

```bash
ansible-galaxy collection install -r ./collections/requirements.yml
```

Verify:

```bash
ansible-galaxy collection list | grep -E "community\.(general|proxmox)|ansible\.(posix|utils)"
```

Expected: all four collections listed at or above the minimum versions.

---

## Step 6: Set Up Ansible Vault

Create the vault password file:

```bash
mkdir -p ~/.ansible
echo "your-vault-password" > ~/.ansible/.vault_pass_immich
chmod 600 ~/.ansible/.vault_pass_immich
```

Use a strong, unique password. Store it in your password manager.

Uncomment `vault_password_file` in `ansible.cfg`:

```ini
vault_password_file = ~/.ansible/.vault_pass_immich
```

Verify vault access:

```bash
ansible-vault view inventories/group_vars/vault.yml
```

Must decrypt without error.

Verify no secrets are still placeholder values:

```bash
ansible-vault view inventories/group_vars/vault.yml | grep CHANGEME
```

Empty output means all secrets are set. Any remaining `CHANGEME` must be replaced before running any playbook.

Generate `vault_immich_jwt_secret` if not yet done:

```bash
openssl rand -hex 32
```

Edit vault:

```bash
ansible-vault edit inventories/group_vars/vault.yml
```

---

## Step 7: Generate the Ansible SSH Key

Generate the key pair on the control host. The key path must match `private_key_file` in `ansible.cfg`.

```bash
ssh-keygen -t ed25519 -C "ansible_immich" -f ~/.ssh/ansible_imch_key
```

Do not set a passphrase. Ansible requires unattended key-based auth.

Confirm the key exists:

```bash
ls -la ~/.ssh/ansible_imch_key*
```

Expected:

```
~/.ssh/ansible_imch_key
~/.ssh/ansible_imch_key.pub
```

The public key is deployed to LXC containers during provisioning. See `immich-deploy.md` for details.

---

## Step 8: Validate Inventory

```bash
ansible-inventory --list | python3 -m json.tool | grep -A5 immich
ansible-inventory --graph
```

Confirm `immich` appears under `lxc`, `ldap_servers`, and `debian` groups.

---

## Stop Point

At this stage:
- Ansible is installed and configured
- Collections are installed
- Vault is initialized and all secrets are populated
- SSH key is generated on the control host
- Inventory validates cleanly

The control host is ready to run playbooks. Proceed to `immich-deploy.md`.

---

## Notes

- This run book covers control host setup only
- Do not add VM, PBS, or service configuration steps here
- Update when the Ansible version, WSL distribution, or project structure changes
