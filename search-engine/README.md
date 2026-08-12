<h1 align="center">search-engine</h1>

<p align="center">A fully private, local search engine stack in a single Docker Compose file. No API keys, no third-party accounts, no telemetry.</p>

<p align="center">
  <a href="#quick-start">Quick start</a> ·
  <a href="#privacy-features">Privacy features</a> ·
  <a href="#pitfalls">Pitfalls</a> ·
  <a href="#license">License</a>
</p>

<p align="center">
  <a href="../LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/License-MIT-yellow.svg?style=flat"></a>
  <a href="docker-compose.yml"><img alt="Docker Compose" src="https://img.shields.io/badge/Docker-Compose-blue.svg?style=flat"></a>
</p>

**Why this exists:**

I wanted a fully private search engine with no API keys, no third-party accounts, and no telemetry. Docker Compose profiles keep the stack self-contained.

**What it does:**

Provides a declarative `docker-compose.yml` describing a private search stack — SearXNG (metasearch frontend, 70+ engines) backed by Valkey 9 (cache/limiter). Pre-configured for single-user privacy: rate limiting on, risky engines disabled, loopback-only binding, internal network, and `cap_drop: [ALL]`.

## Quick start

```bash
cd docker-compose-stack/search-engine
cp .env.example .env
# edit .env: set SEARXNG_SECRET, paths, ports
docker compose --profile core up -d
```

The `core` profile is the only profile — it starts both services (searxng + valkey):

```bash
docker compose --profile core up -d   # searxng + valkey
```

Bring services down with `docker compose --profile core down`.

## Profiles

| Service | Image | Port (host) |
|---------|-------|-------------|
| searxng | searxng/searxng | 127.0.0.1:8080 |
| valkey | valkey/valkey:9-alpine | internal only |

SearXNG is the metasearch frontend. Valkey 9 is its cache/limiter backend — SearXNG migrated from Redis, and the env var is now named `SEARXNG_VALKEY_URL`.

## Network isolation

The `search` network is declared `internal: true`, so the search services can only reach each other and have no external egress:

| Network | Services | Why |
|---------|----------|-----|
| `search` (internal) | searxng, valkey | searxng talks to valkey (cache/limiter) |

Only the loopback port binding (`127.0.0.1:${SEARXNG_PORT}:8080`) publishes anything outward. A service is only reachable from others that share its network(s).

## Privacy features

The stack is pre-configured for a fully private, single-user instance:

- **`server.limiter: true`** — rate limiting on, tuned for a private instance in `limiter.toml`.
- **`server.public_instance: false`** — not a public instance; no link tokens, no public registration.
- **`server.image_proxy: true`** — image results proxied through SearXNG so your IP is not leaked to image hosts.
- **`server.method: "GET"`** — search queries use GET (no query in POST body).
- **Privacy HTTP headers** — `nosniff`, `noopen`, `noindex, nofollow`, `no-referrer` on every response.
- **`ui.query_in_title: false`** — search terms not put in the page `<title>`.
- **`general.enable_metrics: false`** — no Prometheus metrics endpoint.
- **Privacy-risky engines disabled** — Google, Yandex, Yahoo, Bing are disabled by default.
- **Fixed user-agent** — not randomized; add a contact via `outgoing.useragent_suffix` if you like.

## Pre-configured settings

Both `searxng/settings.yml` and `searxng/limiter.toml` are included in the repository and pre-filled for a private instance. You only need to:

1. Set `SEARXNG_SECRET` in `.env` (and optionally copy it into `settings.yml`'s `secret_key`).
2. Ensure the files are readable by the container (uid 977 — see [searxng/README.md](searxng/README.md)).

`settings.yml` pre-fills the privacy features above, the engine basket (Wikipedia, Bing, DuckDuckGo, GitHub, Hacker News, Qwant, Startpage, Yandex, Yahoo, Mojeek), and the four privacy-risky engines disabled. `limiter.toml` pre-fills `botdetection` with `ipv4_prefix = 32` / `ipv6_prefix = 48`, trusted proxy ranges, `link_token` disabled, and `pass_searxng_org` enabled.

## Security hardening

- **`cap_drop: [ALL]`** on every service — no Linux capabilities granted.
- **`security_opt: [no-new-privileges:true]`** on every service.
- **`read_only: true`** on searxng (with a `tmpfs` for `/tmp`).
- **Loopback-only binding** by default — not reachable from the LAN unless you change the bind address.
- **Internal network** — search services have no external egress.
- **`mem_limit`** on every service (searxng 512m, valkey 128m).
- **`json-file` logging with limits** (`10m` × 3).
- **Secrets in `.env` only** — never in the Compose file or committed config.

## Pitfalls

- **uid 977**: permission errors reading `/etc/searxng/settings.yml` mean the file is not `644` or the dir is not `755`.
- **`SEARXNG_VALKEY_URL` naming**: the env var is now `SEARXNG_VALKEY_URL` (SearXNG migrated from `SEARXNG_REDIS_URL`). Use the new name.
- **RDB incompatibility**: Valkey 9 and Redis 7 RDB files are not interchangeable. Do not reuse an old Redis RDB — start with a fresh `valkey-data` volume.
- **`secret_key` is mandatory**: the container exits immediately if it is missing or empty.
- **Alpine images have `wget`, not `curl`**: healthchecks use `wget`.

## Environment variables

All configurable values are declared in `.env.example`. Copy it to `.env` and set your own values. Generate the secret with:

```bash
export SEARXNG_SECRET=$(openssl rand -hex 32)
```

## Assumptions

This stack assumes a reverse proxy (nginx, etc.) exists elsewhere for TLS. There is no TLS terminator in this file — SearXNG binds to `127.0.0.1` by default. Point your reverse proxy at `127.0.0.1:${SEARXNG_PORT}` and terminate TLS there.

## License

[MIT](../LICENSE) &copy; 2026 JD Cordero
