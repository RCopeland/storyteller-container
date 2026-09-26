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

### Books library (auto-import)

By default data (database, covers, transcriptions, uploaded books) persists in
the named volume `storyteller_data` at `/data`. To auto-import books, Storyteller
watches the `storyteller_library` volume mounted at `/library`. It's created and
seeded once on the homelab, then declared `external` in compose so it can't be
accidentally removed by `docker compose down -v`.

**Dropping books in from other containers** — mount the same volume (rw) in the
source container and copy files into it:

```bash
docker run --rm \
  -v storyteller-container_storyteller_library:/library:rw \
  -v /path/from/your/container:/src:ro \
  alpine sh -c 'cp -r /src/. /library/ && chown -R 1000:1000 /library'
```

The volume must stay owned by UID/GID `1000` so the Storyteller process can
read/ingest/move files. On a normal sync, books dropped into `/library` are
ingested automatically (Storyteller scans on start and on a daily cron).

## Security notes

- Storyteller requires a secret key (`STORYTELLER_SECRET_KEY.txt`) used to forge
  auth tokens — keep it private (the file is git-ignored).
- The instance is intentionally **not** exposed to the LAN/internet; it's safe
  behind the tailnet. Do **not** add published ports or funnel access.
- Do **not** set `user:` / `--user` on the Storyteller container — it owns the
  UID/GID setup and can break `/data` permissions if launched as a non-root
  user.

### Secret scanning

Secrets are scanned **before** they reach GitHub, not after. `.gitignore` only
covers `.env` and `STORYTELLER_SECRET_KEY.txt`; the scanner catches a key that
leaks some other way (pasted into the README, a new file, `ts-serve.json`).

One-time setup per clone:

```bash
pip install pre-commit   # or: pipx install pre-commit
pre-commit install
```

Run every hook against the whole tree at any time:

```bash
pre-commit run --all-files
```

Hooks are gitleaks ([`gitleaks.toml`](gitleaks.toml)) plus private-key and
large-file guards. `gitleaks.toml` extends the built-in ruleset with a
**Tailscale `tskey-…` rule** — gitleaks has no Tailscale rule by default, and a
Tailscale auth key can join a node to your tailnet, so it's the most sensitive
credential in this stack.

`git commit --no-verify` skips the hooks. CI is the backstop for that: the
`secrets` job scans full history on every push and PR.

## CI

[`.github/workflows/ci.yml`](.github/workflows/ci.yml) runs two jobs:

| Job | What it does |
|-----|--------------|
| `secrets` | gitleaks full-history scan (`fetch-depth: 0`) — backstop for `--no-verify` |
| `validate` | `docker compose config`, `ts-serve.json` JSON validity, and a check that `.env.example` documents every variable `compose.yaml` references |

It is read-only on purpose: there is no build, no deploy, and no
`docker compose up`. The `storyteller_library` volume is `external: true` and
only exists on the homelab, so an `up` in CI would fail.

### Making the checks required for PRs

After the workflow has run at least once, require both jobs in
**Settings → Branches** (or **Rulesets**) for `master`: enable *Require status
checks to pass before merging* and select `secrets` and `validate`.

Two caveats: required checks only gate **pull requests** — pushing directly to
`master` bypasses them, so add a rule blocking direct pushes if you want them
enforced on your own work too. And GitHub only offers a check in the picker
once it has run at least once.

## Files

| File              | Purpose                                   |
|-------------------|-------------------------------------------|
| `compose.yaml`    | tailscale sidecar + Storyteller web       |
| `ts-serve.json`   | `tailscale serve` config (tailnet-only)   |
| `.env`/`.env.example` | secrets/config                        |
| `STORYTELLER_SECRET_KEY.txt` | instance auth secret (git-ignored) |
| `.pre-commit-config.yaml` | gitleaks + hygiene hooks           |
| `gitleaks.toml`   | gitleaks rules (adds Tailscale keys)      |
| `.github/workflows/ci.yml` | `secrets` + `validate` CI         |
