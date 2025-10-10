# Class: QueueProcess

**One-line purpose**  
Azure Queue-triggered function that processes uploaded files from the PDI queue, orchestrates file validation and transformation, manages blob container transitions, and enqueues successfully processed files to the Publisher import queue.

---

## What this class does (business view)

- Listens to the PDI queue (`%PDI_QueueName%`) for incoming file processing messages.
- Retrieves the file from the `processing/` blob container based on the queue message payload.
- Loads the corresponding template file from `templates/` container if required for validation or transformation.
- Orchestrates the file processing workflow through the `Orchestration` class which validates, transforms, and prepares the file for Publisher import.
- Manages blob lifecycle by moving files between containers:
  - Success: `processing/` → `importing/` → enqueues to Publisher queue
  - Failure: `processing/` → `rejected/`
- Handles notification emails for both individual files and batch completions via `PDISendGrid`.
- Tracks batch progress and triggers batch completion emails when all files in a batch are processed.
- Implements comprehensive error handling with logging and fallback notifications.

---

## Key functions & triggers (what to look for)

- **Queue Trigger Binding** — Activated by messages arriving in `%PDI_QueueName%` (configured via environment variable).
- **Blob Container Transitions** — Files move through: `processing/` → `importing/` (success) or `rejected/` (failure).
- **Template Loading** — Dynamically loads template files from `templates/{defaultTemplateName}` based on file type requirements.
- **Orchestration Delegation** — Core processing logic is delegated to `Orchestration.ProcessFile()` method.
- **Queue Chaining** — Successfully processed files are enqueued to `%PUB_QueueName%` for downstream Publisher import.
- **Notification Logic** — Conditional email sending based on:
  - Single file vs. batch processing
  - Success vs. error conditions
  - Batch completion status
- **Disposable Pattern** — Ensures proper resource cleanup for streams, database connections, and orchestration objects in `finally` block.

---

## Classes & Methods (detailed)

### `QueueProcess` — file: `PDI_Azure_Function/QueueProcess.cs`

**Purpose:** Azure Function class that serves as the queue consumer for the PDI processing pipeline. Acts as the orchestration coordinator between blob storage, database operations, and downstream Publisher queue.

**Key Characteristics:**

- Static class following Azure Functions v3 pattern
- Implements ILogger-based structured logging
- Uses environment variables for configuration (connection strings, queue names, container names)
- Dependency on `Publisher_Data_Operations` namespace for core business logic
- Implements blob container lifecycle management via extension methods

---

### Methods

#### `Run`

```csharp
public static void Run(
    [QueueTrigger("%PDI_QueueName%", Connection = "sapdi")] string myQueueItem, 
    ILogger log, 
    ExecutionContext exCtx)
```

**Purpose:** Main entry point triggered by Azure Queue messages. Orchestrates the entire file processing workflow from queue message to blob movement and downstream queue enqueueing.

**Parameters:**

- `myQueueItem` (string) — JSON-serialized `DataTransferObject` containing file metadata and processing context
- `log` (ILogger) — Azure Functions logger for structured logging and Application Insights telemetry
- `exCtx` (ExecutionContext) — Azure Functions execution context providing invocation ID and function metadata

**Workflow Steps:**

1. **Message Deserialization**
   - Deserializes queue message into `DataTransferObject` (DTO)
   - Extracts file name, batch ID, and retry count from DTO

2. **Environment Configuration Loading**
   - `sapdi` — Azure Storage connection string
   - `PDI_ConnectionString` — PDI database connection string
   - `MainContainer` — Primary blob container name
   - `PUB_QueueName` — Publisher queue name for downstream processing
   - `PUB_ConnectionString` — Publisher database connection string

3. **Stream Initialization**
   - Creates `PDIStream` object from blob at `processing/{dto.FileName}`
   - Assigns `FileRunID` using `exCtx.InvocationId` for execution traceability
   - **Critical:** Stream is loaded before blob movement to prevent content loss

4. **Template Stream Loading**
   - Determines default template name via `curStream.PdiFile.GetDefaultTemplateName()`
   - Attempts to load template from `templates/{templateName}` in blob storage
   - Logs critical error if template is missing but continues processing
   - Template is optional depending on file type requirements

5. **File Processing Orchestration**
   - Invokes `orch.ProcessFile(curStream, templateStream, dto.RetryCount, log)`
   - Returns boolean result indicating processing success
   - `orch.FileStatus` provides additional validation state
   - `orch.ErrorMessage` contains detailed error information if processing fails

6. **Success Path** (if `result && orch.FileStatus`)
   - Moves blob from `processing/{fileName}` to `importing/{fileName}`
   - Creates new `DataTransferObject` from orchestration result
   - Enqueues message to `%PUB_QueueName%` for Publisher import
   - Sets `messageSent = true` to suppress duplicate notifications
   - Logs critical errors if queue is not configured or message enqueueing fails

7. **Failure Path** (if processing fails)
   - Moves blob from `processing/{fileName}` to `rejected/{fileName}`
   - Logs warning if blob movement fails
   - Prepares for error notification in finally block

8. **Exception Handling**
   - Catches all exceptions during processing
   - Captures error details in `localError` variable
   - Logs critical error with full stack trace
   - Ensures notification emails are sent in finally block

9. **Notification Workflow** (finally block - executes only if `!messageSent`)
   - Loads `PDIBatch` object using `dto.Batch_ID`
   - Initializes `PDISendGrid` with SMTP credentials from environment variables
   - Sends error emails if `orch.ErrorMessage` or `localError` contain error details
   - **Single File Processing** (`pdiBatch.Count() < 2`):
     - Sends individual file notification via `SendTemplateMessage(curStream)`
   - **Batch Processing** (`pdiBatch.Count() >= 2`):
     - Marks current file as complete via `pdiBatch.SetComplete(dto.FileName)`
     - Checks if entire batch is complete via `pdiBatch.IsComplete()`
     - Sends batch completion notification via `SendBatchMessage(pdiBatch)`
   - Logs critical errors if email sending fails

10. **Resource Cleanup** (finally block)
    - Disposes `PDIBatch` object (closes database connections)
    - Disposes `Orchestration` object (releases processing resources)
    - Disposes `curStream` (closes blob stream and file handles)
    - Disposes `templateStream` if it was loaded (in try block)

**Error Handling Strategy:**

- **Transient Errors:** Relies on Azure Queue retry mechanism (message remains in queue on failure)
- **Processing Errors:** Captured in `orch.ErrorMessage` and sent via email
- **Runtime Exceptions:** Caught, logged, and notified via `localError`
- **Blob Movement Failures:** Logged but do not halt execution
- **Email Failures:** Logged as critical but do not throw exceptions

**Logging Levels Used:**

- `LogInformation` — Processing start, completion status
- `LogWarning` — Non-critical failures (blob movement to rejected)
- `LogCritical` — Missing templates, queue configuration issues, blob movement failures, email sending failures, processing exceptions

**Dependencies:**

- **Azure SDK:** `BlobServiceClient`, `QueueClient`, `BlobClient`
- **PDI Operations:** `PDIStream`, `Orchestration`, `PDIBatch`, `PDISendGrid`
- **Extensions:** `BlobServiceClient.Open()`, `BlobServiceClient.MoveTo()`, `QueueClient.AddMessage()`

**Key Design Decisions:**

- **Stream Loading Before Movement:** Ensures blob content is in memory before file is moved
- **Template Optional:** Processing continues even if template is missing (logged as critical)
- **Notification Suppression:** `messageSent` flag prevents duplicate emails when Publisher queue is used
- **Batch Awareness:** Different notification logic for single files vs. batch processing
- **Graceful Degradation:** Missing configuration (e.g., PUB_QueueName) logs critical but doesn't crash function

**Environment Variables Required:**

- `sapdi` — Azure Storage Account connection string
- `PDI_ConnectionString` — PDI SQL database connection string
- `PUB_ConnectionString` — Publisher SQL database connection string
- `PDI_QueueName` — Queue name for incoming PDI processing messages
- `MainContainer` — Primary blob container name
- `PUB_QueueName` — Queue name for Publisher import messages
- `SMTP_Password` — SendGrid API key or SMTP password
- `SMTP_FromEmail` — Sender email address for notifications
- `SMTP_FromName` — Sender display name for notifications

**Performance Considerations:**

- **Synchronous Processing:** Queue trigger is processed synchronously; long-running operations block the function
- **Stream Buffering:** Entire file is loaded into memory via `PDIStream`
- **Database Connections:** Multiple database operations per invocation (PDIStream, Orchestration, PDIBatch)
- **Blob Operations:** Multiple blob movements and reads per invocation

**Retry Behavior:**

- Azure Queue default retry policy applies (exponential backoff)
- `dto.RetryCount` is passed to orchestration for custom retry logic
- Failed messages eventually move to poison queue after max retries

---

## Related Components

- **BatchBlob.cs** — Upstream: Places initial messages into `%PDI_QueueName%`
- **QueueImport.cs** — Downstream: Consumes messages from `%PUB_QueueName%`
- **Orchestration** — Core processing engine invoked by this function
- **PDIStream** — File stream wrapper with metadata tracking
- **PDIBatch** — Batch metadata and completion tracking
- **PDISendGrid** — Email notification service

---

## Business Rules Implemented

1. **Template Requirement:** Files may require a template from `templates/` container based on file type
2. **Batch Completion:** Batch notifications are sent only when all files in batch are processed
3. **Error Notification:** Errors are emailed to technical team (SMTP_FromEmail)
4. **File Lifecycle:** Processed files move to `importing/`, failed files move to `rejected/`
5. **Queue Chaining:** Successfully processed files must be enqueued to Publisher queue
6. **Single vs. Batch Notification:** Different email templates for single files vs. batch completions

---

## Configuration Dependencies

| Environment Variable | Purpose | Required |
|---------------------|---------|----------|
| `sapdi` | Azure Storage connection string | Yes |
| `PDI_ConnectionString` | PDI database connection | Yes |
| `PUB_ConnectionString` | Publisher database connection | Yes |
| `PDI_QueueName` | Input queue name | Yes |
| `MainContainer` | Blob container name | Yes |
| `PUB_QueueName` | Publisher queue name | No (logs critical if missing) |
| `SMTP_Password` | Email service credentials | Yes (for notifications) |
| `SMTP_FromEmail` | Notification sender email | Yes (for notifications) |
| `SMTP_FromName` | Notification sender name | Yes (for notifications) |

---

## Troubleshooting Guide

**Symptom:** Files stuck in `processing/` container  
**Possible Causes:**

- Queue trigger not executing (check function app status)
- Exception during processing (check logs for critical errors)
- Blob movement failing (check storage account permissions)

**Symptom:** No notification emails received  
**Possible Causes:**

- `SMTP_*` environment variables not configured
- `messageSent = true` (file successfully enqueued to Publisher)
- `PDISendGrid.SendTemplateMessage()` failing (check logs)

**Symptom:** Files not appearing in Publisher queue  
**Possible Causes:**

- `PUB_QueueName` environment variable not set
- Queue client failing to add message (check logs for critical errors)
- `orch.FileStatus = false` (file failed validation)

**Symptom:** Duplicate email notifications  
**Possible Causes:**

- Publisher queue not configured (messageSent remains false)
- Function retrying due to exception (check for unhandled errors)

---

## Future Enhancement Opportunities

- Implement async/await pattern for I/O operations (blob, queue, database)
- Add telemetry for processing duration metrics
- Implement circuit breaker for downstream queue failures
- Add support for dynamic template selection via file metadata
- Implement dead letter queue handling for poison messages
- Add retry backoff strategy configuration
- Implement parallel batch processing for large batches
