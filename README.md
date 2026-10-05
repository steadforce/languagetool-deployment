# Language Tool Deployment

Helm chart that deploys the [LanguageTool](https://github.com/Erikvl87/docker-languagetool) server with Argo CD.

> [!IMPORTANT]
> Never install the content of this repository on our clusters manually. Argo CD does this.

## Overview

The chart has no dependencies. Its [`templates/`](templates) render:

| Template              | Resource                                                                           |
| --------------------- | ---------------------------------------------------------------------------------- |
| `namespace.yaml`      | The release namespace                                                              |
| `deployment.yaml`     | One `languagetool` replica on port `8010` with a 1 GB Java heap                    |
| `service.yaml`        | `ClusterIP` service `languagetool` on port `80`                                    |
| `storage.yaml`        | 30 Gi `nfs1` volume claim `languagetool-ngrams`, mounted at `/ngrams`              |
| `virtualservice.yaml` | Istio `VirtualService` via the acme gateway, if `networking.istio.io/v1beta1` exists |

[`values.yaml`](values.yaml) sets the istio ingress gateway namespace (`global.istio.ingressGateway.namespace`), the
`erikvl87/languagetool` image, the sub domain `languagetool`, and the container resources. The stage files
(`values-local.yaml`, `values-development.yaml`, `values-production.yaml`) set `global.stage` and the domain
`global.acme.domain`; `values-local.yaml` also removes the container resources.

Argo CD deploys the chart as release `language-tool` into the namespace `language-tool` with the stage file and
passes `global.stage` and `global.istio.ingressGateway.namespace` as parameters.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)

All commands run from the repository root.

## Repository Layout

| Path                  | Purpose                                                  |
| --------------------- | -------------------------------------------------------- |
| `Chart.yaml`          | Chart metadata                                           |
| `values.yaml`         | Image, resources, sub domain, and istio defaults         |
| `values-<stage>.yaml` | Stage and domain for `local`, `development`, `production` |
| `templates/`          | Kubernetes manifests                                     |
| `tests/`              | helm-unittest suites                                     |
| `.github/workflows/`  | Helm unittest and Trufflehog pipelines                   |
| `renovate.json`       | Renovate configuration                                   |

## Rendering

The following command renders every stage like Argo CD does into the gitignored `_local/<stage>` directories:

```sh
 for stage in local development production; do
   docker run \
     -e HOME=/tmp \
     --rm \
     -u $(id -u) \
     -v "$(pwd):/apps" \
     -w /apps \
     alpine/helm template \
       -a networking.istio.io/v1beta1 \
       -f "values-$stage.yaml" \
       --include-crds \
       -n language-tool \
       --output-dir "_local/$stage" \
       --skip-tests \
       language-tool .
 done
```

The `-a` parameter is needed because the chart uses `.Capabilities.APIVersions.Has` to render the Istio
`VirtualService` only when its CRD is installed in the cluster. Helm templating works offline, so it has to be told
which API versions exist.

## Testing

The suites in [`tests/`](tests) render the chart with the stage file (`values-local.yaml`,
`values-development.yaml`, `values-production.yaml`), as Argo CD does. They assert the `erikvl87/languagetool`
image, that local renders no container resources while development and production set CPU and memory limits and
requests, the virtual service host per stage, the acme gateway in the ingress gateway namespace passed by Argo CD,
and that the virtual service is only rendered when the istio networking API is available.

```sh
 docker run \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   helmunittest/helm-unittest .
```

helm-unittest writes XUnit by default. To get a JUnit report in `test-output.xml`, as the pipeline does, add
`-t JUnit -o test-output.xml` before the chart path. Snapshots are written to `tests/__snapshot__/` and are not
committed.

The pipeline also lints the chart:

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm lint .
```

## CI/CD

- [`helm-unittest.yaml`](.github/workflows/helm-unittest.yaml) runs on every push and calls the `helm-unittest.yaml`
  reusable workflow of `steadforce/steadops-workflows` at `v4.2.0`. It installs the latest Helm and the
  helm-unittest plugin from `main`, runs the suites with a published JUnit report, and runs `helm lint`. On
  `renovate/` branches it posts the result to MS Teams: successes to the `STEADOPS_HELM_RENOVATION_MS_TEAMS_WEBHOOK`
  channel, failures to the `STEADOPS_HELM_RENOVATION_ERROR_MS_TEAMS_WEBHOOK` channel.
- [`trufflehog.yaml`](.github/workflows/trufflehog.yaml) scans the pushed or pull request commit range for secrets
  with Trufflehog on pushes and pull requests to `main`, and on manual dispatch.

There is no hydration step: Argo CD renders the chart from this repository directly.

## Dependency Updates

[Renovate](renovate.json) uses `config:recommended` with the dependency dashboard and opens pull requests for the
dependencies it detects, such as the pinned reusable workflows. No automerge is configured.
