# Docker Compose Infrastructure

This repository manages multi-host Docker Compose deployments for personal infrastructure.

## Structure

- `sol/` - Primary host services (Caddy, cloudflared, n8n, Open-WebUI, Jellyfin, Syncthing, UniFi, RustDesk)
- `neptune/` - Neptune host services (Ollama, Kokoro)

## Commands

Deploy to specific host:
```bash
cd sol && docker compose up -d
cd neptune && docker compose up -d
```

View logs:
```bash
docker compose logs -f [service_name]
```

Rebuild single service:
```bash
docker compose build [service_name] && docker compose up -d [service_name]
```

## Style Guidelines

- Use `mise.toml` for DOCKER_HOST configuration per directory
- Store secrets in `.env` files (gitignored)
- Use named networks for service isolation (`web`, `postgres`, `valkey`)
- Bind mount volumes to host paths (`/evil_ripple/`, `/stable_grace/`)
- Use `restart: unless-stopped` for production services
- Caddy reverse proxy and cloudflared tunnels on sol for external access
- Use official images when available
- Network modes: `host` for UniFi/Syncthing/RustDesk, named networks otherwise

## Sol Host Specifications

- Hardware: Intel N100, 16GB RAM
- Storage: ZFS raidz2 array (5 disks) at `/evil_ripple`
- Networking:
  - Caddy: Reverse proxy with Route53 DNS challenge for TLS
  - cloudflared: Cloudflare Tunnel for external service access
- Media locations:
  - Jellyfin library: `/evil_ripple/media/library/` (movies/TV/music)
  - Photos/videos: `/evil_ripple/media/assets` (375GB)
- Ollama server: Runs on neptune at 192.168.10.4:11434
- Memory considerations: For intensive services, reduce workers, monitor available RAM

## Service Notes

- Open-WebUI: Connects to remote Ollama instance on neptune
- Jellyfin: Media server for movies/TV/music libraries
