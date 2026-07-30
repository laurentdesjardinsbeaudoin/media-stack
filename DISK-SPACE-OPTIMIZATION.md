# Disk Space Optimization Configuration

This media stack is now optimized to **prefer 1080p HD** over 4K/2160p to save disk space.

## How It Works

### 1. Quality Profile Configuration (via Recyclarr)

**HD+ Profile (Recommended for Limited Disk Space):**
- ✅ **Cutoff at 1080p** - Won't upgrade beyond Bluray-1080p
- ✅ **4K Discouraged** - 2160p releases get -3000 score
- ✅ **HDR Discouraged** - HDR formats (usually 4K) get -2000 score
- ✅ **Upgrade Allowed** - Can upgrade within 1080p range (HDTV → WEB → Bluray)
- 📦 **4K as Fallback** - Will download 4K only if no 1080p available

**Quality Order (by preference):**
1. Bluray-1080p (highest)
2. Remux-1080p
3. WEB 1080p
4. HDTV-1080p
5. Bluray-2160p (discouraged, fallback only)
6. WEB 2160p (discouraged, fallback only)
7. HDTV-2160p (discouraged, fallback only)

### 2. Seeder Preference (Built-in to Sonarr/Radarr)

**Automatically Prioritized:**
- 🌱 **More seeders** = Higher priority
- 📊 **Peer count** considered in ranking
- ⚡ **Download speed** potential calculated
- 🔄 **Indexer reliability** factored in

**Configuration Applied:**
- `preferIndexerFlags: true` - Respects indexer flags (freeleech, etc.)
- Automatic sorting by seeders/peers
- No manual intervention needed

### 3. Anti-Virus Protection (via Recyclarr)

**Blocked Completely (-10000 score):**
- 💿 ISO files (BR-DISK)
- 💻 Executables
- 🚫 Low-quality/fake release groups
- 🎭 Obfuscated releases
- 📼 x265 HD encodes (often fake)

**Discouraged (-5000 score):**
- Suspicious patterns
- Unknown release groups

**Preferred (+1500 score):**
- Known good release groups
- Scene releases
- Trusted uploaders

## Expected Behavior

### When Requesting Media:

1. **Sonarr/Radarr searches all indexers**
2. **Filters out fakes/viruses** (blocked formats)
3. **Prefers 1080p releases** (HD+ profile)
4. **Sorts by seeders** (more seeders = higher priority)
5. **Downloads best match** (1080p + most seeders)

### If Only 4K Available:

- Will download 4K as fallback
- Won't upgrade existing 1080p to 4K
- Saves disk space by preferring smaller files

### Failed Downloads:

- Automatically removed
- Next best release tried
- Fake releases skipped

## File Sizes (Approximate)

| Quality | Movie (2h) | TV Episode (45min) |
|---------|------------|-------------------|
| 1080p WEB | 3-6 GB | 800 MB - 1.5 GB |
| 1080p Bluray | 8-15 GB | 2-4 GB |
| 1080p Remux | 20-40 GB | 5-10 GB |
| **4K WEB** | **10-20 GB** | **2-5 GB** |
| **4K Bluray** | **40-80 GB** | **10-20 GB** |
| **4K Remux** | **60-100 GB** | **15-30 GB** |

**Disk Space Savings: 60-75% by preferring 1080p over 4K**

## Configuration Files

- **Quality Profiles:** `/opt/media-stack/recyclarr/recyclarr.yml`
- **Sonarr Config:** `/opt/media-stack/sonarr/config.xml`
- **Radarr Config:** `/opt/media-stack/radarr/config.xml`

## Manual Override

If you want a specific show/movie in 4K:

1. Go to Sonarr/Radarr
2. Find the series/movie
3. Change quality profile to "Ultra-HD"
4. Search and grab manually

## Monitoring

Check disk usage:
```bash
df -h /mnt/data0/appdata/data
du -sh /mnt/data0/appdata/data/*
```

Check download sizes:
```bash
docker exec -it qbittorrentvpn qbittorrent-cli torrent list
```

## Recyclarr Sync Schedule

- **Automatic:** Daily at 6:00 AM
- **Manual:** `docker exec recyclarr recyclarr sync`
- **Logs:** `/var/log/recyclarr-sync.log`
