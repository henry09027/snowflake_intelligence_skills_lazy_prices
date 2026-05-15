---
name: risk-changes-detail
description: >-
  For a single company and fiscal year, surface a categorized summary of which
  risk factor themes were added and removed in the current 10-K versus the
  prior year, plus a table of the top subtle wording changes within risks that
  appear in both filings. Calls SP_RISK_FACTOR_DIFF, parses its three
  sections, and renders Added / Removed / Subtle Changes for portfolio-manager
  review. Use as a follow-up to changers-statistics once a high-change company
  has been identified. Triggers: "what changed for company X", "show me the
  added or removed risks for {company}", "what kinds of risks changed for
  {company} in {year}".
---

## Objective

The PM wants a navigable summary of risk-section changes, not the raw diff. Three things matter:

1. **What was added** — new risks that did not exist in the prior filing, summarized as themes.
2. **What was removed** — risks that have dropped out, summarized as themes.
3. **What changed subtly** — risks present in both years where management edited the wording. Per the Lazy Prices literature, these subtle edits are often the most informative signal.

The procedure does the diff. This skill parses the output and renders it for the PM.

---

## Procedure

### 1. Call the stored procedure

```sql
CALL QRSLLM_POC_DB.LAZY_PRICES_DECLARATIVE_SHARE_BETA.SP_RISK_FACTOR_DIFF({COMPANY_ID}, {YEAR});
```

Do not pass a third argument — the default `claude-haiku-4-5` is the sanctioned model.

**Failure handling:**
- No-rows exception → report `No 10-K filing found for company {COMPANY_ID} with fiscal year {YEAR}.` Stop.
- NULL or empty return → report the failure verbatim. Stop. Do not retry or fabricate.

### 2. Parse the three sections

The procedure returns markdown with three sections:

- `#### Risks Removed` — bullet list of `(heading, key sentence)`
- `#### Risks Added` — bullet list of `(heading, key sentence)`
- `#### Top Subtle Changes in Existing Risks` — up to 5 bullets, each with prior phrase, current phrase, and rationale

If a section contains its sentinel placeholder (e.g. `No new risks were added compared to the prior-year filing.`), record that section as empty.

### 3. Categorize Added and Removed

For Added and Removed only, cluster bullets into **3–7 thematic categories per side**.

Use sector-appropriate labels — what counts as "operational" for a bank differs from a pharma. Common labels (illustrative, not prescriptive): Cybersecurity & Data Privacy, AI & Emerging Technology, Regulatory & Legal, Macro & Financial Markets, Geopolitical & Trade, Supply Chain & Operations, Human Capital, Climate & ESG, Competition, Capital Structure, Product & R&D, IP, Tax, M&A, Health.

Rules:
- 3–7 categories per side. Merge tail singletons into "Other" only if needed to stay ≤ 7.
- Each risk goes in exactly one category.
- Order categories by count descending.

### 4. Pick a representative key sentence per category

For each category, pick **one** key sentence that best represents the theme. Prefer the bullet that is most company-specific (names a product, regulator, geography, or counterparty) over generic boilerplate. Trim to ≤ 20 words with `…` if needed; do not rewrite.

### 5. Render the subtle changes as a table

For Subtle Changes, do not categorize. Render the (up to 5) bullets directly as a markdown table with four rows: **Risk Heading**, **Prior Year**, **Current Year**, **Why it matters**.

In the Prior Year and Current Year rows, **bold the specific words that changed** using `**…**`. Apply bolding only to the tokens that differ between the two excerpts — not the whole phrase. Keep each excerpt ≤ 25 words; trim with `…`.

---

## Output Format

Render inline markdown:

```
### Risk Section Changes — Company {COMPANY_ID}, Fiscal Year {YEAR}

#### Added (n={total_added})
- **{Category} — {n} ({pct}%)** — "{representative key sentence}"
- …

#### Removed (n={total_removed})
- **{Category} — {n} ({pct}%)** — "{representative key sentence}"
- …

#### Subtle Changes in Existing Risks

| Risk Heading | {heading} |---|---|
| Prior Year | …**{old phrase}**… |---|---|
| Current Year | …**{new phrase}**… |---|---|
| Why it matters | {one-sentence rationale} |---|---|

**Net shift:** {one sentence — dominant added theme vs dominant removed theme}.
```

If Added or Removed is empty, replace its section with:
`#### Added: none — no new risks compared to the prior-year filing.`
(Same convention for Removed.)

If Subtle Changes is empty:
`#### Subtle Changes: none material.`

If a company name is already established in the surrounding chat (e.g. selected from a changers-statistics leaderboard), use it in the header in place of `Company {COMPANY_ID}`. Do not make a separate lookup query.

---

## Do Not

1. Do **not** present the raw bullet list from the procedure — categorize and compress. If the PM asks for raw output afterwards, then show it.
2. Do **not** fabricate quotes or merge bullets. Every key sentence and every table row traces to exactly one bullet in the procedure output.
3. Do **not** bold entire excerpts in the Subtle Changes table — bold only the differing tokens. If the bolding covers more than half the cell, you are over-bolding.
4. Do **not** categorize Subtle Changes. The table is the entire treatment.
5. Do **not** exceed 7 categories per side or 5 cases in the subtle changes table.
6. Do **not** query the underlying view, re-fetch filing texts, or enrich with cosine similarity or token counts. The procedure is the only sanctioned access path.
7. Do **not** retry the procedure with a different model unless the PM explicitly asks.

---

## Inputs

| Placeholder    | Type         | Description                                                                                                  |
|----------------|--------------|--------------------------------------------------------------------------------------------------------------|
| `{COMPANY_ID}` | NUMBER(38,0) | The S&P Capital IQ company ID. Typically from the `changers-statistics` output.                              |
| `{YEAR}`       | NUMBER(38,0) | Fiscal year of the **current** 10-K (the year of `next_perioddate`). The prior year is implicit in the view. |

---
