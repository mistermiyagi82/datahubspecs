# DataPraatFormaat — Complete Validation Framework

## Overview

| Phase | Name | Type | Total checks | MVP checks | Behaviour on failure |
|-------|------|------|:---:|:---:|------|
| 1 | File Check | Required | 10 | 10 | Stops on FAILED |
| 2 | Structure Recognition | Required | 13 | 13 | Stops on FAILED |
| 3 | Descriptive Statistics | Standard | 7 | 7 | Warnings in report |
| 4 | Content Validation | Standard | 19 | 12 | ERROR per cell/row, WARNING per column |
| 5 | Consistency Checks | Standard | 15 | 2 | ERROR per row per violated rule |
| 6 | Statistical Analysis | Optional | 6 | 2 | WARNING per finding |
| 7 | Intelligent Analysis (AI/ML/LLM) | Optional | 6 | 2 | Explanatory report with score and suggestions |
| | **TOTAL** | | **76** | **48** | |

**Legend:**
- `[MVP]` — included in first version (to be built)
- `[ ]` — not in MVP, on the roadmap (to be built later)
- **NEW** = check does not exist in the original framework, added based on interviews with Thom and Harm Nico

Nothing has been built yet. Everything needs to be implemented.

---

## Phase 1 — File Check

> *"Can we even open and read the file?"*
> Outcome: PASSED or FAILED — application stops on FAILED

| Status | ID | Check | Description | Example output | Format |
|:---:|------|-------|------------|----------------|--------|
| [MVP] | 1.1 | File exists | Is the specified path present and accessible? | PASSED: path found | All |
| [MVP] | 1.2 | Format recognition | Is the file type CSV, Excel, XML or JSON? Check both extension and magic bytes. | PASSED: CSV recognised | All |
| [MVP] | 1.3 | Not empty | Does the file have more than 0 bytes? Are there rows present? | FAILED: 0 bytes | All |
| [MVP] | 1.4 | Encoding detection | Which character set (UTF-8, Latin-1)? Are there unreadable characters? | WARNING: Latin-1, 3 errors at row 44, 201, 889 | All |
| [MVP] | 1.5 | Delimiter detection | Which separator (, ; tab)? Is it consistent? | PASSED: ";" consistent across 4,312 rows | CSV |
| [MVP] | 1.6 | Structural integrity | Is the file undamaged (corrupt Excel, invalid JSON, malformed XML)? | FAILED: JSON syntax error at position 4,821 | All |
| [MVP] | 1.7 | File size | Does the size fall within expected limits? | PASSED: 1.2 MB (max 50 MB) | All |
| [MVP] | 1.8 | Column widths defined | Are column widths (start and end positions) known via configuration? (Fixed-Width) | FAILED: column widths not defined | Fixed-Width |
| [MVP] | 1.9 | Repeating element determined | Which element serves as 'row'? Is this element consistently present? | FAILED: repeating element could not be determined, specify the XPath path | XML/YAML |
| [MVP] | 1.10 | Root type recognised | Is the root element an object {} or an array []? | FAILED: JSON does not start with an object or array | JSON |

**Phase 1 total: 10 checks — all MVP**

---

## Phase 2 — Structure Recognition

> *"Do we understand how the file is structured?"*
> Outcome: PASSED or FAILED — application stops on FAILED

| Status | ID | Check | Description | Example output | Format |
|:---:|------|-------|------------|----------------|--------|
| [MVP] | 2.1 | Column headers valid | Does the first row have column names? Are they non-empty and unique? For XML/YAML: are child elements named? | FAILED: column 4 has no name / "date" appears 2x | All |
| [MVP] | 2.2 | Consistent column count | Does every row have the same number of columns as the header? Deviating rows reported with row number. | WARNING: row 88 (16 instead of 18), row 204 (19 instead of 18) | CSV |
| [MVP] | 2.3 | Empty rows & columns | Are there completely empty rows or columns? If so, how many and where? | WARNING: 2 empty rows, 1 empty column | All |
| [MVP] | 2.4 | File dimensions | Total number of rows (excluding header) and columns as basic information. | INFO: 4,312 rows, 18 columns | All |
| [MVP] | 2.5 | Sheets validated | How many sheets are present? All non-empty sheets are validated. | INFO: 3 sheets; "Employees" and "Historical" validated, "Empty" skipped | Excel/ODS |
| [MVP] | 2.6 | Hidden rows & columns | Are there hidden rows or columns? Content is included but may cause unexpected results. | WARNING: sheet "Results" contains 5 hidden columns (F-J) | Excel/ODS |
| [MVP] | 2.7 | Consistent array structure | Do all objects in an array have the same keys? Missing or extra keys reported per position. | WARNING: item 14 missing key "date_of_birth" | JSON/JSONL |
| [MVP] | 2.8 | No duplicate key names | Do objects contain duplicate key names? | WARNING: object at position 5 contains key "date" twice | JSON/YAML |
| [MVP] | 2.9 | Nesting depth acceptable | How deeply is the structure nested? Excessive nesting often indicates an export error. | WARNING: maximum nesting depth is 9, expected max 3 | JSON/XML/YAML |
| [MVP] | 2.10 | Merged cells | Are there merged cells? Which cells and ranges? Merged cells break parsing. **NEW** | WARNING: 14 merged cells found (A1:C1, D5:D10, ...) | Excel/ODS |
| [MVP] | 2.11 | Formulas in cells | Does the file contain cells with formulas instead of values? Indicates manual editing, not a clean export. **NEW** | WARNING: 23 cells contain formulas (B2: =SUM(C2:F2), ...) | Excel/ODS |
| [MVP] | 2.12 | Repeated header rows | Does the header row appear multiple times in the file? Common in paginated Excel exports. **NEW** | WARNING: header row repeated at row 51, 101, 151 (pagination) | CSV/Excel |
| [MVP] | 2.13 | Signs of manual editing | Meta-check: combination of formulas, hidden rows/columns, merged cells, inconsistent formatting. Each signal alone is mild; together they indicate manual work. **NEW** | WARNING: 3 signs of manual editing (formulas, hidden columns, merged cells) | Excel/ODS |

**Phase 2 total: 13 checks — all MVP**

---

## Phase 3 — Descriptive Statistics

> *"What's broadly in the file — without judging?"*
> Outcome: Informational, warnings on notable values

| Status | ID | Check | Description | Example output | Format |
|:---:|------|-------|------------|----------------|--------|
| [MVP] | 3.1 | Fill rate per column | Percentage of filled values per column. Threshold e.g. < 50%. | WARNING: "comments" 32% filled | All |
| [MVP] | 3.2 | Data type detection | Per column: dominant type (number, date, text, boolean, mixed). | INFO: "age" number 98%, "name" text 100% | All |
| [MVP] | 3.3 | Uniqueness per column | Percentage of unique values. High = possible ID, low = categorical. | INFO: "employee_id" 100% unique | All |
| [MVP] | 3.4 | Most frequent values | Top 5 per column. Helps recognise value lists or errors. | INFO: "status" → OPEN 62%, CLOSED 31%, ... | All |
| [MVP] | 3.5 | Numeric statistics | Min, max, mean, median, stddev for numeric columns. | INFO: "salary" min 1,200, max 12,400, mean 3,847 | All |
| [MVP] | 3.6 | Date range | Earliest and latest date, span in days/months/years. | INFO: "date_of_birth" 1942-03-01 to 2005-11-30 | All |
| [MVP] | 3.7 | Text length | Shortest, longest and average length for text columns. | WARNING: "description" max 2,048 characters | All |

**Phase 3 total: 7 checks — all MVP**

---

## Phase 4 — Content Validation

> *"Is the content of each cell correct?"*
> Outcome: ERROR per cell/row, WARNING per column

| Status | ID | Check | Category | Description | Example output | Format |
|:---:|------|-------|----------|------------|----------------|--------|
| [MVP] | 4.1 | Empty and seemingly filled cells | Required fields | Which columns contain empty, NULL or "n/a"-like values? Are there cells containing only whitespace — appearing filled but effectively empty? | INFO: "date_of_birth" 3.6% empty / WARNING: 8 cells in "name" contain only spaces | All |
| [MVP] | 4.2 | Required columns always filled | Required fields | Are designated key columns always filled? Every empty cell is an ERROR. | ERROR: "employee_id" empty at row 88, 204 | All |
| [MVP] | 4.3 | Correct value type in each column | Type checks | Are there cells in a "number" column that contain text? Or vice versa? | ERROR: "age" contains "unknown" at row 14, 87 | All |
| [MVP] | 4.4 | Valid and consistent dates | Type checks | Is every value a valid date? Is the format used consistently? Are there future or unrealistic dates? | ERROR: 23 invalid dates; 120 rows use DD/MM/YYYY, rest DD-MM-YYYY | All |
| [MVP] | 4.5 | Numbers without formatting noise | Type checks | Do numeric columns contain currency symbols, comma/dot confusion or other formatting that prevents processing? | ERROR: "amount" contains euro sign at row 5, 6, 7 | All |
| [ ] | 4.6 | Valid email address | Format checks | Does it match pattern x@y.z? | ERROR: 5 invalid addresses (row 3, 77, 210) | All |
| [ ] | 4.7 | Valid phone number | Format checks | Does it match a recognised number pattern (NL, international)? | WARNING: 8 numbers match no pattern | All |
| [ ] | 4.8 | Valid postal code format | Format checks | Does it match expected postal code format (e.g. 1234 AB)? | ERROR: 3 invalid postal codes (row 6, 199, 400) | All |
| [ ] | 4.9 | Valid identification number | Format checks | Format and checksum validation for IBAN, BSN (Dutch citizen service number), KvK (Chamber of Commerce) and similar numbers. | ERROR: 2 BSN numbers fail eleven-test (row 14, 892) | All |
| [ ] | 4.10 | Custom pattern check | Format checks | Configurable: custom regex pattern per column (e.g. product number format PRD-XXXXX). | ERROR: 4 values don't match "product_number" | All |
| [MVP] | 4.11 | Value on allowed list | Domain & range | Do values fall within an expected fixed list? | WARNING: 3 unknown status values ("UNKNOWN", "n/a", "—") | All |
| [MVP] | 4.12 | Value outside allowed range | Domain & range | Do numbers or dates fall within specified min–max? Negative values where they're illogical? | ERROR: age 150 and -3 (row 402, 1188) | All |
| [MVP] | 4.13 | Duplicate rows and keys | Duplicates | Fully identical rows? Values in designated ID columns that appear multiple times? | WARNING: 6 identical rows / ERROR: "employee_id" M00412 appears 2x | All |
| [ ] | 4.14 | Null vs missing field consistent | JSON-specific | Is the difference between null and a completely missing field applied consistently? Both mean different things. | WARNING: "end_date" absent in 34 objects, present with value null in 12 objects | JSON/JSONL |
| [ ] | 4.15 | Number as text | JSON-specific | Do fields that should be numbers contain a string type? | ERROR: "amount" contains text value in 67 objects ("1200" instead of 1200) | JSON/JSONL |
| [MVP] | 4.16 | Known placeholder and default values | Placeholders & defaults | Do columns contain known system values for "unknown"? Dates (1900-01-01, 9999-12-31), numbers (999, -1), text (unknown, n/a, test, xxx). | WARNING: "date_of_birth" contains 142x "1900-01-01" (Excel zero date) | All |
| [MVP] | 4.17 | Number notation ambiguity | NL-specific | Is `1.234` Dutch (one thousand two hundred thirty four) or international (one point two three four)? Detect comma/dot confusion using pattern analysis. **NEW** | WARNING: "amount" contains ambiguous notation — 1.234 could be NL (1234) or EN (1.234). Pattern analysis: 94% NL format | All |
| [MVP] | 4.18 | Leading zeros lost | NL-specific | Are there numerically stored columns that should have been text (postal code, BSN, municipality code)? Check against expected length. **NEW** | ERROR: "postal_digits" contains value 162 (expected 4 digits) — presumably 0162 | Excel |
| [MVP] | 4.19 | Inconsistent spelling and whitespace | Data hygiene | Does a column contain the same value in different spellings? Trailing/leading spaces, capitalisation, accents. **NEW** | WARNING: "city" contains "Amsterdam", "amsterdam", "AMSTERDAM", "Amsterdam " (4 variants, presumably 1 intended) | All |

**Phase 4 total: 19 checks — 12 MVP, 7 later**

---

## Phase 5 — Consistency Checks

> *"Do the relationships between fields and external sources hold?"*
> Outcome: ERROR per row per violated rule

| Status | ID | Check | Description | Example output | Format |
|:---:|------|-------|------------|----------------|--------|
| [MVP] | 5.1 | Dates in logical order | Is start date always before end date? | ERROR: end date before start date in 14 rows | All |
| [MVP] | 5.2 | Amounts add up | Does the total equal the sum of the parts? P × Q = Total? | ERROR: total deviates at row 88, 204 | All |
| [ ] | 5.3 | Conditional field rules | If type=legal_entity, then kvk_number required. Configurable if-then rules. | ERROR: kvk_number empty while type=legal_entity (row 18, 77) | All |
| [ ] | 5.4 | Values in reference list | Do values appear in a specified reference list (e.g. CBS municipality codes)? | ERROR: municipality code 9999 not in reference list (row 5, 88, 301) | All |
| [ ] | 5.5 | Postal code and city match | Does the postal code + city combination match known lookup tables? | ERROR: 1011 AB belongs to Amsterdam, not Rotterdam (row 22, 109) | All |
| [ ] | 5.6 | Consistency across sheets | Do key values in one sheet appear in another? Consistently spelled? | WARNING: "Invoices" contains 34 client IDs not in "Clients" | Excel/ODS |
| [ ] | 5.7 | Consistency across files | Do key values appear in a specified reference file? Any missing records? | WARNING: employee file contains 12 IDs not in salary file | All |
| [ ] | 5.8 | Significant row count difference | More than threshold (default 20%) increase/decrease vs. previous upload? | WARNING: 412 rows, previous upload had 4,200 — 90% decrease | All |
| [ ] | 5.9 | Column structure unchanged | Columns added or removed vs. previous upload? Renamed columns? | WARNING: "department_code" missing; "dept_id" is new — possibly renamed | All |
| [ ] | 5.10 | Value distribution stable | Distribution significantly different from previous upload? Sudden shift? | WARNING: mean of "salary" dropped from €3,840 to €890 (-77%) | All |
| [ ] | 5.11 | New or disappeared category values | Values appeared or disappeared in fixed-list columns? | WARNING: "status" contains new value "BLOCKED" not in previous upload | All |
| [ ] | 5.12 | Expected row count | Does the row count fall within a user-specified expectation? **NEW** | WARNING: 412 rows, expectation was 4,000–5,000 (specified at upload) | All |
| [ ] | 5.13 | Expected column names | Are the expected columns present? Are there unexpected columns? **NEW** | ERROR: expected column "invoice_amount" missing; unexpected column "test_column" present | All |
| [ ] | 5.14 | Expected column totals | Does the column total match a user-specified expectation? **NEW** | WARNING: sum of "amount" = €13.5M, expectation was ~€5.2M (source: budget) | All |
| [ ] | 5.15 | Expected period | Does the date range fall within the expected reporting period? **NEW** | WARNING: data runs through October, expected period was full year 2024 | All |

**Phase 5 total: 15 checks — 2 MVP, 13 later**

---

## Phase 6 — Statistical Analysis

> *"Are there hidden anomalies that are hard to define with explicit rules?"*
> Outcome: WARNING per finding

| Status | ID | Check | Description | Example output | Format |
|:---:|------|-------|------------|----------------|--------|
| [MVP] | 6.1 | Statistical distribution and outliers | Are numeric values far outside expected spread (IQR or stddev)? Unusual distribution? | WARNING: "salary" 3 extreme values (€980k, €1.2M, €2.1M) | All |
| [ ] | 6.2 | Unexpected gaps or spikes in time | Gaps or spikes in time series data? | WARNING: no data in "registration_date" from 14 March to 2 April | All |
| [ ] | 6.3 | Expected correlation present | Column combinations that normally correlate but don't here? | WARNING: "hire_date" more than 60 years after "date_of_birth" in 34 rows | All |
| [ ] | 6.4 | Digit distribution (Benford / fraud signal) | Does first-digit distribution follow Benford's Law? Significant deviation may indicate fabricated values. | WARNING: first-digit distribution in "invoice_amount" deviates significantly from Benford | All |
| [ ] | 6.5 | Near-duplicate rows | Rows that closely resemble each other but aren't exactly identical? | WARNING: "Jan de Vries" vs "Jan de Vreis" (date_of_birth identical) — possibly same person | All |
| [MVP] | 6.6 | Homogeneity detection | Is one value dominant (>80%) in a column where variation is expected? | WARNING: "date_of_birth" is "1990-01-01" for 94% of rows (3,760 of 4,000) — possibly default value | All |

**Phase 6 total: 6 checks — 2 MVP, 4 later**

---

## Phase 7 — Intelligent Analysis (AI/ML/LLM)

> *"What can we discover that can't be captured in explicit rules?"*
> Outcome: Explanatory report with score and suggestions

| Status | ID | Check | Description | Example output | Format |
|:---:|------|-------|------------|----------------|--------|
| [MVP] | 7.1 | Semantic column interpretation | LLM identifies what the column likely contains based on name + values. | INFO: "ref_nr" appears to be an internal order number | All |
| [ ] | 7.2 | Automatic rule recognition | LLM recognises business rules based on patterns in the data. | INFO: column X always filled when column Y has value Z | All |
| [ ] | 7.3 | ML anomaly detection | Isolation Forest flags outliers at row level without predefined rules. | WARNING: row 1,204 anomalous on 6 of 18 features | All |
| [ ] | 7.4 | Text quality check | LLM evaluates free-text fields for test data or inconsistent style. | WARNING: "description" contains "aaaaaa" at row 88 | All |
| [MVP] | 7.5 | Data quality score | AI generates overall quality score 0-100 with summary. | Score: 74/100 — 3 columns with fill issues, 12 date errors | All |
| [ ] | 7.6 | Repair suggestions | LLM proposes concrete improvements per column or row. | "date_of_birth" appears to use DD/MM/YYYY, rest is ISO 8601 | All |

**Phase 7 total: 6 checks — 2 MVP, 4 later**

---

## Reporting — Status Values

| Status | Field | Possible values |
|:---:|-------|-----------------|
| [MVP] | report.status | APPROVED · APPROVED_WITH_WARNINGS · REJECTED |
| [MVP] | phase.status | PASSED · WARNINGS · FAILED · SKIPPED |
| [MVP] | finding.severity | ERROR · WARNING · INFO |

---

## Reporting — JSON Structure

### Report object

| Status | ID | Field | Type | Description |
|:---:|------|-------|------|------------|
| [MVP] | R.1 | report.file | string | File name of the validated file |
| [MVP] | R.2 | report.generated_at | ISO 8601 | Timestamp of validation |
| [MVP] | R.3 | report.status | enum | Overall verdict for the entire file |
| [MVP] | R.4 | report.summary | string | Human-friendly explanation of the verdict |
| [MVP] | R.5 | report.totals.errors | integer | Total number of ERROR findings across all phases |
| [MVP] | R.6 | report.totals.warnings | integer | Total number of WARNING findings |
| [MVP] | R.7 | report.totals.info | integer | Total number of INFO findings |
| [MVP] | R.8 | report.phases | array | List of phase objects |

### Phase object

| Status | ID | Field | Type | Description |
|:---:|------|-------|------|------------|
| [MVP] | F.1 | phase | integer | Phase number (1-7) |
| [MVP] | F.2 | name | string | Name of the phase |
| [MVP] | F.3 | status | enum | Result of the phase |
| [MVP] | F.4 | summary | string | Human-friendly explanation of the phase result |
| [MVP] | F.5 | statistics | object | Phase-specific statistics (free-form structure) |
| [MVP] | F.6 | findings | array | List of finding objects |

### Finding object

| Status | ID | Field | Type | Description |
|:---:|------|-------|------|------------|
| [MVP] | B.1 | severity | enum | ERROR · WARNING · INFO |
| [MVP] | B.2 | check_id | string | ID of the executed check (e.g. 4.5) |
| [MVP] | B.3 | column | string or null | Relevant column, or null for row-level findings |
| [MVP] | B.4 | description | string | Human-friendly description of the finding |
| [MVP] | B.5 | details | object | Additional technical details (free-form structure per check) |
| [MVP] | B.6 | rows | array | List of row numbers where the finding occurs |

---

## Summary

| Phase | MVP | Later | Total |
|-------|:---:|:---:|:---:|
| 1 — File Check | 10 | 0 | 10 |
| 2 — Structure Recognition | 13 | 0 | 13 |
| 3 — Descriptive Statistics | 7 | 0 | 7 |
| 4 — Content Validation | 12 | 7 | 19 |
| 5 — Consistency Checks | 2 | 13 | 15 |
| 6 — Statistical Analysis | 2 | 4 | 6 |
| 7 — Intelligent Analysis | 2 | 4 | 6 |
| **Total** | **48** | **28** | **76** |

Nothing has been built yet. Everything needs to be implemented.
