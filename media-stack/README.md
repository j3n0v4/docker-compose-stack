<h1 align="center">media-stack</h1>

<p align="center">Declarative Docker Compose for self-hosted media automation, music, and a Jellyfin media server.</p>

<p align="center">
  <a href="#quick-start">Quick start</a> ·
  <a href="#profiles">Profiles</a> ·
  <a href="#network-isolation">Network isolation</a> ·
  <a href="#license">License</a>
</p>

<p align="center">
  <a href="../LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/License-MIT-yellow.svg?style=flat"></a>
  <a href="docker-compose.yml"><img alt="Docker Compose" src="https://img.shields.io/badge/Docker-Compose-blue.svg?style=flat"></a>
</p>

**Why this exists:**

I run self-hosted media services across several machines. Docker Compose profiles let a single compose file serve every host — each machine only starts the services it needs.

**What it does:**

Provides a declarative `docker-compose.yml` describing every containerised media service — *arr automation, music/audiobooks, and Jellyfin. Host-specific values are declared in `.env.example` and never hardcoded.

## Quick start

```bash
cd docker-compose-stack/media-stack
cp .env.example .env
# edit .env, set your paths and ports
docker compose --profile arr --profile music --profile jellyfin up -d
```

Start a single profile, or everything at once:

```bash
docker compose --profile arr      up -d   # *arr automation
docker compose --profile music    up -d   # music + audiobooks
docker compose --profile jellyfin up -d   # media server
docker compose --profile arr --profile music --profile jellyfin up -d
```

Bring services down with `docker compose --profile <name> down`.

## Profiles

### arr — media automation

| Service | Image | Port (host) |
|---------|-------|-------------|
| sonarr | hotio/sonarr | 8989 |
| radarr | hotio/radarr | 7878 |
| lidarr | hotio/lidarr | 8686 |
| prowlarr | hotio/prowlarr | 9696 |
| bazarr | hotio/bazarr | 6767 |
| recyclarr | recyclarr/recyclarr | CLI only |

All `hotio` images run as `${PUID}:${PGID}` with `${TZ}`. `recyclarr` is a CLI container with no published ports.

### music — music and audiobooks

| Service | Image | Port (host) |
|---------|-------|-------------|
| music-assistant | music-assistant/server | host network |
| audiobookshelf | advplyr/audiobookshelf | 13378 |

`audiobookshelf` uses a `wget`-based healthcheck because its Alpine base image ships no `curl`.

### jellyfin — media server

| Service | Image | Port (host) |
|---------|-------|-------------|
| jellyfin | jellyfin/jellyfin | 8096 / 8920 (HTTPS) |

Jellyfin joins the `arr` network so it can reach sonarr/radarr for metadata lookups. All media volumes are mounted read-only.

**Intel QSV hardware transcoding** is available but commented out. To enable it, uncomment the `devices` and `group_add` lines in the jellyfin service, ensure the `render` group exists on the host, and add your PUID to it. The host must have an Intel GPU with `/dev/dri/renderD128`.

## Network isolation

Each named network exists so a container can only reach the services it actually talks to:

| Network | Services | Why |
|---------|----------|-----|
| `arr` | sonarr, radarr, lidarr, prowlarr, bazarr, recyclarr, jellyfin | Genuinely inter-communicating: prowlarr serves indexers, bazarr talks to sonarr/radarr, recyclarr syncs profiles, jellyfin reaches sonarr/radarr for metadata |
| `music` | audiobookshelf | No real cross-talk; kept on a shared network for consistency |

`music-assistant` uses `network_mode: host` and cannot be isolated. A service is only reachable from others that share its network(s).

## NFS requirements

The `arr`, `music`, and `jellyfin` profiles mount media over NFS from the NAS before any container starts. The compose file expects these mount points on the host, configured via `DOWNLOADS_DIR` and `MEDIA_DIR`:

- `${DOWNLOADS_DIR}` (`/mnt/downloads`)
- `${MEDIA_DIR}/tv`, `/movies`, `/anime`, `/cartoons`, `/music`, `/audiobooks`

If the NFS share is unavailable, the containers start but the mount paths are empty. Mount the shares first, then run `docker compose up`.

## Native services (not in Compose)

These run as systemd/native processes and are intentionally not part of this file:

- Jellyseerr (request manager)
- qBittorrent (download client)
- Transcoder
- Reverse proxy (NPM)

The compose stack references some of these as upstreams (for example, sonarr/radarr connect to qBittorrent at `${QBITTORRENT_HOST}`).

## Environment variables

All configurable values are declared in `.env.example`. Copy it to `.env` and set your own values. API keys are not stored in either file — export them in your shell before `docker compose up`:

```bash
export SONARR_API_KEY=... RADARR_API_KEY=...
```

## Assumptions

This stack assumes a reverse proxy (nginx, etc.) exists elsewhere for TLS. There is no TLS terminator in this file — the services publish plain HTTP to the host LAN. Point your reverse proxy at the published ports and terminate TLS there.

## License

[MIT](../LICENSE) &copy; 2026 JD Cordero
