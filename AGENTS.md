# AGENTS.md

Guidelines for agentic coding agents working in this Docker Compose infrastructure repository.

## Repository Overview

Multi-host Docker Compose infrastructure managing personal services across three physical hosts:
- `sol/` - Primary host (Ryzen 7, 32GB RAM, ZFS raidz2 at `/evil_ripple`)
- `neptune/` - Secondary host (AI workloads, volumes at `/stable_grace`)
- `ionos/` - Remote VPS (Tailscale exit node, proxies)
- `ansible/` - Ansible playbooks for host configuration

## Key Commands

### Docker Compose Deployment
Each host directory contains its own docker-compose.yml. Deploy from host directory:
```bash
cd sol && docker compose up -d
cd neptune && docker compose up -d
cd ionos && docker compose up -d
```

### Service Management
```bash
docker compose logs -f [service_name]     # View logs
docker compose build [service_name]        # Rebuild image
docker compose up -d [service_name]        # Recreate service
docker compose pull && docker compose up -d  # Update all images
docker compose config                       # Validate compose file
```

### Remote Host Execution
Each host directory has a `mise.toml` setting `DOCKER_HOST` for remote execution:
```bash
cd sol && docker compose ps    # Runs on node1 via SSH
cd ionos && docker compose ps  # Runs on ionos via SSH
```

### Ansible Playbooks
```bash
cd ansible && ansible-playbook sshd-docker.yml    # Configure SSH for Docker
cd ansible && ansible-playbook docker-daemon.yml   # Configure Docker daemon
cd ansible && ansible-playbook samba.yml           # Configure Samba shares
cd ansible && ansible-playbook timesyncd.yml       # Configure timesyncd
```

### Validation
```bash
docker compose config --quiet  # Validate compose syntax (per host directory)
ansible-playbook --check playbook.yml  # Ansible dry-run
```

## Code Style Guidelines

### Docker Compose Files

**Structure Order:**
1. `networks:` block at top (define all networks first)
2. `volumes:` block (if using named volumes)
3. `services:` block

**Formatting:**
- Use 2-space indentation
- Service names: lowercase with hyphens (e.g., `open-webui`, `flask-password`)
- Quote port mappings: `"8080:80"` (both sides quoted)
- Use `restart: unless-stopped` for persistent services
- Use `restart: on-failure` or `restart: on-failure:1` for services that may fail

**Image Versioning:**
- Pin specific versions for production: `gollumwiki/gollum:6.1.0`
- Use `:latest` for frequently updated services where tracking latest is desired
- Use digest pinning for custom/private images: `public.ecr.aws/m3l3l1j4/flask-imghost@sha256:...`
- Prefer official images from Docker Hub, GitHub Container Registry, or trusted sources

**Volume Bind Mounts:**
- Sol host: `/evil_ripple/docker-volumes/{service_name}`
- Neptune host: `/stable_grace/docker-volumes/{service_name}`
- Use long syntax for clarity when needed:
  ```yaml
  volumes:
    - type: bind
      source: /evil_ripple/docker-volumes/caddy/data
      target: /data
  ```
- Short syntax acceptable for simple mounts:
  ```yaml
  volumes:
    - /evil_ripple/docker-volumes/n8n:/home/node/.n8n
  ```

**Networks:**
- Define networks at top of compose file
- Use descriptive names: `web`, `postgres`, `valkey`
- For external networks, use `external: true` with explicit `name:`
  ```yaml
  networks:
    postgres:
      external: true
      name: postgres
  ```

**Environment Variables:**
- Non-sensitive config: `environment:` block in compose file
- Secrets: `env_file: service.env` (these files are gitignored)
- Quote string values: `N8N_PORT: "5678"` or `N8N_PORT: 5678`

**Custom Images:**
- Use multi-stage Dockerfiles (see `sol/caddy.Dockerfile`)
- Build from GitHub repos: `build: "https://github.com/user/repo.git#branch"`

### Ansible Playbooks

**Formatting:**
- Start with `---` document marker
- Use 2-space indentation
- Use FQCNs: `ansible.builtin.copy`, `ansible.builtin.file`, etc.
- Quote mode values: `mode: "0644"`

**Structure:**
```yaml
---
- name: Descriptive playbook name
  hosts: target_host
  become: true
  tasks:
    - name: Task description
      ansible.builtin.module:
        parameter: value
      notify: Handler name

  handlers:
    - name: Handler name
      ansible.builtin.systemd_service:
        name: service
        state: restarted
```

### Caddyfile

- Use snippets for reusable config: `(cloudflare_tls)`
- Import snippets: `import cloudflare_tls`
- Group related directives
- Use consistent indentation (4 spaces)

### YAML General Rules

- 2-space indentation throughout
- No trailing whitespace
- Use `"` for strings containing special characters
- Boolean values: `true`/`false` (lowercase)

## Naming Conventions

- **Services:** lowercase with hyphens (`open-webui`, `flask-password`)
- **Container names:** only set `container_name` when necessary for clarity
- **Networks:** lowercase, descriptive (`web`, `postgres`, `valkey`)
- **Volumes:** match service name or describe purpose
- **Files:** `docker-compose.yml` (not `.yaml`), `Dockerfile` or `{name}.Dockerfile`

## Error Handling

- Use `restart: unless-stopped` for automatic recovery
- Add `healthcheck` blocks for services that support it
- Use `depends_on` with condition for service dependencies:
  ```yaml
  depends_on:
    runner-dind:
      condition: service_started
  ```
- Set appropriate `stop_grace_period` for services needing graceful shutdown

## Security Practices

- Never commit `.env` files (they are gitignored)
- Use `env_file` for secrets, not inline `environment`
- Use `network_mode: host` sparingly, only when necessary
- Apply principle of least privilege for `cap_add`
- Mount volumes as `:ro` when read-only access suffices

## Host-Specific Notes

- **Sol:** Ryzen 7, 32GB RAM. Suitable for most workloads.
- **Neptune:** Runs GPU-accelerated AI workloads (Ollama with Vulkan).
- **Ionos:** VPS with Tailscale, acts as exit node and proxy host.
