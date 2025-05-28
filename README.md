# Renovate helm deployment

Repository containing manifests for [Renovate](https://github.com/renovatebot/renovate).
Never install the content of this repo on our clusters manually. This is all done by argocd.

## Dependencies

This chart pulls in `renovate` as a dependency. The version
used is specified in `Chart.yaml` in the `dependencies` section.
If you change the version in there, you need to then run

    $ helm dependency update

in order to have the chart downloaded to the `charts` directory
and then also commit that new version alongside with the altered
`Chart.yaml` file.

See the [Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies)
for details.

## Updating

Check the [Renovate Helm values file](https://github.com/renovatebot/helm-charts/blob/main/charts/renovate/values.yaml)
(select updated tag version) on Github for the used Renovate image version.
This version (without patch number) should also be used in `values.yaml`.

Verify on our [Renovate image repository](https://gitea.cloud01.intern.steadforce.com/steadforce-applications/renovate-image/commits/branch/main),
that this particular version was already pushed to Harbor. If not try to build this version on
[Jenkins](https://jenkins-steadops.k8s01.steadforce.com/job/steadforce-applications/job/renovate-image/job/main/),
if the corresponding Pull Request was already merged. If the version is not present at all on
the repository commits, update the image (and build it with Jenkins) or choose an existing
image, which is nearby the required version.

## Render resource local

```
  helm template -n renovate --release-name renovate --include-crds --skip-tests \
  -a batch/v1/CronJob \
  -f values-local.yaml --output-dir _local .
```

## Render resource locally

### local

```shell
 helm template \
  --include-crds \
  --output-dir _local/local \
  --release-name renovate \
  --skip-tests \
  -a external-secrets.io/v1beta1/ExternalSecret \
  -f values-subchart-overrides.yaml \
  -f values-local.yaml \
  -n renovate \
  .
```

### development

```shell
 helm template \
  --include-crds \
  --output-dir _local/dev \
  --release-name renovate \
  --skip-tests \
  -a external-secrets.io/v1beta1/ExternalSecret \
  -f values-subchart-overrides.yaml \
  -f values-development.yaml \
  -n renovate \
  .
```

### production

```shell
 helm template \
  --include-crds \
  --output-dir _local/prod \
  --release-name renovate \
  --skip-tests \
  -a external-secrets.io/v1beta1/ExternalSecret \
  -f values-subchart-overrides.yaml \
  -f values-production.yaml \
  -n renovate \
  .
```
