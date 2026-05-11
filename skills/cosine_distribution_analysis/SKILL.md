---
name: Cosine Similarity Distribution Analysis Skill
description: Analyzes the distribution of LM cosine similarity scores across pre-defined buckets for S&P 500 10-K/10-Q risk factor filings, filtered by fiscal year. Execution is a single CALL to a deployed Snowflake stored procedure that returns a deterministic, bucket-by-bucket aggregation; the skill renders that result set as a table.
---

## Objective
Returns a flat, tabular result set (similarity buckets × fiscal year × filing counts) for the Lazy-Prices–style YoY risk-factor change analysis. All bucket boundaries, year filtering, projection, and ordering are encapsulated inside the deployed stored procedure `QRSLLM_POC_DB.HENRY_SCHEMA.sp_lm_similarity_distribution`, so every invocation produces an identical schema and identical bucket logic regardless of caller.

---

## 🎯 EXECUTION PROCEDURE — FOLLOW EXACTLY

### Step 1 — Call the stored procedure
Execute exactly:

```sql
CALL QRSLLM_POC_DB.HENRY_SCHEMA.sp_lm_similarity_distribution({YEAR});
```

Substitute `{YEAR}` with the four-digit fiscal year as an integer literal (e.g. `2024`). Do not wrap the call in a `SELECT`, do not add a `LIMIT`, and do not pass additional arguments — the procedure signature accepts exactly one `INTEGER`.

### Step 2 — Display the data in a table
Render the returned result set using the column order and types in the *Output Table Form* below. The procedure returns Snowflake-default UPPERCASE identifiers — preserve them. Do not rename, re-case, or post-process columns.

### Step 3 — Narrate in ≤ 4 sentences
State (i) total filing count for the year, (ii) the shape of the distribution (where mass concentrates), and (iii) any interesting observations across buckets. Do not enumerate every bucket.

---

## Procedure Contract

- **Fully qualified name:** `QRSLLM_POC_DB.HENRY_SCHEMA.sp_lm_similarity_distribution`
- **Input:** `YEAR_INPUT INTEGER` — single fiscal year, e.g. `2024`.
- **Output:** seven-column table, one row per non-`'other'` bucket present in the data for that year, ordered by `BUCKET_ORDER ASC`.
- **Baked-in filters:** `lm_cosine_similarity IS NOT NULL`, `next_perioddate <= CURRENT_DATE`, `similarity_bucket <> 'other'`.
- **Bucket grid:** eight ordered buckets from `below 0.90` (BUCKET_ORDER = 1) to `0.999-1.0` (BUCKET_ORDER = 8). The synthetic `'other'` bucket (floating-point artifacts above 1.0) is filtered out server-side before return.

---

## Output Table Form

| Column              | Type    | Description                                            |
|---------------------|---------|--------------------------------------------------------|
| `SIMILARITY_BUCKET` | STRING  | Ordered bucket label, e.g. `'0.99-0.995'`              |
| `BUCKET_ORDER`      | INT     | Integer 1–8 for deterministic ordinal sorting          |
| `FISCAL_YEAR`       | INT     | Year extracted from `next_perioddate`                  |
| `FILING_COUNT`      | INT     | Filings in this (bucket × year) cell                   |
| `MIN_SIMILARITY`    | FLOAT   | Min cosine similarity in cell                          |
| `MAX_SIMILARITY`    | FLOAT   | Max cosine similarity in cell                          |
| `AVG_SIMILARITY`    | FLOAT   | Mean cosine similarity in cell                         |

---

## ❌ DO NOT

1. **Do not re-create, alter, or replace the procedure** from within this skill. The procedure is managed and deployed out-of-band; the skill is a pure consumer.
2. **Do not inline the bucket-logic SQL** as a substitute for the `CALL`. The point of the procedure is that buckets and filters are defined once, server-side — ad-hoc SQL defeats reproducibility.
3. **Do not append a `LIMIT`** to the `CALL`. The procedure already returns at most 8 rows per year.
4. **Do not lowercase or quote-wrap column names** in display. Preserve Snowflake-default UPPERCASE identifiers.
5. **Do not order by `SIMILARITY_BUCKET`** in any downstream rendering. The labels sort incorrectly as strings (e.g. `'0.999-1.0'` precedes `'0.99-0.995'`). The procedure already orders by `BUCKET_ORDER ASC` — keep that order.

---

## Source Table (reference only)

- **Database:** `QRSLLM_POC_DB`
- **Schema:** `HENRY_SCHEMA`
- **Table:** `master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned`
- **Required columns:** `next_perioddate` (DATE), `lm_cosine_similarity` (FLOAT)

---
