# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

Multi-host Docker Compose infrastructure repository managing personal services across two physical hosts:
- `sol/` - Primary host (Intel N100, 16GB RAM, ZFS raidz2 array)
- `neptune/` - Secondary host (runs Ollama and Kokoro)

## Key Commands

### Deployment
Each host has its own docker-compose.yml file. Deploy by navigating to the host directory:
```bash
cd sol && docker compose up -d
cd neptune && docker compose up -d
```

### Managing Services
```bash
# View logs for a service
docker compose logs -f [service_name]

# Rebuild and redeploy single service
docker compose build [service_name] && docker compose up -d [service_name]

# Pull latest images and restart
docker compose pull && docker compose up -d
```

### Remote Host Management
Uses `mise.toml` files in each directory to configure DOCKER_HOST. For example, `sol/mise.toml` contains `DOCKER_HOST = "ssh://node1"` to execute docker commands on the remote host via SSH.

## Architecture

### Network Topology
Services use named networks for isolation and communication:
- `web` - Local reverse proxy network (Caddy to services on sol)
- `web_cloudflare` - Cloudflare tunnel network (cloudflared to services)
- `postgres` - Database network
- `valkey` - Redis-compatible cache network
- `host` - Used for UniFi, Syncthing, RustDesk that need direct host network access

### External Access Strategy
Two methods for external access:
1. **Caddy reverse proxy** (`web` network) - Uses Cloudflare DNS challenge for TLS certificates via `tls { dns cloudflare {env.CF_API_TOKEN} }` in Caddyfile
2. **cloudflared tunnel** (`web_cloudflare` network) - For services behind Cloudflare Access authentication

### Storage Patterns
- Sol host volumes: `/evil_ripple/docker-volumes/{service_name}`
- Neptune host volumes: `/stable_grace/docker-volumes/{service_name}`
- Media library (Jellyfin): `/evil_ripple/media/library/{movies,tv,music}`

### Custom Images
- Caddy: Built with Cloudflare DNS plugin using multi-stage Dockerfile (`caddy.Dockerfile`)
- Some services build from GitHub repos: `build: "https://github.com/user/repo.git#branch"`

### Service Configuration
- Secrets stored in per-service `.env` files (gitignored)
- Most services use `restart: unless-stopped`
- Custom ports exposed only when needed (most services accessed via reverse proxy)

## Important Service Relationships

- **Open-WebUI** → Ollama: Open-WebUI on sol connects to Ollama on neptune at `192.168.10.4:11434`
- **Flask apps** → Postgres: flask-password and flask-imghost both connect to postgres service via `postgres` network
- **n8n** → Valkey: n8n uses valkey for caching/queue

## Host-Specific Details

### Sol (Primary Host)
- Intel N100, 16GB RAM - resource-constrained for intensive workloads
- ZFS raidz2 array at `/evil_ripple` (5 disks)
- Runs reverse proxy (Caddy), cloudflared tunnel, databases, media servers, and most web applications

### Neptune (Secondary Host)
- Runs compute-intensive AI workloads (Ollama)
- Volumes at `/stable_grace/docker-volumes/`

## Conventions

- Use official Docker images when available
- Pin specific versions for production services where stability is critical (e.g., `gollumwiki/gollum:6.1.0`)
- Use digest pinning for custom/private images (e.g., `public.ecr.aws/m3l3l1j4/flask-imghost@sha256:...`)
- Environment variables in compose file for non-sensitive config, `.env` files for secrets
- Named volumes not used - all volumes are bind mounts to host paths
