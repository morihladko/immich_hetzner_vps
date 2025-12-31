# Immich on Hetzner VPS

Ansible playbook to provision a hardened Ubuntu 24.04 VPS running [Immich](https://immich.app/) (self-hosted photo/video backup) with Hetzner Storage Box for media storage.

## Prerequisites

- Hetzner VPS (recommended: 4 CPU / 8GB RAM)
- Hetzner Storage Box with SMB/CIFS enabled
- Ansible installed locally
- SSH key added to VPS

## Setup

### 1. Configure Secrets

Edit the vault file with your credentials:

```bash
nano group_vars/all/vault.yml
```

Required variables:
```yaml
vault_storagebox_url: "//u123456.your-storagebox.de/backup"
vault_storagebox_user: "u123456"
vault_storagebox_password: "your-password"
vault_postgres_password: "secure-db-password"
```

Encrypt the vault:
```bash
ansible-vault encrypt group_vars/all/vault.yml
```

Optionally store the vault password:
```bash
echo "your-vault-password" > .vault_pass
chmod 600 .vault_pass
```

### 2. Update Inventory

Edit `hosts.ini` with your VPS IP:
```ini
[vps]
your.server.ip
```

### 3. First Run (Bootstrap)

Run immediately after creating the VPS (connects as root):

```bash
ansible-playbook setup.yml -u root -e "ansible_become_method=su"
```

### 4. Access Immich

Open `http://your-server-ip:2283` and create your admin account.

### 5. Enable Storage Template

**Important:** To store images on the Storage Box, you must enable Storage Templates in Immich:

1. Go to **Administration** → **Settings** → **Storage Template**
2. Enable **Storage Template**
3. Configure the template (e.g., `{{y}}/{{MM}}/{{filename}}`)
4. Save

Without this, Immich stores files in the upload buffer instead of the library directory on the Storage Box.

## Maintenance

### Run full playbook
```bash
ansible-playbook setup.yml -u user --ask-become-pass
```

### Run specific components
```bash
# Update system packages
ansible-playbook setup.yml -u user --ask-become-pass --tags update

# Only Immich deployment
ansible-playbook setup.yml -u user --ask-become-pass --tags immich

# Only firewall rules
ansible-playbook setup.yml -u user --ask-become-pass --tags firewall
```

### Update Immich

SSH into the server and run:
```bash
cd /opt/immich
docker compose pull
docker compose up -d
```

Or via Ansible:
```bash
ansible-playbook setup.yml -u user --ask-become-pass --tags immich-deploy
```

### View Immich logs
```bash
ssh user@your-server-ip
cd /opt/immich && docker compose logs -f
```

### Check Storage Box mount
```bash
ssh user@your-server-ip
df -h /mnt/storagebox
```

### Restart Immich
```bash
ssh user@your-server-ip
cd /opt/immich && docker compose restart
```

## Available Tags

| Tag | Description |
|-----|-------------|
| `update` | Apt update and upgrade |
| `install` | Package installations |
| `docker` | Docker setup |
| `firewall` | UFW rules |
| `ssh` | SSH hardening + fail2ban |
| `immich` | Full Immich role |
| `immich-storage` | Storage Box mount |
| `immich-config` | Immich config files |
| `immich-deploy` | Pull and start containers |

## SSH Config

Add to `~/.ssh/config` for easier access:
```
Host immich-vps
    HostName your-server-ip
    User user
```

Then connect with: `ssh immich-vps`
