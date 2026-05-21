# Repository Summary & Routine Ideas

A small **synthetic enterprise ops environment** designed for incident-response / compliance agent routines. All data is fictional but interconnected — the connections matter, because that's where routines find signal.

## Repository summary

### Directories

| Dir | What | Files |
|---|---|---|
| `issues/` | 5 open production incidents (JSON) — all with `severity: null` and no assignee, ready for triage | PROD-4487, 4498, 4506, 4519, 4521 |
| `runbooks/` | 3 ops playbooks with named symptoms, root causes, and step-by-step fixes | auth-502-windows, cdn-upload-latency, payment-service-degraded |
| `deploys/recent.json` | 5 recent deploys with timestamps, summaries, file lists, rollback availability | payment / signing / auth / frontend / tenant-config |
| `contracts/` | 3 vendor MSAs covering data residency, audit rights, termination, liability, subprocessors, breach notice, governing law | Acme (questionable), Globex (clean), Sirius (very dirty) |
| `compliance-policy.md` | 7 policy rules the contracts must comply with | — |

### The interesting *connections* (this is what makes it good demo material)

The data is wired together so a routine that just reads from one file misses the point. Several built-in storylines:

1. **PROD-4521 → payment-service v4.8.2.** The NPE at `PaymentService.java:142` exactly matches the bug in `runbooks/payment-service-degraded.md` step 4 ("guest checkout flow, `customer.savedPaymentMethod` is null"). The deploy `recent.json` shows tom.bryce shipped "Add guest checkout support" 14 min before the incident opened.
2. **PROD-4487 → tenant-config-service v3.2.1.** Acme Corp's "checkout broken" report opens 21 min *after* the deploy that enables the `guest-checkout` flag for cohort B including Acme — same root cause as PROD-4521, just tenant-scoped.
3. **PROD-4519 → signing-service v2.1.4.** "Reduce URL TTL" deploy matches the cdn-upload-latency runbook exactly: "If issue started within 6 hours of signing-service deploy, suspect URL TTL bug."
4. **PROD-4498** matches `auth-502-windows.md` symptom-for-symptom — but with **no** related deploy. Classic capacity issue, not a regression.
5. **PROD-4506** is a feature request mislabelled as an issue — useful negative example.
6. **auth-service v6.0.0** has `rollback_available: false` and a major-version bump with infra (`redis.tf`) changes — a latent risk waiting for a routine to flag.
7. **Sirius contract** violates ~6 of 7 policy rules; **Acme** violates ~3 (96hr breach, weak EU residency, California governing law arguably fine); **Globex** is clean. A compliance scanner has obvious work to do.

## Routine ideas

Grouped roughly by how interesting they'd be to demo.

### Highest-signal (data is wired for them)

1. **Incident triage** — For each open issue: set severity, assign a label, link the matching runbook, and post a hypothesis comment. Easy win since all 5 are unset.
2. **Deploy ↔ incident correlator** — For each open incident, scan deploys in the previous 6h, surface candidates, and (where `rollback_available: true`) propose a rollback. PROD-4521 + payment-service is the textbook case.
3. **Compliance drift scanner** — Walk `contracts/` against `compliance-policy.md` clause-by-clause, open one issue per violation. Sirius will generate ~6 issues.
4. **Tenant-cohort blast-radius watcher** — When a deploy enables a feature flag for a customer cohort, watch the next 24h for tenant-scoped incidents from those customers. Catches the PROD-4487 / Acme thread automatically.

### Solid but slightly less data-rich

5. **Risky deploy auditor** — Daily scan: flag deploys with `rollback_available: false`, major-version bumps, or infra-file changes (auth-service v6.0.0 hits all three).
6. **Runbook-match assistant** — For each new incident, semantic-match runbooks and post the most relevant one with the specific steps pre-filled. PROD-4498 → auth-502-windows is a perfect fit.
7. **Mislabelled-issue catcher** — Detect issues that look like feature requests (PROD-4506) and re-route them.
8. **Daily ops briefing** — Morning digest grouping open incidents by service, with hot deploys, customer-impact tags, and recommended top-3.

### Compliance-side, longer-horizon

9. **Pre-renewal flagger** — Acme renews automatically unless 120 days' notice given; surface contracts inside the notice window.
10. **Policy drift re-eval** — On any change to `compliance-policy.md`, re-evaluate every contract against the new rules.

---

## Build plan: two routines + a dashboard

We're going to ship **two routines** plus a small **web dashboard** that surfaces their output. The dashboard has two pages, one per routine.

### Routine 1 — Incident triage (event-triggered)

**Trigger:** HTTP `POST` to a webhook endpoint with a new issue payload (id, title, body, reporter, opened_at). Posting drops the issue into `issues/` and kicks the routine.

**What the routine does for each new issue:**
1. Read the issue body.
2. Cross-reference `deploys/recent.json` for any deploy within ~6h before `opened_at` that touches a plausibly-related service (keyword / stack-trace match).
3. Match the issue against the `runbooks/` library and pick the best fit.
4. Assign a severity (P0 / P1 / P2 / P3) using the runbook's severity guide where available, plus customer-impact signals.
5. Write a triage comment back onto the issue containing: severity, suspected deploy, recommended runbook, and a one-line hypothesis.

**Dashboard page 1 — "Triage Live":**
- Live-updating list of issues with their triage state (queued → in-progress → triaged).
- Each row expands to show the hypothesis, linked deploy, linked runbook, and assigned severity.
- A "Post new issue" form (or curl snippet) at the top so demos can fire a POST and watch the row appear and resolve in real time.

### Routine 2 — Weekly compliance scan (scheduled)

**Trigger:** Cron, once per week (e.g. Mondays 08:00 UTC).

**What the routine does:**
1. Read `compliance-policy.md` and parse the 7 rule categories.
2. For each file in `contracts/`, evaluate the contract clause-by-clause against each rule.
3. Produce a per-contract verdict: `{ contract, rule, status: pass | violation | unclear, evidence_quote, note }`.
4. Append the run (timestamp + verdicts) to a history store so trends are visible.

**Dashboard page 2 — "Compliance Over Time":**
- Matrix view: contracts (rows) × policy rules (columns), cells coloured pass/violation/unclear, showing the latest run.
- Trend chart: total violations per week, stacked by contract, so a vendor cleaning up its MSA (or a new policy tightening) is visible at a glance.
- Click into a cell for the evidence quote and run history for that contract×rule pair.

### Shared infrastructure

- **Storage:** routine output written to a small store (flat JSON files in the repo, or a lightweight DB — pick during implementation).
- **API:** thin web service exposing the `POST /issues` webhook for Routine 1, a `GET` for dashboard data, and a manual "run now" endpoint for Routine 2.
- **Dashboard:** two-page web UI that polls (or subscribes to) the API. Page 1 needs near-real-time updates; page 2 can be static-per-load.
- **Scheduling:** Routine 2 runs on a weekly cron; Routine 1 runs on webhook.
