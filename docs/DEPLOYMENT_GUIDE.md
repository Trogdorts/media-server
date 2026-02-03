# Deployment & Service Breakdown

This document summarizes each service directory and provides deployment-script building blocks, local testing steps, and a step-by-step guide (starting with PyCharm setup) for building a deployment script for the media-server stack.

## Repository layout highlights

- Root `compose.yaml` aggregates every service via `include:` and defines the `external` and `internal` networks.
- `compose.extend.yaml` provides reusable `extends:` snippets that wire in Cloudflared waits, GPU variants, and Postgres/VPN dependency toggles.
- `.env` (copied from `.env.example`) is the canonical configuration source for profiles, images, credentials, and media paths.

> Note: The “research” in this guide is based on the configuration files in this repository (compose files, scripts, and README). External vendor documentation is not embedded here, so if you need links or provider-specific setup steps (Cloudflare, NordVPN, Plex, etc.), we should add them in a follow-up pass.

## Step 0: Set up PyCharm for this repo

1. **Open the repo** in PyCharm using the repo root (`/workspace/media-server`).
2. **Configure a Python interpreter** (recommended for scripts like `clear-overlay.py` and quick tooling):
   - Create a new virtual environment (venv) in the project.
   - Install basic tooling if needed: `pip install python-dotenv plexapi schedule`.
3. **Add a Docker CLI run configuration** (optional but useful):
   - Point it to the repo root and use `docker compose` commands for quick start/stop.
4. **Mark folders**:
   - Mark `docs/` as “Documentation” to keep navigation clean.
5. **Create a “.env local” file** (optional):
   - Keep a local-only `.env` with minimal profiles to prevent unintentionally booting the full stack.

## Step 1: Understand what each service needs (inputs, dependencies, auto-config)

The service requirements below are derived from each service’s `compose.yaml` and the shared `.env.example`. Key points:

- **Profiles**: each service has a `*_PROFILE` default in `.env.example`. Setting it to `"disabled"` prevents auto-start.
- **Volumes / paths**: media-heavy services expect `PATH_*` values to be set in `.env`.
- **Dependencies**: most UI services depend on Traefik; some services require Postgres or VPN gating via `compose.extend.yaml`.
- **Auto-configuration**: aside from .env-driven values, very few services are auto-configured; most expect you to configure inside the app UI after first boot.

## Directory-by-directory service breakdown

Each row lists the service directory, service name, default image and profile, exposed ports (if any), and runtime dependencies. Use this to decide which services to enable and which credentials/paths you must supply in `.env`.

| Service dir | Service | Image | Profiles | Ports | Depends on |
| --- | --- | --- | --- | --- | --- |
| `adguardhome/compose.yaml` | `adguardhome` | `${ADGUARDHOME_IMAGE:-adguard/adguardhome}:${ADGUARDHOME_IMAGE_TAG:-latest}` | `["${ADGUARDHOME_PROFILE:-network}"]` | — | traefik |
| `audiobookshelf/compose.yaml` | `audiobookshelf` | `${AUDIOBOOKSHELF_IMAGE:-ghcr.io/advplyr/audiobookshelf}:${AUDIOBOOKSHELF_IMAGE_TAG:-latest}` | `["${AUDIOBOOKSHELF_PROFILE:-library}"]` | — | traefik |
| `bazarr/compose.yaml` | `bazarr` | `${BAZARR_IMAGE:-lscr.io/linuxserver/bazarr}:${BAZARR_IMAGE_TAG:-latest}` | `["${BAZARR_PROFILE:-media}"]` | — | traefik |
| `beszel/compose.yaml` | `beszel` | `${BESZEL_IMAGE:-henrygd/beszel}:${BESZEL_IMAGE_TAG:-latest}` | `["${BESZEL_PROFILE:-information}"]` | — | traefik, beszel-agent |
| `beszel/compose.yaml` | `beszel-agent` | `${BESZEL_AGENT_IMAGE:-henrygd/beszel-agent-nvidia}:${BESZEL_AGENT_IMAGE_TAG:-latest}` | `["${BESZEL_AGENT_PROFILE:-information}"]` | — | socket-proxy |
| `chaptarr/compose.yaml` | `chaptarr` | `${CHAPTARR_IMAGE:-robertlordhood/chaptarr}:${CHAPTARR_IMAGE_TAG:-latest}` | `["${CHAPTARR_PROFILE:-media}"]` | — | traefik |
| `cleanuparr/compose.yaml` | `cleanuparr` | `${CLEANUPARR_IMAGE:-ghcr.io/cleanuparr/cleanuparr}:${CLEANUPARR_IMAGE_TAG:-latest}` | `["${CLEANUPARR_PROFILE:-download}"]` | — | traefik |
| `cloudflared/compose.yaml` | `cloudflared` | `${CLOUDFLARED_IMAGE:-cloudflare/cloudflared}:${CLOUDFLARED_IMAGE_TAG:-latest}` | `["${CLOUDFLARED_PROFILE:-network}"]` | — | — |
| `deunhealth/compose.yaml` | `deunhealth` | `${DEUNHEALTH_IMAGE:-qmcgaw/deunhealth}:${DEUNHEALTH_IMAGE_TAG:-latest}` | `["${DEUNHEALTH_PROFILE:-infrastructure}"]` | — | socket-proxy |
| `docker-discord-alerts/compose.yaml` | `docker-discord-alerts` | `${DOCKER_DISCORD_ALERTS_IMAGE:-ghcr.io/rycochet/docker-discord-alerts}:${DOCKER_DISCORD_ALERTS_IMAGE_TAG:-latest}` | `["${DOCKER_DISCORD_ALERTS_PROFILE:-tools}"]` | — | — |
| `dozzle/compose.yaml` | `dozzle` | `${DOZZLE_IMAGE:-amir20/dozzle}:${DOZZLE_IMAGE_TAG:-latest}` | `["${DOZZLE_PROFILE:-information}"]` | — | socket-proxy, traefik |
| `duc/compose.yaml` | `duc` | `${DUC_IMAGE:-tigerdockermediocore/duc-docker}:${DUC_IMAGE_TAG:-latest}` | `["${DUC_PROFILE:-disabled}"]` | — | traefik |
| `emby/compose.yaml` | `emby` | `${EMBY_IMAGE:-linuxserver/emby}:${EMBY_IMAGE_TAG:-latest}` | `["${EMBY_PROFILE:-library}"]` | "8096:8096" | traefik |
| `error-pages/compose.yaml` | `error-pages` | `${ERROR_PAGES_IMAGE:-ghcr.io/tarampampam/error-pages}:${ERROR_PAGES_IMAGE_TAG:-latest}` | `["${ERROR_PAGES_PROFILE:-network}"]` | — | — |
| `flaresolverr/compose.yaml` | `flaresolverr` | `${FLARESOLVERR_IMAGE:-ghcr.io/flaresolverr/flaresolverr}:${FLARESOLVERR_IMAGE_TAG:-latest}` | `["${FLARESOLVERR_PROFILE:-network}"]` | — | traefik |
| `glances/compose.yaml` | `glances` | `${GLANCES_IMAGE:-nicolargo/glances}:${GLANCES_IMAGE_TAG:-ubuntu-latest-full}` | `["${GLANCES_PROFILE:-information}"]` | — | socket-proxy, traefik |
| `homepage/compose.yaml` | `homepage` | `${HOMEPAGE_IMAGE:-ghcr.io/gethomepage/homepage}:${HOMEPAGE_IMAGE_TAG:-latest}` | `["${HOMEPAGE_PROFILE:-information}"]` | — | socket-proxy, traefik |
| `huntarr/compose.yaml` | `huntarr` | `${HUNTARR_IMAGE:-huntarr/huntarr}:${HUNTARR_IMAGE_TAG:-latest}` | `["${HUNTARR_PROFILE:-download}"]` | — | traefik |
| `i2p/compose.yaml` | `i2p` | `${I2P_IMAGE:-geti2p/i2p}:${I2P_IMAGE_TAG:-latest}` | `["${I2P_PROFILE:-disabled}"]` | "4444:4444", "7656:7656", "$I2P_PORT:$I2P_PORT", "$I2P_PORT:$I2P_PORT/udp" | — |
| `imagemaid/compose.yaml` | `imagemaid` | `${IMAGEMAID_IMAGE:-kometateam/imagemaid}:${IMAGEMAID_IMAGE_TAG:-latest}` | `["${IMAGEMAID_PROFILE:-quality}"]` | — | — |
| `jellyfin/compose.yaml` | `jellyfin` | `${JELLYFIN_IMAGE:-lscr.io/linuxserver/jellyfin}:${JELLYFIN_IMAGE_TAG:-latest}` | `["${JELLYFIN_PROFILE:-library}"]` | "8096:8096", "8920:8920", "7359:7359/udp", "1900:1900/udp" | traefik |
| `kapowarr/compose.yaml` | `kapowarr` | `${KAPOWARR_IMAGE:-mrcas/kapowarr}:${KAPOWARR_IMAGE_TAG:-latest}` | `["${KAPOWARR_PROFILE:-library}"]` | — | traefik |
| `kometa/compose.yaml` | `kometa` | `${KOMETA_IMAGE:-lscr.io/linuxserver/kometa}:${KOMETA_IMAGE_TAG:-latest}` | `["${KOMETA_PROFILE:-quality}"]` | — | plex, sonarr, radarr |
| `komga/compose.yaml` | `komga` | `${KOMGA_IMAGE:-gotson/komga}:${KOMGA_IMAGE_TAG:-latest}` | `["${KOMGA_PROFILE:-library}"]` | — | traefik |
| `libretranslate/compose.yaml` | `libretranslate` | `${LIBRETRANSLATE_IMAGE:-libretranslate/libretranslate}:${LIBRETRANSLATE_IMAGE_TAG:-latest}` | `["${LIBRETRANSLATE_PROFILE:-quality}"]` | — | traefik |
| `lidarr/compose.yaml` | `lidarr` | `${LIDARR_IMAGE:-lscr.io/linuxserver/lidarr}:${LIDARR_IMAGE_TAG:-latest}` | `["${LIDARR_PROFILE:-media}"]` | — | traefik |
| `lidarr/compose.yaml` | `lidarr-cache-warmer` | `ghcr.io/devianteng/lidarr-cache-warmer:latest` | `["disabled"]` | — | lidarr |
| `lingarr/compose.yaml` | `lingarr` | `${LINGARR_IMAGE:-lingarr/lingarr}:${LINGARR_IMAGE_TAG:-latest}` | `["${LINGARR_PROFILE:-quality}"]` | — | traefik |
| `manyfold/compose.yaml` | `manyfold` | `${MANYFOLD_IMAGE:-ghcr.io/manyfold3d/manyfold-solo}:${MANYFOLD_IMAGE_TAG:-latest}` | `["${MANYFOLD_PROFILE:-library}"]` | — | traefik |
| `minecraft/compose.yaml` | `minecraft` | `${MINECRAFT_IMAGE:-marctv/minecraft-papermc-server}:${MINECRAFT_IMAGE_TAG:-latest}` | `["${MINECRAFT_PROFILE:-games}"]` | "25565:25565/tcp", "25565:25565/udp" | traefik |
| `mylar/compose.yaml` | `mylar` | `${MYLAR_IMAGE:-lscr.io/linuxserver/mylar3}:${MYLAR_IMAGE_TAG:-latest}` | `["${MYLAR_PROFILE:-media}"]` | — | traefik |
| `notifiarr/compose.yaml` | `notifiarr` | `${NOTIFIARR_IMAGE:-ghcr.io/notifiarr/notifiarr}:${NOTIFIARR_IMAGE_TAG:-latest}` | `["${NOTIFIARR_PROFILE:-information}"]` | — | traefik |
| `onlyfans/compose.yaml` | `onlyfans` | `${ONLYFANS_IMAGE:-ghcr.io/datawhores/of-scraper}:${ONLYFANS_IMAGE_TAG:-latest}` | `["${ONLYFANS_PROFILE:-download}"]` | — | — |
| `openspeedtest/compose.yaml` | `openspeedtest` | `${OPENSPEEDTEST_IMAGE:-openspeedtest/latest}:${OPENSPEEDTEST_IMAGE_TAG:-latest}` | `["${OPENSPEEDTEST_PROFILE:-network}"]` | — | traefik |
| `overseerr/compose.yaml` | `overseerr` | `${OVERSEERR_IMAGE:-lscr.io/linuxserver/overseerr}:${OVERSEERR_IMAGE_TAG:-latest}` | `["${OVERSEERR_PROFILE:-information}"]` | — | traefik |
| `pgadmin/compose.yaml` | `pgadmin` | `${PGADMIN_IMAGE:-dpage/pgadmin4}:${PGADMIN_IMAGE_TAG:-latest}` | `["${PGADMIN_PROFILE:-media}"]` | — | postgres |
| `plex-find-mismatch/compose.yaml` | `plex-find-mismatch` | `${PLEX_FIND_MISMATCH_IMAGE:-rycochet/plex-find-mismatch}:${PLEX_FIND_MISMATCH_IMAGE_TAG:-latest}` | `["${PLEX_FIND_MISMATCH_PROFILE:-quality}"]` | — | plex |
| `plex/compose.yaml` | `plex` | `${PLEX_IMAGE:-plexinc/pms-docker}:${PLEX_IMAGE_TAG:-latest}` | `["${PLEX_PROFILE:-library}"]` | "32400:32400" | traefik |
| `portainer/compose.yaml` | `portainer` | `${PORTAINER_IMAGE:-portainer/portainer-ce}:${PORTAINER_IMAGE_TAG:-alpine}` | `["${PORTAINER_PROFILE:-information}"]` | — | traefik |
| `postgres/compose.yaml` | `postgres` | `${POSTGRES_IMAGE:-postgres}:${POSTGRES_IMAGE_TAG:-17-alpine}` | `["${POSTGRES_PROFILE:-media}"]` | — | — |
| `prowlarr/compose.yaml` | `prowlarr` | `${PROWLARR_IMAGE:-lscr.io/linuxserver/prowlarr}:${PROWLARR_IMAGE_TAG:-latest}` | `["${PROWLARR_PROFILE:-download}"]` | — | traefik |
| `qbittorrent/compose.yaml` | `qbittorrent` | `${QBITTORRENT_IMAGE:-lscr.io/linuxserver/qbittorrent}:${QBITTORRENT_IMAGE_TAG:-latest}` | `["${QBITTORRENT_PROFILE:-download}"]` | "51414:51414" | traefik |
| `radarr/compose.yaml` | `radarr` | `${RADARR_IMAGE:-lscr.io/linuxserver/radarr}:${RADARR_IMAGE_TAG:-latest}` | `["${RADARR_PROFILE:-media}"]` | — | traefik |
| `readarr/compose.yaml` | `readarr` | `${READARR_IMAGE:-lscr.io/linuxserver/readarr}:${READARR_IMAGE_TAG:-0.4.18-develop}` | `["${READARR_PROFILE:-media}"]` | — | traefik |
| `sabnzbd/compose.yaml` | `sabnzbd` | `${SABNZBD_IMAGE:-lscr.io/linuxserver/sabnzbd}:${SABNZBD_IMAGE_TAG:-latest}` | `["${SABNZBD_PROFILE:-download}"]` | — | traefik |
| `scrutiny/compose.yaml` | `scrutiny` | `${SCRUTINY_IMAGE:-ghcr.io/analogj/scrutiny}:${SCRUTINY_IMAGE_TAG:-latest}` | `["${SCRUTINY_PROFILE:-information}"]` | — | traefik |
| `socket-proxy/compose.yaml` | `socket-proxy` | `${SOCKET_PROXY_IMAGE:-lscr.io/linuxserver/socket-proxy}:${SOCKET_PROXY_IMAGE_TAG:-latest}` | `["${SOCKET_PROXY_PROFILE:-infrastructure}"]` | — | — |
| `sonarr/compose.yaml` | `sonarr` | `${SONARR_IMAGE:-lscr.io/linuxserver/sonarr}:${SONARR_IMAGE_TAG:-latest}` | `["${SONARR_PROFILE:-media}"]` | — | traefik |
| `sonarr_youtubedl/compose.yaml` | `sonarr_youtubedl` | `${SONARR_YOUTUBEDL_IMAGE:-whatdaybob/sonarr_youtubedl}:${SONARR_YOUTUBEDL_IMAGE_TAG:-latest}` | `["${SONARR_YOUTUBEDL_PROFILE:-disabled}"]` | — | sonarr |
| `speedtest-tracker/compose.yaml` | `speedtest-tracker` | `${SPEEDTEST_TRACKER_IMAGE:-lscr.io/linuxserver/speedtest-tracker}:${SPEEDTEST_TRACKER_IMAGE_TAG:-latest}` | `["${SPEEDTEST_TRACKER_PROFILE:-media}"]` | — | traefik |
| `stash/compose.yaml` | `stash` | `${STASH_IMAGE:-stashapp/stash}:${STASH_IMAGE_TAG:-latest}` | `["${STASH_PROFILE:-media}"]` | — | traefik |
| `syncthing/compose.yaml` | `syncthing` | `${SYNCTHING_IMAGE:-syncthing/syncthing}:${SYNCTHING_IMAGE_TAG:-latest}` | `["${SYNCTHING_PROFILE:-download}"]` | "21027:21027/udp", "22000:22000/tcp", "22000:22000/udp" | traefik |
| `tautulli/compose.yaml` | `tautulli` | `${TAUTULLI_IMAGE:-lscr.io/linuxserver/tautulli}:${TAUTULLI_IMAGE_TAG:-latest}` | `["${TAUTULLI_PROFILE:-information}"]` | — | traefik |
| `tdarr-node/compose.yaml` | `tdarr-node` | `${TDARR_NODE_IMAGE:-haveagitgat/tdarr_node}:${TDARR_NODE_IMAGE_TAG:-latest}` | `["${TDARR_NODE_PROFILE:-quality}"]` | — | tdarr |
| `tdarr/compose.yaml` | `tdarr` | `${TDARR_IMAGE:-haveagitgat/tdarr}:${TDARR_IMAGE_TAG:-latest}` | `["${TDARR_PROFILE:-quality}"]` | "8266:8266" | traefik |
| `tdarr_inform/compose.yaml` | `tdarr-inform` | `${TDARR_INFORM_IMAGE:-ghcr.io/deathbybandaid/tdarr_inform}:${TDARR_INFORM_IMAGE_TAG:-latest}` | `["${TDARR_INFORM_PROFILE:-quality}"]` | — | traefik, tdarr |
| `tinyauth/compose.yaml` | `tinyauth` | `${TINYAUTH_IMAGE:-ghcr.io/steveiliop56/tinyauth}:${TINYAUTH_IMAGE_TAG:-v4-distroless}` | `["${TINYAUTH_PROFILE:-network}"]` | — | — |
| `titlecardmaker/compose.yaml` | `titlecardmaker` | `${TITLECARDMAKER_IMAGE:-ghcr.io/titlecardmaker/titlecardmaker-webui}:${TITLECARDMAKER_IMAGE_TAG:-latest}` | `["${TITLECARDMAKER_PROFILE:-quality}"]` | — | traefik |
| `traefik/compose.yaml` | `traefik` | `${TRAEFIK_IMAGE:-traefik}:${TRAEFIK_IMAGE_TAG:-v3}` | `["${TRAEFIK_PROFILE:-network}"]` | "80:80", "443:443" | socket-proxy, tinyauth, error-pages |
| `ubooquity/compose.yaml` | `ubooquity` | `${UBOOQUITY_IMAGE:-lscr.io/linuxserver/ubooquity}:${UBOOQUITY_IMAGE_TAG:-latest}` | `["${UBOOQUITY_PROFILE:-media}"]` | — | traefik |
| `vpn/compose.yaml` | `vpn` | `${VPN_IMAGE:-qmcgaw/gluetun}:${VPN_IMAGE_TAG:-latest}` | `["${VPN_PROFILE:-network}"]` | "1080:1080", "8888:8888", "8000:8000" | — |
| `vpn/compose.yaml` | `proxy` | `${PROXY_IMAGE:-serjs/go-socks5-proxy}:${PROXY_IMAGE_TAG:-latest}` | `["${PROXY_PROFILE:-network}"]` | — | vpn |
| `watchstate/compose.yaml` | `watchstate` | `${WATCHSTATE_IMAGE:-ghcr.io/arabcoders/watchstate}:${WATCHSTATE_IMAGE_TAG:-latest}` | `["${WATCHSTATE_PROFILE:-information}"]` | — | traefik |
| `watchtower/compose.yaml` | `watchtower` | `${WATCHTOWER_IMAGE:-nickfedor/watchtower}:${WATCHTOWER_IMAGE_TAG:-latest}` | `["${WATCHTOWER_PROFILE:-infrastructure}"]` | — | socket-proxy |
| `webtop/compose.yaml` | `webtop` | `${WEBTOP_IMAGE:-lscr.io/linuxserver/webtop}:${WEBTOP_IMAGE_TAG:-latest}` | `["${WEBTOP_PROFILE:-desktop}"]` | — | traefik |
| `whisparr/compose.yaml` | `whisparr` | `${WHISPARR_IMAGE:-ghcr.io/hotio/whisparr}:${WHISPARR_IMAGE_TAG:-v3}` | `["${WHISPARR_PROFILE:-media}"]` | — | traefik |
| `whisper/compose.yaml` | `whisper` | `${WHISPER_IMAGE:-mccloud/subgen}:${WHISPER_IMAGE_TAG:-latest}` | `["${WHISPER_PROFILE:-quality}"]` | — | — |
| `windows/compose.yaml` | `windows` | `${WINDOWS_IMAGE:-dockurr/windows}:${WINDOWS_IMAGE_TAG:-latest}` | `["${WINDOWS_PROFILE:-desktop}"]` | — | traefik |
| `zfdash/compose.yaml` | `zfdash` | `${ZFDASH_IMAGE:-ad4mts/zfdash}:${ZFDASH_IMAGE_TAG:-latest}` | `["${ZFDASH_PROFILE:-information}"]` | — | traefik |
| `zfs-discord-alerts/compose.yaml` | `zfs-discord-alerts` | `${ZFS_DISCORD_ALERTS_IMAGE:-ghcr.io/rycochet/zfs-discord-alerts}:${ZFS_DISCORD_ALERTS_IMAGE_TAG:-latest}` | `["${ZFS_DISCORD_ALERTS_PROFILE:-tools}"]` | — | — |

## Supporting scripts and utilities

- `install.sh` is an incomplete bootstrap helper (copies `.env`, tries to fetch NordVPN keys, and has stubs for API key harvesting).
- `overlay.sh` manages overlayfs mounts for media paths (safe-write overlay with up/down/refresh/diff/merge).
- `minimum-keep-alive.sh` ensures Traefik is always running (cron-friendly watchdog).
- `clear-overlay.py` removes the `overlay` label from Plex episodes based on `.env` values.
- `onlyfans/update_login.sh` updates OF scraper authentication (service-specific helper).
- `make_ignore.sh` scripts in *arr services help keep temporary downloads and metadata out of scans.

## Component requirements (what must be passed)

The most common requirements you must set in `.env` are:

- **Global**: `DOMAIN`, `EMAIL`, `TZ`, `PUID`, `PGID` (used across multiple services).
- **Paths**: `PATH_MOVIES`, `PATH_TV`, `PATH_MUSIC`, `PATH_DOWNLOADS`, `PATH_BOOKS`, `PATH_AUDIOBOOKS`, etc.
- **Ingress**: `CLOUDFLARED_TOKEN`, `CLOUDFLARE_DNS_API` (Traefik DNS challenge).
- **VPN**: `VPN_SERVICE_PROVIDER`, `VPN_WIREGUARD_PRIVATE_KEY` (for Gluetun).
- **Media server tokens**: `PLEX_TOKEN`, `PLEX_CLAIM` (first boot), `TAUTULLI_API` (widgets).

When you see a service mount a path like `$PATH_TV` or `$PATH_MOVIES`, that path must exist on the host or Docker will create empty directories (leading to “missing media” and failed scans).

## Step 2: Build a deployment script (design & structure)

Below is a research-backed outline for a production deployment script (`deploy.sh`). This is designed to be safe, repeatable, and explicit about what is and isn’t auto-configured.

### 2.1 Preflight checks

- Verify `docker` and `docker compose` are installed and running.
- Ensure the script runs from the repo root (where `compose.yaml` exists).
- Confirm required files exist: `compose.yaml`, `compose.extend.yaml`, `.env` or `.env.example`.

### 2.2 Environment bootstrap & validation

- If `.env` is missing:
  - Copy from `.env.example`.
  - Exit with a clear error message that `.env` must be filled in.
- Validate required values:
  - Global: `DOMAIN`, `EMAIL`, `TZ`, `PUID`, `PGID`.
  - Paths: `PATH_DOWNLOADS`, `PATH_MOVIES`, `PATH_TV`, `PATH_MUSIC` at minimum.
- Optionally generate secrets:
  - `OAUTH_SECRET` (hex string) if Tinyauth is enabled.

### 2.3 Profile & service plan

- Read `COMPOSE_PROFILES` to determine which stack groups are enabled.
- For any service that requires tokens (Plex, Cloudflare, VPN), verify the required env var is set.
- Warn (but do not fail) if services are enabled without required config.

### 2.4 Deploy actions

- `docker compose pull`
- `docker compose up -d`
- Optional: wait for health checks (Traefik, VPN, Plex).

### 2.5 Post-deploy tasks

- Surface URLs for core UI endpoints (Traefik dashboard, Plex, Jellyfin, etc.).
- Recommend first-boot steps (scan libraries, configure *arr apps, set up indexers).

## Step 3: Local testing & fast iteration

1. **Minimal profile testing**
   - Set `COMPOSE_PROFILES` to a narrow set (e.g., `network,library`).
2. **Start only the routing layer**
   - `docker compose up -d traefik`
3. **Bring up services one at a time**
   - `docker compose up -d plex`
4. **Iterate fast**
   - Use `docker compose logs -f <service>` and `docker compose restart <service>` for rapid changes.
5. **Tear down quickly**
   - `docker compose down` for a clean slate.
