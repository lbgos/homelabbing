# my homelab

Public archive of my homelab and self-hosting setup.

No AI slop here. Everything in this lab was built, tested, broken, fixed, and written down by me. AI helped with debugging, learning faster, and cleaning up the formatting, but the actual work is mine.

This is my first experience with self-hosting, picking parts for a server build, system management, etc. I've been daily-driving Linux for over a year, which made the whole process surprisingly smooth. The best part about all of this is that you control everything that you own (if you buy digital media) or download from torrents. You own your media - no more depending on predatory services like Netflix or Spotify to decide what you can watch or listen to.

AI has been a huge helper throughout this project, from debugging issues to discovering great iOS client apps. I've been using Gemini 3 Pro heavily and it boosted both my productivity and learning by a mile. (don't trust any AI. think before doing anything significant! HAVE YOUR OWN HEAD)

> Small update: I've started writing this repo more than a month ago, and I've changed my opinion on Gemini. It is far behind its compeditors. Codex 5.5 feels magical, but I think I will change my opinion many more times down the road.

I'll keep updating this repo as the setup changes. Service-specific folders and deeper writeups are coming later.

---

## Current stack

- **Hypervisor:** Proxmox VE
- **Containers:** LXC + Docker(Bad decision, explanation coming later)
- **VMs:** Home Assistant, Android/Redroid backup VM
- **Management:** Portainer, Homepage, Uptime Kuma, Netdata, Watchtower
- **Reverse proxy:** Nginx Proxy Manager
- **Auth:** Authelia
- **SSL/DNS:** Cloudflare + wildcard certs
- **Remote access:** Tailscale subnet routing
- **Notes:** Obsidian with CouchDB LiveSync + Memos
- **Network:** OpenWrt, VLANs, faster wired link between my main workstation and server

---

## Network layout

| Network | Purpose | Notes |
|---|---|---|
| Main / management | Proxmox, proxy, management tools, Home Assistant | Main trusted LAN |
| Media | Jellyfin, arr stack, music stack | Separated from the rest of the lab |
| Cloud / data | Immich, Nextcloud, Memos, CouchDB, Android backup VM | Storage-heavy services |
| IoT | Smart home devices | Isolated from the main LAN |
| Lab / testing | Experiments and intentionally isolated services | No access back into the trusted networks |

Tailscale runs on the Proxmox host and advertises the main internal networks, so I can reach the LXC/VM services from my tailnet without exposing every port publicly.

Public access goes through Cloudflare -> reverse proxy -> internal services. Most public services sit behind Authelia.

---

## Proxmox layout

| Type | Name | Main role |
|---|---|---|
| LXC | Media / Arr | Jellyfin, Seerr, Sonarr, Radarr, Prowlarr, qBittorrent, music stack |
| LXC | Cloud / Storage | Immich, Nextcloud, Memos, CouchDB, Fireshare, Degoog, Speedtest |
| VM | Home Assistant | Smart home, HACS, power monitoring, automations |
| VM | Android backup | Redroid + Google Photos ReVanced photo backup experiment |
| LXC | Proxy | Nginx Proxy Manager, Authelia, Let's Encrypt, Fail2ban |
| LXC | Management | Portainer, Homepage, Uptime Kuma, Netdata, Watchtower |

---

## Media / Arr container

### arrstack - movies, shows, and torrents

| Service | Description |
|---|---|
| [Jellyfin](https://jellyfin.org/) | Self-hosted Netflix-like media streaming |
| [Seerr](https://github.com/seerr-team/seerr) | Movie/series discovery and download requests |
| [Sonarr](https://sonarr.tv/) | Automated TV series management |
| [Radarr](https://radarr.video/) | Automated movie management |
| [Prowlarr](https://prowlarr.com/) | Indexer manager for the arr stack |
| [qBittorrent](https://www.qbittorrent.org/) | Torrent client |
| [FlareSolverr](https://github.com/FlareSolverr/FlareSolverr) | Cloudflare bypass helper for indexers |

Jellyfin uses Intel Quick Sync through the iGPU. It is honestly perfect for this kind of home server: low idle power, fast transcoding, and no need for a dedicated GPU.

### musicarr - music streaming and downloads

This is one of my favorite parts of the lab. Navidrome + good clients feels like a real streaming service, except I own the library and can organize it however I want. Just try doing your own music library, you will love it! talking from 200GB+ of flacs on my LAB.

| Service | Description |
|---|---|
| [Navidrome](https://www.navidrome.org/) | Self-hosted music streaming |
| [Lidarr](https://lidarr.audio/) | Automated music library management |
| [Slskd](https://github.com/slskd/slskd) | Soulseek client |
| [Deemix](https://deemix.app/) | Deezer downloader container |

I mostly use this with Nautiline on iOS and Feishin on desktop. Nautiline especially makes Navidrome feel polished enough that I stopped caring about Spotify.

---

## Cloud / Storage container

| Service | Description |
|---|---|
| [Immich](https://immich.app/) | Self-hosted Google Photos replacement |
| [Nextcloud](https://nextcloud.com/) | Self-hosted Google Drive replacement |
| [Memos](https://www.usememos.com/) | Lightweight notes and quick logging |
| [CouchDB](https://couchdb.apache.org/) | Obsidian LiveSync backend |
| [Fireshare](https://github.com/ShaneIsrael/fireshare) | Self-hosted clip/video sharing |
| Degoog | Self-hosted search / utility interface with custom plugins |
| [Speedtest Tracker](https://github.com/alexjustesen/speedtest-tracker) | Scheduled network speed tests |

Immich is the biggest reason this server already feels worth it. Fast photo backup, face/object search, mobile app, local storage, and no "pay monthly or lose convenience" nonsense.

Nextcloud is there for general file sync and remote storage. Memos became my quick-capture notes app because it is simple, fast, and does exactly what I need.

---

## Home Assistant VM

| Service | Description |
|---|---|
| [Home Assistant](https://www.home-assistant.io/) | Smart home dashboard and automation |

Current setup:

- HACS installed
- Power station integration connected through HACS
- A few smart lights connected
- Smart plug tracking server power draw
- Google Home still around for now, but I want to move more control into Home Assistant

The long-term goal is to move as much smart home control as possible into Home Assistant.

---

## Android backup VM

This VM runs [Redroid](https://github.com/remote-android/redroid-doc), basically Android 11 in Docker, for an offsite Immich photo backup experiment.

The short version:

1. Immich library is exported read-only over SMB.
2. Ubuntu VM mounts the Immich share.
3. Redroid gets that folder bind-mounted into Android media storage.
4. Android media scanner picks up the files.
5. Google Photos ReVanced backs them up.

| Component | Role |
|---|---|
| [Redroid](https://github.com/remote-android/redroid-doc) | Android 11 in Docker |
| [Google Photos ReVanced](https://github.com/Unofficial-Life/revanced-gphotos-build/releases/tag/9) | Modded Google Photos build |
| [GmsCore / microG](https://github.com/ReVanced/GmsCore) | Google account support |
| [scrcpy](https://github.com/Genymobile/scrcpy) | Screen mirroring for setup |

The important technical detail: the Immich folder has to be mounted into `/data/media/0/...`, not directly into `/storage/emulated/0/...`, because Android's storage layer wraps `/data/media`.

---

## Reverse proxy container

| Service | Description |
|---|---|
| [Nginx Proxy Manager](https://nginxproxymanager.com/) | Public reverse proxy and SSL termination |
| [Authelia](https://www.authelia.com/) | Authentication portal |
| Fail2ban | Basic brute-force/noise protection around exposed services |

This part moved from the roadmap into production. The lab now has:

- Custom domain
- Wildcard SSL certs via Cloudflare DNS challenge
- NPM routing public subdomains to internal services
- Authelia in front of most public services
- Fail2ban watching NPM logs

This made the setup feel much more like real infrastructure instead of a pile of local ports.

---

## Management container

| Service | Description |
|---|---|
| [Portainer](https://www.portainer.io/) | Docker management UI |
| [Homepage](https://gethomepage.dev/) | Services dashboard |
| [Uptime Kuma](https://github.com/louislam/uptime-kuma) | Uptime monitoring |
| [Netdata](https://www.netdata.cloud/) | System and Docker metrics |
| [Watchtower](https://containrrr.dev/watchtower/) | Automatic Docker image updates |

Homepage is the front page for the lab. Uptime Kuma tracks internal and public endpoints. Netdata gives quick visibility into system and Docker resource usage.

---

## Client apps I actually use

I've tested a lot of clients. These are the ones that stayed.

| App | Platform | Description |
|---|---|---|
| [ProxMobo](https://proxmobo.app/) | iOS | Great Proxmox manager |
| [Yomo](https://apps.apple.com/us/app/yomo-docker-portainer/id6479982236) | iOS | Docker / Portainer management |
| [MoeMemos](https://github.com/mudkipme/MoeMemos) | iOS | Native Memos client |
| [Feishin](https://github.com/jeffvli/feishin) | Desktop | Best desktop Navidrome player |
| [Nautiline](https://nautiline.app/) | iOS | Best iOS Navidrome client, worth the lifetime price |
| [Amperfy](https://github.com/BLeeEZ/amperfy) | iOS | Free Navidrome/Subsonic client, less polished than Nautiline |
| [Swiftfin](https://github.com/jellyfin/Swiftfin) | iOS | Best Jellyfin client on iOS |
| [Moonfin](https://github.com/nicholasgasior/moonfin) | Samsung TV | Seerr + Jellyfin for Tizen OS |
| [qRemote](https://apps.apple.com/us/app/qremote-for-qbittorrent/id6756276747) | iOS | Simple qBittorrent client |
| [Ruddarr](https://github.com/ruddarr/app) | iOS | Radarr / Sonarr management |
| [Pocket](https://apps.apple.com/in/app/pocket-for-seerr/id6746105104) | iOS | Seerr client |

Nautiline is the easiest recommendation here. Just buy it once and forget about Spotify if you already have your own music library.

---

## Server specs

| Part | Notes |
|---|---|
| CPU | i5-12400 |
| GPU | UHD Graphics 730 |
| RAM | 32GB DDR4 |
| Boot drive | NVMe SSD 512GB |
| Data drive | IronWolf 10TB NAS |
| Case | Reused desktop case |
| Network | OpenWrt-based LAN/Wi-Fi, faster wired link for large local transfers(2.5G) |

This is probably more powerful than a first homelab needs, but I wanted headroom. It idles low enough that I do not worry about it, and it should pay for itself by replacing cloud storage, streaming subscriptions, and other services I do not want to rely on.

---

## Roadmap

- [x] Reverse proxy
- [x] Custom domain
- [x] SSL with wildcard certificates
- [x] Authelia auth gateway
- [x] Separate management container
- [x] Uptime monitoring
- [x] Basic metrics
- [x] VLANs for media/cloud/IoT
- [x] Faster local wired transfers
- [ ] Service-specific folders in this repo
- [ ] Better public documentation for each stack
- [ ] Proper backup strategy
- [ ] More Home Assistant automations
- [ ] Maybe faster networking later, if I find a real reason

---

## Notes

This repo is a public archive, not a copy-paste production guide. Some parts are opinionated, some are messy, and some exist because I wanted to learn by building instead of watching another 40-minute tutorial.

Do not blindly trust AI, tutorials, or even this README. Think before running commands on your own server too, I've been screwed up so many times because of blindly listening to AI. Have your own head.

<p align="center">
  <i>AI helped with refining and formatting this README, but the setup and notes are mine.</i>
</p>
