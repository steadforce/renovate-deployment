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

## Secrets

We use [external secrets](https://external-secrets.io) to manage the secrets needed for this deployment.
For documentation on how to provide these secrets, take a look at the
[external-secrets-deployment](https://github.com/steadforce/external-secrets-deployment) README.md.

# Testing

## values-subchart-overrides.yaml

The `values-subchart-overrides.yaml` file is used to override values in the renovate chart.
We have to separate the values for the subcharts from the values for the main chart, to be able to
unit test for incompatible changes in values of the subcharts. This is necessary because helm does not allow
switching off the usage of values.yaml. Now it's possible to test if we use the same registry and repository
for images as the subcharts are using.

## run helm unittests

```shell
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest .
```

Or with output in JUnit format:

```shell
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest -o test-output.xml .
```

## Render resource locally

```shell
 for cluster in $(yq 'keys[]' helm-config.yaml); do 
    helm template \
      -a "$(cluster=$cluster yq '.[env(cluster)].apis | @csv' helm-config.yaml)" \
      -f "$(cluster=$cluster yq '.[env(cluster)].valueFiles | @csv' helm-config.yaml)" \
      -n renovate  \
      --output-dir _local/$cluster \
      --include-crds \
      --release-name renovate \
      --skip-tests \
      .
 done
```

## Run act pipeline locally

To run the pipeline in local environment, startup the workbench, cd into the folder containing this
`README.md` and execute the following command:

```shell
  act
```

On first execution you're asked which flavour of the act image should be used. Using the default `medium`
is a good starting point.

