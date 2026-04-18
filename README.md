# claude-in-a-box

Self-hosted Claude Code behind a ChatGPT-style web UI. Two upstream projects wired together as git submodules: a [brandonros fork of claude-code-openai-wrapper](https://github.com/brandonros/claude-code-openai-wrapper) that exposes Claude Code over the OpenAI API, and [Open WebUI](https://github.com/open-webui/open-webui) as the frontend.

## Architecture

```
┌───────────────────┐    OpenAI-style HTTP    ┌──────────────────────────────┐    Claude Agent SDK    ┌──────────────┐
│    Open WebUI     │ ──────────────────────▶ │ claude-code-openai-wrapper   │ ─────────────────────▶ │  Anthropic   │
│  (Svelte + API)   │   /v1/chat/completions  │      (FastAPI, Python)       │   (or Bedrock/Vertex)  │   / Claude   │
└───────────────────┘                         └──────────────────────────────┘                        └──────────────┘
```

## Components

| Path | Upstream |
| --- | --- |
| [`open-webui/`](open-webui/) | [open-webui/open-webui](https://github.com/open-webui/open-webui) |
| [`claude-code-openai-wrapper/`](claude-code-openai-wrapper/) | [brandonros/claude-code-openai-wrapper](https://github.com/brandonros/claude-code-openai-wrapper) (fork) — the fork owns the container-image publish workflow. |

## Quick start (Kubernetes)

```bash
git clone --recurse-submodules git@github.com:brandonros/claude-in-a-box.git
cd claude-in-a-box

helm dependency update deploy/helm/claude-code-openai-wrapper
helm dependency update deploy/helm/open-webui

kubectl create namespace claude-in-a-box
kubectl -n claude-in-a-box create secret generic claude-code-openai-wrapper-secrets \
  --from-literal=ANTHROPIC_API_KEY='sk-ant-...'

helm upgrade --install claude-code-openai-wrapper deploy/helm/claude-code-openai-wrapper -n claude-in-a-box
helm upgrade --install open-webui                  deploy/helm/open-webui                  -n claude-in-a-box
```

The wrapper image is built and published to `ghcr.io/brandonros/claude-code-openai-wrapper` by [`docker.yml`](https://github.com/brandonros/claude-code-openai-wrapper/blob/main/.github/workflows/docker.yml) in the fork. See [`deploy/helm/README.md`](deploy/helm/README.md) for ingress, `CLAUDE.md`/skills/`.mcp.json` mounting, and upgrades.

## Quick start (local Docker)

Run each submodule's `docker compose up`, then in Open WebUI → *Settings → Connections → OpenAI API*, set base URL `http://host.docker.internal:8000/v1` and any non-empty key.

## Roadmap

- [x] Helm charts for both services ([`deploy/helm/`](deploy/helm/)).
- [x] Wrapper image published to GHCR by the [fork's `docker.yml`](https://github.com/brandonros/claude-code-openai-wrapper/blob/main/.github/workflows/docker.yml).
- [ ] Fix the fork's Dockerfile (drop `--reload`, multi-stage, smaller image) so the Helm `command:` override goes away.
- [ ] Optional profile for exposing Claude Code's tools (Read/Write/Bash) into chat sessions.

## Acknowledgments

- [Open WebUI](https://github.com/open-webui/open-webui)
- [RichardAtCT/claude-code-openai-wrapper](https://github.com/RichardAtCT/claude-code-openai-wrapper) — the original wrapper this project forks.
- [Anthropic's Claude Agent SDK](https://github.com/anthropics/claude-agent-sdk-python).

Each submodule keeps its own upstream license; see `open-webui/LICENSE` and the fork.
