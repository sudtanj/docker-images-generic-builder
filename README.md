# Docker image monorepo

A generic, per-folder Docker image builder. Every top-level folder in this
repo that contains a `Dockerfile` is an independent image: its own Docker
Hub repository, its own version tag sequence, built and published only when
that folder actually changes.

## How it works

[`.github/workflows/docker-publish.yml`](.github/workflows/docker-publish.yml)
runs on every push to `main`:

1. **`discover`** - diffs the push (`git diff <before> <after>`) to find
   which top-level folders changed, then keeps only the ones that contain a
   `Dockerfile`. A brand-new branch (nothing to diff against) or a manual
   run with no folder specified builds every folder that has one.
2. **`build`** (one job per detected folder, run in parallel) - for each:
   - Bumps that folder's own version tag: a git tag named `<folder>-vX.Y.Z`,
     found by listing existing `<folder>-v*` tags and incrementing the
     patch number (starts at `v0.1.0` if the folder has never been tagged).
     Every folder's tag sequence is independent - `foo-v3.2.1` and
     `bar-v0.4.0` don't affect each other.
   - Builds `./<folder>` for `linux/amd64` and `linux/arm64` (via QEMU -
     simple and uniform across every folder, at the cost of a slower arm64
     build than native arm64 runners would give; fine unless a folder's
     build is unusually heavy).
   - Pushes to Docker Hub as `sudtanj/<folder-name>:latest` and
     `sudtanj/<folder-name>:<the new version>`.

Touching only `claude-code-claudish-happy/` triggers only that image's
build; adding a new `my-image/Dockerfile` and pushing triggers only
`my-image`'s first build (`my-image-v0.1.0`), untouched folders are left
alone.

## Adding a new image

1. Create a new top-level folder with a `Dockerfile` in it - that's the
   only requirement. [`claude-code-claudish-happy/`](claude-code-claudish-happy/)
   is a full worked example (its own `README.md`, `docker-compose.yml` /
   `docker-compose.hub.yml`, `.env.example`, entrypoint scripts, ...), but
   none of that is required by the workflow itself - it only ever looks
   for the `Dockerfile`.
2. Push to `main`. The next run detects the new folder and publishes
   `sudtanj/<folder-name>:v0.1.0` (+ `:latest`) automatically.

## Building manually

Via the Actions tab, or:

```bash
# Build every folder that has a Dockerfile:
gh workflow run docker-publish.yml

# Build just one folder (e.g. to force a rebuild without changing anything):
gh workflow run docker-publish.yml -f folder=claude-code-claudish-happy
```

## One-time setup

1. Add a repo secret (Settings -> Secrets and variables -> Actions):
   - `DOCKER_HUB_KEY` - a Docker Hub access token with Read & Write scope
     (Docker Hub -> Account Settings -> Security -> Personal access
     tokens). Used for every image in this repo; the Docker Hub username
     (`sudtanj`) is hardcoded in the workflow since it isn't sensitive.
2. Give the workflow's default token write access so it can push the
   per-folder version tags: Settings -> Actions -> General -> Workflow
   permissions -> "Read and write permissions".

## Images in this repo

| Folder | Docker Hub | What it is |
|---|---|---|
| [`claude-code-claudish-happy/`](claude-code-claudish-happy/) | [`sudtanj/claude-code-claudish-happy`](https://hub.docker.com/r/sudtanj/claude-code-claudish-happy) | Claude Code + Codex CLI + Happy (mobile/web control), full dev toolchain - see its own [README](claude-code-claudish-happy/README.md) |
| [`paseo-codex/`](paseo-codex/) | [`sudtanj/paseo-codex`](https://hub.docker.com/r/sudtanj/paseo-codex) | [Paseo](https://github.com/getpaseo/paseo) (remote daemon + web UI) + Codex CLI, BYOK via env vars - see its own [README](paseo-codex/README.md) |
| [`oci-freetier-creator/`](oci-freetier-creator/) | [`sudtanj/oci-freetier-creator`](https://hub.docker.com/r/sudtanj/oci-freetier-creator) | Runs [oracle-freetier-instance-creation](https://github.com/mohankumarpaluru/oracle-freetier-instance-creation) unattended, config via env vars - see its own [README](oci-freetier-creator/README.md) |
| [`mini-router/`](mini-router/) | [`sudtanj/mini-router`](https://hub.docker.com/r/sudtanj/mini-router) | [mini-router](https://github.com/sudtanj/mini-router): one OpenAI- and Anthropic-compatible endpoint over every LLM provider you use, with pools, spillover and protocol translation - see its own [README](mini-router/README.md) |
