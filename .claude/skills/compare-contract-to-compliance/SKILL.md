---
name: compare-contract-to-compliance
description: How the compliance routine compares each vendor contract in `contracts/` against `compliance-policy.md`. Use whenever the routine needs to produce per-(contract, policy-rule) verdicts. Iterates over every policy rule, and within each rule scans every block of the contract for evidence.
---

# Compare contract to compliance policy

This skill is the procedure the weekly compliance routine follows. It produces one verdict per `(contract, policy_rule)` pair, with an evidence quote pulled from the relevant block of the contract.

## Inputs

- `compliance-policy.md` at the repo root — the source of truth for the rules. Each top-level `##` heading is one **policy rule**.
- `contracts/*.md` — one Markdown file per vendor contract. Each numbered section (`## 1. Services`, `## 2. Data Processing`, …) is one **block**.

## Output shape

For every `(contract, policy_rule)` pair, emit:

```json
{
  "contract": "<contract filename without .md>",
  "rule": "<policy rule slug, e.g. data-residency>",
  "rule_label": "<human-readable rule title>",
  "status": "pass | violation | unclear",
  "evidence_quote": "<a short verbatim quote from the contract, ≤280 chars, or empty>",
  "evidence_block": "<heading of the contract block the quote came from, or empty>",
  "note": "<one short sentence explaining the verdict>"
}
```

Append the full run to `compliance/history.json` so trends are visible over time.

## Procedure

Follow these steps in order. Do not skip the outer loop — the routine must look at **every policy rule against every contract block**, even when an earlier block already produced a verdict, because a later block can contradict it (e.g. a vendor promises EU residency in §2 but allows US processing in §11).

### 1. Parse the policy

Read `compliance-policy.md`. Split on `## ` headings. For each rule, capture:

- **slug** — kebab-case of the heading (e.g. `data-residency`, `audit-rights`, `termination`, `liability-cap`, `subprocessors`, `breach-notification`, `governing-law`).
- **label** — the heading text.
- **acceptable criteria** — what the policy says is allowed.
- **violation criteria** — what the policy says is a violation, including silence-as-violation if the policy calls it out.

### 2. Parse each contract into blocks

For each file in `contracts/`:

- Split on `## ` headings. Each section is a block. Keep the heading text and the body together.
- Treat the contract preamble (everything above the first `##`) as block 0 ("Preamble") — counterparty, effective date, etc. sometimes carry residency hints.

### 3. The nested loop

```
for each contract C in contracts/:
    for each policy rule P in policy:
        verdict = { contract: C, rule: P.slug, status: "unclear", evidence_quote: "", evidence_block: "", note: "" }

        for each block B in C.blocks:
            assessment = evaluate(B, P)
            if assessment.status == "violation":
                verdict = assessment   # violations win — record and keep scanning
            elif assessment.status == "pass" and verdict.status != "violation":
                verdict = assessment   # upgrade unclear → pass, but never overwrite a violation
            # "unclear" never overwrites an existing pass or violation

        emit verdict
```

**Precedence rule:** within one `(contract, rule)` pair, a single **violation** anywhere in the contract trumps any number of passes — record the violation and keep scanning so the verdict reflects the worst clause, not the first one. A **pass** upgrades an `unclear`. An `unclear` never overwrites anything.

### 4. Evaluating a block against a rule

For each `(block, rule)`:

1. Ask: does this block address this rule at all? Look for the rule's topic words (e.g. for `data-residency`: "data", "region", "residency", "process", "EU", "EEA", country names; for `breach-notification`: "breach", "incident", "notify", "hours"). If nothing matches, return `{ status: "unclear" }`.
2. If the block does address the rule, compare its substance to the policy criteria:
   - Meets or exceeds the policy → `pass`.
   - Falls short of the policy → `violation`.
   - Ambiguous, hedged ("reasonable efforts"), or partial → `unclear` with a note explaining the ambiguity.
3. Pull a short verbatim quote (≤280 chars) as `evidence_quote`. Prefer the single sentence that carries the decisive language. Set `evidence_block` to the block's heading.
4. Write a one-sentence `note` explaining the call — what specifically pushed it to pass / violation / unclear.

### 5. Rule-specific cues

Use these as the substantive criteria when evaluating a block. The phrases are illustrative, not exhaustive — judge meaning, not surface text.

- **data-residency** — Pass: "EU/EEA only", "exclusively within the European …". Violation: explicit US/APAC/Singapore/Malaysia processing of EU data, or hedged "reasonable efforts" without a guarantee. Silence across the whole contract → `unclear` then flag in the note.
- **audit-rights** — Pass: ≤30 days' notice. Unclear: 31–90 days. Violation: >90 days, or no audit clause anywhere in the contract.
- **termination** — Pass: termination for convenience on ≤90 days' notice. Violation: >90 days, or "termination for convenience is not permitted".
- **liability-cap** — Pass: cap ≥ 12 months of fees. Violation: cap < 12 months, or cap excludes data-breach liability.
- **subprocessors** — Pass: written notice ≥30 days before adding a subprocessor. Violation: <30 days, no notice requirement, or "without prior notice".
- **breach-notification** — Pass: ≤72 hours. Violation: >72 hours, "in due course", or no specific window.
- **governing-law** — Pass: England & Wales, Ireland, Delaware, New York, or another mature data-protection jurisdiction. Violation: Malaysia, Indonesia, Vietnam, or other weak-enforcement jurisdictions. Unclear: an unfamiliar jurisdiction — say so in the note.

### 6. Silence is sometimes a violation

If a rule requires the contract to say something (e.g. breach-notification window must be stated) and no block addresses it, the final verdict is `violation`, not `unclear`. Set `evidence_quote` to `""`, `evidence_block` to `""`, and explain in `note`: *"Contract does not state a breach notification window; policy treats silence as a violation."* Rules where silence = violation: `data-residency`, `audit-rights`, `subprocessors`, `breach-notification`.

For the other rules (`termination`, `liability-cap`, `governing-law`), silence stays `unclear`.

### 7. Writing the run

After the nested loop, write one run object:

```json
{
  "ran_at": "<ISO-8601 UTC>",
  "source": "claude",
  "verdicts": [ ... ],
  "summary": { "pass": N, "violation": N, "unclear": N, "total": N }
}
```

Append it to `compliance/history.json` (create the file if missing). Do **not** overwrite previous runs — the dashboard's trend chart depends on the full history.

## Worked example

`contracts/acme-data-platform.md`, rule `data-residency`:

- Block "2. Data Processing" says *"Vendor may process customer data … including the United States, Ireland, and Singapore. Vendor will use reasonable efforts to keep EU data within EU regions but does not guarantee residency."*
- US + Singapore processing of EU data, and only "reasonable efforts" for EU → **violation**.
- `evidence_quote`: `"Vendor may process customer data in any region where Vendor maintains infrastructure, including the United States, Ireland, and Singapore."`
- `evidence_block`: `"2. Data Processing"`
- `note`: `"Allows US and Singapore processing of EU data with only reasonable-efforts language."`

## Common pitfalls

- **Don't stop at the first matching block.** A later clause can override an earlier one. Keep scanning every block.
- **Don't confuse vendor obligations with customer obligations.** "Customer must notify vendor within 24 hours" is not a breach-notification clause.
- **Don't read renewal-notice periods as termination-for-convenience periods.** "120 days before end of term" (renewal) is different from "90 days' notice to terminate for convenience".
- **Quote verbatim.** `evidence_quote` is what the dashboard shows on click-through; paraphrasing breaks the audit trail.
- **One verdict per `(contract, rule)`** — not per block. The nested loop is for evidence-gathering; the emitted output is collapsed to one row per pair.
