# Global application templates

Each subfolder defines a open source chart.

In each chart folder there is an `app.yaml` this defines the Argocd application manifest.

The values should be generic enough to be used across environments dev/prod.

Any common fields you need to override omit values with `"` then these can be patched in the relevant environment folders.