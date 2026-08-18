# Malware Auto-Detection and Retry System

## What This Does

This system automatically:
1. ✅ **Monitors downloads** for .exe and .iso files every 60 seconds
2. ✅ **Detects malicious files** when they appear
3. ✅ **Calls Sonarr/Radarr API** to mark as failed
4. ✅ **Blacklists the source** (prevents re-download from same source)
5. ✅ **Triggers automatic retry** from a different source
6. ✅ **Sonarr/Radarr delete files** (no direct rm -rf operations)

## How It Works

### 1. Malware Monitor Service
- Runs as a Docker container (`malware-monitor`)
- Scans downloads directory every 60 seconds for .exe and .iso files
- When malicious file is found:
  - Identifies which download it belongs to
  - Calls Sonarr/Radarr API with `removeFromClient=true&blacklist=true`
  - Sonarr/Radarr handle file deletion safely
  - Source is blacklisted to prevent re-download
  - Logs everything to `/appdata/malware-monitor/malware-monitor.log`

### 2. Sonarr/Radarr Auto-Retry
- Failed download handling is enabled
- When a download is marked failed:
  - Automatically searches for another release
  - Picks a different source (blacklist prevents same source)
  - Downloads the replacement
  - WEBDL releases are **NOT** blocked (you get to keep them!)

### 3. API-Only Deletion (No rm -rf)
- **Zero direct file system operations** from monitor script
- Sonarr/Radarr APIs handle all deletion
- They know exactly what they downloaded
- Safer than shell script deletion

## Deployment

Run Ansible to deploy the updated configuration:

```bash
cd /home/ldbeaudoin/media-stack/ansible
ansible-playbook -i inventory/hosts.yml main.yml --ask-vault-pass
```

This will:
- Deploy the malware-monitor.sh script
- Start the malware-monitor container
- Enable failed download handling in Sonarr/Radarr
- Configure auto-blacklist and retry

## Monitoring

### View Malware Monitor Logs

```bash
# Live tail
docker logs -f malware-monitor

# View log file
cat /opt/media-stack/malware-monitor/malware-monitor.log
```

### Check for Current Malicious Files

```bash
# Manual scan for .exe and .iso files
find /mnt/data0/appdata/downloads -type f \( -iname "*.exe" -o -iname "*.iso" \) 2>/dev/null
```

### View Blacklist

**Sonarr**: http://YOUR_IP:8989 → Activity → Blocklist  
**Radarr**: http://YOUR_IP:7878 → Activity → Blocklist

## Configuration

### Change Scan Interval

Edit the script in [ansible/roles/media-stack/templates/exe-monitor.sh.j2](ansible/roles/media-stack/templates/exe-monitor.sh.j2):

```bash
CHECK_INTERVAL=60  # Change to desired seconds
```

Then redeploy with Ansible.

### Manually Trigger Cleanup

If you want to manually trigger cleanup of an .exe file:

```bash
# The monitor will pick it up automatically within 60 seconds
# Or restart the container to trigger immediate scan
docker restart exe-monitor
```

## Testing

To test that the system works:

1. **Check monitor is running**:
   ```bash
   docker ps | grep exe-monitor
   docker logs exe-monitor
   ```

2. **Simulate a bad download** (testing only!):
   ```bash
   # Create a test .exe in downloads
   touch /path/to/downloads/test-show/test.exe
   
   # Watch the monitor detect and handle it
   docker logs -f exe-monitor
   ```

3. **Verify blacklist**:
   - Check Sonarr/Radarr Activity → Blocklist
   - Should see the bad release listed

## What Gets Blocked vs Allowed

### ✅ ALLOWED (No longer blocked by Recyclarr):
- WEBDL-1080p (your preferred format!)
- WEBDL-2160p
- All other valid video formats

### ⚠️ STILL BLOCKED by Recyclarr (preemptive):
- BR-DISK (.iso files)
- Known bad release groups
- Suspicious naming patterns
- Files with no proper release group

### 🚨 BLOCKED by EXE Monitor (reactive):
- **Any .exe files** - Detected after download, deleted immediately
- Source automatically blacklisted
- New source tried automatically

## Advantages of This Approach

1. **Doesn't block WEBDL** - You get the quality you want
2. **Reactive cleanup** - Only deletes actual malicious files
3. **Automatic retry** - No manual intervention needed
4. **Source blacklisting** - Won't download from bad source again
5. **Comprehensive logging** - Full audit trail
6. **No false positives** - Only targets actual .exe files

## Troubleshooting

### Monitor not detecting .exe files?

```bash
# Check if monitor is running
docker ps | grep exe-monitor

# Check logs for errors
docker logs exe-monitor

# Verify script permissions
ls -la /path/to/appdata/exe-monitor/exe-monitor.sh

# Restart monitor
docker restart exe-monitor
```

### Download not retrying automatically?

1. Check Sonarr/Radarr settings:
   - Settings → Download Clients
   - Verify "Redownload Failed" is enabled
   - Verify "Remove Failed" is enabled

2. Check if the item is in the queue:
   - Activity → Queue
   - Should see a new download starting

3. Check for available sources:
   - Series/Movie → Search
   - Verify other sources are available

### Need to clear the blacklist?

```bash
# Via Sonarr UI
# Activity → Blocklist → Select items → Delete

# Via API (clear all)
curl "http://localhost:8989/api/v3/blocklist" -H "X-Api-Key: YOUR_API_KEY" | \
  jq -r '.[].id' | \
  xargs -I {} curl -X DELETE "http://localhost:8989/api/v3/blocklist/{}" -H "X-Api-Key: YOUR_API_KEY"
```

## Files Modified/Created

- `ansible/roles/media-stack/templates/exe-monitor.sh.j2` - Monitor script (NEW)
- `ansible/roles/media-stack/templates/docker-compose.yml.j2` - Added exe-monitor service
- `ansible/roles/media-stack/tasks/main.yml` - Added deployment tasks and API config

## Logs Location

- **Monitor logs**: `/appdata/exe-monitor/exe-monitor.log`
- **Sonarr logs**: `docker logs sonarr`
- **Radarr logs**: `docker logs radarr`
- **qBittorrent logs**: `docker logs qbittorrentvpn`
