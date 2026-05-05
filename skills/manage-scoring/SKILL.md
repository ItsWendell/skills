---
name: manage-scoring
description: >
  View, edit, and operate scoring — list profiles and rules, tune point values,
  add or remove assignments, trigger recompute, and inspect scores with
  contribution breakdowns. The day-two counterpart to configure-scoring.
---

# Manage Scoring

Use this skill to inspect and adjust scoring after the initial `configure-scoring` setup. For concept definitions (dimensions, profiles, rules, assignments, point-value shapes, auto-trigger behaviour), see [`_shared/scoring.md`](../_shared/scoring.md).

For signal subscriptions (the upstream that feeds scores), use `manage-signals`.

## Saber CLI check

Run `saber --help`. Scoring requires the CLI — if it isn't installed, point the user to saber.app and stop.

## Step 1 — Inventory

List profiles first:

```bash
saber scoring profile list
```

Show in a table grouped by `type`:

| ID | Type | Name | Description |
|---|---|---|---|
| ... | company | EMEA Enterprise | New-business outbound |
| ... | contact | Decision makers | Senior engineering buyers |

Ask which profile the user wants to work with. Then drill in:

```bash
saber scoring rule list <profileId>
```

Show each rule with: signal template, dimension, point-values shape preview.

## Step 2 — Choose an action

Present the menu:

1. **Inspect scores** for one or more objects
2. **Edit a rule** (change point values)
3. **Delete a rule** (remove a signal from the profile)
4. **Add a rule** (cover a new signal template)
5. **Manage assignments** (add / remove / list)
6. **Recompute manually** for specific objects
7. **Delete the profile** (cascades to all its rules, assignments, scores)

## Step 3 — Execute

### 1. Inspect scores

```bash
saber scoring scores --type company --object <domain> --detailed
saber scoring scores --type company --object <domain1> --object <domain2>
```

`--detailed` returns per-rule contributions plus the previous score (Δ). Use it whenever you want to debug *why* an object scores the way it does.

### 2. Edit a rule (change point values)

Upsert is keyed on `(profileId, signalTemplateId, dimension)` — calling it again replaces the rule. **Every upsert triggers a recompute for every assigned object.**

```bash
saber scoring rule upsert <profileId> \
  --signal-template <signalTemplateId> \
  --dimension fit \
  --answer-type boolean \
  --true 25 --false -10
```

Confirm with the user before running — the recompute can be expensive on large profiles.

### 3. Delete a rule

```bash
saber scoring rule delete <profileId> <ruleId>
```

Also triggers a recompute.

### 4. Add a rule

Same as Step 4 in `configure-scoring` — pick a signal template, dimension, answer type, point values. See [`_shared/scoring.md`](../_shared/scoring.md) for the answer-type → shape contract.

### 5. Manage assignments

```bash
# What profiles is one object assigned to?
saber scoring assignment list --type company --object acme.com

# Add a single object
saber scoring assignment create --profile <id> --type company --object acme.com

# Add many at once
saber scoring assignment bulk --profile <id> --type company \
  --object acme.com --object stripe.com --object ramp.com

# Remove (clears that assignment's scores too)
saber scoring assignment delete <assignmentId>
```

Use bulk for full-list assignments — duplicates are skipped silently.

### 6. Recompute manually

Auto-trigger usually keeps scores fresh. Use `compute` only when:
- You just edited rules and want immediate confirmation on a specific object
- Underlying signal data was repaired out-of-band
- You want to force a refresh ahead of a meeting

```bash
saber scoring compute --type company --object acme.com --object stripe.com
```

Returns `{queued, failed}`. `failed > 0` means some object dispatches errored — retry to pick those up. Compute is idempotent: duplicate triggers attach to the running workflow.

### 7. Delete the profile

Cascades to all rules, assignments, and scores under it. Confirm explicitly first:

> "Deleting profile [name] will remove [N] rules, [M] assignments, and all historical scores under it. This is not reversible. Continue?"

```bash
saber scoring profile delete <profileId>
```

## Useful checks

- **Coverage gap** — if `signalCoverage` is much lower than `totalRules` in a score, the underlying signals haven't run yet for that object. Check `manage-signals` to make sure subscriptions cover this list.
- **Stale scores** — `computedAt` more than a few hours after the latest signal completion suggests auto-trigger may have failed. Run `compute` manually to recover.
- **Identical scores across many objects** — usually a rule with too-coarse point values. Use `--detailed` on a few objects and look at which rules actually differentiate.

## Related

- `configure-scoring` — first-time setup
- `score-accounts` — rank a list by current scores
- `manage-signals` — manage the signal subscriptions that feed scores
- `_shared/scoring.md` — reference for all concepts and CLI shapes
