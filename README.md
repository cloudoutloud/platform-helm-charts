# Platform helm charts

A opinionated example on how to deploy open source helm charts to each set environment Kubernetes clusters.

This method makes use of the following open source tools.

- [Argocd](https://argo-cd.readthedocs.io/en/stable/)
- [Kustomize](https://kustomize.io/)

## What problem does this solve?

As a platform or DevOps engineer you will be consuming many open source tools to extend the
functionally of Kubernetes. These typically come packaged in the form of a Helm charts.
You will need to deploy these charts to different environments.

Typically you normally just consume these charts and override the values when needed based on the environments its deployed in or functionally needed.

This repo gives a structure approach to deploy open source charts, to a giving cluster.

### General layout

### Bootstrap and upgrading Argocd

See dedicated readme in path `bootstrap-argocd-chart`

### Adding a open source chart

Add a new folder under the global directory, populate it with an app.yaml to defined the open source helm chart using a helm Argocd application manifest.

## Additional chart extra config

There are additional config apps deployed separate to main helm charts.
This would normally be resources that consume the chart custom resource definitions (CRD) but sit outside the upstream chart in different namespaces.
This config could also be unrelated to helm charts, if you would like a Argocd application to deploy resources in a certain namespace.

## Values files vs Values objects

Within each global Argocd application manifest you can either specifiy `valuesObject` in line vaules or reference a sepertae file `valueFiles` but you can use both.

For `valueFiles` you can specify a common values files in the global dir and then a override file in a overlay environment folder.

An example of this is in the global folder `kube-prometheus-stack`

## Kustomize

[Kustomize](https://kustomize.io/) tool is used for templating.
Global config application config should be generic enough for any environment. Values can then be override or patches in relevant environment sub folders.

There is a GitHub workflow that runs on every pull request to main, this is to check that Kustomize can build the directory.

### Reference a chart in environment

TODO

- Implement updatecli
