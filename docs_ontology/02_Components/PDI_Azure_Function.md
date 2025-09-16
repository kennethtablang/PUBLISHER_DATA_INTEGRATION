# Component: PDI_Azure_Function

**One-line purpose**  
Detects incoming batch files (zip / .xlsx) in blob storage, extracts or registers batches, and enqueues work for the Publisher pipeline. Provides monitoring endpoints and small web UI for status & uploads.

---

## What this component does (business view)

- Listens for uploaded files from fund managers/partners.
- If the upload is a ZIP, extracts entries and places each entry as a processing file.
- If the upload is a single XLSX, registers it as a batch or single job.
- Writes initial metadata / batch records (via `PDIBatch` and `PDIStream`) and ensures a raw archive copy exists.
- Creates lightweight queue messages (DTOs) that tell downstream workers to import or render the file.
- Drives orchestration (via `Orchestration` class) that loads the content into Publisher and triggers further processing.
- Sends notification emails for success/failure using `PDISendGrid`.
- Exposes a simple status HTML page + small HTTP endpoints for uploads, downloads and validation messages.

---

## Key functions & triggers (what to look for)

- **Batch_Blob (`BatchBlob.cs`)** — BlobTrigger on `%IncomingBatchContainer%`. Primary entry point for batch or single-file uploads.
  - Handles `.zip` and `.xlsx`.
  - For zip: extracts entries and writes them to `processing/{entry}`.
  - For xlsx: records batch and moves file to `processing/{filename}`.
  - Enqueues a message into `%PDI_QueueName%`.

- **Queue_Process (`QueueProcess.cs`)** — QueueTrigger on `%PDI_QueueName%`. Processes files in `processing/{file}`; calls `Orchestration.ProcessFile` and (on success) moves file to `importing/{file}` and enqueues `%PUB_QueueName%` for Publisher import.

- **Queue_Import (`QueueImport.cs`)** — QueueTrigger on `%PUB_QueueName%`. Consumes messages intended for Publisher, runs `Orchestration.PublisherImport(jobID)` and moves blob into `completed/` or `rejected/`. Sends batch or per-file notification emails.

- **Blob_Process (`BlobProcess.cs`)** — present but currently commented out (not active).

- **Status (StatusPage.cs)** — HTTP endpoint `GET /api/Status/{offSet?}` that returns an HTML status page with:
  - Blob container listing (incoming batch container),
  - Queue lengths for PDI and PUB queues,
  - Small table of recent jobs and quick upload UI,
  - Download/validation endpoints (`DownloadBlob`, `ValidationMessages`, `FileUpload`).

- **TimerCheck.cs** — scheduled/periodic checks (health or housekeeping behaviour).

---

## Classes & Methods (detailed)

### `BatchBlob` — file: `PDI_Azure_Function/BatchBlob.cs`

- **Purpose:** Blob trigger that handles incoming batch ZIPs or standalone XLSX files.
- **Methods**
  - `Run(Stream myBlob, string name, Uri uri, BlobProperties properties, ILogger log)`  
    Trigger entry point. Detects file extension, handles ZIP extraction or single XLSX registration, records batch via `PDIBatch`, moves blobs between containers (processing/archive/rejected), and enqueues DTO messages to `%PDI_QueueName%`.

### `BlobProcess` — file: `PDI_Azure_Function/BlobProcess.cs` (commented out)

- **Purpose:** Previously intended BlobTrigger process; currently disabled. If re-enabled, would process single blob uploads and orchestrate render flow.
- **Methods** (all commented out)
  - `Run(...)` — processing logic (commented out). Check with dev lead whether it should be removed or re-enabled.

### `DataTransferObject` — file: `PDI_Azure_Function/DataTransferObjects.cs`

- **Purpose:** DTO that is the payload placed on queues (PDI and PUB) to pass file/batch/job context.
- **Properties**
  - `File_ID`, `FileName`, `FileRunID`, `FileStatus`, `Job_ID`, `Batch_ID`, `ProcessStatus`, `FileDetails`, `ErrorMessage`, `RetryCount`, `NotificationEmailAddress`.
- **Constructors / Methods**
  - `DataTransferObject()` — default constructor (initializes defaults).
  - `DataTransferObject(string jsonString)` — parses JSON into DTO fields.
  - `DataTransferObject(string jsonString, NameValueCollection values)` — parse + overlay form values.
  - `DataTransferObject(string fileName, int batchID, int retryCount)` — create DTO for queueing with minimal fields.
  - `DataTransferObject(Orchestration orch)` — build DTO from an `Orchestration` result.
  - `Base64Encode()` — returns Base64-encoded JSON suitable for Azure Queue messages.

### `QueueProcess` — file: `PDI_Azure_Function/QueueProcess.cs`

- **Purpose:** Consumer of the PDI queue; processes files placed into `processing/{file}` and prepares them for Publisher import.
- **Methods**
  - `Run(string myQueueItem, ILogger log, ExecutionContext exCtx)`  
    Trigger entry point. Decodes DTO, loads the `processing/{FileName}` stream into `PDIStream`, loads template if present, calls `Orchestration.ProcessFile(...)`, moves blob to `importing/{file}` on success, enqueues Publisher DTO to `%PUB_QueueName%`. On failure, moves file to `rejected/{file}` and triggers notification emails.

### `QueueImport` — file: `PDI_Azure_Function/QueueImport.cs`

- **Purpose:** Consumes Publisher queue messages (from `%PUB_QueueName%`) and runs `Orchestration.PublisherImport` to perform final import into the Publisher DB/service.
- **Methods**
  - `Run(string myQueueItem, ILogger log)`  
    Trigger: decodes DTO, initialises `Orchestration` with PUB connection, calls `PublisherImport(jobID)`, handles batch completion, sends notifications (batch or single), and moves blob to `completed/` or `rejected/` accordingly.

### `StatusPage` — file: `PDI_Azure_Function/StatusPage.cs`

- **Purpose:** Lightweight web UI / HTTP endpoints for health, status, upload, download, and validation message display.
- **Methods**
  - `Run(HttpRequest req, int? offSet, ILogger log, ClaimsPrincipal claimIdentity)` — HTTP GET `/api/Status/{offSet?}`; returns full HTML status page.
  - `GetStatusHTMLPage(int? offSet, HttpRequest req)` — builds the HTML content for the status page (includes blob listing, queue counts, recent jobs).
  - `GetHeader(string title)`, `GetJavaScript(string baseURL)`, `GetFooter()` — small HTML helper blocks used in the Status page.
  - `TableToHtml(DataTable dt, string baseURL = "../")` — converts DataTable to HTML table used in Status page.
  - `DownloadBlob(HttpRequest req, string fileName)` — HTTP GET `/api/DownloadBlob/{fileName}`; streams a blob back as an attachment when present.
  - `ValidationMessages(HttpRequest req, string fileName)` — HTTP GET `/api/ValidationMessages/{fileName}`; returns HTML with validation messages for a file.
  - `FileUpload(HttpRequest req)` — HTTP POST `/api/FileUpload`; accepts form file upload and stores to `IncomingBatchContainer`.
  - `EventGridHandlerHTTP(HttpRequest req)` — HTTP POST handler that receives Event Grid messages and forwards them to `Orchestration.HandleEventMessage`.

### `BlobServiceExtensions` — file: `PDI_Azure_Function/Extensions/BlobServiceExtensions.cs`

- **Purpose:** Convenience helpers to copy/move/save/open blobs and produce simple listings used by the status page and processing flows.
- **Methods**
  - `MoveTo(BlobServiceClient blobService, string sourceContainer, string sourcePath, string destContainer, string destPath, ILogger log = null)`  
    Copy the blob then delete the source (atomic “move” behavior). Waits for copy to complete and returns success/failure. Used to move files between `incoming/`, `processing/`, `importing/`, `completed/`, `rejected/`, and `archive/`.
  - `CopyTo(BlobServiceClient blobService, string sourceContainer, string sourcePath, string destContainer, string destPath, bool deleteSource = false, ILogger log = null)`  
    Core copy logic. Starts a server-side copy from source URI, waits for copy completion, optionally deletes the source. Returns boolean success.
  - `SaveTo(BlobServiceClient blobService, Stream sourceStream, string filePath, string destContainer)`  
    Uploads a `Stream` into the destination container/path, creating the container if needed. Returns true when upload succeeded.
  - `Open(BlobServiceClient blobService, string filePath, string sourceContainerName)`  
    Returns a `Stream` for a blob in the specified container (or `null` if not found). Used to feed `PDIStream` with blob content.
  - `Open(BlobClient source)`  
    Downloads the blob into a `MemoryStream` and returns it (helper overload).
  - `Find(BlobServiceClient blobService, string fileName, string sourceContainerName)`  
    Attempts to locate a blob by name searching common paths (`fileName`, `completed/fileName`, `rejected/fileName`, `processing/fileName`). Returns the `BlobClient` or `null`.
  - `GetListing(BlobContainerClient blobContainerClient, int? segmentSize)`  
    Returns a small HTML table (string) listing blobs, used by the `StatusPage` UI for quick blob container previews.
  - `GetBlobsAsync(BlobContainerClient blobContainerClient, int? segmentSize)` *(private async helper)*  
    Implements paged listing and returns an HTML string; called by `GetListing`.

---

### `HttpExtenstions` — file: `PDI_Azure_Function/Extensions/HttpExtensions.cs`

- **Purpose:** Simple HTTP helpers to compute predictable base URLs for use by the status page and client-side JavaScript.
- **Methods**
  - `GetBaseURL(HttpRequest req)`  
    Builds the base URL from the incoming request (scheme + host + trimmed path). Used to construct links and the upload form action on the status page.

---

### `QueueServiceExtensions` — file: `PDI_Azure_Function/Extensions/QueueServiceExtensions.cs`

- **Purpose:** Lightweight helpers wrapping `QueueClient` behaviors: read approximate queue length and safely add messages.
- **Methods**
  - `GetQueueLength(QueueClient queueClient)`  
    Returns the approximate message count for the queue (or -1 if the queue is missing). Used by `StatusPage` for quick queue metrics.
  - `AddMessage(QueueClient queueClient, DataTransferObject dto, ILogger log = null)`  
    Ensures the queue exists, encodes the DTO as Base64 JSON, sends the message, and logs the operation. Returns true when the message was enqueued.

> Note: I observed `QueueServiceExtensions` appears more than once in the pasted text (duplicate block). Keep only one copy of the file in the project to avoid maintenance confusion.

---

## Important environment / config names (do NOT store secrets in docs)

These are the keys the component reads at runtime — operations will map them to KeyVault or environment values:

- `sapdi` (Azure Storage connection string)
- `PDI_ConnectionString` (staging / PDI DB)
- `PUB_ConnectionString` (Publisher DB)
- `IncomingBatchContainer`
- `MainContainer`
- `PDI_QueueName`
- `PUB_QueueName`
- SMTP keys: `SMTP_Password`, `SMTP_FromEmail`, `SMTP_FromName`

---

## Runtime storage locations (blob paths used)

- `incoming/` (IncomingBatchContainer) — where partner uploads arrive
- `processing/{file}` — files being processed
- `importing/{file}` — files ready for Publisher import
- `completed/{file}` — successfully imported files
- `rejected/{file}` — files that failed validation/import
- `archive/{name}` — archived originals (for zip processing flow)
- `templates/{templateName}` — template blobs used during processing

---

## Where to look (files)

The function app consists of these files:

- `PDI_Azure_Function.csproj` — project file listing dependencies and build targets.
- `PDI_Azure_Function/BatchBlob.cs` — batch blob trigger (primary entry point).
- `PDI_Azure_Function/BlobProcess.cs` — currently commented out (not active).
- `PDI_Azure_Function/QueueImport.cs` — queue trigger for Publisher import.
- `PDI_Azure_Function/DataTransferObjects.cs` — DTO used in queue messages.
- `PDI_Azure_Function/QueueProcess.cs` — queue trigger for processing files.
- `PDI_Azure_Function/StatusPage.cs` — HTTP status page and endpoints.
- `PDI_Azure_Function/TimerCheck.cs` — periodic timer trigger for health checks.
- `PDI_Azure_Function/Extensions/*` — helper methods for Blob/Queue/HTTP.

---

## QA validation / quick test (simple steps)

1. Upload a small `sample.xlsx` to the QA incoming batch container (value referenced by `IncomingBatchContainer`).
2. Confirm the `Batch_Blob` function logs the file pickup and moves it to `processing/{filename}` in `MainContainer`.
3. Check that a message was added to the PDI queue (`%PDI_QueueName%`) — message payload should include `FileName` and `Batch_ID`.
4. Observe `Queue_Process` consume the message; confirm it calls `Orchestration.ProcessFile` and that the blob moves to `importing/{filename}`.
5. Confirm `Queue_Import` picks the message from `%PUB_QueueName%` and `importing/{filename}` moves to `completed/{filename}` (or `rejected/{filename}` on failure).
6. If something fails, review function logs and the `ValidationMessages` page for the file (Status page endpoint).

---

## Operations notes & quick runbook pointers

- **If files pile up in `processing/`**: check PDI queue length (status page shows approximate queue counts) and function logs for repeated exceptions.
- **To reprocess a file**: move or copy the blob back to `processing/{file}` and re-add a DTO into `%PDI_QueueName%` (use queue tools). Prefer using provided DB/SP requeue procedures if available.
- **Template missing**: processing will log missing template errors — templates live under `templates/{templateName}` in `MainContainer`.
- **Commented-out code**: `BlobProcess` is commented out — confirm with dev lead whether it’s intentionally deprecated.

---

