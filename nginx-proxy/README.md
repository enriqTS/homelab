# Nginx Proxy Manager

Nginx Proxy Manager (NPM) is available at
`http://henrique-notebook.tail640e58.ts.net:81`. It is the optional friendly-name
and TLS layer for the services on `henrique-notebook`.

## Before creating proxy hosts

A trusted certificate for a Tailscale-only service requires a domain you control
and a DNS provider supported by NPM's DNS challenge. HTTP-01 validation will not
work because ports 80 and 443 are intentionally not publicly reachable.

Recommended design:

1. Choose an internal subdomain of a domain you own, for example
   `home.example.com`.
2. In NPM, create a DNS-provider credential and request a DNS-01 wildcard
   certificate for `*.home.example.com` (and `home.example.com`). Keep provider
   API credentials in NPM, never in this repository.
3. Configure split DNS so Tailscale clients resolve `*.home.example.com` to
   `100.77.248.73` (`henrique-notebook`). A Pi-hole reachable through Tailscale
   can provide that zone; add it as a Tailscale split-DNS nameserver.
4. Create NPM proxy hosts using the mappings below, enable the wildcard
   certificate, **Force SSL**, and WebSocket support where required.

Until split DNS and the certificate exist, use the MagicDNS URLs and ports listed
on Homepage.

## Proxy-host targets

The NPM container can reach host-published services through
`host.docker.internal`, configured with Docker's `host-gateway` mapping.

| Friendly name | Forward host | Port |
|---|---|---:|
| `vane.home.example.com` | `host.docker.internal` | 3003 |
| `search.home.example.com` | `host.docker.internal` | 8080 |
| `grafana.home.example.com` | `host.docker.internal` | 3000 |
| `prometheus.home.example.com` | `host.docker.internal` | 9090 |
| `alerts.home.example.com` | `host.docker.internal` | 9093 |
| `portainer.home.example.com` | `host.docker.internal` | 9443 |
| `home.home.example.com` | `host.docker.internal` | 3001 |

Do not proxy NPM's own port 81 through itself. Keep management interfaces
restricted to the tailnet.
