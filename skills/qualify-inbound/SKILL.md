---
name: qualify-inbound
description: >
  Qualify an inbound lead by running signals against their company domain and
  reading native fit + urgency scores. Falls back to manual scoring when the
  Saber CLI or a scoring profile isn't available.
---

# Qualify Inbound

Use this skill to qualify an inbound lead fast — run the signal questions against their company, then read calibrated **fit** and **urgency** scores.

For scoring concepts, see [`_shared/scoring.md`](../_shared/scoring.md).

## Saber CLI check

Run `saber --help`. With the CLI you get native scoring — without it, the skill still works using a manual qualification fallback (Path B).

## Step 1 — Get the lead

Ask for the inbound company's domain (e.g. `acme.com`). If only a name is provided, ask for the domain or attempt to infer it.

## Step 2 — Choose the path

```bash
saber scoring profile list
saber scoring assignment list --type company --object <domain>
```

- **At least one company scoring profile assigned to this domain → Path A** (read native scores).
- **A profile exists but isn't assigned to this domain → Path A**, with one extra step: assign first.
- **No company scoring profile at all → Path A is unavailable.** Either route to `configure-scoring` (best one-time investment) or take **Path B**.
- **No CLI → Path B**.

---

## Path A — Native scoring

### A1. Check for fresh signal data

Before running new signals (which costs credits), check for recent results on this domain:

```bash
saber subscription list
```

If subscriptions covering the relevant signals have already run for the domain's list, scores already exist and you can skip straight to A3.

### A2. Run signals if needed

Show the credit balance:

```bash
saber credits
```

Tell the user: "Running [N] signals against [domain] will use [N] credits." Confirm, then run:

```bash
saber signal --domain <domain> --question "<signal question>" --answer-type boolean --no-wait
saber signal get <signalId>
```

Auto-trigger will update assigned scores as each signal completes.

### A3. Assign profile if needed

If the lead's domain isn't already assigned to the profile:

```bash
saber scoring assignment create --profile <profileId> --type company --object <domain>
```

Assignment triggers immediate compute.

### A4. Read scores

```bash
saber scoring scores --type company --object <domain> --detailed
```

You get one row per dimension (fit, urgency) with the contributing rules.

### A5. Map to inbound tiering

| Fit | Urgency | Tier | Recommended action |
|---|---|---|---|
| ≥ 70 | ≥ 50 | **Hot** | Fast-track to AE; reach out within hours |
| ≥ 70 | < 50 | **Right fit, slow timing** | Keep warm — light-touch nurture, re-check on next signal cycle |
| 40–69 | ≥ 70 | **Reactive — wrong fit, hot moment** | Discovery call to verify; could be a one-off rather than a long-term fit |
| 40–69 | < 70 | **Possible fit** | Use `research-account` before reaching out |
| < 40 | any | **Not a fit** | Polite nurture or disqualify |

If a disqualifier rule contributed strongly negative points, surface it explicitly — that's usually a hard "no fit" regardless of the rest.

### A6. Present qualification summary

```
## Qualification: [Company] ([domain])

**Tier:** Hot / Right fit, slow timing / Reactive / Possible fit / Not a fit

**Fit:** 84 (Δ +6)
- New VP Sales hired (+20)
- 51–200 employees (+15)
- HubSpot user (+10)

**Urgency:** 72 (Δ +12)
- Series B in last 6 months (+25)
- Hiring SDRs (+15)

**Recommendation:** [1–2 sentences explaining the tier and suggested next action]
```

---

## Path B — Manual qualification (fallback)

Used when there's no CLI or no scoring profile.

### B1. Load signal definitions

Check conversation context for approved signals (from `signal-discovery` or informal). If none, ask:

> "Do you have defined signal questions for your ICP? If not, run `signal-discovery` first to set them up."

### B2. Run the signals manually

Without the CLI, research each signal question against the domain — web search, LinkedIn, news, the company's own careers page.

### B3. Score

Count positive signals; weight by the user's stated priorities. Apply disqualifiers as in `generate-signals`. Map to:

- **Strong fit** — majority positive, no disqualifiers
- **Possible fit** — mixed
- **Weak fit** — few positives or a disqualifier fired
- **Not a fit** — disqualifier or zero positive

### B4. Present

```
## Qualification: [Company] ([domain])

**Score:** Strong fit / Possible fit / Weak fit / Not a fit

| Signal | Result | Notes |
|---|---|---|
| [question] | Yes / No | [context] |

**Recommendation:** [1–2 sentences]
```

---

## Step 3 — Suggest next steps

- **Hot / Strong fit:** use `write-outreach` to draft a personalised first touch
- **Right fit, slow timing:** add to nurture; re-check when the next signal cycle runs
- **Reactive / Possible fit:** use `research-account` to validate before investing AE time
- **Not a fit / Weak fit:** route to a generic nurture sequence or disqualify

## Related

- [`_shared/scoring.md`](../_shared/scoring.md) — scoring concepts and CLI reference
- `configure-scoring` — set up scoring once, qualify many
- `research-account` — deeper context for borderline leads
- `write-outreach` — turn a hot qualification into a message
