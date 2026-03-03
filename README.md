# homelab-cluster-apps

ArgoCD App of Apps repository for my homelab Kubernetes cluster. This repo is the single source of truth for all cluster applications — each app lives in its own repository, and this repo contains only the ArgoCD `Application` manifests that point to them.

## Structure

```
homelab-cluster-apps/
├── apps/                          # ArgoCD Application manifests
│   ├── home-media-server.yaml
│   ├── netbox.yaml
│   ├── ollama.yaml
│   └── open-webui.yaml
├── bootstrap/
│   └── root-app.yaml              # One-time bootstrap manifest
└── renovate.json                  # Renovate config for chart version updates
```

## Bootstrap

Apply the root Application once to hand management of this repo over to ArgoCD:

```bash
kubectl apply -f bootstrap/root-app.yaml
```

ArgoCD will then watch the `apps/` directory and automatically sync any Application manifests found there.

## Apps

| App | Source | Version / Branch | Values Repo | Namespace |
|-----|--------|-----------------|-------------|-----------|
| home-media-server | [home-media-server](https://github.com/BryanR77/home-media-server) | `generic-k8s` | [home-media-server-values](https://github.com/BryanR77/home-media-server-values) | `home-media-server` |
| netbox | [netbox-chart](https://charts.netbox.oss.netboxlabs.com/) | `8.0.6` | [netbox-values](https://github.com/BryanR77/netbox-values) | `netbox` |
| ollama | [ollama-helm](https://otwld.github.io/ollama-helm/) | `1.48.0` | [ollama-values](https://github.com/BryanR77/ollama-values) | `ollama` |
| open-webui | [open-webui](https://helm.openwebui.com/) | `12.5.0` | [ollama-values](https://github.com/BryanR77/ollama-values) | `open-webui` |

## Networking

Services are exposed via **Cilium Gateway API** (`homelab-gateway`, namespace: `default`).

| App | Hostname |
|-----|----------|
| open-webui | `ollama.homelab.rawlinsnet.net` |

## Adding a New App

1. Create a new manifest in `apps/<app-name>.yaml`
2. Commit and push — ArgoCD will pick it up automatically

For apps with a separate private values repo, use the [multi-source](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/) pattern (requires ArgoCD ≥ v2.6):

```yaml
sources:
  - repoURL: https://<helm-chart-repo>/
    chart: <chart-name>
    targetRevision: <version>
    helm:
      valueFiles:
        - $values/values.yaml
  - repoURL: https://github.com/BryanR77/<app-values-repo>.git
    targetRevision: HEAD
    ref: values
```

For apps that also need raw manifests deployed alongside the chart (e.g. Gateway API HTTPRoutes), add a third source pointing to a directory in the values repo:

```yaml
  - repoURL: https://github.com/BryanR77/<app-values-repo>.git
    targetRevision: HEAD
    path: <app-name>/manifests
```

> Private repos must be registered in ArgoCD credentials before use:
> ```bash
> argocd repo add https://github.com/BryanR77/<private-repo>.git \
>   --username <user> --password <token>
> ```

## Chart Version Management

[Renovate](https://docs.renovatebot.com/) is configured to automatically open PRs when new Helm chart versions are available. It tracks the `targetRevision` field in all `apps/*.yaml` manifests.
