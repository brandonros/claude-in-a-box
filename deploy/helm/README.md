# Helm deployment

Two umbrella charts, each wrapping [bjw-s `app-template` v4.6.2](https://bjw-s-labs.github.io/helm-charts/docs/app-template/) as a pinned subchart:

- [`open-webui/`](open-webui/) — the chat frontend.
- [`claude-code-openai-wrapper/`](claude-code-openai-wrapper/) — Claude Code behind an OpenAI-compatible API.

## Prerequisites

- Kubernetes 1.27+, Helm 3.13+, a `StorageClass` supporting `ReadWriteOnce`.
- The wrapper image lives at `ghcr.io/brandonros/claude-code-openai-wrapper`, published by [`docker.yml`](https://github.com/brandonros/claude-code-openai-wrapper/blob/main/.github/workflows/docker.yml) in the fork. If you use a different fork, override `app-template.controllers.claude-code-openai-wrapper.containers.app.image.repository`.

## Install

```bash
helm dependency update open-webui
helm dependency update claude-code-openai-wrapper

kubectl create namespace claude-in-a-box

kubectl -n claude-in-a-box create secret generic claude-code-openai-wrapper-secrets \
  --from-literal=ANTHROPIC_API_KEY='sk-or-v1-...'   # OpenRouter key by default

helm upgrade --install claude-code-openai-wrapper ./claude-code-openai-wrapper -n claude-in-a-box
helm upgrade --install open-webui                  ./open-webui                 -n claude-in-a-box
```

Open WebUI's values default to `http://claude-code-openai-wrapper:8000/v1`, which matches the Service the wrapper chart creates in the same namespace. Changing the wrapper release name means overriding `OPENAI_API_BASE_URLS` in `open-webui/values.yaml`.

The wrapper chart defaults `ANTHROPIC_BASE_URL` to `https://openrouter.ai/api`, so the `ANTHROPIC_API_KEY` above should be your OpenRouter key (`sk-or-v1-...`). To go direct to Anthropic instead, override that env var to `https://api.anthropic.com` and put an Anthropic key in the secret.

Pin a specific wrapper image with `--set 'app-template.controllers.claude-code-openai-wrapper.containers.app.image.tag=sha-<abbrev>'` — tags are produced by `docker/metadata-action` in the publish workflow.

## Ingress

Off by default. Flip `ingress.app.enabled: true` under `app-template:` in [`open-webui/values.yaml`](open-webui/values.yaml) (or a values-override file) and fill in `hosts` / `tls`. Pattern is the [app-template ingress schema](https://bjw-s-labs.github.io/helm-charts/docs/app-template/).

## Skills, CLAUDE.md, .mcp.json

[`claude-code-openai-wrapper/values.yaml`](claude-code-openai-wrapper/values.yaml) has three disabled `persistence.*` entries pre-wired to ConfigMaps. Create them, then flip `enabled: true`:

```bash
kubectl -n claude-in-a-box create configmap claude-md        --from-file=CLAUDE.md=./path/to/CLAUDE.md
kubectl -n claude-in-a-box create configmap claude-skills    --from-file=./path/to/skills/
kubectl -n claude-in-a-box create configmap claude-mcp-config --from-file=.mcp.json=./path/to/.mcp.json
```

## Updating the model list

Open WebUI's model dropdown reflects what the wrapper's `/v1/models` endpoint returns, which is a **curated list** in the fork — not proxied from OpenRouter/Anthropic.

**Fast path** (no fork edits): set `CLAUDE_MODELS_OVERRIDE` in the wrapper's env to a comma-separated slug list and restart the pod. There's a commented-out example in [values.yaml](claude-code-openai-wrapper/values.yaml).

**Permanent path** (updates the default for everyone): edit `DEFAULT_CLAUDE_MODELS` in [`src/constants.py`](https://github.com/brandonros/claude-code-openai-wrapper/blob/main/src/constants.py) in the fork, push to `main`, let [`docker.yml`](https://github.com/brandonros/claude-code-openai-wrapper/blob/main/.github/workflows/docker.yml) rebuild, then:

```bash
kubectl -n claude-in-a-box rollout restart deploy/claude-code-openai-wrapper
```

First entry in the list becomes Open WebUI's default, so order it best-first.

## Uninstall

```bash
helm -n claude-in-a-box uninstall open-webui claude-code-openai-wrapper
kubectl -n claude-in-a-box delete pvc -l app.kubernetes.io/instance=open-webui   # chat history
```
