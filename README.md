# gitops-linkerd

Progressive Delivery with Linkerd, Flagger and Flux v2

## Prerequisites

In order to install the workshop prerequisites you'll need a Kubernetes cluster 1.18
or newer with Load Balancer support and RBAC enabled.

### Install Flux v2

Install the CLI on MacOS and Linux using Homebrew run:

```sh
brew install fluxcd/tap/flux
```

Verify that your cluster satisfies the prerequisites with:

```console
$ flux check --pre

► checking prerequisites
✔ kubectl 1.19.2 >=1.18.0
✔ Kubernetes 1.18.9 >=1.16.0
✔ prerequisites checks passed
```

Install the controllers on your cluster:

```console
$ flux install

✚ generating manifests
✔ manifests build completed
► installing components in flux-system namespace
✔ install completed
◎ verifying installation
✔ source-controller ready
✔ kustomize-controller ready
✔ helm-controller ready
✔ notification-controller ready
✔ install finished
```

## Infrastructure setup

Create a source that points to this repository:

```sh
flux create source git gitops-linkerd \
--url=https://github.com/stefanprodan/gitops-linkerd \
--branch=main
```

Create a Kustomization to reconcile Linkerd on your cluster:

```sh
flux create kustomization linkerd \
--source=gitops-linkerd \
--path="./infrastructure/linkerd" \
--prune=true \
--validation=client \
--interval=1m \
--health-check="Deployment/linkerd-proxy-injector.linkerd"
```

Configure Flagger reconciliation specifying Linkerd as a dependency:

```sh
flux create kustomization flagger \
--depends-on=linkerd \
--source=gitops-linkerd \
--path="./infrastructure/flagger" \
--prune=true \
--validation=client \
--interval=1m \
--health-check="Deployment/flagger.linkerd"
```

Configure Contour reconciliation specifying Linkerd as a dependency:

```sh
flux create kustomization contour \
--depends-on=linkerd \
--source=gitops-linkerd \
--path="./infrastructure/contour" \
--prune=true \
--validation=client \
--interval=1m \
--health-check="Deployment/contour.projectcontour" \
--health-check="DaemonSet/envoy.projectcontour"
```

## Workloads setup

Configure the frontend workload with A/B testing deployment strategy and
the backend workload with progressive traffic shifting:

```sh
flux create kustomization workloads \
--depends-on=linkerd \
--source=gitops-linkerd \
--path="./workloads" \
--prune=true \
--validation=client \
--interval=1m
```
