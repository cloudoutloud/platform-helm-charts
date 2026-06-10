---
name: update-chart
description: Bump an existing Helm chart in this repo to a newer upstream version and surface any breaking values changes. Invoke when the user asks to "update", "upgrade", or "bump" a chart already present under global/<chart>/. Resolves the latest stable upstream version, diffs the chart's default values against the currently pinned version, flags removed/renamed keys that the repo actually uses, updates each dev overlay's `targetRevision` patch (overlays only — never writes to global/, since prod consumes it), validates with `kustomize build`, and reports what changed. Dev only — never touches environments/production/.
---

# Update a chart

Bump an existing Helm chart in this repo to a newer upstream version. The chart must already live under `global/<chart>/`; this skill is *not* for onboarding new charts (use `add-chart` for that). It works the version bump end-to-end:

1. resolves the latest stable version,
2. diffs the chart's published default `values.yaml` between the old and new version,
3. cross-references that diff with the values *this repo actually overrides* (in `valuesObject`, `values-common.yaml`, and each cluster's `override-values.yaml`) so you only flag changes that will actually bite,
4. updates each active dev overlay's `targetRevision` patch (overlays only — never writes to `global/`, because `global/` is the shared base that prod also consumes),
5. validates each dev cluster with `kustomize build`.

Dev-only, same as `add-chart` — never edits `environments/production/` and never edits `global/`. Promotion to prod is a separate workflow.

## When to use

- User asks to update, upgrade, or bump an existing chart by name (e.g. "update cert-manager", "bump kube-prometheus-stack to latest", "upgrade external-dns to 1.22.0").
- User wants to know if a chart has a newer version and what would break before pulling the trigger.
- User points at an existing `updatecli` PR and wants the values-impact analysis the bot doesn't do.

## When NOT to use

- Chart is not yet in `global/` — use `add-chart` to onboard it first.
- User wants to change *values* on a chart without changing the version — edit the relevant `valuesObject` / `values-common.yaml` / `override-values.yaml` directly.
- User asks to bump a chart in production — refuse and stop. Production bumps go through a separate promotion flow.
- User wants to change the upstream repo URL — that's an onboarding-level change, not an update. Treat it as removing and re-adding via `add-chart`.
- Bumping Argocd itself — see `bootstrap-argocd-chart/`.

## Required inputs

- `chart-name` — the folder under `global/` (e.g. `cert-manager`, `kube-prometheus-stack`). **Required.** If the user names something not present under `global/`, stop and suggest `add-chart`.
- `targetRevision` — the version to bump to. **Optional.** If omitted, resolve the latest stable upstream version (see step 2) and confirm it back in the final report before writing.

Do not ask which cluster — every overlay under `environments/development/<cluster>/<chart-name>/` is updated in lockstep so the dev fleet stays on one version. Skipping a cluster would silently introduce drift.

## Instructions

1. **Locate the chart and find the actually-deployed version.** Read `global/<chart-name>/app.yaml`. Capture:
   - `repoURL` and `chart` (from `spec.source` or `spec.sources[0]`).
   - Whether the chart uses `valuesObject` (single-source) or `valueFiles` (multi-source). This determines the JSON-patch path: `/spec/source/targetRevision` vs `/spec/sources/0/targetRevision`.
   - The `valuesObject` content if present, or the `values-common.yaml` file if using `valueFiles`. Save this — step 4 needs to know which keys this repo actually sets.

   If the path doesn't exist, stop and tell the user the chart isn't onboarded yet — point them at `/add-chart`.

   **`global/<chart>/app.yaml` is read-only for this skill — read it, never write to it.** Global is the shared base that both dev *and prod* overlays consume; writing to it would silently change the prod-deployed version on the next sync. Reading it is fine and necessary (we need `targetRevision` from global as the fallback when an overlay doesn't patch it). Capture it as `GLOBAL_VERSION`.

   For each active dev cluster's `environments/development/<cluster>/<chart-name>/kustomization.yaml`:
   - If the overlay contains a `replace` op on the `targetRevision` path → that cluster's `OLD_VERSION` is the `value:` on that op.
   - Otherwise → that cluster's `OLD_VERSION` is `GLOBAL_VERSION` (the dev overlay is inheriting from global). The bump in step 7 will *add* a new `replace` patch op to the overlay rather than touching global.

   **Drift check:** if active dev overlays disagree on `OLD_VERSION`, stop and tell the user — the fleet is already out of sync, and a single "bump everyone to NEW_VERSION" change would silently mask the drift. Let them converge first.

   Note the version-string format each overlay uses (e.g. `1.16.0` vs `v1.16.0`). Preserve whatever format the overlay already uses when writing `NEW_VERSION` back in step 7. If an overlay had no patch (inherited from global), match the format `helm search` returns rather than copying global's format — that pins the overlay to a clean upstream-tagged version going forward.

   **Active dev clusters.** When iterating `environments/development/<cluster>/`, treat `dev-cluster-02` as a structural placeholder and skip it silently — it isn't a real cluster and exists only to demonstrate how a second dev cluster would be wired up. The current active dev cluster is `dev-cluster-01`. If a future second active cluster lands, this rule should be revisited.

2. **Resolve `NEW_VERSION`.** Skip if the user gave one explicitly; otherwise look it up against the same `repoURL` already in `app.yaml`:
   - Preferred (Helm CLI). Use a per-invocation repo alias of the form `update-chart-<chart-name>` so it's traceable and won't collide with other skill runs or with the user's own helm repos. Clean it up at the end of this step so state doesn't leak across sessions:
     ```sh
     ALIAS="update-chart-<chart-name>"
     helm repo add "$ALIAS" <repoURL> >/dev/null
     helm repo update "$ALIAS" >/dev/null
     helm search repo "$ALIAS/<chart>" --versions -o json \
       | jq -r '[.[] | select(.version | test("-") | not)] | .[0].version'
     helm repo remove "$ALIAS" >/dev/null
     ```
     The `test("-") | not` filter drops pre-releases (`-alpha`, `-rc`, etc.) — match the same rule `add-chart` uses so the two skills stay consistent. Never use a generic alias like `tmp` or `test`: those collide and leave orphaned repo entries that are hard to attribute later.
   - Fallback (no Helm CLI):
     ```sh
     curl -sSL <repoURL>/index.yaml \
       | yq '.entries["<chart>"] | map(select(.version | test("-") | not)) | .[0].version'
     ```
   - If `NEW_VERSION == OLD_VERSION`, stop and report "already on latest stable (X)". Do not write any files. If the user wants to force-pin to a specific older version anyway, they can re-run with `targetRevision` explicit.

   Pin the literal version — no `^`/`~` ranges (Argocd `targetRevision` is a literal).

3. **Diff the chart's default values.** Pull the upstream `values.yaml` for both versions and diff them. This is the analysis users actually want:

   ```sh
   mkdir -p /tmp/update-chart-<chart-name>
   helm show values <repoURL>/<chart> --version <OLD_VERSION> > /tmp/update-chart-<chart-name>/old.yaml
   helm show values <repoURL>/<chart> --version <NEW_VERSION> > /tmp/update-chart-<chart-name>/new.yaml
   diff -u /tmp/update-chart-<chart-name>/old.yaml /tmp/update-chart-<chart-name>/new.yaml
   ```

   If the repo is OCI (e.g. `oci://...`), prefer `helm show values oci://<repo>/<chart> --version <X>`. If `helm show values` fails (rare, usually private repos), fall back to pulling and untarring:
   ```sh
   helm pull <repoURL>/<chart> --version <VERSION> --untar --untardir /tmp/update-chart-<chart-name>/<VERSION>
   ```
   and read `values.yaml` from the extracted directory.

   Bucket the diff into three categories — this is what the final report will surface:
   - **Removed or renamed keys** (top priority — likely breakage).
   - **Default value changes** (medium — only matters if they shift behaviour you depend on).
   - **New keys / new defaults** (low — informational, no action needed unless user wants to opt in).

4. **Cross-reference with this repo's overrides.** A values diff that touches keys the repo doesn't override is mostly noise. The real question is: of the keys this repo *sets*, which ones are removed, renamed, or have changed meaning?

   Collect the override surface:
   - `valuesObject` from `global/<chart-name>/app.yaml` (single-source charts).
   - `global/<chart-name>/values-common.yaml` (multi-source charts).
   - Every `environments/development/<cluster>/<chart-name>/override-values.yaml` that exists.

   Flatten each to dotted key paths (e.g. `webhook.resources.limits.cpu`). For each key the repo overrides, check whether it still appears in `new.yaml`. Report any that don't — those are the changes the user *must* address before the chart will reconcile cleanly.

   Do NOT auto-rewrite these overrides. Surface them for the user to triage — silent renames break things and erase audit trail.

5. **Check the upstream changelog/release notes** for anything that doesn't show up in a values diff:
   - CRD changes — many charts ship CRDs and require a manual CRD apply between major versions (cert-manager, kube-prometheus-stack, external-secrets are repeat offenders).
   - Minimum Kubernetes version bumps.
   - Hard-coded image-tag or registry changes (relevant if the org mirrors images).

   `helm show chart <repoURL>/<chart> --version <NEW_VERSION>` plus a quick fetch of the project's GitHub releases page is usually enough. Surface the relevant bullet points in the final report — do not paste the whole changelog.

6. **Discover the dev clusters and their overlays.** List subdirectories of `environments/development/` (excluding `kustomization.yaml`). For each cluster, check whether `environments/development/<cluster>/<chart-name>/kustomization.yaml` exists. Clusters without an overlay for this chart are skipped silently — they don't deploy it.

   If the user named a non-dev environment (e.g. `production`), refuse and stop. Do not write to `environments/production/` under any circumstances.

7. **Update `targetRevision` — overlays only.** Edits land exclusively in active dev overlay files. `global/<chart>/app.yaml` is never written by this skill (writing would also bump prod). For each active dev overlay:

   a. **Overlay already has a `replace` op on the `targetRevision` path** → update that op's `value:` to `NEW_VERSION`. Use `/spec/source/targetRevision` for single-source charts, `/spec/sources/0/targetRevision` for multi-source. Leave every other patch op untouched.

   b. **Overlay does NOT patch `targetRevision` (it was inheriting global)** → add a new `replace` patch op for the same path pinning `NEW_VERSION`. The overlay is now pinned to the upstream version going forward; global is untouched, so prod's inherited version stays exactly where it was. Insert the new op alongside the existing ops in the same `patch: |-` block — don't introduce a second patch target.

   Production overlays under `environments/production/` and any other non-dev environment are NEVER touched.

8. **Validate.** For each dev cluster that has an overlay for this chart, run from the repo root:
   ```sh
   kustomize build environments/development/<cluster>
   ```
   Confirm it builds and grep the output to verify the `Application` shows `NEW_VERSION`. CI runs the same check on every PR — if local passes, CI should too.

9. **Report what changed.** Structure the final report so the user can act on it without re-deriving anything:

   - **Version**: `<chart-name>: <OLD_VERSION> → <NEW_VERSION>` (state how you resolved `NEW_VERSION` — user-supplied vs latest-stable lookup — so they can sanity-check).
   - **Files modified**: bullet list. The global `app.yaml` plus one entry per cluster overlay touched.
   - **Breaking values changes you must address**: keys this repo overrides that are gone or renamed upstream (from step 4). If empty, say so explicitly — silence is too easy to misread as "didn't check".
   - **Other values changes worth a look**: defaults that changed in keys the repo *doesn't* set, but that affect observable behaviour. Keep this short — link to the diff file path instead of pasting the diff.
   - **Out-of-band actions**: CRD bumps to apply manually, k8s minimum version, image registry changes (from step 5). Mark as **REQUIRED** vs FYI.
   - **Next step**: open a PR; once merged, Argocd will reconcile each cluster on its next sync cycle.

   Leave `/tmp/update-chart-<chart-name>/` in place so the user can re-inspect the raw old/new `values.yaml` and the diff. Mention the directory in the report.

## Conventions to preserve

- Every dev cluster's overlay for the chart moves to the same version in a single change — never half-update the fleet.
- Pin a literal version. No version ranges, no `latest`.
- JSON 6902 patches (`op: replace`, `path: /...`) stay JSON 6902 patches — match the existing overlay style.
- Don't modify `valuesObject`, `values-common.yaml`, or `override-values.yaml` content as part of an update. Bumping the version and rewriting values in the same PR makes review harder and rollbacks ambiguous. Surface what needs to change; let the user do it in a follow-up commit (or the same PR after seeing your report).
- Don't add new overlays for clusters that didn't previously deploy the chart — that's a deployment decision, not a version bump.

## Examples

### Example 1: simple bump, no values impact

**User:** "Update external-dns"

**Skill behavior:**
- Reads `global/external-dns/app.yaml` for `repoURL`, `chart`, single-source `valuesObject` (`sources`, `policy`, `replicaCount`, `resources`, `serviceAccount.create`), and `GLOBAL_VERSION` as fallback only.
- Reads the active dev overlay (`environments/development/dev-cluster-01/external-dns/kustomization.yaml`). It patches `targetRevision`, so `OLD_VERSION` = the overlay's patched value, not global.
- Resolves latest stable from `https://kubernetes-sigs.github.io/external-dns/` → e.g. `1.22.0`.
- Pulls `helm show values --repo … external-dns --version <OLD>` and `--version 1.22.0`, diffs. None of the keys the repo overrides have been removed or renamed.
- Updates only `environments/development/dev-cluster-01/external-dns/kustomization.yaml` — bumps the existing `replace` op's `value:` to `1.22.0`. `global/external-dns/app.yaml` is left exactly as it was.
- `kustomize build` succeeds.
- Reports: `external-dns: <OLD> → 1.22.0`, no breaking values changes, no out-of-band actions, no global edits.

### Example 2: breaking values rename

**User:** "Bump cert-manager to latest"

**Skill behavior:**
- Reads `global/cert-manager/app.yaml` for `repoURL` + `valuesObject` overrides (`crds.install`, `resources.*`, `webhook.resources.*`, `cainjector.resources.*`). Global's `targetRevision: v1.8.0` is captured as `GLOBAL_VERSION` for fallback only.
- Reads the active dev overlay (`dev-cluster-01`). It patches `targetRevision` to `1.16.0` → `OLD_VERSION = 1.16.0`. Note the format drops the `v` prefix; `NEW_VERSION` will be written the same way.
- Resolves latest stable, e.g. `v1.20.2`. Normalises to `1.20.2` to match the overlay's format.
- Diff shows the repo's `crds.install` key is dead in both v1.16.0 and v1.20.2 (correct key is `crds.enabled`). Pre-existing bug, not a v1.20.2 regression. Flag for the user.
- Release notes mention CRD manifests must be applied separately for the upgrade. Marked **REQUIRED**.
- Updates `targetRevision` to `1.20.2` in the dev overlay only. **Does NOT** touch `global/cert-manager/app.yaml`. **Does NOT** edit the dead `crds.install` automatically — leaves the user to decide on `crds.enabled: true` and CRD handling.
- Final report leads with the `crds.install` finding and the CRD apply step before listing the overlay diff.

### Example 3: no newer stable version

**User:** "Upgrade kube-prometheus-stack"

**Skill behavior:**
- Reads `global/kube-prometheus-stack/app.yaml` for `repoURL`, multi-source `valueFiles`, and `GLOBAL_VERSION` as fallback.
- Reads each active dev overlay's `targetRevision` patch. If they all agree on, say, `65.6.0`, that's `OLD_VERSION`.
- Resolves latest stable from `https://prometheus-community.github.io/helm-charts`. If it returns `65.6.0` (already current), stops, reports "already on latest stable (65.6.0) — no changes made", does not run the values diff, does not touch any files.
- If a newer version exists, proceeds with steps 3–9, using `/spec/sources/0/targetRevision` (multi-source) as the overlay patch path and diffing `values-common.yaml` (read-only) plus every cluster's `override-values.yaml` against the new defaults.

### Example 4: user asks to bump in production

**User:** "Update cert-manager in production"

**Skill behavior:**
- Refuses. Replies that this skill only updates `environments/development/` and points at whatever the team's promotion-to-prod workflow is.
- Does not create or modify any files.

### Example 5: chart not yet onboarded

**User:** "Update loki"

**Skill behavior:**
- No `global/loki/` directory. Stops, tells the user loki isn't onboarded yet, and suggests `/add-chart` to add it first.
- Does not modify any files.

## Notes

- This skill complements `updatecli`. `updatecli` opens raw version-bump PRs on a schedule (major versions only, per `updatecli/default.yaml`); this skill is what you reach for when you want the *values-impact analysis* `updatecli` doesn't do, or when bumping minor/patch versions ad-hoc.
- If `helm` is not installed locally, fall back to `curl`/`yq` against `<repoURL>/index.yaml` for version resolution and to `helm pull --untar` substitute via direct chart download — but call out the degraded mode in the report so the user knows the diff was approximated.
- Keep `/tmp/update-chart-<chart-name>/` around for the duration of the session; don't clean it up at the end of the run — the user often wants to re-grep the diff while reviewing the PR.
- If `kustomize build` fails after the version bump, do not roll back automatically. Surface the error and let the user decide — a build failure often means the new chart schema needs a values change the skill deliberately didn't make.
