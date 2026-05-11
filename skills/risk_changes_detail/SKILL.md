---
name: risk-changes-detail
description: >-
  For a single company and fiscal year, surface a categorized summary of which
  risk factor themes were added in the current 10-K and which were removed
  versus the prior year — and, within each category, highlight the most
  notable items so subtle, company-specific changes are not lost in the
  category aggregation. Calls the SP_RISK_FACTOR_DIFF stored procedure, parses
  its bullet list, clusters items into thematic categories, and presents
  category counts, shares, and 1–2 notable quoted items per category. Use as a
  follow-up to the changers-statistics skill, once a portfolio manager has
  identified a high-change company. Triggers: "what changed for company X",
  "show me the added or removed risks for {company}", "what kinds of risks
  changed for {company} in {year}".
---

## Objective

The portfolio manager wants a **categorized, navigable summary** of risk-section changes — not the raw exhaustive list. A full 10-K diff may run to 20+ added and 15+ removed risks; presenting all of them as flat bullets is too noisy to act on.

But aggregate counts alone hide the signal. Two companies can both add "4 AI risks (33%)" while one is bolting on generic AI boilerplate and the other is disclosing something specific — a named regulator, a product line, a contractual exposure, an unusual framing. The role of the **notable items** column is to surface those subtle, company-specific or unexpected formulations within each category, so the PM can decide whether to drill in without reading the full bullet list.

The stored procedure `QRSLLM_POC_DB.HENRY_SCHEMA.SP_RISK_FACTOR_DIFF` does the heavy lift: it reads both filing texts inside Snowflake and produces the raw added/removed bullets. This skill's job is the second pass — parse, categorize, highlight.

---

## 🎯 Execution Procedure — Follow Exactly

### Step 1 — Call the stored procedure

```sql
CALL QRSLLM_POC_DB.HENRY_SCHEMA.SP_RISK_FACTOR_DIFF({COMPANY_ID}, {YEAR});
```

The procedure returns a single VARCHAR containing markdown with two bullet lists — `#### Risks Removed` and `#### Risks Added` — followed by a `#### Brief Narration`. Do not pass a third argument; the default `claude-haiku-4-5` is the sanctioned model.

Failure handling:
- No-rows exception → report `No 10-K filing found for company {COMPANY_ID} with fiscal year {YEAR}.` Stop.
- NULL or empty return → report the failure verbatim. Stop. Do not retry or fabricate.

### Step 2 — Parse the bullets

From the returned markdown, extract two lists of `(heading, paraphrase)` tuples:
- All bullets under `#### Risks Removed`
- All bullets under `#### Risks Added`

If a section contains the sentinel `No risks were removed compared to the prior-year filing.` (or its added-side equivalent), record that side as empty and skip categorization for it.

Discard the `#### Brief Narration` paragraph — it is replaced by a tighter one-line "Net shift" in Step 4.

### Step 3 — Categorize each side independently

Cluster the bullets into **3–7 thematic categories per side**. Added and removed are categorized independently.

**Category taxonomy.** No fixed list. Use thematic labels that emerge from the actual content. Common labels (illustrative, not prescriptive):

- Cybersecurity & Data Privacy
- AI & Emerging Technology
- Regulatory & Legal
- Macroeconomic & Financial Markets
- Geopolitical & Trade
- Supply Chain & Operations
- Human Capital & Labor
- Climate, ESG & Environmental
- Competition & Market Structure
- Capital Structure & Liquidity
- Product, Technology & R&D
- Intellectual Property
- Tax
- M&A & Integration
- Health & Pandemic

Use sector-appropriate labels — what counts as "operational" for a bank differs from what counts as "operational" for a pharma. Trust the source text; do not force a generic label when a sector-specific one is clearer (e.g. "Clinical Trial Risk" for a biotech, "Loan Loss Provisioning" for a lender).

**Granularity rules:**

- Target 3–7 categories per side. Fewer than 3 buries the signal; more than 7 reproduces the noise the categorization is meant to remove.
- A standalone singleton category is acceptable when the theme is genuinely distinct. Merge tail singletons into an "Other" bucket only when needed to keep the total ≤ 7.
- Each risk is assigned to **exactly one** category — no double-counting.

### Step 4 — Select notable items per category

Within each category, pick **1–2 items to highlight as quoted bullets**. These are the items the PM should look at if they only had time for one or two per theme.

**Selection criteria, in priority order:**

1. **Company-specific** beats boilerplate. A bullet that names a specific product, customer, regulator, geography, or business line is more interesting than a generic statement of the risk.
2. **Unexpected framing** within the theme. The subtle items — a named entity, a quantitative threshold, an unusual concession, a forward commitment — beat the obvious headline risk every issuer in the sector is adding.
3. **Departures from sector boilerplate**. If a bullet reads identically to what 10-Ks across the sector would say, it is *not* notable. The notable items are the ones that distinguish *this* company's filing.
4. When a category has only 1 item, use that 1. When it has only 2, both qualify.

**Quote rules:**

- Quotes are drawn from the procedure's paraphrased bullets — typically the heading or a tight excerpt of the paraphrase. They are **not** verbatim 10-K text and should not be presented as such.
- Each quote ≤ 15 words. Trim with `…` to fit; do not rewrite the wording to compress it (rewriting compounds paraphrase drift).
- Every quote must trace back to a specific bullet in the procedure output. Do not invent or merge bullets.

### Step 5 — Render the output

Use the format in **Output Format** below. Compute `Share = Count / total_on_that_side`, rounded to whole percent. Order categories by Count descending within each side. End with a single "Net shift" sentence that names the dominant added theme and the dominant removed theme.

---

## Output Format

Render as inline markdown (not a file):

```
### Risk Section Changes — Company {COMPANY_ID}, Fiscal Year {YEAR}

**Added (n={total_added})**

- **{Category 1} — {n} ({pct}%)**
  - "{notable quote 1, ≤ 15 words}"
  - "{notable quote 2, ≤ 15 words}"  *(omit if only one item is notable)*
- **{Category 2} — {n} ({pct}%)**
  - "{notable quote}"
- …

**Removed (n={total_removed})**

- **{Category 1} — {n} ({pct}%)**
  - "{notable quote}"
- …

**Net shift:** {one sentence — dominant added theme vs dominant removed theme}.

*Quotes are paraphrased excerpts from the procedure's diff output, not verbatim 10-K text.*
```

If a side is empty, replace its block with a single line:
`**Added:** none — no new risks compared to the prior-year filing.`
(Same convention on the removed side.)

If a company name is already established in the surrounding chat context (e.g. the PM just selected a row from a changers-statistics leaderboard), use it in the header in place of `Company {COMPANY_ID}`. Do **not** make a separate query to look it up.

---

## ❌ Do Not

1. Do **not** present the raw bullet list returned by the procedure. The purpose of this skill is to compress that list — surfacing it defeats the point. If the user explicitly asks for the raw list afterwards, it can be shown then.
2. Do **not** fabricate quotes or merge two bullets into one composite quote. Every quote must trace verbatim (or with `…` trimming) to a single bullet in the procedure output.
3. Do **not** present the quotes as verbatim 10-K text — they are paraphrases from the procedure's `AI_COMPLETE` pass. The footer line in the output format makes this explicit; keep it.
4. Do **not** use more than 7 categories per side, or fewer than 3 (unless the side has fewer than 3 items). Do **not** show more than 2 notable items per category.
5. Do **not** double-count. Each risk goes in exactly one category.
6. Do **not** pick the most boilerplate item as the notable item — the column has no value if it surfaces what everyone is saying. When all items in a category are boilerplate, pick the one with the most company-specific anchor (named regulation, named product, named geography, named counterparty).
7. Do **not** query the underlying view, re-fetch filing texts, or enrich with metadata (cosine similarity, token counts, etc.). The procedure is the only sanctioned access path.
8. Do **not** retry the procedure with a different model unless the user explicitly asks. If the call fails, surface the error verbatim.
9. Do **not** infer materiality, directionality, or trade ideas — describe the shift, do not prescribe action.
10. Do **not** narrate beyond the single "Net shift" sentence. The categories and quotes are the answer; prose is decoration.

---

## Source

- **Procedure:** `QRSLLM_POC_DB.HENRY_SCHEMA.SP_RISK_FACTOR_DIFF`
- **Signature:** `(COMPANY_ID NUMBER(38,0), FISCAL_YEAR NUMBER(38,0), MODEL_NAME VARCHAR DEFAULT 'claude-haiku-4-5')`
- **Returns:** `VARCHAR` — pre-formatted markdown with bullet lists. A no-rows match raises an exception.
- **Underlying view (do not query directly):** `QRSLLM_POC_DB.HENRY_SCHEMA.master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned`. Grain: one row per `(companyid, next_perioddate)`.

---

## Inputs

| Placeholder    | Type         | Description                                                                                                  |
|----------------|--------------|--------------------------------------------------------------------------------------------------------------|
| `{COMPANY_ID}` | NUMBER(38,0) | The S&P Capital IQ company ID. Typically from the `changers-statistics` output.                              |
| `{YEAR}`       | NUMBER(38,0) | Fiscal year of the **current** 10-K (the year of `next_perioddate`). The prior year is implicit in the view. |
