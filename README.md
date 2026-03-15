# Vibe Kanban — Self-Hosted Deployment

A collaborative kanban platform backed by a **Rust API server**, **ElectricSQL** real-time sync, and **PostgreSQL**. This repository contains the production-ready Helm chart and tooling to build, push, and deploy Vibe Kanban on any Kubernetes cluster.

## Architecture

| Component       | Description                                                                     | Default Port |
|-----------------|---------------------------------------------------------------------------------|--------------|
| **Server**      | Rust API + Vite frontend (serves the UI and REST/WebSocket endpoints)           | 8081         |
| **ElectricSQL** | Real-time sync engine — streams PostgreSQL changes to clients                   | 3000         |
| **PostgreSQL**  | Primary data store (`wal_level=logical` for ElectricSQL replication)            | 5432         |
| **Relay**       | Optional tunnel/relay server for NAT traversal and desktop-client connectivity  | 8082         |
| **Worker**      | Coding-agent runner / desktop-client daemon (Node.js, includes Claude Code CLI) | 3000         |

## Building Docker Images

The helper script `build-docker-images.sh` clones the upstream [BloopAI/vibe-kanban](https://github.com/BloopAI/vibe-kanban) repository and builds three images. Run it from the repo root:

```bash
./build-docker-images.sh
```

You will be prompted for:

| Prompt                    | Default                         | Notes                                                           |
|---------------------------|---------------------------------|-----------------------------------------------------------------|
| `REMOTE_SERVER_TAG`       | `remote-v0.1.22`                | Git tag to checkout; used as the server image tag               |
| `WORKER_TAG`              | `v0.1.30`                       | npm package version for `vibe-kanban` worker                    |
| `DOCKER_REGISTRY`         | `your-registry`                 | Container registry prefix (e.g. `harbor.example.com/myproject`) |
| `VITE_RELAY_API_BASE_URL` | `https://relay.your-domain.com` | **Build-time** argument baked into the Vite frontend bundle     |

### Images produced

| Image                                                     | Dockerfile                                  | Purpose                            |
|-----------------------------------------------------------|---------------------------------------------|------------------------------------|
| `<DOCKER_REGISTRY>/vibe-kanban:<REMOTE_SERVER_TAG>`       | `crates/remote/Dockerfile` (upstream)       | Server — Rust API + Vite SPA       |
| `<DOCKER_REGISTRY>/vibe-kanban:relay-<REMOTE_SERVER_TAG>` | `crates/relay-tunnel/Dockerfile` (upstream) | Relay / tunnel server              |
| `<DOCKER_REGISTRY>/vibe-kanban:worker-<WORKER_TAG>`       | `Dockerfile-worker` (this repo)             | Worker — Node.js + Claude Code CLI |

> **Important:** `VITE_RELAY_API_BASE_URL` is a **build-time** variable. It is embedded into the compiled frontend assets during `docker build`. Changing the relay URL requires rebuilding the server image.

### Worker Image Details

The worker image (`Dockerfile-worker`) is based on `node:24-alpine` and installs:

- **git** — for worktree operations in workspaces
- **@anthropic-ai/claude-code** — Claude Code CLI (globally installed)
- **vibe-kanban** — the worker npm package at the specified version

The container runs as the non-root `node` user.

## Known Issues

### Virtuoso.dev License Warning

The Vite frontend uses the [Virtuoso](https://virtuoso.dev) library for virtualized list rendering. In self-hosted deployments, you may see the following console error:

> `Your VirtuosoMessageListLicense is missing a license key. Purchase one from https://virtuoso.dev/pricing/`

This is a **known upstream issue** ([BloopAI/vibe-kanban#770](https://github.com/BloopAI/vibe-kanban/issues/770)). The warning does not block functionality but may cause visual glitches in the message list component on the server-hosted UI. Possible workarounds:

1. **Purchase a Virtuoso license** and provide the key via the application configuration
2. **Track the upstream issue** for a resolution or migration to an alternative library
3. The warning is cosmetic in most scenarios and can be safely ignored for internal/dev deployments

### CORS Failure When Remote Server Uses a Private IP

If you run the remote server on a **private IP** (e.g. `10.x.x.x`, `172.16.x.x`, `192.168.x.x`) and attempt to connect a worker or browser client from a **public IP**, the connection will fail with CORS errors. The client-side code constructs queries without Origin headers, so the server doesnt build responses with CORS headers and the browser blocks the request from public networks to private. Some kind of default browser policy.

**Workaround:** Bind the remote server to ingress with a **public IP**. You can use allowlist to restrict access to the remote server from only your network.

## Helm Chart

### Prerequisites

- Kubernetes 1.26+
- Helm 3.12+
- A container registry with the built images
- (Optional) cert-manager for automated TLS
- (Optional) ingress-nginx controller

### Quick Start

```bash
helm install vibe-kanban ./helm/vibe-kanban \
  --namespace vibe-kanban --create-namespace \
  --set server.image.repository=harbor.example.com/myproject/vibe-kanban \
  --set server.image.tag=remote-v0.1.22 \
  --set relay.image.repository=harbor.example.com/myproject/vibe-kanban \
  --set relay.image.tag=relay-remote-v0.1.22 \
  --set worker.image.repository=harbor.example.com/myproject/vibe-kanban \
  --set worker.image.tag=worker-v0.1.30 \
  --set ingress.enabled=true \
  --set ingress.host=kanban.example.com \
  --set relay.ingress.enabled=true \
  --set relay.ingress.host=relay.example.com
```

### Secrets Management

The chart supports two approaches:

**Option A — External secret (recommended for production):**

```bash
--set secrets.existingSecret=my-vibe-kanban-secret
```

**Option B — Chart-managed secret:**

```bash
--set secrets.jwtSecret=$(openssl rand -base64 48) \
--set secrets.dbPassword=$(openssl rand -base64 24) \
--set secrets.electricRolePassword=$(openssl rand -base64 24)
```

When using Option B with empty values, the chart auto-generates secure random secrets on first install and preserves them across upgrades via Helm `lookup`.

#### Required Secret Keys

| Key                            | Consumed By             | Description                           |
|--------------------------------|-------------------------|---------------------------------------|
| `VIBEKANBAN_REMOTE_JWT_SECRET` | server, worker, relay   | JWT signing secret                    |
| `DB_PASSWORD`                  | postgres, server, relay | PostgreSQL application user password  |
| `ELECTRIC_ROLE_PASSWORD`       | electric                | ElectricSQL replication role password |

#### Optional Secret Keys

| Key                                                     | Description                       |
|---------------------------------------------------------|-----------------------------------|
| `GITHUB_OAUTH_CLIENT_ID` / `GITHUB_OAUTH_CLIENT_SECRET` | GitHub OAuth app credentials      |
| `GOOGLE_OAUTH_CLIENT_ID` / `GOOGLE_OAUTH_CLIENT_SECRET` | Google OAuth app credentials      |
| `LOOPS_EMAIL_API_KEY`                                   | Loops transactional email API key |

### Ingress Configuration

The chart creates separate Ingress resources for the server, relay, and worker (each independently toggled).

#### Enabling HTTP/2

**HTTP/2 must be enabled on the ingress** for optimal performance. The Vite frontend and ElectricSQL real-time sync benefit significantly from HTTP/2 multiplexing — without it, users will experience degraded performance due to head-of-line blocking on concurrent API and sync requests.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
data:
  use-http2: "true"
```

> **Note:** The relay ingress explicitly sets `proxy-http-version: "1.1"` because its WebSocket/tunnel connections require HTTP/1.1 upgrade semantics. Do **not** force HTTP/2 on the relay ingress.

#### Relay Wildcard Ingress

The relay ingress creates two rules: one for the base host and one for `*.<host>` to support tunnel subdomains. Ensure your TLS certificate covers the wildcard domain (e.g. `*.relay.example.com`).

### Network Policies

Enable the default-deny network policies for a hardened deployment:

```bash
--set networkPolicy.enabled=true
```

This creates fine-grained policies that restrict traffic between components to only the required paths, following the principle of least privilege. The policies are fully customizable via `values.yaml`:

```yaml
networkPolicy:
  enabled: true
  ingressController:
    namespaceSelector:
      kubernetes.io/metadata.name: ingress-nginx
    podSelector:
      app.kubernetes.io/instance: ingress-nginx
  externalEgressExcludeCIDRs:
    - 10.0.0.0/8
    - 172.16.0.0/12
    - 192.168.0.0/16
  # Per-component extra rules (postgres, electric, server, relay, worker)
  postgres:
    extraIngress: []
    extraEgress: []
```

### Worker Pre-Start Script

The worker supports an optional init container that runs before the main process starts — useful for cloning repositories, installing packages, or bootstrapping SSH keys:

```yaml
worker:
  preStartScript:
    enabled: true
    runAsRoot: false
    script: |
      git clone https://github.com/org/repo /home/workspace/myrepo
```

### Pod Disruption Budget

A PodDisruptionBudget is enabled by default for the server, keeping at least one replica available during voluntary disruptions (node drains, upgrades).

## Security Hardening

The chart follows [NSA/CISA Kubernetes Hardening Guide](https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF) recommendations:

- All workloads run as **non-root** (except ElectricSQL, which requires root due to upstream image constraints)
- `allowPrivilegeEscalation: false` on all containers
- `capabilities.drop: [ALL]` — no Linux capabilities
- `seccompProfile: RuntimeDefault` — syscall filtering
- `automountServiceAccountToken: false` — no unnecessary API server access
- Secret rotation via `checksum/secret` pod annotation (rolling restart on change)
- Optional `NetworkPolicy` for microsegmentation

## Configuration Reference

All configurable values are documented inline in [`helm/vibe-kanban/values.yaml`](helm/vibe-kanban/values.yaml). Key sections:

| Section                 | Description                                                      |
|-------------------------|------------------------------------------------------------------|
| `secrets.*`             | Credentials management (JWT, DB, OAuth, email)                   |
| `postgres.*`            | Bundled PostgreSQL StatefulSet (image, storage, resources)       |
| `electric.*`            | ElectricSQL sync engine                                          |
| `server.*`              | Rust API server (image, env, resources, security)                |
| `relay.*`               | Tunnel/relay server (image, env, ingress, resources)             |
| `worker.*`              | Worker daemon (image, env, persistence, preStartScript, ingress) |
| `ingress.*`             | Main server ingress (host, TLS, annotations)                     |
| `networkPolicy.*`       | Network segmentation toggle                                      |
| `podDisruptionBudget.*` | PDB for server availability                                      |
