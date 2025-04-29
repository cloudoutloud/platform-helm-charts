# Environment root apps

These are too be applied manually to each clusters where argocd is running, then these apps will track the environment sub folders.

The app of apps pattern allows us to manage multiple Argocd apps from a single source.

To apply run command on cluster. This will create a Argocd project and application.
```
kubectl apply -f development-root-app.yaml -n argocd
```

For example the development root app has `path` set as `path: environment/development` Argocd will deploy all of this folder.