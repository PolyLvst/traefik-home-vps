# traefik-home-vps

A public-facing Traefik entrypoint (runs on a VPS) that **terminates TLS**,
runs edge security (geofencing + CrowdSec L7), and **re-encrypts** all
`*.<your-domain>` traffic to a home server reachable over Tailscale. The domain
and the home server's IP are set in `.env`.

```
client ──TLS#1──▶ VPS Traefik (:443) ─────────────▶ home server (Tailscale, :443)
                  terminates TLS,       TLS#2         terminates TLS again
                  geofence + CrowdSec   (re-encrypt)  + serves apps
```

- **TLS bridging (double TLS):** the VPS terminates the client's TLS so it can
  inspect L7, then opens a *second* TLS connection to the home server (which
  still terminates its own TLS). The VPS holds a wildcard `*.DOMAIN` cert from
  Let's Encrypt (DNS-01); the home server keeps its own certs.
- **Geofencing** (GeoBlock plugin) and **CrowdSec L7** (bouncer plugin + Traefik
  access-log scenarios) run as middleware on the decrypted request — see
  [Edge security](#edge-security-geofencing--crowdsec-l7).
- The VPS→home hop rides inside Tailscale (WireGuard), so TLS#2 is about letting
  the home server keep its own certs, not about confidentiality on that hop.

> **Prefer a thin, blind forwarder?** If you don't need edge L7, the simpler
> pure **TLS-passthrough** design (route on SNI, never decrypt, no certs on the
> VPS) lives in this repo's history — that's what commit `c139391` and earlier
> shipped.

## Setup

1. **Configure** — copy `.env.example` to `.env` and fill it in:
   ```bash
   cp .env.example .env      # then edit .env:
   #   DOMAIN=poly.my.id
   #   HOME_SERVER=100.x.y.z
   #   ACME_EMAIL=you@example.com     # Let's Encrypt account
   #   CF_DNS_API_TOKEN=...           # Cloudflare token for the DNS-01 wildcard
   #   GEO_BLACKLIST=false            # false=allowlist, true=blocklist
   #   GEO_COUNTRIES=ID               # ISO codes, e.g. "ID, SG"
   #   CROWDSEC_LAPI_KEY=...          # filled in after step 5 below
   ```
2. **DNS** — point `*.DOMAIN` at this VPS's public IP. The wildcard cert is
   issued via **DNS-01**, so you also need a DNS provider API token
   (`CF_DNS_API_TOKEN` for Cloudflare; to use another provider, change the
   resolver's `provider` in `docker-compose.yml` and set that provider's env
   vars — see the [lego docs](https://go-acme.github.io/lego/dns/)).
3. **Tailscale** — put this VPS on the same tailnet as the home server
   (`tailscale up`); `HOME_SERVER` is that server's Tailscale IP (`100.x.y.z`).
4. **Create the CrowdSec bouncer key** — the config render needs it, so bring up
   the engine first, mint the key, and paste it into `.env`:
   ```bash
   docker compose up -d crowdsec
   docker compose exec crowdsec cscli bouncers add traefik-plugin   # copy the key
   #   -> put it in .env as CROWDSEC_LAPI_KEY=...
   ```
5. **Run**
   ```bash
   docker compose up -d
   docker compose logs -f traefik      # watch the cert issue + plugins load
   ```

## How the config is built

The routing rules live in `traefik-data/templates/poly.yml.tmpl` (committed, with
`__DOMAIN__` and `__HOME_SERVER__` placeholders). On `docker compose up`, a
one-shot `config` container renders that template into
`traefik-data/dynamic/poly.yml`, filling in `DOMAIN` and `HOME_SERVER` from
`.env` (the domain's dots are escaped as `[.]` so they stay literal inside the
regex rules). Traefik's file provider then loads the dynamic dir and watches it.

This keeps the real domain and Tailscale IP out of version control — only `.env`
(git-ignored) holds them, so the repo is safe to publish.

- Edit routing in **`traefik-data/templates/poly.yml.tmpl`**, not the rendered
  `traefik-data/dynamic/poly.yml` (git-ignored and overwritten on every `up`).
- Change the domain or IP in **`.env`**, then `docker compose up -d` to re-render.

## Adding more routes

Add another HTTP router to the template (it can reuse `__DOMAIN__`). To also
serve the bare apex `DOMAIN` (not just subdomains), widen the rule:

```yaml
rule: 'HostRegexp(`^(.+\.)?__DOMAIN__$`)'
```

Attach the `geo-allow` and `crowdsec` middlewares to any new router that should
get edge filtering.

## Edge security (geofencing + CrowdSec L7)

Because the VPS now decrypts traffic, two middlewares run on every `*.DOMAIN`
request (defined in `traefik-data/templates/poly.yml.tmpl`):

- **`geo-allow`** — the [GeoBlock plugin](https://github.com/PascalMinder/geoblock).
  `GEO_BLACKLIST=false` makes `GEO_COUNTRIES` an **allowlist** (only those
  countries connect); `=true` makes it a **blocklist**. Country lookups use the
  free geojs.io API with an in-memory cache. The real client IP is used directly
  (this VPS is the internet edge), and `allowLocalRequests: true` keeps your
  tailnet/localhost exempt.
- **`crowdsec`** — the [CrowdSec bouncer plugin](https://github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin)
  in `live` mode. Each request's IP is checked against the CrowdSec engine's
  decisions (community blocklist + scenarios fired from Traefik's access log via
  the `crowdsecurity/traefik` collection). Banned IPs get a `403`.

To change countries or switch allow/blocklist, edit `.env` and re-run
`docker compose up -d` (re-renders the dynamic config). Reference: the geofence
tradeoff (edge vs home box) and the passthrough-vs-bridging decision are why
this box terminates TLS at all.

## Dashboard

Published on `127.0.0.1:8080` only. Reach it through an SSH tunnel:

```bash
ssh -L 8080:localhost:8080 your-vps    # then open http://localhost:8080
```

## CrowdSec (Option 2: engine in Docker, bouncer on host)

CrowdSec has an **engine** (decides which IPs are bad — from the community
blocklist, SSH brute-force, and now L7 web-attack scenarios off Traefik's access
log) and **bouncers** that enforce the bans. There are two bouncers here:

- **CrowdSec bouncer plugin (in Traefik, L7):** already wired up as the
  `crowdsec` middleware — checks each HTTP request and returns `403` for banned
  IPs. This is what makes CrowdSec a real web firewall now that the VPS
  terminates TLS. Its key is `CROWDSEC_LAPI_KEY` (created in Setup step 4).
- **Firewall bouncer (on the host, L3/L4):** drops banned IPs at nftables before
  they reach Traefik at all. Only the kernel can drop packets, so this one
  *must* live on the host even though the engine runs in Docker. Install steps
  below.

The two are complementary: the firewall bouncer sheds load at the packet level;
the plugin catches anything that reaches Traefik and gives clean `403`s.

### 1. Start the engine (in this compose stack)

```bash
docker compose up -d crowdsec
docker compose logs -f crowdsec           # confirm collections installed
docker compose exec crowdsec cscli metrics
```

The engine's Local API is published on `127.0.0.1:8081` (8080 is the Traefik
dashboard). It auto-registers with the central API and pulls the community
blocklist; SSH detection reads the host's `/var/log/auth.log`.

### 2. Create an API key for the bouncer

```bash
docker compose exec crowdsec cscli bouncers add firewall-bouncer-host
# copy the key it prints
```

### 3. Install + configure the bouncer (on the VPS host)

```bash
curl -s https://install.crowdsec.net | sudo sh          # add the repo
sudo apt install crowdsec-firewall-bouncer-nftables
```

Edit `/etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml` to match
[`crowdsec/crowdsec-firewall-bouncer.yaml.example`](crowdsec/crowdsec-firewall-bouncer.yaml.example)
— set `api_url: http://127.0.0.1:8081/`, paste the `api_key`, and keep the
`nftables_hooks: [input, forward]` block (the `forward` hook is what lets it
block attacks on Traefik's Docker-published 80/443, not just host-local SSH).

```bash
sudo systemctl restart crowdsec-firewall-bouncer
```

### 4. Verify it's actually wired up

```bash
docker compose exec crowdsec cscli bouncers list         # bouncer shows "valid"
docker compose exec crowdsec cscli decisions list        # bans (grows over time)
sudo nft list table ip crowdsec                          # nftables set exists
# End-to-end test: ban a throwaway IP and confirm the firewall gets it.
docker compose exec crowdsec cscli decisions add --ip 203.0.113.10 --duration 1m
sudo nft list set ip crowdsec crowdsec-blacklists        # 203.0.113.10 appears
```

> **journald-only hosts:** if `/var/log/auth.log` doesn't exist (no rsyslog),
> SSH detection won't fire. Either install rsyslog, or switch `crowdsec/acquis.yaml`
> to a `journalctl` source and mount the journal into the container.

## Troubleshooting

- **Can't reach the tailnet from the container** — if `100.x` addresses aren't
  reachable from the default bridge network, switch to host networking in
  `docker-compose.yml` (remove the `ports:` block, add `network_mode: host`).
- **502 / connection refused** — confirm the home server is listening on `:443`
  for that Tailscale IP and that Tailscale ACLs let the VPS reach it.
- **Traefik starts with no routes** — check the render step:
  `docker compose logs config` and confirm `traefik-data/dynamic/poly.yml` exists.
