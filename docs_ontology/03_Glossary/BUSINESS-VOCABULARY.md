# Business Vocabulary / Glossary (Publisher Data Integration)

This document defines key terms, objects, and paths used in the Publisher Data Integration (PDI) system. It is intended to help operations, support, and development teams understand the components and workflow.

---

### Batch file (Batch)

**Source:** `PDI_Azure_Function/BatchBlob.cs`  
**Meaning:** An uploaded package (zip) or a single Excel file that contains one or more document jobs for Publisher. The system registers a BatchID to track all entries in the package.  
**Example:** `funds_upload_20250901.zip` or `invest_20250901_001.xlsx`  
**Related objects:** `BatchID`, `processing/{filename}`, `archive/{name}`

---

### PDIBatch

**Source:** `Publisher_Data_Operations` (class used by PDI functions)  
**Meaning:** Server-side object that creates/loads/manages batch metadata and status (e.g., Count, SetComplete, MarkEntryExtracted). Used for batch lifecycle management and notifications.

---

### PDIStream

**Source:** `Publisher_Data_Operations`  
**Meaning:** Wrapper around a stream containing a blob's data plus file metadata. Passed into `Orchestration` for validation, transformation and import. Also contains helpers like `OnlyFileName` and `GetDefaultTemplateName`.

---

### DataTransferObject (DTO)

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

### Orchestration

**Source:** `Publisher_Data_Operations/Orchestration.cs`  
**Meaning:** Central import engine that validates, transforms and inserts file contents into Publisher. Main operations are:

- `ProcessFile(PDIStream curStream, PDIStream templateStream, int retryCount, ILogger log)` — validate and stage file data.
- `PublisherImport(int jobID, ILogger log)` — final import step for Publisher (called by `QueueImport`).

---

### PDISendGrid / Notification

**Source:** `Publisher_Data_Operations` wrappers used by the functions  
**Meaning:** Notification utility that sends template-based emails on success or error to configured addresses (uses SMTP / SendGrid credentials from environment).

---

### Template file

**Source:** Blob container path `templates/{templateName}`  
**Meaning:** Predefined template used during processing to render or validate content. If template is missing, processing logs a critical error and the job may fail.

---

### Status page endpoints (quick map)

- `GET /api/Status/{offSet?}` — status HTML page showing blobs, queues and recent jobs.
- `GET /api/DownloadBlob/{fileName}` — downloads a blob as an attachment.
- `GET /api/ValidationMessages/{fileName}` — shows validation errors for a file.
- `POST /api/FileUpload` — upload a file via the status page form (web UI).

---

### BlobServiceExtensions (Extensions)

**Source:** `PDI_Azure_Function/Extensions/BlobServiceExtensions.cs`  
**Meaning:** Utility helpers for copying, moving, saving and opening blobs and for producing a small HTML listing of container contents. These methods centralize blob copy/move semantics (including waiting for server-side copy completion) so the rest of the pipeline can move files across `processing/`, `importing/`, `completed/`, `rejected/` and `archive/` without duplicating code.  
**Key methods / business meaning**

- `MoveTo(...)` — move a blob from `source` → `destination` (copy then delete) — used whenever the pipeline needs to change a file’s lifecycle folder.
- `CopyTo(...)` — copy without delete (or with optional delete) — core operation used by MoveTo.
- `SaveTo(...)` — upload a stream to a named container/path (used when extracting ZIP entries).
- `Open(...)` — read a blob into a stream (used to hand file contents to orchestration).
- `Find(...)` — fallback lookup across common folders (helps download and recovery flows).
- `GetListing(...)` — used by the status page to show the container contents.

---

### HttpExtenstions (Extensions)

**Source:** `PDI_Azure_Function/Extensions/HttpExtensions.cs`  
**Meaning:** Small helper to determine the base URL used by the status page and its JavaScript, ensuring links and upload form actions point to the right host & root path.  
**Key method**

- `GetBaseURL(HttpRequest req)` — returns the base URL used by client JS and status links.

---

### QueueServiceExtensions (Extensions)

**Source:** `PDI_Azure_Function/Extensions/QueueServiceExtensions.cs`  
**Meaning:** Convenience wrappers around Azure `QueueClient` used to get queue length (for status/monitoring) and to create/add messages reliably (used by `BatchBlob` and other producers). Keeps queue creation/encoding logic centralized.  
**Key methods**

- `GetQueueLength(QueueClient queueClient)` — approximate message count (for status display).
- `AddMessage(QueueClient queueClient, DataTransferObject dto, ILogger log = null)` — ensures the queue exists and enqueues a Base64-encoded DTO.

---

### Container / Path glossary

- `incoming/` — initial upload area (IncomingBatchContainer).
- `processing/` — files currently being processed.
- `importing/` — files ready for Publisher import.
- `completed/` — successfully imported files.
- `rejected/` — failed files (need manual action or re-upload).
- `archive/` — original uploads stored for audit.

---

---

### pdi_Publisher_Documents (table)

**Source:** referenced in `DocumentProcessing.processDataStaging`  
**Meaning:** The Publisher-facing document catalogue table that stores per-client document metadata (Document_Number, FundCode, InceptionDate, FFDocAgeStatusID, IsActiveStatus, Last_Filing_Date, etc.). Updated by `DocumentProcessing` as staging rows are processed.

---

### sp_DataStagingPivot (stored procedure)

**Source:** `DocumentProcessing.GetStagingPivotTable(jobID)`  
**Meaning:** Stored procedure that returns a pivot of staging rows for a given Job_ID. The pivot provides one row per document code with standardized column names used by `DocumentProcessing`.

---

### FFDocAge (enum/status)

**Source:** `DocumentProcessing.SetFFDocAge` logic  
**Meaning:** Business status representing Fund Facts age classification (e.g., `TwelveConsecutiveMonths`, `BrandNewFund`, `NewFund`, `NewSeries`) used in Fund Facts rules and decisioning. `SetFFDocAge` encodes the decision tree using staging values like `InceptionDate`, `FilingReferenceID`, and existing `pdi_Publisher_Documents` state.

---

### UpdateIfChanged (helper pattern)

**Source:** `DocumentProcessing.UpdateIfChanged`  
**Meaning:** Helper that updates a DataRow column only if the new value is different (reduces unnecessary DB writes and sets `Last_Updated` when true).

---

### ValidationList / ExcelHelper / AsposeLoader

**Source:** `FileIntegrityCheck.cs` and `Helper/*`  
**Meaning:** Validation framework:

- `ValidationList` — collection of worksheet validators & rules.
- `ExcelHelper` — helper that reads Excel workbooks and exposes typed values for validation and logging.
- `AsposeLoader.TableUpdate` — determines additional worksheets in "Table UPDATE" scenarios and assists mapping them to template sections.

---

### PDIStream

**Source:** used by both classes  
**Meaning:** Wrapper around a `Stream` plus `PDIFile` metadata (OnlyFileName, FileID, DataID, etc.). Used to pass file content and metadata into validator and orchestration routines.

---

### PDIBatch (related term)

**Meaning:** A batch grouping with `BatchID` used by the Azure functions when a zip or multi-file upload arrives. `DocumentProcessing` will process staging rows produced by batch imports.

---

### Aspose.Cells (third-party)

**Meaning:** Excel-processing library used to load template and uploaded workbooks (`Excel.Workbook`) for validation and to preserve workbook comments/errors when validation fails. Requires `Aspose.Total.lic` to be present.

---

---

### pdi_Global_Text_Language

**Source:** `Generic.searchGlobalScenarioText`, `loadMonths`  
**Meaning:** Global lookup table for commonly used English/French phrases (month names, short/long forms, other standardized labels). Used for localized date and wording generation.

---

### pdi_Client_Field_Content_Scenario_Language

**Source:** `Generic.searchClientScenarioText`  
**Meaning:** Client-specific scenario table mapping `Field_Name` + scenario flags to English/French content templates. Scenario-ranking logic picks the best match and drives scenario-based content generation.

---

### pdi_Client_Translation_language

**Source:** `Generic.searchClientFrenchText`  
**Meaning:** Per-client table storing known English→French translations. Used for direct lookups; if a translation is missing, missing-French flow is triggered.

---

### pdi_Client_Translation_Language_Missing_Log` / `pdi_Client_Translation_Language_Missing_Log_Details`

**Source:** `Generic.AddMissingFrench`, `Generic.SaveFrench`  
**Meaning:** Log tables that track missing French translations discovered during import/transform. The system accumulates missing entries and bulk-inserts them for BA/translation teams to act upon.

---

### pdi_Publisher_Documents

**Source:** `Generic.getPublisherDocumentFields`  
**Meaning:** Publisher catalog table accessed to get document-level metadata (InceptionDate, FFDocAgeStatusID, IsProforma, Document names). `Generic` caches client-specific rows to serve field lookups used by transforms and FF business rules.

---

### MissingFrench (in-memory)

**Source:** `Generic.LoadMissingFrench` / `AddMissingFrench`  
**Meaning:** In-memory representation of missing translation items, deduplicated via a UniqueConstraint then persisted via `SaveFrench()`.

---
