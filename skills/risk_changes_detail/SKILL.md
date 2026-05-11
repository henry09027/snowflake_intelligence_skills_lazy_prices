---
name: risk-changes-detail
description: >-
  For a single company and fiscal year, surface a CATEGORIZED summary of which
  risk factor themes were added in the current 10-K and which were removed
  versus the prior year. Calls the SP_RISK_FACTOR_DIFF stored procedure, parses
  its bullet list, clusters items into thematic categories, and presents
  category counts and shares (not the raw bullets). Use as a follow-up to the
  changers-statistics skill, once a portfolio manager has identified a
  high-change company. Triggers: "what changed for company X", "show me the
  added or removed risks for {company}", "what kinds of risks changed for
  {company} in {year}".
---

## Objective

The portfolio manager wants a **categorized, navigable summary** of risk-section changes — not the raw exhaustive list. A full 10-K diff may run to 20+ added and 15+ removed risks; presenting all of them as flat bullets is too noisy to act on. This skill clusters each side into thematic categories and reports count and share per category, so the PM can see at a glance whether the changes are concentrated (e.g. AI and regulatory exposure) or spread evenly across the book.

The stored procedure `QRSLLM_POC_DB.HENRY_SCHEMA.SP_RISK_FACTOR_DIFF` does the heavy lift: it reads both filing texts inside Snowflake and produces the raw added/removed bullets. This skill's job is the second pass — parse, categorize, summarize.

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

If a section contains the sentinel `No risks were removed compared to the prior-year filing.` (or its added-side equivalent), record that side as empty and skip categorization for it. Do not pad an empty side with manufactured categories.

Discard the `#### Brief Narration` paragraph — it will be replaced by a tighter narration of your own in Step 4.

### Step 3 — Categorize each side independently

Cluster the bullets into **3–7 thematic categories per side**. The added and removed sides are categorized independently — they need not share a taxonomy.

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
- Each risk is assigned to **exactly one** category — no double-counting. When a risk plausibly fits two, place it in the category that better captures its primary driver.

### Step 4 — Render the categorized output

Use the format in **Output Format** below. Compute `Share = Count / total_on_that_side`, rounded to whole percent. Order categories by Count descending within each side. End with a single "Net shift" sentence that names the dominant added theme and the dominant removed theme — no further commentary.

---

## Output Format

Render as inline markdown (not a file):

```
### Risk Section Changes — Company {COMPANY_ID}, Fiscal Year {YEAR}

**Added (n={total_added})**

| Category | Count | Share | Examples |
|---|---:|---:|---|
| {Category 1} | {n} | {pct}% | {2–3 short heading fragments, comma-separated} |
| {Category 2} | {n} | {pct}% | … |

**Removed (n={total_removed})**

| Category | Count | Share | Examples |
|---|---:|---:|---|
| {Category 1} | {n} | {pct}% | … |

**Net shift:** {one sentence — dominant added theme vs dominant removed theme}.
```

**Examples-column style** (form, not content): `genAI model bias, third-party LLM dependency, AI hiring discrimination` — short fragments separated by commas, each ≤ 6 words, never full sentences. Their job is to make the category concrete at a glance, not to summarize each underlying bullet.

If a side is empty, replace its table with a single line:
`**Added:** none — no new risks compared to the prior-year filing.`
(Same convention on the removed side.)

If a company name is already established in the surrounding chat context (e.g. the PM just selected a row from a changers-statistics leaderboard), use it in the header in place of `Company {COMPANY_ID}`. Do **not** make a separate query to look it up.

---

## ❌ Do Not

1. Do **not** present the raw bullet list returned by the procedure. The purpose of this skill is to compress that list — surfacing it defeats the point. If the user explicitly asks for the raw list afterwards, it can be shown then.
2. Do **not** use more than 7 categories per side, or fewer than 3 (unless the side itself has fewer than 3 items).
3. Do **not** double-count. Each risk goes in exactly one category.
4. Do **not** fabricate a category to inflate coverage. Every category must be supported by at least one bullet from the procedure output.
5. Do **not** query the underlying view, re-fetch filing texts, or enrich the output with metadata (cosine similarity, token counts, filing dates, sector). The procedure is the only sanctioned access path.
6. Do **not** retry the procedure with a different model unless the user explicitly asks. If the call fails, surface the error verbatim.
7. Do **not** infer materiality, directionality, or trade ideas — describe the shift, do not prescribe action.
8. Do **not** narrate beyond the single "Net shift" sentence. The tables are the answer; prose is decoration.

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
