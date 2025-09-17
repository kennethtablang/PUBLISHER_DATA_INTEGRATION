# Extensions — Allocation (analysis)

## One-line purpose

Provide a domain structure and helpers to build "allocation" table content (fund-level table rows) and map them into the PDI data staging format. Used during Transform/Extract to convert Excel table-like allocation sections into `pdi_Data_Staging` rows for downstream processing.

---

## Files analyzed

* `Publisher_Data_Operations/Extensions/Allocation.cs`

---

## What this code contains (high level)

1. `AllocationItem` — represents a single allocation row or header cell for a fund.
2. `AllocationTable` — groups `AllocationItem` instances per `FundCode`.
3. `AllocationList` — collection of `AllocationTable` objects with routines to build, order and output those allocations into a `DataTable` that matches the expected staging shape.

This is purely an in-memory helper to build the rows that the pipeline later inserts into the staging table.

---

## Classes, properties and methods (developer reference)

### `AllocationItem` - Class

**Purpose:** unit of allocation content (cell or header) containing English/French text and an optional `FieldName` mapping (when a field already exists in `pdi_Publisher_Documents`).

#### Properties

* `string Allocation` — raw allocation identifier / label from input (may contain markup/tags).
* `string EnglishText` — English content (table HTML or text).
* `string FrenchText` — French content (table HTML or text).
* `string FieldName` — mapped target field name if previously matched (e.g. "Static30h").
* `string EnglishHeader` — header text (EN) for this allocation item.
* `string FrenchHeader` — header text (FR).
* `string MatchText` — computed field used for fuzzy/normalized comparisons; set in ctor to `allocation.StripTagsExceptAlpha()`.

#### Methods

* `AllocationItem(...)` — constructor sets props and computes `MatchText`.
* `int FieldNumber()` — extracts the numeric portion of `FieldName` (returns parsed integer or -1).
* `bool Equals(AllocationItem other)` — equality based on `FieldName`.
* `int CompareTo(AllocationItem other)` — sort by `FieldName` with blank/null values sorted to the end.

#### Notes / Behaviour

* `MatchText` is used to find equivalent allocation entries even if tags differ. `StripTagsExceptAlpha()` is a helper in `Extensions` (not shown here).

---

### `AllocationTable` - Class

**Purpose:** groups allocation rows for a single `FundCode`.

#### Properties of `AllocationTable`

* `string FundCode` — identifier for the fund/series.
* `List<AllocationItem> Allocation` — list of allocation items for this fund.

#### Methods of `AllocationTable`

* `AllocationTable(string fundCode, string allocation, string englishText, string frenchText, string fieldName = null, string englishHead = null, string frenchHead = null)` — create with a first AllocationItem.
* `void ExistingOrder()` — sorts `Allocation` using `AllocationItem.CompareTo`.
* `Queue<AllocationItem> MatchedQueue()` — queue of AllocationItems where `FieldName` is populated.
* `Queue<AllocationItem> UnMatchedQueue()` — queue of AllocationItems without `FieldName`.

---

### `AllocationList : List<AllocationTable>` - Class inheriting from `AllocationTable`

**Purpose:** main helper used during transform to collect allocation entries from the input and then append them to the staging `DataTable` in a predictable order and field naming schema.

#### Constructor

* `AllocationList(object con, PDIFile fileName, Logger log)` — needs a DB connection/context wrapper (or connection string) via `con` plus `PDIFile` and `Logger`. Internally constructs a `Generic` helper which is used for French lookups.

#### Important fields

* `_gen` — `Generic` helper for translations & lookups.
* `_fileName` — `PDIFile` containing metadata: ClientID, LOBID, DocumentTypeID, JobID.
* `_clientID`, `_lobID`, `_docTypeID`, `_jobID` — extracted IDs used for translation/context.

#### Key public methods

* `void AddCode(string fundCode, string allocation, string englishText, string frenchText, string fieldName = null, string englishHead = null, string frenchHead = null, string altFieldName = null)`

  * Adds a new allocation row to a `AllocationTable` for `fundCode`. If `fundCode` exists it will merge the allocation entry.
  * Uses `_gen.verifyFrenchTableText(...)` to populate `frenchText` and header translations when `frenchText`/`frenchHead` are blank.
  * Calls `Generic.MakeTableString(...)` to ensure table-like content is wrapped as `<table>...</table>` where appropriate.

* `void OrderAll()` — sorts allocation items inside each `AllocationTable` and sorts tables by `FundCode`.

* `string GetFieldName(int index, string prepend, int offset = 30)` — helper that returns field name like `${prepend}{index+offset}` (e.g. `STATIC30`).

* `void AppendToDataTable(DataTable dt, int jobID, string sheetName, string dataType, bool overrideHeaders = false, int offset = 30)`

  * The main export routine. For each `AllocationTable` it builds two queues (matched/unmatched) and loops through 9 possible field indices (0..8).
  * For each index it computes a generated field name (`GetFieldName`) and decides whether to take a `matchedQueue` item (which maps to an existing FieldName in Publisher) or use an `unMatchedQueue` item.
  * Adds rows to the provided `DataTable` with the following columns (order of values as added):

    1. `jobID`
    2. `at.FundCode`  (Document / FundCode)
    3. `sheetName`
    4. `fieldName` (e.g. `STATIC30_EN`, `STATIC30_FR`, `STATIC30h_EN`)
    5. `i` (index sequence)
    6. `0` (unknown flag/placeholder; kept as `0` in current code)
    7. `content` (English or French table/text, or `Transform.EmptyTable` / `Transform.EmptyText`)
  * When no `AllocationItem` is available it writes default `Transform.EmptyTable` / `Transform.EmptyText` placeholders.
  * Logs errors when queue counts remain after filling.

**Notes on `AppendToDataTable`**

* The routine assumes a specific `DataTable` structure (column order and meaning). You **must confirm** the exact schema expected by the caller (likely `pdi_Data_Staging` or an intermediate staging table used by `Transform`). I list a likely mapping below.
* The code loops `i` from 0 to 8 (9 slots) — this is the maximum number of allocation columns per fund the system expects.

---

## Data flow / integration points (where this is used)

* `AddCode(...)` is typically called by the ETL `Transform` or `Extract` logic when it finds an "allocation" block in the input workbook (e.g., an asset allocation table for a fund).
* After populating the `AllocationList`, `AppendToDataTable(...)` is called to append rows to the in-memory staging `DataTable` which the pipeline later persists into `pdi_Data_Staging`.
* Later steps (Transform/Orchestration.Import) read `pdi_Transformed_Data` / temp table and import into Publisher.

**Action item**: search the repository for `AddCode(` and `AppendToDataTable(` usages to confirm callers and the exact `DataTable` column schema.

---
