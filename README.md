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
| Autoscaling | KEDA + KEDA HTTP Add-on |

## Apps

| App | Description |
|---|---|
| [Jellyfin](https://jellyfin.org/) | Media server (scales to 0 when idle, wakes on HTTP request via KEDA) |
| [n8n](https://n8n.io/) | Workflow automation with MCP access (scales to 0 when idle, wakes on HTTP request via KEDA) |
| [Homarr](https://homarr.dev/) | Dashboard (scales to 0 when idle, wakes on HTTP request via KEDA) |
| [Scrypted](https://www.scrypted.app/) | Home automation / camera hub |
| [AdGuard Home](https://adguard.com/adguard-home.html) | Network-wide DNS ad blocking |
| [Grafana MCP](https://github.com/grafana/mcp-grafana) | MCP server for Grafana |
| [Kubernetes MCP](https://github.com/manusa/kubernetes-mcp-server) | MCP server for Kubernetes |

## Structure

```
helmfile.yaml.gotmpl              # Root helmfile (all releases)
environments/
  local/
    env.yaml                      # Environment variables (domain, server IP)
    values/
      <chart>/values.yaml         # Per-release Helm values
manifests/
  traefik/
    gateway.yaml                  # Gateway resource
    http-redirect.yaml            # HTTP → HTTPS redirect
  cert-manager/
    cluster-issuer.yaml           # Let's Encrypt ClusterIssuer
    wildcard-certificate.yaml     # Wildcard TLS cert
  keda/
    reference-grant.yaml          # ReferenceGrant allowing HTTPRoutes to reference KEDA services
  jellyfin/
    http-route.yaml               # Gateway API HTTPRoute
    interceptor-route.yaml        # InterceptorRoute (KEDA HTTP Add-on routing + scaling metric)
    scaled-object.yaml            # ScaledObject (min 0, max 1, 10 min cooldown)
    pvcs.yaml                     # PersistentVolumeClaims
  n8n/
    http-route.yaml               # Gateway API HTTPRoute
    interceptor-route.yaml        # InterceptorRoute (KEDA HTTP Add-on routing + scaling metric)
    scaled-object.yaml            # ScaledObject (min 0, max 1, 10 min cooldown)
    pvcs.yaml                     # PersistentVolumeClaims
    daily-brief-cronjob.yaml      # CronJob that POSTs to the daily brief webhook at 07:00 EET
  homarr/
    http-route.yaml               # Gateway API HTTPRoute
    interceptor-route.yaml        # InterceptorRoute (KEDA HTTP Add-on routing + scaling metric)
    scaled-object.yaml            # ScaledObject (min 0, max 1, 10 min cooldown)
    pvc.yaml                      # PersistentVolumeClaim
  adguard/
    http-route.yaml               # Gateway API HTTPRoute
    pvcs.yaml                     # PersistentVolumeClaims
  scrypted/
    http-route.yaml               # Gateway API HTTPRoute
    pvcs.yaml                     # PersistentVolumeClaims
  monitoring/
    http-route.yaml               # Gateway API HTTPRoute (Grafana)
    pvcs.yaml                     # PersistentVolumeClaims
    dashboard-homelab.yaml        # Grafana dashboard ConfigMap
  mcp/
    http-route.yaml               # Gateway API HTTPRoute (MCP servers)
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
| `openclaw-gateway` | n8n daily brief workflow access to the OpenClaw Gateway |

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
