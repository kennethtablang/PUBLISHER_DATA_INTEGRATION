# Classes: QueueServiceExtensions & HttpExtensions

**One-line purpose**  
Extension method utility classes providing simplified Azure Queue operations (message count retrieval and DTO enqueueing) and HTTP request URL manipulation for API base path extraction.

---

## What these classes do (business view)

### QueueServiceExtensions

- Provides simplified access to Azure Storage Queue operations with minimal boilerplate.
- Retrieves approximate queue message counts for monitoring and status dashboard display.
- Handles DataTransferObject (DTO) serialization and enqueueing with automatic queue creation.
- Implements null-safe queue operations with existence checks and optional logging.
- Abstracts Base64 encoding complexity from calling code.

### HttpExtensions

- Extracts base URL from HTTP requests for dynamic link generation in web UI.
- Constructs proper API base paths for relative URL building in status pages.
- Handles various hosting scenarios (different schemes, hosts, ports).
- Supports Azure Functions routing patterns with "api/" path prefix.

---

## Key functions & patterns (what to look for)

### QueueServiceExtensions key function

- **Existence Checking** — All operations verify queue existence before proceeding.
- **Automatic Queue Creation** — `AddMessage()` creates queue if it doesn't exist.
- **Approximate Count** — Uses Azure's cached message count (not real-time).
- **DTO Serialization** — Automatically Base64-encodes DTO for queue compatibility.
- **Optional Logging** — `AddMessage()` accepts optional ILogger for telemetry.

### HttpExtensions key function

- **URI Builder Pattern** — Constructs URLs from request components programmatically.
- **Path Truncation** — Extracts base path up to and including "api/" segment.
- **Port Handling** — Properly handles optional port values in URLs.
- **Null-Safe Port Access** — Uses ternary operator for HasValue check.

---

## Classes & Methods (detailed)

---

## Class: QueueServiceExtensions

### File: `PDI_Azure_Function/Extensions/QueueServiceExtensions.cs`

**Purpose:** Static extension class for `QueueClient` that simplifies common queue operations used in the PDI processing pipeline.

**Key Characteristics:**

- All methods are static extension methods on `QueueClient`
- Implements defensive programming with existence checks
- Integrates with PDI's `DataTransferObject` serialization pattern
- Provides monitoring capabilities for status dashboard

---

### Methods

#### `GetQueueLength`

```csharp
public static int GetQueueLength(this QueueClient queueClient)
```

**Purpose:** Retrieves the approximate number of messages currently in the queue for monitoring and status display purposes.

**Parameters:**

- `queueClient` (QueueClient) — Azure Storage Queue client instance

**Workflow:**

1. **Queue Existence Check**
   - Calls `queueClient.Exists()` to verify queue presence
   - **Early Exit:** Returns `-1` if queue doesn't exist

2. **Properties Retrieval**
   - Calls `queueClient.GetProperties()` to get queue metadata
   - Returns `QueueProperties` object with queue statistics

3. **Message Count Extraction**
   - Accesses `properties.ApproximateMessagesCount`
   - Returns cached count from Azure Storage service

**Return Value:**

- `int` (>= 0) — Approximate number of messages in queue
- `-1` — Queue doesn't exist

**Important Characteristics:**

**Approximate Count:**

- Value is cached by Azure Storage service
- May lag behind actual count by several seconds
- Suitable for monitoring, not for precise counting
- Updates approximately every 30-60 seconds

**Performance:**

- Single API call to Azure Storage
- Fast operation (metadata retrieval only)
- No message enumeration required

**Use Cases:**

- Status page queue length display
- Monitoring queue backlog
- Health check endpoints
- Capacity planning metrics

**Example Usage:**

```csharp
QueueClient pdiQueue = new QueueClient(connectionString, "pdi-queue");
int messageCount = pdiQueue.GetQueueLength();

if (messageCount > 100)
{
    log.LogWarning("Queue backlog detected: {count} messages", messageCount);
}
```

**Considerations:**

- `-1` return value indicates missing queue (not empty queue)
- Empty queue returns `0`, not `-1`
- Caller should handle `-1` as error condition
- Not suitable for exact message counting or billing

---

#### `AddMessage`

```csharp
public static bool AddMessage(
    this QueueClient queueClient, 
    DataTransferObject dto, 
    ILogger log = null)
```

**Purpose:** Serializes a DataTransferObject to Base64-encoded JSON, enqueues it to Azure Storage Queue with automatic queue creation, and optionally logs the operation.

**Parameters:**

- `queueClient` (QueueClient) — Azure Storage Queue client instance
- `dto` (DataTransferObject) — PDI data transfer object containing file metadata and processing context
- `log` (ILogger, optional) — Logger for telemetry and debugging

**Workflow:**

1. **Automatic Queue Creation**
   - Calls `queueClient.CreateIfNotExists()` to ensure queue exists
   - Idempotent operation (safe to call on existing queues)
   - Creates queue if first message is being sent

2. **Queue Existence Verification**
   - Calls `queueClient.Exists()` after creation attempt
   - Verifies queue is ready for message operations
   - **Early Exit:** Returns `false` if queue doesn't exist after creation attempt

3. **DTO Serialization and Enqueueing**
   - Calls `dto.Base64Encode()` to serialize DTO:
     - Converts DTO to JSON string
     - Base64-encodes JSON for queue compatibility
   - Calls `queueClient.SendMessage(encodedMessage)` to enqueue
   - Message becomes visible to consumers immediately (default visibility timeout)

4. **Logging** (if logger provided)
   - Logs informational message: "Batch Blob Trigger: Created Queue message in {queueName}"
   - Includes queue name for tracking
   - **Note:** Log message mentions "Batch Blob Trigger" but method used by multiple callers

5. **Success Return**
   - Returns `true` to indicate successful enqueueing

**Return Value:**

- `true` — Message successfully enqueued
- `false` — Queue doesn't exist (even after creation attempt)

**Message Format:**

```note
Base64(JSON({
  "File_ID": "123",
  "FileName": "data.xlsx",
  "FileRunID": "guid",
  "Job_ID": "456",
  "Batch_ID": "789",
  ...
}))
```

**Queue Creation Behavior:**

- Queue created with default Azure Storage settings:
  - Message TTL: 7 days (default)
  - Visibility timeout: 30 seconds (default for dequeue)
  - Max message size: 64 KB
  - Max queue size: 500 TB

**Error Scenarios:**

- **Queue creation fails** → `CreateIfNotExists()` throws exception (not caught)
- **SendMessage fails** → Exception propagates to caller
- **Queue deleted between creation and exists check** → Returns `false`

**Use Cases:**

- Enqueueing files for PDI processing (QueueProcess)
- Enqueueing files for Publisher import (QueueImport)
- Chaining processing stages via queue messages
- Retry operations by re-enqueueing failed messages

**Example Usage:**

```csharp
DataTransferObject dto = new DataTransferObject {
    FileName = "report.xlsx",
    Batch_ID = 123,
    RetryCount = 0
};

QueueClient pdiQueue = new QueueClient(connectionString, "pdi-queue");
bool success = pdiQueue.AddMessage(dto, log);

if (!success)
{
    log.LogCritical("Failed to enqueue message for {fileName}", dto.FileName);
}
```

**Performance Considerations:**

- Single API call to Azure Storage (fast operation)
- Base64 encoding adds ~33% size overhead
- Message size limit: 64 KB after Base64 encoding
- Synchronous operation (blocks until enqueued)

**Logging Behavior:**

- Only logs on success (no failure logging)
- Log level: Information (not warning or error)
- Log message hardcoded to "Batch Blob Trigger" (misleading for other callers)

**Integration with PDI Pipeline:**

- **QueueProcess → PUB Queue:** Enqueues DTO for Publisher import
- **BatchBlob → PDI Queue:** Enqueues DTO for initial processing
- **Retry Scenarios:** Re-enqueues failed messages with incremented retry count

---

## Class: HttpExtensions

### File: `PDI_Azure_Function/Extensions/HttpExtensions.cs`

**Purpose:** Static extension class for `HttpRequest` that extracts base URL from HTTP requests for dynamic link generation in web UI components.

**Key Characteristics:**

- Single-purpose extension for URL manipulation
- Handles Azure Functions routing patterns
- Supports various hosting scenarios
- No error handling (exceptions propagate)

---

### Method

#### `GetBaseURL`

```csharp
public static string GetBaseURL(this HttpRequest req)
```

**Purpose:** Constructs the base API URL from an HTTP request by extracting scheme, host, port, and path up to the "api/" segment for use in relative link generation.

**Parameters:**

- `req` (HttpRequest) — ASP.NET Core HTTP request object

**Workflow:**

1. **URI Builder Initialization**
   - Creates `UriBuilder` with request components:
     - `Scheme` — "http" or "https"
     - `Host` — Request hostname (e.g., "functions.azurewebsites.net")
     - `Port` — Port number if specified, otherwise -1 (default port)
     - `Path` — Full request path (e.g., "/api/Status/10")

2. **Port Handling**
   - Checks `req.Host.Port.HasValue` for explicit port
   - Uses port value if present, otherwise passes -1
   - `-1` causes UriBuilder to use default port (80 for HTTP, 443 for HTTPS)

3. **Path Truncation**
   - Searches for "api/" substring in path using `IndexOf("api/")`
   - If found:
     - Truncates path to include "api/" and everything before it
     - Example: "/api/Status/10" → "/api/"
   - If not found:
     - Path remains unchanged (full path retained)

4. **URI Construction**
   - Calls `uri.ToString()` to build complete URL
   - Returns formatted URL string

**Return Value:**

- `string` — Base URL ending with "api/" if present in path
- Example returns:
  - "<https://myapp.azurewebsites.net/api/>"
  - "<http://localhost:7071/api/>"
  - "<https://customdomain.com:8080/api/>"

**Example Transformations:**

| Input Request URL | Output Base URL |
|-------------------|-----------------|
| `https://app.com/api/Status/10` | `https://app.com/api/` |
| `http://localhost:7071/api/DownloadBlob/file.xlsx` | `http://localhost:7071/api/` |
| `https://functions.net:8080/api/ValidationMessages/123` | `https://functions.net:8080/api/` |
| `https://app.com/health` | `https://app.com/health` |

**Use Cases:**

- Status page link generation (Previous/Next pagination)
- Download link construction (DownloadBlob/{fileName})
- Validation message links (ValidationMessages/{fileId})
- File upload endpoint URLs (FileUpload)
- Relative URL construction in HTML tables

**Path Truncation Logic:**

```csharp
// Before: uri.Path = "/api/Status/10"
if (uri.Path.IndexOf("api/") >= 0)
    uri.Path = uri.Path.Substring(0, uri.Path.IndexOf("api/") + 4);
// After: uri.Path = "/api/"
```

**Edge Cases:**

**Multiple "api/" Occurrences:**

- Uses first occurrence only (`IndexOf` returns first match)
- Example: "/api/test/api/Status" → "/api/"

**No "api/" in Path:**

- Path unchanged, returns full URL
- Example: "/Status/10" → "<https://app.com/Status/10>"

**Case Sensitivity:**

- `IndexOf("api/")` is case-sensitive
- "/API/Status" would not match (path unchanged)
- Azure Functions typically use lowercase "api/"

**Port Handling:**

- Port `-1` omitted from URL string by UriBuilder
- Default ports (80, 443) automatically applied by UriBuilder
- Custom ports included in output

**Integration with StatusPage:**

```csharp
// In GetStatusHTMLPage():
string baseURL = req.GetBaseURL(); // "https://app.com/api/"

// Link generation:
builder.Append($"<a href=\"{baseURL}Status/{offset}\">Next</a>");
// Renders: <a href="https://app.com/api/Status/20">Next</a>

builder.Append($"<a href=\"{baseURL}DownloadBlob/{fileName}\">Download</a>");
// Renders: <a href="https://app.com/api/DownloadBlob/data.xlsx">Download</a>
```

**Security Considerations:**

- No validation of scheme (accepts any protocol)
- No validation of host (trusts request headers)
- Vulnerable to Host header injection if not behind reverse proxy
- Azure Functions runtime provides host validation

**Performance:**

- Lightweight string manipulation
- Single `IndexOf` operation
- No network calls or I/O
- Suitable for per-request execution

**Limitations:**

- Only handles Azure Functions "api/" routing pattern
- Assumes standard Azure Functions project structure
- No support for custom route prefixes
- Case-sensitive "api/" matching

---

## Usage Patterns in PDI System

### QueueServiceExtensions Usage

**Status Page Monitoring:**

```csharp
// StatusPage.cs - GetStatusHTMLPage()
QueueClient pdiQueue = new QueueClient(connectionString, "pdi-queue");
QueueClient pubQueue = new QueueClient(connectionString, "pub-queue");

builder.Append($"PDI Queue: {pdiQueue.GetQueueLength()} messages<br/>");
builder.Append($"PUB Queue: {pubQueue.GetQueueLength()} messages<br/>");
```

**Message Enqueueing:**

```csharp
// QueueProcess.cs - After successful processing
DataTransferObject dto = new DataTransferObject(orch);
QueueClient publisherQueue = new QueueClient(connectionString, pubQueueName);

if (!publisherQueue.AddMessage(dto, log))
{
    log.LogCritical("Failed to enqueue to Publisher queue");
}
```

**Batch Processing:**

```csharp
// BatchBlob.cs - After extracting ZIP entries
foreach (var entry in zipEntries)
{
    DataTransferObject dto = new DataTransferObject(
        fileName: entry.Name,
        batchID: batchRecord.BatchID,
        retryCount: 0
    );
    
    pdiQueue.AddMessage(dto, log);
}
```

### HttpExtensions Usage

**Pagination Links:**

```csharp
// StatusPage.cs - GetStatusHTMLPage()
string baseURL = req.GetBaseURL(); // Extract base URL once

// Previous button
builder.Append($"<a href=\"{baseURL}Status/{offset - PageSize}\">Previous</a>");

// Next button  
builder.Append($"<a href=\"{baseURL}Status/{offset + PageSize}\">Next</a>");
```

**Download Links in HTML Table:**

```csharp
// StatusPage.cs - TableToHtml()
string baseURL = req.GetBaseURL();

// File download link
builder.Append($"<a href=\"{baseURL}DownloadBlob/{fileName}\">{fileName}</a>");

// Validation messages link
builder.Append($"<a href=\"{baseURL}ValidationMessages/{fileId}\">View Errors</a>");
```

**JavaScript File Upload:**

```csharp
// StatusPage.cs - GetJavaScript()
string baseURL = req.GetBaseURL();

string script = $@"
const fileUploadURL = '{baseURL}FileUpload';
fetch(fileUploadURL, requestProperties);
";
```

---

## Design Patterns Implemented

### Extension Method Pattern (Both Classes)

- Extends Azure SDK and ASP.NET Core types
- Provides fluent, discoverable API
- No inheritance or wrapper classes required

### Defensive Programming (QueueServiceExtensions)

- Existence checks before operations
- Automatic resource creation (queue creation)
- Graceful failure with boolean returns

### Idempotent Operations (QueueServiceExtensions)

- `CreateIfNotExists()` safe to call multiple times
- No side effects from repeated calls

### Single Responsibility Principle

- Each method has one clear purpose
- QueueServiceExtensions: Queue operations
- HttpExtensions: URL manipulation

---

## Dependencies

### QueueServiceExtension

- `Azure.Storage.Queues` — Queue client SDK
- `Azure.Storage.Queues.Models` — Queue metadata models
- `Microsoft.Extensions.Logging` — Logging abstractions
- `DataTransferObject` — PDI DTO class (in PDI_Azure_Function namespace)

### HttpExtension

- `Microsoft.AspNetCore.Http` — HTTP request abstractions
- `System` — UriBuilder class

---

## Known Issues and Technical Debt

### QueueServiceExtension known issues

1. **No Exception Handling**
   - `CreateIfNotExists()` can throw exceptions (not caught)
   - `SendMessage()` can throw exceptions (not caught)
   - **Recommendation:** Add try-catch with logging

2. **Misleading Log Message**
   - Hardcoded "Batch Blob Trigger" in log message
   - Used by multiple callers (not just BatchBlob)
   - **Recommendation:** Make log message dynamic or remove trigger reference

3. **No Logging on Failure**
   - Returns `false` silently when queue doesn't exist
   - Caller may not log the failure
   - **Recommendation:** Add optional error logging

4. **Approximate Count Documentation**
   - Method doesn't document approximate nature of count
   - Developers may assume real-time accuracy
   - **Recommendation:** Add XML documentation with caveat

5. **No Message Validation**
   - Doesn't validate DTO before encoding
   - No size check before enqueueing (64 KB limit)
   - **Recommendation:** Add size validation and warning

6. **Synchronous Operations**
   - No async alternatives provided
   - Blocks calling thread during network I/O
   - **Recommendation:** Add async method overloads

### HttpExtensions known issues

1. **Case-Sensitive Path Matching**
   - Only matches lowercase "api/"
   - May fail with custom routing configurations
   - **Recommendation:** Use case-insensitive comparison

2. **No URL Validation**
   - Trusts all request components
   - Vulnerable to Host header injection
   - **Recommendation:** Add URL validation or document security requirements

3. **Single "api/" Pattern Assumption**
   - Hardcoded to Azure Functions default pattern
   - Not flexible for custom route prefixes
   - **Recommendation:** Accept route prefix as parameter

4. **No Error Handling**
   - Exceptions from UriBuilder propagate
   - Invalid request components cause crashes
   - **Recommendation:** Add try-catch with fallback behavior

5. **Limited Documentation**
   - No XML documentation comments
   - Behavior not obvious from method signature
   - **Recommendation:** Add comprehensive XML docs

---

## Testing Recommendations

### QueueServiceExtensions Tests

**GetQueueLength():**

- Test with existing queue (returns count >= 0)
- Test with non-existent queue (returns -1)
- Test with empty queue (returns 0)
- Test after multiple enqueues (count increases)

**AddMessage():**

- Test with non-existent queue (creates and enqueues)
- Test with existing queue (enqueues successfully)
- Test with large DTO (verify 64 KB limit)
- Test with null logger (no logging errors)
- Test with invalid queue name (exception handling)

### HttpExtensions Tests

**GetBaseURL():**

- Test with standard Azure Functions URL
- Test with custom domain
- Test with explicit port
- Test without port (default port handling)
- Test with multiple "api/" occurrences
- Test without "api/" in path
- Test with uppercase "API/" (edge case)
- Test with localhost development URL

---
