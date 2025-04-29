# Platform helm charts

A opinionated example on how to deploy open source helm charts to each set environment Kubernetes clusters.

This method makes use of the following open source tools.

- Argocd
- Kustomize
- kubernetes

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


### Reference a chart in environment

TODO

- Implement updatecli