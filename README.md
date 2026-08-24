# jellyfin-dxp4800-runbook

Runbook for deploying Jellyfin on a Ugreen NASync DXP4800 Plus (UGOS Pro), tuned
for efficiency: Direct Play first, Intel Quick Sync transcoding, NVMe-resident
database/cache, minimal idle power.

- Full procedure: [RUNBOOK.md](RUNBOOK.md)

## TL;DR

1. Create NVMe volume for app data; media lives on the HDD array
2. Deploy official `jellyfin/jellyfin` (pinned) via Docker Compose Project in UGOS — host networking, `/dev/dri` passthrough, `group_add` render GID (`getent group render`)
3. Dashboard → Playback: **QSV**, all hardware *decode*, H.264/HEVC *encode only* (iGPU has no AV1 encode), hardware tone mapping on
4. Scheduled tasks at night; chapter extraction off; UGOS media-indexing off so HDDs can sleep
5. LAN-only by default; remote access via Tailscale/WireGuard; weekly config backups with the container briefly stopped

See RUNBOOK.md for exact commands, verification steps (vainfo, forced-transcode test), capacity expectations, and troubleshooting.
