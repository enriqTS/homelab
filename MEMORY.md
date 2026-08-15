# Memory

## Repository conventions
- This homelab repository deploys Ollama on the CachyOS desktop through Docker Compose at `llm-stack/ollama/docker-compose.yml`.
- Ollama model data is runtime state and includes private key material. Keep `llm-stack/ollama/ollama/` ignored and never commit it.

## Current hardware and runtime
- Desktop GPU: NVIDIA RTX 3060 12 GB; host RAM: 46 GB.
- Ollama API is expected on `http://127.0.0.1:11434` for local coding agents.
- Docker is enabled at system boot. Compose uses `restart: always`, one resident model, a 32K context, and a five-minute idle keep-alive.

## Coding-agent integration
- Host `codex`, `claude`, and `pi` commands are OpenShell launchers. Their disposable sandboxes do not import host local-provider files; use `codex direct`, `claude direct`, and `pi direct` for host-local Ollama unless OpenShell is intentionally extended later.
- OpenCode runs directly on the host.
- Active host provider files are `~/.codex/config.toml`, `~/.claude/settings.json`, `~/.pi/agent/models.json`, and `~/.config/opencode/opencode.jsonc`. LM Studio entries were removed in August 2026.
- Model selection itself does not reliably notify Ollama. First inference JIT-loads a model; `OLLAMA_MAX_LOADED_MODELS=1` replaces the resident model on a switch, and `OLLAMA_KEEP_ALIVE=5m` handles deselection/client-exit by idle expiry.
