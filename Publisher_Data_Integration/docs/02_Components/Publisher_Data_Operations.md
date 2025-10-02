# Component: Publisher_Data_Operations

**One-line purpose**  
Core library that validates, transforms and stages data from uploaded files into the Publisher system. Contains validation logic (Excel/Aspose), staging processing, business rule calculations, and helpers used by the PDI Azure Functions.

---

## What this component does (business view)

- Validates uploaded Excel/CSV files against templates and business rules.
- Transforms validated staging data into Publisher document records (pdi_Publisher_Documents).
- Applies business rules for document flags (age, active status, proxy handling, MER availability, performance reset).
- Provides helper utilities for file streams, logging, and database access used across PDI and Publisher import flows.
- Uses Aspose.Cells for Excel reading and template-driven validation.

---

## Key files & folders (where to look)

- `DocumentProcessing.cs` — maps staging rows into Publisher document records and writes updated rows back to DB.
- `FileIntegrityCheck.cs` — workbook validation using Aspose + `ValidationList` and `ExcelHelper`.
- `Orchestration.cs` — (core import orchestration — send next for analysis).
- `Generic.cs` — generic helpers & DB access wrappers (send next).
- Folders: `Aspose/`, `Extensions/`, `Helper/`, `Properties/` contain helpers, validators, and entity helpers.

---

## Important DB objects & stored-proc references (discovered)

- `sp_DataStagingPivot` — used to get a pivot of staging rows for a Job_ID (via `GetStagingPivotTable`).
- `pdi_Publisher_Documents` — target table updated by document processing.
- `Processing.UpdateFilingReferenceID` (method) — called to set a job's FilingReferenceID (calls underlying DB update).
- Note: Orchestration and Generic will contain further stored-proc / SP names (send those files next).

---

## Classes & Methods (detailed)

### `DocumentProcessing` — file: `Publisher_Data_Operations/DocumentProcessing.cs`

- **Purpose:** Read pivoted staging data for a `Job_ID`, map/update `pdi_Publisher_Documents` rows with business rules, and persist changes back to DB.
- **Constructors**
  - `DocumentProcessing(object sqlConnection, Logger log)` — initialize DB connection wrapper and logger.
- **Public methods (entry points)**
  - `processStaging(int jobID, string dataType, string docType)` — parse data/doc type names then call `processStaging(jobID)`.
  - `processStaging(int jobID, int dataType, int docType)` — accepts numeric type ids then call `processStaging(jobID)`.
  - `processStaging(int jobID, DataTypeID dataType, DocumentTypeID docType)` — sets private type fields and calls core `processStaging`.
  - `processStaging(int jobID, string fileName)` — infers types from filename (via `PDIFile`) and calls core `processStaging`.
  - `processStaging(PDIFile fn)` — accepts a `PDIFile` object (with JobID) and calls core `processStaging`.
- **Internal / core**
  - `processDataStaging(int jobID)` — (internal) main algorithm:
    - Calls `GetStagingPivotTable(jobID, _dbCon)` which invokes `sp_DataStagingPivot`.
    - Validates that the staging pivot contains a single Client/DocumentType/LOB combination.
    - Loads existing `pdi_Publisher_Documents` for the client into `_publisherDocs` DataTable.
    - For each staging row:
      - Skips aggregation rows (`Code == "All"`).
      - Finds matching existing document row or creates a new one.
      - Applies a long list of business field updates (flag changes, date logic, MER availability, FundCode fallback).
      - Uses `UpdateIfChanged(matchedDoc, column, value)` to only write changed values.
      - Computes/updates `Last_Updated`, `InceptionDate`, `Is120ConsecutiveMonths`, `PerformanceReset` logic, and FFDocAge via `SetFFDocAge`.
    - Persists changes with `_dbCon.UpdateDataTable(...)`.
- **Helpers within class**
  - `GetStagingPivotTable(int jobID, DBConnection dbCon)` — static helper that calls `sp_DataStagingPivot` and returns a DataTable.
  - `UpdateIfChanged(DataRow row, string column, object value)` — sets column value only if changed.
  - `SetFFDocAge(Dictionary<string,string> staging, DataRow matchedRow, DataTable pubDocs)` — complex business rule returning `FFDocAge?` for Fund Facts age status (brand new / new series / 12 months etc).

## Business impact / notes

- This class is the canonical place for many domain rules: FFDocAge calculation, FFDocAge and IsActiveStatus updates, PerformanceReset decisions, and multiple field mappings. Business analysts should review `fieldList` and `SetFFDocAge` logic to confirm policy alignment.

---

### `FileIntegrityCheck` — file: `Publisher_Data_Operations/FileIntegrityCheck.cs`

- **Purpose:** Validate the uploaded Excel workbook against a template and business validation rules using Aspose.Cells and `ValidationList` / `ExcelHelper`.
- **Constructors**
  - `FileIntegrityCheck(PDIStream processStream, PDIStream templateStream, object conn, Logger log = null)` — sets Aspose license, initializes a `ValidationList` (with `ExcelHelper`), logs initial invalid-file conditions, and writes initial validation errors to DB.
- **Public methods**
  - `bool FileCheck()` — main validation routine:
    - Loads the input workbook and template workbook using Aspose with memory-optimized settings.
    - Validates presence of required sheets (logs missing sheets unless marked optional).
    - Checks for macros and other static-updates via `validationList.ExHelper.CheckStaticUpdate`.
    - Adds additional worksheets mapping for "Table UPDATE" patterns via `AsposeLoader.TableUpdate`.
    - Calls `validationList.Validate()` to run configured worksheet checks.
    - Writes validation issues/errors to DB and returns whether data is valid (`true/false`).
- **Business impact / notes**
  - If template stream is empty or missing, a critical validation error is logged and file may be rejected or flagged.
  - Aspose license usage is required (`Aspose.Total.lic`) — ensure licensing is provisioned in QA/Prod.
  - Validation writes additive errors back to the source stream when invalid (used for diagnostics).

---

## QA validation / quick tests (component)

1. Run validation test:
   - Provide a known-good template and sample file that should pass. Run `FileIntegrityCheck.FileCheck()` and verify `validationList.ExHelper.IsValidData == true`.
2. Run staging process test:
   - Use a test `Job_ID` with sample staging rows. Run `DocumentProcessing.processStaging(jobID, DataTypeID, DocumentTypeID)` and verify `pdi_Publisher_Documents` rows are created/updated as expected.
3. Check FFDocAge logic:
   - Provide controlled staging values and confirm `SetFFDocAge` returns expected enum values for each business case.
4. Aspose memory behavior:
   - Test with larger Excel files to ensure Aspose memory settings are acceptable for function environment.

---

## Risks & suggested follow-ups

- **Business-rule audit:** `SetFFDocAge` and the `fieldList` updates implement domain logic—BA should review these lines to confirm regulatory intent and edge cases.
- **Aspose licensing:** `Aspose.Total.lic` is required and must be provisioned for QA/Prod. Missing license may change behavior.
- **Performance:** `processDataStaging` loads an entire `pdi_Publisher_Documents` table for the client into memory; for large clients this may be heavy — consider pagination or targeted queries if needed.
- **Unit tests:** The presence of `Publisher_Data_Operations_Tests` is valuable—add/expand tests around FFDocAge, UpdateIfChanged, and FileCheck to guard refactors.
- **Orchestration dependency:** `DocumentProcessing` calls into `Processing.UpdateFilingReferenceID` and relies on `_dbCon` helpers — we need `Orchestration`/`Generic` to map exact SPs & DB schema references.

---

## Where to look (files)

- `Publisher_Data_Operations/DocumentProcessing.cs`  
- `Publisher_Data_Operations/FileIntegrityCheck.cs`  
- `Publisher_Data_Operations/Extensions/` and `Helper/` folders for `ExcelHelper`, `ValidationList`, `AsposeLoader` etc.  
- `Publisher_Data_Operations/Orchestration.cs` (next — contains DB import flow and stored-proc usage).

---

### `Orchestration` — file: `Publisher_Data_Operations/Orchestration.cs`

- **Purpose:** The end-to-end engine that controls validation, extraction, transformation, staging load and (when allowed) import into Publisher. Coordinates the `Processing` lifecycle, interacts with DB connections for both PDI and Publisher, and writes logs/notifications.

- **Constructors**
  - `Orchestration(object con, object con2 = null)`  
    Initialize PDI DB connection and optional Publisher DB connection.
  - `Orchestration(object con, object con2, PDIStream processFile)`  
    Initialize DB connections and set an incoming `PDIStream` for processing (sets `FileID` if available).
  - `Orchestration(object con, object con2, string fileID)`  
    Initialize and set `FileID` from string (creates Logger for that file).

- **Key public properties**
  - `int FileID` — internal file identifier persisted in DB.
  - `string ErrorMessage` — last error message recorded.
  - `Guid RunID` — run identifier (present but not assigned in current code).
  - `int RetryCount` — retry counter for transient retry logic.
  - `string NotificationEmailAddress` — fetched from `pdi_Publisher_Client`.
  - `string FileRunID` — forwarded from `PDIStream.PdiFile.FileRunID`.
  - `bool FileStatus` — combined validity flag that includes file validation results.

- **Main public methods (entry points)**
  - `bool ProcessFile(PDIStream processStream, PDIStream templateStream, int retryCount, ILogger logger = null)`  
    Primary entry used by the Azure Functions. Sets `FileID` (calls `ProcessAfterLoadOnly()` or `Retry()`), stores streams, and calls `InternalProcessFile`.
  - `bool ProcessFile(string processFile, string templatePath, int retryCount = 0)` *(obsolete; local testing only)*  
    Runs local test path (constructs PDIFile/PDIStream from path).
  - `void HandleEventMessage(EventGridEvent eventGridEvent)`  
    Handles Event Grid events (example: `CreateCustomerEvent`) — inserts client & company rows into Publisher/PDI DBs and creates related LOB rows.
  - `bool PublisherImport(int jobID, ILogger logger = null)`  
    Final import flow called when local-run or when publishing step is triggered immediately: creates temp table, bulk-copies transformed data, calls import SP(s), triggers final set-document-name step (for STATIC data) and sets processing stage final states.
  - `bool LoadStoredProcedure(int jobID, string spName = "sp_LoadTransformedData")`  
    Wrapper to call a stored proc to load transformed data.
  - `bool ImportStoredProcedure()` / `ImportStoredProcedure(PDIFile pdiFile)` / `ImportStoredProcedure(string jobID, string companyID, string docTypeID)` / `ImportStoredProcedure(int jobID, int companyID, int docTypeID)`  
    Overloads that call `sp_pdi_IMPORT_DATA` (or variants) to import staged data into Publisher DB.
  - `bool CheckImportSP()`  
    Verifies presence of `sp_pdi_IMPORT_DATA_temp` in PUB DB; if missing attempts to recreate via `StoredProcedure.ImportDataSP()`.
  - `void ClearImportSP()`  
    Clears staging stored procedure records (used for staging env maintenance).
  - `bool LoadTempTable(DBConnection dbConPub, int jobID, bool keepTable = false)`  
    Creates a (temporary or named) table in Publisher DB, queries `pdi_Transformed_Data` + joins to build a data table, and bulk copies it into the temp table for import.
  - `bool ImportTempTable(DBConnection dbConPub)`  
    Calls `sp_pdi_IMPORT_DATA_temp` to ingest the temp table into Publisher DB.
  - `bool SetDocumentName()`  
    For STATIC data: obtains document name field/template mapping and calls `sp_UPDATE_DOCUMENT_NAME` on PUB DB.
  - `bool Cleanup(int jobID)`  
    Deletes staging/transformed/validation rows in PDI DB tied to the job to support retry/cleanup paths.
  - `string LoadNotifiactionEmail()`  
    Loads `Notification_Email_Address` for the client from `pdi_Publisher_Client` if not already present.

- **Internal workflow (what `InternalProcessFile` does)**
  1. Initialize `Logger` and `Processing` state; set `NotificationEmailAddress`.
  2. Validate `PDIStream` exists and size > 0.
  3. If `RetryCount > 0`, call `Cleanup(jobID)` to clear prior data for retry.
  4. If data type is *not* `FSMRFP`, perform `FileIntegrityCheck` (Aspose-based validation). If validation fails, update processing stage and stop.
  5. Run `Extract` to populate staging tables from validated file. If extraction fails, mark error and stop.
  6. Run `DocumentProcessing.processStaging` to update `pdi_Publisher_Documents`.
  7. Run `Transform.RunTransform` to generate `pdi_Transformed_Data`. If transform fails, error.
  8. If running locally (`_runningLocal`) call `PublisherImport(jobID)` to bulk-load and call import SPs; else set processing stage to `Import_Ready` and let downstream import runner handle import.

- **Lifecycle & disposal**
  - Implements `IDisposable`. `Dispose()` disposes `Logger` and clears references.

- **Event handling**
  - `HandleEventMessage` parses `CreateCustomerEvent` Event Grid event and performs database inserts/updates in both PDI and Publisher DBs; note: the method swallows exceptions without logging — see Risks.

### `Generic` — file: `Publisher_Data_Operations/Generic.cs`

- **Purpose (one-line):** Utility and domain helper class used across the pipeline for text lookups, language/translation logic, date formatting, publisher-document field access, scenario matching, status table queries and small data-caching for per-client operations.

- **Important constants / semantics**
  - `MISSING_EN_TEXT`, `MISSING_FR_TEXT` — markers prepended when translations are missing.
  - `DAYSPERYEAR` — used for any year/day calculations.
  - `FLAGPRE/FLAGPOST`, `TABLEDELIMITER` — token syntax used when replacing tokens in templates or parsing table-like fields.
  - `FSMRFP_ROWTYPE_COLUMN` — constant row-type name used by FSMRFP processing.

- **State / cached DataTables (per instance)**
  - `GlobalTextLanguage`, `ClientScenario`, `PublisherDocuments`, `ClientTranslation`, `MissingFrench`, `MissingFrenchDetails`, `DocumentTemplates`.
  - These are lazy-loaded and reloaded when ClientID/LobID/DocTypeID change — used to reduce DB calls for repeated lookups.
  - Note: instance-level caching means one `Generic` instance should be scoped per job/flow to avoid stale data across clients.

- **Constructor**
  - `Generic(object connectionObject, Logger log)` — accepts `DBConnection` or raw connection object, stores a `Logger`.

- **Key public methods (what they do + where used)**
  - `string[] searchGlobalScenarioText(string scenario)`  
    Returns English/French pair for a global translation/label (from `pdi_Global_Text_Language`).

  - `Tuple<string,string,string> searchClientScenarioText(RowIdentity rowIdentity, string fieldName, Dictionary<string,string> stageFields = null)`  
    Overload that resolves to client-specific scenario lookup. Matches best scenario based on flags and returns tuple: (flagList, EnglishText, FrenchText). Used in template token replacement for STATIC-type content.

  - `Tuple<string,string,string> searchClientScenarioText(int documentTypeID, int clientID, int lobID, string fieldName, string documentNumber, Dictionary<string,string> stageFields = null)`  
    Core scenario matching: loads `pdi_Client_Field_Content_Scenario_Language` for the client+lob+docType and finds best matching scenario by ranking.

  - `string[] assembleDateText(DateTime theDate, string[] monthNames)` / `string[] assembleDateText(string rawData, string[] monthNames)`  
    Builds long-form English/French date text (e.g., "January 1, 2020" / localized French).

  - `string[] longFormDate(string rawDate, Dictionary<string,string[]> dateNames = null, string prefix = "")`  
    Returns localized long-form date by looking month names up in `pdi_Global_Text_Language`.

  - `string[] shortFormDate(string rawDate, Dictionary<string,string[]> dateNames = null)` and `shortFormDateUS(...)`  
    Short date formats with localization support.

  - `Dictionary<string,string[]> loadMonths(bool shortForm = false)` / `loadShortMonths()`  
    Loads month name translations (uses `searchGlobalScenarioText` for each month code).

  - `bool usePrelimDateFF4(int documentTypeID, int clientID, int lobID, string documentNumber)` / `usePrelimDate(RowIdentity)`  
    Business decision helper to decide whether to use Prelim or Filing date for specific FF rules. Uses `getPublisherDocumentFields` internally.

  - `string getPublisherDocumentField(int documentTypeID, int clientID, int lobID, string documentNumber, string fieldName)`  
    Returns a single field value (string) from `pdi_Publisher_Documents` using cached `PublisherDocuments` DataTable.

  - `Dictionary<string,string> getPublisherDocumentFields(int documentTypeID, int clientID, int lobID, string documentNumber, List<string> fieldNames, bool convertDates = false, bool french = false)`  
    Loads `pdi_Publisher_Documents` for the specified client/docType/lob once per instance and returns requested columns as strings. Booleans normalized to "1"/"0".

  - `string searchClientFrenchText(string english, RowIdentity rowIdentity)`  
    Uses `pdi_Client_Translation_language` to find the French equivalent for a client-specific English string.

  - `string verifyFrenchTableText(string french, string english, RowIdentity rowIdentity, int jobID, string fieldName)`  
    If French is missing, look up French using English; if still missing, register missing-French entry and return `MISSING_FR_TEXT + english`.

  - `string GenerateFrench(string english, RowIdentity rowIdentity, int jobID, string fieldName)`  
    High-level utility that can handle `<cell>...</cell>` structured text and call `SearchFrench` for pieces; constructs composite French text and handles numeric/date patterns.

  - `void AddMissingFrench(string english, RowIdentity rowIdentity, int jobID, string fieldName)`  
    Adds a missing-French record into the cached `MissingFrench` and `MissingFrenchDetails` DataTables (in-memory). Used by `verifyFrenchTableText`.

  - `bool SaveFrench()`  
    Persists any accumulated missing-french tables to DB via `dbCon.BulkCopy` into `pdi_Client_Translation_Language_Missing_Log` and details table.

  - `Tuple<string,string> SearchFrench(RowIdentity rowIdentity, string english, int jobID, string fieldName)`  
    Lookup logic that handles several special cases (dates, numeric patterns with %, currency, numeric + parenthesis), fallback to `verifyFrenchTableText`. Returns tuple (english, french).

  - `void AddStaticFields(Dictionary<string,string> tokensEN, Dictionary<string,string> tokensFR, List<string> extras)`  
    Adds calculated token fields such as long-form dates or filing year variants used in template substitution.

  - `string[] PrepareStaticTypeFields(RowIdentity rowIdentity, string fieldName, Dictionary<string,string> tokenListEN, Dictionary<string,string> tokenListFR, Dictionary<string,string> rowData = null, int jobID = -1)`  
    Handles STATIC2 (token replacement) and STATIC3 (table rows) content generation by combining `searchClientScenarioText`, `tokenList` replacements, and building HTML `<table>` rows when needed.

  - `bool CheckDocumentTemplates(string clientCode, string documentType, out bool active)`  
    Validates presence of document templates in `pdi_Publisher_Document_Templates` cache and returns active flag.

  - `static string MakeTableString(string value)` and `MakeTableString(Byte[] value)`  
    Ensures table-type fields are wrapped into a `<table>` if not already.

  - `static string GetInceptionDate(Dictionary<string,string> documentFields, string altDateField = null)`  
    Returns the best inception-like date considering `PerformanceResetDate`/`InceptionDate`/alt date/`FilingDate`/`Last_Filing_Date`.

  - `Dictionary<string,string> loadValidParameters()`  
    Loads valid parameter list from `ExcelHelper.loadValidParameters(dbCon)` used by transforms/templates.

  - `DataTable GetStatusTable(int offSet, int pageSize = 10)`  
    Returns the set of recent file/job rows used by Status page; it runs a paged SQL query joining `pdi_File_Receipt_Log`, `pdi_File_Log`, `pdi_Processing_Queue_Log` and validation counts.

  - `DataTable GetValidationMessages(string fileNameOrID)`  
    Returns `pdi_File_Validation_Log` rows for a file id or latest file with matching filename.

## Business impact / note

- `Generic` centralizes translation and scenario logic — any change here affects how templates are built for STATIC content and how missing translations are captured.
- The class uses in-memory DataTables cached per client/docType/lob to avoid repeated DB queries. For very large clients these caches may be heavy — see performance notes below.
