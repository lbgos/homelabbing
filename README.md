# my homelab

Public archive of my homelab and self-hosting setup.

No AI slop here. Everything in this lab was built, tested, broken, fixed, and written down by me. AI helped with debugging, learning faster, and cleaning up the formatting, but the actual work is mine.

This is my first experience with self-hosting, picking parts for a server build, system management, etc. I've been daily-driving Linux for over a year, which made the whole process surprisingly smooth. The best part about all of this is that you control everything that you own (if you buy digital media) or download from torrents. You own your media. No more depending on predatory services like Netflix or Spotify to decide what you can watch or listen to.

AI has been a huge helper throughout this project, from debugging issues to discovering great iOS client apps. I've been using Gemini 3 Pro heavily and it boosted both my productivity and learning by a mile. (don't trust any AI. think before doing anything significant! HAVE YOUR OWN HEAD)

> Small update: I started writing this repo more than a month ago, and I've changed my opinion on Gemini. It is far behind its competitors. Codex 5.5 feels magical, but I think I will change my opinion many more times down the road.

> Bigger update: the lab grew from one server into **two Proxmox nodes**. The first one stayed the "life" server (media, cloud, smart home), and the second one became a dedicated AI machine with a GPU for local LLM inference and agents. I also added a proper VPN gateway, a GPU passthrough gaming VM, backups, and Wake-on-LAN so the servers don't have to run 24/7.

> September 2026 update: public websites moved off the home connection to a free Oracle Cloud ARM VM. I started encrypted offsite backups to Google Drive. Home Assistant got a local voice assistant built from an old OnePlus 5 and the RTX 3080. PVE2 now runs more agents than I can keep track of, which is why this README was two months behind.

I'll keep updating this repo as the setup changes.

---

## Current stack

- **Hypervisors:** 2x Proxmox VE 9 nodes (PVE1 = media/cloud/services, PVE2 = AI/GPU)
- **Cloud:** Oracle Cloud Free Tier ARM VM for public websites
- **Containers:** LXC + Docker (bad decision, explanation coming later)
- **VMs:** Home Assistant, Windows gaming, Linux LLM VM (RTX 3090), RTX 3080 VM (voice + small models), security lab, Android/Redroid backup, privacy desktop
- **Local AI:** llama.cpp + llama-swap + Ollama + Open WebUI on an RTX 3090, plus an RTX 3080 on PVE1 for voice and small models
- **Agents:** T3 Code as the frontend, OpenCode over the API, hermes-agent (Discord bot + OpenAI-compatible API), Claude Code / Codex living in dedicated containers
- **Management:** Portainer, Homepage, Uptime Kuma, Netdata, Watchtower, Termix, a Telegram bot for lab control
- **Reverse proxy:** Nginx Proxy Manager (one at home, one on Oracle)
- **Auth:** Authelia
- **SSL/DNS:** Cloudflare + wildcard certs
- **Remote access:** Tailscale subnet routing + self-hosted WireGuard gateway with a Mullvad uplink
- **Notes:** Obsidian with CouchDB LiveSync + Memos, plus a knowledge vault that agents maintain
- **Backups:** USB ZFS pool, encrypted Google Drive copy in progress
- **Network:** OpenWrt (2x routers, 802.11r fast roaming), VLANs, dedicated 2.5G links
- **Power:** Wake-on-LAN from the router on a schedule, servers sleep at night

---

## Network layout

| Network | Purpose | Notes |
|---|---|---|
| Main / management | Proxmox nodes, proxy, management tools, Home Assistant | Main trusted LAN |
| Media | Jellyfin, arr stack, music stack | Separated from the rest of the lab |
| Cloud / data | Immich, Nextcloud, Memos, CouchDB, AI containers, LLM VMs | Storage-heavy and AI services |
| IoT | Smart home devices | Isolated from the main LAN, 2.4GHz only |
| Honeypot | Guests and untrusted devices | WAN-only, strictly isolated from everything internal |
| Lab / testing | Experiments and intentionally isolated services | No access back into the trusted networks |

Two OpenWrt routers (main + dumb AP) with a VLAN trunk between them, 802.11r fast roaming on 5GHz, and adblock on the router. The two Proxmox nodes are connected to each other with a direct 2.5G point-to-point link. It was originally for fast container migration, then I used it to pool both GPUs over llama.cpp RPC (results below).

Remote access is layered:

- **Tailscale** runs on PVE1 and advertises the internal networks, so I can reach LXC/VM services from my tailnet without exposing every port publicly. The Oracle VM is on the same tailnet.
- **WireGuard gateway** (see below) for devices where I want LAN access *and* all internet traffic pushed through Mullvad.

Public access goes through Cloudflare -> reverse proxy -> internal services. Most public services sit behind Authelia. Static websites no longer touch the home connection at all, they live on Oracle.

---

## Proxmox layout

### PVE1, the "life" node

| Type | Name | Main role |
|---|---|---|
| LXC | Media / Arr | Jellyfin, Seerr, Sonarr, Radarr, Prowlarr, qBittorrent, music stack |
| LXC | Cloud / Storage | Immich, Nextcloud, Memos, CouchDB, Fireshare, Degoog, Speedtest, Browserless |
| VM | Home Assistant | Smart home, HACS, power monitoring, voice pipeline |
| LXC | Proxy | Nginx Proxy Manager, Authelia, Let's Encrypt, Fail2ban |
| LXC | Management | Portainer, Homepage, Uptime Kuma, Netdata, Termix, lab Telegram bot |
| VM | 3080 node | RTX 3080 passthrough: STT/TTS and a small Qwen for the voice assistant, Bonsai 2 27B on demand, JupyterLab |
| LXC | Static web | Tiny Alpine nginx container. The sites moved to Oracle, this is the old copy |
| LXC | WireGuard gateway | Local WireGuard server with a Mullvad uplink |
| VM | Android backup | Redroid + Google Photos ReVanced photo backup experiment |
| VM | Privacy desktop | Artix + KDE, LUKS-encrypted, all traffic through Mullvad |

### PVE2, the AI node

| Type | Name | Main role |
|---|---|---|
| VM | LLM GPU | RTX 3090 passthrough, llama-swap + llama.cpp + Ollama + Open WebUI |
| LXC | Hermes | hermes-agent: Discord bot, OpenAI-compatible agent API, music tools for Home Assistant |
| LXC | Agents / tools | T3 Code, Playwright MCP, Stonehush dev, benchmark UI |
| LXC | Knowledge | Knowledge vault, Claude Code / Codex, CLIProxyAPI, Aviary reader, my personal site, study app |
| VM | Windows gaming | Same RTX 3090 + Sunshine/Moonlight streaming |
| VM | Security lab | Debian VM for HTB, CTFs and my own lab, runs Stonehush |

The 3090 can only belong to one VM at a time. A Proxmox hookscript on both GPU VMs handles that: starting the gaming VM shuts down the LLM VM, and the other way around. No more "why won't this VM boot" moments.

---

## Media / Arr container

### arrstack: movies, shows, and torrents

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

### musicarr: music streaming and downloads

This is one of my favorite parts of the lab. Navidrome + good clients feels like a real streaming service, except I own the library and can organize it however I want. Try building your own music library, you will love it. Speaking from 200GB+ of FLACs in my lab.

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
| [Degoog](https://github.com/fccview/degoog) | Self-hosted search with custom plugins |
| [Speedtest Tracker](https://github.com/alexjustesen/speedtest-tracker) | Scheduled network speed tests |
| [Browserless](https://github.com/browserless/browserless) | Headless Chromium for scraping and agents |

Immich is the biggest reason this server already feels worth it. Fast photo backup, face/object search, mobile app, local storage, and no "pay monthly or lose convenience" nonsense.

Nextcloud is there for general file sync and remote storage. Memos became my quick-capture notes app because it is simple, fast, and does exactly what I need.

Degoog is the web-search backend for all my local agents. It also has a small bridge service next to it that answers search queries with a local model, grounded on the search results, so I get AI answers in my own search engine without sending anything to a cloud API.

---

## Local AI stack

The whole reason PVE2 exists. An RTX 3090 is passed through into a dedicated Ubuntu VM. The RTX 3080 on PVE1 lives in its own VM and handles voice and small models.

| Component | Role |
|---|---|
| [llama.cpp](https://github.com/ggml-org/llama.cpp) | Inference engine |
| [llama-swap](https://github.com/mostlygeek/llama-swap) | Model switching behind one OpenAI-compatible API |
| [Ollama](https://ollama.com/) | Fallback runtime + model blob storage |
| [Open WebUI](https://github.com/open-webui/open-webui) | Chat UI for the local models |
| [OpenCode](https://opencode.ai/) | Coding agent, talks to the models over the API |
| [T3 Code](https://github.com/pingdotgg/t3code) | My main agent frontend, runs headless in the agents container |
| [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) | Serves my GPT and Gemini subscriptions as one OpenAI-compatible API |
| [hermes-agent](https://github.com/NousResearch/hermes-agent) | Agent harness: Discord bot + OpenAI-compatible full agent-loop API |

Models I actually use, all on the 3090 alone:

| Model | Notes |
|---|---|
| Qwen3.8 27B (Unsloth Q4_K_XL) | Daily driver. Vision, MTP speculative decoding, 192k context. Preloaded when the VM boots |
| Qwen3.8 27B HauhauCS (uncensored) | Same setup, low and xhigh reasoning variants |

How it works in practice:

- **llama-swap** puts every model behind one endpoint and swaps them on request. Each model is a small launch wrapper with its own context size, KV cache type, reasoning effort and sampling flags. Models stay loaded until another one is requested, so the daily driver answers instantly.
- 192k context of a 27B model fits into 24GB because the KV cache is quantized to 5/4 bits, and MTP drafting on the same GPU speeds up generation.
- **OpenCode** connects straight to llama-swap for local models and to CLIProxyAPI for GPT and Gemini. Cheap stuff runs on Gemini Flash-Lite, heavy thinking goes to GPT subagents, private stuff stays on the 3090.
- The 3080 VM runs Ternary Bonsai 2 27B (2.13 bits per weight, 6.8GB) at ~62 t/s with 128k context. A small supervisor stops the voice services when Bonsai is called and brings them back after 20 minutes idle, since both don't fit into 10GB.
- **hermes-agent** runs in its own LXC and talks to the LLM VM. Discord for me, an OpenAI-compatible API with a full agent loop for everything else (tools, web search through Degoog, a Camoufox browser, custom personalities).
- The knowledge container is the canonical home of my notes vault. CLI agents work on it directly in tmux sessions, and **Aviary** is a small web reader for the vault backed by the local models.

### Pooling the 3090 and 3080 over RPC

I tried running Qwen3.8 27B across both GPUs (34GB of VRAM together) with llama.cpp RPC over the 2.5G link, so I could fit Q8 weights or two agents at once. It needed a KV-cache-quant fork of llama.cpp, and two fixes in that fork before its RPC backend even worked.

| Setup | Context | Decode |
|---|---|---|
| Q4, 3090 alone | 262k | 31.7 t/s |
| Q4, 3090 alone + MTP | 262k | ~48 t/s |
| Q8, pool | 160k | 23.4 t/s |
| Q8, pool + MTP | 96k | 41 t/s |
| Q4, pool, 2 slots + MTP | 2x 192k | ~53 t/s on a 9k prompt |

On short prompts the pool was fast, and MTP roughly doubled decode speed. On big contexts the speed fell apart, because RPC has to push the KV cache between the cards over the 2.5G link, and the more context there is, the more it has to move. MoE models over RPC were even worse, expert routing broke. So the pool is gone and the daily model runs on the 3090 alone.

---

## Security lab VM

Debian 13 VM for Hack The Box, Academy, CTFs and my own lab. nmap and OpenVPN for the HTB connection, nothing fancy. Ingress is nftables default-drop: SSH, plus the Stonehush UI from the cloud VLAN only.

**Stonehush** (used to be called Blackglass) is my own security workspace. I got tired of juggling terminals, notes and screenshots between tools, so it keeps targets, scans, evidence, notes and reports together per engagement:

- nmap service discovery, HTTP probing and ffuf path discovery through a separate runner
- saved runs, output and evidence per engagement
- Markdown notes, findings and reports, exported as Markdown or JSON
- a saved scope that warns before a scan leaves it
- optional AI explanations of evidence through any OpenAI-compatible endpoint, so it can use my local models

TypeScript, React, Vite, Tailwind, Fastify and SQLite. Development happens in the agents container, this VM runs it against lab targets. Still early: single user, some screens unfinished, runner setup is manual.

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
- Local voice assistant (below)

### Voice assistant

I planned to buy a Voice PE satellite. Instead I rooted an old OnePlus 5 with LineageOS and made it the satellite. Cheaper, and it already had a mic, a speaker and a headphone jack.

```text
OnePlus 5 (mic, wake word, VLC)
  -> Home Assistant voice pipeline
    |-- STT/TTS: faster-whisper + TTS on the RTX 3080 VM
    |-- lights, pause/next: local intents, never reach an LLM
    |-- questions: voice harness -> Gemini Flash-Lite, local Qwen fallback on the 3080
    |-- music: Hermes music tools -> Navidrome -> small bridge on the phone -> VLC
```

- Simple commands never reach a model. Pause, skip and lights by name are handled by Home Assistant intents.
- Everything else goes to a small harness addon. Gemini Flash-Lite answers short prompts in under a second and costs cents a month on the free tier. If it stays quiet for 20 seconds, a small local Qwen on the 3080 takes over.
- The music part is my favorite. Hermes keeps a read-only copy of the Navidrome catalog (~5,400 tracks) and can play a track, an album, or a mood queue. "Play some jazz" pulls Getz and Sinatra from my own library.
- A Magisk module on the phone runs a tiny HTTP bridge that drives VLC and switches output between the Bluetooth speaker and the amplifier. It pauses music while the assistant talks.

Google Home is basically retired now.

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

The lab has:

- Custom domains
- Wildcard SSL certs via Cloudflare DNS challenge
- NPM routing public subdomains to internal services
- Authelia in front of most public services
- Fail2ban watching NPM logs

This made the setup feel much more like real infrastructure instead of a pile of local ports.

---

## Oracle Cloud VM

Static websites and small public tools don't need to run on my home connection, so I moved them to an Oracle Cloud Free Tier VM (Ampere ARM, 2 cores, 12GB RAM, free).

- Its own Nginx Proxy Manager + Authelia in Docker, same Cloudflare setup as at home.
- A friend's website, my personal site, and my drafts/file-upload services live here now. Hosting someone else's site on your own infra is still a great milestone feeling.
- NPM admin is not public. It is reachable only through Tailscale.
- Backups are pulled, not pushed: PVE1 pulls one compressed archive every day through a dedicated account that can only run the export script. Oracle can't open a shell on anything at home. The last 30 archives are kept.

---

## Management container

| Service | Description |
|---|---|
| [Portainer](https://www.portainer.io/) | Docker management UI |
| [Homepage](https://gethomepage.dev/) | Services dashboard |
| [Uptime Kuma](https://github.com/louislam/uptime-kuma) | Uptime monitoring |
| [Netdata](https://www.netdata.cloud/) | System and Docker metrics |
| [Watchtower](https://github.com/nicholas-fedor/watchtower) | Automatic Docker image updates (maintained fork, on every Docker container) |
| [Termix](https://github.com/LukeGus/Termix) | Web SSH/RDP client |

Homepage is the front page for the lab. Uptime Kuma tracks internal and public endpoints. Netdata gives quick visibility into system and Docker resource usage. A small Telegram bot runs next to them for checking and controlling the lab from my phone.

---

## Backups

- An external USB HDD is a single-disk **ZFS pool** (lz4 compression, auto-import on boot) used as the local backup target.
- Weekly vzdump of both nodes lands on that pool. PVE2 writes to it over NFS through PVE1.
- A script on PVE1 uploads guest archives, database dumps, and Immich/Nextcloud/Memos files to Google Drive through rclone crypt, so Google only sees encrypted blobs. It is not done: a wrong path in the script meant guest archives weren't uploading, and I haven't confirmed one complete offsite set yet.
- The Oracle VM is pulled daily into the same pool (see above).

Closer to 3-2-1 than before, but not there. The USB disk drops off the bus from time to time, and a backup pool that silently disappears is not much of a backup.

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

### PVE1: media / cloud / services

| Part | Notes |
|---|---|
| CPU | i5-12400 |
| Motherboard | Asus Prime B660M-K D4 |
| GPU | UHD Graphics 730 (Quick Sync transcoding) |
| GPU 2 | RTX 3080 (passthrough, voice + small models) |
| RAM | 32GB DDR4-3200 |
| Boot drive | NVMe SSD 512GB |
| Data drive | WD Red Pro 10TB (ZFS) |
| Backup drive | 4TB external USB HDD (ZFS pool) |
| PSU | Chieftec 750W |

### PVE2: AI node

| Part | Notes |
|---|---|
| CPU | i7-10700F |
| Motherboard | ASRock B460M Pro4 |
| GPU | RTX 3090 (passthrough, shared between the LLM VM and gaming VM) |
| RAM | 32GB DDR4 |
| Boot drive | NVMe SSD 512GB |
| VM drive | NVMe SSD 1TB (LVM-thin) |
| PSU | EVGA 1200W Platinum (GPU headroom) |

The two GPUs swapped places at some point: the 3090 went to the AI node, the 3080 went to PVE1, and the 1TB SSD followed the VMs to PVE2.

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
- [x] Public websites moved to Oracle Cloud
- [x] Local voice assistant for Home Assistant
- [ ] Fix WoL on the second node (BIOS ErP/Deep Sleep)
- [ ] Replace or fix the flaky USB backup disk
- [ ] Finish encrypted offsite backups (Google Drive via rclone crypt)
- [ ] Proper 3-2-1 backup strategy
- [ ] Service-specific folders in this repo
- [ ] Better public documentation for each stack
- [ ] More Home Assistant automations
- [ ] Maybe faster networking later, if I find a real reason

---

## Notes

This repo is a public archive, not a copy-paste production guide. Some parts are opinionated, some are messy, and some exist because I wanted to learn by building instead of watching another 40-minute tutorial.

Do not blindly trust AI, tutorials, or even this README. Think before running commands on your own server too, I've screwed up so many times by blindly listening to AI. Have your own head.

<p align="center">
  <i>AI helped with refining and formatting this README, but the setup and notes are mine.</i>
</p>
