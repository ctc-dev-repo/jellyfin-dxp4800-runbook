# Runbook — Jellyfin on Ugreen NASync DXP4800 Plus (UGOS Pro)

Goal: production-quality Jellyfin deployment on the DXP4800 Plus, tuned for efficiency: Direct Play first, Intel Quick Sync (QSV) hardware transcoding for everything else, fast local metadata on NVMe, minimal idle power, safe upgrades.

Audience: operator comfortable with web UIs and basic SSH. Effort: ~60–90 minutes.

---

## 1. Platform summary and design decisions

| Component | DXP4800 Plus detail | Consequence for Jellyfin |
|---|---|---|
| CPU | Intel Pentium Gold 8505 (Alder Lake, 1P+4E, up to 4.4 GHz) | Plenty for a media server; never software-transcode |
| iGPU | Intel UHD Graphics (Xe-LP) with Quick Sync | QSV: H.264/HEVC/VP9/**AV1 decode**, H.264/HEVC encode, HDR→SDR tone mapping. **No AV1 encode** |
| RAM | 8 GB DDR5-4800, expandable (officially to 64 GB) | Enough for several direct streams + 2–4 transcodes |
| System disk | Dedicated 128 GB SSD (OS lives here, not on your HDDs) | Don't put app data here; reserve it for UGOS |
| Fast storage | 2× M.2 NVMe slots | Put Jellyfin config/database/cache here — biggest single efficiency win |
| Bulk storage | 4× SATA bays (RAID 0/1/5/6/10/JBOD) | Media files only |
| Network | 1× 10GbE + 1× 2.5GbE | Serve clients over 10GbE; keep WAN/internet on the 2.5GbE if you split networks |

### Design principles applied in this runbook

1. **Direct Play is the goal** — every transcode you avoid is free capacity. Hardware is configured so transcodes *do* happen correctly when unavoidable.
2. **Official `jellyfin/jellyfin` image via Docker Compose** — not the UGOS App Center app: you control versioning, mounts, and GPU access, and upgrades are deliberate.
3. **Database/config/cache on NVMe** — Jellyfin 10.11 uses a single SQLite database (`jellyfin.db`) that is latency-sensitive; browsing a large library wakes nothing and stutters nothing when it's on flash.
4. **Read-only media mounts** — server can never damage your library.
5. **Host networking** — best throughput and enables DLNA/discovery if ever wanted.
6. **LAN-only by default** — remote access via VPN tunnel, not exposed ports.

### Expected capacity (approximate, stock 8 GB)

| Workload | Capacity |
|---|---|
| Direct Play (any bitrate ≤ 10GbE link) | Limited only by network/clients |
| 1080p H.264 → 1080p transcode | ~5–7 concurrent streams |
| 4K HDR HEVC → 1080p SDR (tone mapped) | ~2–3 concurrent streams |
| Audio-only transcode (TrueHD→AAC etc.) | Many; cheap |

---

## 2. Prerequisites

- UGOS Pro set up, firmware updated (Control Panel → About → check updates)
- Storage configured:
  - **NVMe pool created** in Storage & RAID (either one big volume or a dedicated volume). This runbook assumes your NVMe volume is `/volume1` and the bulk HDD volume is `/volume2` — adjust paths throughout if yours differ
  - HDD array with your media, folder structure ready (see §5)
- A client device on the LAN for testing
- SSH access enabled: Control Panel → Terminal → Enable SSH (port 22). Connect with `ssh <admin>@<nas-ip>`, elevate with `sudo -i`

Find your UIDs/GIDs once and record them (used in §4):

```bash
id                                  # your admin user (uid/gid)
getent group render | cut -d: -f3   # render group GID — commonly 105 on UGOS Pro
ls -la /dev/dri                     # must show card0 + renderD128 (iGPU driver loaded)
```

If `/dev/dri` is missing after a UGOS update, reboot once before troubleshooting.

---

## 3. Folder layout

In the UGOS Files/Shared Folder app, or via shell:

| Path | Volume | Purpose |
|---|---|---|
| `/volume1/docker/jellyfin/config` | NVMe | Server config + `jellyfin.db` (the precious data) |
| `/volume1/docker/jellyfin/cache` | NVMe | Transcode segments, image cache, trickplay |
| `/volume2/media/movies` | HDD | Movies (example) |
| `/volume2/media/tv` | HDD | TV shows (example) |
| `/volume1/backup/jellyfin` | NVMe | Config backups |

```bash
mkdir -p /volume1/docker/jellyfin/{config,cache} /volume1/backup/jellyfin
chown -R 1000:1000 /volume1/docker/jellyfin     # match the uid/gid recorded in §2
chmod -R u+rwX /volume1/docker/jellyfin
```

Media folders stay owned as they are; the container gets read-only access.

---

## 4. Deploy via Docker Compose

UGOS Pro → Docker app → **Project → Create** → name `jellyfin` → paste:

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:10.11.11        # pin the version; upgrade deliberately
    container_name: jellyfin
    restart: unless-stopped
    network_mode: host                       # native throughput, discovery, optional DLNA
    hostname: jellyfin
    user: "1000:1000"                        # match uid/gid from §2
    group_add:
      - "105"                                # render GID from §2 — REQUIRED for GPU
    environment:
      - TZ=America/Los_Angeles               # set yours
      - JELLYFIN_PublishedServerUrl=http://YOUR-NAS-IP:8096
    volumes:
      - /volume1/docker/jellyfin/config:/config
      - /volume1/docker/jellyfin/cache:/cache
      - /volume2/media:/media:ro             # read-only, whole tree; subfolders per library
    devices:
      - /dev/dri:/dev/dri                    # iGPU passthrough for QSV
```

Deploy and wait ~30 s, then open `http://YOUR-NAS-IP:8096`.

Notes:
- `network_mode: host` means the container shares the NAS's stack — port 8096 binds directly, no port mapping lines needed. If you later add a reverse proxy on 80/443 there is no conflict.
- If you prefer bridge networking instead, remove `network_mode` and add `ports: ["8096:8096"]`; DLNA and auto-discovery will be degraded.
- Running as root (omit `user:`) also works and sidesteps permission issues, but the explicit uid/gid above is the cleaner setup.

### Verify GPU access before configuring anything

```bash
docker exec jellyfin ls -l /dev/dri                       # expect card0 + renderD128
docker exec jellyfin /usr/lib/jellyfin-ffmpeg/vainfo       # expect iHD driver + profile list incl. HEVC / AV1
```

Both must succeed before continuing. `vainfo` failing here is a permissions problem (wrong `group_add` GID) — fix before wasting time in the UI.

---

## 5. Setup wizard and library hygiene

First-run wizard: create admin account (don't share it as a viewing account), language, skip library creation until folders are named properly.

Library rules (Dashboard → Libraries):

- One library per type (Movies, Shows, Music) pointing at the respective top folder
- Real-time monitoring: **on** (inotify-based — no polling, no HDD wakeups while browsing)
- Metadata downloaders: leave defaults; enable **NFO savers** if you want portable metadata
- Preferred download language/subtitle settings per household taste

File naming saves more CPU than any setting: `Movies/Movie Name (Year)/Movie Name (Year).mkv`, `Shows/Show Name/Season 01/Show Name S01E01.mkv`. Properly named/remuxed files Direct Play everywhere; garbage names cause guessing, mismatches, and transcodes.

---

## 6. Transcoding configuration (the important part)

Dashboard → Playback → Transcoding:

| Setting | Value |
|---|---|
| Hardware acceleration | **Intel QuickSync (QSV)** |
| QSV device | `/dev/dri/renderD128` |
| Hardware decoding — enable | H.264, HEVC, VP9, **AV1**, MPEG-2, VC1 |
| Hardware encoding — enable | H.264, HEVC (**not** AV1 — unsupported by this iGPU, would silently fall back to CPU) |
| Tone mapping method | **Tone mapping** (hardware, VPP) |
| Enable tone mapping | **On** — required for HDR→SDR clients |
| Allow HDR decode/tone-map enhancements | On |

Efficiency-relevant toggles elsewhere:

- Dashboard → Playback → general: leave **Throttling** enabled (stops encoding when client buffer is full)
- Encoding preset/QSV quality defaults are sensible; lower "quality/speed" gains little on fixed-function hardware
- Dashboard → Networking: **disable "Enable automatic port mapping"**; set Local network subnets explicitly (e.g. `192.168.1.0/24`)

### Prove hardware transcoding works

1. Play a 4K HEVC file from a browser client, set quality to 720p/4 Mbps (forces a transcode)
2. Dashboard → Activity: the stream row should show the codec with a **transcode/hardware indicator**
3. From SSH, confirm low CPU and GPU engagement:

```bash
top -b -n1 | grep -i ffmpeg        # ffmpeg CPU should be well under ~40%, not 400%
docker exec jellyfin cat /proc/jellyfin/stat 2>/dev/null || true
```

Success criteria: smooth playback, CPU roughly idle, no `vaapi` errors in Dashboard → Logs. If CPU pegs all cores, you're software-transcoding — see Troubleshooting.

---

## 7. Efficiency tuning (scheduled tasks, storage, power)

Dashboard → Scheduled Tasks:

| Task | Recommendation | Why |
|---|---|---|
| Scan Media Library | Daily ~03:30 | Off-peak; real-time monitoring handles additions anyway |
| Extract embedded images | On library add only | Cheap, avoids full-library churn |
| Chapter image extraction | **Disabled** | Brutal CPU/IO cost, near-zero value |
| Trickplay previews | Optional, weekly 04:00 | Nice feature; first pass is heavy — keep it on NVMe (it writes to /cache) |
| Check for plugin updates | Weekly | Hygiene |

Storage/power practices:

- Browsing the UI reads only NVMe-resident database/images — HDDs stay asleep; playback spins them up. Keep it that way by leaving real-time monitoring on and **disabling UGOS's own media indexing/photo AI on the media folders** (App Center services) — duplicate scanners cause constant HDD wakeups
- Enable HDD hibernation in UGOS Storage settings; verify drives actually sleep overnight (they should — Jellyfin won't touch them without a request)
- Fan profile: Standard; Quiet (29–34 dB) is fine for transcoding loads on this chassis
- Transcoding temp path: Dashboard → Paths → set to `/cache/transcodes` (lands on NVMe). Add a monthly task to clear leftovers after crashes:

```bash
find /volume1/docker/jellyfin/cache/transcodes -type f -mtime +2 -delete 2>/dev/null || true
```

Optional hardware upgrade worth considering for many-user households: RAM to 16 GB (single SO-DIMM slot occupied; check Ugreen compatibility list).

---

## 8. Networking and remote access

LAN:

- Give the NAS a DHCP reservation or static IP; plug the **10GbE** port into your main switch
- Clients on Wi-Fi will cap at their radio, not the server

Remote access — pick one, in order of preference:

1. **VPN tunnel (recommended):** Tailscale/WireGuard container on the NAS; clients connect to the tailnet IP. Nothing is exposed to the internet
2. **Reverse proxy:** Nginx Proxy Manager container with HTTPS + strong auth; forward only 443, keep Jellyfin's "Remote access" restricted to your known domains

Never: port-forward 8096 raw, enable UPnP on your router for it, or expose DLNA to the WAN side.

---

## 9. Backup and upgrade procedures

### Backups (weekly, automated)

UGOS Control Panel → Task Scheduler → Create → Scheduled task running as root:

```bash
#!/bin/sh
systemctl stop docker 2>/dev/null || docker stop jellyfin
tar czf "/volume1/backup/jellyfin/config-$(date +%F).tgz" \
    -C /volume1/docker/jellyfin config
docker start jellyfin
find /volume1/backup/jellyfin -name 'config-*.tgz' -mtime +28 -delete
```

Stopping the container gives a consistent `jellyfin.db` snapshot; downtime is ~10 s. Keep at least 4 weeks of copies, and periodically copy one archive off-NAS.

Restore = extract archive back into `/volume1/docker/jellyfin/config`, recreate container.

### Upgrades

1. Read the release notes for the target version (major versions like 10.11 migrated the database — first boot after such upgrades can take minutes on big libraries)
2. Take a fresh backup per §9
3. Edit the compose `image:` tag (e.g. `10.11.11` → next minor), deploy, watch logs: `docker logs -f jellyfin`
4. Rollback = set the previous tag back and redeploy (config schema downgrades aren't supported — this is why we pin tags and keep backups rather than floating `latest`)
5. After any UGOS system update, spot-check §4's GPU verification — OS updates occasionally reset group IDs or device permissions

---

## 10. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `vainfo` fails inside container, works pattern unclear | Wrong `group_add` GID | Re-run `getent group render \| cut -d: -f3` on host; update compose; recreate container |
| Transcode plays fine but CPU hits 100% | Silent software fallback (AV1 encode selected, or GPU init failed earlier in log) | Remove AV1 from *encoding* list; check Logs for `No VA display found` |
| Container exits 139 on startup | Device permission/driver race after UGOS update | Reboot NAS; re-verify §2 checks; recreate container |
| HDR content looks washed out in SDR clients | Hardware tone mapping disabled or wrong method | Enable Tone mapping (VPP) per §6 |
| UI browsing slow, HDD clicking while navigating | Config/DB on spinning volume | Move `/config` to NVMe volume per §3 |
| Leftover partial `.ts` transcode files | Container killed mid-stream (crash/update) | Monthly cleanup command in §7; harmless otherwise |
| Discovery doesn't find server from apps | Bridge-mode container or client subnet mismatch | Use host networking (§4); verify Local subnets in Networking settings |
| Remote playback buffers on 4K originals | Client link slow (Wi-Fi) forcing repeated seeks | Prefer Direct Play-capable client apps (native Jellyfin apps, Infuse, Kodi addon); raise client bitrate limit |
| Drives never sleep | Another service indexing media (UGOS photo/AI apps) or polling mount | Disable UGOS media indexing on those folders; check Access Logs |

Debug escalation order: `docker logs --tail 200 jellyfin` → `vainfo` test (§4) → privileged-mode test (temporarily set `privileged: true` — if transcoding then works, it's purely permissions) → Jellyfin forums with the ffmpeg line from Logs.

---

## 11. References

- Jellyfin Intel hardware acceleration (official): <https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/intel/>
- Current releases: <https://github.com/jellyfin/jellyfin/releases>
- UGREEN's Jellyfin guide (DXP series, QSV section): <https://nas.ugreen.com/blogs/how-to/install-jellyfin-setup-step-by-step>
- DXP4800 Plus hardware review/specs: <https://www.techpowerup.com/review/ugreen-nasync-dxp4800-plus/>
- UGREEN product page (RAM expansion, ports): <https://ai.ugreen.com/products/ugreen-nasync-dxp4800-plus-nas-storage>

---

## Appendix A — Command cheat sheet

```bash
# Status
docker ps --filter name=jellyfin && docker logs --tail 50 jellyfin
docker exec jellyfin /usr/lib/jellyfin-ffmpeg/vainfo | head -20

# Restart after compose edit
cd <project dir> && docker compose up -d     # or redeploy via UGOS Docker → Project

# Force-transcode sanity check while a stream runs
top -b -n1 | grep ffmpeg

# Manual backup now
tar czf /volume1/backup/jellyfin/config-manual.tgz -C /volume1/docker/jellyfin config

# Clean stale transcode segments
find /volume1/docker/jellyfin/cache/transcodes -type f -mtime +2 -delete
```
