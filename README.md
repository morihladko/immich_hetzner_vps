# Immich on Hetzner VPS

Ansible playbook to provision a hardened Ubuntu 24.04 VPS running [Immich](https://immich.app/) with Hetzner Storage Box for media storage and Caddy for HTTPS.

## Architecture

```
Internet → Caddy (HTTPS/443) → Immich Server → Storage Box (CIFS)
                                    ↓
                              PostgreSQL + Redis (local)
```

**Storage layout:**
- **Storage Box**: library, thumbs, backups, encoded-video
- **Local SSD**: upload buffer, profile, PostgreSQL, Redis, ML cache

**Resource limits (optimized for 4 CPU / 8GB RAM):**

| Service | Memory |
|---------|--------|
| Immich Server | 2GB |
| Immich ML | 3GB |
| PostgreSQL | 1GB |
| Redis | 256MB |

**Why CIFS/SMB for Storage Box:**
- Better throughput than SSHFS for large files
- `cache=strict` with `actimeo=60` reduces network roundtrips
- Native kernel support, handles reconnects gracefully

## Prerequisites

- Hetzner VPS (recommended: 4 CPU / 8GB RAM)
- Hetzner Storage Box with SMB/CIFS enabled
- Domain pointing to VPS IP (for HTTPS)
- Ansible installed locally
- SSH key added to VPS

## Setup

### 1. Configure Secrets

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

Encrypt and optionally save password:
```bash
ansible-vault encrypt group_vars/all/vault.yml
echo "your-vault-password" > .vault_pass && chmod 600 .vault_pass
```

### 2. Configure Domain

Edit `roles/immich/defaults/main.yml`:
```yaml
immich_domain: "your-domain.com"
```

### 3. Update Inventory

Edit `hosts.ini`:
```ini
[vps]
your.server.ip
```

### 4. Bootstrap VPS

```bash
ansible-playbook setup.yml -u root -e "ansible_become_method=su"
```

### 5. Access Immich

Open `https://your-domain.com` and create your admin account.

### 6. Enable Storage Template

**Important:** To store images on the Storage Box:

1. Go to **Administration** → **Settings** → **Storage Template**
2. Enable **Storage Template**
3. Configure template (e.g., `{{y}}/{{MM}}/{{filename}}`)
4. Save

Without this, files stay in the upload buffer instead of the library.

## Maintenance

### Run playbook
```bash
# Full run
ansible-playbook setup.yml -u user --ask-become-pass

# Specific roles
ansible-playbook setup.yml -u user --ask-become-pass --tags common
ansible-playbook setup.yml -u user --ask-become-pass --tags immich
```

### Update Immich
```bash
# Via Ansible
ansible-playbook setup.yml -u user --ask-become-pass --tags immich-deploy

# Or manually
ssh user@your-server
cd /opt/immich && docker compose pull && docker compose up -d
```

### Logs and status
```bash
ssh user@your-server
cd /opt/immich && docker compose logs -f          # All logs
cd /opt/immich && docker compose logs -f immich-server  # Immich only
df -h /mnt/storagebox                             # Storage Box usage
```

## Roles

| Role | Description |
|------|-------------|
| `common` | User setup, packages, updates |
| `docker` | Docker CE installation |
| `security` | SSH hardening, fail2ban, UFW |
| `immich` | Immich + Caddy deployment |

## Tags

| Tag | Description |
|-----|-------------|
| `common` | Base system setup |
| `docker` | Docker installation |
| `security` | Security hardening |
| `immich` | Full Immich role |
| `immich-storage` | Storage Box mount |
| `immich-config` | Config files |
| `immich-deploy` | Pull and start containers |

## Backup

- **Database**: Back up `/opt/immich/postgres` regularly
- **Photos/Videos**: Already on Storage Box (consider Hetzner snapshots)
- **Config**: Managed by Ansible (commit to git)

## Troubleshooting

### Storage Box not mounting
1. Check credentials: `cat /etc/storagebox-credentials`
2. Verify SMB enabled in Hetzner Robot
3. Test manually: `mount -t cifs //host/backup /mnt/test -o user=u123456`

### Immich slow
1. Check ML container isn't OOM-killed: `docker stats`
2. Increase `actimeo` for less metadata refreshes
3. Ensure thumbs are generating (check `/mnt/storagebox/immich/thumbs`)

### Out of memory
1. Check limits: `docker stats`
2. Reduce ML memory in `docker-compose.yml` if not using face detection
3. Add swap: `fallocate -l 2G /swapfile && mkswap /swapfile && swapon /swapfile`

## SSH Config

Add to `~/.ssh/config`:
```
Host immich
    HostName your-server-ip
    User user
```

Then: `ssh immich`
