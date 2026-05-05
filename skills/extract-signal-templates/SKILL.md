---
name: extract-signal-templates
description: >
  Cluster historical ad-hoc signal questions into reusable signal templates so
  they can be referenced by scoring rules. One-shot migration tool — most orgs
  run it once before configuring scoring on legacy data.
---

# Extract Signal Templates

Use this skill once per org to convert historical ad-hoc signals (signals run via `saber signal --question ...` or via `saber subscription create` without a template) into reusable signal templates. Without this, scoring rules can't reference that historical data — `signal_template_id` is null on those executions.

This is a **v1.5 stop-gap**. New signals created via the latest API path already get a template attached automatically. The skill stops finding work once historical ad-hoc data is consolidated.

For scoring concepts and why templates matter for rules, see [`_shared/scoring.md`](../_shared/scoring.md).

## When to run this skill

- An org has been running ad-hoc signals (or one-off `saber signal --domain ... --question ...` calls) for a while.
- The user is now setting up scoring (`configure-scoring`) and wants historical signal data to count.
- `saber scoring scores --detailed` shows low `signalCoverage` and the user expected coverage to be near `totalRules`.

If the org only runs templated signals, this skill will report zero clusters and exit — that's the expected end-state.

## Saber CLI check

Run `saber --help`. The CLI is required.

## Step 1 — Decide which signal types to process

The flow runs separately for `company` and `contact` signals. Ask the user which they have ad-hoc data for; if unsure, run propose for both.

## Step 2 — Propose

```bash
saber template extract propose --type company --json > /tmp/extract-company.json
```

The propose call runs LLM clustering server-side and **costs credits** (one LLM call per answer-type bucket). The CLI prompts for credit confirmation; pass `--yes` to skip.

The response shape:

```json
{
  "clusters": [
    {
      "kind": "new",
      "name": "Hiring Status",
      "question": "Is the company actively hiring?",
      "answerType": "boolean",
      "executionIds": ["...", "..."],
      "sampleQuestions": ["Is Acme hiring SDRs?", "Is Stripe hiring engineers?"],
      "notes": ""
    },
    {
      "kind": "existing",
      "templateId": "tmpl_...",
      "executionIds": ["...", "..."],
      "sampleQuestions": ["What is Acme's revenue?", "What was Stripe's revenue last year?"]
    }
  ],
  "totalCandidates": 21,
  "processedCandidates": 21,
  "hasMore": false
}
```

If `clusters` is empty (`totalCandidates: 0`), the org has no score-compatible ad-hoc executions — the skill is done.

If `hasMore` is `true`, more than `--max-candidates` (default 100, hard cap 500) ad-hoc executions exist. Run again after applying the first batch.

Repeat for `--type contact` if relevant.

## Step 3 — Review with the user

Present each cluster grouped by `kind`:

```
## Proposed templates — company

### Reuse existing templates ([N] clusters)

[1] → existing template "Question 1" (number)
    Sample questions:
      - What was Acme's revenue last year?
      - What was Stripe's revenue?
    Will attach: 8 executions

### New templates to create ([N] clusters)

[2] new "Hiring Status" (boolean)
    Question: "Is the company actively hiring?"
    Sample questions:
      - Is Acme hiring SDRs?
      - Is Beta hiring engineers?
    Will attach: 3 executions

[3] new "Funding Raised" (number)
    Question: "How much funding has the company raised in the last 12 months?"
    Sample questions:
      - How much did Stripe raise?
    Notes: ambiguous — could overlap with "Funding stage"
    Will attach: 1 execution
```

Watch for these issues — call them out:

- **Hardcoded entity names** — `"What was Stripe's revenue?"` should be genericised to `"What was the company's revenue?"`. The model usually does this, but verify.
- **Over-clustering** — `gross revenue` and `net revenue` should stay separate. If one cluster's sample questions span clearly different intents, split it (drop the cluster, propose again, or hand-edit).
- **Bad existing matches** — if `kind: existing` attaches semantically different signals to a template, the resulting scoring rule will fire on irrelevant data. Drop those clusters from the plan.
- **`notes` field** — surface anything the model flagged as ambiguous.

## Step 4 — Edit the plan

The plan is a JSON file the user can edit before applying. Common edits:

- Drop a cluster — remove it from the `clusters` array entirely.
- Rename or rephrase a `kind: new` cluster — edit `name` and `question`.
- Move an `executionId` between clusters — usually a sign you should drop and re-propose, but possible.

**Don't touch:**
- `executionIds` outside of moving them (the API validates these)
- `sampleQuestions` and `notes` — propose-side only, ignored on apply

## Step 5 — Apply

```bash
saber template extract apply --from-file /tmp/extract-company.json
```

Or pipe straight from propose if no review needed (rare):

```bash
saber template extract propose --type company --yes --json | \
  saber template extract apply --from-file -
```

The response:

```json
{
  "created": [
    { "kind": "new",      "templateId": "...", "versionId": "...", "name": "Hiring Status", "attached": 3 },
    { "kind": "existing", "templateId": "...", "versionId": "...", "name": "Question 1",    "attached": 8 }
  ]
}
```

The whole apply is a single transaction — partial failure isn't possible. Either every cluster lands or none do.

## Step 6 — Confirm and hand off

Show the user a summary:

- N templates created, M templates extended
- X executions attached
- Y candidates remaining (`totalCandidates - processedCandidates` from propose, or run propose again if unsure)

Then:

- **If scoring isn't set up yet:** route to [`configure-scoring`](../configure-scoring/SKILL.md). The new templates are immediately available as rule targets.
- **If scoring is already set up:** route to [`manage-scoring`](../manage-scoring/SKILL.md) to add rules pointing at the freshly-extracted templates. Existing rules that already target the matched-existing templates will pick up the back-filled executions on the next compute (auto-trigger doesn't fire on attachment alone — call `saber scoring compute` for affected objects).

## Troubleshooting

- **409 on apply** — at least one execution in the plan is already attached to a template (likely from a prior partial run). The error response lists the offending IDs; remove them from the plan and retry.
- **422 `INVALID_UUID`** — a malformed `executionId` or `templateId`. The plan was hand-edited incorrectly; re-propose.
- **422 unsupported answerType for `kind: new`** — only `boolean | number | percentage | currency | list` are score-compatible. If you see open-text or json proposals, the propose call wouldn't have included them — likely a hand-edit error.
- **Re-running propose returns zero clusters** — nothing left to extract. Done.

## Limits

- Score-compatible answer types only. Open-text and JSON ad-hoc signals are intentionally skipped because scoring rules can't consume them. They stay as raw executions.
- One-shot migration tool. Re-run only after new ad-hoc signals accumulate; the steady state is zero candidates.
- Per-call cap of 500 candidates server-side; default 100. Use `--max-candidates` to tune.

## Related

- [`_shared/scoring.md`](../_shared/scoring.md) — scoring concepts and why templates matter
- [`configure-scoring`](../configure-scoring/SKILL.md) — natural next step once templates are in place
- [`manage-scoring`](../manage-scoring/SKILL.md) — add new rules referencing the extracted templates
- [`create-company-signals`](../create-company-signals/SKILL.md), [`create-contact-signals`](../create-contact-signals/SKILL.md) — going forward, prefer these to ad-hoc signal calls
