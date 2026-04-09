# DataPraat Validation Engine — Implementation Plan

## Context

DataPraat is a web application for Dutch accountants that validates uploaded data files (CSV, Excel, XML, JSON) before they're used for dashboarding and AI analysis. This plan covers building the **backend validation engine** — a Python + FastAPI service that accepts a file upload and returns a structured JSON validation report.

**Important:** The code must be built in a **separate repository** (not this spec repo). Copy the spec documents listed in the Reference section into the code repo or reference them as context.

The core requirement: a **pluggable check architecture** where adding a new check means creating one file in the right folder — no registration, no imports to update. The system runs 48 MVP checks across 7 phases, growing to 76 over time.

---

## Architecture Overview

```
POST /v1/validate (file + optional config)
         │
         ▼
    FileContext created (mutable state object)
         │
    ┌────┴────┐
    │ Phase 1  │  File-level checks → populates format, encoding, dataframe
    │ Phase 2  │  Structure checks → populates column_names, row_count
    └────┬────┘  FAILED here = pipeline stops, report = REJECTED
         │
    ┌────┴────┐
    │ Phase 3  │  Descriptive stats → populates column_types
    │ Phase 4  │  Content validation
    │ Phase 5  │  Consistency checks
    └────┬────┘  Findings accumulate, never blocks
         │
    ┌────┴────┐
    │ Phase 6  │  Statistical analysis
    │ Phase 7  │  AI/LLM analysis (Claude)
    └────┬────┘
         │
         ▼
    ValidationReport (JSON)
```

**Sections (logical groupings for the frontend):**
- Section 1 (Phase 1+2): File-level — "can we open it, is it structured?" Says nothing about the data.
- Section 2 (Phase 3+4+5): Data-level — statistics, content validation, consistency.
- Section 3 (Phase 6+7): Advanced — statistical anomalies, AI/LLM analysis.

---

## Project Structure

```
datapraat-backend/
├── pyproject.toml
├── .env.example
├── app/
│   ├── __init__.py
│   ├── main.py                          # FastAPI app, CORS, lifespan
│   ├── config.py                        # Settings (pydantic-settings)
│   │
│   ├── api/
│   │   └── v1/
│   │       ├── upload.py                # POST /v1/validate
│   │       └── health.py               # GET /v1/health
│   │
│   ├── models/
│   │   ├── enums.py                     # FileFormat, Severity, PhaseStatus, ReportStatus
│   │   ├── finding.py                   # Finding model
│   │   ├── report.py                    # ValidationReport, PhaseResult, Totals
│   │   ├── file_context.py              # FileContext (mutable dataclass)
│   │   └── check_config.py             # CheckConfig (per-upload settings)
│   │
│   ├── parsing/
│   │   ├── detect.py                    # Magic bytes + extension detection
│   │   ├── csv_parser.py
│   │   ├── excel_parser.py
│   │   ├── json_parser.py
│   │   └── xml_parser.py
│   │
│   ├── engine/
│   │   ├── registry.py                  # Auto-discovery of checks via pkgutil
│   │   └── pipeline.py                  # PipelineRunner (phase orchestration)
│   │
│   ├── checks/
│   │   ├── base.py                      # BaseCheck ABC
│   │   ├── phase1_file/                 # 10 checks
│   │   │   ├── __init__.py
│   │   │   ├── c0101_file_exists.py
│   │   │   ├── c0102_format_recognition.py
│   │   │   ├── c0103_not_empty.py
│   │   │   ├── c0104_encoding_detection.py
│   │   │   ├── c0105_delimiter_detection.py
│   │   │   ├── c0106_structural_integrity.py
│   │   │   ├── c0107_file_size.py
│   │   │   ├── c0108_column_widths.py
│   │   │   ├── c0109_repeating_element.py
│   │   │   └── c0110_root_type.py
│   │   ├── phase2_structure/            # 13 checks
│   │   │   ├── __init__.py
│   │   │   ├── c0201_column_headers.py
│   │   │   ├── c0202_consistent_column_count.py
│   │   │   ├── c0203_empty_rows_columns.py
│   │   │   ├── c0204_file_dimensions.py
│   │   │   ├── c0205_sheets_validated.py
│   │   │   ├── c0206_hidden_rows_columns.py
│   │   │   ├── c0207_consistent_array_structure.py
│   │   │   ├── c0208_no_duplicate_keys.py
│   │   │   ├── c0209_nesting_depth.py
│   │   │   ├── c0210_merged_cells.py
│   │   │   ├── c0211_formulas_in_cells.py
│   │   │   ├── c0212_repeated_header_rows.py
│   │   │   └── c0213_manual_editing_signs.py
│   │   ├── phase3_statistics/           # 7 checks
│   │   │   ├── __init__.py
│   │   │   ├── c0301_fill_rate.py
│   │   │   ├── c0302_datatype_detection.py
│   │   │   ├── c0303_uniqueness.py
│   │   │   ├── c0304_most_frequent_values.py
│   │   │   ├── c0305_numeric_statistics.py
│   │   │   ├── c0306_date_range.py
│   │   │   └── c0307_text_length.py
│   │   ├── phase4_content/              # 12 MVP checks
│   │   │   ├── __init__.py
│   │   │   ├── c0401_empty_cells.py
│   │   │   ├── c0402_required_columns_filled.py
│   │   │   ├── c0403_correct_value_types.py
│   │   │   ├── c0404_valid_dates.py
│   │   │   ├── c0405_number_formatting.py
│   │   │   ├── c0411_allowed_value_list.py
│   │   │   ├── c0412_value_range.py
│   │   │   ├── c0413_duplicate_rows_keys.py
│   │   │   ├── c0416_placeholder_values.py
│   │   │   ├── c0417_nl_number_notation.py
│   │   │   ├── c0418_leading_zeros.py
│   │   │   └── c0419_inconsistent_spelling.py
│   │   ├── phase5_consistency/          # 2 MVP checks
│   │   │   ├── __init__.py
│   │   │   ├── c0501_date_order.py
│   │   │   └── c0502_amounts_add_up.py
│   │   ├── phase6_statistical/          # 2 MVP checks
│   │   │   ├── __init__.py
│   │   │   ├── c0601_outlier_detection.py
│   │   │   └── c0606_homogeneity.py
│   │   └── phase7_ai/                   # 2 MVP checks
│   │       ├── __init__.py
│   │       ├── c0701_semantic_columns.py
│   │       └── c0705_quality_score.py
│   │
│   └── util/
│       ├── encoding.py                  # chardet wrapper
│       └── llm.py                       # Claude API client for Phase 7
│
└── tests/
    ├── conftest.py
    ├── fixtures/                         # Sample CSV, Excel, JSON, XML files
    ├── test_checks/
    │   ├── test_phase1/
    │   ├── test_phase2/
    │   ├── test_phase3/
    │   ├── test_phase4/
    │   ├── test_phase5/
    │   ├── test_phase6/
    │   └── test_phase7/
    ├── test_engine/
    │   ├── test_pipeline.py
    │   └── test_registry.py
    └── test_api/
        └── test_upload.py
```

Check file naming: `c{phase:02d}{check:02d}_{name}.py` — e.g., `c0102_format_recognition.py`

---

## Key Design Decisions

### 1. BaseCheck ABC

Every check is a class extending BaseCheck with class-level metadata and one `execute()` method:

```python
from abc import ABC, abstractmethod
from app.models.file_context import FileContext
from app.models.finding import Finding
from app.models.check_config import CheckConfig
from app.models.enums import FileFormat

class BaseCheck(ABC):
    check_id: str                              # "1.2"
    name: str                                  # "Format recognition"
    phase: int                                 # 1-7
    description: str                           # One-liner for reports
    formats: list[FileFormat] | None = None    # None = all formats

    @abstractmethod
    def execute(self, ctx: FileContext, config: CheckConfig) -> list[Finding]: ...

    def applies_to(self, ctx: FileContext) -> bool:
        if self.formats is None:
            return True
        return ctx.format in self.formats
```

Returns `list[Finding]` because one check can produce multiple findings (e.g., 23 invalid dates + a format inconsistency).

### 2. Auto-discovery registry

`pkgutil.walk_packages` scans the `checks/` directory tree and instantiates every BaseCheck subclass. No decorators, no import lists. Drop a file in the right phase folder → it's discovered on next startup.

```python
import importlib
import pkgutil
from app.checks.base import BaseCheck

class CheckRegistry:
    def __init__(self):
        self._checks: list[BaseCheck] = []

    def discover(self) -> None:
        import app.checks as checks_pkg
        for importer, modname, ispkg in pkgutil.walk_packages(
            checks_pkg.__path__, prefix="app.checks."
        ):
            module = importlib.import_module(modname)
            for attr_name in dir(module):
                attr = getattr(module, attr_name)
                if (
                    isinstance(attr, type)
                    and issubclass(attr, BaseCheck)
                    and attr is not BaseCheck
                    and hasattr(attr, "check_id")
                ):
                    self._checks.append(attr())
        self._checks.sort(key=lambda c: (c.phase, c.check_id))

    def get_checks_for_phase(self, phase: int) -> list[BaseCheck]:
        return [c for c in self._checks if c.phase == phase]

    def is_enabled(self, check: BaseCheck, disabled: list[str]) -> bool:
        return check.check_id not in disabled
```

### 3. FileContext (mutable dataclass, not Pydantic)

Flows through the entire pipeline. Phase 1 populates `format`, `encoding`, `dataframe`. Phase 3 populates `column_types`. Checks read and enrich it. Uses `metadata: dict` as escape hatch for format-specific objects (e.g., openpyxl Workbook). It's a dataclass because it holds a pandas DataFrame which doesn't serialize to JSON — this is internal working state, not a response model.

```python
from pathlib import Path
from dataclasses import dataclass, field
import pandas as pd
from app.models.enums import FileFormat

@dataclass
class FileContext:
    # Set at creation
    file_path: Path
    original_filename: str
    file_size_bytes: int

    # Set by Phase 1
    format: FileFormat = FileFormat.UNKNOWN
    encoding: str | None = None
    delimiter: str | None = None

    # Set by parsing (Phase 1 check 1.6)
    dataframe: pd.DataFrame | None = None
    sheets: dict[str, pd.DataFrame] = field(default_factory=dict)
    raw_json: dict | list | None = None
    raw_xml_tree: object | None = None

    # Set by Phase 2
    column_names: list[str] = field(default_factory=list)
    row_count: int = 0
    column_count: int = 0

    # Set by Phase 3, consumed by Phase 4+
    column_types: dict[str, str] = field(default_factory=dict)

    # Escape hatch for anything else
    metadata: dict = field(default_factory=dict)
```

### 4. Pipeline blocking logic

Phases 1-2: any ERROR → phase status FAILED → remaining phases SKIPPED → report status REJECTED.
Phases 3-7: errors produce findings but don't stop the pipeline. They are informational, not structural failures.

```python
BLOCKING_PHASES = {1, 2}

class PipelineRunner:
    def run(self, ctx: FileContext, config: CheckConfig) -> ValidationReport:
        phase_results = []
        pipeline_stopped = False

        for phase_num in range(1, 8):
            if pipeline_stopped:
                phase_results.append(PhaseResult(
                    phase=phase_num, name=PHASE_NAMES[phase_num],
                    status=PhaseStatus.SKIPPED, summary="Skipped due to earlier failure",
                ))
                continue

            checks = self.registry.get_checks_for_phase(phase_num)
            findings = []
            for check in checks:
                if not self.registry.is_enabled(check, config.disabled_checks):
                    continue
                if not check.applies_to(ctx):
                    continue
                findings.extend(check.execute(ctx, config))

            has_errors = any(f.severity == Severity.ERROR for f in findings)

            if has_errors and phase_num in BLOCKING_PHASES:
                status = PhaseStatus.FAILED
                pipeline_stopped = True
            elif has_errors or any(f.severity == Severity.WARNING for f in findings):
                status = PhaseStatus.WARNINGS
            else:
                status = PhaseStatus.PASSED

            phase_results.append(PhaseResult(
                phase=phase_num, name=PHASE_NAMES[phase_num],
                status=status, findings=findings,
                summary=self._summarize(phase_num, findings),
            ))

        return self._build_report(ctx, phase_results)
```

### 5. Per-upload CheckConfig (Pydantic model)

Parameterizes checks per upload:
- `disabled_checks: list[str]` — check IDs to skip (e.g., ["4.17", "6.1"])
- `required_columns: list[str]` — for check 4.2
- `key_columns: list[str]` — for duplicate detection (4.13)
- `date_columns`, `amount_columns` — column type hints
- `start_date_column`, `end_date_column` — for check 5.1
- `price_column`, `quantity_column`, `total_column` — for P×Q check (5.2)
- `column_rules: dict[str, ColumnRule]` — per-column allowed values, ranges
- Thresholds: `fill_rate_threshold`, `homogeneity_threshold`, `max_file_size_mb`, `outlier_method`

### 6. Parsing happens inside Phase 1

Check 1.6 (structural integrity) invokes the appropriate parser based on the format detected by check 1.2. If parsing succeeds, `ctx.dataframe` is populated and Phase 2+ can use it. If parsing fails (corrupt file), the check returns an ERROR finding and the pipeline stops.

### 7. No database for MVP

Stateless: file in, report out. Database for audit trail comes later.

### 8. Phase 7 uses Claude (Anthropic SDK)

Check 7.1 sends column names + sample values to Claude for semantic interpretation. Check 7.5 sends Phase 3 statistics + Phase 4-6 findings summary for an overall quality score 0-100.

### 9. Example check implementation

```python
# app/checks/phase1_file/c0107_file_size.py
from app.checks.base import BaseCheck
from app.models.file_context import FileContext
from app.models.finding import Finding
from app.models.check_config import CheckConfig
from app.models.enums import Severity

class FileSizeCheck(BaseCheck):
    check_id = "1.7"
    name = "File size"
    phase = 1
    description = "Does the size fall within expected limits?"
    formats = None  # all formats

    def execute(self, ctx: FileContext, config: CheckConfig) -> list[Finding]:
        max_bytes = config.max_file_size_mb * 1024 * 1024
        size_mb = ctx.file_size_bytes / (1024 * 1024)

        if ctx.file_size_bytes > max_bytes:
            return [Finding(
                severity=Severity.ERROR,
                check_id=self.check_id,
                description=f"File is {size_mb:.1f} MB, exceeds limit of {config.max_file_size_mb} MB",
                details={"size_bytes": ctx.file_size_bytes, "limit_bytes": int(max_bytes)},
            )]

        return [Finding(
            severity=Severity.INFO,
            check_id=self.check_id,
            description=f"File size: {size_mb:.1f} MB (limit: {config.max_file_size_mb} MB)",
            details={"size_bytes": ctx.file_size_bytes, "limit_bytes": int(max_bytes)},
        )]
```

---

## Pydantic Models: Report Output

```python
# app/models/enums.py
from enum import Enum

class FileFormat(str, Enum):
    CSV = "csv"
    EXCEL = "excel"
    JSON = "json"
    XML = "xml"
    FIXED_WIDTH = "fixed_width"
    UNKNOWN = "unknown"

class Severity(str, Enum):
    ERROR = "ERROR"
    WARNING = "WARNING"
    INFO = "INFO"

class PhaseStatus(str, Enum):
    PASSED = "PASSED"
    WARNINGS = "WARNINGS"
    FAILED = "FAILED"
    SKIPPED = "SKIPPED"

class ReportStatus(str, Enum):
    APPROVED = "APPROVED"
    APPROVED_WITH_WARNINGS = "APPROVED_WITH_WARNINGS"
    REJECTED = "REJECTED"
```

```python
# app/models/finding.py
from pydantic import BaseModel
from app.models.enums import Severity

class Finding(BaseModel):
    severity: Severity
    check_id: str
    column: str | None = None
    description: str
    details: dict = {}
    rows: list[int] = []
```

```python
# app/models/report.py
from datetime import datetime
from pydantic import BaseModel
from app.models.enums import PhaseStatus, ReportStatus
from app.models.finding import Finding

class PhaseResult(BaseModel):
    phase: int
    name: str
    status: PhaseStatus
    summary: str
    statistics: dict = {}
    findings: list[Finding] = []

class Totals(BaseModel):
    errors: int = 0
    warnings: int = 0
    info: int = 0

class ValidationReport(BaseModel):
    file: str
    generated_at: datetime
    status: ReportStatus
    summary: str
    totals: Totals
    phases: list[PhaseResult]
```

---

## Tech Stack

```
fastapi + uvicorn          # API server
pydantic v2                # Response models
pydantic-settings          # App config from env vars
python-multipart           # File upload parsing
pandas                     # Tabular data profiling + CSV parsing
openpyxl                   # Excel (merged cells, formulas, hidden rows, sheets)
lxml                       # XML parsing
chardet                    # Encoding detection
filetype                   # Magic bytes file type detection
anthropic                  # Claude API client for Phase 7
pytest                     # Testing
httpx                      # Test client for API tests
```

---

## Implementation Sequence

### Step 1: Project scaffold
- Init git repo
- `pyproject.toml` with all dependencies
- `app/main.py` with FastAPI app + CORS
- `app/config.py` with settings (ANTHROPIC_API_KEY, MAX_FILE_SIZE, etc.)
- `.env.example`
- `.gitignore`

### Step 2: Core models
- `app/models/enums.py`
- `app/models/finding.py`
- `app/models/report.py`
- `app/models/file_context.py`
- `app/models/check_config.py`

### Step 3: Engine
- `app/checks/base.py` — BaseCheck ABC
- `app/engine/registry.py` — CheckRegistry with auto-discovery
- `app/engine/pipeline.py` — PipelineRunner with blocking logic

### Step 4: Parsing layer
- `app/parsing/detect.py` — format detection
- `app/parsing/csv_parser.py`
- `app/parsing/excel_parser.py`
- `app/parsing/json_parser.py`
- `app/parsing/xml_parser.py`

### Step 5: Phase 1 checks (10 files)
All file-level checks. After this step, a file can be parsed and a basic report generated.

### Step 6: Phase 2 checks (13 files)
Structure recognition. After this step, Section 1 (file-level) is complete.

### Step 7: API endpoint + integration test
- `POST /v1/validate` endpoint
- `GET /v1/health` endpoint
- Integration test: upload a CSV → get a full report back
- Test: upload corrupt file → phases 3-7 skipped, REJECTED

### Step 8: Phase 3 checks (7 files)
Descriptive statistics. Populates `ctx.column_types` for Phase 4.

### Step 9: Phase 4 checks (12 files)
Content validation — the bulk of data-level checks.

### Step 10: Phase 5 + 6 checks (4 files)
Consistency + statistical analysis. After this, Section 2 + most of Section 3 done.

### Step 11: Phase 7 checks (2 files)
Claude-powered semantic column interpretation and quality scoring. Requires ANTHROPIC_API_KEY.

---

## Reference: Spec Documents (in this repo)

These documents define the full requirements:

1. **VALIDATION_FRAMEWORK_COMPLETE_EN.md** — Master checklist: all 76 checks, MVP/Later markings, JSON report structure
2. **VALIDATIE_FRAMEWORK_COMPLEET.md** — Same as above, in Dutch
3. **HARM-NICO-DATA-PHILOSOPHY.md** — Data philosophy: 5-step validation ladder, norms, deviations, financial data principle
4. **harm-nico-vision-to-features.md** — Vision → feature mapping: norm engine, deviation scoring, confidence indicators
5. **20260804-initial-file-sanity-check-thom (1).md** — Thom's real-world eyeball check workflow: what a data consultant does manually today

---

## Verification

1. **Unit tests**: Each check has its own test file with a minimal FileContext + assertion on findings
2. **Integration test**: Upload a real CSV/Excel file via the API, verify the full report JSON structure
3. **Manual test**: `curl -F "file=@test.csv" http://localhost:8000/v1/validate | jq .`
4. **Check discovery test**: Registry discovers exactly 48 checks, sorted by phase
5. **Blocking test**: Upload a corrupt Excel file → phases 3-7 SKIPPED, status REJECTED
6. **Format-specific test**: Upload JSON → Excel-only checks (2.10, 2.11) are skipped automatically
