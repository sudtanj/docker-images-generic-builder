# mini-router

> Part of a multi-image repo - see the [root README](../README.md) for how
> the generic per-folder build workflow works. Everything below is scoped
> to this folder; run these commands from inside it
> (`cd mini-router` first if you're at the repo root).

[mini-router](https://github.com/sudtanj/mini-router) is one
OpenAI-compatible **and** Anthropic-compatible endpoint in front of every
LLM provider you already pay for.

Point any OpenAI client *or* any Anthropic client at it, ask for a model,
and it picks a provider, translates the dialect if it has to, and falls
through to the next provider the moment anything goes wrong.

It is built to sit on a small always-on box on your own network - an Orange
Pi Zero 3 is the design target - in a ~2 MB binary that idles under 5 MB of
RAM and stays there while streaming.

## Quick start

Two compose files, depending on whether you want to build from source or
just run the published image:

- **`docker-compose.yml`** - builds locally from the `Dockerfile` here
  (which fetches mini-router's source itself).
- **`docker-compose.hub.yml`** - pulls
  [`sudtanj/mini-router`](https://hub.docker.com/r/sudtanj/mini-router).

```bash
cp .env.example .env
$EDITOR .env                 # at least one provider key + MINI_ROUTER_API_KEYS
docker compose -f docker-compose.hub.yml up -d
```

Then point your clients at it:

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8080/v1", api_key="<MINI_ROUTER_API_KEYS>")
client.chat.completions.create(model="fast", messages=[{"role": "user", "content": "hi"}])

from anthropic import Anthropic
client = Anthropic(base_url="http://localhost:8080", api_key="<MINI_ROUTER_API_KEYS>")
client.messages.create(model="fast", max_tokens=256,
                       messages=[{"role": "user", "content": "hi"}])
```

Both of those can land on the *same* provider. `fast` resolves to whichever
member is healthy, and the answer is reshaped to match whichever SDK asked.

## Configuration

There is no config file. Every setting is an environment variable, which is
why this image mounts nothing: the `environment:` block in the compose files
*is* the configuration. The full list:

```bash
docker compose run --rm mini-router --help
```

And to see what your settings actually resolved to, before starting
anything for real:

```bash
docker compose run --rm mini-router --check
```

The three that matter most:

| Variable | What it does |
|---|---|
| `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GROQ_API_KEY`, ... | A provider key on its own registers that provider, with the right URL and dialect. Ten are recognised; see `.env.example`. |
| `MINI_ROUTER_POOL_<NAME>` | `provider:model,provider:model` - one client-facing name over several provider models, tried in that order. |
| `MINI_ROUTER_API_KEYS` | What *your* clients present. The compose files turn auth on, so set this. |

Anything not on the recognised list is spelled out with
`MINI_ROUTER_PROVIDER_<NAME>_URL` and friends. A variable mini-router does
not recognise is a startup error naming the variable, not a silent default -
so a typo costs you a failed start, not a week of confusion.

## What you get

- **Pools with spillover on anything.** Not just rate limits: a refused
  connection, a timeout, an expired key, a 400, a model the provider has
  never heard of - all of them move the request to the next member. When the
  list runs out the client gets the last provider's own error, status and
  message intact, reshaped into its own dialect.
- **Protocol translation both ways**, including streaming, tool calls and
  images. An OpenAI-shaped request can be served by Anthropic and come back
  OpenAI-shaped, and the reverse.
- **`Retry-After` is honoured**, so a rate-limited provider is parked for
  exactly as long as it asked rather than being hammered.
- **Prometheus metrics** at `/metrics` and a JSON view of the provider pool
  at `/admin/upstreams` - both removable with `MINI_ROUTER_METRICS=false` /
  `MINI_ROUTER_ADMIN=false`. There is no web UI anywhere in it.

## Notes on this image

- **Multi-arch, cross-compiled.** `linux/amd64` and `linux/arm64`. The Rust
  build runs on the build platform and cross-compiles, rather than being
  emulated under QEMU - a release build (LTO, one codegen unit) under
  emulation takes tens of minutes, and this keeps the arm64 image roughly as
  cheap as the amd64 one.
- **Pin the source.** `MINI_ROUTER_REF` is a build arg naming the git ref to
  build. Empty (the default) builds mini-router's default branch; set it to a
  tag for a reproducible image.
- **No shell tooling for the healthcheck.** mini-router probes itself with
  `--healthcheck`, so the image carries no `curl` or `wget` for it.
  `/healthz` is unauthenticated on purpose, so it keeps working with
  `MINI_ROUTER_REQUIRE_AUTH=true`.
- **Runs as uid 10001 with a read-only filesystem**, no capabilities, and
  `no-new-privileges`. Nothing is written to disk.
- **`ca-certificates` is the only runtime dependency.** Every provider API is
  https; without the trust store every request fails at the handshake.
