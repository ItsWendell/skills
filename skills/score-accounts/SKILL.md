---
name: score-accounts
description: >
  Rank a list of accounts by signal strength to surface the highest-priority targets
  for outreach. Reads native fit + urgency scores from Saber when available, and
  falls back to client-side weighted scoring for Apollo, HubSpot, or pasted data.
---

# Score Accounts

Use this skill to rank accounts so outreach focuses on the highest-intent targets first.

Two paths:
- **Path A — Saber CLI**: read pre-computed `fit` and `urgency` scores from native scoring. No client-side math.
- **Path B — any other source** (Apollo / HubSpot exports, pasted data): client-side weighted scoring against the model from `generate-signals`.

For scoring concepts (dimensions, profiles, contributions, auto-trigger), see [`_shared/scoring.md`](../_shared/scoring.md).

## Step 1 — Determine the path

Run `saber --help` to confirm the CLI is installed.

**If installed:**

```bash
saber list company list
saber scoring profile list
```

If a relevant company list and at least one company-scoped scoring profile exist, take **Path A**.

If a list exists but no scoring profile is assigned to its accounts, route to `configure-scoring` first, then return here.

**If not installed:** take **Path B** with whatever signal data the user has.

---

## Path A — Read native scores

### A1. Pull the list

```bash
saber list company companies <listId>
```

Capture the domains.

### A2. Read scores

Read scores for all domains in a single call (the `--object` flag is repeatable):

```bash
saber scoring scores --type company \
  --object acme.com --object stripe.com --object ramp.com [...] \
  --detailed
```

`--detailed` returns per-rule contributions and the previous score (for Δ display). Reading scores does **not** consume credits.

You'll get one row per `(profile, domain, dimension)`. Group by domain to get fit and urgency side-by-side.

### A3. Decide the sort

Ask the user how to rank when both dimensions exist:

- **Combined (default)** — `(fit + urgency) / 2`. Balanced.
- **Fit-first** — sort by `fit` desc, break ties by `urgency`. Best for cold outbound where ICP-match comes first.
- **Urgency-first** — sort by `urgency` desc, break ties by `fit`. Best for time-sensitive plays (event-triggered campaigns).

### A4. Present

```
## Account Scores — [List Name]

### High priority — combined ≥ 70
| Rank | Company | Domain | Fit | Urgency | Δ | Top contributing rules |
|---|---|---|---|---|---|---|
| 1 | Acme Corp | acme.com | 84 | 72 | +6 | New VP Sales hired (+20), Hiring SDRs (+15) |
| 2 | Beta Inc | beta.io | 76 | 80 | +4 | Series B (+25), HubSpot migration (+15) |

### Medium priority — combined 50–69
| Rank | Company | Domain | Fit | Urgency | Δ | Top contributing rules |
|---|---|---|---|---|---|---|
| ... | | | | | | |

### Watch list — combined 30–49
| Rank | Company | Domain | Fit | Urgency | Δ |
|---|---|---|---|---|---|
| ... | | | | | |

### Deprioritise — combined < 30
| Company | Domain | Notes |
|---|---|---|
| ... | | |
```

Include the Δ (`score - previousScore`) — it's the single best signal for "what changed since last week".

### A5. Watch for low coverage

If `signalCoverage` is much lower than `totalRules` for many objects, the underlying signals haven't fully run yet. Note this in the output:

> "[N] accounts have partial signal coverage — scores will firm up as remaining subscriptions complete."

Auto-trigger keeps scores fresh; you generally don't need to call `saber scoring compute`.

---

## Path B — Client-side weighted scoring

Used when there's no Saber CLI or no scoring profile is set up. Works with Apollo enrichment, HubSpot exports, manual research — any signal data the user can describe.

### B1. Get the signal data

Ask the user for their data. Accepted formats:
- Paste a table or CSV with columns: Company, Domain, [Signal 1], [Signal 2], ...
- Describe what they know per account ("Acme raised a round, Beta is hiring SDRs")
- HubSpot export, Apollo enrichment output

Confirm which signals are present and what a positive result looks like.

### B2. Apply the weighted algorithm

If signal metadata from `generate-signals` is in conversation context, use the full model:

```
1. For each signal, determine if the answer is positive per interpretation rules
2. earnedWeight = sum of weights for positive-answer signals
3. totalWeight  = sum of all signal weights
4. rawScore     = (earnedWeight / totalWeight) × 100

5. Apply penalties:
   — Any disqualifier fires wrong  → score = 0 immediately
   — Zero buying_signal hits       → score = 0 (no pain evidence)
   — Zero urgency hits             → score − 15 (right fit, wrong time)

6. Final score: 0–100
```

**Thresholds:**
- **70+** — High fit, prioritise outreach
- **50–69** — Moderate fit, worth pursuing
- **30–49** — Low fit, monitor for triggers
- **< 30** — Poor fit, deprioritise

If no metadata is available, ask which signals matter most, then:
- **Base:** +1 per positive signal
- **Priority signals:** 2× weight for the user's flagged signals
- **Recency bonus:** +0.5 if from the last 7 days
- Normalise to 0–10 relative to the highest scorer.

### B3. Present

Same output structure as Path A's table, but with one combined score column instead of separate fit + urgency.

---

## Step 4 — Suggest next steps

- **High priority accounts:** use `write-outreach` to draft personalised messages referencing the top contributing signals
- **Watch list:** re-read scores in 2–4 weeks; auto-trigger will have updated them as signals run
- **Low scorers:** pause or remove from active list; revisit if a trigger fires
- Use `deal-coaching` on any high-priority account already in pipeline
- Use `manage-scoring` to tune rules if the rankings don't match your gut

## Related

- [`_shared/scoring.md`](../_shared/scoring.md) — concepts and CLI reference
- `configure-scoring` — first-time scoring setup
- `manage-scoring` — tune rules and assignments
- `qualify-inbound` — single-lead version of this flow
