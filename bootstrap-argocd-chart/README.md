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

### Interact with argocd

Currently there is no ingress configured you can port-forward

```
kubectl port-forward svc/argo-cd-argocd-server -n argocd 8080:443
```

Then you can use argocd cli to login and interact via terminal

```
argocd login localhost:8080
```

Password for the argocd instance can be found in secret named `argocd-initial-admin-secret` username is `admin`

### Adding external cluster to Argocd instance

Adding an external cluster to one of the tooling Argocd instance.

Once logged in with the previous command argocd login run

```
argocd cluster add <cluster-context-name>
```

### Upgrade Argocd instance

Upgrading will effect both dev/prod Argocd instance, so test on dev before fulling syncing prod.

When upgrading version check offical [docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/upgrading/overview/) for any breaking changes and address.

Your have to check what app verison alligns with what chart version, use command.

```
helm search repo argo/argo-cd --versions
```

To upgrade update both version in `Chart.yaml`, and update comment on app version.

Then generate new lock using command.

```
helm dep update bootstrap-argocd-chart/argo-cd/
