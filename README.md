# gitops-argocd

A self-managing Argo CD install, GitOps'd end to end: Argo CD deploys and upgrades itself
from this repo, deploys Prometheus to monitor itself (dashboards + alerts included), and
runs with a pinned Helm 3.7.2 binary in its repo-server instead of whatever ships in the
base image.

## Repo layout

```
bootstrap/
  app-of-apps.yaml          # the one thing you kubectl apply by hand. sync-wave: -10
apps/
  argocd/
    application.yaml        # Argo CD managing itself, via the argo-helm chart
    values.yaml              # ServiceMonitors on + Helm 3.7.2 override for repo-server
  prometheus/
    application.yaml        # kube-prometheus-stack + repo-local extras, multi-source
    values.yaml              # ServiceMonitor/PrometheusRule discovery tuned wide-open
    extras/
      argocd-alerts-prometheusrule.yaml
      argocd-dashboard-configmap.yaml
      kustomization.yaml
custom-image/                # alternative, build-time route to the same Helm 3.7 pin
```

## How the pieces fit together (app-of-apps)

`bootstrap/app-of-apps.yaml` is a single Argo CD `Application` whose source is the `apps/`
directory in this repo, with `directory.recurse: true`. Argo CD renders every manifest it
finds under `apps/` -- which in this repo means exactly two more `Application` objects
(`argocd` and `prometheus`). Those, in turn, deploy the actual workloads. That's the whole
app-of-apps pattern: a parent Application whose "resources" are child Applications.

Bootstrap sequence:

```bash
# 1. One-time manual install of Argo CD itself, just enough to exist and take over.
kubectl create namespace argocd
helm install argocd argo/argo-cd -n argocd --version 7.7.11

# 2. Hand the reins to git: apply the root Application.
kubectl apply -f bootstrap/app-of-apps.yaml
```

From that point on:
- Argo CD reconciles `app-of-apps`, which creates the `argocd` Application.
- The `argocd` Application's source is the *same* `argo-cd` Helm chart, with values from
  this repo -- so Argo CD immediately starts managing (and, on future commits, upgrading)
  the exact install you just did by hand in step 1. Any drift or manual `kubectl edit` on
  the Argo CD deployment itself gets self-healed back to what's in git.
- The `prometheus` Application deploys kube-prometheus-stack into `monitoring`, configured
  to auto-discover the ServiceMonitors the `argocd` chart creates for its own components.

## Requirement-by-requirement

**1. Argo CD manages its own configuration/lifecycle**
`apps/argocd/application.yaml` is an Argo CD `Application` that installs Argo CD via the
upstream chart, sourced from this git repo. Once bootstrapped, every future change to
Argo CD's config (RBAC, resource limits, the Helm binary pin, a version bump) is a git
commit, not a `helm upgrade` run by a human. `syncPolicy.automated.selfHeal: true` means
manual `kubectl` changes against the argocd namespace get reverted back to git state.

**2. Prometheus deployed and configured to monitor Argo CD**
- `apps/prometheus/values.yaml` sets `serviceMonitorSelector: {}` /
  `serviceMonitorNamespaceSelector: {}` so Prometheus Operator picks up ServiceMonitors
  cluster-wide, specifically the ones the `argo-cd` chart creates when
  `<component>.metrics.serviceMonitor.enabled: true` (set for controller, server,
  repo-server, redis, applicationSet, and notifications in `apps/argocd/values.yaml`).
- `apps/prometheus/extras/argocd-dashboard-configmap.yaml` is a ConfigMap labeled
  `grafana_dashboard: "1"`; Grafana's sidecar (enabled in `apps/prometheus/values.yaml`)
  watches all namespaces for that label and auto-imports it -- no manual dashboard import.
  Panels cover app sync/health status breakdowns, per-component `up`, reconciliation queue
  depth, and repo-server gRPC latency/error rate.
- `apps/prometheus/extras/argocd-alerts-prometheusrule.yaml` defines alerts for
  out-of-sync apps, unhealthy apps, failed syncs, component downtime, a backed-up
  reconciliation queue, and elevated repo-server error rate.

**3. Replace the bundled Helm binary with 3.7**
`apps/argocd/values.yaml` adds an `initContainer` to the `repo-server` pod that downloads
`helm-v3.7.2-linux-amd64` and writes it into an `emptyDir` volume, then mounts that single
file with `subPath` directly over `/usr/local/bin/helm` in the main container -- the exact
path the stock image already uses. Argo CD's tool-invocation code doesn't need to change;
it just gets 3.7.2 when it shells out to `helm`. This is Argo CD's documented "custom
tooling" pattern (init container + volume overlay), applied here to *pin* a specific Helm
minor version rather than add a new tool. `custom-image/` has a build-time alternative
(bake the binary into the image) for teams that prefer not to have an internet-fetching
initContainer at pod start.

**4. Sync wave -10 on the app-of-apps**
`bootstrap/app-of-apps.yaml` carries `argocd.argoproj.io/sync-wave: "-10"`. Waves are
evaluated at the resource level within whatever an Application manages; giving the root
Application a very negative wave keeps it -- and by extension the children it defines --
ordered ahead of anything else that might later be added at the default wave (0) or higher
under the same parent. The child Applications additionally carry their own waves (`argocd`
at 0, `prometheus` at 1) so Argo CD's own install finishes reconciling, and its
ServiceMonitors exist, before Prometheus is deployed to scrape them.

## Things worth discussing live

- Why multi-source Applications (`sources:` + `ref: values`) instead of a Helm subchart or
  a plain umbrella chart: it keeps the values files in plain git, diffable and reviewable,
  without vendoring or forking the upstream charts.
- The chicken-and-egg of self-management: step 1 (manual `helm install`) is unavoidable --
  something has to exist before it can adopt itself into GitOps. Everything after that is
  automated.
- `selfHeal: true` is a double-edged sword on the `argocd` Application itself: it protects
  against config drift, but also means a bad commit to `apps/argocd/values.yaml` will be
  applied automatically. In a real environment I'd gate this Application behind a
  `SyncWindows` policy or a manual-approval `Application` for its own upgrades, or at least
  require PR review + `kubectl argo rollouts`-style promotion.
- `ApplyOutOfSyncOnly` / `ServerSideApply` sync options are there because the CRD bundle in
  the argo-cd chart is large enough to blow past the `kubectl.kubernetes.io/last-applied-
  configuration` annotation size limit under client-side apply.
- Replacing the Helm binary via subPath mount vs. custom image: covered in
  `custom-image/README.md` -- runtime injection avoids image maintenance overhead but adds
  an external download dependency at every repo-server pod start; baked-in avoids that at
  the cost of maintaining and rebuilding a custom image on every Argo CD version bump.

## Before you push this

Replace every occurrence of `https://github.com/YOUR-ORG/gitops-argocd.git` (in
`bootstrap/app-of-apps.yaml`, `apps/argocd/application.yaml`, and
`apps/prometheus/application.yaml`) with your actual repo URL once it exists on your git
host, and update `targetRevision` if you're not using `main`.
