# Publisher_Data_Operations

---

## pdi_Global_Text_Language

**Source:** `Generic.searchGlobalScenarioText`, `loadMonths`  
**Meaning:** Global lookup table for commonly used English/French phrases (month names, short/long forms, other standardized labels). Used for localized date and wording generation.

---

## pdi_Client_Field_Content_Scenario_Language

**Source:** `Generic.searchClientScenarioText`  
**Meaning:** Client-specific scenario table mapping `Field_Name` + scenario flags to English/French content templates. Scenario-ranking logic picks the best match and drives scenario-based content generation.

---

## pdi_Client_Translation_language

**Source:** `Generic.searchClientFrenchText`  
**Meaning:** Per-client table storing known English→French translations. Used for direct lookups; if a translation is missing, missing-French flow is triggered.

---

## pdi_Client_Translation_Language_Missing_Log` / `pdi_Client_Translation_Language_Missing_Log_Details`

**Source:** `Generic.AddMissingFrench`, `Generic.SaveFrench`
**Meaning:** Log tables that track missing French translations discovered during import/transform. The system accumulates missing entries and bulk-inserts them for BA/translation teams to act upon.

---

## pdi_Publisher_Documents

**Source:** `Generic.getPublisherDocumentFields`  
**Meaning:** Publisher catalog table accessed to get document-level metadata (InceptionDate, FFDocAgeStatusID, IsProforma, Document names). `Generic` caches client-specific rows to serve field lookups used by transforms and FF business rules.

---

## MissingFrench (in-memory)

**Source:** `Generic.LoadMissingFrench` / `AddMissingFrench`  
**Meaning:** In-memory representation of missing translation items, deduplicated via a UniqueConstraint then persisted via `SaveFrench()`.

---

## PDIBatch

**Source:** `Publisher_Data_Operations` (class used by PDI functions)  
**Meaning:** Server-side object that creates/loads/manages batch metadata and status (e.g., Count, SetComplete, MarkEntryExtracted). Used for batch lifecycle management and notifications.

---

## PDIStream

**Source:** `Publisher_Data_Operations`  
**Meaning:** Wrapper around a stream containing a blob's data plus file metadata. Passed into `Orchestration` for validation, transformation and import. Also contains helpers like `OnlyFileName` and `GetDefaultTemplateName`.

---

## DataTransferObject (DTO)

**Source:** `PDI_Azure_Function/DataTransferObjects.cs`  
**Meaning:** Lightweight message payload placed on Azure queues to tell downstream services which file to process and include context such as `FileName`, `Batch_ID`, `Job_ID`, and `RetryCount`. DTOs are Base64-encoded JSON when placed on Azure queues.

**Key fields**:

- `FileName` — blob filename placed in `processing/` or `importing/`.
- `Batch_ID` — batch identifier for grouped uploads.
- `Job_ID` — job identifier used by Publisher import.
- `RetryCount` — number of times pipeline retried this file.
- `ErrorMessage` — textual error to include in notifications.
- `NotificationEmailAddress` — optional override for who receives notifications.

---

## Orchestration

**Source:** `Publisher_Data_Operations/Orchestration.cs`  
**Meaning:** Central import engine that validates, transforms and inserts file contents into Publisher. Main operations are:

- `ProcessFile(PDIStream curStream, PDIStream templateStream, int retryCount, ILogger log)` — validate and stage file data.
- `PublisherImport(int jobID, ILogger log)` — final import step for Publisher (called by `QueueImport`).

---

## PDISendGrid / Notification

**Source:** `Publisher_Data_Operations` wrappers used by the functions  
**Meaning:** Notification utility that sends template-based emails on success or error to configured addresses (uses SMTP / SendGrid credentials from environment).

---

## AllocationItem

**Source:** `Publisher_Data_Operations/Extensions/Allocation.cs`
**Meaning:** Represents a single allocation cell (or header) for a fund. Holds English/French content, optional mapping to an existing field name in the Publisher schema, and a normalized match key used to detect duplicates or merges.

**Key fields:**

- `Allocation` — original allocation label/string passed to the constructor (raw source text).
- `EnglishText` — English content for the allocation cell (may contain HTML table markup).
- `FrenchText` — French content for the allocation cell (may contain HTML table markup).
- `FieldName` — optional mapped field name (e.g., Publisher field ID). `null` or empty means “unmapped”.
- `EnglishHeader` — English header/title for the allocation item.
- `FrenchHeader` — French header/title for the allocation item.
- `MatchText` — normalization of `Allocation` computed at construction with `StripTagsExceptAlpha()`; used for matching allocations ignoring markup.

**Key methods / behaviour:**

- `AllocationItem(string allocation, string englishText, string frenchText, string fieldName = null, string englishHead = null, string frenchHead = null)` — constructor; sets fields and computes `MatchText`.
- `int FieldNumber()` — returns numeric portion extracted from `FieldName` (digits only). Returns `-1` if none.
- `bool Equals(AllocationItem other)` — equality based on `FieldName` equality.
- `int CompareTo(AllocationItem other)` — sorts by `FieldName`; treats `null`/empty as greater (puts them at the end).

---

## AllocationTable

**Source:** `Publisher_Data_Operations/Extensions/Allocation.cs`
**Meaning:** Grouping of multiple `AllocationItem` objects for a single fund (`FundCode`). Represents the set of allocation rows/headers for that fund.

**Key fields:**

- `FundCode` — the fund identifier (string) this table belongs to.
- `Allocation` — `List<AllocationItem>`: the items (cells/headers) for this fund.

**Key methods / behaviour:**

- `AllocationTable(string fundCode, string allocation, string englishText, string frenchText, string fieldName = null, string englishHead = null, string frenchHead = null)` — constructor; creates a table with one `AllocationItem`.
- `void ExistingOrder()` — sorts `Allocation` using `AllocationItem.CompareTo()` (used to stable-order items before export).
- `Queue<AllocationItem> MatchedQueue()` — returns queue of items where `FieldName` is present (mapped).
- `Queue<AllocationItem> UnMatchedQueue()` — returns queue of items where `FieldName` is absent (unmapped).

---

## AllocationList

**Source:** `Publisher_Data_Operations/Extensions/Allocation.cs`
**Meaning:** A collection of `AllocationTable` objects with helpers to build allocations from source content and to export them into the staging `DataTable` shape used by the ETL (Transform). Also performs client/fund-specific French lookup via `Generic`.

**Constructor / context:**

- `AllocationList(object con, PDIFile fileName, Logger log)` — creates an instance and initializes a `Generic` helper (`con`) and extracts `ClientID`, `LOBID`, `DocumentTypeID`, `JobID` from the given `PDIFile`. `log` is used for error logging.

**Key methods / behaviour:**

- `void AddCode(string fundCode, string allocation, string englishText, string frenchText, string fieldName = null, string englishHead = null, string frenchHead = null, string altFieldName = null)`

  - Adds or merges an `AllocationItem` into the `AllocationTable` for `fundCode`.
  - If `englishHead` is blank, sets `englishHead = allocation` and uses `_gen.verifyFrenchTableText(...)` to derive `frenchHead`.
  - If `englishText` present, uses `_gen.verifyFrenchTableText(...)` to set `frenchText` and wraps `EnglishText`/`FrenchText` using `Generic.MakeTableString(...)`.
  - Matching/updating existing items uses `allocation.StripTagsExceptAlpha()` compared to existing `MatchText`.

- `void OrderAll()` — sorts each table’s items (`ExistingOrder()`) and sorts the list by `FundCode`.

- `string GetFieldName(int index, string prepend, int offset = 30)` — deterministic field-name generator used by export. Implementation: `prepend + (index + offset).ToString()`.

  - Example: `GetFieldName(0, "STATIC")` ⇒ `"STATIC30"` (default `offset=30`).

- `void AppendToDataTable(DataTable dt, int jobID, string sheetName, string dataType, bool overrideHeaders = false, int offset = 30)` — exports allocations into the provided DataTable in the exact shape expected by downstream ETL. This method is the canonical source for the staging row format:

  - Steps (literal behavior from code):

    1. Calls `OrderAll()` then for each `AllocationTable` builds `matchedQueue` (items with `FieldName`) and `unMatchedQueue` (items without).
    2. For `i` from `0` to `8` (nine slots) compute `genFieldName = GetFieldName(i, dataType, offset)`.
    3. Dequeue any matched items with `FieldNumber() < i + offset` (these are logged as errors).
    4. Select `curItem`:
       - If `matchedQueue.Peek().FieldName == genFieldName + "h"`, use that matched item.
       - Else if `unMatchedQueue` has items use next unmatched item.
       - Else `curItem = null`.
    5. If `curItem == null`, append four explicit rows to `dt` (literal code):
       - `{ jobID, FundCode, sheetName, genFieldName + "_EN", i, 0, Transform.EmptyTable }`
       - `{ jobID, FundCode, sheetName, genFieldName + "_FR", i, 0, Transform.EmptyTable }`
       - `{ jobID, FundCode, sheetName, genFieldName + "h_EN", i, 0, Transform.EmptyText }`
       - `{ jobID, FundCode, sheetName, genFieldName + "h_FR", i, 0, Transform.EmptyText }`
    6. If `curItem != null`, the code:
       - Appends English/French content rows for `genFieldName + "_EN"` / `genFieldName + "_FR"` **only if** `curItem.EnglishText` is not blank.
       - Appends header rows for `genFieldName + "h_EN"` / `genFieldName + "h_FR"` when `overrideHeaders` is `true` **or** `genFieldName + "h" != curItem.FieldName`. Header values are `curItem.EnglishHeader` / `curItem.FrenchHeader`.
    7. After finishing a fund, logs an error if queues still contain leftover items.
  - **Important concrete constants from code**:
    - Loop index bounds: `i = 0..8` (inclusive) — *nine* positions per fund.
    - Default `offset = 30`.
    - Uses placeholders `Transform.EmptyTable` and `Transform.EmptyText` for empty content/header cells.

**Error handling / logging:** The method logs leftover queue items and unexpected matched items via `Logger.AddError(_log, ...)` — these exact log calls are present in the code.

---
