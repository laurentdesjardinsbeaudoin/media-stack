# Media Stack - Automated Deployment

Complete media server stack with Sonarr, Radarr, Prowlarr, Overseerr, Plex, qBittorrent (VPN), and more.

## What This Playbook Configures

**✅ Automatically Configured:**
- Docker containers for all services
- Quality profiles via Recyclarr (with TRaSH Guides best practices)
- **Anti-virus/fake release protection** (blocks ISOs, executables, low-quality groups)
- Download client (qBittorrent) in Sonarr/Radarr
- Prowlarr app sync (Sonarr + Radarr auto-configured)
- Automatic failed download removal
- HAProxy reverse proxy with SSL
- Daily Recyclarr sync for profile updates

**⚙️ Manual Configuration Still Needed:**
- Indexer selection in Prowlarr (then auto-syncs to Sonarr/Radarr)
- Root folders in Sonarr/Radarr
- Plex library setup
- User accounts and passwords
- Fine-tuning quality preferences

**This playbook now handles ~80% of the setup automatically!**

## Prerequisites

1. **VPN Configuration Files**
   
   Before deployment, you need your VPN provider's configuration files:
   - `vpn.ovpn` - Your VPN provider's OpenVPN configuration
   
   After deployment, copy these to:
   ```bash
   {{ appdata_dir }}/binhex-qbittorrentvpn/openvpn/
   ```

2. **Server Requirements**
   - Rocky Linux 9 / AlmaLinux 9 (or similar RHEL-based)
   - Sufficient disk space for media and downloads
   - SSH access with root or sudo privileges

3. **Client Requirements** (your local machine)
   - Docker and Docker Compose installed

## Quick Start

### 1. Configure Variables

Edit `ansible/group_vars/media-stack.yml`:


### 2. Update Inventory

Edit `ansible/inventory/hosts.yml`:

### 3. Deploy Infrastructure

#### Build the Ansible Docker image:

```bash
docker compose -f docker-compose.yml build
```

#### Run the Ansible playbook:

```bash
docker compose -f docker-compose.yml run --rm ansible-shell
```

Once inside the container:

```bash
cd ansible
ansible-playbook -i inventory/hosts.yml main.yml --ask-pass --ask-become --ask-vault-pass
```

The container automatically mounts your SSH keys from `$HOME/.ssh`.

**Time: ~15 minutes** (downloads images, starts containers)

### 4. Copy VPN Configuration

After deployment completes:

```bash
# Copy your VPN files to the server
scp credentials.conf vpn.ovpn root@192.168.2.3:{{ appdata_dir }}/binhex-qbittorrentvpn/openvpn/

# Restart qBittorrent to apply VPN config
ssh root@192.168.2.3 "cd /opt/media-stack && docker compose restart qbittorrentvpn"
```

### 5. Configure Services (Manual)

Now configure each service through their web interfaces:

#### **Prowlarr** (http://ip:9696)
1. Add your preferred indexers
2. Go to Settings → Apps
3. Add Sonarr: `http://ip:8989`, API key from Sonarr
4. Add Radarr: `http://ip:7878`, API key from Radarr
5. Sync indexers

#### **Sonarr** (http://ip:8989)
1. Settings → Download Clients → Add qBittorrent
   - Host: `ip`, Port: `8080`
   - Username: `root`, Password: `adminadmin`
2. Settings → Media Management → Add Root Folder: `/tv`
3. Configure quality profiles as needed

#### **Radarr** (http://ip:7878)
1. Settings → Download Clients → Add qBittorrent
   - Host: `ip`, Port: `8080`
   - Username: `root`, Password: `adminadmin`
2. Settings → Media Management → Add Root Folder: `/movies`
3. Configure quality profiles as needed

#### **qBittorrent** (http://ip:8080)
1. Login: `root` / `adminadmin`
2. **Change password immediately**: Tools → Options → Web UI → Authentication

#### **Overseerr** (http://ip:5055)
1. Login with Plex account
2. Add Sonarr: `http://ip:8989`, API key from Sonarr
3. Add Radarr: `http://ip:7878`, API key from Radarr
4. Select Plex libraries to monitor

#### **Plex** (http://ip:32400/web)
1. Claim server with your Plex account
2. Add libraries:
   - Movies: `/data`
   - TV Shows: `/data`

**Total configuration time: ~30-45 minutes**

## Services

| Service | Port | Description |
|---------|------|-------------|
| Dashboard | 80 | Landing page with service links |
| HAProxy Stats | 8404 | Proxy statistics and health |
| Sonarr | 8989 | TV show management (auto-configured) |
| Radarr | 7878 | Movie management (auto-configured) |
| Prowlarr | 9696 | Indexer manager (auto-syncs to Sonarr/Radarr) |
| Overseerr | 5055 | Media requests |
| Plex | 32400 | Media server |
| qBittorrent | 8080 | Download client (VPN-protected, auto-configured) |
| Bazarr | 6767 | Subtitle management |
| Jackett | 9117 | Torrent indexer proxy |
| Recyclarr | - | Quality profile automation (background service) |
| Samba | 445 | File sharing |

