---
name: changers-statistics
description: Flag the S&P 500 companies whose 10-K risk factor section changed the most year-over-year, ranked by semantic similarity between the prior and current filing. Use when a portfolio manager asks "which companies changed their risk factors this year", "biggest risk section changes", "10-K risk factor changers", or any variant referencing material/largest/biggest year-over-year shifts in 10-K risk language. Returns a fixed-size leaderboard (lowest cosine similarity = largest change), not a distribution.
---

## Objective

The audience is a portfolio manager. The job is to surface S&P 500 companies whose 10-K risk factor section changed substantially year-over-year, so the PM can read the underlying text and decide whether the change is material to their position.

Execution is a single `CALL` to the deployed Snowflake stored procedure `QRSLLM_POC_DB.HENRY_SCHEMA.SP_YOY_RISK_CHANGERS`. The procedure encapsulates the filters, projection, ordering, and the 20-row limit, so every invocation returns an identical schema and identical ranking logic regardless of caller.

---

## 🎯 Execution Procedure — Follow Exactly

### Step 1 — Call the stored procedure
Execute exactly:

```sql
CALL QRSLLM_POC_DB.HENRY_SCHEMA.SP_YOY_RISK_CHANGERS({YEAR});
```

Substitute `{YEAR}` with the four-digit fiscal year as an integer literal (e.g. `2024`). Do not wrap the call in a `SELECT`, do not add a `LIMIT`, and do not pass additional arguments — the procedure signature accepts exactly one `INTEGER`.

### Step 2 — Render the result as a table
Use the column mapping in the **Output Table Form** below. Format `LM_COSINE_SIMILARITY` to 2 decimal places. Format `RISK_ADDED_REMOVED_FLAG` values as the human-readable strings in the mapping table (e.g. `RISK_ADDED` → "Risk Added"). Preserve the row order returned by the procedure — it is already sorted ascending by similarity with `COMPANYID` as deterministic tie-break.

### Step 3 — If the procedure returns fewer than 20 rows
Return whatever rows were produced and state the actual count in one short line above the table. Do **not** silently fall back to a different year or relax any filter.

---

## Procedure Contract

- **Fully qualified name:** `QRSLLM_POC_DB.HENRY_SCHEMA.SP_YOY_RISK_CHANGERS`
- **Input:** `YEAR_INPUT INTEGER` — single fiscal year, e.g. `2024`.
- **Output:** nine-column table, at most 20 rows, ordered by `LM_COSINE_SIMILARITY ASC, COMPANYID ASC`.
- **Grain:** one row per `(companyid, next_perioddate)` — i.e. one row per company per fiscal-year 10-K.
- **Baked-in filters:** `DATE_PART('year', next_perioddate) = YEAR_INPUT`, `lm_cosine_similarity IS NOT NULL`, `next_perioddate <= CURRENT_DATE`.
- **Baked-in limit:** `LIMIT 20`.

---

## Output Table Form

| Display column     | Source column                    | Type   | Notes                                                                                          |
|--------------------|----------------------------------|--------|------------------------------------------------------------------------------------------------|
| Company            | `COMPANYNAME`                    | STRING |                                                                                                |
| Company ID         | `COMPANYID`                      | INT    |                                                                                                |
| Sector             | `SECTOR`                         | STRING | GICS sector                                                                                    |
| Cosine Similarity  | `LM_COSINE_SIMILARITY`           | FLOAT  | Format to 2 decimals. Lower = larger change.                                                   |
| Filing Date        | `NEXT_FILINGDATE`                | DATE   | Date the current-year 10-K was filed.                                                          |
| Period End         | `NEXT_PERIODDATE`                | DATE   | Fiscal period end date of the current-year 10-K.                                               |
| Prev Tokens        | `PREV_FILING_TEXT_TOKEN_COUNT`   | INT    | Token count of prior-year risk section.                                                        |
| Curr Tokens        | `NEXT_FILING_TEXT_TOKEN_COUNT`   | INT    | Token count of current-year risk section.                                                      |
| Risk Change Status | `RISK_ADDED_REMOVED_FLAG`        | STRING | Map: `RISK_ADDED` → "Risk Added", `RISK_REMOVED` → "Risk Removed", `UNCHANGED` → "Unchanged".  |

---

## ❌ Do Not

1. Do **not** re-create, alter, or replace the procedure from within this skill. The procedure is managed and deployed out-of-band; the skill is a pure consumer.
2. Do **not** inline the ranking SQL as a substitute for the `CALL`. The point of the procedure is that filters, ordering, and the row limit are defined once, server-side.
3. Do **not** apply additional filters (sector, market cap, token-count thresholds, etc.) unless the user explicitly asks. The skill returns the unconditional top-20 by similarity for the requested year.
4. Do **not** append a `LIMIT` to the `CALL`. The procedure already caps the result at 20 rows.
5. Do **not** lowercase or quote-wrap column names. Preserve Snowflake-default UPPERCASE identifiers.
6. Do **not** re-sort the result client-side. The procedure already orders by `LM_COSINE_SIMILARITY ASC, COMPANYID ASC` — keep that order.
7. Do **not** interpret a low cosine similarity as a directional signal (bullish/bearish). It only flags magnitude of change.
8. Do **not** silently fall back to a different year if `{YEAR}` returns fewer than 20 rows — return whatever rows match and say so in one line.
9. Do **not** output anything other than the required table (and the one-line short-row notice if applicable). Keep it clean and professional.
