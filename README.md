# Media Stack - Automated Deployment

Complete media server stack with Sonarr, Radarr, Prowlarr, Overseerr, Plex, qBittorrent (VPN), and more, deployed with Ansible and Docker Compose.

## What This Playbook Configures

**✅ Automatically Configured:**
- Docker containers for all services
- Firewall ports (firewalld)
- OpenVPN config and credentials for qBittorrent (deployed from Ansible Vault)
- qBittorrent settings, including a file-exclusion list (blocks `.exe`, `.iso`, archives, etc.)
- Quality profiles via Recyclarr (TRaSH Guides custom formats)
- Download client (qBittorrent) in Sonarr/Radarr
- Prowlarr app sync (Sonarr + Radarr added as applications)
- Failed download handling (auto-remove, blocklist and re-search)
- Malware monitor container (see [EXE-AUTO-CLEANUP.md](EXE-AUTO-CLEANUP.md))
- HAProxy with a self-signed certificate, serving the dashboard
- Daily Recyclarr sync (06:00)

**⚙️ Manual Configuration Still Needed:**
- Indexers in Prowlarr (they then sync to Sonarr/Radarr)
- Root folders in Sonarr/Radarr
- Plex server claim and library setup
- Overseerr setup
- Passwords for Portainer, qBittorrent and the *arr apps

## Prerequisites

1. **Server**
   - Rocky Linux 9 / AlmaLinux 9 (or similar RHEL-based)
   - Docker CE repository already configured (the playbook installs `docker-ce` but does not add the repo)
   - Sufficient disk space for media and downloads
   - SSH access with root or sudo privileges

2. **Local machine**
   - Docker and Docker Compose (Ansible runs inside a container)

3. **Secrets** (Ansible Vault)
   - `ansible/group_vars/all/vault` — encrypted variables:
     - `v_sonarr_api_key`
     - `v_radarr_api_key`
     - `v_prowlarr_api_key`
   - `ansible/group_vars/all/openvpn/vpn.ovpn` — your VPN provider's OpenVPN config (vault-encrypted file)
   - `ansible/group_vars/all/openvpn/credentials.conf` — VPN username/password (vault-encrypted file)

   Edit them with:
   ```bash
   ansible-vault edit ansible/group_vars/all/vault
   ansible-vault edit ansible/group_vars/all/openvpn/vpn.ovpn
   ansible-vault edit ansible/group_vars/all/openvpn/credentials.conf
   ```

   To replace the VPN files later, see [Updating the VPN Configuration](#updating-the-vpn-configuration).

   > **New install:** Sonarr, Radarr and Prowlarr generate their API keys on first start. Run the playbook once, copy each key from the app's *Settings → General*, store them in the vault, then run the playbook again so the API configuration steps succeed.

## Quick Start

### 1. Configure Variables

Edit `ansible/group_vars/media-stack.yml`:

| Variable | Default | Purpose |
|----------|---------|---------|
| `appdata_dir` | `/opt/media-stack` | App configs and the generated `docker-compose.yml` |
| `media_dir` | `/mnt/data0/appdata/data` | Media library |
| `downloads_dir` | `/mnt/data0/appdata/downloads` | qBittorrent downloads |
| `transcode_dir` | `/mnt/data0/appdata/transcode` | Plex transcode scratch space (10–50 GB+) |
| `lan_network` | `192.168.0.0/16` | LAN allowed to reach qBittorrent while the VPN is up |
| `name_servers` | Google + Cloudflare | DNS used by the qBittorrent VPN container |

The time zone (`America/New_York`) is currently hard-coded in `ansible/roles/media-stack/templates/docker-compose.yml.j2`.

### 2. Update Inventory

Edit `ansible/inventory/hosts.yml` (INI format despite the extension) and put your server's IP under the `media-stack` group:

```ini
[media-stack]
192.168.2.3
```

### 3. Deploy

Build the Ansible image and open a shell in it:

```bash
docker compose build
docker compose run --rm ansible-shell
```

Inside the container (the `ansible/` folder is mounted as the working directory, and your `~/.ssh` is mounted read-only):

```bash
ansible-playbook main.yml --ask-pass --ask-become-pass --ask-vault-pass
```

Drop `--ask-pass` if you use SSH keys.

**Time: ~15 minutes** (downloads images, starts containers)

> Every run recreates all containers (`docker compose up -d --force-recreate`), so expect a short outage.

### 4. Configure Services (Manual)

Replace `ip` with your server's address.

#### **Prowlarr** (http://ip:9696)
1. Add your preferred indexers.
2. Sonarr and Radarr are already added under *Settings → Apps*; indexers sync to them automatically.

#### **Sonarr** (http://ip:8989)
1. The qBittorrent download client is already configured.
2. *Settings → Media Management* → add root folder `/tv`.

#### **Radarr** (http://ip:7878)
1. The qBittorrent download client is already configured.
2. *Settings → Media Management* → add root folder `/movies`.

> Note: `/tv`, `/movies` and Plex's `/data` currently all point to the same host folder (`media_dir`).

#### **qBittorrent** (http://ip:8080)
1. Web UI username is `root`. Clients on private networks (10.x, 172.16.x, 192.168.x) skip the login.
2. **Change the password**: *Tools → Options → Web UI → Authentication*.

#### **Overseerr** (http://ip:5055)
1. Sign in with your Plex account.
2. Add Sonarr (`http://ip:8989`) and Radarr (`http://ip:7878`) with their API keys.
3. Select the Plex libraries to monitor.

#### **Plex** (http://ip:32400/web)
1. Claim the server with your Plex account.
2. Add libraries pointing at `/data`.

#### **Portainer** (http://ip:9000)
1. Create the admin account straight away — Portainer has full control of Docker on the host.

## Services

| Service | Port | Description |
|---------|------|-------------|
| Dashboard (HAProxy) | 80 / 443 | Landing page with service links (self-signed cert on 443) |
| HAProxy Stats | 8404 | Proxy statistics (no authentication) |
| Portainer | 9000 | Docker management UI |
| Sonarr | 8989 | TV show management (auto-configured) |
| Radarr | 7878 | Movie management (auto-configured) |
| Prowlarr | 9696 | Indexer manager (syncs to Sonarr/Radarr) |
| Overseerr | 5055 | Media requests |
| Plex | 32400 | Media server (host networking) |
| qBittorrent | 8080 | Download client, all traffic through the VPN (`tun0`) |
| Bazarr | 6767 | Subtitle management |
| Jackett | 9117 | Torrent indexer proxy (overlaps with Prowlarr) |
| Samba | 139 / 445 | File shares: `appdata`, `media`, `downloads` |
| Recyclarr | - | Quality profile sync (background) |
| Malware monitor | - | Removes downloads containing `.exe`/`.iso` files (background) |

## Scheduled Jobs (root crontab)

| Schedule | Job | Log |
|----------|-----|-----|
| Daily 06:00 | `recyclarr sync` | `/var/log/recyclarr-sync.log` |
| Hourly | Restart qBittorrent (see Known Issues) | `/var/log/qbittorrent-restart.log` |

## Updating the VPN Configuration

Do this whenever your provider gives you a new `vpn.ovpn` or your VPN credentials change.

1. **Replace the encrypted file(s)** from the repo root:
   ```bash
   # New .ovpn file from your provider
   ansible-vault encrypt /path/to/new.ovpn --output ansible/group_vars/all/openvpn/vpn.ovpn

   # Or edit the existing one in place
   ansible-vault edit ansible/group_vars/all/openvpn/vpn.ovpn

   # Credentials: exactly two lines, username then password, Unix line endings
   ansible-vault edit ansible/group_vars/all/openvpn/credentials.conf
   ```
   Don't remove the file's `auth-user-pass` line. You don't need to edit it either: during deployment the playbook rewrites it to `auth-user-pass /config/openvpn/credentials.conf`. If the line is missing, OpenVPN never sends your credentials.

2. **Redeploy** (see [Deploy](#3-deploy)):
   ```bash
   ansible-playbook main.yml --ask-become-pass --ask-vault-pass
   ```
   The playbook recreates all containers, including `qbittorrentvpn`.

3. **Check the tunnel came up**:
   ```bash
   docker logs -f qbittorrentvpn
   ```
   Look for `Initialization Sequence Completed`. Common errors:

   | Log message | Cause |
   |-------------|-------|
   | `can't ask for 'Enter Auth Username:'` | `vpn.ovpn` on the server still has a bare `auth-user-pass` — it was copied there by hand instead of deployed by the playbook |
   | `AUTH_FAILED` | Wrong username/password, or Windows line endings in `credentials.conf` |
   | `Error opening 'Auth' username/password source` | `credentials.conf` missing from `/opt/media-stack/binhex-qbittorrentvpn/openvpn/` |

## Further Reading

- [DISK-SPACE-OPTIMIZATION.md](DISK-SPACE-OPTIMIZATION.md) — quality profiles and disk usage
- [EXE-AUTO-CLEANUP.md](EXE-AUTO-CLEANUP.md) — malware monitor and auto-retry

## Known Issues

Found during review and not fixed yet:

- **Hourly qBittorrent restart cron never works** — it restarts service `qbittorrent`, but the service is called `qbittorrentvpn`.
- **qBittorrent exclusion list is too broad** — short patterns like `*TS*` and `*TC*` are case-insensitive and match normal titles (e.g. "Nights", "Witcher"), and `*.rar`/`*.zip` exclude packed releases. This can leave a torrent with nothing to import.
- **Download client credentials mismatch** — the playbook adds qBittorrent to Sonarr/Radarr as `admin`/`adminadmin`, but the qBittorrent user is `root`. It works only because of the private-network login bypass.
- **Samba shares allow guest read/write**, including `appdata` (configs, API keys, VPN credentials, TLS key).
- **No hardlinks** — downloads and media are mounted as separate paths in the containers, so imports copy files and seeding torrents use double the disk space.
- **SELinux is disabled** permanently by the playbook.
