# Deployment for renovate

Repository containing manifests for [Renovate](https://github.com/renovatebot/renovate).
Never install the content of this repo on our clusters manually. This is all done by argocd.

## Dependencies

This chart references `renovate` as a dependency. The version
used is specified in `Chart.yaml` in the `dependencies` section.

See the [Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies)
for details.

# Testing

## Usage of values-subchart-overrides.yaml

The `values-subchart-overrides.yaml` file is used to override values in the subchart(s) used by this chart.
We have to separate the values for the subcharts from the values for the main chart, to be able to
unit test for incompatible changes in values of the subcharts. This is necessary because helm does not allow
switching off the usage of values.yaml. Now it's possible to test if we use the same registry and repository
for images as the subcharts are using.

## Run helm unittests

```shell
 helm dependency update && \
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest .
```

Or with output in JUnit format:

```shell
 helm dependency update && \
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest -o test-output.xml .
```

## Render all manifests locally

```shell
 helm dependency update && \
 for cluster in $(yq '.environments | keys[]' helm-config.yaml); do
    helm template \
      -a "$(cluster=$cluster yq '.environments.[env(cluster)].apis | @csv' helm-config.yaml)" \
      -f "$(cluster=$cluster yq '.environments.[env(cluster)].valueFiles | @csv' helm-config.yaml)" \
      -n $(yq 'explode(.) | .namespace // ""' helm-config.yaml) \
      --output-dir _local/$cluster \
      --include-crds \
      --release-name $(yq 'explode(.) | .releaseName // ""' helm-config.yaml) \
      --skip-tests \
      .
 done
```

## Run GitHub workflows locally

To run all workflows in a local environment, start up the workbench, cd into the folder containing this
`README.md` and execute the following command:

```shell
  act
```

On first execution you're asked which flavour of the act image should be used. Using the default `medium`
is a good starting point.
