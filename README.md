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

`helm-config.yaml` is consumed by the pipeline hydration step to render manifests for each cluster. Every
environment renders `values-subchart-overrides.yaml` first, then its own value file.

| Environment | Value Files |
| --- | --- |
| `local` | `values-subchart-overrides.yaml`, `values-local.yaml` |
| `sf-k8s01-dev`, `sf-k8s02-dev` | `values-subchart-overrides.yaml`, `values-development.yaml` |
| `sf-k8s01-prod` | `values-subchart-overrides.yaml`, `values-production.yaml` |

Notable overrides per environment:

- `local`: zero CPU and memory requests, standalone Valkey on a `100Mi` `nfs1` PVC, 1-minute cronjob schedule,
  `LOG_LEVEL=debug`.
- `sf-k8s01-dev`, `sf-k8s02-dev`: base resource limits, replicated Valkey (1 primary + 3 replicas, one `100Mi`
  `nfs1` volume per pod).
- `sf-k8s01-prod`: increased Renovate resource limits, 51-minute `activeDeadlineSeconds`, `128Mi` Valkey memory
  limit, and no Valkey RDB snapshots.

> [!NOTE]
> The Valkey chart has no separate resources for primary and replicas. Valkey `resources` apply to every pod of the
> StatefulSet, so the replicated environments request them four times.

## Prerequisites

Commands below assume the `SteadOps-Steadies-K8s-Workplace` workbench, which ships `helm`, `yq`, `kubectl`,
`hetzner-k3s`, and `act` pre-installed. Dependency, testing, and rendering commands also have a containerized
alternative for running outside the workbench.

## Fetch Chart Dependencies

All commands run from the repository root. Download the subchart archives pinned in `Chart.lock` into the
git-ignored `charts/` folder after cloning, and after every pull that changes `Chart.lock`. `helm dependency build`
resolves repositories by name, so every `http(s)` repository from `Chart.yaml` is added first, as the pipeline does.

In the workbench:

```sh
 yq 'explode(.) | .dependencies[] | select(.repository == "http*") | .name + " " + .repository' Chart.yaml |
   while read -r name repo; do helm repo add --force-update "$name" "$repo"; done
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
     yq "explode(.) | .dependencies[] | select(.repository == \"http*\") | .name + \" \" + .repository" Chart.yaml |
       while read -r name repo; do helm repo add --force-update "$name" "$repo"; done &&
     helm dependency build .
   '
```

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
> Add `-t JUnit -o test-output.xml` after `helmunittest/helm-unittest` to also write a JUnit report, as the pipeline
> does. Without `-t`, helm-unittest writes the report in XUnit format.

### Render All Manifests Locally

After [fetching the chart dependencies](#fetch-chart-dependencies), in the workbench:

```sh
 for cluster in $(yq 'explode(.) | .environments | keys[]' helm-config.yaml); do
   helm template \
     -a "$(cluster=$cluster yq 'explode(.) | .environments.[env(cluster)].apis // [] | @csv' helm-config.yaml)" \
     -f "$(cluster=$cluster yq 'explode(.) | .environments.[env(cluster)].valueFiles // [] | @csv' helm-config.yaml)" \
     --include-crds \
     -n "$(yq 'explode(.) | .namespace' helm-config.yaml)" \
     --output-dir "_local/$cluster" \
     --release-name \
     --skip-tests \
     "$(yq 'explode(.) | .releaseName' helm-config.yaml)" \
     .
 done
```

The release name is passed positionally; `--release-name` is a boolean flag that, as in the hydration pipeline, adds
the release name to the output path. Rendered manifests are written to `_local/<cluster>/`, which is git-ignored.

### Render All Manifests Locally Without the Workbench

`alpine/helm` ships `yq`, so the same loop runs in one container:

```sh
 docker run \
   -e HOME=/tmp \
   --entrypoint sh \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm -c '
     for cluster in $(yq "explode(.) | .environments | keys[]" helm-config.yaml); do
       helm template \
         -a "$(cluster=$cluster yq "explode(.) | .environments.[env(cluster)].apis // [] | @csv" helm-config.yaml)" \
         -f "$(cluster=$cluster yq "explode(.) | .environments.[env(cluster)].valueFiles // [] | @csv" helm-config.yaml)" \
         --include-crds \
         -n "$(yq "explode(.) | .namespace" helm-config.yaml)" \
         --output-dir "_local/$cluster" \
         --release-name \
         --skip-tests \
         "$(yq "explode(.) | .releaseName" helm-config.yaml)" \
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

`act` reads workflow secrets, such as `STEADOPS_HELM_RENOVATION_MS_TEAMS_WEBHOOK` and
`STEADOPS_HELM_RENOVATION_ERROR_MS_TEAMS_WEBHOOK`, from a `.secrets` file in `KEY=value` format. The file is
git-ignored. The unittest workflow never sends Teams notifications under `act`.

> [!NOTE]
> The hydration workflow skips opening pull requests under `act`, so a local run only renders the manifests.

## Continuous Integration

All three workflows call reusable workflows from
[`steadforce/steadops-workflows`](https://github.com/steadforce/steadops-workflows), pinned to `v4.2.0`.

- `helm-unittest.yaml` runs on every push. It installs the dependencies pinned in `Chart.lock`, runs the Helm
  unittest suite including subchart tests, publishes a JUnit test report, and runs `helm lint`.
- `helm-hydration.yaml` runs on pushes to `main` and on demand. For every environment in `helm-config.yaml` it
  ensures the `environments/<env>` branch and an `env: <env>` label exist, renders the manifests with
  `helm template`, and opens one pull request per environment against the `environments/<env>` branch watched by
  ArgoCD. It also adds the ArgoCD `ServerSideApply=true` sync option and sync wave `-1` to CRDs. The workflow grants
  `contents`, `pull-requests`, and `issues` write permissions, the last one for creating the labels.
- `trufflehog.yaml` scans the commits of pushes and pull requests to `main`, and runs on demand, for leaked
  secrets.

### Microsoft Teams Notifications

On branches starting with `renovate/`, the unittest workflow posts its result to Microsoft Teams, so Renovate
updates can be merged with confidence or are flagged right away:

| Result | Repository Secret |
| --- | --- |
| Success | `STEADOPS_HELM_RENOVATION_MS_TEAMS_WEBHOOK` |
| Failure | `STEADOPS_HELM_RENOVATION_ERROR_MS_TEAMS_WEBHOOK`, a separate error channel |

Both secrets are optional and hold a Microsoft Teams Workflows webhook URL. When the error webhook is not set,
failures go to `STEADOPS_HELM_RENOVATION_MS_TEAMS_WEBHOOK` instead. Without either secret, no notification is sent.

### Hydrate a Branch Manually

Open **Actions > Helm hydration > Run workflow**, pick the branch under **Use workflow from**, and start the run.

> [!WARNING]
> A manual run opens real pull requests against the `environments/<env>` branches. The pull request branch is named
> `hydration-pull-request/<env>-<version>`, where `<version>` is the `renovate` subchart version from `Chart.lock`
> with the patch level replaced by `x`. A branch with the same minor version as `main` therefore updates the same
> pull request and replaces its manifests.

## Dependency Updates

Renovate keeps the dependencies of this repository up to date, as configured in `renovate.json`:

- Minor and patch updates of all dependencies, including the `renovate` and `valkey` subcharts in `Chart.yaml`,
  are automerged with a squash commit. Major updates need a manual merge, and each `renovate` major version gets
  its own pull request.
- GitHub Actions updates, including `steadforce/steadops-workflows`, are automerged for all update types.
- Image references in `values*.yaml` files are updated as well.

Renovate updates `Chart.lock` together with `Chart.yaml`. When changing a dependency in `Chart.yaml` by hand,
regenerate the lock file and commit it together with `Chart.yaml`; hydration fails when the two disagree.

In the workbench:

```sh
 helm dependency update .
```

Without the workbench:

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm dependency update .
```

## Resources

- [Renovate documentation](https://docs.renovatebot.com/)
- [Renovate Helm chart](https://github.com/renovatebot/helm-charts/tree/main/charts/renovate)
- [Valkey Helm chart](https://github.com/valkey-io/valkey-helm)
- [redis_exporter](https://github.com/oliver006/redis_exporter)
- [Helm chart dependencies](https://helm.sh/docs/topics/charts/#chart-dependencies)
