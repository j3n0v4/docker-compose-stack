<h1 align="center">frigate</h1>

<p align="center">Declarative Docker Compose for a self-hosted Frigate NVR — real-time object detection with optional USB Coral TPU and Intel QSV acceleration.</p>

<p align="center">
  <a href="#quick-start">Quick start</a> ·
  <a href="#hardware-acceleration">Hardware acceleration</a> ·
  <a href="#network-isolation">Network isolation</a> ·
  <a href="#license">License</a>
</p>

<p align="center">
  <a href="../LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/License-MIT-yellow.svg?style=flat"></a>
  <a href="docker-compose.yml"><img alt="Docker Compose" src="https://img.shields.io/badge/Docker-Compose-blue.svg?style=flat"></a>
</p>

**Why this exists:**

I wanted a local network video recorder with real-time object detection that does not phone home. Docker Compose profiles keep the stack self-contained and easily reproducible.

**What it does:**

Provides a declarative `docker-compose.yml` describing a Frigate NVR container with network isolation, security hardening, and all host-specific values parameterized via `.env.example`. Includes a minimal `config/config.yml` template as a camera reference — not a deployment-ready config.

## Quick start

```bash
cd docker-compose-stack/frigate
cp .env.example .env
# edit .env: set FRIGATE_RTSP_PASSWORD, paths, ports
# copy and customize config/config.yml for your cameras
docker compose --profile nvr up -d
```

Bring services down with `docker compose --profile nvr down`.

## Hardware acceleration

### USB Coral TPU

The Coral USB Accelerator provides hardware-accelerated object detection. To enable it, uncomment the `devices` block in the frigate service (`/dev/bus/usb`), ensure the Coral is plugged in, and restart the container. Frigate auto-detects the Coral and uses it instead of the CPU detector.

### Intel QSV transcoding

Intel Quick Sync Video (QSV) accelerates ffmpeg decoding. To enable it, uncomment the `devices` and `group_add` lines in the frigate service (`/dev/dri/renderD128` and the `render` group), then add the `hwaccel_args` block in `config/config.yml`. The host must have an Intel GPU with `/dev/dri/renderD128`.

Both options can be used simultaneously — the Coral handles detection and QSV handles decoding.

## Network isolation

The `nvr` network is declared `internal: true`, so the Frigate container has no external egress:

| Network | Services | Why |
|---------|----------|-----|
| `nvr` (internal) | frigate | Isolated NVR — no external network access |
| `proxy` (optional) | frigate, reverse proxy | Allows a reverse proxy to reach the web UI |

Only the port bindings (`127.0.0.1:${FRIGATE_HOST}:5000` and `:8971`) publish anything outward. A service is only reachable from others that share its network(s).

## Security hardening

- **`cap_drop: [ALL]`** — no Linux capabilities granted beyond what the container needs.
- **`security_opt: [no-new-privileges:true]`** — prevents privilege escalation.
- **`internal: true` network** — Frigate cannot reach the internet or other stacks.
- **Loopback-only binding** by default — not reachable from the LAN unless you change `FRIGATE_HOST` in `.env`.
- **`json-file` logging with limits** (`10m` × 3).
- **Secrets in `.env` only** — never in the Compose file or committed config.

Note: Frigate cannot run `read_only: true` because it writes to `/db`, `/media/frigate`, `/clips`, and `/tmp/cache`. These paths are covered by bind mounts and tmpfs.

## Configuration

Copy `config/config.yml` to your `FRIGATE_CONFIG_DIR` and customize it for your cameras. The template contains commented-out camera placeholders — uncomment and fill in your RTSP URLs and detection zones.

The `FRIGATE_RTSP_PASSWORD` environment variable (set in `.env`) is available inside the container for use in RTSP URL strings.

## Environment variables

All configurable values are declared in `.env.example`. Copy it to `.env` and set your own values. Generate the RTSP password with:

```bash
export FRIGATE_RTSP_PASSWORD=$(openssl rand -hex 24)
```

## Assumptions

This stack assumes a reverse proxy (nginx, etc.) exists elsewhere for TLS. There is no TLS terminator in this file — Frigate binds to `127.0.0.1` by default. Point your reverse proxy at `127.0.0.1:5000` and terminate TLS there.

## License

[MIT](../LICENSE) &copy; 2026 JD Cordero