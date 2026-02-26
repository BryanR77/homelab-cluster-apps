# homelab-cluster-apps

ArgoCD App of Apps repository for my homelab Kubernetes cluster. This repo is the single source of truth for all cluster applications — each app lives in its own repository, and this repo contains only the ArgoCD `Application` manifests that point to them.

## Structure

```
homelab-cluster-apps/
├── apps/                          # ArgoCD Application manifests
│   └── home-media-server.yaml
└── bootstrap/
    └── root-app.yaml              # One-time bootstrap manifest
```

## Bootstrap

Apply the root Application once to hand management of this repo over to ArgoCD:

```bash
kubectl apply -f bootstrap/root-app.yaml
```

ArgoCD will then watch the `apps/` directory and automatically sync any Application manifests found there.

## Adding a New App

1. Create a new manifest in `apps/<app-name>.yaml`
2. Commit and push — ArgoCD will pick it up automatically

For apps with a separate private values repo, use the [multi-source](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/) pattern (requires ArgoCD ≥ v2.6):

```yaml
sources:
  - repoURL: https://github.com/BryanR77/<app-repo>.git
    targetRevision: <branch>
    helm:
      valueFiles:
        - $values/values.yaml
  - repoURL: https://github.com/BryanR77/<app-values-repo>.git
    targetRevision: HEAD
    ref: values
```

> Private repos must be registered in ArgoCD credentials before use:
> ```bash
> argocd repo add https://github.com/BryanR77/<private-repo>.git \
>   --username <user> --password <token>
> ```

## Apps

| App | Chart Repo | Branch | Values Repo | Namespace |
|-----|-----------|--------|-------------|-----------|
| home-media-server | [home-media-server](https://github.com/BryanR77/home-media-server) | `generic-k8s` | [home-media-server-values](https://github.com/BryanR77/home-media-server-values) | `home-media-server` |
