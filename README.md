# homelab-cluster-apps

ArgoCD App of Apps repository for my homelab Kubernetes cluster. This repo is the single source of truth for all cluster applications — each app lives in its own repository, and this repo contains only the ArgoCD `Application` manifests that point to them.

## Structure

```
homelab-cluster-apps/
├── apps/                          # ArgoCD Application manifests
│   ├── diode.yaml
│   ├── home-media-server.yaml
│   ├── homepage.yaml
│   ├── myspeed.yaml
│   ├── netbox.yaml
│   ├── ollama.yaml
│   ├── open-webui.yaml
│   ├── orb-agent.yaml
│   └── patchmon.yaml
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

| App | Source | Values Repo | Namespace |
|-----|--------|-------------|-----------|
| diode | [diode](https://netboxlabs.github.io/diode/charts) | [homelab-cluster-apps-values](https://github.com/BryanR77/homelab-cluster-apps-values) | `diode` |
| home-media-server | [home-media-server](https://github.com/BryanR77/home-media-server) | [home-media-server-values](https://github.com/BryanR77/home-media-server-values) | `home-media-server` |
| homepage | [jameswynn/helm-charts](http://jameswynn.github.io/helm-charts/) | [homelab-cluster-apps-values](https://github.com/BryanR77/homelab-cluster-apps-values) | `homepage` |
| myspeed | [MySpeed](https://github.com/gnmyt/MySpeed) (raw manifests, no chart) | [homelab-cluster-apps-values](https://github.com/BryanR77/homelab-cluster-apps-values) | `myspeed` |
| netbox | [netbox-chart](https://charts.netbox.oss.netboxlabs.com/) | [homelab-cluster-apps-values](https://github.com/BryanR77/homelab-cluster-apps-values) | `netbox` |
| ollama | [ollama-helm](https://otwld.github.io/ollama-helm/) | [ollama-values](https://github.com/BryanR77/ollama-values) | `ollama` |
| open-webui | [open-webui](https://helm.openwebui.com/) | [ollama-values](https://github.com/BryanR77/ollama-values) | `open-webui` |
| orb-agent | [orb-agent](https://github.com/netboxlabs/orb-agent) (raw manifests, no chart) | [homelab-cluster-apps-values](https://github.com/BryanR77/homelab-cluster-apps-values) | `orb-agent` |
| patchmon | [patchmon-helm](https://github.com/BryanR77/patchmon-helm) (fork of [HellstromIT/patchmon-helm](https://github.com/HellstromIT/patchmon-helm)) | [homelab-cluster-apps-values](https://github.com/BryanR77/homelab-cluster-apps-values) | `patchmon` |

Current versions are pinned via `targetRevision` in each `apps/*.yaml` manifest, kept up to date by Renovate — see [Chart Version Management](#chart-version-management).

## Networking

Services are exposed via **Cilium Gateway API** (`homelab-gateway`, namespace: `default`).

| App | Hostname |
|-----|----------|
| homepage | `homepage.homelab.rawlinsnet.net` |
| myspeed | `myspeed.homelab.rawlinsnet.net` |
| open-webui | `ollama.homelab.rawlinsnet.net` |
| patchmon | `patchmon.homelab.rawlinsnet.net`, `patchmon.rawlinsnet.net` (public, via Cloudflare Tunnel) |

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

For NetBox specifically, plugin installation and `plugins`/`pluginsConfig` values live in the external NetBox values repo. The [netbox-proxbox](https://github.com/emersonfelipesp/netbox-proxbox) plugin is enabled there: NetBox runs a custom image (built from [docker/netbox-proxbox/Dockerfile](docker/netbox-proxbox/Dockerfile) via [.github/workflows/netbox-proxbox-image.yml](.github/workflows/netbox-proxbox-image.yml)) with the plugin pip-installed, and its `proxbox-api` backend is deployed alongside NetBox via `netbox/manifests/proxbox-api.yaml` in the values repo. Connect it to Proxmox from the NetBox UI under Plugins > Proxbox.

The same custom image also bakes in the [netbox-diode-plugin](https://github.com/netboxlabs/diode-netbox-plugin), which talks to a [Diode](https://github.com/netboxlabs/diode) server deployed separately as its own app (`apps/diode.yaml`, values in `diode/values.yaml` of the values repo). Diode is internal-only — nothing outside the cluster can reach it — but it still needs the chart's bundled ingress-nginx (pinned to `ClusterIP`, not the chart default `LoadBalancer`) as an internal reverse proxy: the plugin derives its Diode Auth REST URL from `diode_target` (grpc scheme swapped for http, `/auth` appended), which only resolves correctly through that combined ingress routing to `diode-auth`/`diode-ingester`/`diode-reconciler`. Pointing `diode_target` straight at any one backend service does not work. That config (`diode_target`, `netbox_to_diode_client_secret`), plus all of Diode's own secrets (bundled Postgres/Redis/Hydra passwords and OAuth2 client credentials), are created imperatively with `kubectl create secret` rather than committed to git — see the chart's [step-by-step install docs](https://github.com/netboxlabs/diode/tree/develop/charts/diode#step-by-step-installation) for the exact secrets required in the `diode` and `netbox` namespaces.

Actual network discovery (finding devices, not just ingesting pushed data) is a separate component: [orb-agent](https://github.com/netboxlabs/orb-agent) (`apps/orb-agent.yaml`, raw manifests in `orb-agent/manifests/` of the values repo, no Helm chart upstream). It runs an SNMPv2c discovery policy against the homelab LAN and pushes results into Diode the same way the NetBox UI's "Add Client Credential" feature does — via its own dedicated Diode client credential (scope `diode:ingest`), created once through Plugins > Diode > Client Credentials in NetBox. It runs with `hostNetwork: true` since it needs real reachability to the LAN subnet being scanned, which a normal cluster-networked pod wouldn't have. The Diode credentials and SNMP community string live in an imperatively-created `orb-agent-secrets` Secret (not committed) — see the comments in `deployment.yaml` for the exact command.

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
