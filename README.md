# claude-in-a-box

A self-hosted, batteries-included stack that puts Claude Code behind a familiar ChatGPT-style web UI. It bundles two upstream projects as git submodules: an OpenAI-compatible shim in front of the Claude Agent SDK, and [Open WebUI](https://github.com/open-webui/open-webui) as the frontend.

> **Status:** early scaffolding. The submodules are wired up; top-level orchestration (Docker Compose, `.env` templates, startup scripts) is still being assembled. See [Roadmap](#roadmap).

## Why

- **One endpoint, any client.** Talk to Claude Code from anything that speaks the OpenAI API — Open WebUI, Continue, Cursor, curl, your own scripts.
- **Keep your auth local.** Reuse an existing `claude auth` CLI login, an `ANTHROPIC_API_KEY`, AWS Bedrock, or GCP Vertex AI credentials — whatever you already have.
- **Own your chat history.** Open WebUI stores conversations, RAG corpora, and user accounts on your own box instead of someone else's.
- **Use Claude Code's tools through chat.** Optionally expose the agent's Read/Write/Bash tools to the chat session when you want an agentic loop instead of plain completions.

## Architecture

```
┌───────────────────┐    OpenAI-style HTTP    ┌──────────────────────────────┐    Claude Agent SDK    ┌──────────────┐
│    Open WebUI     │ ──────────────────────▶ │ claude-code-openai-wrapper   │ ─────────────────────▶ │  Anthropic   │
│  (Svelte + API)   │   /v1/chat/completions  │      (FastAPI, Python)       │   (or Bedrock/Vertex)  │   / Claude   │
└───────────────────┘                         └──────────────────────────────┘                        └──────────────┘
        ▲                                                     ▲
        │ browser                                             │ optional: Claude Code tools
        │                                                     │ (Read, Write, Bash, …)
     you :)                                                  sandbox / workspace
```

Open WebUI points at the wrapper as if it were OpenAI. The wrapper translates OpenAI chat completions into Claude Agent SDK calls, handles auth, streams tokens back, and surfaces real token/cost accounting from the SDK.

## Components

| Path | Upstream | Role |
| --- | --- | --- |
| [`open-webui/`](open-webui/) | [open-webui/open-webui](https://github.com/open-webui/open-webui) | ChatGPT-style web frontend + backend (accounts, history, RAG, tools). |
| [`claude-code-openai-wrapper/`](claude-code-openai-wrapper/) | [brandonros/claude-code-openai-wrapper](https://github.com/brandonros/claude-code-openai-wrapper) (fork of [RichardAtCT/...](https://github.com/RichardAtCT/claude-code-openai-wrapper)) | FastAPI server exposing `/v1/chat/completions` and `/v1/messages` on top of the Claude Agent SDK. The fork owns the container-image publish workflow. |

Both are tracked as git submodules — see [`.gitmodules`](.gitmodules).

## Quick start

Clone with submodules:

```bash
git clone --recurse-submodules git@github.com:brandonros/claude-in-a-box.git
cd claude-in-a-box
```

Or, if you already cloned without `--recurse-submodules`:

```bash
git submodule update --init --recursive
```

### Kubernetes (recommended)

Two Helm charts live under [`deploy/helm/`](deploy/helm/), each wrapping [bjw-s `app-template`](https://bjw-s-labs.github.io/helm-charts/docs/app-template/) as a pinned subchart:

```bash
helm dependency update deploy/helm/claude-code-openai-wrapper
helm dependency update deploy/helm/open-webui

kubectl create namespace claude-in-a-box
kubectl -n claude-in-a-box create secret generic claude-code-openai-wrapper-secrets \
  --from-literal=ANTHROPIC_API_KEY='sk-ant-...'

helm upgrade --install claude-code-openai-wrapper deploy/helm/claude-code-openai-wrapper -n claude-in-a-box
helm upgrade --install open-webui                  deploy/helm/open-webui                  -n claude-in-a-box
```

See [`deploy/helm/README.md`](deploy/helm/README.md) for ingress, mounting `CLAUDE.md` / skills / `.mcp.json` from ConfigMaps, and upgrade notes. The wrapper image is built and published to `ghcr.io/brandonros/claude-code-openai-wrapper` by the [`docker.yml`](https://github.com/brandonros/claude-code-openai-wrapper/blob/main/.github/workflows/docker.yml) workflow in the [wrapper fork](https://github.com/brandonros/claude-code-openai-wrapper). This repo only contains orchestration (Helm, docs); if you want local patches to the wrapper, commit them in the fork and bump the submodule pointer here.

### Local Docker

- **Wrapper (port 8000):** follow [`claude-code-openai-wrapper/README.md`](claude-code-openai-wrapper/README.md). The fastest path is `docker compose up` inside that directory, or `poetry install && poetry run uvicorn src.main:app`.
- **Open WebUI:** follow [`open-webui/README.md`](open-webui/README.md). `docker compose up` inside that directory is the usual path.
- **Point the UI at the wrapper:** in Open WebUI → *Settings → Connections → OpenAI API*, set the base URL to `http://host.docker.internal:8000/v1` (or wherever the wrapper is reachable) and any non-empty API key.

Pulling upstream updates later:

```bash
git submodule update --remote --merge
```

## Configuration

The meaningful knobs live in the two submodules; they read environment variables at startup. The ones you'll usually care about:

| Variable | Consumed by | Purpose |
| --- | --- | --- |
| `ANTHROPIC_API_KEY` | wrapper | Direct API key auth (skip if using `claude auth` CLI login). |
| `CLAUDE_AUTH_METHOD` | wrapper | Force a specific auth path (`api_key`, `cli`, `bedrock`, `vertex`). |
| `AWS_*` / `GOOGLE_APPLICATION_CREDENTIALS` | wrapper | Bedrock / Vertex AI auth. |
| `OPENAI_API_BASE_URLS` / `OPENAI_API_KEYS` | Open WebUI | Semicolon-separated lists; point the UI at the wrapper. |
| `WEBUI_AUTH` | Open WebUI | Disable signups / lock down the instance. |

See each submodule's README for the full list.

## Roadmap

- [x] Helm charts for both services under [`deploy/helm/`](deploy/helm/) (bjw-s `app-template` based).
- [x] Wrapper container image published to GHCR by the `docker.yml` workflow in the [wrapper fork](https://github.com/brandonros/claude-code-openai-wrapper).
- [ ] Top-level `docker-compose.yml` that brings up both services on one network.
- [ ] `.env.example` covering the common auth paths.
- [ ] Preconfigured Open WebUI connection so the wrapper is wired in on first boot.
- [ ] Optional profile for exposing Claude Code's tools (Read/Write/Bash) into chat sessions.
- [ ] Optional LiteLLM chart for when multiple backends / virtual keys / budgets are needed.

## Acknowledgments

Standing on the shoulders of:

- [Open WebUI](https://github.com/open-webui/open-webui) — the chat frontend.
- [claude-code-openai-wrapper](https://github.com/RichardAtCT/claude-code-openai-wrapper) — the OpenAI-compatible shim.
- [Anthropic's Claude Agent SDK](https://github.com/anthropics/claude-agent-sdk-python).

## License

This repository is a thin wrapper; each submodule retains its own upstream license. Check `open-webui/LICENSE` and `claude-code-openai-wrapper/` for terms before redistributing.
