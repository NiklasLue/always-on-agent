---
name: triage-incoming-issue
description: Triage a newly-opened production incident in `issues/<ID>.json`. Use whenever the triage routine fires for a new issue (webhook POST, GitHub `issues.opened`, or manual invocation with an issue ID). Reads the issue, cross-references recent deploys and runbooks, assigns a severity, and writes a triage comment back onto the issue file.
---

# Triage an incoming issue

You are the **incident triage routine**. You have been handed a single issue ID. Your job is to enrich that issue with a severity, a suspected deploy, a matching runbook, and a one-line hypothesis — then persist the result.

Do not ask clarifying questions. Make the call from the data on disk and move on.

## Inputs

- **Issue file**: `issues/<ID>.json` — contains `title`, `body`, `opened_at`, `reporter`, `labels`, `comments`.
- **Deploys**: `deploys/recent.json` — `{ deploys: [{ service, version, deployed_at, summary, files_changed, rollback_available }] }`.
- **Runbooks**: `runbooks/*.md` — each has a `Symptoms`, `First checks`, and `Severity guide` section.

All paths are relative to the repo root (the routine is launched with `cwd = repoRoot`).

## Procedure

Work in this order. Don't skip steps even when the answer looks obvious — the value of the routine is consistency.

### 1. Read the issue

Load `issues/<ID>.json`. Pay attention to:
- **Stack traces and file:line references** (e.g. `PaymentService.java:142`) — these are the strongest runbook signal.
- **Scope language**: "all customers", "p99", "error rate up X%" → broad blast radius. "Tenant X", "Acme Corp", "their seats" → tenant-scoped. A single user → narrow.
- **`opened_at`** — you'll need this for the deploy window.
- **`labels`** — `customer-impact` shifts severity upward.

### 2. Decide if this is even an incident

Some reports are feature requests or questions misfiled as issues (see PROD-4506 for the canonical example). If the body reads like a request ("could we add…", "it would be nice if…") with no error, no stack trace, no degraded metric:
- Set `severity = "P3"`.
- Set `matched_runbook = null`, `suspected_deploy = null`.
- Hypothesis: `"Looks like a feature request, not an incident — recommend re-route to product backlog."`
- Skip the rest of the procedure and jump to step 6.

### 3. Cross-reference recent deploys

Walk `deploys.recent.json`. A deploy is a candidate if **both** are true:
- `deployed_at` is within **6 hours before** `issue.opened_at` (never after — a deploy can't cause an incident that opened before it).
- The deploy's `service`, `summary`, or `files_changed` plausibly relates to the issue text. Look for service names, file paths, stack-trace classes, and feature names mentioned in either side.

If multiple deploys match, prefer the most recent one in the window. If `rollback_available: true`, note it — that affects the recommendation.

If nothing matches, `suspected_deploy = null`. Don't force a match; an unmatched incident is a valid output (PROD-4498 is the canonical case — a capacity issue with no related deploy).

### 4. Match a runbook

For each file in `runbooks/`, compare its `Symptoms` section against the issue text. Score based on:
- Specific identifiers (stack-trace class:line, error code, endpoint path) — strongest signal.
- Symptom phrases ("p99 latency", "502 on login", "upload timeouts").
- Service / subsystem names.

Pick the single best match. If nothing scores above a weak threshold, `matched_runbook = null` and flag for human triage in the hypothesis.

### 5. Assign severity

Default ladder:
- **P0** — broad customer impact ("all customers", "error rate up", "p99 spiked"), or runbook's severity guide says so.
- **P1** — single tenant / cohort affected, OR `customer-impact` label present.
- **P2** — narrow impact but a runbook matched (i.e. it's a known failure mode worth fixing today).
- **P3** — single user, unclear impact, or feature-request misfile.

If the matched runbook has its own `Severity guide`, that overrides the default ladder. The runbook authors know their service better than the heuristic does.

### 6. Write the result back

Update `issues/<ID>.json` in place:

```json
{
  "...": "existing fields preserved",
  "severity": "<P0|P1|P2|P3>",
  "triage": {
    "ran_at": "<ISO timestamp>",
    "matched_runbook": "<filename or null>",
    "suspected_deploy": {
      "service": "...",
      "version": "...",
      "deployed_at": "...",
      "summary": "...",
      "rollback_available": true
    },
    "hypothesis": "<one sentence>",
    "source": "claude"
  },
  "comments": [
    "...existing comments...",
    {
      "author": "triage-routine",
      "posted_at": "<same ISO timestamp>",
      "body": "Triage: severity=<P?>. <hypothesis>"
    }
  ]
}
```

Rules:
- `suspected_deploy` is `null` if none matched — don't fabricate one.
- `hypothesis` is one sentence, present-tense, leads with the suspected cause if known. Examples:
  - `"Likely caused by payment-service v4.8.2 (guest checkout NPE) — matches runbook payment-service-degraded.md step 4; rollback is available."`
  - `"Symptoms match auth-502-windows.md but no recent auth-service deploy — suspected capacity issue, not a regression."`
  - `"No matching deploy or runbook — needs human triage."`
- Preserve all existing fields and comments. Append, don't overwrite.
- Write valid JSON with a trailing newline (`JSON.stringify(issue, null, 2) + "\n"`).

### 7. Report

Print one line to stdout in the form:
```
triaged <ID> -> <SEVERITY> (<runbook or "no runbook">)
```

The server watches the file and the stdout line; that's how the dashboard updates.

## Things to avoid

- **Don't invent a deploy correlation** to look thorough. A null `suspected_deploy` is correct when nothing matches.
- **Don't change `status`, `assignee`, or `reporter`.** Those are owned elsewhere.
- **Don't post more than one triage comment per run.** If `issue.triage` already exists, you are re-triaging — replace the prior `triage` object and append a fresh comment, but don't duplicate.
- **Don't escalate severity past what the evidence supports.** P0 means real, observable broad impact — not "this could theoretically be bad."
- **Don't fall back to the keyword stub** in `routines/triage.js`. That file is the placeholder you're replacing; read the data yourself.

## Reference cases (for calibration)

These exist in the seed data; use them to sanity-check your output.

| Issue | Expected severity | Expected runbook | Expected deploy |
|---|---|---|---|
| PROD-4521 (checkout NPE) | P0 | payment-service-degraded.md | payment-service v4.8.2 (rollback available) |
| PROD-4487 (Acme checkout broken) | P1 | payment-service-degraded.md | tenant-config-service v3.2.1 (guest-checkout flag rollout) |
| PROD-4519 (slow uploads) | P1 or P2 | cdn-upload-latency.md | signing-service v2.1.4 |
| PROD-4498 (login 502s) | P1 | auth-502-windows.md | none — capacity, not a regression |
| PROD-4506 (feature request) | P3 | none | none — re-route, not an incident |

If your output disagrees with these, re-read the issue body before persisting.
