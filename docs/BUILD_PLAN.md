# DataPraat Validation Engine -- Build Plan

## Overview

This plan breaks the work into 11 sequential batches. Each batch produces a testable, runnable increment. A developer (or AI agent) starts with an empty repo and follows these batches in order. Every batch lists the exact files to create, what each file does, the tests to write, and the "done" criteria.

**Total: 48 checks across 7 phases, plus infrastructure.**

Format support order: CSV first (simplest, most common for Dutch accountants), then Excel (richest feature set), then JSON, then XML.

---

## Batch 0 -- Project Scaffold

**Goal:** A bootable FastAPI app with dependencies installed, health endpoint responding, and CI-ready test runner.

### Files to create

**`pyproject.toml`**
- Python 3.12+
- Dependencies: `fastapi`, `uvicorn[standard]`, `pydantic>=2.0`, `pydantic-settings`, `python-multipart`, `pandas>=2.0`, `openpyxl`, `lxml`, `chardet`, `filetype`, `anthropic`, `pytest`, `httpx`, `pytest-asyncio`
- Project metadata: name `datapraat-backend`, version `0.1.0`
- `[tool.pytest.ini_options]`: `asyncio_mode = "auto"`, `testpaths = ["tests"]`
- Script entry: `dev = "uvicorn app.main:app --reload"`

**`.env.example`**
```
ANTHROPIC_API_KEY=sk-ant-xxx
MAX_FILE_SIZE_MB=50
ALLOWED_ORIGINS=http://localhost:3000
```

**`.gitignore`**
- Standard Python: `__pycache__/`, `*.pyc`, `.venv/`, `.env`, `dist/`, `*.egg-info/`
- IDE: `.vscode/`, `.idea/`
- Uploads: `uploads/`

**`app/__init__.py`** -- empty

**`app/main.py`**
- Create FastAPI app with title "DataPraat Validation Engine", version "0.1.0"
- CORS middleware reading origins from settings
- Lifespan context manager (placeholder for registry init)
- Include `api.v1` router

**`app/config.py`**
- `class Settings(BaseSettings)` with:
  - `anthropic_api_key: str = ""`
  - `max_file_size_mb: int = 50`
  - `allowed_origins: list[str] = ["http://localhost:3000"]`
  - `upload_dir: str = "uploads"`
- `model_config = SettingsConfigDict(env_file=".env")`
- Module-level `settings = Settings()` singleton

**`app/api/__init__.py`** -- empty
**`app/api/v1/__init__.py`** -- empty

**`app/api/v1/health.py`**
- `GET /v1/health` returning `{"status": "ok", "version": "0.1.0"}`

**`tests/__init__.py`** -- empty
**`tests/conftest.py`**
- `@pytest.fixture` for `client` using `httpx.AsyncClient` with `ASGITransport(app=app)`

**`tests/test_api/__init__.py`** -- empty
**`tests/test_api/test_health.py`**
- Test that `GET /v1/health` returns 200 with `{"status": "ok"}`

### Done criteria
- `pytest tests/test_api/test_health.py` passes
- `uvicorn app.main:app` starts without errors
- `curl localhost:8000/v1/health` returns `{"status":"ok","version":"0.1.0"}`

---

## Batch 1 -- Core Models

**Goal:** All Pydantic models, enums, the FileContext dataclass, and the CheckConfig model exist and are importable. These are the data contracts the rest of the system depends on.

### Files to create

**`app/models/__init__.py`** -- empty

**`app/models/enums.py`**
- `FileFormat(str, Enum)`: CSV, EXCEL, JSON, XML, FIXED_WIDTH, UNKNOWN
- `Severity(str, Enum)`: ERROR, WARNING, INFO
- `PhaseStatus(str, Enum)`: PASSED, WARNINGS, FAILED, SKIPPED
- `ReportStatus(str, Enum)`: APPROVED, APPROVED_WITH_WARNINGS, REJECTED

**`app/models/finding.py`**
- `class Finding(BaseModel)`:
  - `severity: Severity`
  - `check_id: str`
  - `column: str | None = None`
  - `description: str`
  - `details: dict = {}`
  - `rows: list[int] = []`

**`app/models/report.py`**
- `class PhaseResult(BaseModel)`:
  - `phase: int`
  - `name: str`
  - `status: PhaseStatus`
  - `summary: str`
  - `statistics: dict = {}`
  - `findings: list[Finding] = []`
- `class Totals(BaseModel)`:
  - `errors: int = 0`, `warnings: int = 0`, `info: int = 0`
- `class ValidationReport(BaseModel)`:
  - `file: str`
  - `generated_at: datetime`
  - `status: ReportStatus`
  - `summary: str`
  - `totals: Totals`
  - `phases: list[PhaseResult]`

**`app/models/file_context.py`**
- `@dataclass class FileContext`:
  - Creation fields: `file_path: Path`, `original_filename: str`, `file_size_bytes: int`
  - Phase 1 fields: `format: FileFormat = FileFormat.UNKNOWN`, `encoding: str | None = None`, `delimiter: str | None = None`
  - Parsing fields: `dataframe: pd.DataFrame | None = None`, `sheets: dict[str, pd.DataFrame] = field(default_factory=dict)`, `raw_json: dict | list | None = None`, `raw_xml_tree: object | None = None`
  - Phase 2 fields: `column_names: list[str] = field(default_factory=list)`, `row_count: int = 0`, `column_count: int = 0`
  - Phase 3 fields: `column_types: dict[str, str] = field(default_factory=dict)`
  - Escape hatch: `metadata: dict = field(default_factory=dict)`

**`app/models/check_config.py`**
- `class ColumnRule(BaseModel)`:
  - `allowed_values: list[str] | None = None`
  - `min_value: float | None = None`
  - `max_value: float | None = None`
- `class CheckConfig(BaseModel)`:
  - `disabled_checks: list[str] = []`
  - `required_columns: list[str] = []`
  - `key_columns: list[str] = []`
  - `date_columns: list[str] = []`
  - `amount_columns: list[str] = []`
  - `start_date_column: str | None = None`
  - `end_date_column: str | None = None`
  - `price_column: str | None = None`
  - `quantity_column: str | None = None`
  - `total_column: str | None = None`
  - `column_rules: dict[str, ColumnRule] = {}`
  - `fill_rate_threshold: float = 0.5`
  - `homogeneity_threshold: float = 0.8`
  - `max_file_size_mb: int = 50`
  - `outlier_method: str = "iqr"` (either "iqr" or "stddev")
  - `max_nesting_depth: int = 5`

### Tests

**`tests/test_models/__init__.py`** -- empty

**`tests/test_models/test_models.py`**
- Test `Finding` serializes to dict with correct keys
- Test `ValidationReport` round-trips through `.model_dump()` and `.model_validate()`
- Test `FileContext` is a proper dataclass with defaults
- Test `CheckConfig` defaults are sane (e.g., `max_file_size_mb == 50`)
- Test `FileFormat` values are lowercase strings
- Test `Severity` values are uppercase strings

### Done criteria
- All model imports work: `from app.models.enums import FileFormat` etc.
- `pytest tests/test_models/` passes
- `ValidationReport.model_json_schema()` produces valid JSON schema

---

## Batch 2 -- Engine (BaseCheck, Registry, Pipeline)

**Goal:** The pluggable check architecture works end-to-end. A dummy check dropped into a phase folder is auto-discovered and executed by the pipeline.

### Files to create

**`app/checks/__init__.py`** -- empty (critical: this makes `app.checks` a package that `pkgutil.walk_packages` can scan)

**`app/checks/base.py`**
- `class BaseCheck(ABC)`:
  - Class-level attributes: `check_id: str`, `name: str`, `phase: int`, `description: str`, `formats: list[FileFormat] | None = None`
  - `@abstractmethod def execute(self, ctx: FileContext, config: CheckConfig) -> list[Finding]`
  - `def applies_to(self, ctx: FileContext) -> bool`: returns True if `self.formats is None` or `ctx.format in self.formats`

**`app/engine/__init__.py`** -- empty

**`app/engine/registry.py`**
- `class CheckRegistry`:
  - `_checks: list[BaseCheck]` initialized empty
  - `def discover(self) -> None`: uses `pkgutil.walk_packages` on `app.checks.__path__` with prefix `"app.checks."`. For each module, iterates `dir(module)`, finds classes that are subclasses of `BaseCheck`, are not `BaseCheck` itself, and have a `check_id` attribute. Instantiates and appends. Sorts by `(phase, check_id)`.
  - `def get_checks_for_phase(self, phase: int) -> list[BaseCheck]`
  - `def is_enabled(self, check: BaseCheck, disabled: list[str]) -> bool`
  - `def all_checks(self) -> list[BaseCheck]`: returns the full sorted list
- Module-level constant: `PHASE_NAMES = {1: "File Check", 2: "Structure Recognition", 3: "Descriptive Statistics", 4: "Content Validation", 5: "Consistency Checks", 6: "Statistical Analysis", 7: "Intelligent Analysis"}`

**`app/engine/pipeline.py`**
- `BLOCKING_PHASES = {1, 2}`
- `class PipelineRunner`:
  - `__init__(self, registry: CheckRegistry)`
  - `def run(self, ctx: FileContext, config: CheckConfig) -> ValidationReport`:
    - Iterates phases 1-7
    - If `pipeline_stopped`, appends SKIPPED phase result
    - Otherwise, gets checks for phase, filters by `is_enabled` and `applies_to`
    - Collects findings from `check.execute(ctx, config)`
    - Determines phase status: ERROR in blocking phase = FAILED + stop; ERROR/WARNING in non-blocking = WARNINGS; else PASSED
    - Builds `ValidationReport` with overall status:
      - Any FAILED phase -> REJECTED
      - Any WARNINGS phase -> APPROVED_WITH_WARNINGS
      - All PASSED -> APPROVED
    - Computes totals (count of ERROR, WARNING, INFO findings)
    - Generates summary string

### Phase folder stubs (empty `__init__.py` in each -- required for pkgutil discovery)

- `app/checks/phase1_file/__init__.py`
- `app/checks/phase2_structure/__init__.py`
- `app/checks/phase3_statistics/__init__.py`
- `app/checks/phase4_content/__init__.py`
- `app/checks/phase5_consistency/__init__.py`
- `app/checks/phase6_statistical/__init__.py`
- `app/checks/phase7_ai/__init__.py`

### Tests

**`tests/test_engine/__init__.py`** -- empty

**`tests/test_engine/test_registry.py`**
- Create a minimal check class inline (not in the checks/ folder) to test that `issubclass` logic works
- Test that `discover()` finds 0 checks when phase folders are empty (at this point)
- Test `is_enabled` returns False when check_id is in disabled list
- Test `get_checks_for_phase` returns empty list for phase with no checks

**`tests/test_engine/test_pipeline.py`**
- Create two dummy check classes: one that returns INFO, one that returns ERROR
- Manually register them in a registry (bypass discovery, just append to `_checks`)
- Test: all INFO findings -> APPROVED
- Test: ERROR in phase 1 -> REJECTED, phases 2-7 SKIPPED
- Test: ERROR in phase 3 -> APPROVED_WITH_WARNINGS (non-blocking)
- Test: WARNING in phase 2 -> APPROVED_WITH_WARNINGS (not FAILED)
- Test: totals are correctly computed

### Done criteria
- Pipeline produces a valid `ValidationReport` with correct blocking behavior
- `pytest tests/test_engine/` passes
- SKIPPED phases have no findings

---

## Batch 3 -- Parsing Layer + Phase 1 Checks (CSV focus)

**Goal:** Upload a CSV file, Phase 1 runs all 10 checks, `ctx.dataframe` is populated, and the pipeline continues. This is the first end-to-end format.

### Files to create

**`app/util/__init__.py`** -- empty

**`app/util/encoding.py`**
- `def detect_encoding(file_path: Path) -> tuple[str, float]`: reads first 64KB with `chardet.detect()`, returns `(encoding_name, confidence)`
- `def find_encoding_errors(file_path: Path, encoding: str) -> list[int]`: attempts to decode each line, returns list of line numbers with decode errors

**`app/parsing/__init__.py`** -- empty

**`app/parsing/detect.py`**
- `def detect_format(file_path: Path, original_filename: str) -> FileFormat`:
  - Uses `filetype.guess()` for magic bytes. Maps Excel MIME to `FileFormat.EXCEL`.
  - Falls back to extension: `.csv` -> CSV, `.xlsx`/`.xls` -> EXCEL, `.json`/`.jsonl` -> JSON, `.xml` -> XML
  - Returns `FileFormat.UNKNOWN` if neither matches

**`app/parsing/csv_parser.py`**
- `def parse_csv(ctx: FileContext) -> pd.DataFrame`:
  - Reads using `pd.read_csv(ctx.file_path, encoding=ctx.encoding, sep=ctx.delimiter, dtype=str)` -- `dtype=str` is critical: prevents pandas from auto-converting, which loses leading zeros and mangles Dutch numbers
  - Sets `keep_default_na=False` so empty strings stay as empty strings (not NaN) -- important for distinguishing truly empty from "n/a" in Phase 4
  - Returns the DataFrame
  - Raises `ValueError` on parse failure (caught by check 1.6)

**`app/parsing/excel_parser.py`**
- `def parse_excel(ctx: FileContext) -> tuple[pd.DataFrame, dict[str, pd.DataFrame]]`:
  - Opens with `openpyxl.load_workbook(ctx.file_path, data_only=True)` for values
  - Also opens with `data_only=False` and stores in `ctx.metadata["workbook_with_formulas"]` for formula detection (check 2.11)
  - Reads first non-empty sheet as the primary DataFrame: `pd.read_excel(ctx.file_path, sheet_name=0, dtype=str, keep_default_na=False)`
  - Reads all sheets into `sheets` dict: `{name: pd.read_excel(..., sheet_name=name, dtype=str, keep_default_na=False)}`
  - Returns `(primary_df, sheets_dict)`
  - Raises `ValueError` on corrupt file

**`app/parsing/json_parser.py`**
- `def parse_json(ctx: FileContext) -> tuple[dict | list, pd.DataFrame | None]`:
  - Reads file with detected encoding, calls `json.loads()`
  - If result is a list of dicts: normalizes to DataFrame with `pd.json_normalize()`
  - If result is a dict with one key whose value is a list of dicts: normalizes that list
  - Otherwise: returns raw_json, DataFrame = None
  - Stores the raw parsed object in `ctx.raw_json`
  - Raises `json.JSONDecodeError` on parse failure

**`app/parsing/xml_parser.py`**
- `def parse_xml(ctx: FileContext) -> tuple[etree._Element, pd.DataFrame | None]`:
  - Uses `lxml.etree.parse()` with `encoding=ctx.encoding`
  - Detects repeating element (most frequent child tag of root, or of root's first child)
  - Extracts repeating elements into a list of dicts, normalizes to DataFrame
  - Stores `ctx.raw_xml_tree` = the parsed tree
  - Raises `lxml.etree.XMLSyntaxError` on parse failure

### Phase 1 checks (10 files)

Each check follows the pattern from the implementation plan. I will specify the non-obvious logic for each.

**`app/checks/phase1_file/c0101_file_exists.py`** -- `FileExistsCheck`
- `check_id = "1.1"`, `phase = 1`, `formats = None`
- `execute`: checks `ctx.file_path.exists()` and `ctx.file_path.is_file()`
- ERROR if not exists, INFO if exists

**`app/checks/phase1_file/c0102_format_recognition.py`** -- `FormatRecognitionCheck`
- `check_id = "1.2"`, `phase = 1`, `formats = None`
- `execute`: calls `detect_format(ctx.file_path, ctx.original_filename)`
- Sets `ctx.format = detected_format`
- ERROR if UNKNOWN, INFO with detected format otherwise
- Details include both magic-bytes result and extension

**`app/checks/phase1_file/c0103_not_empty.py`** -- `NotEmptyCheck`
- `check_id = "1.3"`, `phase = 1`, `formats = None`
- `execute`: checks `ctx.file_size_bytes > 0`. Also does a quick content check: reads first 1KB, checks if it is all whitespace.
- ERROR if 0 bytes or all whitespace, INFO otherwise

**`app/checks/phase1_file/c0104_encoding_detection.py`** -- `EncodingDetectionCheck`
- `check_id = "1.4"`, `phase = 1`, `formats = [CSV, XML, JSON]` (not Excel -- binary format)
- `execute`: calls `detect_encoding()`, sets `ctx.encoding`
- INFO if UTF-8 with high confidence
- WARNING if non-UTF-8 (e.g., Latin-1) -- Dutch files often are Latin-1
- If encoding errors found, include row numbers in `rows` field

**`app/checks/phase1_file/c0105_delimiter_detection.py`** -- `DelimiterDetectionCheck`
- `check_id = "1.5"`, `phase = 1`, `formats = [CSV]`
- `execute`: reads first 20 lines, uses `csv.Sniffer().sniff()` to detect delimiter. Falls back to counting occurrences of `,`, `;`, `\t` across lines and picking the most consistent.
- Sets `ctx.delimiter`
- INFO if consistent delimiter found. WARNING if inconsistent (different counts per line). Dutch CSV files very commonly use `;` -- the check must handle this.

**`app/checks/phase1_file/c0106_structural_integrity.py`** -- `StructuralIntegrityCheck`
- `check_id = "1.6"`, `phase = 1`, `formats = None`
- This is the BIG check. It calls the appropriate parser based on `ctx.format`:
  - CSV: `parse_csv(ctx)` -> sets `ctx.dataframe`
  - EXCEL: `parse_excel(ctx)` -> sets `ctx.dataframe` and `ctx.sheets`
  - JSON: `parse_json(ctx)` -> sets `ctx.raw_json` and optionally `ctx.dataframe`
  - XML: `parse_xml(ctx)` -> sets `ctx.raw_xml_tree` and optionally `ctx.dataframe`
- Wraps each parser call in try/except. On failure: returns ERROR with the exception message.
- On success: returns INFO ("File parsed successfully")
- This check MUST run after 1.2 (format known), 1.4 (encoding known), 1.5 (delimiter known). The pipeline runs checks in order within a phase, so file naming ensures this (`c0106` runs after `c0105`).

**`app/checks/phase1_file/c0107_file_size.py`** -- `FileSizeCheck`
- `check_id = "1.7"`, `phase = 1`, `formats = None`
- `execute`: compares `ctx.file_size_bytes` against `config.max_file_size_mb * 1024 * 1024`
- ERROR if over limit, INFO with size otherwise

**`app/checks/phase1_file/c0108_column_widths.py`** -- `ColumnWidthsCheck`
- `check_id = "1.8"`, `phase = 1`, `formats = [FIXED_WIDTH]`
- `execute`: Since FIXED_WIDTH is not in the MVP format support, this check will effectively never run (no file will have `format = FIXED_WIDTH`). Implement it as a stub that checks `ctx.metadata.get("column_widths")` and returns ERROR if not defined. This is future-proofing.

**`app/checks/phase1_file/c0109_repeating_element.py`** -- `RepeatingElementCheck`
- `check_id = "1.9"`, `phase = 1`, `formats = [XML]`
- `execute`: accesses `ctx.raw_xml_tree`, finds the most frequent direct child element tag of root (or root's first child). Stores in `ctx.metadata["repeating_element"]`.
- ERROR if no clear repeating element found, INFO with the tag name otherwise

**`app/checks/phase1_file/c0110_root_type.py`** -- `RootTypeCheck`
- `check_id = "1.10"`, `phase = 1`, `formats = [JSON]`
- `execute`: checks `ctx.raw_json` -- is it a dict (object) or list (array)?
- Stores `ctx.metadata["json_root_type"] = "object" | "array"`
- ERROR if raw_json is None or neither dict nor list, INFO with type otherwise

### Test fixtures

**`tests/fixtures/`** -- directory
- `valid.csv`: 10 rows, 5 columns, semicolon-delimited, UTF-8, with header. Columns: `id;naam;bedrag;datum;status` with realistic Dutch data.
- `valid_comma.csv`: same data but comma-delimited
- `empty.csv`: 0 bytes
- `whitespace_only.csv`: only whitespace/newlines
- `latin1.csv`: same data but encoded as Latin-1 with special chars (accents, umlauts)
- `corrupt.xlsx`: a text file renamed to .xlsx (will fail openpyxl parsing)
- `valid.json`: `[{"id": 1, "naam": "test", "bedrag": 100}]`
- `invalid.json`: truncated JSON (`[{"id": 1, "naam": "te`)
- `valid.xml`: `<data><row><id>1</id><naam>test</naam></row>...</data>`
- `malformed.xml`: XML with unclosed tags
- `oversized_header.csv`: a CSV file >51MB (or mock this in the test by setting `config.max_file_size_mb = 0`)

### Tests

**`tests/test_checks/__init__.py`** -- empty
**`tests/test_checks/test_phase1/__init__.py`** -- empty

**`tests/test_checks/test_phase1/test_c0101.py`**
- Test with existing file: INFO
- Test with non-existent path: ERROR

**`tests/test_checks/test_phase1/test_c0102.py`**
- Test CSV detection (magic bytes fail for CSV, extension succeeds): INFO with CSV
- Test XLSX detection: INFO with EXCEL
- Test unknown file (e.g., .txt with random content): ERROR

**`tests/test_checks/test_phase1/test_c0103.py`**
- Test 0-byte file: ERROR
- Test whitespace-only: ERROR
- Test valid CSV: INFO

**`tests/test_checks/test_phase1/test_c0104.py`**
- Test UTF-8 file: INFO with "utf-8"
- Test Latin-1 file: WARNING with "latin-1"

**`tests/test_checks/test_phase1/test_c0105.py`**
- Test semicolon CSV: INFO with delimiter ";"
- Test comma CSV: INFO with delimiter ","

**`tests/test_checks/test_phase1/test_c0106.py`**
- Test valid CSV (after ctx.format, ctx.encoding, ctx.delimiter are set): INFO, ctx.dataframe is not None
- Test corrupt XLSX: ERROR
- Test invalid JSON: ERROR
- Test malformed XML: ERROR

**`tests/test_checks/test_phase1/test_c0107.py`**
- Test file within size: INFO
- Test with `config.max_file_size_mb = 0`: ERROR (any file exceeds 0 MB)

**`tests/test_checks/test_phase1/test_c0109.py`**
- Test valid XML with repeating elements: INFO
- Test XML with no children: ERROR

**`tests/test_checks/test_phase1/test_c0110.py`**
- Test JSON array: INFO with "array"
- Test JSON object: INFO with "object"

**`tests/conftest.py`** (update from Batch 0)
- Add fixture `def csv_context(tmp_path)` that creates a valid CSV, returns a `FileContext` with path, filename, size set
- Add fixture `def default_config()` that returns `CheckConfig()` with defaults

### Done criteria
- `pytest tests/test_checks/test_phase1/` -- all pass
- Registry discovers exactly 10 checks for phase 1
- A CSV file runs through all 10 Phase 1 checks and `ctx.dataframe` is populated
- A corrupt file returns at least one ERROR finding

---

## Batch 4 -- Phase 2 Checks (Structure Recognition, 13 checks)

**Goal:** After Phase 1 populates `ctx.dataframe`, Phase 2 analyzes the structure. This completes Section 1 (file-level validation). After this batch, a file can be REJECTED or passed to data-level checks.

### Files to create

**`app/checks/phase2_structure/c0201_column_headers.py`** -- `ColumnHeadersCheck`
- `check_id = "2.1"`, `phase = 2`, `formats = None`
- `execute`: reads `ctx.dataframe.columns`. Checks:
  - No column is empty string or only whitespace -> ERROR per empty column
  - No duplicate column names -> ERROR per duplicate (with positions)
  - Sets `ctx.column_names = list(ctx.dataframe.columns)`
- INFO if all good

**`app/checks/phase2_structure/c0202_consistent_column_count.py`** -- `ConsistentColumnCountCheck`
- `check_id = "2.2"`, `phase = 2`, `formats = [CSV]`
- `execute`: For CSV files, pandas may have already handled this (ragged rows get NaN). Re-read the raw file line by line and count delimiters per line. Report rows where delimiter count differs from the header row.
- WARNING with `rows` listing the deviating row numbers. INFO if consistent.

**`app/checks/phase2_structure/c0203_empty_rows_columns.py`** -- `EmptyRowsColumnsCheck`
- `check_id = "2.3"`, `phase = 2`, `formats = None`
- `execute`: 
  - Empty rows: `ctx.dataframe` rows where all values are empty string or NaN. Count and list row numbers.
  - Empty columns: columns where all values are empty string or NaN. List column names.
  - WARNING if any found (with details dict containing `empty_rows` and `empty_columns`), INFO otherwise.

**`app/checks/phase2_structure/c0204_file_dimensions.py`** -- `FileDimensionsCheck`
- `check_id = "2.4"`, `phase = 2`, `formats = None`
- `execute`: 
  - `ctx.row_count = len(ctx.dataframe)`
  - `ctx.column_count = len(ctx.dataframe.columns)`
  - Always returns INFO with row and column counts.

**`app/checks/phase2_structure/c0205_sheets_validated.py`** -- `SheetsValidatedCheck`
- `check_id = "2.5"`, `phase = 2`, `formats = [EXCEL]`
- `execute`: inspects `ctx.sheets`. Counts total sheets, identifies non-empty ones (at least one non-empty cell).
- INFO with sheet names and which are non-empty / skipped.

**`app/checks/phase2_structure/c0206_hidden_rows_columns.py`** -- `HiddenRowsColumnsCheck`
- `check_id = "2.6"`, `phase = 2`, `formats = [EXCEL]`
- `execute`: uses `ctx.metadata["workbook_with_formulas"]` (the openpyxl Workbook). Iterates sheets:
  - Hidden rows: `ws.row_dimensions[r].hidden`
  - Hidden columns: `ws.column_dimensions[c].hidden`
- WARNING listing hidden rows/columns per sheet, INFO if none found.

**`app/checks/phase2_structure/c0207_consistent_array_structure.py`** -- `ConsistentArrayStructureCheck`
- `check_id = "2.7"`, `phase = 2`, `formats = [JSON]`
- `execute`: if `ctx.raw_json` is a list of dicts, compares keys of each dict to the first dict's keys. Reports items with missing or extra keys.
- WARNING per item with deviations, INFO if all consistent.

**`app/checks/phase2_structure/c0208_no_duplicate_keys.py`** -- `NoDuplicateKeysCheck`
- `check_id = "2.8"`, `phase = 2`, `formats = [JSON]`
- `execute`: Re-reads the raw file using `json.loads()` with an `object_pairs_hook` that detects duplicate keys within a single object. Reports position and key name.
- WARNING per duplicate key, INFO if none.

**`app/checks/phase2_structure/c0209_nesting_depth.py`** -- `NestingDepthCheck`
- `check_id = "2.9"`, `phase = 2`, `formats = [JSON, XML]`
- `execute`: recursively measures max depth.
  - JSON: recursive traversal of `ctx.raw_json`
  - XML: recursive traversal of `ctx.raw_xml_tree`
- WARNING if depth exceeds `config.max_nesting_depth`, INFO otherwise.

**`app/checks/phase2_structure/c0210_merged_cells.py`** -- `MergedCellsCheck`
- `check_id = "2.10"`, `phase = 2`, `formats = [EXCEL]`
- `execute`: uses openpyxl Workbook from `ctx.metadata`. Reads `ws.merged_cells.ranges` for each sheet.
- WARNING listing all merged cell ranges, INFO if none.

**`app/checks/phase2_structure/c0211_formulas_in_cells.py`** -- `FormulasInCellsCheck`
- `check_id = "2.11"`, `phase = 2`, `formats = [EXCEL]`
- `execute`: uses the `data_only=False` Workbook from `ctx.metadata["workbook_with_formulas"]`. Scans cells: if `cell.value` is a string starting with `=`, it is a formula.
- WARNING listing formula cells (up to 50, then truncate), INFO if none.
- Stores `ctx.metadata["formula_count"]` for check 2.13.

**`app/checks/phase2_structure/c0212_repeated_header_rows.py`** -- `RepeatedHeaderRowsCheck`
- `check_id = "2.12"`, `phase = 2`, `formats = [CSV, EXCEL]`
- `execute`: takes the header row values and searches for identical rows in the data. Reports row numbers where the header repeats (common in paginated exports).
- WARNING with row numbers, INFO if no repeats.

**`app/checks/phase2_structure/c0213_manual_editing_signs.py`** -- `ManualEditingSignsCheck`
- `check_id = "2.13"`, `phase = 2`, `formats = [EXCEL]`
- `execute`: composite meta-check. Counts signals:
  - `ctx.metadata.get("formula_count", 0) > 0`
  - Any hidden rows/columns (reads from earlier findings or re-checks)
  - Any merged cells (reads from earlier findings or re-checks)
- For simplicity: reads `ctx.metadata` for counts set by checks 2.6, 2.10, 2.11. Those checks should store their counts in `ctx.metadata` (e.g., `ctx.metadata["hidden_count"]`, `ctx.metadata["merged_count"]`, `ctx.metadata["formula_count"]`).
- WARNING if 2+ signals present, INFO otherwise. Description lists which signals were found.

### Test fixtures (add to existing)
- `valid.xlsx`: a proper Excel file with 3 sheets, one empty. 10 rows, 5 columns.
- `merged_cells.xlsx`: Excel file with merged cells in A1:C1
- `formulas.xlsx`: Excel file with `=SUM(B2:B10)` in a cell
- `hidden_cols.xlsx`: Excel file with column D hidden
- `paginated.csv`: a CSV where the header row repeats at rows 26 and 51
- `duplicate_keys.json`: `[{"id": 1, "name": "a", "name": "b"}]`
- `nested_deep.json`: JSON nested 10 levels deep
- `inconsistent_array.json`: `[{"a":1,"b":2}, {"a":1,"c":3}]`

### Tests

**`tests/test_checks/test_phase2/__init__.py`** -- empty

One test file per check, following the pattern from Phase 1. Key tests:

- `test_c0201.py`: Test with duplicate column names -> ERROR. Test with empty column name -> ERROR. Test normal -> INFO + `ctx.column_names` populated.
- `test_c0202.py`: Test CSV with inconsistent row lengths -> WARNING with row numbers. Test consistent CSV -> INFO.
- `test_c0203.py`: Test with empty rows -> WARNING. Test clean file -> INFO.
- `test_c0204.py`: Test dimensions set correctly on ctx.
- `test_c0205.py`: Test multi-sheet Excel -> INFO listing sheets.
- `test_c0206.py`: Test Excel with hidden columns -> WARNING.
- `test_c0207.py`: Test inconsistent JSON array -> WARNING.
- `test_c0208.py`: Test duplicate keys -> WARNING.
- `test_c0209.py`: Test deep nesting -> WARNING with depth.
- `test_c0210.py`: Test merged cells Excel -> WARNING.
- `test_c0211.py`: Test formulas Excel -> WARNING.
- `test_c0212.py`: Test paginated CSV with repeated headers -> WARNING.
- `test_c0213.py`: Test Excel with formulas + hidden cols + merged cells -> WARNING "3 signs".

### Done criteria
- `pytest tests/test_checks/test_phase2/` -- all pass
- Registry discovers exactly 23 checks (10 Phase 1 + 13 Phase 2)
- Integration: a valid CSV through phases 1-2 produces APPROVED with all Phase 2 findings being INFO
- Integration: an Excel file with merged cells + formulas through phases 1-2 produces APPROVED_WITH_WARNINGS

---

## Batch 5 -- API Endpoint + First Integration Test

**Goal:** The `POST /v1/validate` endpoint works. You can upload a file and get a full validation report (phases 1-2 running, phases 3-7 skipped until those checks are built). This is the first user-facing feature.

### Files to create

**`app/api/v1/upload.py`**
- `POST /v1/validate`:
  - Accepts `file: UploadFile` and optional `config: str = Form(None)` (JSON string of CheckConfig)
  - Saves uploaded file to `settings.upload_dir / uuid_filename`
  - Parses config JSON string into `CheckConfig` (or uses defaults)
  - Creates `FileContext(file_path=..., original_filename=file.filename, file_size_bytes=...)`
  - Gets or creates `CheckRegistry` (initialized at app startup via lifespan)
  - Creates `PipelineRunner(registry)`, calls `runner.run(ctx, config)`
  - Returns `ValidationReport` as JSON
  - Cleans up temp file in a finally block
  - Error handling: 413 if file too large (before saving), 422 if config JSON is invalid

**`app/main.py`** (update)
- In lifespan, create registry, call `registry.discover()`, store on `app.state.registry`
- Include the upload router

### Tests

**`tests/test_api/test_upload.py`**
- Test upload valid CSV -> 200, response has `status: "APPROVED"`, `phases` has 7 items (2 with findings, 5 skipped until checks exist -- or PASSED with 0 findings once phases have no checks)
- Test upload empty file -> 200, response has `status: "REJECTED"` (Phase 1 ERROR from check 1.3)
- Test upload corrupt .xlsx -> 200, response has `status: "REJECTED"` (Phase 1 ERROR from check 1.6)
- Test upload with config disabling a check -> that check's findings absent
- Test upload with invalid config JSON -> 422
- Test response structure matches `ValidationReport` schema exactly

**`tests/test_engine/test_registry_integration.py`**
- Test that `registry.discover()` finds exactly 23 checks (after batches 3+4)
- Test checks are sorted by (phase, check_id)

### Done criteria
- `curl -F "file=@tests/fixtures/valid.csv" http://localhost:8000/v1/validate | python -m json.tool` returns a valid report
- Upload a corrupt file -> REJECTED with phases 3-7 SKIPPED
- Upload a valid Excel with merged cells -> APPROVED_WITH_WARNINGS
- Response validates against `ValidationReport.model_json_schema()`

---

## Batch 6 -- Phase 3 Checks (Descriptive Statistics, 7 checks)

**Goal:** Phase 3 populates `ctx.column_types` (critical for Phase 4) and produces per-column statistics. These are mostly pandas one-liners wrapped in the check architecture.

### Files to create

**`app/checks/phase3_statistics/c0301_fill_rate.py`** -- `FillRateCheck`
- `check_id = "3.1"`, `phase = 3`, `formats = None`
- `execute`: for each column, compute `(non_empty_count / total_rows) * 100`. An empty value is `""`, `NaN`, `None`.
- Returns one Finding per column: INFO if fill rate >= threshold, WARNING if below `config.fill_rate_threshold`.
- `details` dict: `{"column": name, "fill_rate": 0.32, "filled": 1380, "total": 4312}`
- `statistics` on the phase result should aggregate fill rates.

**`app/checks/phase3_statistics/c0302_datatype_detection.py`** -- `DataTypeDetectionCheck`
- `check_id = "3.2"`, `phase = 3`, `formats = None`
- `execute`: for each column, sample non-empty values and classify each value:
  - Try `int()` / `float()` -> "number"
  - Try date parsing (multiple formats: ISO, DD-MM-YYYY, DD/MM/YYYY, etc.) -> "date"
  - Check for boolean-like ("true", "false", "ja", "nee", "0", "1") -> "boolean"
  - Otherwise -> "text"
  - Dominant type = the type with >50% of values. If no majority -> "mixed"
- **Critical:** Sets `ctx.column_types[column_name] = dominant_type`. Phase 4 checks depend on this.
- Returns INFO per column with type breakdown in details: `{"dominant_type": "number", "breakdown": {"number": 98, "text": 2}}`

**`app/checks/phase3_statistics/c0303_uniqueness.py`** -- `UniquenessCheck`
- `check_id = "3.3"`, `phase = 3`, `formats = None`
- `execute`: for each column, compute `nunique / count_non_empty * 100`.
- INFO per column with `{"uniqueness_pct": 100.0, "unique_values": 4312, "total_non_empty": 4312}`
- Useful heuristic in details: 100% unique = "possible ID", <10 unique = "categorical"

**`app/checks/phase3_statistics/c0304_most_frequent_values.py`** -- `MostFrequentValuesCheck`
- `check_id = "3.4"`, `phase = 3`, `formats = None`
- `execute`: for each column, `value_counts().head(5)` -> top 5 with counts and percentages.
- INFO per column with `{"top_values": [{"value": "OPEN", "count": 2674, "pct": 62.0}, ...]}`

**`app/checks/phase3_statistics/c0305_numeric_statistics.py`** -- `NumericStatisticsCheck`
- `check_id = "3.5"`, `phase = 3`, `formats = None`
- `execute`: for each column where `ctx.column_types[col] == "number"`:
  - Convert to float (handling errors), compute min, max, mean, median, stddev
  - Store in `ctx.metadata[f"numeric_stats_{col}"]` for use by Phase 6 outlier detection
- INFO per numeric column with stats dict.
- Skip non-numeric columns entirely.

**`app/checks/phase3_statistics/c0306_date_range.py`** -- `DateRangeCheck`
- `check_id = "3.6"`, `phase = 3`, `formats = None`
- `execute`: for each column where `ctx.column_types[col] == "date"`:
  - Parse all values to dates (using pd.to_datetime with `dayfirst=True` for Dutch format), find min and max
  - Compute span in days
  - Store in `ctx.metadata[f"date_stats_{col}"]`
- INFO per date column: `{"earliest": "1942-03-01", "latest": "2005-11-30", "span_days": 23283}`

**`app/checks/phase3_statistics/c0307_text_length.py`** -- `TextLengthCheck`
- `check_id = "3.7"`, `phase = 3`, `formats = None`
- `execute`: for each column where `ctx.column_types[col] == "text"`:
  - Compute min, max, mean length of non-empty string values
- WARNING if max length > 1000 (potentially problematic for downstream), INFO otherwise.

### Tests

**`tests/test_checks/test_phase3/__init__.py`** -- empty

For each check, create a `FileContext` with a pre-built DataFrame (no need to go through parsing -- just set `ctx.dataframe = pd.DataFrame(...)` directly).

- `test_c0301.py`: Column with 50% nulls -> WARNING. Column 100% filled -> INFO.
- `test_c0302.py`: Column of numbers -> type "number". Column of dates -> type "date". Mixed column -> type "mixed". **Verify `ctx.column_types` is populated.**
- `test_c0303.py`: All unique column -> 100%. Single-value column -> uniqueness near 0%.
- `test_c0304.py`: Verify top 5 values are correct for a known distribution.
- `test_c0305.py`: Numeric column stats are correct. Non-numeric column skipped.
- `test_c0306.py`: Date column range correct. Handles DD-MM-YYYY (Dutch format).
- `test_c0307.py`: Text length stats correct. Long text triggers WARNING.

### Done criteria
- `pytest tests/test_checks/test_phase3/` -- all pass
- After Phase 3, `ctx.column_types` is populated for all columns
- Upload valid CSV -> report now includes Phase 3 findings (previously PASSED with 0 findings)
- Phase 3 `statistics` field on PhaseResult contains aggregated stats

---

## Batch 7 -- Phase 4 Checks (Content Validation, 12 checks)

**Goal:** The data-level validation checks. This is the most complex batch with real logic for type validation, Dutch number formats, placeholder detection, etc. Depends on `ctx.column_types` from Phase 3.

### Files to create

**`app/checks/phase4_content/c0401_empty_cells.py`** -- `EmptyCellsCheck`
- `check_id = "4.1"`, `phase = 4`, `formats = None`
- `execute`: for each column, find cells that are:
  - Truly empty (empty string / NaN)
  - Whitespace-only (spaces, tabs -- appears filled but effectively empty)
  - "n/a"-like: "n/a", "N/A", "na", "NA", "-", "--", "null", "NULL", "none", "None"
- INFO per column with empty count. WARNING for whitespace-only cells (these are deceptive).
- `rows` field lists row numbers for whitespace-only cells.

**`app/checks/phase4_content/c0402_required_columns_filled.py`** -- `RequiredColumnsFilledCheck`
- `check_id = "4.2"`, `phase = 4`, `formats = None`
- `execute`: for each column in `config.required_columns`:
  - Find rows where value is empty, NaN, or whitespace-only
  - ERROR per empty occurrence with row number
- If `config.required_columns` is empty, return single INFO "No required columns configured".

**`app/checks/phase4_content/c0403_correct_value_types.py`** -- `CorrectValueTypesCheck`
- `check_id = "4.3"`, `phase = 4`, `formats = None`
- `execute`: for each column, using `ctx.column_types[col]`:
  - If dominant type is "number": find cells that are not parseable as numbers (ignoring empty)
  - If dominant type is "date": find cells that are not parseable as dates (ignoring empty)
  - If dominant type is "boolean": find cells that are not boolean-like
- ERROR per mismatched cell with row number.

**`app/checks/phase4_content/c0404_valid_dates.py`** -- `ValidDatesCheck`
- `check_id = "4.4"`, `phase = 4`, `formats = None`
- `execute`: for date columns (from `ctx.column_types` or `config.date_columns`):
  - Parse each value with multiple format attempts
  - Check consistency: are formats mixed? (DD-MM-YYYY vs DD/MM/YYYY vs YYYY-MM-DD)
  - Check plausibility: no dates before 1900 or after 2100 (configurable)
  - Check for future dates (> today)
- ERROR per invalid date. WARNING for format inconsistency (report the counts of each format found). WARNING for future dates.

**`app/checks/phase4_content/c0405_number_formatting.py`** -- `NumberFormattingCheck`
- `check_id = "4.5"`, `phase = 4`, `formats = None`
- `execute`: for numeric columns:
  - Detect currency symbols (EUR, $, etc.)
  - Detect thousands separators (1.000 or 1,000)
  - Detect percentage signs
  - These prevent clean float conversion
- ERROR per cell with formatting noise, with row numbers. Details include what was found.

**`app/checks/phase4_content/c0411_allowed_value_list.py`** -- `AllowedValueListCheck`
- `check_id = "4.11"`, `phase = 4`, `formats = None`
- `execute`: for columns with `config.column_rules[col].allowed_values` defined:
  - Find values not in the allowed list
- WARNING per non-allowed value with count and examples.
- If no column_rules defined, return INFO "No allowed value lists configured".

**`app/checks/phase4_content/c0412_value_range.py`** -- `ValueRangeCheck`
- `check_id = "4.12"`, `phase = 4`, `formats = None`
- `execute`: for columns with `config.column_rules[col].min_value` or `max_value`:
  - Convert to float, check against min/max
  - Also check for negative values in amount columns (rarely valid)
- ERROR per out-of-range value with row number.

**`app/checks/phase4_content/c0413_duplicate_rows_keys.py`** -- `DuplicateRowsKeysCheck`
- `check_id = "4.13"`, `phase = 4`, `formats = None`
- `execute`:
  - Fully duplicate rows: `ctx.dataframe.duplicated()`. WARNING with count and first 10 row numbers.
  - If `config.key_columns` defined: check uniqueness of those columns. ERROR per duplicate key.

**`app/checks/phase4_content/c0416_placeholder_values.py`** -- `PlaceholderValuesCheck`
- `check_id = "4.16"`, `phase = 4`, `formats = None`
- Known placeholder patterns:
  - Dates: "1900-01-01", "9999-12-31", "1899-12-30" (Excel zero date)
  - Numbers: 0, -1, 999, 9999, 99999
  - Text: "unknown", "n/a", "test", "xxx", "aaa", "todo", "temp", "dummy", "default", "TBD"
- `execute`: for each column, scan for these patterns (case-insensitive for text).
- WARNING per column with placeholder values, including count and which placeholders found.

**`app/checks/phase4_content/c0417_nl_number_notation.py`** -- `NLNumberNotationCheck`
- `check_id = "4.17"`, `phase = 4`, `formats = None`
- The Dutch number problem: `1.234` could be Dutch for 1234 or international for 1.234.
- `execute`: for numeric columns, analyze the notation pattern:
  - Count values matching `\d{1,3}\.\d{3}` (Dutch thousands: 1.234, 12.345)
  - Count values matching `\d+\.\d{1,2}` (decimal: 1.23, 12.5)
  - Count values with comma as decimal separator (Dutch: 1.234,56)
  - If ambiguous (both patterns present without comma as tie-breaker), WARNING.
- Details: `{"pattern_analysis": {"dutch_format_count": 3800, "international_format_count": 12, "likely_format": "dutch"}}`

**`app/checks/phase4_content/c0418_leading_zeros.py`** -- `LeadingZerosCheck`
- `check_id = "4.18"`, `phase = 4`, `formats = None`
- `execute`: for columns that are numeric but might be identifiers:
  - Check if column name suggests an identifier (contains "postcode", "postal", "bsn", "zipcode", "code", "gemeente")
  - If values are shorter than expected (e.g., "162" instead of "0162" for Dutch postcodes, or 8 digits instead of 9 for BSN)
  - ERROR per value that appears truncated, with expected length.
- Special handling: Dutch postcodes are 4 digits (0000-9999), BSN is 9 digits.

**`app/checks/phase4_content/c0419_inconsistent_spelling.py`** -- `InconsistentSpellingCheck`
- `check_id = "4.19"`, `phase = 4`, `formats = None`
- `execute`: for text columns with low cardinality (<100 unique values -- likely categorical):
  - Normalize: strip whitespace, lowercase
  - Group values that normalize to the same string
  - Report groups with >1 variant
- WARNING per column with variants: `{"variants": {"amsterdam": ["Amsterdam", "amsterdam", "AMSTERDAM", "Amsterdam "]}}`
- This catches trailing spaces, inconsistent casing, and accented vs non-accented.

### Tests

**`tests/test_checks/test_phase4/__init__.py`** -- empty

Build each test with a synthetic DataFrame and pre-populated `ctx.column_types`.

- `test_c0401.py`: Whitespace-only cells detected. "n/a"-like values detected.
- `test_c0402.py`: Required column with empties -> ERROR per row. No config -> INFO.
- `test_c0403.py`: "unknown" in a number column -> ERROR. Valid numbers -> no error.
- `test_c0404.py`: "2025-13-01" (invalid month) -> ERROR. Mixed DD/MM and MM/DD formats -> WARNING. Future dates -> WARNING.
- `test_c0405.py`: "EUR 1.234,56" -> ERROR (currency symbol). "1,234.56" -> ERROR (formatting noise).
- `test_c0411.py`: Value not in allowed list -> WARNING. All values in list -> INFO.
- `test_c0412.py`: Value -5 when min is 0 -> ERROR. Value 200 when max is 100 -> ERROR.
- `test_c0413.py`: Duplicate rows detected. Duplicate keys detected.
- `test_c0416.py`: "1900-01-01" in a date column -> WARNING. "unknown" in text column -> WARNING.
- `test_c0417.py`: Column with "1.234" and "5.678" and no decimals -> WARNING (ambiguous). Column with "1.234,56" -> INFO (clearly Dutch).
- `test_c0418.py`: Postcode column with value "162" -> ERROR (should be "0162"). BSN column with 8-digit values -> ERROR.
- `test_c0419.py`: "Amsterdam" and "amsterdam" and "Amsterdam " in same column -> WARNING with 3 variants.

### Done criteria
- `pytest tests/test_checks/test_phase4/` -- all pass
- Registry discovers 35 checks (10 + 13 + 7 + 12 -- assuming Phase 3 checks already exist)
- Wait, correction: Phase 3 has 7 checks, so total = 10 + 13 + 7 + 12 = 42
- Upload a CSV with known issues -> report contains Phase 4 findings with specific row numbers
- Dutch number notation check correctly identifies `1.234` ambiguity

---

## Batch 8 -- Phase 5 + Phase 6 Checks (Consistency + Statistical, 4 checks)

**Goal:** Cross-field consistency checks and statistical anomaly detection. These are the "smart" checks that look at relationships between columns and statistical properties.

### Files to create

**`app/checks/phase5_consistency/c0501_date_order.py`** -- `DateOrderCheck`
- `check_id = "5.1"`, `phase = 5`, `formats = None`
- `execute`: requires `config.start_date_column` and `config.end_date_column` to be set.
  - If not configured, return INFO "Date order check not configured (no start/end date columns specified)".
  - For each row: parse both dates, check start <= end.
  - ERROR per row where end < start, with row number.

**`app/checks/phase5_consistency/c0502_amounts_add_up.py`** -- `AmountsAddUpCheck`
- `check_id = "5.2"`, `phase = 5`, `formats = None`
- `execute`: requires `config.price_column`, `config.quantity_column`, `config.total_column`.
  - If not configured, return INFO "P x Q check not configured".
  - For each row: parse P, Q, Total as floats. Check `abs(P * Q - Total) < tolerance` (tolerance = 0.01 or 1% of Total, whichever is larger -- accounts for rounding).
  - ERROR per row where they do not match, with expected vs actual.

**`app/checks/phase6_statistical/c0601_outlier_detection.py`** -- `OutlierDetectionCheck`
- `check_id = "6.1"`, `phase = 6`, `formats = None`
- `execute`: for each numeric column (from `ctx.column_types`):
  - Convert to float, drop NaN
  - If `config.outlier_method == "iqr"`: compute Q1, Q3, IQR. Outliers are values < Q1 - 1.5*IQR or > Q3 + 1.5*IQR.
  - If `config.outlier_method == "stddev"`: outliers are values > 3 standard deviations from mean.
  - Uses stats from `ctx.metadata[f"numeric_stats_{col}"]` if available (set by check 3.5).
- WARNING per column with outliers. Details: `{"outlier_count": 3, "method": "iqr", "bounds": {"lower": -500, "upper": 15000}, "extreme_values": [980000, 1200000, 2100000]}`

**`app/checks/phase6_statistical/c0606_homogeneity.py`** -- `HomogeneityCheck`
- `check_id = "6.6"`, `phase = 6`, `formats = None`
- `execute`: for each column:
  - Find the most frequent value and its percentage
  - If percentage > `config.homogeneity_threshold` (default 80%), WARNING
  - This catches default/placeholder value columns that look filled but contain no real information
- WARNING per column: `{"most_common_value": "1990-01-01", "percentage": 94.0, "count": 3760, "total": 4000}`

### Tests

**`tests/test_checks/test_phase5/__init__.py`** -- empty
**`tests/test_checks/test_phase6/__init__.py`** -- empty

- `test_c0501.py`: Start date after end date -> ERROR. No config -> INFO skip. Normal dates -> no error.
- `test_c0502.py`: P * Q does not equal Total -> ERROR. Rounding within tolerance -> no error. No config -> INFO skip.
- `test_c0601.py`: Column with obvious outlier (99% of values 1-100, one value 1000000) -> WARNING. All normal values -> no warning.
- `test_c0606.py`: Column with 95% same value -> WARNING. Column with good distribution -> no warning.

### Done criteria
- `pytest tests/test_checks/test_phase5/ tests/test_checks/test_phase6/` -- all pass
- Total checks discovered: 46 (10 + 13 + 7 + 12 + 2 + 2)
- Upload CSV with date order violations -> ERROR findings with row numbers

---

## Batch 9 -- Phase 7 Checks (AI/LLM, 2 checks)

**Goal:** Claude-powered semantic analysis. These checks require an API key and make external calls, so they need special handling for testing (mocking).

### Files to create

**`app/util/llm.py`**
- `class LLMClient`:
  - `__init__(self, api_key: str)`: creates `anthropic.Anthropic(api_key=api_key)`
  - `def analyze_columns(self, column_info: list[dict]) -> list[dict]`:
    - Sends column names + sample values (5 per column) to Claude
    - System prompt: "You are a data analyst. For each column, identify what the column likely contains based on its name and sample values. Return a JSON array."
    - Uses `claude-sonnet-4-20250514` (fastest model sufficient for this task)
    - Returns list of `{"column": name, "interpretation": "...", "confidence": "high|medium|low"}`
  - `def score_quality(self, stats_summary: dict, findings_summary: dict) -> dict`:
    - Sends Phase 3 statistics + Phase 4-6 findings counts to Claude
    - System prompt: "You are a data quality assessor. Given these statistics and validation findings, produce a quality score from 0 to 100 and a brief summary. Return JSON."
    - Returns `{"score": 74, "summary": "...", "strengths": [...], "issues": [...]}`

**`app/checks/phase7_ai/c0701_semantic_columns.py`** -- `SemanticColumnsCheck`
- `check_id = "7.1"`, `phase = 7`, `formats = None`
- `execute`:
  - If no API key configured (`settings.anthropic_api_key` is empty), return INFO "Skipped: no API key configured"
  - Build column_info: for each column, take name + 5 sample non-empty values
  - Call `LLMClient.analyze_columns(column_info)`
  - Return INFO per column with the interpretation
  - Catch API errors gracefully: return WARNING "LLM analysis failed: {error}"

**`app/checks/phase7_ai/c0705_quality_score.py`** -- `QualityScoreCheck`
- `check_id = "7.5"`, `phase = 7`, `formats = None`
- `execute`:
  - If no API key, return INFO "Skipped"
  - Build stats_summary from `ctx.column_types`, Phase 3 statistics
  - Build findings_summary: count of errors, warnings per phase from previous findings. The pipeline must pass accumulated findings somehow. Strategy: store running totals in `ctx.metadata["findings_summary"]` (the pipeline updates this after each phase).
  - Call `LLMClient.score_quality(stats_summary, findings_summary)`
  - Return INFO with score and summary
  - `statistics` field in PhaseResult should include the score

### Pipeline modification (update `app/engine/pipeline.py`)
After each phase completes, store a findings summary in `ctx.metadata`:
```python
ctx.metadata["findings_summary"] = {
    "total_errors": sum(...),
    "total_warnings": sum(...),
    "total_info": sum(...),
    "errors_by_phase": {...},
}
```
This allows check 7.5 to access accumulated findings.

### Tests

**`tests/test_checks/test_phase7/__init__.py`** -- empty

- `test_c0701.py`:
  - Test with no API key -> INFO "Skipped"
  - Test with mocked LLMClient (use `unittest.mock.patch`): mock returns interpretations -> INFO per column
  - Test with mocked API failure -> WARNING with error message
- `test_c0705.py`:
  - Test with no API key -> INFO "Skipped"
  - Test with mocked LLMClient: mock returns score 74 -> INFO with score
  - Verify the score appears in the finding details

### Done criteria
- `pytest tests/test_checks/test_phase7/` -- all pass (all tests use mocked API)
- Total checks discovered: 48 (10 + 13 + 7 + 12 + 2 + 2 + 2)
- Without API key: Phase 7 returns INFO "Skipped" findings, pipeline still produces a report
- With API key (manual test): Phase 7 produces semantic interpretations and a quality score

---

## Batch 10 -- Full Integration Tests + Polish

**Goal:** Comprehensive end-to-end tests proving the entire system works. Cover all four formats, error paths, and edge cases.

### Test fixtures to add

- `dutch_financial.csv`: realistic Dutch financial data. Semicolon-delimited, Latin-1 encoding, columns: `factuurnummer;datum;bedrag;btw;totaal;klant;status`. Include: Dutch number notation (1.234,56), dates in DD-MM-YYYY, some empty cells, one duplicate row, a placeholder "1900-01-01", and a leading-zero-stripped postcode.
- `multi_sheet.xlsx`: Excel with 3 sheets ("Facturen", "Klanten", "Leeg"), merged cells in header, one formula, one hidden column.
- `nested_api_response.json`: deeply nested JSON (4 levels), inconsistent keys, root is object with `{"data": [{...}, ...]}`.
- `simple_export.xml`: XML with `<export><row>...</row>...</export>` pattern.
- `edge_case_empty_columns.csv`: CSV where 2 of 5 columns are 100% empty.

### Test files to create

**`tests/test_integration/__init__.py`** -- empty

**`tests/test_integration/test_full_pipeline.py`**
- `test_csv_happy_path`: Upload `dutch_financial.csv` -> report covers all 7 phases. Verify:
  - Phase 1: encoding detected (Latin-1 WARNING), delimiter ";" detected
  - Phase 2: dimensions correct, no structural issues
  - Phase 3: column types detected (bedrag = number, datum = date, etc.)
  - Phase 4: Dutch number notation flagged, placeholder "1900-01-01" flagged, duplicate row found, leading zeros found
  - Status: APPROVED_WITH_WARNINGS

- `test_excel_with_issues`: Upload `multi_sheet.xlsx` -> report includes:
  - Phase 2: merged cells WARNING, formula WARNING, hidden column WARNING, manual editing WARNING (3 signals)

- `test_json_nested`: Upload `nested_api_response.json` -> 
  - Phase 1: root type "object" detected
  - Phase 2: nesting depth WARNING, inconsistent keys flagged

- `test_xml_basic`: Upload `simple_export.xml` ->
  - Phase 1: repeating element detected
  - Phase 2: structure validated

- `test_rejected_pipeline`: Upload `corrupt.xlsx` ->
  - Phase 1: FAILED (check 1.6 ERROR)
  - Phases 2-7: all SKIPPED
  - Report status: REJECTED

- `test_disabled_checks`: Upload CSV with `config.disabled_checks = ["4.17", "4.18"]` -> those findings absent from report

- `test_required_columns`: Upload CSV with `config.required_columns = ["factuurnummer"]` and a row with empty factuurnummer -> ERROR from check 4.2

- `test_pxq_check`: Upload CSV with price, quantity, total columns configured and one row where P*Q does not equal total -> ERROR from check 5.2

**`tests/test_integration/test_format_specific.py`**
- `test_csv_only_checks_skip_for_excel`: checks 1.5, 2.2 should not run for Excel (format filtering)
- `test_excel_only_checks_skip_for_csv`: checks 2.5, 2.6, 2.10, 2.11, 2.13 should not run for CSV
- `test_json_only_checks_skip_for_csv`: checks 1.10, 2.7, 2.8 should not run for CSV
- `test_xml_only_checks_skip_for_csv`: check 1.9 should not run for CSV

**`tests/test_integration/test_report_structure.py`**
- Validate that every report matches the JSON schema defined in VALIDATION_FRAMEWORK_COMPLETE_EN.md
- Every finding has a valid `check_id` matching a known check
- Every phase has a valid status
- Totals match the actual count of findings by severity

### Done criteria
- `pytest tests/` -- all tests pass (full suite)
- Registry discovers exactly 48 checks
- All four formats (CSV, Excel, JSON, XML) produce valid reports
- Corrupt files of all formats result in REJECTED
- Format-specific checks are correctly skipped for non-matching formats
- The `dutch_financial.csv` test catches all the Dutch-specific issues (notation, leading zeros, placeholder dates)

---

## Summary Table

| Batch | Focus | Files | Checks | Running Total | Key Milestone |
|-------|-------|-------|--------|---------------|---------------|
| 0 | Scaffold | 8 | 0 | 0 | FastAPI boots, health endpoint works |
| 1 | Models | 6 | 0 | 0 | All data contracts defined |
| 2 | Engine | 5 + 7 stubs | 0 | 0 | Pipeline runs with dummy checks |
| 3 | Parsing + Phase 1 | 4 parsers + 10 checks + 2 utils | 10 | 10 | CSV parsed end-to-end |
| 4 | Phase 2 | 13 checks | 13 | 23 | Section 1 complete (file-level) |
| 5 | API endpoint | 1 endpoint + update main | 0 | 23 | First curl works |
| 6 | Phase 3 | 7 checks | 7 | 30 | Column types detected |
| 7 | Phase 4 | 12 checks | 12 | 42 | Content validation working |
| 8 | Phase 5 + 6 | 4 checks | 4 | 46 | Consistency + statistics |
| 9 | Phase 7 | 2 checks + LLM util | 2 | 48 | All 48 checks implemented |
| 10 | Integration tests | 0 new production files | 0 | 48 | Full system validated |

## Key Dependency Chain

```
Batch 0 (scaffold)
  -> Batch 1 (models) -- everything imports these
    -> Batch 2 (engine) -- needs models
      -> Batch 3 (parsing + Phase 1) -- needs engine + models
        -> Batch 4 (Phase 2) -- needs Phase 1 to populate ctx
          -> Batch 5 (API) -- needs engine + at least some checks
            -> Batch 6 (Phase 3) -- needs ctx.dataframe from Phase 1
              -> Batch 7 (Phase 4) -- needs ctx.column_types from Phase 3
                -> Batch 8 (Phase 5+6) -- needs Phase 3+4 data
                  -> Batch 9 (Phase 7) -- needs all prior phases for scoring
                    -> Batch 10 (integration) -- validates everything
```

## Critical Design Notes for the Implementing Agent

1. **Always read CSV/Excel as `dtype=str` with `keep_default_na=False`.** This is the single most important design decision. If you let pandas auto-detect types, it will silently eat leading zeros (postcodes "0162" -> 162), convert Dutch numbers wrong (1.234 -> 1.234 instead of 1234), and replace empty strings with NaN (making it impossible to distinguish "truly empty" from "n/a"). All type detection happens explicitly in Phase 3.

2. **Check ordering within a phase matters.** Files are named `c0101`, `c0102`, etc., and the registry sorts by check_id. Within Phase 1: format must be detected (1.2) before encoding (1.4), encoding before delimiter (1.5), all before parsing (1.6). The naming convention enforces this.

3. **ctx.metadata is the escape hatch.** When a check produces data that another check needs (formula count for the manual editing composite check, numeric stats for outlier detection), store it in `ctx.metadata` with a clear key name. This avoids adding fields to FileContext for every inter-check dependency.

4. **Phase 7 must degrade gracefully.** No API key = INFO "Skipped", not ERROR. API failure = WARNING, not crash. The validation report must always be produced even if Claude is unavailable.

5. **Dutch specifics are first-class concerns.** The semicolon delimiter, Latin-1 encoding, DD-MM-YYYY dates, `1.234,56` number notation, and leading-zero-stripped postcodes/BSN are not edge cases -- they are the primary use case. Every relevant check must handle Dutch conventions as the default, not as a special case.

6. **Findings should be actionable.** Every finding must include enough detail for a human to find and fix the problem: row numbers in the `rows` field, column name in `column`, specific values in `details`. "12 invalid dates" is not enough; "12 invalid dates in column 'geboortedatum' at rows [14, 87, 103, ...]" is.

7. **The parsers in `app/parsing/` are invoked only from check 1.6.** They are not called directly by the API or the pipeline. This keeps parsing within the check architecture and ensures parse failures are reported as proper ERROR findings.

8. **For check 2.13 (manual editing signs), use `ctx.metadata` counts set by checks 2.6, 2.10, 2.11.** Those earlier checks should store `ctx.metadata["hidden_count"]`, `ctx.metadata["merged_count"]`, `ctx.metadata["formula_count"]`. Check 2.13 reads these rather than re-scanning the workbook.

### Critical Files for Implementation
- `/Users/daan/VS Studio/datapraat-checks/docs/IMPLEMENTATION_PLAN.md`
- `/Users/daan/VS Studio/datapraat-checks/docs/VALIDATION_FRAMEWORK_COMPLETE_EN.md`
- `/Users/daan/VS Studio/datapraat-checks/docs/PRODUCT_VISION.md`
- `/Users/daan/VS Studio/datapraat-checks/docs/20260804-initial-file-sanity-check-thom (1).md`
- `/Users/daan/VS Studio/datapraat-checks/docs/HARM-NICO-DATA-PHILOSOPHY.md`