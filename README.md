# Homelab

Kubernetes homelab running on a single-node k3s cluster. Deployed with [Helmfile](https://helmfile.readthedocs.io/).

## Architecture

### Network entry

All traffic originates from the internet and resolves through **Cloudflare**, which acts as the authoritative DNS provider for the domain. Cloudflare is also used for the DNS-01 ACME challenge: **cert-manager** issues wildcard TLS certificates from Let's Encrypt by writing temporary DNS records via the Cloudflare API, so no port 80 exposure is needed.

Every inbound HTTPS request lands on **Traefik**, running as the cluster's Gateway API implementation. Traefik terminates TLS using the wildcard certificate and routes requests to the appropriate backend based on Gateway API `HTTPRoute` resources.

### Scale-to-zero apps

Three apps — **Jellyfin**, **n8n**, and **Homarr** — are configured to scale down to zero replicas when idle, using the **KEDA HTTP Add-on**. Each has an `InterceptorRoute` that sits between Traefik and the app: when a request arrives for a scaled-down pod, the interceptor holds the connection, signals KEDA to scale the deployment to one replica, and forwards traffic once the pod passes its readiness probe. After 10 minutes of no traffic the pod scales back to zero. A `ReferenceGrant` allows the KEDA namespace to be referenced by the app HTTPRoutes.

### Always-on apps

**Scrypted** runs continuously as the home automation and camera hub. **AdGuard Home** provides network-wide DNS ad blocking and also serves as the cluster's internal DNS resolver. Two **MCP servers** run as always-on services: one for Grafana and one for Kubernetes — both expose their respective APIs over the Model Context Protocol so n8n workflows can query them directly without additional credentials being embedded in workflow definitions.

### Monitoring

**Alloy** runs as a cluster-wide collector, scraping Prometheus metrics from all instrumented workloads and collecting container logs. Metrics are shipped to **Prometheus** and logs to **Loki**, both running in the `monitoring` namespace with persistent storage. **Grafana** is the single UI for both data sources and is exposed through Traefik like any other app. A homelab-specific Grafana dashboard is pre-loaded via a `ConfigMap`.

### Storage

All stateful workloads — Jellyfin, n8n, Homarr, AdGuard Home, Scrypted, Prometheus, and Loki — use **OpenEBS hostpath** `PersistentVolumeClaims`. Since this is a single-node cluster there is no need for distributed storage; hostpath provisioning gives direct access to the node filesystem with minimal overhead.

### Secrets

All credentials (Cloudflare API token, Grafana admin password, SMTP credentials, Homarr encryption key, Grafana MCP API key, OpenClaw gateway token) are stored as Kubernetes `Secret` objects and referenced by name in Helm values and manifests.

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
