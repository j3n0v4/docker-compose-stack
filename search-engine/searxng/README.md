# SearXNG config

This directory holds your SearXNG `settings.yml` and `limiter.toml`. Both are committed to the repository as pre-filled templates for a fully private instance. You only need to set your `secret_key` and ensure the files are readable by the container.

## Requirements

The SearXNG container image runs as **uid 977**. For the container to read your config, the files and directory must have the correct permissions:

```bash
# settings.yml and limiter.toml must be world-readable (644), dir must be 755
chmod 644 settings.yml limiter.toml
chmod 755 .          # the searxng/ directory
```

The config dir is mounted read-only (`:ro`) into the container at `/etc/searxng`, so the container reads it without write access.

### Mandatory keys

At minimum `settings.yml` must contain a valid `secret_key`:

```yaml
# REQUIRED — the instance boots only with a valid secret_key.
# Generate with: openssl rand -hex 32
secret_key: "CHANGE_ME"
```

The `server` block is pre-filled:

```yaml
server:
  # Must match SEARXNG_BASE_URL in .env
  base_url: "http://127.0.0.1:8080/"
  # Bind to the container's internal port (8080). The loopback-only host
  # binding in docker-compose.yml is what keeps it private.
  bind_address: "0.0.0.0"
  port: 8080
```

The valkey backend is pre-filled:

```yaml
valkey:
  # Must match SEARXNG_VALKEY_URL in .env (valkey://valkey:6379/0)
  url: "valkey://valkey:6379/0"
```

## Pitfalls

- **uid 977**: if the container logs a permission error reading `/etc/searxng/settings.yml`, your file is not `644` or the dir is not `755`.
- **`SEARXNG_VALKEY_URL` naming**: the env var is now `SEARXNG_VALKEY_URL` (SearXNG migrated from `SEARXNG_REDIS_URL`). Use the new name.
- **RDB incompatibility**: Valkey 9 and Redis 7 RDB files are not interchangeable. If you migrate from a Redis-backed SearXNG, do not reuse the old RDB — start with a fresh `valkey-data` volume.
- **`secret_key` is mandatory**: the container exits immediately if it is missing or empty.

## Reference

The full upstream template is maintained by the SearXNG project. Copy it as a starting point, then apply the mandatory keys above:

- https://github.com/searxng/searxng-docker/blob/master/searxng/settings.yml
