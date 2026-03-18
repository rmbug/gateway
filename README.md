# rmBug Gateway

The rmBug Gateway is a self-hosted database proxy that sits on your infrastructure. Your engineers connect through it using identity-based access — no shared credentials, no VPN required.

## Quick start

```bash
docker run -d \
  -e RMBUG_GATEWAY_ACCESS_KEY=rmb_gw_your_key_here \
  -v rmbug-data:/var/lib/rmbug \
  ghcr.io/rmbug/gateway:latest
```

The gateway registers itself with the rmBug control plane and starts accepting connections. Get your access key from the rmBug dashboard under **Settings → Gateways**.

## Docker Compose

```yaml
services:
  gateway:
    image: ghcr.io/rmbug/gateway:latest
    restart: unless-stopped
    environment:
      RMBUG_GATEWAY_ACCESS_KEY: ${RMBUG_GATEWAY_ACCESS_KEY}
    volumes:
      - rmbug-data:/var/lib/rmbug

volumes:
  rmbug-data:
```

## Environment variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `RMBUG_GATEWAY_ACCESS_KEY` | Yes | — | API key from the rmBug dashboard (`rmb_gw_...`) |
| `RMBUG_GATEWAY_FQDN` | No | `localhost` | Your gateway's public hostname. Set this when deploying with a public domain |
| `RMBUG_RELAY_FQDN` | No | — | Override the relay FQDN (auto-discovered from the control plane if not set) |
| `RMBUG_STORAGE_URI` | No | `sqlite:///var/lib/rmbug/gateway.db` | Audit log storage. SQLite (default), PostgreSQL (`postgres://...`), or MySQL (`mysql://...`) |
| `RMBUG_TLS_LISTEN_PORT` | No | `8443` | TLS port inside the container. Map to 443 externally |
| `RMBUG_AUDIT_BUFFER_SIZE` | No | `1000` | Max audit events buffered in memory before flushing |
| `RMBUG_AUDIT_FLUSH_INTERVAL` | No | `10s` | How often buffered audit events are written to storage |

## Supported databases

| Database | Versions |
|----------|----------|
| PostgreSQL | 14, 15, 16, 17, 18 |
| MySQL | 8.0+ |

## How it works

The gateway forwards database traffic transparently after authentication — it doesn't parse the database protocol. This means prepared statements and every other protocol feature work exactly as they would with a direct connection.

When an engineer connects:
1. Their rmBug agent sends a JWT to the gateway
2. The gateway validates the JWT against the control plane
3. The gateway opens a connection to the target database using the stored credentials
4. Traffic flows transparently in both directions

Your database credentials never leave your infrastructure.

## Audit log persistence

The gateway stores audit logs locally using SQLite by default (`/var/lib/rmbug/gateway.db`). **If you don't mount a persistent volume, audit logs are lost when the container restarts.**

There are three ways to make audit logs durable:

**Option 1: Named Docker volume (simplest)**
```bash
docker run -d \
  -e RMBUG_GATEWAY_ACCESS_KEY=rmb_gw_your_key_here \
  -v rmbug-data:/var/lib/rmbug \         # ← persistent named volume
  ghcr.io/rmbug/gateway:latest
```

**Option 2: External PostgreSQL**
```bash
docker run -d \
  -e RMBUG_GATEWAY_ACCESS_KEY=rmb_gw_your_key_here \
  -e RMBUG_STORAGE_URI=postgres://user:pass@db-host/gateway \
  ghcr.io/rmbug/gateway:latest
```

**Option 3: External MySQL**
```bash
docker run -d \
  -e RMBUG_GATEWAY_ACCESS_KEY=rmb_gw_your_key_here \
  -e RMBUG_STORAGE_URI=mysql://user:pass@db-host/gateway \
  ghcr.io/rmbug/gateway:latest
```

PostgreSQL or MySQL are the right choice if you're running multiple gateway replicas — they share a single audit store. SQLite with a persistent volume works fine for a single instance.

Without a persistent store, the gateway still works — engineers can still connect, and connections are still logged in the rmBug control plane. Only the local audit copy is lost on restart.

## Upgrading

```bash
docker pull ghcr.io/rmbug/gateway:latest
docker stop <container> && docker rm <container>
# re-run with the same docker run / compose command
```

The volume persists across upgrades, so audit history is preserved.

## Full documentation

→ [rmbug.com/docs/gateway](https://rmbug.com/docs/gateway)
