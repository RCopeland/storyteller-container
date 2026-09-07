# Storyteller on the homelab

Self-hosted [Storyteller](https://storyteller-platform.dev/) — a platform for
creating and reading ebooks with synced narration — reachable **only over the
tailnet** using the same tailscale-sidecar pattern as
`~/Dev/bb-app-container`, `~/Dev/foundry-vtt-container`, and `~/Dev/n8n`. No
funnel, no published host ports — the web UI is available only at
`https://storyteller.<your-tailnet>.ts.net` while you're on the tailnet.

## What Storyteller is

- A self-hosted platform for taking audiobooks and ebooks you already own and
  automatically **synchronizing** them (forced alignment of narration to text).
- Includes a REST API server, a web interface for managing your library, and
  mobile apps for reading/listening.
- Published as an official Docker image
  `registry.gitlab.com/storyteller-platform/storyteller:latest` — no local
  build needed.

## How it works

- A **tailscale sidecar** joins the tailnet as host `storyteller`, provisions an
  HTTPS cert, and runs `tailscale serve` from `ts-serve.json`. No
  `AllowFunnel` → tailnet-only.
- **`web`** shares the sidecar's namespace (`network_mode: service:tailscale`)
  and binds `127.0.0.1:8001`. It is not published on the host, so nothing on
  the LAN/internet can reach it.
- `ts-serve.json` proxies `443` → `127.0.0.1:8001` for
  `${TS_CERT_DOMAIN}` (web UI + API).

## Run

Fill `.env` (`TAILSCALE_AUTH_KEY`, `TS_CERT_DOMAIN`) if you haven't, then:

```bash
docker compose up -d
docker compose logs -f web
```

Verify over the tailnet:

```bash
tailscale status | grep storyteller
curl -sI https://storyteller.<your-tailnet>.ts.net
```

## First-time setup

- Scale/config lives at `https://storyteller.<your-tailnet>.ts.net`. Create your
  admin account there, then configure the library and (optionally) OAuth
  providers.

## Configuration

Storyteller is configured through the settings UI and/or a JSON config file
(`STORYTELLER_CONFIG`). Notable env vars (see
[self-hosting docs](https://storyteller-platform.dev/docs/installation/self-hosting/)):

| Variable | Purpose | Default |
|----------|---------|---------|
| `STORYTELLER_SECRET_KEY_FILE` | Path to the secret key file | — |
| `PUID` / `PGID` | UID/GID to run as | `1000` / `1000` |
| `ENABLE_WEB_READER` | Enable experimental web reader | `false` |
| `READIUM_PORT` | Port for the Readium server | `8002` |
| `TZ` | Timezone | `America/New_York` |

### Books library (optional)

By default data (database, covers, transcriptions, uploaded books) persists in
the named `storyteller_data` volume at `/data`. To auto-import from an existing
books folder, uncomment this in `compose.yaml`:

```yaml
- ~/Documents/Books:/library:rs
```

## Security notes

- Storyteller requires a secret key (`STORYTELLER_SECRET_KEY.txt`) used to forge
  auth tokens — keep it private (the file is git-ignored).
- The instance is intentionally **not** exposed to the LAN/internet; it's safe
  behind the tailnet. Do **not** add published ports or funnel access.
- Do **not** set `user:` / `--user` on the Storyteller container — it owns the
  UID/GID setup and can break `/data` permissions if launched as a non-root
  user.

## Files

| File              | Purpose                                   |
|-------------------|-------------------------------------------|
| `compose.yaml`    | tailscale sidecar + Storyteller web       |
| `ts-serve.json`   | `tailscale serve` config (tailnet-only)   |
| `.env`/`.env.example` | secrets/config                        |
| `STORYTELLER_SECRET_KEY.txt` | instance auth secret (git-ignored) |