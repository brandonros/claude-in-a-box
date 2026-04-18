# Helm deployment

Two umbrella charts, each wrapping [bjw-s `app-template` v4.6.2](https://bjw-s-labs.github.io/helm-charts/docs/app-template/) as a subchart so values stay in-repo and the chart version is pinned:

- [`open-webui/`](open-webui/) — the chat frontend.
- [`claude-code-openai-wrapper/`](claude-code-openai-wrapper/) — the OpenAI-compatible shim over the Claude Agent SDK.

They're deliberately decoupled: install either on its own, point the UI at any OpenAI-compatible endpoint you want, or swap the wrapper out for [LiteLLM](https://github.com/BerriAI/litellm) later without touching the UI chart.

## Prerequisites

- Kubernetes 1.27+
- Helm 3.13+
- A `StorageClass` that supports `ReadWriteOnce` (Open WebUI uses a PVC for SQLite)
- Access to the wrapper image at `ghcr.io/brandonros/claude-in-a-box/claude-code-openai-wrapper` — published by [`publish-wrapper-image.yml`](../../.github/workflows/publish-wrapper-image.yml) on every push to `master`. If your fork is elsewhere, override `app-template.controllers.claude-code-openai-wrapper.containers.app.image.repository` in a values file.

> **Note:** the wrapper chart overrides the upstream Dockerfile's `CMD` to drop `--reload` (upstream ships a dev-mode CMD). If you override `image`, you may want to keep the `command:` override in [`claude-code-openai-wrapper/values.yaml`](claude-code-openai-wrapper/values.yaml) as-is.

## First-time install

```bash
# Fetch the app-template subchart for both umbrella charts.
helm dependency update deploy/helm/open-webui
helm dependency update deploy/helm/claude-code-openai-wrapper

# Create the namespace and the wrapper's secrets.
kubectl create namespace claude-in-a-box

kubectl -n claude-in-a-box create secret generic claude-code-openai-wrapper-secrets \
  --from-literal=ANTHROPIC_API_KEY='sk-ant-...' \
  --from-literal=API_KEY="$(openssl rand -hex 32)"   # wrapper's own auth

# Install the wrapper first so Open WebUI has something to talk to.
# Pin to a specific submodule SHA with e.g. `--set ...image.tag=wrapper-abc1234`
# instead of the floating `latest` default; tags are produced by the publish
# workflow.
helm upgrade --install claude-code-openai-wrapper deploy/helm/claude-code-openai-wrapper \
  -n claude-in-a-box

helm upgrade --install open-webui deploy/helm/open-webui \
  -n claude-in-a-box
```

The default Open WebUI values already point at `http://claude-code-openai-wrapper:8000/v1`, which matches the default Service name of the wrapper chart when both are released with the names shown above. If you change the wrapper release name, override `OPENAI_API_BASE_URLS` in `open-webui/values.yaml`.

## Exposing the UI

Ingress is off by default. Turn it on with a values override file:

```yaml
# openwebui-ingress.values.yaml
app-template:
  ingress:
    app:
      enabled: true
      className: nginx
      hosts:
        - host: openwebui.internal.corp
          paths:
            - path: /
              pathType: Prefix
              service:
                identifier: app
                port: http
      tls:
        - hosts: [openwebui.internal.corp]
          secretName: openwebui-tls
```

```bash
helm upgrade --install open-webui deploy/helm/open-webui \
  -n claude-in-a-box \
  -f deploy/helm/open-webui/values.yaml \
  -f openwebui-ingress.values.yaml
```

## Mounting skills, CLAUDE.md, .mcp.json

The wrapper values file has three disabled `persistence.*` entries (`claude-md`, `skills`, `mcp-config`) wired to ConfigMaps. Create the ConfigMaps from your config repo and flip `enabled: true`:

```bash
kubectl -n claude-in-a-box create configmap claude-md \
  --from-file=CLAUDE.md=./path/to/CLAUDE.md

kubectl -n claude-in-a-box create configmap claude-skills \
  --from-file=./path/to/skills/

kubectl -n claude-in-a-box create configmap claude-mcp-config \
  --from-file=.mcp.json=./path/to/.mcp.json
```

## Upgrading the app-template subchart

```bash
# Bump the version in Chart.yaml, then:
helm dependency update deploy/helm/open-webui
helm dependency update deploy/helm/claude-code-openai-wrapper
```

Check the [app-template changelog](https://github.com/bjw-s-labs/helm-charts/releases?q=app-template) for breaking schema changes before bumping a major.

## Uninstall

```bash
helm -n claude-in-a-box uninstall open-webui claude-code-openai-wrapper
# Open WebUI's PVC is retained by default — delete manually if you want the chat history gone.
kubectl -n claude-in-a-box delete pvc -l app.kubernetes.io/instance=open-webui
```
