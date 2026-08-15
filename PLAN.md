# Plan

## Objective
Configure the desktop Ollama runtime and Codex, Claude Code, pi, and OpenCode to use it instead of LM Studio, with automatic startup and predictable single-model loading/unloading.

## Approach
1. Inventory the existing Ollama deployment and CLI configurations without exposing credentials.
2. Confirm each CLI's supported Ollama/provider configuration from installed documentation and help.
3. Configure Ollama for automatic restart, one loaded model at a time, and idle unloading.
4. Remove LM Studio provider settings and add Ollama settings for each CLI where supported.
5. Validate Compose/config syntax and document operational behavior and limitations.

## Progress
- [x] Inspected repository and existing Docker Compose deployment.
- [x] Inspected CLI configurations and relevant documentation.
- [x] Added single-model/idle-unload Compose settings and removed LM Studio client providers.
- [x] Validated Compose, all client configurations, all four clients against Ollama, 32K runtime context, and five-minute expiry metadata.
- [x] Reviewed changes; ready for the final commit.

## Handoff
- Ollama is managed by Docker Compose; Docker is enabled and active at boot.
- The host commands `codex`, `claude`, and `pi` are OpenShell wrappers whose sandboxes do not import these host provider files. Local Ollama usage must use each launcher's `direct` subcommand unless OpenShell policies/state synchronization are separately extended.
- Host configurations now target Ollama at localhost: Codex OSS provider, Claude Anthropic-compatible endpoint, pi custom provider, and OpenCode provider/default model.
- Runtime model data at `llm-stack/ollama/ollama/` is ignored and must never be committed.
