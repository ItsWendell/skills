# Scoring concepts (shared reference)

A shared reference linked from every scoring-aware skill. Skills should not duplicate these concepts — link here.

Scoring turns signal results into structured, calibrated **fit** and **urgency** scores (0–100) for companies and contacts. The platform handles compute; skills configure profiles, assign objects, and read scores.

## Object types and IDs

| Object type | Identifier used everywhere |
|---|---|
| `company` | The company **domain** (e.g. `acme.com`) |
| `contact` | The contact's **LinkedIn profile URL** (e.g. `https://linkedin.com/in/jane`) |

Profile type is fixed at creation and cannot change later.

## Dimensions

Every score is one **(profile, object, dimension)** triple. Two dimensions:

- **`fit`** — does this object match the ICP? Built from `icp_fit` signals plus persistent buying-pain evidence.
- **`urgency`** — is the timing right? Built from time-bounded `urgency` and `buying_signal` signals.

A profile can have rules in either dimension or both. Reading scores returns one row per dimension that has at least one rule.

## Profile

A named, org-scoped container of rules, bound to one object type:

```bash
saber scoring profile create --type company --name "EMEA Enterprise"
saber scoring profile list
saber scoring profile get <profileId>
saber scoring profile update <profileId> --name "..." [--description "..."]
saber scoring profile delete <profileId>   # cascades: rules, assignments, scores
```

## Rule

A rule maps one **signal template** to a **dimension** with **typed point values**. Rules are keyed on template IDs — ad-hoc signals (those run via `saber signal --question ...`, or via subscriptions created without a template attached) are invisible to rules until they're consolidated. See `extract-signal-templates` for the one-shot migration flow that converts historical ad-hoc signals into reusable templates.

Point-values shape must match the signal template's answer type; the server returns 422 `INVALID_POINT_VALUES` on mismatch rather than failing silently at compute.

| Signal answer type | Point-values shape | CLI flags |
|---|---|---|
| `boolean` | `{ true: <points>, false: <points> }` | `--true 20 --false -5` |
| `number` / `percentage` / `currency` | `[{ min, max, points }]` (max exclusive) | `--range "0:500:5" --range "500:5000:15"` |
| `list` | `{ "<choice>": <points> }` | `--choice "Salesforce:10" --choice "HubSpot:8"` |

Or pass raw JSON:
```bash
--points '{"ranges":[{"min":0,"max":500,"points":5}]}'
--points-file rules.json
```

```bash
saber scoring rule upsert <profileId> \
  --signal-template <id> --dimension fit --answer-type boolean --true 20 --false 0
saber scoring rule list <profileId>
saber scoring rule delete <profileId> <ruleId>
```

Upsert is keyed on `(profileId, signalTemplateId, dimension)` — calling it again replaces the rule. **Every upsert and every delete triggers a recompute for every object assigned to the profile.**

## Assignment

Assignments link a profile to a single company or contact. Creating an assignment triggers an immediate compute.

```bash
saber scoring assignment create --profile <id> --type company --object acme.com
saber scoring assignment bulk    --profile <id> --type company \
  --object acme.com --object stripe.com --object ramp.com
saber scoring assignment list --type company --object acme.com
saber scoring assignment delete <assignmentId>   # also clears that assignment's scores
```

Bulk is idempotent — duplicate `(profile, object)` pairs are skipped silently (only newly created rows are returned).

## Compute

Compute runs asynchronously via Temporal. Scores recompute automatically on:

- **Signal completion** — a signal underlying any rule finishes; every object assigned to a profile that uses that signal recomputes.
- **Rule upsert or delete** — every object assigned to the affected profile recomputes.
- **Assignment create** — a single compute fires for the new (profile, object) pair.

You usually do not need to call compute manually. For an immediate refresh independent of those triggers (e.g. just before a meeting, or after out-of-band signal data changes):

```bash
saber scoring compute --type company --object acme.com --object stripe.com
```

Idempotent — duplicate triggers attach to the running workflow. Returns 202 with `{queued, failed}`. A `failed > 0` count means some object dispatches errored; retry to pick those up.

## Reading scores

```bash
saber scoring scores --type company --object acme.com [--detailed]
saber scoring scores --type company --object acme.com --object stripe.com
```

Returns one row per `(profile, object, dimension)`:

| Field | Meaning |
|---|---|
| `score` | Current 0–100 score |
| `previousScore` | Last value before this compute (for Δ display) |
| `contributions[]` | Per-rule breakdown: rule id, signal template, matched value, points earned, max points |
| `previousContributions[]` | Same shape, from the previous score (use for diffs) |
| `signalCoverage` | Number of rules where the underlying signal had a value |
| `totalRules` | Total rules in the dimension |
| `computedAt` | Last compute timestamp |
| `version` | Score version, monotonic per object+dimension |

Reading scores does **not** consume credits.

## Translating from `generate-signals`

`generate-signals` outputs categories (`icp_fit`, `urgency`, `buying_signal`) and weights (1–3). Default mapping when you materialize that model as scoring rules:

| `generate-signals` category | Default `dimension` |
|---|---|
| `icp_fit` | `fit` |
| `buying_signal` | `urgency` (treat pain evidence as a timing trigger) |
| `urgency` | `urgency` |

Translating weights to point values:

- **Boolean signals** — `--true <weight × 10> --false 0`. For disqualifiers, set false to a strong negative (e.g. `--false -100`).
- **Number signals** — bucket the meaningful ranges; assign points scaled by weight.
- **List signals** — pick the relevant choices and assign points scaled by weight.

These are starting points — `manage-scoring` covers tuning after you see real scores.

## Common flow

```bash
# 1. Profile per object type
saber scoring profile create --type company --name "ICP scoring"

# 2. One rule per signal template
saber scoring rule upsert <profileId> --signal-template <sigA> --dimension fit \
  --answer-type boolean --true 20 --false 0
saber scoring rule upsert <profileId> --signal-template <sigB> --dimension urgency \
  --answer-type boolean --true 15 --false 0

# 3. Bulk-assign a list (compute kicks off)
saber list company companies <listId> --json | jq -r '.[].domain' \
  | xargs -I{} -n1 echo --object {} \
  | xargs saber scoring assignment bulk --profile <profileId> --type company

# 4. Read
saber scoring scores --type company --object acme.com --detailed
```

## Related skills

- `extract-signal-templates` — consolidate historical ad-hoc signals into templates so they can be referenced by rules (one-shot migration)
- `configure-scoring` — first-time setup of a profile and its rules
- `manage-scoring` — inspect, tune, recompute, clean up
- `score-accounts` — rank a list by current scores
- `qualify-inbound` — qualify a single inbound lead using fit + urgency
- `find-expansion-accounts` — expansion-tuned profile against existing customers
