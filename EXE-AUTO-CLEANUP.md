# Malware Auto-Detection and Retry System

## What This Does

Bad releases are handled in three layers:

1. **qBittorrent exclusion list (preventive)** — `.exe`, `.iso`, scripts, archives and many other file types are never written to disk. Configured in `ansible/roles/media-stack/templates/qBittorrent.conf.j2` (`Session\ExcludedFileNames`).
2. **Recyclarr custom formats (preventive)** — Sonarr/Radarr give BR-DISK, LQ and similar releases a score of -10000, so they are never grabbed.
3. **Malware monitor (reactive)** — a background container that, every 60 seconds:
   - finds `.exe`/`.iso` files in the downloads folder and removes the matching download through Sonarr/Radarr
   - finds queue items stuck with "no files eligible for import" (usually because the exclusion list removed every file) and removes those too

In both cases the release is blocklisted and Sonarr/Radarr search for a replacement automatically.

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
