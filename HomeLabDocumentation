# Home Lab Setup Documentation

**Last updated:** July 2026
**Host machine:** Windows 11 Pro PC (static IP `192.168.1.11`), left on 24/7
**Planned migration:** Ubuntu Server on a used ASUS Mini PC (Intel i7 8th gen)

---

## 1. Infrastructure Overview

| Component | Details |
|---|---|
| Host OS | Windows 11 Pro, version 25H2 |
| Linux layer | WSL2, Ubuntu 24.04.3 LTS, user `anish` |
| Networking mode | Mirrored (WSL2 shares host's IP directly) |
| Container runtime | Docker Desktop, WSL2 integration enabled |
| Static IP | `192.168.1.11` (DHCP reservation or manual static config) |
| Storage | WD Elements 2TB external USB HDD, reformatted to ext4 |
| Mount points | `E:\` (Windows) → `/mnt/e` (WSL) |

### Drive folder structure
```
E:\NextCloudBackupDrive\
├── Photos\            → Nextcloud auto-upload + Immich External Library source
├── MusicLibrary\       → Jellyfin music library
├── ImmichUploads\      → Immich's own managed upload storage (UPLOAD_LOCATION)
├── ROMs\
└── backups\            → (planned) location for DB backup dumps
```

---

## 2. Services Running

### Nextcloud — port 8080
- **Location:** `~/nextcloud-app`
- **Stack:** Docker Compose — `nextcloud:latest` + `mariadb:11`
- **External Storage:** mounted at `/mnt/nasdrive` → `E:\NextCloudBackupDrive`
- **Access:** LAN-wide via `http://192.168.1.11:8080`, trusted domains configured
- **Mobile app:** configured for browsing + auto-upload of camera photos into the `Photos` folder
- **Known issue:** external storage folders created directly via Windows (not through Nextcloud) don't appear until a manual or scheduled `occ files:scan --all` is run

### Immich — port 2283
- **Location:** `~/immich-app`
- **Stack:** Official Docker Compose (immich-server, immich-machine-learning, redis, postgres)
- **`.env` config:**
  - `UPLOAD_LOCATION=/mnt/e/NextCloudBackupDrive/ImmichUploads`
  - `DB_DATA_LOCATION=./postgres` (kept on internal storage — NOT the external drive, for reliability)
  - `TZ=Asia/Kolkata`
  - `DB_PASSWORD` changed from default
- **External Library:** points at `/mnt/nextcloud-photos`, mapped to `E:\NextCloudBackupDrive\Photos` — same files visible in both Nextcloud and Immich, no duplication
- **Known issue:** accidentally pointed the External Library mount at `ImmichUploads` instead of `Photos` at one point — corrected back
- **Known issue:** admin account was lost after repeated container restarts (database existed but had no user records) — had to recreate the admin account from scratch via the setup wizard

### Jellyfin — port 8096
- **Purpose:** movies, TV, music, photos
- **Music library:** moved from an old Spotiflac mount to `E:\NextCloudBackupDrive\MusicLibrary`
- **Access:** LAN via Windows Firewall rule, tested from iPad
- **Fixes applied:** resolved `network_mode: host` incompatibility with explicit port mapping; fixed missing cover art via the Image Extractor setting; generated 24 custom playlist cover art images with Pillow

---

## 3. Networking & Remote Access

- **Static IP:** `192.168.1.11` set via router DHCP reservation (preferred) or manual Windows static IP config
- **SSH into WSL:** OpenSSH server installed, reachable directly thanks to mirrored networking mode
- **Tailscale:** chosen for secure remote access to all services from outside the home network, without exposing any ports publicly
- **HTTPS (planned):** Nginx Proxy Manager + mkcert for a locally-trusted CA, avoiding browser security warnings on LAN-only hostnames (e.g. `nextcloud.home.local`)
- **Headscale:** considered as a self-hosted alternative to Tailscale's coordination server — would require a stable public address (static IP or DDNS) for the Headscale server itself, unlike plain Tailscale which needs no exposed ports at all. Not yet implemented.

---

## 4. Known Recurring Issue: Drive Mount Race Condition

**Symptom:** After a reboot, Docker containers sometimes start before the WD Elements drive finishes mounting in WSL. This causes bind-mounted folders to appear empty inside containers (Linux creates an empty directory at a missing bind-mount path rather than erroring).

**Temporary fix (in use now):**
```bash
ls /mnt/e/NextCloudBackupDrive/        # confirm drive is actually mounted
cd ~/nextcloud-app && docker compose down && docker compose up -d
cd ~/immich-app && docker compose down && docker compose up -d
```

**Permanent fix (outlined, not yet applied):** a systemd service that waits for the drive to be mounted before starting containers — planned via a wait script + `homelab-startup.service`, using WSL's `systemd=true` support. This entire class of problem goes away once migrated to Ubuntu Server (native `fstab` mounts at boot, no WSL handoff).

---

## 5. Backups

### Nextcloud
- **Automatic filesystem rescan** via cron, to catch files/folders added directly through Windows rather than through Nextcloud itself:
  ```
  0 * * * * docker exec -u www-data nextcloud-app-app-1 php occ files:scan --all >> /home/anish/nextcloud-scan.log 2>&1
  ```

### Immich
- No built-in backup UI available in the current version — using manual/scheduled `pg_dumpall` instead:
  ```
  0 3 * * * docker exec -t immich_postgres pg_dumpall -c -U postgresimmich > /home/anish/immich-db-backup-$(date +\%F).sql 2>> /home/anish/immich-backup.log
  30 3 * * * find /home/anish/immich-db-backup-*.sql -mtime +14 -delete
  ```
- Consideration: point dumps at the WD drive (`E:\NextCloudBackupDrive\backups`) so they're covered by the planned drive-to-drive redundancy sync too.

### Data redundancy (planned, not yet implemented)
- A second external drive to hold a copy of the WD Elements drive
- Chose **scheduled sync** (Robocopy `/MIR`, via Windows Task Scheduler) over real-time mirroring (Storage Spaces), since Storage Spaces mirroring is not well-suited to external USB drives
- Second drive not yet purchased

---

## 6. Abandoned / Parked Ideas

- **Raspberry Pi 3B as NAS** — considered first, pivoted to the Windows PC since it was already running 24/7 for Jellyfin/Immich anyway; the Pi is now free for other use
- **Plex as a Jellyfin alternative** — considered, but decided against since Plex requires an account and phones home to Plex's servers for metadata, and paywalls hardware transcoding behind Plex Pass — both run counter to the self-hosted, no-external-dependency approach used everywhere else in this setup
- **Google Photos backup via rooted WSA** — set up a rooted Windows Subsystem for Android instance with Magisk, spoofing a Pixel 5 identity, to get free-tier Google Photos Storage Saver backup; hit a March 2025 Google Photos Library API restriction on the metadata-export side and pivoted toward Google Takeout instead

---

## 7. Planned Migration: Ubuntu Server on ASUS Mini PC

**Hardware:** used ASUS Mini PC, Intel i7 8th gen, currently running Windows 11 Pro

**Decision:** fresh reinstall, not a data migration — Nextcloud users, Immich albums/faces, and Jellyfin watch history will all be recreated from scratch. Only the actual media files (on the WD external drive) carry over, since the drive itself moves to the new machine.

**Base OS decision:** skip Windows entirely, install **Ubuntu Server 24.04 LTS** — this removes WSL2, Docker Desktop, mirrored networking mode, and the drive-mount race condition entirely, since Linux handles drive mounts and Docker natively.

**Migration outline:**
1. Install Ubuntu Server, enable SSH during setup
2. Set static IP via Netplan
3. Install Docker natively (`get.docker.com` script) — no Docker Desktop needed
4. Move the WD Elements drive physically to the new machine, mount via `/etc/fstab` (UUID-based, reliable at boot)
5. Recreate Nextcloud, Immich, and Jellyfin Compose stacks with paths adjusted (`/mnt/nasdrive/...` instead of `/mnt/e/...`)
6. Reconfigure firewall via `ufw`
7. Reinstall Tailscale (native Linux client)
8. Decommission Docker/WSL on the old primary PC once confirmed working

**Note on old data:** photos uploaded directly through the Immich app (not via Nextcloud sync) live in `~/immich-app/library` — this is on the old PC's internal storage, not the external drive, so it will NOT carry over automatically unless manually copied before decommissioning. Given the fresh-install decision, this is expected to be left behind.

---

## 8. Hardware Notes

- Considered various mini PC alternatives (Beelink, GMKtec, Minisforum, GEEKOM) but the existing ASUS i7 8th gen unit is already better-specced than most budget options for this workload — no need to buy new hardware unless specifically wanting internal SATA bays to eliminate USB-drive power/reliability issues
- 2.5" internal HDD sourcing in India: Seagate BarraCuda and WD Blue both viable at 2TB; stock fluctuates on Amazon.in/Flipkart — an SSD is also worth considering for the redundancy drive specifically, since it has no moving parts and this drive's whole purpose is reliability
