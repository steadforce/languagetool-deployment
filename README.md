# Language Tool

Repository containing manifests for [languagetool](https://github.com/Erikvl87/docker-languagetool).
Never install the content of this repo on our clusters manually. This is all done by argocd.

## Testing

The suites in [`tests/`](tests) render the chart with the stage file (`values-local.yaml`,
`values-development.yaml`, `values-production.yaml`), as Argo CD does. They assert the `erikvl87/languagetool`
image, that local renders no container resources while development and production set CPU and memory limits and
requests, the virtual service host per stage, the acme gateway in the ingress gateway namespace passed by Argo CD,
and that the virtual service is only rendered when the istio networking API is available.

```shell
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

## Render resource locally

```
 helm template -n languagetool --release-name languagetool --include-crds --skip-tests \
  -a networking.istio.io/v1beta1 \
  -f values-local.yaml --output-dir _local .
```
