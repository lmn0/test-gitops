# test-gitops

A GitOps repository that bootstraps Argo CD and makes Argo CD manage itself, monitor
itself via Prometheus, and run a pinned Helm 3.7.2 binary — all reconciled through a
single root `take-home-test` Application.

## Repo layout

bootstrap/
  take-home-test.yaml           # the one manifest you kubectl apply by hand
apps/
  argocd/
    crds/                       # Argo CD's own CRDs (Application, AppProject)
    configmaps/                 # argocd-cm, argocd-rbac-cm, argocd-cmd-params-cm, etc.
    secrets/                    # argocd-notifications-secret (no data — see note below)
    deployments/                # one manifest per component: server, repo-server,
                                 # applicationset-controller, notifications-controller,
                                 # dex-server, redis
    statefulsets/                # argocd-application-controller
    services/                   # one per component, including metrics Services
    rbac/
      clusterroles/ clusterrolebindings/
      roles/ rolebindings/
      serviceaccounts/
    networkpolicies/            # one per component
  kube-prometheus-stack/
    crds/crds.yaml               # Prometheus Operator CRDs (Prometheus, ServiceMonitor,
                                 # PrometheusRule, Alertmanager, etc.)
    charts/
      grafana/templates/
      kube-state-metrics/templates/
      prometheus-node-exporter/templates/
    templates/
      prometheus-operator/
      prometheus/                # Prometheus CR, rules-1.14/ (recording + alerting rules)
      alertmanager/
      grafana/                  # datasources, dashboards-1.14/ (stock Kubernetes dashboards)
      exporters/                # per-target ServiceMonitors: kube-api-server, kube-etcd,
                                 # kube-scheduler, kube-controller-manager, kube-proxy,
                                 # kubelet, core-dns
      exporters/argocd/
        argocd-svcmon.yaml       # ServiceMonitors for every Argo CD component
        argocd-alert-rules.yaml  # PrometheusRule: Argo CD sync/health/component-down alerts
        argocd-dashboard-cm.yaml # Grafana dashboard ConfigMap for Argo CD

## 1. Getting Started Steps  - install ArgoCD and deploy Prometheus; configure and manage both

`bootstrap/take-home-test.yaml` is a single Argo CD `Application` whose source is the `apps/`
directory in this repo, with `directory.recurse: true`. Once bootstrapped, it's the only
manifest ever applied by hand — every Argo CD component manifest under `apps/argocd/` is
discovered and reconciled automatically, and `syncPolicy.automated.selfHeal: true` reverts
any manual drift on the live cluster back to what's in git.

```bash
kubectl create namespace argocd
kubectl create namespace monitoring
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for a couple of minutes before bootstrapping the apps:
```bash
kubectl apply -f bootstrap/take-home-test.yaml
```

Grafana will fail to start as the secret is not in the gitops repository. Generate a secret and apply it.
```bash
kubectl create secret generic kube-prometheus-stack-grafana \
  --namespace monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$(openssl rand -base64 24)" \
  --dry-run=client -o yaml > secret.yaml

kubectl apply -f secret.yaml
```

Give it a couple of minutes to make changes to argocd-server to be exposed as type LoadBalancer to get an external IP. View the external IP using the following commands. Give it a few minutes to show up.
```bash
kubectl get svc argocd-server -n argocd -o=jsonpath='{.status.loadBalancer.ingress[0].ip}'
kubectl get svc kube-prometheus-stack-grafana -n monitoring -o=jsonpath='{.status.loadBalancer.ingress[0].ip}'
```


Obtain the 'admin' password for the newly set up ArgoCD and Grafana web consoles:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
kubectl get secret --namespace monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

From that point on, Argo CD's own Deployments/StatefulSet (`argocd-server`,
`argocd-repo-server`, `argocd-application-controller`, `argocd-applicationset-controller`,
`argocd-notifications-controller`, `argocd-dex-server`, `argocd-redis`) are all managed as
resources owned by `take-home-test`, sourced from this repo.

## 2. Prometheus configured to monitor Argo CD

- `apps/kube-prometheus-stack/templates/exporters/argocd/argocd-svcmon.yaml` — one
  `ServiceMonitor` per Argo CD component's metrics Service (`argocd-metrics`,
  `argocd-server-metrics`, `argocd-repo-server` metrics port,
  `argocd-applicationset-controller` metrics port, `argocd-notifications-controller-metrics`,
  and `argocd-dex-server-metrics` if SSO via Dex is in use), each with
  `namespaceSelector.matchNames: [argocd]` so Prometheus (running in `monitoring`) can
  discover Services in a different namespace, and `selector.matchLabels` pinned to each
  Service's `app.kubernetes.io/name` so the correct port gets scraped on Services that
  expose more than one (e.g. repo-server's `server` vs `metrics` ports).
  - **Label alignment matters here and is release-name-dependent.** Each ServiceMonitor
    carries `release: <name>`, and Prometheus's `serviceMonitorSelector.matchLabels`
    only picks up ServiceMonitors whose `release` label matches the Helm release name
    Prometheus itself was installed under (`{{ .Release.Name }}` in the chart's default
    values) — e.g. `release: prometheus-stack` if installed as `helm install prometheus-stack ...`,
    or `release: kube-prometheus-stack` if installed as `helm install kube-prometheus-stack ...`.
    A mismatch here doesn't error — the ServiceMonitor just silently never gets scraped.
    Confirm the label here matches whichever release name is the actual source of truth
    for this cluster before assuming Argo CD monitoring is live.
- `apps/kube-prometheus-stack/templates/exporters/argocd/argocd-alert-rules.yaml` — a
  `PrometheusRule` alerting on out-of-sync apps, unhealthy apps, and component downtime
  (`up == 0` per Argo CD ServiceMonitor job).
- `apps/kube-prometheus-stack/templates/exporters/argocd/argocd-dashboard-cm.yaml` — a
  Grafana dashboard ConfigMap labeled `grafana_dashboard: "1"`, auto-imported by the
  kube-prometheus-stack Grafana sidecar with no manual dashboard import step.

## 3. Helm binary replaced with 3.7.2

`argocd-repo-server` is the only Argo CD component that actually invokes Helm (chart
cloning, `helm template`), so that's where the override is load-bearing; the same pattern
is also applied to the other components' manifests for fleet-wide consistency. Each
patched Deployment adds:

- An `initContainer` (`download-helm-3-7-2`, `alpine:3.20`) that downloads
  `helm-v3.7.2-linux-amd64` into a shared `emptyDir` volume, using Alpine's built-in
  `wget`/`tar` — no `apk add` needed, which matters because `apk add` requires root and
  the pod runs `runAsNonRoot: true`.
- A `volumeMounts` entry on the main container mounting just that one file via `subPath`
  onto `/usr/local/bin/helm` — the exact path the stock binary already occupies, so no
  other config changes are needed.
- `runAsUser: 1000` on the init container and `fsGroup: 1000` at the pod level, so the
  non-root init container can both run and write into the shared volume.

Verify:
```bash
kubectl exec -n argocd deploy/argocd-repo-server -- helm version --short
# v3.7.2+g...
```
