---
name: find-expansion-accounts
description: >
  Identify existing customers showing buying signals for upsell or cross-sell using
  a dedicated expansion scoring profile. Falls back to manual signal counting when
  the Saber CLI isn't available.
---

# Find Expansion Accounts

Use this skill to surface upsell and cross-sell opportunities within your customer base. The right setup is a **separate scoring profile** tuned for expansion (different signals, different fit definition than new business), assigned to your customer list.

For scoring concepts, see [`_shared/scoring.md`](../_shared/scoring.md).

## Step 1 — Get the customer list

Ask the user how their customer data is available:

### Path A — Saber CLI

```bash
saber list company list
```

Ask which list contains current customers. If a customer list doesn't exist yet, offer to create one (a manually-imported list is fine — `saber list company import` or filter by a HubSpot lifecycle property).

### Path B — HubSpot MCP (if available)

Pull customers via the HubSpot MCP:
```
Fetch HubSpot companies where lifecycle_stage = "customer"
```

If the Saber CLI is also available, import the result into a Saber list to use Path A. Otherwise work with HubSpot data directly under Path C.

### Path C — Manual list

Ask for a list of customer domains. Work with that list directly under the manual fallback below.

---

## Path A — Expansion scoring profile (Saber CLI)

### A1. Check for an existing expansion profile

```bash
saber scoring profile list
```

Look for a `type: company` profile clearly named for expansion (e.g. `Expansion`, `Upsell`, `Existing customers`). If one exists, jump to A4. Otherwise create one.

### A2. Create the expansion profile

```bash
saber scoring profile create --type company \
  --name "Expansion (company)" \
  --description "Upsell / cross-sell signals against existing customers"
```

Capture the `profileId`.

### A3. Add expansion rules

For each expansion signal in scope, upsert a rule. Default category → dimension mapping:

| Signal type | Dimension | Reasoning |
|---|---|---|
| **Growth** — hiring, funding, geographic expansion | `urgency` | Time-sensitive triggers |
| **Problem** — they have the pain the expansion product solves | `fit` | Persistent need, not a moment in time |
| **Intent** — evaluating an adjacent or complementary tool | `urgency` | Active buying window |
| **At-risk** — layoffs, original buyer churned, declining usage | `urgency` (negative points) | Retention flag, not expansion |

If signal templates already exist (from `create-company-signals`), reuse them. Otherwise create them first via `create-company-signals` and return.

```bash
# Growth example (boolean)
saber scoring rule upsert <profileId> \
  --signal-template <hiringSigId> --dimension urgency \
  --answer-type boolean --true 20 --false 0

# At-risk example — negative points
saber scoring rule upsert <profileId> \
  --signal-template <layoffsSigId> --dimension urgency \
  --answer-type boolean --true -30 --false 0
```

For unfamiliar signal templates, see [`_shared/scoring.md`](../_shared/scoring.md) for the answer-type → point-values shape contract. For the full first-time setup walkthrough, use `configure-scoring` instead of doing it inline here.

### A4. Bulk-assign the customer list

```bash
saber list company companies <customerListId>
# extract domains, then:
saber scoring assignment bulk --profile <profileId> --type company \
  --object acme.com --object stripe.com ...
```

Compute kicks off automatically; auto-trigger keeps scores fresh as expansion signals run.

### A5. Read and bucket

```bash
saber scoring scores --type company \
  --object acme.com --object stripe.com [...] --detailed
```

Bucket the customer list into:

| Bucket | Definition | Action |
|---|---|---|
| **Expand now** | `urgency ≥ 60 AND fit ≥ 50` | Reach out — reference the strongest growth/intent signal |
| **Educate** | `fit ≥ 60 AND urgency < 60` | Persistent need but no live trigger — content / EBR / proactive value-prop conversation |
| **At risk** | Any rule contributing strongly negative points to `urgency` | Flag to AM — proactive check-in, not expansion outreach |
| **Stable** | None of the above fire | Monitor only |

Present:

```
## Expansion Opportunities — [Customer List Name]

### Expand now ([N] accounts)
| Company | Domain | Fit | Urgency | Top trigger |
|---|---|---|---|---|
| Acme Corp | acme.com | 78 | 84 | Series B 2 months ago, hiring 6 SDRs |

### Educate ([N] accounts)
| Company | Domain | Fit | Urgency | Persistent pain |
|---|---|---|---|---|
| Beta Inc | beta.io | 72 | 32 | Manual reporting still |

### At risk ([N] accounts)
| Company | Domain | Risk signal |
|---|---|---|
| Gamma Ltd | gamma.io | Layoffs announced last month |

### Stable ([N] accounts)
[count only — list omitted unless requested]
```

---

## Path B / C — Manual expansion review (no Saber CLI)

Used when the CLI isn't available. Less calibrated than Path A but still useful.

### B1. Define expansion signals

Work with the user on 3–5 questions covering:

- **Growth** — "Is this customer actively hiring in roles that would use our product?", "Have they raised funding in the last 6 months?"
- **Problem** — "Are they posting about [the challenge our expansion product addresses]?"
- **Intent** — "Are they evaluating [complementary tool]?"
- **At-risk** — "Are they reducing headcount?", "Is the original buyer still in role?"

### B2. Research per customer

For each customer × each question, research using web search, LinkedIn, HubSpot MCP notes/properties, news.

### B3. Bucket

Use the same buckets as Path A. Flag any account with at-risk evidence regardless of expansion signals.

```
| Company | Growth | Problem | Intent | At-risk | Bucket |
|---|---|---|---|---|---|
| Acme | Yes | — | Yes | — | Expand now |
| Gamma | — | — | — | Yes | At risk |
```

---

## Step 3 — Hand off

- **Expand now:** use `write-outreach` with expansion-focused messaging — reference the specific growth or intent signal as the reason for reaching out
- **Educate:** schedule a value-prop conversation; light-touch nurture; revisit when an urgency signal fires
- **At risk:** flag for the AM team for a retention check-in; consider `deal-coaching` for any open renewal
- **Stable:** monitor — auto-trigger will surface them in the next bucket if signals change

## Related

- [`_shared/scoring.md`](../_shared/scoring.md) — scoring concepts
- `configure-scoring` — full first-time profile setup
- `manage-scoring` — tune the expansion profile after seeing scores
- `score-accounts` — generic ranking version of this skill
