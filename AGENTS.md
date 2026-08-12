# AGENTS.md

Guidance for AI coding agents working in this repository.

## Project overview

docker-compose-stack is a collection of declarative Docker Compose files for self-hosted media, search, and NVR infrastructure. It is not a generator and has no CLI — each subdirectory is a standalone stack described by a single `docker-compose.yml` and a matching `.env.example`.

The companion project [elitebook-docker-server](https://github.com/j3n0v4/elitebook-docker-server) handles host setup (Docker, NFS, systemd). This project provides the compose stacks that run on top of that host.

## Repo structure

| Path | Purpose |
|------|---------|
| `README.md` | Project front door |
| `AGENTS.md` | This file |
| `LICENSE` | MIT license |
| `.gitignore` | Portfolio standard ignore set |
| `media-stack/` | *arr automation, music/audiobooks, Jellyfin |
| `media-stack/docker-compose.yml` | Compose file with `arr`, `music`, `jellyfin` profiles |
| `media-stack/.env.example` | Placeholder environment variables |
| `media-stack/README.md` | Media stack documentation |
| `search-engine/` | SearXNG + Valkey private search stack |
| `search-engine/docker-compose.yml` | Compose file with `core` profile |
| `search-engine/.env.example` | Placeholder environment variables |
| `search-engine/README.md` | Search stack documentation |
| `search-engine/searxng/` | SearXNG config templates |
| `search-engine/searxng/settings.yml` | Pre-filled private-instance settings |
| `search-engine/searxng/limiter.toml` | Pre-filled rate-limiter config |
| `search-engine/searxng/README.md` | Config directory docs |
| `search-engine/.gitignore` | Sub-directory ignore set |
| `frigate/` | Frigate NVR with optional Coral TPU and Intel QSV |
| `frigate/docker-compose.yml` | Compose file with `nvr` profile |
| `frigate/.env.example` | Placeholder environment variables |
| `frigate/README.md` | Frigate stack documentation |
| `frigate/config/config.yml` | Minimal Frigate config template (camera placeholders) |
| `frigate/.gitignore` | Sub-directory ignore set |

## Conventions

- Follow the `general-project-readme-style` skill (the author's canonical style guide for GitHub portfolio projects) when writing or editing any markdown, README, or documentation content.
- README structure: centered `<h1>` + `<p>` tagline, badge row, **bold** "Why this exists" / "What it does" labels, sentence-case H2 headings.
- No contractions in formal prose. Use full forms: `it is`, `does not`, `cannot`.
- No `> [!NOTE]`-style GitHub callouts. Use plain paragraphs.
- Tagged code fences only.
- License is MIT, copyright JD Cordero.
- GitHub username is `j3n0v4`.

### Docker Compose Conventions

- Every host-specific value (port, path, image tag, PUID/PGID, TZ, secret) is a `${VARIABLE}` declared in `.env.example` — never hardcoded in the compose file.
- Services use Docker Compose profiles so each host only starts what it needs.
- Named networks isolate services that genuinely inter-communicate.
- API keys are injected via shell environment, never stored in `.env` or `.env.example`.
- Healthchecks use `curl` for Debian-based images and `wget` for Alpine-based images.

## OPSEC rules

**Never commit real infrastructure identifiers.** All public files must use generic placeholders:

| What | Use | Never use |
|------|-----|-----------|
| IP addresses | `10.0.0.x` (RFC1918) | Real LAN IPs, `172.21.x.x`, `172.22.x.x` |
| Hostnames | Role-based: `media-host`, `search-host` | Real hostnames: `codex`, `auspex`, `vltx` |
| Paths | `/opt/...`, `/mnt/...` | Real home paths: `/home/vltx_admin`, `/Users/jcordero` |
| Domains | `.example.com` | Real domains: `vltx.nl` |
| MACs | `00:11:22:33:44:55` | Real MACs |
| TZ | `UTC` or `Etc/UTC` | Specific timezones that reveal region |
| VLANs | Generic descriptions | Real VLAN IDs or interface names |

The `.env` file is gitignored for a reason — real values go there, never in tracked files.

## Scope

### In scope

- Declarative Docker Compose stacks for self-hosted services
- `.env.example` files with placeholder values
- Pre-filled config templates (e.g. SearXNG `settings.yml`, `limiter.toml`, Frigate `config.yml`)
- Documentation for each stack (README per subdirectory)
- Hardware acceleration options (e.g. Intel QSV, USB Coral TPU) documented as optional

### Out of scope

- Host setup, NFS mounts, Docker installation (handled by elitebook-docker-server)
- TLS termination / reverse proxy configuration
- Custom-built Docker images (use upstream images only)
- Legally questionable containers (CAPTCHA bypass, P2P clients, adult content)
- Systemd/native services (Jellyseerr, qBittorrent, NPM) — mentioned in docs but not composed

## Build and edit commands

```bash
# No build step — this is a collection of compose files.
# Validate compose syntax:
docker compose -f media-stack/docker-compose.yml --profile arr config
docker compose -f search-engine/docker-compose.yml --profile core config
docker compose -f frigate/docker-compose.yml --profile nvr config

# Edit files in place; no compilation or bundling.
```

## Security note

Do not commit secrets, API keys, or private audit data. The `.env` file and any real infrastructure values belong in `.gitignore` and must remain untracked.