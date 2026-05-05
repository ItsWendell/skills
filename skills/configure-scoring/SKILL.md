---
name: configure-scoring
description: >
  Set up native scoring — create a profile, translate signal templates into rules, and
  bulk-assign a list so fit and urgency scores compute automatically. Bridges the
  weighted model from generate-signals into the platform.
---

# Configure Scoring

Use this skill once per ICP / use case to wire up native scoring. After this runs, every assigned company or contact will have **fit** and **urgency** scores that auto-recompute when signals fire.

For concept definitions (dimensions, profiles, rules, assignments, point-value shapes, auto-trigger behaviour), see [`_shared/scoring.md`](../_shared/scoring.md).

## Saber CLI check

Run `saber --help`. If the CLI isn't installed, scoring requires it — point the user to saber.app and stop. The platform doesn't expose a non-CLI fallback for scoring configuration.

## Step 1 — Choose object type

Scoring is typed: a profile scores **either** companies (keyed by domain) **or** contacts (keyed by LinkedIn profile URL). You'll need a separate profile per type.

Ask:
> "Are we scoring companies, contacts, or both? Each needs its own profile."

For the rest of the flow, do one type end-to-end before starting the next.

## Step 2 — Create the profile

```bash
saber scoring profile create --type company --name "<descriptive name>" \
  [--description "<optional context>"]
```

Naming guidance: include the use case (e.g. `New business — EMEA mid-market`, `Expansion — existing customers`, `Enterprise outbound`). Profile type cannot be changed later; a typo means deleting and recreating.

Capture the returned `profileId` for the next steps.

## Step 3 — Gather inputs

Before writing rules, collect:

1. **Signal templates already in Saber.** List existing templates the user wants in scope. If signals haven't been activated yet, route to `create-company-signals` / `create-contact-signals` first — rules need a signal template ID to point at.
2. **The weighted model from `generate-signals`** (categories + weights), or whatever the user has informally.

If neither exists, route to `signal-discovery` first.

## Step 4 — Translate signals to rules

For each signal template in scope, decide:

1. **Dimension** — `fit` or `urgency`. Default mapping from `generate-signals` categories:

   | `generate-signals` category | Default dimension |
   |---|---|
   | `icp_fit` | `fit` |
   | `buying_signal` | `urgency` |
   | `urgency` | `urgency` |

   Override if the user has a strong opinion. Disqualifiers (e.g. "is a competitor") usually go in `fit` with strongly negative `false` points.

2. **Answer type** — must match the signal template's answer type exactly. Pulling the wrong shape returns 422 `INVALID_POINT_VALUES`.

3. **Point values** — translate weight (1–3) into points. Default scaling is `points = weight × 10`:

   | Answer type | CLI |
   |---|---|
   | `boolean` | `--true 20 --false 0` (disqualifier: `--false -100`) |
   | `number` / `percentage` / `currency` | `--range "0:500:5" --range "500:5000:15"` (max exclusive, repeatable) |
   | `list` | `--choice "Salesforce:10" --choice "HubSpot:8"` (repeatable) |

Then upsert each rule:

```bash
saber scoring rule upsert <profileId> \
  --signal-template <signalTemplateId> \
  --dimension fit \
  --answer-type boolean \
  --true 20 --false 0
```

For complex shapes you can pass JSON instead:
```bash
saber scoring rule upsert <profileId> \
  --signal-template <id> --dimension urgency --answer-type number \
  --points-file ranges.json
```

Repeat per signal template. Show the user the running list of rules as you go and confirm before assignments.

## Step 5 — Verify rules with one object

Pick one company / contact you can sanity-check (an existing customer is ideal). Assign the profile to just that one object first:

```bash
saber scoring assignment create --profile <profileId> --type company --object <domain>
```

The assignment triggers compute. Read the result with contributions:

```bash
saber scoring scores --type company --object <domain> --detailed
```

Walk through with the user: are the contributing signals what you expected? Any rule firing the wrong way? If a rule is wrong, `rule upsert` again — every upsert recomputes.

## Step 6 — Bulk-assign the full list

Once the rules look right, bulk-assign the rest of the target list:

```bash
saber list company companies <listId>
# extract domains, then:
saber scoring assignment bulk --profile <profileId> --type company \
  --object <domain1> --object <domain2> ...
```

For contact profiles, use LinkedIn profile URLs as object IDs:

```bash
saber scoring assignment bulk --profile <profileId> --type contact \
  --object https://linkedin.com/in/jane --object https://linkedin.com/in/bob
```

Bulk assign is idempotent — duplicates are skipped silently.

## Step 7 — Confirm and hand off

Show a summary:
- Profile name + id
- Rule count by dimension
- Assigned object count

Tell the user:

- **Scores are live.** Read any score with `saber scoring scores --type ... --object ...`.
- **Auto-trigger is on.** Scores recompute automatically when underlying signals complete; you don't need to call `compute` manually.
- **Next steps:**
  - `score-accounts` — rank the assigned list by current fit + urgency
  - `manage-scoring` — tune rules, add or remove assignments later
  - `qualify-inbound` — qualify a new inbound lead against this profile

## Troubleshooting

- **422 `INVALID_POINT_VALUES`** — the `--answer-type` doesn't match the shape you provided. Fix the shape and retry the upsert.
- **Score is 0 with no contributions** — no signal data exists yet for that object, or no rules cover the signals it has. Check `signalCoverage` vs. `totalRules` from `--detailed` output.
- **Wrong type** — profile type is immutable. Delete and recreate with the correct type.
