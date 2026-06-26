# hermes-agent

Custom image built on top of the official [Hermes Agent](https://hub.docker.com/r/nousresearch/hermes-agent) image (`nousresearch/hermes-agent`). It is meant to run in a homelab where the container lives on a **remote Docker host**: the agent and its skills execute **inside the container filesystem**, not on the machine you SSH from or the host's OS. Anything a skill invokes by name (shell commands, subprocesses, `PATH` lookups) must therefore be **installed into this image** (or mounted in deliberately). This repo bakes in the CLIs that bundled skills expect so they work the same on a remote engine as they would on a local one.

## What this image adds

The [`Dockerfile`](Dockerfile) extends the upstream Hermes image and installs tooling in standard locations (`/usr/local/bin`, global npm prefix `/usr/local`):

| Layer | Purpose |
| --- | --- |
| **Go build stage** (`cgr.dev/chainguard/go:latest-dev`) | Static binaries built with `CGO_ENABLED=0` and copied into the final image: `blogwatcher`, `gifgrep`, `gog`, `goplaces`, `spogo` |
| **apt** | Browser runtime libraries/fonts for `agent-browser`, `pipx`, `jq`, `vim-tiny` for `vi`, and **GitHub CLI** (`gh`) from GitHub's official apt repo |
| **pipx** | **`nano-pdf`** — Python CLI installed with `PIPX_BIN_DIR=/usr/local/bin` so the executable is on the default `PATH` (`uv` is already provided by the upstream image) |
| **npm** | **`agent-browser`**, **`mcporter`**, and **`summarize`** installed globally under `/usr/local`; on amd64 the build runs `agent-browser install` so Chrome is preloaded in-container, on arm64 it installs system `chromium` instead (Chrome for Testing has no Linux arm64 build) |

Upstream base tag is parameterized as `HERMES_BASE` (default `nousresearch/hermes-agent:latest`) so you can pin a digest or version when upgrading.

The upstream image already includes `ripgrep`, `uv`, `curl`, `git`, `ffmpeg`, Playwright/Chromium, `docker-cli`, and `openssh-client`. This derived image does not reinstall those.

## Build locally

```bash
docker build -t hermes-agent-custom:local \
  --build-arg HERMES_BASE=nousresearch/hermes-agent:latest \
  .
```

On a remote host, push or load this image there and reference it in your compose/stack so the running gateway uses this build instead of plain upstream.

## Run

```bash
docker run -d \
  --name hermes \
  --restart unless-stopped \
  -v ~/.hermes:/opt/data \
  -p 8642:8642 \
  hermes-agent-custom:local gateway run
```

See the [Hermes Docker documentation](https://hermes-agent.nousresearch.com/docs/user-guide/docker) for setup, upgrades, and optional features such as mounting `/var/run/docker.sock`.

## spogo auth on remote/browser-separated setups

If Chrome runs on a different host, `spogo auth import --browser chrome` cannot read that browser profile directly from inside this container. Also, `spogo auth paste` may fail with `unexpected argument paste` in current builds.

Use a manual cookie file instead:

1. Grab `sp_dc` (required) and `sp_t` (recommended) from `https://open.spotify.com` cookies in your external browser.
2. Write `spogo` config:

```toml
default_profile = "default"

[profile.default]
engine = "connect"
cookie_path = "/opt/data/.config/spogo/cookies/default.json"
```

3. Write cookie JSON at `/opt/data/.config/spogo/cookies/default.json`:

```json
[
  {
    "name": "sp_dc",
    "value": "YOUR_SP_DC",
    "domain": ".spotify.com",
    "path": "/",
    "secure": true,
    "http_only": true
  },
  {
    "name": "sp_t",
    "value": "YOUR_SP_T",
    "domain": ".spotify.com",
    "path": "/",
    "secure": true,
    "http_only": true
  }
]
```

4. Lock down permissions and verify:

```bash
mkdir -p /opt/data/.config/spogo/cookies
chmod 700 /opt/data/.config/spogo /opt/data/.config/spogo/cookies
chmod 600 /opt/data/.config/spogo/cookies/default.json
spogo auth status
```

### Troubleshooting

- `missing sp_t`: add the `sp_t` cookie value from your browser and re-run `spogo auth status`.
- Auth still fails with valid cookies: make sure `domain` is `.spotify.com` and `path` is `/` in `default.json`.
- Worked before, broken now: Spotify session cookies can expire; refresh cookies in your browser and replace values in `default.json`.

## CI

[`.github/workflows/docker-weekly.yml`](.github/workflows/docker-weekly.yml) builds and pushes to GHCR on `main`, on a weekly schedule, and on `workflow_dispatch`, resolving upstream `latest` to a digest for reproducible builds and tagging with the upstream git revision when available.
