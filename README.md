# Homelab

Kubernetes homelab running on a single-node k3s cluster. Deployed with [Helmfile](https://helmfile.readthedocs.io/).

## Stack

| Layer | Tool |
|---|---|
| Orchestration | k3s |
| Deployment | Helmfile |
| Ingress | Traefik (Gateway API) |
| TLS | cert-manager + Let's Encrypt (Cloudflare DNS-01) |
| Storage | OpenEBS hostpath |
| Monitoring | Prometheus + Grafana + Loki + Alloy |
| DNS | AdGuard Home |

## Apps

| App | Description |
|---|---|
| [Jellyfin](https://jellyfin.org/) | Media server |
| [n8n](https://n8n.io/) | Workflow automation |
| [Homarr](https://homarr.dev/) | Dashboard |
| [Scrypted](https://www.scrypted.app/) | Home automation / camera hub |
| [AdGuard Home](https://adguard.com/adguard-home.html) | Network-wide DNS ad blocking |
| [Grafana MCP](https://github.com/grafana/mcp-grafana) | MCP server for Grafana |
| [Kubernetes MCP](https://github.com/manusa/kubernetes-mcp-server) | MCP server for Kubernetes |
| [dav-mcp](https://github.com/PhilflowIO/dav-mcp) | MCP server for CalDAV calendars, CardDAV contacts, and VTODO tasks |

## Structure

```
helmfile.yaml.gotmpl          # Root helmfile (all releases)
environments/
  local/
    env.yaml                  # Environment variables (domain, server IP)
    values/
      <chart>/values.yaml     # Per-release Helm values
manifests/
  <app>/
    http-route.yaml           # Gateway API HTTPRoute
    pvcs.yaml                 # PersistentVolumeClaims
  cert-manager/
    cluster-issuer.yaml       # Let's Encrypt ClusterIssuer
    wildcard-certificate.yaml # Wildcard TLS cert
  traefik/
    gateway.yaml              # Gateway resource
    http-redirect.yaml        # HTTP → HTTPS redirect
```

## Secrets

All secrets are stored in Kubernetes and referenced by name — no secret values live in this repository.

| Secret | Used by |
|---|---|
| `cloudflare-api-token` | cert-manager (DNS-01 challenge) |
| `grafana-admin` | Grafana admin credentials |
| `grafana-smtp` | Grafana SMTP password |
| `homarr-secret` | Homarr encryption key |
| `grafana-mcp-apikey` | Grafana API key for the MCP server |
| `dav-mcp-secret` | CalDAV/CardDAV credentials and MCP bearer token |

## Usage

```bash
# Deploy everything
helmfile -e local apply

# Deploy only infra components
helmfile -e local apply -l domain=infra

# Deploy only apps
helmfile -e local apply -l domain=apps

# Deploy a single release
helmfile -e local apply -l component=jellyfin
```
