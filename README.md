# Renovate Deployment

Umbrella Helm chart that packages and configures [Renovate](https://github.com/renovatebot/renovate) together with
a [Valkey](https://valkey.io/) cache for the `local`, `sf-k8s01-dev`, `sf-k8s02-dev`, and `sf-k8s01-prod`
environments.

> [!IMPORTANT]
> Never install the content of this repository on our clusters manually. Deployment is fully managed by ArgoCD.

## Overview

- Runs Renovate as a Kubernetes `CronJob` every hour at minute 13, backed by a `valkey` cache for faster repeat runs.
- Renders Renovate's onboarding config into a `ConfigMap` that is mounted into the job.
- Fetches the Gitea credentials used by Renovate through an `ExternalSecret`, scoped per environment.
- Exposes Valkey metrics to the cluster Prometheus through a `redis_exporter` sidecar and a `ServiceMonitor`,
  see [Monitoring](#monitoring).
- Ships Helm unittest coverage for every rendered resource, see [Testing](#testing).
- Hydrates the manifests of every environment into pull requests for ArgoCD, see
  [Continuous Integration](#continuous-integration).

## Repository Structure

| File / Directory | Purpose |
| --- | --- |
| `Chart.yaml` | Declares the `renovate` and `valkey` chart dependencies for this umbrella chart. |
| `Chart.lock` | Pins the resolved dependency versions, so local and CI builds render the same manifests. |
| `values-subchart-overrides.yaml` | Overrides for the `renovate` and `valkey` chart dependencies. |
| `values-local.yaml`, `values-development.yaml`, `values-production.yaml` | Per-environment value overrides. |
| `helm-config.yaml` | Maps each cluster environment to its `valueFiles` and required Kubernetes `apis`. |
| `tests/` | Helm unittest suites, one per rendered resource, with git-ignored snapshots. |
| `renovate.json` | This repository's own Renovate configuration, for keeping its dependencies up to date. |

> [!NOTE]
> `values-subchart-overrides.yaml` is kept separate from the environment value files so that dependency-compatible
> settings, such as the image registry and repository used by the `valkey` chart, can be unit tested on their own.
> Helm does not allow disabling `values.yaml` merging, so this split is the only way to isolate dependency defaults
> from per-environment overrides.

## Environments

`helm-config.yaml` is consumed by the pipeline hydration step to render manifests for each cluster.

| Environment | Value Files | Notable Overrides |
| --- | --- | --- |
| `local` | `values-subchart-overrides.yaml`, `values-local.yaml` | Zero CPU and memory requests, standalone Valkey on a `100Mi` `nfs1` PVC, 1-minute cronjob schedule, `LOG_LEVEL=debug`. |
| `sf-k8s01-dev`, `sf-k8s02-dev` | `values-subchart-overrides.yaml`, `values-development.yaml` | Base resource limits, replicated Valkey (1 primary + 3 replicas, one `100Mi` `nfs1` volume per pod). |
| `sf-k8s01-prod` | `values-subchart-overrides.yaml`, `values-production.yaml` | Increased Renovate resource limits, 51-minute `activeDeadlineSeconds`, `128Mi` Valkey memory limit. |

> [!NOTE]
> The Valkey chart has no separate resources for primary and replicas. Valkey `resources` apply to every pod of the
> StatefulSet, so the replicated environments request them four times.

## Prerequisites

Commands below assume the `SteadOps-Steadies-K8s-Workplace` workbench, which ships `helm`, `yq`, `kubectl`,
`hetzner-k3s`, and `act` pre-installed. Dependency, testing, and rendering commands also have a containerized
alternative for running outside the workbench.

## Fetch Chart Dependencies

Download the subchart archives pinned in `Chart.lock` into the git-ignored `charts/` folder before testing or
rendering. `helm dependency build` resolves repositories by name, so every repository from `Chart.yaml` is added
first, as the hydration workflow does.

In the workbench:

```sh
 yq -N -r 'explode(.) | .dependencies[] | [.name, .repository] | @tsv' Chart.yaml |
   while read -r name url; do
     helm repo add "$name" "$url" \
       --force-update
   done
 helm dependency build .
```

Without the workbench:

```sh
 docker run \
   -e HOME=/tmp \
   --entrypoint sh \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm -c '
     helm repo add renovate https://docs.renovatebot.com/helm-charts/ --force-update
     helm repo add valkey https://valkey.io/valkey-helm/ --force-update
     helm dependency build .
   '
```

> [!TIP]
> Renovate keeps `Chart.yaml` and `Chart.lock` in sync. When changing a dependency by hand, run
> `helm dependency update .` instead and commit the regenerated `Chart.lock`: hydration fails when the two disagree.

## Testing

### Run Helm Unittests

After [fetching the chart dependencies](#fetch-chart-dependencies):

```sh
 docker run \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   helmunittest/helm-unittest \
   .
```

> [!TIP]
> Add `-o test-output.xml` after `helmunittest/helm-unittest` to also produce a JUnit report.

### Render All Manifests Locally

After [fetching the chart dependencies](#fetch-chart-dependencies), in the workbench:

```sh
 for cluster in $(yq '.environments | keys[]' helm-config.yaml); do
   helm template \
     -a "$(cluster=$cluster yq '.environments.[env(cluster)].apis | @csv' helm-config.yaml)" \
     -f "$(cluster=$cluster yq '.environments.[env(cluster)].valueFiles | @csv' helm-config.yaml)" \
     --include-crds \
     -n "$(yq 'explode(.) | .namespace // ""' helm-config.yaml)" \
     --output-dir "_local/$cluster" \
     --release-name "$(yq 'explode(.) | .releaseName // ""' helm-config.yaml)" \
     --skip-tests \
     .
 done
```

Rendered manifests are written to `_local/<cluster>/`, which is git-ignored.

### Render All Manifests Locally Without the Workbench

```sh
 docker run \
   -e HOME=/tmp \
   --entrypoint sh \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   ghcr.io/steadforce/steadops/workbenches/k8s:main -c '
     for cluster in $(yq ".environments | keys[]" helm-config.yaml); do
       helm template \
         -a "$(cluster=$cluster yq ".environments.[env(cluster)].apis | @csv" helm-config.yaml)" \
         -f "$(cluster=$cluster yq ".environments.[env(cluster)].valueFiles | @csv" helm-config.yaml)" \
         --include-crds \
         -n "$(yq "explode(.) | .namespace // \"\"" helm-config.yaml)" \
         --output-dir "_local/$cluster" \
         --release-name "$(yq "explode(.) | .releaseName // \"\"" helm-config.yaml)" \
         --skip-tests \
         .
     done
   '
```

## Monitoring

With `valkey.metrics.enabled`, every Valkey pod runs a `redis_exporter` sidecar on port `9121`, exposed through the
`renovate-valkey-metrics` Service. The `renovate-valkey` `ServiceMonitor` carries the label
`prometheus: cluster-monitoring`, which the cluster Prometheus uses to discover scrape targets.

To check the exporter from the workbench, first forward the metrics port (this blocks the terminal):

```sh
 kubectl get servicemonitor renovate-valkey \
   -n renovate
 kubectl port-forward svc/renovate-valkey-metrics 9121:9121 \
   -n renovate
```

Then, in a second terminal:

```sh
 curl -s http://localhost:9121/metrics | grep '^redis_memory_used_bytes'
```

> [!TIP]
> `redis_memory_used_bytes` against the container memory limit is the quickest way to tell whether Valkey is close to
> being OOM killed.

## Run GitHub Workflows Locally

In the workbench, from the folder containing this `README.md`:

```sh
 act
```

On first execution you are asked which flavour of the `act` image to use. The default `medium` is a good starting
point.

`act` reads workflow secrets, such as `STEADOPS_HELM_RENOVATION_MS_TEAMS_WEBHOOK`, from a `.secrets` file in
`KEY=value` format. The file is git-ignored.

> [!NOTE]
> The hydration workflow skips opening pull requests under `act`, so a local run only renders the manifests.

## Continuous Integration

- `helm-unittest.yaml` runs the Helm unittest suite on every push and reports results to Microsoft Teams.
- `helm-hydration.yaml` renders manifests for every environment in `helm-config.yaml` on pushes to `main` and on
  demand. It opens one pull request per environment against the `environments/<env>` branch watched by ArgoCD.
- `trufflehog.yaml` scans the repository for leaked secrets on pushes and pull requests to `main`, and on demand.

All three call reusable workflows from
[`steadforce/steadops-workflows`](https://github.com/steadforce/steadops-workflows).

### Hydrate a Branch Manually

Open **Actions > Helm hydration > Run workflow**, pick the branch under **Use workflow from**, and start the run.

> [!WARNING]
> A manual run opens real pull requests against the `environments/<env>` branches. The pull request branch is named
> after the environment and chart version only, so a branch with the same chart version as `main` updates the same
> pull request and replaces its manifests.

## Resources

- [Renovate documentation](https://docs.renovatebot.com/)
- [Renovate Helm chart](https://github.com/renovatebot/helm-charts/tree/main/charts/renovate)
- [Valkey Helm chart](https://github.com/valkey-io/valkey-helm)
- [redis_exporter](https://github.com/oliver006/redis_exporter)
- [Helm chart dependencies](https://helm.sh/docs/topics/charts/#chart-dependencies)
