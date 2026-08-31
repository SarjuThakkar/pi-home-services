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
| dreame-mcp | [dreame-vacuum-mcp](https://github.com/SarjuThakkar/dreame-vacuum-mcp) | Yes -- `vacuum.sarjuthakkar.com` | 8003 |
| nest-mcp | [nest-thermostat-mcp](https://github.com/SarjuThakkar/nest-thermostat-mcp) | Yes -- `thermostat.sarjuthakkar.com` | 8004 |
| matter-server | (upstream image) | No (LAN only) | 5580 |

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
  dreame-mcp.env              <- real secrets, NOT in git
  nest-mcp.env                <- real secrets, NOT in git
  skylight-mcp.env            <- real secrets, NOT in git
  chess-mcp/                  <- clone of trmnl-chess-mcp
  skylight-mcp/               <- clone of skylight-mcp-pebble
  treehouse-library/          <- clone of TreehouseLibrary
  dreame-mcp/                 <- clone of dreame-vacuum-mcp
  nest-mcp/                   <- clone of nest-thermostat-mcp
```

**The live tunnel config is `/etc/cloudflared/config.yml`** -- that is what the
systemd unit loads (`cloudflared --config /etc/cloudflared/config.yml tunnel
run`). It is a **symlink to this repo's `cloudflared/config.yml`**, so editing
the file here *is* editing the live config; only a `systemctl restart
cloudflared` is needed to apply it.

This is worth stating plainly because it was previously three separate copies
(`/etc/cloudflared/`, `~/.cloudflared/`, and this repo) that silently drifted:
a hostname added to the repo copy had no effect and returned 404 from the
tunnel's catch-all. `~/.cloudflared/config.yml` still exists but nothing reads
it. The tunnel credentials JSON (never committed) does live in
`~/.cloudflared/`.

treehouse-library's live `library.db` and `.env` are bind-mounted from
their original location at `/home/treehouse/Documents/TreehouseLibrary-main/`
rather than living inside `~/services/` -- that's the pre-existing data
directory from before this was containerized, kept in place on purpose so
there's one source of truth for it.

## Matter (the vacuum)

`matter-server` is the Matter controller; `dreame-mcp` is a thin MCP layer on
top of it. Two things about this setup are easy to get wrong:

**IPv6 must be enabled on the LAN interface.** Matter runs over IPv6
link-local multicast and will silently discover nothing without it. This Pi is
on WiFi, and `wlan0` had IPv6 disabled outright
(`net.ipv6.conf.wlan0.disable_ipv6 = 1`) because its netplan config specified
no IPv6 settings at all, so NetworkManager defaulted to `ipv6.method: ignore`.
Fixed by adding `dhcp6: true` and `accept-ra: true` under `wlan0` in
`/etc/netplan/90-NM-*.yaml` and running `netplan apply`. Verify with:

```bash
ip -6 addr show dev wlan0        # want an fe80::/64 link-local address
```

**`network_mode: host` on the matter-server container is required**, not a
convenience — Docker's bridge NAT breaks mDNS and IPv6 link-local. That's also
why `dreame-mcp` reaches it via `host.docker.internal` rather than a compose
service name.

Commissioning a device is a one-time step. Put the device into Matter pairing
mode in its vendor app (the window is short — around 15 minutes), then:

```bash
docker exec -i matter-server python3 -c "
import asyncio, json, aiohttp
async def m():
    async with aiohttp.ClientSession() as s:
        async with s.ws_connect('http://localhost:5580/ws') as ws:
            await ws.receive_json()
            await ws.send_json({'message_id':'1','command':'commission_with_code',
                                'args':{'code':'<PAIRING CODE>','network_only':True}})
            while True:
                r = await ws.receive_json()
                if r.get('message_id')=='1': print(json.dumps(r)[:400]); break
asyncio.run(m())"
```

Fabric state persists in the `matter-data` volume, so this survives container
restarts and redeploys. Re-commissioning is only needed if that volume is
deleted or the device is factory reset.

## First-time setup on a fresh Pi

1. Install Docker: `curl -fsSL https://get.docker.com | sudo sh && sudo usermod -aG docker $USER` (re-login after)
2. Install `cloudflared` (see [Cloudflare's docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/install-and-setup/installation/)), then `cloudflared tunnel login` and `cloudflared tunnel create pebble-services`
3. Clone this repo and each service repo as siblings under `~/services/` (see layout above)
4. Copy `.env.example/*.env.example` to `~/services/*.env` and fill in real values
5. Update this repo's `cloudflared/config.yml` so `tunnel:` and `credentials-file:` match your tunnel's actual ID, then point the live path at it:
   `sudo ln -s ~/services/cloudflared/config.yml /etc/cloudflared/config.yml`
6. `cloudflared tunnel route dns pebble-services <hostname>` for each public service, then `sudo cloudflared service install && sudo systemctl start cloudflared`
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

## Adding a new service

1. Clone the new service's repo into `~/services/<new-service>/`
2. Add a service block to `docker-compose.yml` here, following the existing
   pattern (build path, a host port not already in use, `restart:
   unless-stopped`, an `env_file` if it needs secrets)
3. If it needs to be reachable by Pebble's cloud agent, add an `env.example`
   for it here, add a hostname entry to `cloudflared/config.yml`'s
   `ingress:` list **above the `http_status:404` catch-all** (rules match in
   order, so anything below it is dead), then on the Pi:
   `cloudflared tunnel route dns pebble-services <new-hostname>` and
   `sudo systemctl restart cloudflared`
4. If it needs local-network-only access (like Matter/Thread device
   control, which requires being on the same LAN as the device and won't
   work through a tunnel), skip the Cloudflare Tunnel step entirely --
   same as treehouse-library.
5. Add the new service's directory to `.gitignore` here -- each service is
   its own clone, and without this its whole source tree gets committed into
   this repo by accident.
6. Commit and push the updated `docker-compose.yml` / cloudflared config
   here so this repo stays the accurate source of truth.

## SSH access

Key-based auth only (`treehouse@192.168.1.135`, or `treehouse@treehouse-pi`
over Tailscale if not on the same LAN). No password auth.

## Secrets

Every `*.env` file with real values is gitignored -- only the
`.env.example/` templates are committed. The Cloudflare Tunnel credentials
JSON (`~/.cloudflared/<tunnel-id>.json`) is never committed either; it's
tied to this specific Pi's tunnel registration.
