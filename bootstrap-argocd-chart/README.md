# Bootstrap Argocd chart

Official [chart](https://github.com/argoproj/argo-helm)

This initial chart can be applied manually for the first time on the cluster to bootstrap argocd.

This is to solve the chicken and egg problem you need argocd deployed to automate the deployment of other apps.

## Where to deploy Argocd ?
Argocd can be deployed on any clusters.
You may want a dedicated operations clusters were argocd runs and then can connect to other clusters to manage apps in a more centralized  way.
You may want to deploy one instance of Argocd per a clusters for a more secure approach.

## Steps to deploy manually

You will need the helm cli and kubectl installed.

On clusters run the following to create the argocd namespace.

```
kubectl create ns argocd
```

### Generate a lock file for the chart locally

```
helm repo add argo-cd https://argoproj.github.io/argo-helm
# Will generate a Chart.lock file
helm dep update bootstrap-argocd-chart/argo-cd/
```

### Install the chart

Form the root of the repo

```
helm install argo-cd bootstrap-argocd-chart/argo-cd/ -n argocd
```

Will take a minute to deploy once finished should see output below

```
NAME: argo-cd
LAST DEPLOYED: Wed Apr 30 21:16:26 2025
NAMESPACE: argocd
STATUS: deployed
REVISION: 1
TEST SUITE: None
```