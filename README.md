# Homelab

Home server infrastructure for self-hosted services. Split across 3 nodes:
Raspberry Pi (DNS), Mini PC (services), Desktop (LLM). Tailscale runs natively on each node.

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
| Tailscale (native daemon) | — | All | VPN/mesh network |

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

### Search

| Service | Port | Node | Purpose |
|---------|------|------|---------|
| SearXNG | 8080 | Mini PC | Privacy-focused metasearch engine |
| Vane (Perplexica) | 3003 | Mini PC | AI answering engine on top of SearXNG (uses Ollama) |

### Local LLM

| Service | Port | Node | Purpose |
|---------|------|------|---------|
| Ollama | 11434 | Desktop | LLM runtime (API) |
| Open WebUI | 3002 | Desktop | Web chat interface |

## Deployment

Deploy nodes with the Ansible playbooks in [`ansible/`](ansible/README.md), rather
than running individual Compose projects manually. Ansible installs Docker, clones
this repository to `/opt/homelab` on each managed host, and starts that node's
stacks in the required order. The desktop playbook also installs the NVIDIA
Container Toolkit and pulls its configured Ollama models. Install and authorize
native Tailscale separately on every node before deployment.

### 1. Configure the inventory

On the Ansible control machine, clone this repository and update
[`ansible/inventory/hosts.yml`](ansible/inventory/hosts.yml) with each node's IP
address, SSH user, repository URL/branch, and the Ollama models to pull. The
included inventory contains example private-network addresses and usernames.

### 2. Install the Ansible dependency

```bash
cd ~/Projetos/homelab/ansible
ansible-galaxy collection install -r requirements.yml
```

Ensure the control machine can SSH to every host and that the configured user has
sudo access. For password-based SSH or sudo, add `--ask-pass` and/or
`--ask-become-pass` to the commands below.

### 3. Run the playbook for each node

```bash
cd ~/Projetos/homelab/ansible

# Raspberry Pi: Pi-hole
ansible-playbook playbooks/pihole.yml

# Mini PC: proxy, monitoring, dashboard, search, and Vane
ansible-playbook playbooks/minipc.yml

# Desktop: Ollama, Open WebUI, NVIDIA toolkit, and Ollama models
ansible-playbook playbooks/desktop.yml
```

The playbooks are idempotent; rerun the relevant one after changing the repository
or a node's configuration. See [`ansible/README.md`](ansible/README.md) for the
managed stacks and common commands.

### 4. Complete first-run setup

Confirm that native Tailscale is authorized on every node:

```bash
tailscale status
```

Then set the Pi-hole password and point the router's DNS at the Pi:

```bash
ssh pi@<pi-ip> 'sudo docker exec -it pihole pihole -a -p'
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

RTX 3060 12GB — NVIDIA GPU support is pre-configured in Ollama's Compose file.
The desktop Ansible playbook installs the NVIDIA Container Toolkit and restarts
Docker when needed. Install the NVIDIA drivers on CachyOS before running it.

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
- Tailscale runs as the native `tailscaled` system service on every node; it is not deployed as a Compose stack.
- Authorize each node with `sudo tailscale up`, then verify it with `tailscale status`.
- Use MagicDNS for internal service discovery (for example, `<host>.<tailnet>.ts.net`).

### Pi-hole
- Admin UI: `http://<pi-ip>:8081`
- Set upstream DNS in admin panel (e.g., `1.1.1.1`, `9.9.9.9`)
- Optional: enable DHCP server in admin panel (may conflict with router)

### SearXNG
- Admin UI: `http://<mini-pc-ip>:8080`
- `searxng/config/settings.yml` enables JSON output and the wolframalpha engine, both required by Vane; it is tracked in git despite `**/config/` being ignored
- Secret key comes from `SEARXNG_SECRET` in the compose file (`openssl rand -hex 32`); replace the placeholder before exposing the instance
- Valkey provides caching/rate-limiting for the limiter
- Set `SEARXNG_BASE_URL` to your Tailscale URL (e.g., `http://searxng.tailnet.ts.net/`) for correct links
- Expose via Nginx Proxy Manager or Tailscale for remote access

### Vane (Perplexica)
- UI: `http://<mini-pc-ip>:3003` — an AI answering engine that queries SearXNG and answers using the local Ollama model (`lfm2.5`)
- `SEARXNG_API_URL` points at the co-located SearXNG over the shared `searxng-net` Docker network
- `OLLAMA_BASE_URL` points at Ollama on the Desktop over Tailscale (e.g., `http://henrique-desktop.tailnet.ts.net:11434`); replace with your Desktop's MagicDNS name
- On first start, the Ollama provider is auto-configured from env and lists all local models (`lfm2.5` included); pick it in the setup screen at `http://<mini-pc-ip>:3003`
- Data/config is persisted in `perplexica/data/` (gitignored)

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
