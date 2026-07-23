# my homelab

Public archive of my homelab and self-hosting setup.

No AI slop here. Everything in this lab was built, tested, broken, fixed, and written down by me. AI helped with debugging, learning faster, and cleaning up the formatting, but the actual work is mine.

This is my first experience with self-hosting, picking parts for a server build, system management, etc. I've been daily-driving Linux for over a year, which made the whole process surprisingly smooth. The best part about all of this is that you control everything that you own (if you buy digital media) or download from torrents. You own your media - no more depending on predatory services like Netflix or Spotify to decide what you can watch or listen to.

AI has been a huge helper throughout this project, from debugging issues to discovering great iOS client apps. I've been using Gemini 3 Pro heavily and it boosted both my productivity and learning by a mile. (don't trust any AI. think before doing anything significant! HAVE YOUR OWN HEAD)

> Small update: I've started writing this repo more than a month ago, and I've changed my opinion on Gemini. It is far behind its compeditors. Codex 5.5 feels magical, but I think I will change my opinion many more times down the road.

> Bigger update: the lab grew from one server into **two Proxmox nodes**. The first one stayed the "life" server (media, cloud, smart home), and the second one became a dedicated AI machine with a GPU for local LLM inference and agents. Also: proper VPN gateway, GPU passthrough gaming VM, backups, and Wake-on-LAN so the servers don't have to run 24/7.

I'll keep updating this repo as the setup changes. Service-specific folders and deeper writeups are coming later.

---

## Current stack

- **Hypervisors:** 2x Proxmox VE nodes (PVE1 = media/cloud/services, PVE2 = AI/GPU)
- **Containers:** LXC + Docker(Bad decision, explanation coming later)
- **VMs:** Home Assistant, Android/Redroid backup VM, Windows gaming VM, Linux LLM VM, privacy desktop
- **Local AI:** llama.cpp + llama-swap + Ollama + Open WebUI on an RTX 3090, distributed RPC inference across both nodes
- **Agents:** hermes-agent (Discord bot + OpenAI-compatible API), coding agents living in dedicated containers
- **Management:** Portainer, Homepage, Uptime Kuma, Netdata, Watchtower
- **Reverse proxy:** Nginx Proxy Manager
- **Auth:** Authelia
- **SSL/DNS:** Cloudflare + wildcard certs
- **Remote access:** Tailscale subnet routing + self-hosted WireGuard gateway with a Mullvad uplink
- **Notes:** Obsidian with CouchDB LiveSync + Memos
- **Network:** OpenWrt (2x routers, 802.11r fast roaming), VLANs, dedicated 2.5G links
- **Power:** Wake-on-LAN from the router on a schedule - servers sleep at night

---

## Network layout

| Network | Purpose | Notes |
|---|---|---|
| Main / management | Proxmox nodes, proxy, management tools, Home Assistant | Main trusted LAN |
| Media | Jellyfin, arr stack, music stack | Separated from the rest of the lab |
| Cloud / data | Immich, Nextcloud, Memos, CouchDB, AI containers, LLM VM | Storage-heavy and AI services |
| IoT | Smart home devices | Isolated from the main LAN, 2.4GHz only |
| Honeypot | Guests and untrusted devices | WAN-only, strictly isolated from everything internal |
| Lab / testing | Experiments and intentionally isolated services | No access back into the trusted networks |

Two OpenWrt routers (main + dumb AP) with a VLAN trunk between them, 802.11r fast roaming on 5GHz, and adblock on the router. The two Proxmox nodes are connected to each other with a direct 2.5G point-to-point link - originally for fast container migration, now it carries distributed LLM inference traffic between the GPUs.

Remote access is layered:

- **Tailscale** runs on PVE1 and advertises the internal networks, so I can reach LXC/VM services from my tailnet without exposing every port publicly.
- **WireGuard gateway** (see below) for devices where I want LAN access *and* all internet traffic pushed through Mullvad.

Public access goes through Cloudflare -> reverse proxy -> internal services. Most public services sit behind Authelia.

---

## Proxmox layout

### PVE1 - the "life" node

| Type | Name | Main role |
|---|---|---|
| LXC | Media / Arr | Jellyfin, Seerr, Sonarr, Radarr, Prowlarr, qBittorrent, music stack |
| LXC | Cloud / Storage | Immich, Nextcloud, Memos, CouchDB, Fireshare, Degoog, Speedtest |
| VM | Home Assistant | Smart home, HACS, power monitoring, automations |
| VM | Android backup | Redroid + Google Photos ReVanced photo backup experiment |
| LXC | Proxy | Nginx Proxy Manager, Authelia, Let's Encrypt, Fail2ban |
| LXC | Management | Portainer, Homepage, Uptime Kuma, Netdata, Watchtower |
| LXC | Static web | Tiny Alpine nginx container hosting a friend's static site behind Cloudflare |
| LXC | WireGuard gateway | Local WireGuard server with a Mullvad uplink |
| VM | Privacy desktop | Artix + KDE, LUKS-encrypted, all traffic through Mullvad |
| VM | RPC node | RTX 3080 passthrough, llama.cpp RPC worker for distributed inference |

### PVE2 - the AI node

| Type | Name | Main role |
|---|---|---|
| VM | LLM GPU | RTX 3090 passthrough, llama-swap + llama.cpp + Ollama + Open WebUI |
| LXC | Hermes | hermes-agent: Discord bot + OpenAI-compatible agent API |
| LXC | Agents / tools | Headless coding agent, self-hosted chat frontend, notes web reader |
| LXC | Knowledge | Knowledge vault + CLI agents (Claude Code / Codex) working inside it |
| VM | Windows gaming | GPU passthrough + Sunshine/Moonlight streaming |
| VM | Security lab | Debian workstation for authorized security assessment (HTB/CTF/lab) |

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

Nextcloud is there for general file sync and remote storage. Memos became my quick-capture notes app because it is simple, fast, and does exactly what I need. Degoog also doubles as the web-search backend for my local AI agents.

---

## Local AI stack

The whole reason PVE2 exists. An RTX 3090 is passed through into a dedicated Ubuntu VM, and the RTX 3080 on PVE1 joins in over the 2.5G link as a second inference node.

| Component | Role |
|---|---|
| [llama.cpp](https://github.com/ggml-org/llama.cpp) | Inference engine |
| [llama-swap](https://github.com/mostlygeek/llama-swap) | On-demand model loading/unloading behind one OpenAI-compatible API |
| [Ollama](https://ollama.com/) | Fallback runtime + model blob storage |
| [Open WebUI](https://github.com/open-webui/open-webui) | Chat UI for the local models |
| [hermes-agent](https://github.com/NousResearch/hermes-agent) | Agent harness: Discord bot + OpenAI-compatible full agent-loop API |

How it works in practice:

- **llama-swap** loads models on request and evicts them after an idle timeout, so the GPU is free when nothing is running. Cold start is ~30 seconds, which is fine for a homelab.
- Several quantized **27-35B models** (dense reasoning models + one MoE) live behind one endpoint. Each has its own tuned launch wrapper (context size, parallel slots, sampling flags).
- **Distributed inference:** dense models can spill onto the 3080 in PVE1 through llama.cpp's RPC backend over the dedicated 2.5G link. Lesson learned: MoE models over RPC = broken routing, keep those on a single GPU.
- **hermes-agent** runs in its own LXC and talks to the LLM VM. OpenAI-compatible API platform with a full agent loop (tools, web search through Degoog, a hardened Firefox-based browser, custom personalities). Other services consume the agent through that API.
- A separate container hosts a **headless coding agent**, a **self-hosted chat frontend** that can dispatch to Claude Code / Codex CLI as virtual models, and a small **web reader** for my notes vault backed by the local models.
- Another container is the canonical home of my **knowledge vault**, where CLI agents work on the notes directly in tmux sessions.

---

## Security lab VM

A dedicated Debian workstation VM for **authorized** security assessment work - Hack The Box, Academy labs, CTF events within event rules, and systems I own or am explicitly permitted to test. Standard toolage (nmap, OpenVPN for lab access, etc.).

It also hosts a tool I'm building myself, **Blackglass** - a local-first, modular security-assessment platform (TypeScript monolith: Fastify API + SQLite + a React workbench). The design principles are the interesting part and where most of the learning is:

- **One engagement/scope/policy/evidence model** across all work modes, so nothing runs without an active engagement, an in-scope target, and an audit record.
- **Capability tiers** (passive OSINT → safe active → active assessment → intrusive) with destructive/DoS actions prohibited by policy.
- **Fail-closed** module system: a tool can only run if its submitted contract exactly matches the registered immutable contract.
- **Local-first data handling**: no telemetry, no third-party posting, secrets never in source or logs, append-only audit trail. Services default to loopback; LAN exposure is an explicit, firewalled opt-in.
- The local AI models can plan, correlate, and draft, but they never grant scope, approve actions, or bypass policy - a locally served model is still treated as a separate data boundary.

This is very much a work in progress and a learning project, not a finished product. But building the *policy and authorization* layer of a security tool taught me more about doing this stuff safely than any tutorial would have.

---

## WireGuard gateway

A small unprivileged LXC that is a proper VPN gateway instead of "Mullvad app on every device":

- Clients connect to a local WireGuard server.
- All client internet traffic exits **only** through a Mullvad WireGuard uplink (nftables forward policy is `drop`, so there is no way to leak out the LAN uplink).
- Clients still reach the internal main/media/cloud networks through NAT.
- The IoT VLAN is not reachable at all.
- A helper script generates new client peers + configs in one command.

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

This made the setup feel much more like real infrastructure instead of a pile of local ports. The proxy also serves a friend's public static website from a 256MB Alpine container - hosting someone else's site on your own infra is a great milestone feeling.

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

## Backups

- LXC backups go to the data pool on PVE1.
- An external USB HDD is a single-disk **ZFS pool** (lz4 compression, auto-import on boot) used as a cold-ish backup target via rsync. Not 3-2-1 yet, but way better than nothing.

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
| [Moonlight](https://moonlight-stream.org/) | Everywhere | Game streaming client for the Windows VM |
| [qRemote](https://apps.apple.com/us/app/qremote-for-qbittorrent/id6756276747) | iOS | Simple qBittorrent client |
| [Ruddarr](https://github.com/ruddarr/app) | iOS | Radarr / Sonarr management |
| [Pocket](https://apps.apple.com/in/app/pocket-for-seerr/id6746105104) | iOS | Seerr client |

Nautiline is the easiest recommendation here. Just buy it once and forget about Spotify if you already have your own music library.

---

## Server specs

### PVE1 - media / cloud / services

| Part | Notes |
|---|---|
| CPU | i5-12400 |
| GPU | UHD Graphics 730 (Quick Sync transcoding) |
| GPU 2 | RTX 3080 (passthrough, llama.cpp RPC inference node) |
| RAM | 32GB DDR4 |
| Boot drive | NVMe SSD 512GB |
| Data drive | IronWolf 10TB NAS |
| Backup drive | 4TB external USB HDD (ZFS pool) |

### PVE2 - AI node

| Part | Notes |
|---|---|
| CPU | i7-10700F |
| GPU | RTX 3090 (passthrough, shared between the LLM VM and gaming VM) |
| RAM | 32GB DDR4 |
| Boot drive | NVMe SSD 512GB |
| VM drive | NVMe SSD 1TB (LVM-thin) |
| PSU | 1200W Platinum (GPU headroom) |

Network: OpenWrt-based LAN/Wi-Fi, 2.5G wired links for the workstation and between the two nodes.

This is probably more powerful than a first homelab needs, but I wanted headroom. It idles low enough (and sleeps at night thanks to WoL) that I do not worry about it, and it should pay for itself by replacing cloud storage, streaming subscriptions, AI API bills, and other services I do not want to rely on.

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
- [x] Second Proxmox node for AI
- [x] Local LLM inference with on-demand model loading
- [x] Distributed inference across two GPUs
- [x] Self-hosted agent platform (Discord + API)
- [x] WireGuard gateway with Mullvad uplink
- [x] GPU passthrough gaming VM with streaming
- [x] Dedicated security-lab VM (HTB/CTF/authorized work)
- [x] Wake-on-LAN power schedule
- [x] Basic backup drive (ZFS on external HDD)
- [ ] Fix WoL on the second node (BIOS ErP/Deep Sleep)
- [ ] Proper 3-2-1 backup strategy
- [ ] Service-specific folders in this repo
- [ ] Better public documentation for each stack
- [ ] More Home Assistant automations
- [ ] Maybe faster networking later, if I find a real reason

---

## Notes

This repo is a public archive, not a copy-paste production guide. Some parts are opinionated, some are messy, and some exist because I wanted to learn by building instead of watching another 40-minute tutorial.

Do not blindly trust AI, tutorials, or even this README. Think before running commands on your own server too, I've been screwed up so many times because of blindly listening to AI. Have your own head.

<p align="center">
  <i>AI helped with refining and formatting this README, but the setup and notes are mine.</i>
</p>
