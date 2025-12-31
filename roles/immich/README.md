# Immich Ansible Role

Deploys [Immich](https://immich.app/) (self-hosted photo/video backup) on a Hetzner VPS using Docker Compose, with Hetzner Storage Box for media storage.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Hetzner VPS (4 CPU / 8GB RAM)           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    Docker                            │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │   Immich    │  │   Immich    │  │   Redis     │  │   │
│  │  │   Server    │  │     ML      │  │  (256MB)    │  │   │
│  │  │   (2GB)     │  │   (3GB)     │  └─────────────┘  │   │
│  │  └─────────────┘  └─────────────┘                    │   │
│  │  ┌─────────────┐  ┌─────────────────────────────┐   │   │
│  │  │  Postgres   │  │  Local Cache & Thumbnails   │   │   │
│  │  │  (1GB)      │  │  /opt/immich/cache          │   │   │
│  │  └─────────────┘  └─────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                     CIFS/SMB v3.0                          │
│                      (port 445)                             │
└───────────────────────────┼─────────────────────────────────┘
                            │
                            ▼
              ┌─────────────────────────────┐
              │   Hetzner Storage Box       │
              │   /backup/immich/           │
              │   (Photos & Videos)         │
              └─────────────────────────────┘
```

## Storage Strategy

| Data Type | Location | Reason |
|-----------|----------|--------|
| Photos/Videos | Storage Box (CIFS) | Large capacity, cost-effective, persistent |
| Thumbnails/Cache | Local VPS | Fast access for UI responsiveness |
| PostgreSQL | Local VPS | Database needs low latency |
| Redis | Local VPS | In-memory cache, local only |
| ML Model Cache | Local VPS | Frequent access during processing |

## Why CIFS/SMB for Storage Box

Hetzner Storage Boxes support multiple protocols. CIFS/SMB v3.0 is recommended because:

- **Performance**: Better throughput than SSHFS for large files
- **Caching**: `cache=strict` with `actimeo=60` reduces network roundtrips
- **Reliability**: Native kernel support, handles reconnects gracefully
- **Compatibility**: Works well with Docker bind mounts

Mount options used:
```
vers=3.0,cache=strict,actimeo=60,uid=1000,gid=1000
```

## Resource Limits (Optimized for 4 CPU / 8GB RAM)

| Service | Memory Limit | Notes |
|---------|--------------|-------|
| immich-server | 2GB | Main API and background jobs |
| immich-machine-learning | 3GB | ML models for face/object detection |
| postgres | 1GB | Database with pgvecto-rs extension |
| redis | 256MB | Session and job queue cache |
| **Total Reserved** | ~6.25GB | Leaves headroom for OS and cache |

## Required Variables

These must be set in your playbook or inventory:

```yaml
# Hetzner Storage Box credentials
storagebox_host: "u123456.your-storagebox.de"
storagebox_user: "u123456"
storagebox_password: "your-password"

# Security - change in production!
postgres_password: "secure-random-password"
```

## Optional Variables

```yaml
# Immich version (default: release)
immich_version: "release"

# Directories
immich_base_dir: "/opt/immich"

# Network
immich_port: 2283

# Timezone
immich_timezone: "Europe/Berlin"

# Disable storage box (use local storage only)
storagebox_enabled: false
```

## Firewall

The role automatically opens the Immich port (default 2283) in UFW. For production, consider:

1. **Reverse proxy**: Use Caddy/Nginx with HTTPS instead of exposing Immich directly
2. **IP restriction**: Limit access to specific IPs if not using a reverse proxy

## Usage

### 1. Configure Storage Box

First, enable SMB/CIFS on your Hetzner Storage Box:
1. Log into Hetzner Robot
2. Go to Storage Box settings
3. Enable "Samba" under "Storage Box Services"

### 2. Set Variables

Create `host_vars/your-host.yml` or set in your playbook:

```yaml
storagebox_host: "u123456.your-storagebox.de"
storagebox_user: "u123456"
storagebox_password: "your-secure-password"
postgres_password: "another-secure-password"
```

### 3. Run Playbook

```bash
ansible-playbook -i hosts.ini setup.yml -u user --ask-become-pass
```

### 4. Access Immich

Open `http://your-server-ip:2283` and create your admin account.

## Maintenance

### View logs
```bash
cd /opt/immich && docker compose logs -f
```

### Update Immich
```bash
cd /opt/immich && docker compose pull && docker compose up -d
```

### Check Storage Box mount
```bash
df -h /mnt/storagebox
mount | grep storagebox
```

### Restart services
```bash
cd /opt/immich && docker compose restart
```

## Backup Considerations

- **Database**: Back up `/opt/immich/postgres` regularly
- **Photos/Videos**: Already on Storage Box (consider enabling Hetzner snapshots)
- **Config**: The `.env` and `docker-compose.yml` are managed by Ansible

## Troubleshooting

### Storage Box not mounting
1. Check credentials in `/etc/storagebox-credentials`
2. Verify SMB is enabled in Hetzner Robot
3. Test manually: `mount -t cifs //host/backup /mnt/test -o user=u123456`

### Immich slow with large library
1. Ensure thumbnails are on local disk (not Storage Box)
2. Check ML container isn't OOM-killed: `docker stats`
3. Consider increasing `actimeo` value for less metadata refreshes

### Out of memory
1. Check container limits: `docker stats`
2. Reduce ML memory limit if not using heavy ML features
3. Add swap as last resort: `fallocate -l 2G /swapfile`
