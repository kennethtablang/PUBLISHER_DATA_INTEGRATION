# Class: QueueImport

**One-line purpose**  
Azure Queue-triggered function that consumes Publisher import messages, executes data import into the Publisher database, manages final blob lifecycle transitions, and sends completion or error notifications.

---

## What this class does (business view)

- Listens to the Publisher queue (`%PUB_QueueName%`) for files ready to be imported into the Publisher system.
- Retrieves files from the `importing/` blob container that have already passed PDI validation.
- Orchestrates the Publisher database import operation via `Orchestration.PublisherImport()`.
- Manages final blob lifecycle transitions:
  - Success: `importing/` → `completed/`
  - Failure: `importing/` → `rejected/`
- Handles batch-aware notification logic:
  - Sends individual file completion emails for single files
  - Tracks batch progress and sends batch completion emails when all files in a batch are imported
- Implements error notification and recovery with comprehensive logging.
- Acts as the final stage in the PDI-to-Publisher data pipeline.

---

## Key functions & triggers (what to look for)

- **Queue Trigger Binding** — Activated by messages arriving in `%PUB_QueueName%` (downstream from QueueProcess).
- **Publisher Import Execution** — Core operation is `Orchestration.PublisherImport(jobID)` which loads data into Publisher database.
- **Final Blob Movement** — Files reach their terminal state: `completed/` (success) or `rejected/` (failure).
- **Batch Completion Detection** — Uses `PDIBatch.IsComplete()` to determine if all batch files have been imported.
- **Template Loading for Batch Notifications** — Loads static template for batch completion emails.
- **Error Recovery** — Exception handling with batch completion marking to prevent stuck batches.
- **Notification Dispatch** — Different email templates for single files vs. batch completions.

---

## Classes & Methods (detailed)

### `QueueImport` — file: `PDI_Azure_Function/QueueImport.cs`

**Purpose:** Azure Function class that serves as the final stage consumer in the PDI processing pipeline. Responsible for importing validated data into the Publisher database and completing the file processing lifecycle.

**Key Characteristics:**

- Static class following Azure Functions v3 pattern
- Implements ILogger-based structured logging
- Consumes messages from `%PUB_QueueName%` (enqueued by QueueProcess)
- Final authority on file lifecycle (completed vs. rejected)
- Batch-aware with conditional notification logic
- No retry count handling (assumes upstream validation is complete)

---

### Methods

#### `Run`

```csharp
public static void Run(
    [QueueTrigger("%PUB_QueueName%", Connection = "sapdi")] string myQueueItem, 
    ILogger log)
```

**Purpose:** Main entry point triggered by Azure Queue messages from the Publisher queue. Orchestrates the Publisher database import operation and finalizes file processing with appropriate notifications.

**Parameters:**

- `myQueueItem` (string) — JSON-serialized `DataTransferObject` containing file metadata, job ID, and batch context
- `log` (ILogger) — Azure Functions logger for structured logging and Application Insights telemetry

**Workflow Steps:**

1. **Environment Configuration Loading**
   - `PDI_ConnectionString` — PDI database connection string
   - `MainContainer` — Primary blob container name
   - `sapdi` — Azure Storage connection string
   - `PUB_ConnectionString` — Publisher database connection string
   - `SMTP_*` — Email notification credentials

2. **Message Deserialization and Initialization**
   - Deserializes queue message into `DataTransferObject` (DTO)
   - Initializes `PDIBatch` object using `dto.Batch_ID` for batch tracking
   - Initializes `PDISendGrid` with SMTP credentials for notification handling
   - Creates `BlobServiceClient` for blob operations

3. **Stream Loading**
   - Creates `PDIStream` from blob at `importing/{dto.FileName}`
   - **Critical:** File must exist in `importing/` container (moved there by QueueProcess)
   - Stream includes file metadata and database context

4. **Orchestration Initialization**
   - Creates `Orchestration` object with both PDI and Publisher connection strings
   - Passes `curStream` to orchestration for file context
   - **Commented Code Note:** Staging environment SP clearing logic removed (20221114) — production and staging are now identical

5. **Job ID Validation**
   - Attempts to parse `dto.Job_ID` to integer
   - **Critical:** Job_ID must be valid integer for Publisher import
   - If parsing fails, function logs error and completes without import

6. **Publisher Import Execution**
   - Invokes `orch.PublisherImport(jobID, log)`
   - Returns boolean indicating import success
   - Import operation loads data from PDI staging tables into Publisher database

7. **Success Path** (if `orch.PublisherImport()` returns true)

   **7a. Batch Processing** (if `pdiBatch.BatchID > -1 && pdiBatch.Count() > 1`)
   - Marks current file as complete: `pdiBatch.SetComplete(dto.FileName)`
   - Checks if entire batch is complete: `pdiBatch.IsComplete()`
   - If batch complete:
     - Loads static template from `templates/{curStream.PdiFile.GetDefaultStaticTemplateName()}`
     - Logs critical error if template missing but continues
     - Sends batch completion email: `pdiSendGrid.SendBatchMessage(pdiBatch)`
     - Logs critical error if email fails but continues

   **7b. Single File Processing** (if not a batch or batch count = 1)
   - Sends individual file completion email: `pdiSendGrid.SendTemplateMessage(curStream)`
   - Logs critical error if email fails but continues

   **7c. Final Blob Movement**
   - Moves blob from `importing/{dto.FileName}` to `completed/{dto.FileName}`
   - File reaches terminal success state

8. **Failure Path** (if `orch.PublisherImport()` returns false)
   - Marks file as complete in batch tracking: `pdiBatch.SetComplete(dto.FileName)`
   - Sends error notification email: `pdiSendGrid.SendErrorMessage(curStream, orch.ErrorMessage)`
   - Logs critical error with Job_ID and error message
   - Moves blob from `importing/{dto.FileName}` to `rejected/{dto.FileName}`
   - File reaches terminal failure state

9. **Exception Handling** (catch block)
   - Catches all unhandled exceptions during import process
   - Logs critical error with full stack trace
   - Sends error notification email with exception details
   - Marks file as complete in batch: `pdiBatch.SetComplete(dto.FileName)`
   - If batch is now complete, sends batch completion email (which will show failed files)
   - Logs critical error if batch email fails

10. **Resource Cleanup** (finally block)
    - Disposes `curStream` (closes blob stream and database connections)
    - Disposes `pdiBatch` (closes database connections)
    - **Note:** `Orchestration` is disposed in try block, not in finally

**Error Handling Strategy:**

- **Import Failures:** Captured via boolean return from `PublisherImport()`, file moved to rejected
- **Runtime Exceptions:** Caught, logged, notified, and file marked complete to unblock batch
- **Email Failures:** Logged as critical but do not halt execution or cause retries
- **Template Missing:** Logged as critical but batch notification still attempted
- **Blob Movement Failures:** Not explicitly handled (relies on extension method error handling)

**Logging Levels Used:**

- `LogInformation` — Process start and completion
- `LogCritical` — Missing templates, email sending failures, import failures, runtime exceptions

**Dependencies:**

- **Azure SDK:** `BlobServiceClient`, `BlobClient`
- **PDI Operations:** `PDIStream`, `Orchestration`, `PDIBatch`, `PDISendGrid`
- **Extensions:** `BlobServiceClient.Open()`, `BlobServiceClient.MoveTo()`

**Key Design Decisions:**

- **No Retry Logic:** Assumes file has already passed validation in QueueProcess; failures are terminal
- **Batch Completion on Error:** Files are marked complete even on error to prevent batch deadlock
- **Template Types:** Uses `GetDefaultStaticTemplateName()` for batch emails (different from processing template)
- **Conditional Notification:** Single file vs. batch logic based on `pdiBatch.Count() > 1`
- **Error Email Always Sent:** Both import failures and exceptions trigger error notifications
- **Orchestration Disposal:** Disposed in try block rather than finally (potential resource leak on exception)

**Batch Processing Logic:**

- **Batch Identification:** `pdiBatch.BatchID > -1` (valid batch) AND `pdiBatch.Count() > 1` (multiple files)
- **Completion Tracking:** Each file calls `SetComplete(fileName)` regardless of success/failure
- **Completion Check:** `IsComplete()` returns true when all files in batch are marked complete
- **Notification Timing:** Batch email sent only when last file completes (success or failure)

**Template Loading Differences:**

- **Processing Template** (QueueProcess): `GetDefaultTemplateName()` — used for validation/transformation
- **Static Template** (QueueImport): `GetDefaultStaticTemplateName()` — used for batch notification emails
- **Purpose:** Different templates serve different purposes in the pipeline

**Environment Variables Required:**

- `sapdi` — Azure Storage Account connection string
- `PDI_ConnectionString` — PDI SQL database connection string
- `PUB_ConnectionString` — Publisher SQL database connection string
- `PUB_QueueName` — Queue name for Publisher import messages (trigger binding)
- `MainContainer` — Primary blob container name
- `SMTP_Password` — SendGrid API key or SMTP password
- `SMTP_FromEmail` — Sender email address for notifications
- `SMTP_FromName` — Sender display name for notifications

**Performance Considerations:**

- **Synchronous Processing:** Import operation is synchronous; long-running imports block the function
- **Database Connection Pooling:** Multiple database connections per invocation (PDI and Publisher)
- **Blob Operations:** Stream loading and movement operations are synchronous I/O
- **Email Blocking:** Email sending is synchronous and may impact function duration

**Retry Behavior:**

- **Azure Queue Default:** Exponential backoff retry policy applies
- **No Custom Retry Logic:** Unlike QueueProcess, no retry count tracking
- **Poison Queue:** Failed messages eventually move to poison queue after max retries
- **Exception Recovery:** Exceptions are logged but message processing completes (dequeued)

**Differences from QueueProcess:**

| Aspect | QueueProcess | QueueImport |
|--------|-------------|-------------|
| Queue Source | `%PDI_QueueName%` | `%PUB_QueueName%` |
| Blob Input Location | `processing/` | `importing/` |
| Blob Success Location | `importing/` | `completed/` |
| Blob Failure Location | `rejected/` | `rejected/` |
| Core Operation | `ProcessFile()` | `PublisherImport()` |
| Template Type | `GetDefaultTemplateName()` | `GetDefaultStaticTemplateName()` |
| Retry Count | Passed to orchestration | Not used |
| Queue Chaining | Enqueues to PUB queue | Terminal stage (no downstream) |
| Notification Logic | Sent only if not enqueued | Always sent (success or failure) |

---

## Related Components

- **QueueProcess.cs** — Upstream: Enqueues messages to `%PUB_QueueName%` after successful processing
- **Orchestration** — Executes `PublisherImport()` method to load data into Publisher database
- **PDIStream** — File stream wrapper with metadata and database context
- **PDIBatch** — Batch metadata and completion tracking
- **PDISendGrid** — Email notification service

---

## Business Rules Implemented

1. **Terminal Processing Stage:** Files reach final state (completed or rejected) with no further queue chaining
2. **Batch Completion Tracking:** All files in batch must complete before batch notification is sent
3. **Error Tolerance:** Files are marked complete even on error to prevent batch deadlock
4. **Dual Notification Paths:**
   - Single files receive individual completion emails
   - Batch files receive batch summary email when last file completes
5. **Static Template Requirement:** Batch notifications require static template from blob storage
6. **Import Atomicity:** Each file import is independent; batch tracking is metadata-only

---

## Configuration Dependencies

| Environment Variable | Purpose | Required |
|---------------------|---------|----------|
| `sapdi` | Azure Storage connection string | Yes |
| `PDI_ConnectionString` | PDI database connection | Yes |
| `PUB_ConnectionString` | Publisher database connection | Yes |
| `PUB_QueueName` | Input queue name (trigger) | Yes |
| `MainContainer` | Blob container name | Yes |
| `SMTP_Password` | Email service credentials | Yes (for notifications) |
| `SMTP_FromEmail` | Notification sender email | Yes (for notifications) |
| `SMTP_FromName` | Notification sender name | Yes (for notifications) |

---

## Troubleshooting Guide

**Symptom:** Files stuck in `importing/` container  
**Possible Causes:**

- Queue trigger not executing (check function app status)
- Exception during import causing retries (check logs for critical errors)
- Job_ID parsing failure (check DTO message format)
- Publisher database connection issues (check connection string)

**Symptom:** No completion emails received  
**Possible Causes:**

- `SMTP_*` environment variables not configured
- `PDISendGrid.SendTemplateMessage()` failing silently (check logs for critical errors)
- Email service (SendGrid) quota exceeded or credentials invalid

**Symptom:** Batch notification never sent  
**Possible Causes:**

- One or more files in batch failed to call `SetComplete()` (check batch tracking table)
- Batch count mismatch in database
- Static template file missing from `templates/` container
- `PDIBatch.IsComplete()` logic error

**Symptom:** Files moved to `rejected/` unexpectedly  
**Possible Causes:**

- `PublisherImport()` returning false (check `orch.ErrorMessage` in logs)
- Database constraint violations during import
- Data type mismatches or invalid data in source files

**Symptom:** Duplicate notifications for same batch  
**Possible Causes:**

- Function retrying due to exception before `SetComplete()` call
- Race condition if multiple files complete simultaneously
- Batch completion check logic error

**Symptom:** Resource leak or memory issues  
**Possible Causes:**

- `Orchestration` not disposed on exception (not in finally block)
- Large file streams not being disposed properly
- Database connection pool exhaustion

---

## Known Issues and Technical Debt

1. **Orchestration Disposal:** `orch.Dispose()` is called in try block, not finally. If exception occurs, orchestration resources may leak.
   - **Recommendation:** Move `orch.Dispose()` to finally block

2. **Blob Movement Error Handling:** No explicit error handling for `MoveTo()` operations. Silent failures possible.
   - **Recommendation:** Add explicit error handling and logging for blob movements

3. **Template Loading in Success Path:** Template is loaded inside batch completion logic, potentially causing unnecessary I/O.
   - **Recommendation:** Load template before `PublisherImport()` call if batch completion is likely

4. **Commented Code:** Staging environment SP clearing logic remains in comments (removed 20221114).
   - **Recommendation:** Remove commented code if no longer needed

5. **No Async/Await:** All I/O operations are synchronous, blocking the function thread.
   - **Recommendation:** Refactor to async pattern for better throughput

6. **Job_ID Parsing:** Silent failure if `Job_ID` cannot be parsed to integer.
   - **Recommendation:** Add explicit logging and error notification for parsing failures

---

## Future Enhancement Opportunities

- Refactor to async/await pattern for all I/O operations (blob, queue, database, email)
- Move `Orchestration` disposal to finally block for proper resource management
- Add explicit error handling for blob movement operations
- Implement telemetry for import duration and success rates
- Add retry backoff strategy for transient Publisher database errors
- Implement parallel batch import for independent files
- Add support for partial batch notifications (progress updates)
- Implement dead letter queue handling for poison messages
- Add health check endpoint for Publisher database connectivity
- Remove legacy commented code for environment-specific logic
