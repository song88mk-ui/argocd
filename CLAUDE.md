# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

GitOps config for the song88 EKS clusters: two Helm charts plus per-environment values files.
There is no build, no CI, and no test suite in this repo. A merge to `main` deploys: every
ArgoCD `Application` tracks `HEAD` with `automated: {prune: true, selfHeal: true}`.

## Where the Applications are defined

Not here. They are generated in the sibling `infrastructure` repo:

- Template: `../infrastructure/helmfile/charts/argocd-apps/templates/applications.yaml`
- App lists: `../infrastructure/helmfile/common/*-apps.yaml.gotmpl` and
  `../infrastructure/helmfile/environments/{dev,stg,prd}/*-apps.yaml.gotmpl`

Adding a new service needs a change in both repos: an entry in `infrastructure`, plus values
files here. Values files alone deploy nothing.

## Values layering

For a standard app, the Application passes these files, in order (later wins):

1. `apps/<appsDir>/<config>/<env>.yaml`: hand-maintained config
2. `argocd-image-updater/<appsDir>/<config>/<env>.yaml`: `image.repository` and `image.tag` only
3. `apps/<appsDir>/<config>/<extra>`: optional `valueFiles` from the app entry

`<appsDir>` is `platform`, `oddsengine` or `corp-apps`. `<config>` defaults to the app name.

- **`ignoreMissingValueFiles: true` is set.** A misnamed file or directory is skipped silently.
  The app then runs on chart defaults: no secrets, no env, and image tag `0.1.0`.
- **Sharded apps** share one base file plus a suffix file. In `apps/oddsengine/sportsbook-service`,
  `<env>.yaml` is layered with `<env>-00.yaml` … `<env>-10.yaml`. Each shard sets its own
  `fullnameOverride`, SA name, queue and routing key (`r.oe.adapter.#.NN`).
  `apps/oddsengine/uof-client` works the same way, with `dev-g01`, `stg-g02` and `prd-g00`.
- **bet-genius apps break the pattern.** Their values live at top-level `apps/bg-*/<env>.yaml`,
  not under a project dir. They use `valueFilesOverride` in `infrastructure`. Image-updater
  files are at `argocd-image-updater/bet-genius-apps/bet-genius-apps-<name>/` for stg/prd and
  `argocd-image-updater/dev/<name>/` for dev.

Render locally exactly as ArgoCD does; helm 3.5.4 is installed:

```
helm template t charts/platform-app \
  -f apps/oddsengine/sportsbook-service/stg.yaml \
  -f argocd-image-updater/oddsengine/sportsbook-service/stg.yaml \
  -f apps/oddsengine/sportsbook-service/stg-00.yaml
```

## Image updates

`argocd-image-updater/**` is written by ArgoCD Image Updater. Its commits are titled
`build: automatic update of <app>` and make up nearly all of `git log`. Do not hand-edit
those files; the next push overwrites them. The `image.tag` in `apps/**` (often `latest`) is
a placeholder that the updater file overrides.

- A tag is picked when it matches the env regex `.*-dev`, `.*-stg` or `.*-prd`, using the `newest-build` strategy.
- ECR: dev and stg share `172065883315.dkr.ecr.ap-south-1`; prd uses `525375333516.dkr.ecr.ap-east-1`.
- Image names are `<appPrefix>-<config>`, e.g. `platform-platform-api` or `oev2-sportsbook-service`, unless
  `appImageOverride` is set.

## Charts

`charts/platform-app` (most services) and `charts/betgenius-app` (the `bg-*` apps) are near-copies.
They diverge, so check which chart an app uses before editing a template:

- Only `platform-app` has `httproute.yaml`, `service.create`, `service.headless`, and
  `ingress.hosts[].thirdPartyPaths`.
- Ingress `pathType` is `Prefix` in `platform-app` and `Exact` in `betgenius-app`.
- `platform-app` probes `/swagger` by default. `betgenius-app` has no probes.

A chart change redeploys every app on that chart in every environment at once.

Both charts:

- `env:` is a flat map. Keys use `A__B` for .NET config key `A:B`, and every value is quoted.
- `secrets:` renders one `ExternalSecret` per entry. The default store is `ClusterSecretStore`
  `parameterstore`, i.e. SSM paths like `/stg/apps/...`. Each `data[]` item becomes an env var
  via `envVariable`.
- The ingress always 404s `/metrics` and `/healthz` through an ALB fixed-response action.
- Default ports: app on `3000`, metrics on `8080`. A `ServiceMonitor` is rendered when `metrics.enabled`.

## Namespaces

| Env | Namespace / ArgoCD project |
|---|---|
| dev | everything in `dev`, project `dev` |
| stg, prd | `platform`, `oddsengine`, `corp-apps`, `bet-genius-apps` |

Many prd and stg apps have no `dev.yaml`. Check before assuming an app runs in dev.

## Editing config

- Compare `dev.yaml`, `stg.yaml` and `prd.yaml` (and any shard files) side by side. They drift.
  A missing or misplaced key falls back to the service's `appsettings.json` default, usually
  `localhost` (DEVOPS-389).
- In-cluster service URLs use the Service name and port 3000, e.g. `http://betting-api:3000`.
  Some names carry a prefix, e.g. `platform-webprops-api`.
- Credentials never go in `env:`. Add a `secrets` entry that points at an SSM path.
- Commit style: `[TICKET-123] Imperative summary`, e.g. `[DEVOPS-370] Wire missing secret env vars
  for oddsengine services`. CODEOWNERS is `@FlyingCockatoo/devops`.
