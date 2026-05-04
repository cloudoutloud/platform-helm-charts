---
name: add-chart
description: Add a new open-source Helm chart to this repo as an Argocd Application targeting the development environment. Invoke when the user asks to "add a chart", "onboard a chart", or "deploy a new helm chart" in this repo. Creates the global app.yaml and a dev overlay for every cluster under environments/development/, updates kustomizations and project sourceRepos, and validates with `kustomize build`. Refuses to target production or any non-dev environment.
---

# Add a chart

Onboard a new open-source Helm chart so Argocd deploys it to the **development** environment in this repo. The repo follows a global-template + environment-overlay pattern: a generic `Application` lives under `global/<chart>/`, and each dev-cluster overlay under `environments/development/<cluster>/<chart>/` patches it with cluster-specific values.

This skill is dev-only by design — it never writes outside `environments/development/`. Do not ask which cluster: discover the dev clusters from disk and add the overlay to every one of them. Promotion to production is a separate, deliberate workflow.

## When to use

- User asks to add, onboard, install, or deploy a new open-source chart in this repo.
- User has a chart name, repo URL, and target version, and wants the right files in the right places.

## When NOT to use

- User asks to deploy to production or any non-dev environment — refuse and tell them this skill only targets `environments/development/`.
- Modifying values on a chart that already exists — edit the relevant `override-values.yaml` or `app.yaml` directly.
- Bumping a chart version — that flows through `updatecli` PRs; don't hand-edit unless asked.
- Adding bespoke manifests that aren't a Helm chart — use a separate `Application` with `repoURL` pointing at this repo (see `external-secrets-operator-extra-config` for the pattern), not this skill.
- Bootstrapping or upgrading Argocd itself — see `bootstrap-argocd-chart/`.

## Required inputs

Ask the user for any missing values before writing files. **Do not ask which cluster** — the target is always every dev cluster discovered under `environments/development/` (see step 1).

- `chart-name` — folder name and Argocd `metadata.name` (e.g. `cert-manager`, `kube-prometheus-stack`).
- `chart` — the Helm chart name as published. **If not given, default to `chart-name` and don't ask** — only the rare case where the published chart name differs from the folder name needs an override.
- `repoURL` — upstream Helm repo (e.g. `https://charts.jetstack.io`). **If not given, do not ask — resolve it from `chart-name` via ArtifactHub (see step 2).** The user can still override with an explicit URL.
- `targetRevision` — chart version to pin (e.g. `1.16.0`). Required at both global and overlay layers. **If the user does not supply one, do not ask — resolve the latest stable upstream version yourself (see step 2) and confirm it back in the final report.**
- `namespace` — destination namespace in the cluster.
- `values-style` — `valuesObject` (in-line, simple charts) or `valueFiles` (separate `values-common.yaml` + per-cluster `override-values.yaml`, for charts with large value trees like kube-prometheus-stack).

## Instructions

1. **Discover the dev clusters.** List subdirectories of `environments/development/` (excluding `kustomization.yaml`). Every directory there is a dev cluster overlay target. Do not ask the user which cluster — apply the overlay to all of them. If the user explicitly names a non-dev environment (e.g. `production`), refuse and stop.

2. **Resolve the chart repo and version.** Do not ask the user for either if they didn't supply them — look them up.

   a. **Resolve `repoURL` from `chart-name`** (skip if the user gave one). Two-stage lookup:

   **Stage 1 — direct lookup (preferred).** ArtifactHub's package endpoint resolves the canonical upstream for most well-known charts, even when the repo's ArtifactHub slug differs from the chart name:
   ```sh
   curl -sSL "https://artifacthub.io/api/v1/packages/helm/<chart-name>/<chart-name>" \
     | jq -r '.repository.url // empty'
   ```
   This works for e.g. `grafana` → `https://grafana.github.io/helm-charts` and `cert-manager` → `https://charts.jetstack.io`. If it returns a non-empty URL, use it.

   **Stage 2 — search fallback.** Only run this when stage 1 returns empty/404. ArtifactHub's text search is noisy: packaged forks (e.g. Bitnami) are often `verified_publisher: true` and outrank the upstream, so don't trust verified status alone.
   ```sh
   curl -sSL "https://artifacthub.io/api/v1/packages/search?ts_query_web=<chart-name>&kind=0&limit=20" \
     | jq -r '.packages
         | map(select(.name == "<chart-name>"))
         | sort_by(
             (if .repository.official then 0 else 1 end),
             (if .repository.verified_publisher then 0 else 1 end),
             -(.stars // 0)
           )
         | .[0:5]
         | .[]
         | "\(.repository.url)\t\(.repository.name)\tofficial=\(.repository.official)\tverified=\(.repository.verified_publisher)\tstars=\(.stars // 0)"'
   ```
   In stage 2, if the top row has `official=true`, use it. Otherwise **show the top 3 candidates to the user and ask which to use** — don't silently pick a fork over the upstream.

   Once chosen, **state the resolved repo URL in the final report so the user can override before merging**.

   b. **Resolve `targetRevision`** (skip if the user gave one): use the resolved `repoURL` to find the latest stable version.
   - Preferred (Helm CLI): `helm repo add tmp <repoURL> && helm repo update tmp >/dev/null && helm search repo tmp/<chart> --versions -o json | jq -r '[.[] | select(.version | test("-") | not)] | .[0].version'` — filters out pre-releases (anything containing `-`, e.g. `1.17.0-alpha.1`) and returns the newest stable.
   - Fallback (no Helm CLI): `curl -sSL <repoURL>/index.yaml | yq '.entries["<chart>"] | map(select(.version | test("-") | not)) | .[0].version'`.

   Pin to the exact version (no `^` or `~` ranges — Argocd `targetRevision` takes a literal). Surface both the resolved repo and version in your final report so the user can sanity-check before merging.

3. **Pick the values style.** Use `valuesObject` for small, mostly-generic configs (see `global/cert-manager/app.yaml`). Use `valueFiles` with a `$repo/global/<chart>/values-common.yaml` plus an empty placeholder slot for the overlay file when the value tree is large or differs meaningfully per cluster (see `global/kube-prometheus-stack/app.yaml`). If unsure, ask.

4. **Create the global template** at `global/<chart-name>/`:
   - `app.yaml` — an `argoproj.io/v1alpha1` `Application` in namespace `argocd`. Set `spec.source` (or `spec.sources` for multi-source `valueFiles`), `destination.name: in-cluster`, `destination.namespace: <namespace>`, and `syncPolicy.syncOptions: [CreateNamespace=true]`. Leave `spec.project: ""` — the overlay patches it. For overrideable fields (e.g. `releaseName`), set them to `""` so Kustomize JSON-patch `replace` ops work cleanly in overlays. See `global/README.md`.
   - `kustomization.yaml` — `resources: [app.yaml]`.
   - `values-common.yaml` (only when using `valueFiles`) — generic values that hold across environments.

5. **For each dev cluster from step 1, create the overlay** at `environments/development/<cluster>/<chart-name>/`:
   - `kustomization.yaml` referencing `../../../../global/<chart-name>` under `resources`, plus a `patches` block targeting the `Application` by `group: argoproj.io / version: v1alpha1 / kind: Application / name: <chart-name>`.
   - At minimum, the patch must `replace`:
     - `/metadata/name` → `<cluster>-<chart-name>` (prevents Application-name collisions across clusters in the same Argocd).
     - `/spec/source/targetRevision` (or `/spec/sources/0/targetRevision` for multi-source) → pinned version from step 2.
     - `/spec/project` → `<cluster>-charts` (the per-cluster `AppProject` defined in `project.yaml`).
   - For `valueFiles` charts, also patch `/spec/sources/0/helm/valueFiles/1` to `$repo/environments/development/<cluster>/<chart-name>/override-values.yaml` and `/spec/sources/1/path` to the cluster-config path if applicable. Cross-reference `environments/development/dev-cluster-01/kube-prometheus-stack/kustomization.yaml`.
   - Create `override-values.yaml` if using `valueFiles`.

6. **Register each overlay** in its `environments/development/<cluster>/kustomization.yaml` by appending `- ./<chart-name>` to `resources`.

7. **Authorise the chart's repo** in each per-cluster `AppProject` at `environments/development/<cluster>/project.yaml` — append the upstream `repoURL` to `spec.sourceRepos`. Argocd will refuse to sync the Application if the repo isn't whitelisted; this is the most common omission.

8. **Validate locally.** For each dev cluster, run `kustomize build environments/development/<cluster>` from the repo root and confirm it builds without error and the resulting `Application` shows the patched name, project, and version. The repo's PR CI runs this same check on every change to main.

9. **Report what changed.** List each file created or modified per cluster, **call out the resolved chart version explicitly** (especially when you looked it up rather than being told), and state the follow-up — typically: open a PR; once merged, the development root-app under `env-root-apps/` will pick the new Applications up automatically (no manual `kubectl apply` needed).

## Conventions to preserve

- Folder name MUST equal the chart-name used in `metadata.name` of the global `Application`. Overlays prefix it with `<cluster>-` via patch.
- Generic values live in `global/`; cluster-specific values live in the overlay. Don't put cluster-specific config in `global/`.
- Use JSON 6902 patches in overlays (`op: replace`, `path: /…`), not strategic merge — matches existing files and is what Kustomize honours for `Application` CRDs without an OpenAPI schema.
- `syncPolicy.automated.prune: true` is the default in this repo; keep it unless the user explicitly opts out.
- Don't invent new top-level directories. The structure is fixed: `global/`, `environments/<env>/<cluster>/`, `env-root-apps/`, `bootstrap-argocd-chart/`.

## Examples

### Example 1: simple chart with in-line values

**User:** "Add metrics-server"

**Skill behavior:**
- Lists `environments/development/` and finds the active dev clusters (e.g. `dev-cluster-01`, `dev-cluster-02`).
- Asks for `repoURL` (`https://kubernetes-sigs.github.io/metrics-server/`), `targetRevision`, `namespace` (`kube-system`).
- Picks `valuesObject` style — small chart, no per-cluster divergence expected.
- Creates `global/metrics-server/app.yaml` + `kustomization.yaml`.
- For each dev cluster, creates `environments/development/<cluster>/metrics-server/kustomization.yaml` with patches for name, project, version; appends `- ./metrics-server` to that cluster's `kustomization.yaml`; adds `https://kubernetes-sigs.github.io/metrics-server/` to `sourceRepos` in that cluster's `project.yaml`.
- Runs `kustomize build environments/development/<cluster>` for each and reports the diff.

### Example 2: chart with split values files

**User:** "Add loki, the values are going to be substantial"

**Skill behavior:**
- Same dev-cluster discovery as example 1.
- Uses `valueFiles` style. Mirrors the `kube-prometheus-stack` shape: multi-source `Application` with one source for the chart and one source ref pointing back at this repo.
- Creates `global/loki/app.yaml`, `kustomization.yaml`, and `values-common.yaml`.
- For each dev cluster, creates `environments/development/<cluster>/loki/kustomization.yaml` (with the `valueFiles/1` and `sources/1/path` patches) plus an `override-values.yaml`.
- Wires up each cluster's overlay kustomization and project sourceRepos as in example 1.

### Example 3: minimal input — chart name only

**User:** "Add ingress-nginx"

**Skill behavior:**
- Discovers dev clusters silently — does not ask about clusters.
- Queries ArtifactHub for `ingress-nginx`, picks the verified `kubernetes.github.io/ingress-nginx` repo, then queries that repo's `index.yaml` for the latest stable version. Does not ask for `repoURL` or `targetRevision`.
- Defaults `chart` to `ingress-nginx` (same as `chart-name`).
- Stops and asks only for the still-ambiguous inputs: `namespace` and `valuesObject` vs `valueFiles`.
- Reports the resolved repo URL and version in the final summary so the user can override either before merging.

### Example 4: production request

**User:** "Add cert-manager to prod-cluster-01"

**Skill behavior:**
- Refuses. Replies that this skill only targets `environments/development/` and points the user at whatever the team's promotion-to-prod workflow is (out of scope here).
- Does not create any files.

## Notes

- The PR CI workflow runs `kustomize build` on the repo; if your local build passes, CI should too. If CI fails, read the error rather than re-trying — usually a missing `sourceRepos` entry or a typo'd JSON-patch path.
- `updatecli` will pick the new chart up automatically on its next run and propose major-version bumps via PR (see `updatecli/default.yaml` and the root README's Updatecli section). No extra wiring needed.
- For charts that ship CRDs and need extra config alongside (e.g. ClusterIssuers, ClusterSecretStores), follow the `external-secrets-operator` pattern: a second `Application` in the same `app.yaml` whose `source.repoURL` points at this repo and whose `path` is patched per-cluster to the overlay's `config/` folder.
