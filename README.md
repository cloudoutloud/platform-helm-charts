# Platform helm charts

An opinionated example on how to deploy open-source Helm charts to each set of environment Kubernetes clusters.

This method makes use of the following open-source tools.

- [Argocd](https://argo-cd.readthedocs.io/en/stable/)
- [Kustomize](https://kustomize.io/)

## What problem does this solve?

As a platform or DevOps engineer, you will be consuming many open-source tools to extend the
functionality of Kubernetes. These typically come packaged in the form of a Helm chart.
You will need to deploy these charts to different environments.

Typically, you normally just consume these charts and override the values when needed based on the environments it's deployed in or functionally needed.

This repo gives a structured approach to deploy open-source charts to a given cluster.

Clusters and any additional infrastructure would be provisioned outside this repo, using popular infrastructure as code.

### General layout

- bootstrap-argocd-chart - Core Argocd chart applied manually
- env-root-apps - Applied manually Argocd apps to track sub environment folders
- environments > development > dev-cluster-01 - Example environment folder
- global > cert-manager - Example 

```
├── bootstrap-argocd-chart
│   ├── README.md
│   └── argo-cd
│       ├── Chart.lock
│       ├── Chart.yaml
│       └── values.yaml
├── env-root-apps
│   ├── README.md
│   ├── development-root-app.yaml
├── environments
│   ├── development
│   │   ├── dev-cluster-01
│   │   │   ├── cert-manager
│   │   │   │   └── kustomization.yaml
│   │   │   ├── kustomization.yaml
│   │   │   └── project.yaml
│   │   ├── dev-cluster-02
│   │   └── kustomization.yaml
│   └── production
└── global
    ├── README.md
    ├── argocd
    │   ├── app.yaml
    │   └── kustomization.yaml
    ├── cert-manager
    │   ├── app.yaml
    │   └── kustomization.yaml
```


### Bootstrap and upgrading Argocd

See dedicated readme in path `bootstrap-argocd-chart`

### Adding a open-source chart

Add a new folder under the global directory, populate it with an app.yaml to define the open source helm chart using a helm Argocd application manifest.

## Additional chart extra config

There are additional config apps deployed separate to main helm charts.
This would normally be resources that consume the chart custom resource definitions (CRD) but sit outside the upstream chart in different namespaces.
This config could also be unrelated to helm charts, if you would like a Argocd application to deploy resources in a certain namespace.

## Values files vs Values objects

Within each global Argocd application manifest you can either specify `valuesObject` in line values or reference a separate file `valueFiles` but you can use both.

For `valueFiles` you can specify a common values files in the global dir and then a override file in a overlay environment folder.

An example of this is in the global folder `kube-prometheus-stack`

## Kustomize

[Kustomize](https://kustomize.io/) tool is used for templating.
Global config application config should be generic enough for any environment. Values can then be override or patches in relevant environment sub folders.

There is a GitHub workflow that runs on every pull request to main, this is to check that Kustomize can build the directory.

TODO

- Implement updatecli
