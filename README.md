# pi-home-services

Docker Compose orchestration for everything running on `treehouse-pi` (a
Raspberry Pi 4, hostname `treehouse-pi` on Tailscale, LAN IP
`192.168.1.135`, user `treehouse`). Each service's actual code lives in its
own repo; this repo is just the glue that runs them together on one box,
plus the Cloudflare Tunnel config that exposes the ones Pebble needs to
reach publicly.

## Why self-hosted instead of Railway

Originally chess-mcp and skylight-mcp ran on Railway. Migrated here to cut
hosting cost, since the Pi was already on 24/7 for TreehouseLibrary and had
plenty of headroom (4 cores, 4GB RAM, ~1GB used at idle). Tradeoff: uptime
now depends on home power/internet instead of a managed cloud platform.

## Services

| Service | Repo | Public? | Port |
|---|---|---|---|
| treehouse-library | [TreehouseLibrary](https://github.com/SarjuThakkar/TreehouseLibrary) | No (LAN only, kiosk app) | 8000 |
| chess-mcp | [trmnl-chess-mcp](https://github.com/SarjuThakkar/trmnl-chess-mcp) | Yes -- `chess.sarjuthakkar.com` | 8001 |
| skylight-mcp | [skylight-mcp-pebble](https://github.com/SarjuThakkar/skylight-mcp-pebble) | Yes -- `skylight.sarjuthakkar.com` | 8002 |

Public exposure is via a named Cloudflare Tunnel (`pebble-services`,
`cloudflared` running as a systemd service), not port forwarding. Only
services Pebble's cloud agent needs to reach directly are tunneled --
treehouse-library is a household kiosk app, so it stays LAN-only.

## Directory layout on the Pi

Everything lives under `~/services/`, as siblings:

```
~/services/
  docker-compose.yml         <- from this repo
  chess-mcp.env               <- real secrets, NOT in git (see below)
  skylight-mcp.env            <- real secrets, NOT in git
  chess-mcp/                  <- clone of trmnl-chess-mcp
  skylight-mcp/                <- clone of skylight-mcp-pebble
  treehouse-library/          <- clone of TreehouseLibrary
```

`~/.cloudflared/config.yml` (from this repo's `cloudflared/config.yml`) and
the tunnel credentials JSON (never committed) live outside `~/services/`,
in the default `cloudflared` location.

treehouse-library's live `library.db` and `.env` are bind-mounted from
their original location at `/home/treehouse/Documents/TreehouseLibrary-main/`
rather than living inside `~/services/` -- that's the pre-existing data
directory from before this was containerized, kept in place on purpose so
there's one source of truth for it.

## First-time setup on a fresh Pi

1. Install Docker: `curl -fsSL https://get.docker.com | sudo sh && sudo usermod -aG docker $USER` (re-login after)
2. Install `cloudflared` (see [Cloudflare's docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/install-and-setup/installation/)), then `cloudflared tunnel login` and `cloudflared tunnel create pebble-services`
3. Clone this repo and each service repo as siblings under `~/services/` (see layout above)
4. Copy `.env.example/*.env.example` to `~/services/*.env` and fill in real values
5. Copy `cloudflared/config.yml` to `~/.cloudflared/config.yml`, update the `tunnel:` and `credentials-file:` lines to match your tunnel's actual ID
6. `cloudflared tunnel route dns pebble-services <hostname>` for each public service, then `sudo cloudflared --config ~/.cloudflared/config.yml service install && sudo systemctl start cloudflared`
7. `cd ~/services && docker compose build && docker compose up -d`

## Updating an existing service

```bash
ssh treehouse@192.168.1.135
cd ~/services/<service-name>
git pull
cd ~/services
docker compose build <service-name>
docker compose up -d <service-name>
```

For chess-mcp specifically: rebuilds recompile lc0 from source, which takes
20-40+ minutes natively on a Pi 4. Not a failure, just slow -- let it run.

## Adding a new service (e.g. a future vacuum or Nest MCP)

1. Clone the new service's repo into `~/services/<new-service>/`
2. Add a service block to `docker-compose.yml` here, following the existing
   pattern (build path, a host port not already in use, `restart:
   unless-stopped`, an `env_file` if it needs secrets)
3. If it needs to be reachable by Pebble's cloud agent, add an `env.example`
   for it here, add a hostname entry to `cloudflared/config.yml`'s
   `ingress:` list pointing at its local port, then on the Pi:
   `cloudflared tunnel route dns pebble-services <new-hostname>` and
   `sudo systemctl restart cloudflared`
4. If it needs local-network-only access (like Matter/Thread device
   control, which requires being on the same LAN as the device and won't
   work through a tunnel), skip the Cloudflare Tunnel step entirely --
   same as treehouse-library.
5. Commit and push the updated `docker-compose.yml` / cloudflared config
   here so this repo stays the accurate source of truth.

## SSH access

Key-based auth only (`treehouse@192.168.1.135`, or `treehouse@treehouse-pi`
over Tailscale if not on the same LAN). No password auth.

## Secrets

Every `*.env` file with real values is gitignored -- only the
`.env.example/` templates are committed. The Cloudflare Tunnel credentials
JSON (`~/.cloudflared/<tunnel-id>.json`) is never committed either; it's
tied to this specific Pi's tunnel registration.
