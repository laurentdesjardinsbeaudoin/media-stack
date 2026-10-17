# Malware Auto-Detection and Retry System

## What This Does

Bad releases are handled in three layers:

1. **qBittorrent exclusion list (preventive)** — `.exe`, `.iso`, scripts, archives and many other file types are never written to disk. Configured in `ansible/roles/media-stack/templates/qBittorrent.conf.j2` (`Session\ExcludedFileNames`).
2. **Recyclarr custom formats (preventive)** — Sonarr/Radarr give BR-DISK, LQ and similar releases a score of -10000, so they are never grabbed.
3. **Malware monitor (reactive)** — a background container that, every 60 seconds:
   - finds `.exe`/`.iso` files in the downloads folder and removes the matching download through Sonarr/Radarr
   - finds queue items stuck with "no files eligible for import" (usually because the exclusion list removed every file) and removes those too

In both cases the release is blocklisted and Sonarr/Radarr search for a replacement automatically.

4. **Cleanuparr (being introduced to replace the malware monitor)** — see [Cleanuparr](#cleanuparr) below. It covers everything the monitor does and also removes stalled downloads, failed imports, and files in `downloads` that no torrent uses.

## How It Works

### 1. Malware Monitor Service
- Runs as the `malware-monitor` Docker container (Alpine, installs `bash`, `curl` and `jq` on start).
- Script: `ansible/roles/media-stack/templates/exe-monitor.sh.j2`, deployed to `{{ appdata_dir }}/malware-monitor/malware-monitor.sh`.
- **File scan** — when an `.exe` or `.iso` is found:
  - Takes the name of the folder containing the file.
  - Looks for a Sonarr queue item whose title contains that name, then Radarr.
  - Calls `DELETE /api/v3/queue/{id}?removeFromClient=true&blocklist=true`.
  - If no queue item matches, it only logs a "manual cleanup required" message (it repeats every scan until the file is gone).
- **Import-warning scan** — queue items with status `warning` and a message matching "eligible for import" or "no files found" are removed and blocklisted the same way.
- Logs to `docker logs malware-monitor` and to `{{ appdata_dir }}/malware-monitor/malware-monitor.log`.

### 2. Sonarr/Radarr Auto-Retry
The playbook enables failed download handling in both apps:
- Completed download handling on
- Remove completed / failed downloads on
- Redownload failed (including from interactive search) on

When a download is removed with `blocklist=true`, that release is blocklisted and a new search runs. WEBDL releases are **not** blocked.

### 3. API-Only Deletion
- The monitor script never deletes files itself.
- `removeFromClient=true` tells Sonarr/Radarr to remove the torrent and its data from qBittorrent.

## Cleanuparr

[Cleanuparr](https://github.com/Cleanuparr/Cleanuparr) runs as the `cleanuparr` container (web UI on port 11011). Ansible deploys the container and its blocklist (`{{ appdata_dir }}/cleanuparr/blocklist.txt`). **All other settings are made in the web UI** and stored in its database under `{{ appdata_dir }}/cleanuparr/`, so they survive redeploys but are not in this repo.

### Compared with the malware monitor

| Malware monitor does | Cleanuparr equivalent |
|---|---|
| Finds `.exe`/`.iso` on disk and removes + blocklists the download | Torrents whose files were all excluded by qBittorrent (complete with 0 bytes) are removed + blocklisted by the **Queue Cleaner** on its next run. **Malware Blocker** also skips any file matching its blocklist (409 extensions) and removes the torrent if every file is blocked |
| Removes items stuck with "no files eligible for import" | **Queue Cleaner** failed-import handling with the `No files found are eligible for import` pattern |
| Runs every 60 s | Runs every 5 min; failed imports need 3 strikes (about 15 min) |

Things to know:
- Cleanuparr only acts on downloads in the Sonarr/Radarr queue. Torrents added by hand are protected only by qBittorrent's exclusion list, the same as today.
- Cleanuparr ignores files that qBittorrent has already excluded. "Delete if any file is blocked" therefore never triggers for `.exe` and other excluded types, so leave it off. A torrent containing a video **and** an `.exe` still downloads the video only (the same as today).
- The blocklist file has no comments on purpose: Cleanuparr treats every line as a pattern.

### Rollout

1. Deploy with the playbook. The malware monitor keeps running.
2. Configure Cleanuparr as below, with **Dry Run on**.
3. For about a week, compare `docker logs cleanuparr` with `docker logs malware-monitor`. Check that Cleanuparr would remove what you expect, and nothing else.
4. Turn **Dry Run off**.
5. Once it has run live without problems, remove the `malware-monitor` service and its tasks.

### Web UI settings

Open `http://YOUR_IP:11011` and create the admin account.

**General**
- Dry Run: **on** (until step 4 of the rollout)

**Arrs**
- Sonarr: `http://YOUR_IP:8989`, API key from Sonarr → Settings → General
- Radarr: `http://YOUR_IP:7878`, API key from Radarr → Settings → General

**Download Client**
- qBittorrent: `http://YOUR_IP:8080`, username `root` and your Web UI password
- No path mapping needed (Cleanuparr mounts downloads at `/downloads`, the same as qBittorrent)

**Malware Blocker**
- Enabled, every 5 minutes
- Sonarr and Radarr: Enable Blocklist **on**, Blocklist Path `/config/blocklist.txt`, Type **Blacklist**
- Delete if any file is blocked: **off**

**Queue Cleaner**
- Enabled, every 5 minutes
- Failed Import Max Strikes: `3`
- Pattern mode: **Include**, patterns:
  ```
  No files found are eligible for import
  title mismatch
  not found in the grabbed release
  was unexpected considering
  not imported or missing
  Unable to parse
  Not an upgrade
  Not a Custom Format upgrade
  Not a quality revision upgrade
  ```
  The `Not ... upgrade` patterns clear downloads Sonarr/Radarr refused because you already have an equal or better version (the "same thing twice" case).
- Stalled rule: public torrents, 0–100 %, max strikes `6` (about 30 min with no progress), reset strikes on progress **on**
- Downloading Metadata max strikes: `6`
- Slow rules: none

**Download Cleaner**
- Enabled, hourly
- Seeding rules: none. Sonarr/Radarr already remove finished torrents after qBittorrent reaches ratio 1 or 7 days of seeding.
- **Unlinked Download Settings: leave OFF.** Hardlinks are not set up in this stack, so every download would look unlinked and be removed.
- Orphaned Files (inside the qBittorrent client section):
  - Enabled
  - Scan directory: `/downloads`
  - Orphaned directory: `/downloads/orphaned`
  - Minimum age: `24` hours
  - Purge after: leave empty at first; once you trust it, `336` (14 days)

  Check the layout first with `ls /mnt/data0/appdata/downloads`. If finished downloads sit in `sonarr/` and `radarr/` subfolders, use `/downloads/sonarr` and `/downloads/radarr` as scan directories instead. Cleanuparr never moves a folder that contains an active torrent.

### Monitoring

```bash
docker logs -f cleanuparr
```

Orphaned files are moved to `/mnt/data0/appdata/downloads/orphaned` for review before anything is deleted.

## Deployment

Deployed by the main playbook. From the Ansible container (see [README.md](README.md#3-deploy)):

```bash
ansible-playbook main.yml --ask-become-pass --ask-vault-pass
```

This deploys the script, starts the `malware-monitor` container and enables failed download handling in Sonarr/Radarr.

## Monitoring

### View Malware Monitor Logs

```bash
# Live tail
docker logs -f malware-monitor

# Log file (default appdata_dir)
cat /opt/media-stack/malware-monitor/malware-monitor.log
```

The log file is not rotated.

### Check for Current Malicious Files

```bash
find /mnt/data0/appdata/downloads -type f \( -iname "*.exe" -o -iname "*.iso" \) 2>/dev/null
```

### View Blocklist

**Sonarr**: http://YOUR_IP:8989 → Activity → Blocklist
**Radarr**: http://YOUR_IP:7878 → Activity → Blocklist

## Configuration

### Change Scan Interval

Edit `CHECK_INTERVAL` in `ansible/roles/media-stack/templates/exe-monitor.sh.j2`:

```bash
CHECK_INTERVAL=60  # seconds
```

Then redeploy with Ansible.

### Trigger an Immediate Scan

```bash
docker restart malware-monitor
```

## Testing

1. **Check the monitor is running**:
   ```bash
   docker ps | grep malware-monitor
   docker logs malware-monitor
   ```

2. **Simulate a bad download** (testing only):
   ```bash
   # Use the folder of an item that is currently in the Sonarr/Radarr queue,
   # otherwise the monitor only logs "manual cleanup required"
   touch /mnt/data0/appdata/downloads/<download-folder>/test.exe
   docker logs -f malware-monitor
   ```

3. **Verify the blocklist**: Sonarr/Radarr → Activity → Blocklist should show the release.

## What Gets Blocked vs Allowed

### ✅ Allowed
- WEBDL-1080p and WEBDL-2160p
- Other normal video releases

### ⚠️ Blocked by Recyclarr (before grabbing)
- BR-DISK (disc images)
- LQ / suspicious release groups and titles
- Bad dual groups (Sonarr)
- 3D, Extras, Upscaled and releases with no group (Radarr)

### 🚫 Excluded by qBittorrent (never downloaded)
- Executables, scripts, disc images, archives, Office macro files, samples, trailers, and cam/telesync markers. See the full list in `qBittorrent.conf.j2`.

### 🚨 Removed by the Malware Monitor (after download)
- Downloads containing `.exe` or `.iso` files
- Downloads stuck with no importable files

## Limitations

- The file scan matches the **folder name** against the queue **title** (case-sensitive). Single-file torrents saved directly in the downloads root, or folders named differently from the release title, are not matched.
- Some exclusion patterns are broad (e.g. `*TS*`, `*TC*`) and can exclude legitimate files. The import-warning scan then blocklists the release, which may repeat for the same title.

## Troubleshooting

### Monitor not detecting files?

```bash
docker ps | grep malware-monitor
docker logs malware-monitor
ls -la /opt/media-stack/malware-monitor/malware-monitor.sh
docker restart malware-monitor
```

### Download not retrying automatically?

1. Sonarr/Radarr → Settings → Download Clients: check "Redownload Failed" and "Remove Failed" are enabled.
2. Activity → Queue: a new download should appear.
3. Series/Movie → Search: check other releases are available.

### Need to clear the blocklist?

Via the UI: Activity → Blocklist → select items → Delete.

Via the API (Sonarr shown; use port 7878 for Radarr):

```bash
API_KEY=YOUR_API_KEY
curl -s "http://localhost:8989/api/v3/blocklist?pageSize=1000" -H "X-Api-Key: $API_KEY" | \
  jq -r '.records[].id' | \
  xargs -I {} curl -s -X DELETE "http://localhost:8989/api/v3/blocklist/{}" -H "X-Api-Key: $API_KEY"
```

## Related Files

- `ansible/roles/media-stack/templates/exe-monitor.sh.j2` — monitor script
- `ansible/roles/media-stack/templates/qBittorrent.conf.j2` — exclusion list
- `ansible/roles/media-stack/templates/recyclarr.yml.j2` — custom formats
- `ansible/roles/media-stack/templates/docker-compose.yml.j2` — `malware-monitor` service
- `ansible/roles/media-stack/tasks/main.yml` — deployment and Sonarr/Radarr API configuration

## Logs Location

- **Monitor**: `docker logs malware-monitor` or `/opt/media-stack/malware-monitor/malware-monitor.log`
- **Sonarr**: `docker logs sonarr`
- **Radarr**: `docker logs radarr`
- **qBittorrent**: `docker logs qbittorrentvpn`
