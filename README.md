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

## Troubleshooting

- **Can't reach the tailnet from the container** — if `100.x` addresses aren't
  reachable from the default bridge network, switch to host networking in
  `docker-compose.yml` (remove the `ports:` block, add `network_mode: host`).
- **502 / connection refused** — confirm the home server is listening on `:443`
  for that Tailscale IP and that Tailscale ACLs let the VPS reach it.
- **Traefik starts with no routes** — check the render step:
  `docker compose logs config` and confirm `traefik-data/dynamic/poly.yml` exists.
