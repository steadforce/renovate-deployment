# Renovate Deployment

Umbrella Helm chart that packages and configures [Renovate](https://github.com/renovatebot/renovate) together with
a [Valkey](https://valkey.io/) cache for the `local`, `sf-k8s01-dev`, `sf-k8s02-dev`, and `sf-k8s01-prod`
environments.

> [!IMPORTANT]
> Never install the content of this repository on our clusters manually. Deployment is fully managed by ArgoCD.

## Overview

- Runs Renovate as a Kubernetes `CronJob`, backed by a `valkey` cache for faster repeat runs.
- Renders Renovate's onboarding config into a `ConfigMap` that is mounted into the job.
- Fetches the Gitea credentials used by Renovate through an `ExternalSecret`, scoped per environment.
- Ships Helm unittest coverage for every rendered resource, see [Testing](#testing).

## Repository Structure

| File / Directory | Purpose |
| --- | --- |
| `Chart.yaml` | Declares the pinned `renovate` and `valkey` chart dependencies for this umbrella chart. |
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
| `local` | `values-subchart-overrides.yaml`, `values-local.yaml` | Zero resource requests, standalone Valkey (no replicas), 1-minute cronjob schedule, `LOG_LEVEL=debug`. |
| `sf-k8s01-dev`, `sf-k8s02-dev` | `values-subchart-overrides.yaml`, `values-development.yaml` | Base resource limits, replicated Valkey (1 primary + 3 replicas). |
| `sf-k8s01-prod` | `values-subchart-overrides.yaml`, `values-production.yaml` | Increased resource limits, 51-minute `activeDeadlineSeconds`. |

## Prerequisites

Commands below assume the `SteadOps-Steadies-K8s-Workplace` workbench, which ships `helm`, `yq`, `kubectl`,
`hetzner-k3s`, and `act` pre-installed. Each section also lists a containerized alternative for running outside
the workbench.

## Testing

### Run Helm Unittests

```sh
 helm dependency update .
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

In the workbench:

```sh
 helm dependency update .
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
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm dependency update .
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

## Run GitHub Workflows Locally

In the workbench, from the folder containing this `README.md`:

```sh
 act
```

On first execution you are asked which flavour of the `act` image to use. The default `medium` is a good starting
point.

## Continuous Integration

- `helm-unittest.yaml` runs the Helm unittest suite on every push and reports results to Microsoft Teams.
- `helm-hydration.yaml` renders manifests for every environment in `helm-config.yaml` on pushes to `main`.
- `trufflehog.yaml` scans the repository for leaked secrets on pushes and pull requests to `main`, and on demand.

All three call reusable workflows from
[`steadforce/steadops-workflows`](https://github.com/steadforce/steadops-workflows).

## Resources

- [Renovate documentation](https://docs.renovatebot.com/)
- [Renovate Helm chart](https://github.com/renovatebot/helm-charts/tree/main/charts/renovate)
- [Helm chart dependencies](https://helm.sh/docs/topics/charts/#chart-dependencies)
