# traefik-home-vps

A public-facing Traefik entrypoint (runs on a VPS) that **TLS-passthroughs** all
`*.<your-domain>` traffic to a home server reachable over Tailscale. The domain
and the home server's IP are set in `.env`.

```
client ──TLS──▶ VPS Traefik (:443) ──raw TLS stream──▶ home server (Tailscale, :443)
                 routes on SNI                          terminates TLS + serves apps
```

- TLS is **terminated on the home server**, which holds the certificates (issued
  via a DNS-01 challenge). This VPS never decrypts traffic and needs no certs.
- Traefik routes purely on the **SNI** in the TLS ClientHello, so any
  `*.DOMAIN` hostname is forwarded to the home server, which decides what to do
  with it.
- No ACME, no docker socket — just a thin, low-attack-surface forwarder.

## Setup

1. **Configure** — set your domain and the home server's Tailscale IP:
   ```bash
   cp .env.example .env      # then edit .env:
   #   DOMAIN=poly.my.id
   #   HOME_SERVER=100.x.y.z
   ```
2. **DNS** — point `*.DOMAIN` at this VPS's public IP.
3. **Tailscale** — put this VPS on the same tailnet as the home server
   (`tailscale up`); `HOME_SERVER` is that server's Tailscale IP (`100.x.y.z`).
4. **Run**
   ```bash
   docker compose up -d
   docker compose logs -f traefik
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

Add another router to the template (it can reuse `__DOMAIN__`). To also pass
through the bare apex `DOMAIN` (not just subdomains), widen the rule:

```yaml
rule: 'HostSNIRegexp(`^(.+\.)?__DOMAIN__$`)'
```

## Dashboard

Published on `127.0.0.1:8080` only. Reach it through an SSH tunnel:

```bash
ssh -L 8080:localhost:8080 your-vps    # then open http://localhost:8080
```

## CrowdSec (Option 2: engine in Docker, bouncer on host)

CrowdSec has two halves: the **engine** (decides which IPs are bad — from the
community blocklist and from SSH brute-force on this box) and the **firewall
bouncer** (enforces the bans in the host firewall). Only the kernel can drop
packets, so the bouncer *must* live on the host even though the engine runs in
Docker — that split is the whole shape of Option 2.

Because this VPS does pure TLS passthrough, CrowdSec here is an **edge blocklist
+ SSH protector**, not an L7 web firewall (Traefik never sees decrypted HTTP).

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
