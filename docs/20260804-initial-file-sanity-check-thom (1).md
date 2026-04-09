# Initial File Sanity Check — What Thom Does First

When a client delivers a data file, Thom's first action is an **eyeball / common-sense check** before any deeper analysis. It's the cheapest, fastest form of validation and catches the majority of obvious problems before time is wasted on processing. This is the first rung of the validation ladder — the file is not trusted until it passes.

## 1. Structural checks (does the file even make sense?)

- **Row count** — is it roughly what you'd expect? 500 rows when you expected 50.000 is a red flag. 2M rows when you expected 100k is also a red flag.
- **Column count & headers** — are all expected columns present? Any renamed? Any extra columns you didn't ask for?
- **File encoding & delimiter** — UTF-8? Semicolon vs comma (Dutch exports love `;`)? BOM markers?
- **Date range** — does the period covered match what was requested? (e.g. "2024 Q1–Q4" but the file stops in October)
- **Duplicate header rows / merged cells** — common in Excel exports from financial systems

## 2. Completeness checks (is anything missing?)

- **Empty / null columns** — a column that's 100% empty usually means a broken export
- **Null rate per column** — BSN missing in 40% of rows? That's a problem
- **Gaps in time series** — missing months, missing weeks
- **Missing categories** — e.g. you expected 14 Jeugdzorg productcategorieën and only see 11

## 3. Cross-checks against known truths (the "common sense" layer)

This is the part Thom is really good at, and it's the hardest to automate because it requires domain knowledge.

- **Totals** — does total spend match the number the client mentioned in the email / previous report / begroting?
- **Headcount / client count** — does it match what's on their website, annual report, or the previous delivery?
- **Order of magnitude** — is average cost per client €8.000 or €800.000? Is it plausible for this type of care?
- **Known benchmarks** — a gemeente of ~50k inhabitants typically has X jeugdzorg clients; does this match?
- **Previous delivery comparison** — if you had last quarter's file, are the numbers within a reasonable delta, or did something jump 300%?

## 4. Internal consistency checks

- **Sum of parts = whole?** — do the product categories add up to the total?
- **P × Q = Total?** — does price × quantity actually equal the reported total cost? Rounding is fine, 15% off is not.
- **Referential integrity** — every client_id in the cost table should exist in the client table
- **Date logic** — start_date ≤ end_date, no traject ending before it started
- **Impossible values** — negative costs, ages over 120, durations longer than the reporting period

## 5. Format / type checks

- **Numbers that look like text** — `"1.234,56"` as a string instead of a float (Dutch number formatting is a classic trap)
- **Dates in mixed formats** — `01-03-2024` vs `2024-03-01` vs `1/3/24` in the same column
- **Leading zeros stripped** — postcodes, BSN, gemeentecodes (Excel loves to eat these)
- **Trailing whitespace, inconsistent casing** in category names

## 6. Red flags that mean "stop and call the client"

- Row count off by >20% from expectations
- Totals off by >5% from a known source
- A whole category/column missing
- Evidence the file was manually edited (weird formatting inconsistencies, formulas left in, hidden sheets)
- Different file structure than last delivery with no warning

---

## Why this step matters so much

Every hour spent on the eyeball check saves ten hours downstream. If you skip it and feed a broken file into your pipeline, you end up producing a beautiful analysis of garbage — and worse, the client trusts it because it *looks* polished. Thom's instinct here is exactly right: **the trust layer starts before any code runs.** It starts with a human who knows what the numbers *should* look like, comparing them to what's in the file.

## Relevance for DataPraat

This is essentially **Step 1 of Harm Nico's validation ladder**. The product has to replicate Thom's eyeballing as an automated first gate:

- row counts vs. expectations
- totals vs. known truths (begroting, previous delivery, CBS benchmarks)
- internal consistency (P × Q = Total, sum of parts = whole)
- format / type sanity
- delta vs. previous delivery

Anything that fails gets **flagged, not silently passed through to the AI**. Per the must-haves in CLAUDE.md: unvalidated data is never shown to the user without a flag.
