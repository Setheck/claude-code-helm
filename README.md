# Claude Code Helm Chart

[![Helm 3](https://img.shields.io/badge/Helm-3.0+-0f1689?logo=helm&logoColor=white)](https://helm.sh/)
[![Kubernetes 1.19+](https://img.shields.io/badge/Kubernetes-1.19+-326ce5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A Helm chart for running [Claude Code CLI](https://github.com/anthropics/claude-code) in Kubernetes as a long-lived pod with a persistent HOME directory.

> Forked from [Chrisbattarbee/claude-code-helm](https://github.com/Chrisbattarbee/claude-code-helm) — see the upstream repo for the original chart and the [Claude Code on Kubernetes blog post](https://metoro.io/blog/claude-code-kubernetes) that inspired it.

---

## Quick Start

For an in-depth installation walkthrough, see the [Claude Code on Kubernetes blog post](https://metoro.io/blog/claude-code-kubernetes).

```bash
helm repo add claude-code https://setheck.github.io/claude-code-helm
helm repo update
helm install claude claude-code/claude-code
```

Wait for the pod and open a shell:

```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=claude --timeout=120s
kubectl exec -it deploy/claude-claude-code -c claude -- sh
claude
```

---

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- Docker (if you want to build/publish your own image)

> The chart does not install Claude Code at startup. It expects `image.repository:image.tag` to be ready-to-run (defaults to `ghcr.io/setheck/claude-code:2.1.37`).

---

## Persistence Model

By default, the chart mounts a PersistentVolumeClaim at `/home/ubuntu`. This means:

- `~/.claude` (auth/config/logs) persists across pod restarts
- interactive login state survives restarts
- any additional files in `/home/ubuntu` persist

Disable persistence if needed:

```bash
helm install claude claude-code/claude-code \
  --set image.repository=<your-claude-image> \
  --set image.tag=<your-tag> \
  --set persistence.enabled=false
```

---

## Authentication Options

### 1) Use an existing secret

Create a secret with your provider keys:

```bash
kubectl create secret generic claude-credentials \
  --from-literal=ANTHROPIC_API_KEY=sk-ant-xxx
```

Install with:

```bash
helm install claude claude-code/claude-code \
  --set image.repository=<your-claude-image> \
  --set image.tag=<your-tag> \
  --set credentials.existingSecret=claude-credentials
```

### 2) Let the chart create a secret

```bash
helm install claude claude-code/claude-code \
  --set image.repository=<your-claude-image> \
  --set image.tag=<your-tag> \
  --set credentials.anthropicApiKey=sk-ant-xxx
```

### 3) Login interactively inside the pod

`claude` login artifacts are written under `/home/ubuntu/.claude` and persist because HOME is PVC-backed by default.

---

## Image Publishing

This repository includes a multi-arch image workflow at [`.github/workflows/build-image.yaml`](.github/workflows/build-image.yaml).

- Push to `main` publishes:
  - `ghcr.io/setheck/claude-code:latest`
  - `ghcr.io/setheck/claude-code:sha-<shortsha>`
- Push a tag named `claude-X.Y.Z` publishes:
  - `ghcr.io/setheck/claude-code:X.Y.Z`
  - `ghcr.io/setheck/claude-code:latest`
- Images are built for both `linux/amd64` and `linux/arm64`.

Claude Code version baked into the image is controlled by the workflow and installed via the official native installer (`https://claude.ai/install.sh`):

- `main` builds install the `latest` Claude Code release
- `claude-X.Y.Z` tag builds install Claude Code `X.Y.Z`

For reproducibility, Helm defaults should point to explicit version tags rather than `latest`.

---

## Key Values

| Parameter                    | Description                                         | Default        |
| ---------------------------- | --------------------------------------------------- | -------------- |
| `image.repository`           | Prebuilt image containing `claude`                 | `ghcr.io/setheck/claude-code` |
| `image.tag`                  | Image tag                                           | `2.1.37`       |
| `command`                    | Container command (idle by default)                 | `sh -lc sleep infinity` |
| `credentials.existingSecret` | Existing secret for env vars                        | `""`           |
| `credentials.anthropicApiKey`| API key for chart-managed secret                    | `""`           |
| `credentials.secretData`     | Extra chart-managed secret key/value pairs          | `{}`           |
| `persistence.enabled`        | Persist `/home/ubuntu`                                | `true`         |
| `persistence.size`           | PVC size                                            | `5Gi`          |
| `persistence.existingClaim`  | Use existing PVC instead of creating one            | `""`           |

See [`charts/claude-code/values.yaml`](charts/claude-code/values.yaml) for the full configuration.

---

## Uninstall

```bash
helm uninstall claude
```

PVC is not deleted automatically. Delete it manually if you want to remove persisted HOME data:

```bash
kubectl delete pvc claude-claude-code
```
