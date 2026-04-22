---
name: cluster-analysis
description: Read-only inspection of Kubernetes workloads across all namespaces to surface errors, crashes, failed pods, and unhealthy resources. Invoke when the user asks to audit, diagnose, check health of, or find errors in a cluster. MUST NOT delete, remove, patch, or otherwise modify any resources — inspection only.
---

# Cluster Analysis

Inspect Kubernetes workloads across every namespace in the currently-configured cluster and report errors, crashes, restart loops, failed scheduling, image pull failures, and other unhealthy states. This skill is strictly read-only — it never mutates cluster state.

## When to use

- User asks to audit, diagnose, or check the health of a cluster.
- User asks "what's broken", "what's failing", "find errors", or similar across the cluster.
- User wants a cluster-wide summary of pod/deployment/workload status.
- User is triaging an incident and needs a quick scan of unhealthy resources.

## When NOT to use

- User wants to fix, restart, patch, scale, or delete a resource — this skill is inspection-only; hand off to the user or a different workflow.
- User is asking about a single known resource by name — use `kubectl describe` directly instead of running a full-cluster sweep.
- User is asking about cluster configuration, RBAC, or manifests on disk — those are not runtime workload errors.

## Read-only guarantee

This skill uses ONLY these verbs: `get`, `describe`, `logs`, `top`, `events`, `explain`, `api-resources`, `version`, `cluster-info`.

NEVER run: `delete`, `remove`, `apply`, `create`, `patch`, `edit`, `replace`, `scale`, `rollout restart`, `cordon`, `drain`, `label --overwrite`, `annotate --overwrite`, or any command that writes to the API server. If the user asks for a fix during analysis, report the finding and ask them to confirm before taking any mutating action (which is out of scope for this skill).

## Output handling

Claude Code's UI always shows a collapsed tool-output panel for every Bash call — that is a harness feature and cannot be suppressed by the skill. What the skill controls is the **reply text**: do NOT re-quote or paste raw `kubectl` output into your message to the user. Parse the tool result internally and surface only a synthesized report. If the user explicitly asks to see raw output for a specific resource, include just that slice — never dump the full sweep in chat. Rationale: the tool panel already gives the user access to raw output on demand; the value of the reply text is the summary, not a duplicate of the table.

## Instructions

1. **Use default context.** Use whatever context `kubectl` is currently set to — no confirmation needed. Only override if the user explicitly passes the `context` input or names a different context in their request. Run `kubectl cluster-info` and include its output in the final summary report so the user can see which cluster was inspected.
2. **Sweep pods cluster-wide.** Run `kubectl get pods --all-namespaces -o wide` and flag any pod not in `Running`/`Completed` with 0 restarts. Pay attention to: `CrashLoopBackOff`, `ImagePullBackOff`, `ErrImagePull`, `CreateContainerConfigError`, `Error`, `Pending`, `Evicted`, `OOMKilled`, `Terminating` (stuck), and high restart counts.
3. **Pull recent warning events.** Run `kubectl get events --all-namespaces --field-selector type=Warning --sort-by=.lastTimestamp` to capture FailedScheduling, FailedMount, BackOff, Unhealthy probes, etc.
4. **Check higher-level workloads.** Run `kubectl get deployments,statefulsets,daemonsets,jobs --all-namespaces` and flag any where desired ≠ ready/available, or jobs with failures.
5. **Drill into findings.** For each flagged resource, run `kubectl describe` and (for pods) `kubectl logs --previous` if the pod has restarted, so the root cause shows up in the report.
6. **Summarize.** Produce a grouped report: one section per failure class (crash loops, image pull errors, scheduling failures, etc.), with namespace/name, likely cause from events/logs, and a suggested next step — but do not perform that next step.

## Inputs

Optional args passed to the Skill tool:

- `all-namespaces`: sweep every namespace in the cluster (the default behavior). Equivalent to `kubectl -A`. Set explicitly when the user wants to override a previously-scoped request.
- `namespace`: restrict the sweep to a single namespace instead of all namespaces. Mutually exclusive with `all-namespaces`.
- `since`: time window for events/logs (e.g. `1h`, `24h`). Defaults to whatever `kubectl` returns by default.
- `context`: kubeconfig context to use. Defaults to the current context.

## Examples

### Example 1: cluster-wide health check

**User:** "Check the cluster for any errors"

**Skill behavior:**
- Uses the default kubectl context.
- Runs the pod sweep, warning-event query, and workload status checks.
- Groups findings by failure class and reports namespace/name, root-cause hint from events or previous-container logs, and a suggested remediation the user can choose to apply.
- Does not restart, delete, or patch anything.

### Example 2: scoped diagnosis

**User:** "What's failing in the `payments` namespace right now?"

**Skill behavior:**
- Runs the same sweep but scoped with `-n payments`.
- Describes each unhealthy pod and pulls previous-container logs for anything in `CrashLoopBackOff`.
- Reports findings; stops short of any fix.

### Example 3: user asks for a fix mid-analysis

**User:** "Just delete the stuck pod so it reschedules"

**Skill behavior:**
- Declines to delete within this skill (read-only guarantee).
- Reports the finding and suggests the user run the delete themselves, or invoke a different workflow that's authorized to mutate resources.

## Notes

- If `kubectl` is not configured or the cluster is unreachable, stop and surface that clearly — don't guess at state.
- Large clusters: prefer `-o wide` plus filtering over `-o yaml` dumps to keep output scannable.
- `kubectl top` requires metrics-server; if it's missing, skip that step rather than erroring out.
- This skill pairs well with a separate remediation workflow — keep them distinct so the read-only guarantee holds.
