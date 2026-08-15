# Homelab

Home server infrastructure for self-hosted services. Split across 3 nodes:
Raspberry Pi (DNS), Mini PC (services), Desktop (LLM).

## Architecture

| Node | Role | Specs |
|------|------|-------|
| **Raspberry Pi** | DNS/Ad-blocking, Tailscale client | Pi-hole |
| **Mini PC** (Ubuntu Server) | All services, reverse proxy, monitoring, dashboard | 4-8GB RAM, NVMe SSD |
| **Desktop** (CachyOS) | Local LLM runtime | RTX 3060 12GB, 46GB RAM |

## Services

### DNS / Networking

| Service | Port | Node | Purpose |
|---------|------|------|---------|
| Pi-hole | 53 (DNS), 8081 (UI) | Pi | DNS + ad-blocking |
| Tailscale | — | All | VPN/mesh network |

### Reverse Proxy

| Service | Port | Node | Purpose |
|---------|------|------|---------|
| Nginx Proxy Manager | 80/443 (HTTP/HTTPS), 81 (UI) | Mini PC | Reverse proxy, SSL termination |

### Monitoring

| Service | Port | Node | Purpose |
|---------|------|------|---------|
| Node Exporter | 9100 (host network) | Mini PC | System metrics |
| Prometheus | 9090 | Mini PC | Time-series database |
| Grafana | 3000 | Mini PC | Dashboards + visualization |
| Alertmanager | 9093 | Mini PC | Alert routing |

### Dashboard / Management

| Service | Port | Node | Purpose |
|---------|------|------|---------|
| Homepage | 3001 | Mini PC | Service launcher dashboard |
| Portainer | 9443 (HTTPS) | Mini PC | Container management UI |

### Local LLM

| Service | Port | Node | Purpose |
|---------|------|------|---------|
| Ollama | 11434 | Desktop | LLM runtime (API) |
| Open WebUI | 3002 | Desktop | Web chat interface |

## Quick Start

### 1. Raspberry Pi (DNS)

```bash
# Clone and deploy
cd ~/Projetos/homelab/pi-hole
docker compose up -d

# Set admin password (first run)
docker exec -it pihole pihole -a -p
```

Set your router's DNS to the Pi's IP address.

### 2. Mini PC (Services)

```bash
# Clone and deploy in order
cd ~/Projetos/homelab

# Tailscale (connect all nodes)
cd tailscale && docker compose up -d
docker exec -it tailscale tailscale up   # follow auth URL

# Nginx Proxy Manager
cd ../nginx-proxy && docker compose up -d

# Grafana Stack
cd ../grafana-stack && docker compose up -d

# Homepage
cd ../homepage && docker compose up -d

# Portainer
cd ../portainer && docker compose up -d
```

### 3. Desktop (LLM)

```bash
# Ensure NVIDIA Container Toolkit is installed
# https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

cd ~/Projetos/homelab/llm-stack/ollama
docker compose up -d

# Pull models
docker exec -it ollama ollama pull llama3.2
docker exec -it ollama ollama pull mistral-nemo

# Optional: Open WebUI
cd ../open-webui && docker compose up -d
```

### Coding agents

Ollama is configured for coding-agent use with these lifecycle rules:

- Docker starts at boot and Compose uses `restart: always`, so the Ollama daemon returns after a reboot.
- A model is loaded just in time by the first inference request; merely highlighting it in a client selector may not issue a request.
- `OLLAMA_MAX_LOADED_MODELS=1` makes a request for another model replace the currently loaded model.
- `OLLAMA_CONTEXT_LENGTH=32768` avoids Ollama's 4K default while fitting this desktop's 12 GB GPU; client metadata uses the same limit.
- Coding clients do not consistently notify Ollama when a model is deselected or a client exits. `OLLAMA_KEEP_ALIVE=5m` therefore unloads it after five idle minutes.

The local client configurations live outside this repository:

| Client | Configuration / invocation |
|--------|----------------------------|
| Codex | `~/.codex/config.toml`; run `codex direct --oss -m llama3.2` |
| Claude Code | `~/.claude/settings.json`; run `claude direct --model llama3.2` |
| pi | `~/.pi/agent/models.json`; run `pi direct --model ollama/llama3.2` |
| OpenCode | `~/.config/opencode/opencode.jsonc`; select `ollama/llama3.2` |

`codex`, `claude`, and `pi` on this desktop are OpenShell launchers. Their disposable sandboxes intentionally do not import the host's provider configuration, and `localhost` inside a sandbox is not the host. Use their `direct` subcommand for the local Ollama endpoint. OpenCode currently runs directly on the host.

Add every newly pulled Ollama model to pi's and OpenCode's model lists before selecting it. Claude accepts an installed Ollama model ID through `--model`; pass the model ID to Codex with `--oss -m <model>`.

Useful lifecycle checks:

```bash
curl http://127.0.0.1:11434/api/version  # daemon health
curl http://127.0.0.1:11434/api/ps       # models currently in memory
docker exec ollama ollama stop llama3.2  # unload immediately when needed
```

## GPU Setup (Desktop)

RTX 3060 12GB — NVIDIA GPU support is pre-configured in Ollama's compose.

**Prerequisites:**
1. NVIDIA drivers installed on CachyOS
2. NVIDIA Container Toolkit: `sudo pacman -S nvidia-container-toolkit`
3. Restart Docker: `sudo systemctl restart docker`

### Model Recommendations (12GB VRAM)

| Model | Quant | VRAM | Use Case |
|-------|-------|------|----------|
| Llama 3.2 8B | Q4_K_M | ~5.5GB | General purpose, fast |
| Mistral Nemo 12B | Q4_K_M | ~7.5GB | General purpose |
| Gemma 2 9B | Q4_K_M | ~6GB | General purpose |
| Qwen 2.5 14B | Q4_K_M | ~9GB | Code + general |
| Mistral 7B | Q8_0 | ~7.5GB | Very fast |

**List models:** `docker exec -it ollama ollama list`
**Remove model:** `docker exec -it ollama ollama rm <name>`

## Configuration Notes

### Grafana
- Default credentials: `admin` / `changeme` (change in compose file!)
- Prometheus datasource auto-provisioned
- Import dashboards from Grafana.com

### Homepage
- Edit `homepage/config/settings.yaml` for layout and services
- Edit `homepage/config/services.yaml` for service groups
- Icons: use filenames from https://gethomepage.dev/latest/configs/icons/

### Nginx Proxy Manager
- Default credentials: `admin@example.com` / `changeme`
- Set up proxy hosts for internal services after deployment

### Tailscale
- Run `docker exec -it <container> tailscale up` on each node
- Use Tailscale's MagicDNS for internal service discovery
- Services reachable at `*.tailscale.local` (e.g., `grafana.tailscale.local`)

### Pi-hole
- Admin UI: `http://<pi-ip>:8081`
- Set upstream DNS in admin panel (e.g., `1.1.1.1`, `9.9.9.9`)
- Optional: enable DHCP server in admin panel (may conflict with router)

## Future Additions

- [ ] Jellyfin (media server)
- [ ] Servarr stack (Sonarr, Radarr, Lidarr, Prowlarr)
- [ ] Glance dashboard (feed aggregator)
- [ ] Watchtower (auto-update containers)
- [ ] External access setup (Cloudflare Tunnel / Tailscale Funnel)

## Notes

- All services use `restart: unless-stopped`
- Data directories are mounted as volumes for persistence
- Tailscale enables access to all services without port forwarding
- Mini PC should have Intel iGPU if adding Jellyfin for hardware transcoding
- Consider adding a UPS for the Pi and mini PC
