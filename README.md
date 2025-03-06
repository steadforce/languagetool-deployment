# Language Tool

Repository containing manifests for [languagetool](https://github.com/Erikvl87/docker-languagetool).
Never install the content of this repo on our clusters manually. This is all done by argocd.

## run helm unittests

```shell
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest .
```

Or with output in JUnit format:

```shell
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest -o test-output.xml .
```

## Render resource locally

```
 helm template -n languagetool --release-name languagetool --include-crds --skip-tests \
  -a networking.istio.io/v1beta1 \
  -f values-local.yaml --output-dir _local .
```
