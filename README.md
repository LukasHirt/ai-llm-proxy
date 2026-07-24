# ai-llm-proxy Docker Image

[![Docker Pulls](https://img.shields.io/docker/pulls/owncloud/ai-llm-proxy.svg)](https://hub.docker.com/r/owncloud/ai-llm-proxy)
[![License: Apache-2.0](https://img.shields.io/github/license/owncloud-docker/ai-llm-proxy)](https://github.com/owncloud-docker/ai-llm-proxy/blob/main/LICENSE)

Docker image for [`ai-llm-proxy`](https://github.com/owncloud/web-extensions/tree/main/packages/ai-llm-proxy),
the server-side proxy shipped alongside ownCloud Web's AI extensions. It validates
an oCIS bearer token against the oCIS OIDC `userinfo` endpoint, sanitizes the
client's chat-completion request, and forwards it to a configured LLM endpoint.
This repo builds and publishes the image; the application source lives in
[owncloud/web-extensions](https://github.com/owncloud/web-extensions).

This repository is built the same way as
[`owncloud-docker/ocis`](https://github.com/owncloud-docker/ocis): a multi-stage
`Dockerfile.multiarch` that compiles the application from source, built and
published via GitHub Actions using ownCloud's reusable
[`owncloud-docker/ubuntu`](https://github.com/owncloud-docker/ubuntu) workflows
(multi-arch buildx, Trivy scanning, a smoke test, and a Docker Hub push).

## Quick Start

```bash
docker run --rm \
  -p 3030:3030 \
  -e OCIS_URL=https://your-ocis-instance.example.com \
  -e LLM_ENDPOINT=https://your-llm-endpoint.example.com/v1 \
  -e LLM_API_KEY=changeme \
  owncloud/ai-llm-proxy:latest
```

The proxy expects `POST /v1/chat/completions` requests carrying an
`Authorization` header with a valid oCIS bearer token, and an `Origin` header
matching `OCIS_URL` (browsers set this automatically for same/cross-origin
requests; non-browser callers may omit it).

## Supported Tags

`owncloud/ai-llm-proxy` has no upstream release tags yet, so this image currently
tracks the unreleased `main` branch of `web-extensions` directly (a rolling
build), rather than a pinned-version matrix.

| Tag | Meaning |
|-----|---------|
| `latest` | Most recent build of `packages/ai-llm-proxy` from `main` |
| `YYYYMMDD` | Immutable build for a specific day (e.g. `20260602`) |
| `sha-<short>` | Build of a specific `web-extensions` `main` commit (e.g. `sha-a1b2c3d`) |

> Once `ai-llm-proxy` starts cutting semver releases, this repo's
> `.github/workflows/main.yml` should be extended with a version matrix (see
> [owncloud-docker/ocis](https://github.com/owncloud-docker/ocis)'s `main.yml`
> for the pattern) to publish pinned, immutable version tags.

Built weekly (Sundays at 00:00 UTC), and on every push to this repository's `main` branch.

```bash
docker pull owncloud/ai-llm-proxy:latest
```

## Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `PORT` | `3030` | Port the proxy listens on |
| `OCIS_URL` | *(required)* | Base URL of the oCIS instance; also used to derive the allowed CORS/request origin |
| `LLM_ENDPOINT` | *(required)* | Base URL of an OpenAI-compatible chat-completions endpoint |
| `LLM_API_KEY` | `""` | Bearer token forwarded to `LLM_ENDPOINT`, if required |
| `LLM_MODEL` | `""` | When set, overrides whatever model the client requests |
| `LLM_MAX_TOKENS` | `4096` | Hard ceiling on `max_tokens` forwarded to the LLM |
| `MAX_BODY_BYTES` | `6291456` | Maximum request body size the proxy buffers (bytes) |
| `RATE_LIMIT_RPM` | `20` | Max requests per authenticated user per rolling minute |

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| `3030` | TCP | HTTP API |

The service is stateless — it holds no data on disk, so no volumes are defined.

## Build Arguments

| ARG | Default | Purpose |
|-----|---------|---------|
| `GIT_REF` | `main` | `web-extensions` git ref (branch or tag) to clone and build |
| `GIT_SHA` | `""` | Optional exact commit to check out after cloning `GIT_REF`. Pins the build to a resolved commit and busts the clone-layer cache when it changes, keeping rolling builds fresh. |
| `REVISION` | `""` | Git SHA embedded in the OCI labels |
| `TARGETARCH` | set by buildx | Target architecture (`amd64`, `arm64`) |

## Building

The image is built entirely from source via a two-stage `Dockerfile.multiarch`:

**`build`** — sparse, blobless clone of `owncloud/web-extensions` (only
`packages/ai-llm-proxy` is fetched — the package has no workspace dependencies),
then `pnpm install --ignore-scripts && pnpm build` (a plain `tsc` compile) to
produce `dist/index.js`.

**Runtime** — minimal Alpine + Node image with the compiled `dist/` copied in,
running as a non-root user. The stage runs `apk upgrade` to refresh all
installed OS packages to the latest available Alpine patch releases at build
time, so security fixes are picked up immediately rather than waiting for a
base-image tag bump.

```bash
docker buildx build \
  --build-arg REVISION=$(git rev-parse HEAD) \
  --platform linux/amd64 \
  -f Dockerfile.multiarch -t ai-llm-proxy:test .
```

## CI

The GitHub Actions workflow (`.github/workflows/main.yml`) builds and validates
the image on every push, pull request, weekly schedule, and manual dispatch:

1. **Build** — multi-arch image (`linux/amd64`, `linux/arm64`) built with
   BuildKit and GHA layer cache.
2. **Trivy scan** — scans for HIGH/CRITICAL CVEs; accepted exceptions live in
   `.trivyignore`.
3. **Smoke test** — starts the container for real and asserts its actual
   request-handling behaviour: an `OPTIONS` preflight to `/v1/chat/completions`
   must return `204`, and an unauthenticated `POST` to the same path must
   return `401`.
4. **Publish** — pushes to Docker Hub with `latest`, `YYYYMMDD`, and
   `sha-<short>` tags (skipped on pull requests).

## Community & Support

- [ownCloud Website](https://owncloud.com)
- [Community Discussions](https://github.com/orgs/owncloud/discussions)
- [Matrix Chat](https://app.element.io/#/room/#owncloud:matrix.org)
- [Documentation](https://doc.owncloud.com)

Bug reports and feature requests for `ai-llm-proxy` itself belong upstream in
[`owncloud/web-extensions`](https://github.com/owncloud/web-extensions) — this
repository tracks only the Docker packaging.

## Security

**Do not open a public GitHub issue for security vulnerabilities.**

Report vulnerabilities at **<https://security.owncloud.com>** — see [SECURITY.md](SECURITY.md).

## License

Apache-2.0 — see [LICENSE](LICENSE).
