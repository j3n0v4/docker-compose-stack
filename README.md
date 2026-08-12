<h1 align="center">docker-compose-stack</h1>

<p align="center">Declarative Docker Compose for self-hosted media, search, and NVR infrastructure.</p>

<p align="center">
  <a href="#quick-start">Quick start</a> ·
  <a href="#profiles">Profiles</a> ·
  <a href="#network-isolation">Network isolation</a> ·
  <a href="#license">License</a>
</p>

<p align="center">
  <a href="LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/License-MIT-yellow.svg?style=flat"></a>
  <a href="media-stack/docker-compose.yml"><img alt="Docker Compose" src="https://img.shields.io/badge/Docker-Compose-blue.svg?style=flat"></a>
  <img alt="CI" src="https://github.com/j3n0v4/docker-compose-stack/actions/workflows/ci.yml/badge.svg">
</p>

**Why this exists:**

I run self-hosted media, search, and NVR services across several machines. Docker Compose profiles let a single repository serve every host — each machine only starts the services it needs.

**What it does:**

Provides declarative `docker-compose.yml` files for three self-hosted stacks — media (*arr automation, music/audiobooks, Jellyfin), search (SearXNG + Valkey), and NVR (Frigate with optional Coral TPU and Intel QSV acceleration). Every host-specific value is a `${VARIABLE}` declared in `.env.example`, so nothing real reaches a public push.

## Quick start

```bash
git clone https://github.com/j3n0v4/docker-compose-stack.git
cd docker-compose-stack
```

Each stack is self-contained — pick the one you need:

```bash
cd media-stack && cp .env.example .env   # *arr, music, Jellyfin
cd search-engine && cp .env.example .env  # SearXNG + Valkey
cd frigate && cp .env.example .env        # Frigate NVR
```

Edit `.env`, set your paths and ports, then start a profile:

```bash
# media-stack
docker compose --profile arr --profile music --profile jellyfin up -d

# search-engine
docker compose --profile core up -d

# frigate
docker compose --profile nvr up -d
```

Bring services down with `docker compose --profile <name> down`.

## Profiles

| Stack | Profile | Services | Host |
|-------|---------|----------|------|
| media-stack | `arr` | sonarr, radarr, lidarr, prowlarr, bazarr, recyclarr | media |
| media-stack | `music` | music-assistant, audiobookshelf | music |
| media-stack | `jellyfin` | jellyfin | media |
| search-engine | `core` | searxng, valkey | search |
| frigate | `nvr` | frigate | nvr |

See [media-stack/README.md](media-stack/README.md), [search-engine/README.md](search-engine/README.md), and [frigate/README.md](frigate/README.md) for the full service tables.

## Network isolation

Each named network exists so a container can only reach the services it actually talks to:

| Network | Services | Why |
|---------|----------|-----|
| `arr` | sonarr, radarr, lidarr, prowlarr, bazarr, recyclarr, jellyfin | Genuinely inter-communicating: prowlarr serves indexers, bazarr talks to sonarr/radarr, recyclarr syncs profiles, jellyfin reaches sonarr/radarr for metadata |
| `music` | audiobookshelf | No real cross-talk; kept on a shared network for consistency |
| `search` (internal) | searxng, valkey | searxng talks to valkey (cache/limiter) |
| `nvr` (internal) | frigate | Isolated NVR — no external network access |

`music-assistant` uses `network_mode: host` and cannot be isolated. A service is only reachable from others that share its network(s).

## NFS requirements

The `arr`, `music`, and `jellyfin` profiles mount media over NFS from the NAS before any container starts. The compose file expects these mount points on the host, configured via `DOWNLOADS_DIR` and `MEDIA_DIR`:

- `${DOWNLOADS_DIR}` (`/mnt/downloads`)
- `${MEDIA_DIR}/tv`, `/movies`, `/anime`, `/cartoons`, `/music`, `/audiobooks`

If the NFS share is unavailable, the containers start but the mount paths are empty. Mount the shares first, then run `docker compose up`.

## Environment variables

All configurable values are declared in `.env.example`. Copy it to `.env` and set your own values. API keys are not stored in either file — export them in your shell before `docker compose up`:

```bash
export SONARR_API_KEY=... RADARR_API_KEY=...
```

## Assumptions

This repository assumes a reverse proxy (nginx, etc.) exists elsewhere for TLS. Neither stack terminates TLS — point your reverse proxy at the published ports and terminate TLS there.

## License

[MIT](LICENSE) &copy; 2026 JD Cordero