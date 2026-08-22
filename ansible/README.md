# Ansible deployment

These playbooks provision and deploy the homelab from an Ansible control machine.
They install Docker and its Compose support, clone the repository to `/opt/homelab`
on each target, and bring up the Compose stacks assigned to that host.

## Prerequisites

- Ansible installed on the control machine
- SSH access to the Pi, Mini PC, and desktop
- A configured SSH user with sudo privileges on every target
- Network access from targets to GitHub, Docker package repositories, and container
  registries

Install the required collection once:

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
```

## Configure targets

Edit [`inventory/hosts.yml`](inventory/hosts.yml) before deploying:

- Replace the example `ansible_host` addresses and `ansible_user` values.
- Set `repo_url`, `repo_branch`, and `repo_path` if the defaults do not apply.
- Adjust `deploy_stacks` to change the stacks deployed to a node.
- Set `ollama_models` to the models that should be pulled on the desktop.

Keep credentials out of the inventory. Use SSH keys, Ansible Vault, or external
secret management for anything sensitive.

## Deploy

Run the playbook for the node you want to manage:

```bash
cd ansible
ansible-playbook playbooks/pihole.yml
ansible-playbook playbooks/minipc.yml
ansible-playbook playbooks/desktop.yml
```

For password-authenticated SSH or sudo, append `--ask-pass` and/or
`--ask-become-pass`. Limit a run to one inventory host when needed, for example:

```bash
ansible-playbook playbooks/minipc.yml --limit minipc
```

## What each playbook deploys

| Playbook | Stacks |
|---|---|
| `pihole.yml` | Tailscale, Pi-hole |
| `minipc.yml` | Tailscale, Nginx Proxy Manager, Grafana stack, Homepage, Portainer, SearXNG, Vane (Perplexica) |
| `desktop.yml` | Tailscale, Ollama, Open WebUI; also NVIDIA Container Toolkit and configured Ollama models |

Stack order comes from `deploy_stacks` in the inventory. In particular, SearXNG
starts before Vane because they share a Docker network.

## First deployment and updates

Tailscale still requires interactive authorization after its container is started:

```bash
ssh <user>@<node> 'sudo docker exec -it tailscale tailscale up'
```

Set the Pi-hole password separately after the Pi playbook's first run:

```bash
ssh pi@<pi-ip> 'sudo docker exec -it pihole pihole -a -p'
```

The playbooks are idempotent. Rerun the appropriate playbook to fetch the configured
repository branch and reconcile its assigned Compose stacks after a change.
