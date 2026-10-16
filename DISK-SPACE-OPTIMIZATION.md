# Disk Space Optimization Configuration

This media stack **prefers 1080p HD** over 4K/2160p to save disk space. Quality profiles are managed by Recyclarr from `ansible/roles/media-stack/templates/recyclarr.yml.j2`.

## How It Works

### 1. Quality Profiles (via Recyclarr)

**HD+ Profile (recommended for limited disk space):**
- ✅ **Cutoff at Bluray-1080p** — stops upgrading once Bluray-1080p is reached
- ✅ **Upgrades allowed** within the profile (e.g. HDTV → WEB → Bluray)
- 📦 **4K as fallback** — 2160p qualities are ranked below every 1080p quality, so 4K is only grabbed when no 1080p release is available
- ⚠️ 4K and HDR releases are **not** given a negative score; preference comes from quality order only

**Quality order (Radarr; Sonarr is the same without the Remux entries):**
1. Bluray-1080p (highest)
2. Remux-1080p
3. WEBDL-1080p
4. HDTV-1080p
5. Bluray-2160p (fallback)
6. Remux-2160p (fallback)
7. WEBDL-2160p (fallback)
8. HDTV-2160p (fallback)

**Ultra-HD Profile:**
- 2160p qualities only
- Upgrades disabled

### 2. Release Ranking (Sonarr/Radarr built-in)

Sonarr and Radarr rank releases by **quality first, then custom format score**. Seeders and peers are only used to choose between otherwise equal releases.

**Applied by the playbook:**
- `preferIndexerFlags: true` — respects indexer flags (freeleech, etc.)
- RSS sync every 60 minutes

### 3. Anti-Virus / Fake Release Protection (via Recyclarr)

**Blocked (-10000 score):**
- 💿 BR-DISK (disc images)
- 🚫 LQ release groups and LQ release titles
- Sonarr: bad dual groups
- Radarr: 3D, Extras, Upscaled, releases with no release group

**Preferred (+1500 score):**
- Sonarr: Anime Web Tier 01, WEB Tier 01–02
- Radarr: HD Bluray Tier 01–03, WEB Tier 01–02

qBittorrent also refuses to download executables, disc images, archives and other risky file types — see [EXE-AUTO-CLEANUP.md](EXE-AUTO-CLEANUP.md).

## Expected Behavior

### When Requesting Media

1. Sonarr/Radarr search all indexers.
2. Blocked custom formats are filtered out.
3. The highest-ranked quality in the HD+ profile wins (1080p before 4K).
4. Within the same quality, the higher custom format score wins; seeders break ties.
5. The best match is downloaded.

### If Only 4K Is Available

- 4K is downloaded as a fallback.
- Existing 1080p files are not upgraded to 4K (the cutoff is Bluray-1080p).

### Failed Downloads

- Removed automatically and blocklisted
- The next best release is searched for automatically

## File Sizes (Approximate)

| Quality | Movie (2h) | TV Episode (45min) |
|---------|------------|-------------------|
| 1080p WEB | 3-6 GB | 800 MB - 1.5 GB |
| 1080p Bluray | 8-15 GB | 2-4 GB |
| 1080p Remux | 20-40 GB | 5-10 GB |
| **4K WEB** | **10-20 GB** | **2-5 GB** |
| **4K Bluray** | **40-80 GB** | **10-20 GB** |
| **4K Remux** | **60-100 GB** | **15-30 GB** |

**Disk space savings: roughly 60-75% by preferring 1080p over 4K**

> Note: Remux-1080p is ranked above WEB-1080p in Radarr, and Remux files are large. Remove it from the HD+ profile if space is tight.

## Hardlinks (Not Currently Enabled)

Downloads and media are mounted into the containers as separate paths (`/downloads` and `/movies`/`/tv`/`/data`). Sonarr/Radarr therefore **copy** completed downloads instead of hardlinking them, so a file uses double the space until qBittorrent stops seeding it (ratio 1 or 7 days, whichever comes first).

## Configuration Files

- **Quality profiles:** `/opt/media-stack/recyclarr/recyclarr.yml`
- **Sonarr config:** `/opt/media-stack/sonarr/config.xml`
- **Radarr config:** `/opt/media-stack/radarr/config.xml`

## Manual Override

To get a specific show or movie in 4K:

1. Open Sonarr/Radarr.
2. Find the series or movie.
3. Change its quality profile to **Ultra-HD**.
4. Search and grab.

## Monitoring

Disk usage of the library:
```bash
df -h /mnt/data0/appdata/data
du -sh /mnt/data0/appdata/data/*
```

Disk usage of downloads (includes torrents still seeding):
```bash
du -sh /mnt/data0/appdata/downloads/*
```

## Recyclarr Sync Schedule

- **Automatic:** daily at 06:00 (root crontab)
- **Manual:** `docker exec recyclarr recyclarr sync`
- **Logs:** `/var/log/recyclarr-sync.log`
