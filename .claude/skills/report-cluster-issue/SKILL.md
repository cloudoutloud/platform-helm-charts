---
name: report-cluster-issue
description: File a GitHub issue against a target repo when cluster-analysis (or the user) reports an unhealthy resource. Dedupes against existing open issues by a stable fingerprint, then creates or comments. Uses the GitHub MCP server — never the cluster API — so this skill cannot mutate Kubernetes state. Invoke when the user asks to "raise an issue", "open a ticket", "report this to GitHub", or similar after a cluster finding.
---

# Report Cluster Issue

Take findings produced by `cluster-analysis` (or supplied directly by the user) and open a GitHub issue against a target repo using the GitHub MCP server. Dedupes against existing open issues with a matching fingerprint so re-running on the same problem comments on the existing issue instead of spawning duplicates.

## When to use

- User asks to "raise an issue", "open a ticket", "report this on GitHub" after a cluster sweep.
- An on-call workflow wants a durable record of a cluster failure outside the chat.
- User wants to track a `CrashLoopBackOff`, `ImagePullBackOff`, scheduling failure, etc. as a GitHub issue.

## When NOT to use

- No finding has been produced yet — run `cluster-analysis` first.
- The finding is a transient post-reboot blip that has already cleared. Filing for noise burns the bot's signal.
- User wants to fix the cluster — this skill only files issues; hand off to a remediation workflow.
- Current working directory is not a git repo with a GitHub `origin` — the skill cannot resolve a target repo.

## Read-only guarantee on the cluster

This skill MUST NOT call `kubectl` with any write verb (`delete`, `apply`, `patch`, `edit`, `scale`, `rollout restart`, etc.). It only reads findings already in the conversation and writes to GitHub. If the user asks for a fix mid-run, decline and suggest a remediation workflow.

## GitHub MCP server

All writes go through a single GitHub MCP server named `github`, configured with one fine-grained PAT. That PAT must have `Issues: Read and write` on every repo the bot is allowed to file against — the PAT's repo scope IS the allow-list, so there is no in-skill mapping file.

Tools used (loaded from the `github` server's namespace):

- `mcp__github__list_issues`
- `mcp__github__create_issue`
- `mcp__github__add_issue_comment`

If `ToolSearch` shows them as deferred, load them first with `select:mcp__github__create_issue,mcp__github__list_issues,mcp__github__add_issue_comment`.

If the MCP server is unreachable, stop and surface that clearly. Do not fall back to `gh` — that would file under the wrong identity.

## Instructions

1. **Resolve the target repo.** Run `git -C <cwd> remote get-url origin` and parse it into `owner/name` (handle both `https://github.com/<owner>/<name>(.git)?` and `git@github.com:<owner>/<name>(.git)?`). The skill always files against the repo it lives in — no `repo` input. If the remote isn't GitHub or git isn't initialised, stop and surface that.
2. **Resolve findings.** Accept `findings` either inline from the user message or by reading the most recent `cluster-analysis` summary in the conversation. If neither is present, stop and ask.
2. **Build a fingerprint** for each finding:
   `cluster=<context>; ns=<namespace>; kind=<Pod|Deployment|...>; name=<resource>; class=<CrashLoopBackOff|ImagePullBackOff|FailedScheduling|...>`
3. **Dedupe.** Call `mcp__github__list_issues` with `state=open, labels=cluster-bot` on `repo`, scan bodies for a line `Fingerprint: <key>`. If a match exists:
   - Add a comment via `mcp__github__add_issue_comment` summarising the new occurrence (timestamp, restart count delta, any new event reasons).
   - Do not open a new issue. Report the existing issue URL back to the user.
4. **Create.** If no match, call `mcp__github__create_issue` with:
   - **Title:** `[<cluster>] <class>: <namespace>/<name>` where `<cluster>` is the normalised context (see below).
   - **Labels:** `cluster-bot`, a class label (`crashloop`, `imagepull`, `scheduling`, `oomkilled`, `evicted`, …), and a `cluster:<normalised-context>` label so issues are filterable per cluster.
   - **Body:** the template in "Issue body template" below. The `Fingerprint:` line is required so future runs can dedupe.

   **Cluster-name normalisation.** kube context names can be long or contain characters GitHub trims awkwardly (e.g. `arn:aws:eks:eu-west-1:123:cluster/foo`). Derive the label value as:
   - If the context contains `/`, take the segment after the last `/`.
   - Lowercase; replace any char outside `[a-z0-9._-]` with `-`; collapse repeats; trim to 40 chars.
   - Examples: `kind-kind` → `kind-kind`; `arn:aws:eks:eu-west-1:123:cluster/prod-eu` → `prod-eu`; `gke_my-proj_europe-west2_dev01` → `gke_my-proj_europe-west2_dev01` (already safe).
5. **Report back.** Reply with the issue URL (created or commented) and a one-line summary. Do not paste the full issue body into chat.

## Issue body template

```
## Summary
<one-line description of the failure>

## Resource
- Cluster: <context>
- Namespace: <ns>
- Kind/Name: <kind>/<name>
- Node: <node>

## Evidence
- Status: <Pending|CrashLoopBackOff|...>
- Restarts: <n>
- Last warning events:
  - <timestamp> <reason>: <message>
- Previous-container log tail (if applicable):
  ```
  <last 10–20 lines>
  ```

## Suggested next step
<from cluster-analysis report — do NOT take this action from this skill>

---
Fingerprint: cluster=<ctx>; ns=<ns>; kind=<kind>; name=<name>; class=<class>
Filed by: report-cluster-issue
```

## Inputs

- `findings` (optional): inline finding payload. If omitted, the skill uses the most recent `cluster-analysis` summary in the conversation.
- `dry_run` (optional, default `false`): when `true`, show the title/body/labels that would be filed without calling the MCP server.

Target repo is not an input — it is derived from `git remote get-url origin` in the current working directory. The configured PAT must grant Issues write on that repo.

## Examples

### Example 1: file after a cluster sweep

**User:** "Open an issue for the local-path-provisioner crash loop"

**Skill behavior:**
- Resolves repo from `git remote get-url origin` → e.g. `cloudoutloud/platform-helm-charts`.
- Fingerprint: `cluster=kind-kind; ns=local-path-storage; kind=Pod; name=local-path-provisioner-...; class=CrashLoopBackOff`.
- Normalised cluster label: `cluster:kind-kind`.
- `mcp__github__list_issues` with `labels=cluster-bot, state=open` → no match.
- `mcp__github__create_issue` with title `[kind-kind] CrashLoopBackOff: local-path-storage/local-path-provisioner-...`, labels `cluster-bot, crashloop, cluster:kind-kind`.
- Reports the new issue URL.

### Example 2: dedupe on re-run

**User:** "File this one again" (same finding as Example 1)

**Skill behavior:**
- Same fingerprint; `list_issues` finds the existing open issue.
- `mcp__github__add_issue_comment` posts a re-occurrence note.
- Reports the existing URL — does not create a duplicate.

### Example 3: not in a GitHub repo

**User:** runs the skill from a directory with no GitHub `origin`.

**Skill behavior:**
- `git remote get-url origin` fails or returns a non-GitHub URL.
- Stops and reports that the target repo cannot be resolved. Does not fall back.

## Setup

1. **Create one fine-grained PAT** at https://github.com/settings/personal-access-tokens.
   - Resource owner: the org/user that owns the target repos.
   - Repository access: every repo this bot may file issues against.
   - Permissions: `Issues: Read and write`, `Metadata: Read-only`. Nothing else.
2. **Add the `github` MCP server** to `.mcp.json` at the repo root (committed) or `~/.claude/mcp.json` (personal):

   ```json
   {
     "mcpServers": {
       "github": {
         "command": "npx",
         "args": ["-y", "@modelcontextprotocol/server-github"],
         "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_BOT_PAT}" }
       }
     }
   }
   ```

   Export `GITHUB_BOT_PAT` in your shell rc — never commit the token itself.

3. **Restart Claude Code** so it picks up the MCP server, then confirm the `mcp__github__*` tools are available.
4. **Labels are auto-created** when first used — `create_issue` creates any label it references with a default grey colour and no description. Pre-create `cluster-bot`, the class labels (`crashloop`, `imagepull`, `scheduling`, `oomkilled`, `evicted`), and `cluster:<name>` per cluster only if you want to control their colour/description for readability in the issue list.

## Notes

- Dedupe is per **open** issue. A closed-then-reopening problem files a new issue — deliberate so each incident gets its own thread.
- This skill never deletes or closes issues. Closing is a human decision; the bot only opens and comments.
- If you later need multiple bot identities (different GitHub accounts as issue author), add additional servers under different names (`github-apps`, etc.) and reintroduce a repo→server mapping. For one identity scoped via PAT, you do not need that.
