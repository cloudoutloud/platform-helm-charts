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


### Bootstrap Argocd


### Upgrading Argocd


### Adding a open source chart


### Reference a chart in environment


TODO

- Implement updatecli