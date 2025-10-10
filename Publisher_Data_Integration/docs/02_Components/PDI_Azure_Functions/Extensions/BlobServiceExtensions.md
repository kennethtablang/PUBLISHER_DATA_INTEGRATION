# Class: BlobServiceExtensions

**One-line purpose**  
Static extension methods for Azure Blob Storage operations that provide simplified blob lifecycle management including move, copy, save, open, find, and listing functionality with synchronous wrappers and error handling.

---

## What this class does (business view)

- Provides extension methods that simplify Azure Blob Storage SDK operations for common file management tasks.
- Implements blob movement between containers and directories (copy + delete pattern).
- Handles blob copying with synchronous waiting for completion and status verification.
- Enables stream-based blob saving with automatic container creation and overwrite support.
- Provides blob opening functionality that loads entire blob into memory stream for processing.
- Implements intelligent blob search across multiple standard directories (completed, rejected, processing).
- Generates HTML listings of blob container contents with metadata (name, creation date, size).
- Wraps asynchronous Azure SDK operations with synchronous methods for Azure Functions compatibility.
- Implements robust error handling with optional logging integration.

---

## Key functions & patterns (what to look for)

- **Move Pattern** — `MoveTo()` delegates to `CopyTo()` with delete flag set to true.
- **Synchronous Copy** — Uses `WaitForCompletion()` and polling loop to ensure blob copy finishes before continuing.
- **Automatic Container Creation** — Destination containers are created automatically if they don't exist.
- **Multi-Directory Search** — `Find()` searches standard directories: root → completed → rejected → processing.
- **Memory Stream Loading** — `Open()` downloads entire blob into `MemoryStream` for in-memory processing.
- **Task.Result Blocking** — Wraps async methods with `Task.Run()` and `.Result` for synchronous execution.
- **Optional Logging** — Most methods accept optional `ILogger` parameter for error tracking.
- **HTML Generation** — `GetListing()` returns HTML table via `TableToHtml()` extension for web UI integration.

---

## Classes & Methods (detailed)

### `BlobServiceExtensions` — file: `PDI_Azure_Function/Extensions/BlobServiceExtensions.cs`

**Purpose:** Static class providing extension methods for `BlobServiceClient`, `BlobClient`, and `BlobContainerClient` that simplify blob operations and integrate with the PDI processing pipeline.

**Key Characteristics:**

- All methods are static extension methods
- Synchronous wrappers around asynchronous Azure SDK operations
- Designed for Azure Functions synchronous execution context
- Implements polling-based wait patterns for async operations
- Optional logging integration for error tracking
- Memory-efficient stream handling with position reset

---

### Extension Methods

#### `MoveTo`

```csharp
public static bool MoveTo(
    this BlobServiceClient blobService, 
    string sourceContainer, 
    string sourcePath, 
    string destContainer, 
    string destPath, 
    ILogger log = null)
```

**Purpose:** Moves a blob from source location to destination location by copying then deleting the source blob.

**Parameters:**

- `blobService` (BlobServiceClient) — Azure Blob Service client instance
- `sourceContainer` (string) — Source container name
- `sourcePath` (string) — Source blob path/filename (e.g., "processing/file.xlsx")
- `destContainer` (string) — Destination container name
- `destPath` (string) — Destination blob path/filename (e.g., "completed/file.xlsx")
- `log` (ILogger, optional) — Logger for error tracking

**Implementation:**

- Delegates to `CopyTo()` method with `deleteSource = true`
- Atomic at the operation level (copy must succeed before delete)
- Source blob deleted only if copy completes successfully

**Return Value:**

- `true` — Blob successfully copied and source deleted
- `false` — Operation failed (source blob remains)

**Use Cases:**

- Moving files between processing stages (processing → importing → completed)
- Moving failed files to rejected container
- Blob lifecycle management in processing pipeline

**Example Usage:**

```csharp
bool success = blobServiceClient.MoveTo(
    "maincontainer", 
    "processing/data.xlsx", 
    "maincontainer", 
    "completed/data.xlsx", 
    log);
```

---

#### `CopyTo`

```csharp
public static bool CopyTo(
    this BlobServiceClient blobService, 
    string sourceContainer, 
    string sourcePath, 
    string destContainer, 
    string destPath, 
    bool deleteSource = false, 
    ILogger log = null)
```

**Purpose:** Copies a blob from source location to destination location with optional source deletion, automatic container creation, and synchronous completion waiting.

**Parameters:**

- `blobService` (BlobServiceClient) — Azure Blob Service client instance
- `sourceContainer` (string) — Source container name
- `sourcePath` (string) — Source blob path/filename
- `destContainer` (string) — Destination container name
- `destPath` (string) — Destination blob path/filename
- `deleteSource` (bool, default: false) — Delete source blob after successful copy
- `log` (ILogger, optional) — Logger for error tracking

**Workflow:**

1. **Container Client Initialization**
   - Gets source container client
   - Gets destination container client
   - **Early Exit:** Returns `false` if source container doesn't exist

2. **Destination Container Creation**
   - Calls `destContainer.CreateIfNotExists()`
   - Ensures destination container exists before copy operation
   - Idempotent operation (safe to call multiple times)

3. **Blob Client Initialization**
   - Gets source blob client from `sourcePath`
   - Gets destination blob client from `destPath`
   - **Early Exit:** Returns `false` if source blob doesn't exist

4. **Copy Operation Initiation**
   - Calls `dest.StartCopyFromUri(source.Uri)` to begin async copy
   - Azure performs server-side copy (efficient for large files)
   - Returns `CopyFromUriOperation` handle for status tracking

5. **Synchronous Wait for Completion**
   - Calls `res.WaitForCompletion()` to block until initial response
   - Gets destination blob properties via `dest.GetProperties()`
   - **Polling Loop:**
     - While `CopyStatus == CopyStatus.Pending`:
       - Sleeps for 1000ms (1 second)
       - Re-checks properties via `dest.GetProperties()`
     - Ensures copy operation completes before proceeding

6. **Success Path** (if `CopyStatus == CopyStatus.Success`)
   - **If deleteSource = true:**
     - Deletes source blob via `source.DeleteIfExists()`
     - Returns boolean from delete operation
   - **If deleteSource = false:**
     - Returns `true` (copy succeeded, source retained)

7. **Failure Path** (if CopyStatus != Success)
   - Logs critical error with copy status description (if logger provided)
   - Returns `false`

8. **Exception Handling**
   - Catches all exceptions during copy operation
   - Logs critical error with message and stack trace (if logger provided)
   - Returns `false`

**Copy Status Values:**

- `CopyStatus.Success` — Copy completed successfully
- `CopyStatus.Pending` — Copy in progress (polling continues)
- `CopyStatus.Aborted` — Copy was aborted
- `CopyStatus.Failed` — Copy failed

**Performance Considerations:**

- Server-side copy (efficient for large files, no data transfer through client)
- Polling interval: 1 second (may be slow for small files, efficient for large files)
- Blocks calling thread during copy operation
- Memory efficient (no buffer required)

**Error Scenarios:**

- Source container doesn't exist → Returns `false` (no logging)
- Source blob doesn't exist → Returns `false` (no logging)
- Copy operation fails → Logs critical, returns `false`
- Exception during copy → Logs critical, returns `false`
- Delete fails (when deleteSource = true) → Returns `false` from `DeleteIfExists()`

**Return Value:**

- `true` — Copy succeeded (and source deleted if requested)
- `false` — Operation failed at any stage

---

#### `SaveTo`

```csharp
public static bool SaveTo(
    this BlobServiceClient blobService, 
    Stream sourceStream, 
    string filePath, 
    string destContainer)
```

**Purpose:** Saves a stream to blob storage with automatic container creation, stream position reset, and overwrite support.

**Parameters:**

- `blobService` (BlobServiceClient) — Azure Blob Service client instance
- `sourceStream` (Stream) — Stream containing data to upload
- `filePath` (string) — Destination blob path/filename
- `destContainer` (string) — Destination container name

**Workflow:**

1. **Container Preparation**
   - Gets destination container client
   - Calls `CreateIfNotExists()` to ensure container exists
   - Idempotent operation (safe if container already exists)

2. **Blob Client Initialization**
   - Gets blob client for `filePath` in destination container

3. **Stream Position Reset**
   - Checks if stream is seekable (`sourceStream.CanSeek`)
   - If seekable and position != 0, resets position to 0
   - Ensures entire stream content is uploaded

4. **Upload Operation**
   - Calls `dest.Upload(sourceStream, overwrite: true)`
   - Overwrites existing blob if present
   - Uploads entire stream from current position to end

5. **Success Verification**
   - Checks HTTP response status code == 201 (Created)
   - Verifies blob exists via `dest.Exists()`
   - Both conditions must be true for success

**Return Value:**

- `true` — Stream uploaded successfully (HTTP 201) and blob exists
- `false` — Upload failed or blob doesn't exist after upload

**Use Cases:**

- Saving processed streams to archive containers
- Creating backup copies of uploaded files
- Storing transformed data as new blobs

**Stream Handling:**

- Does not dispose stream (caller's responsibility)
- Resets position if seekable (safe for reuse)
- Uploads from current position if not seekable

**Error Handling:**

- No exception handling (exceptions propagate to caller)
- No logging integration

---

#### `Open` (BlobServiceClient Extension)

```csharp
public static Stream Open(
    this BlobServiceClient blobService, 
    string filePath, 
    string sourceContainerName)
```

**Purpose:** Opens a blob and returns null if blob or container doesn't exist, or returns a stream for reading blob content.

**Parameters:**

- `blobService` (BlobServiceClient) — Azure Blob Service client instance
- `filePath` (string) — Blob path/filename
- `sourceContainerName` (string) — Source container name

**Workflow:**

1. **Container Existence Check**
   - Gets container client for `sourceContainerName`
   - Checks `Exists()` on container
   - **Early Exit:** Returns `null` if container doesn't exist

2. **Blob Client Initialization**
   - Gets blob client for `filePath`
   - Checks `Exists()` on blob

3. **Stream Opening**
   - **If blob exists:**
     - Calls `blobClient.Open()` (delegates to overload below)
     - Returns stream for reading
   - **If blob doesn't exist:**
     - Returns `null`

**Return Value:**

- `Stream` — Memory stream containing blob content (if blob exists)
- `null` — Container or blob doesn't exist

**Use Cases:**

- Loading files for processing (e.g., `PDIStream` initialization)
- Reading configuration blobs (e.g., templates)
- Validating blob existence before processing

**Error Handling:**

- No exception handling (exceptions propagate to caller)
- Returns null for missing blobs (caller should check for null)

---

#### `Open` (BlobClient Extension)

```csharp
public static Stream Open(this BlobClient source)
```

**Purpose:** Downloads a blob into a memory stream and returns the stream positioned at the beginning for reading.

**Parameters:**

- `source` (BlobClient) — Blob client for the blob to open

**Workflow:**

1. **Memory Stream Creation**
   - Creates new `MemoryStream` instance
   - Empty stream ready for download

2. **Blob Download**
   - **If blob exists:**
     - Calls `source.DownloadTo(memStream)` to download entire blob
     - Blob content written to memory stream
   - **If blob doesn't exist:**
     - Stream remains empty

3. **Stream Position Reset**
   - Checks if stream is seekable and position != 0
   - Resets position to 0 if needed
   - Ensures stream is ready for reading from beginning

**Return Value:**

- `MemoryStream` — Stream containing blob content positioned at start
- Empty stream if blob doesn't exist (not null)

**Memory Considerations:**

- **Critical:** Entire blob loaded into memory
- Suitable for files up to ~100MB (Azure Functions memory limits)
- Not suitable for large files (consider streaming approach)
- Caller must dispose stream to release memory

**Use Cases:**

- Loading Excel files for EPPlus processing
- Loading templates for validation
- In-memory file processing

**Differences from BlobServiceClient.Open:**

- Always returns a stream (never null)
- Downloads entire blob into memory immediately
- Returns empty stream if blob doesn't exist

---

#### `Find`

```csharp
public static BlobClient Find(
    this BlobServiceClient blobService, 
    string fileName, 
    string sourceContainerName)
```

**Purpose:** Searches for a blob across multiple standard directories in a container, checking root and common subdirectories in order.

**Parameters:**

- `blobService` (BlobServiceClient) — Azure Blob Service client instance
- `fileName` (string) — File name to search for (without directory path)
- `sourceContainerName` (string) — Container name to search within

**Search Order:**

1. **Root directory:** `{fileName}`
2. **Completed directory:** `completed/{fileName}`
3. **Rejected directory:** `rejected/{fileName}`
4. **Processing directory:** `processing/{fileName}`

**Workflow:**

1. **Container Existence Check**
   - Gets container client for `sourceContainerName`
   - Checks `Exists()` on container
   - **Early Exit:** Returns `null` if container doesn't exist

2. **Sequential Directory Search**
   - Gets blob client for `{fileName}` (root)
   - If not exists, gets blob client for `completed/{fileName}`
   - If not exists, gets blob client for `rejected/{fileName}`
   - If not exists, gets blob client for `processing/{fileName}`
   - Returns first matching blob client found

3. **Return Value**
   - Returns `BlobClient` for found blob (may or may not exist)
   - Returns `null` if container doesn't exist
   - **Note:** Last returned blob client may not exist (caller should check)

**Use Cases:**

- Downloading files from status page (file could be in any stage)
- Finding files for reprocessing
- Locating files for audit/debugging

**Important Behavior:**

- Returns blob client even if blob doesn't exist in final checked location
- Caller must call `.Exists()` on returned blob client to verify
- Does not check `importing/` directory (not in search path)

**Example Usage:**

```csharp
BlobClient blob = blobService.Find("data.xlsx", "maincontainer");
if (blob != null && blob.Exists())
{
    // Process blob
}
```

**Missing Directories:**

- `importing/` — Not included in search (between processing and completed)
- `archive/` — Not included in search
- Custom directories — Not included in search

---

#### `GetListing`

```csharp
public static string GetListing(
    this BlobContainerClient blobContainerClient, 
    int? segmentSize)
```

**Purpose:** Synchronous wrapper that retrieves blob container listing and converts it to HTML table format for web UI display.

**Parameters:**

- `blobContainerClient` (BlobContainerClient) — Container to list
- `segmentSize` (int?, optional) — Maximum number of blobs to retrieve

**Implementation:**

- Wraps `GetBlobsAsync()` in `Task.Run()` for synchronous execution
- Blocks on `.Result` to wait for completion
- Returns HTML string directly

**Return Value:**

- `string` — HTML table with blob listing
- HTML contains: file names, creation dates, and sizes

**Use Cases:**

- Status page blob container display
- Monitoring incoming batch container
- Debugging blob storage state

**Performance Considerations:**

- Blocks calling thread until listing completes
- Suitable for small containers (< 1000 blobs)
- Segment size limits memory usage

---

#### `GetBlobsAsync` (Private)

```csharp
private static async Task<string> GetBlobsAsync(
    this BlobContainerClient blobContainerClient,
    int? segmentSize)
```

**Purpose:** Asynchronous method that retrieves blob listing with paging, builds a DataTable, and converts it to HTML table format.

**Parameters:**

- `blobContainerClient` (BlobContainerClient) — Container to list
- `segmentSize` (int?, optional) — Page size for blob listing

**Workflow:**

1. **DataTable Initialization**
   - Creates DataTable named "Blob Contents"
   - Defines columns:
     - `File Name` (string) — Blob name/path
     - `Created On` (DateTimeOffset) — Creation timestamp
     - `File Size (KB)` (long) — File size in kilobytes

2. **Paged Blob Retrieval**
   - Calls `GetBlobsAsync().AsPages(default, segmentSize)` for paged results
   - Returns `IAsyncEnumerable<Page<BlobItem>>` for async iteration
   - Azure SDK handles pagination automatically

3. **Blob Enumeration**
   - Iterates through each page asynchronously (`await foreach`)
   - For each page, iterates through blob items
   - **Directory Filtering:** Skips blobs ending with "/" (directory markers)
   - Adds row to DataTable:
     - Blob name
     - Creation timestamp
     - Content length / 1024 (converts bytes to KB)

4. **DataTable Finalization**
   - Calls `AcceptChanges()` to commit all rows
   - Prepares DataTable for HTML conversion

5. **HTML Conversion**
   - Calls `TableToHtml()` extension method on DataTable
   - Returns HTML table string

6. **Exception Handling**
   - Catches `RequestFailedException` from Azure SDK
   - Logs to console (not ILogger)
   - Re-throws exception after logging

**Return Value:**

- `string` — HTML table with blob listing
- Exception propagated if Azure request fails

**DataTable Structure:**

```note
File Name       | Created On                | File Size (KB)
----------------|---------------------------|---------------
data.xlsx       | 2025-01-15 10:30:00 +00:00| 256
report.pdf      | 2025-01-15 11:45:00 +00:00| 1024
```

**Error Scenarios:**

- Azure request fails → Exception logged and re-thrown
- Container doesn't exist → Azure SDK throws exception
- Network issues → Azure SDK throws exception

**Performance Considerations:**

- Async enumeration (efficient memory usage)
- Paging reduces memory footprint for large containers
- DataTable held in memory (suitable for small result sets)

---

## Design Patterns Implemented

### Extension Method Pattern

- All methods extend Azure SDK types (`BlobServiceClient`, `BlobClient`, `BlobContainerClient`)
- Provides fluent, discoverable API
- Integrates seamlessly with existing Azure SDK code

### Synchronous Wrapper Pattern

- `GetListing()` wraps `GetBlobsAsync()` with `Task.Run().Result`
- Enables synchronous execution in Azure Functions v3 context
- Blocks calling thread (trade-off for simpler code)

### Polling Pattern (CopyTo)

- Waits for async operation completion using polling loop
- Checks status every 1 second
- Ensures operation completes before returning

### Automatic Resource Creation Pattern

- `CopyTo()` and `SaveTo()` create destination containers automatically
- Idempotent operations (safe to call multiple times)
- Reduces caller's responsibility for resource management

### Search Pattern (Find)

- Searches multiple standard locations sequentially
- Returns first match found
- Abstracts directory structure from caller

---

## Error Handling Strategy

### Logged Errors (with ILogger)

- `CopyTo()` — Logs critical errors for copy failures and exceptions

### Silent Failures (Return False)

- `MoveTo()` — Returns false on failure (delegates to CopyTo)
- `CopyTo()` — Returns false for non-existent containers/blobs
- `SaveTo()` — Returns false if upload fails or blob doesn't exist after upload

### Null Returns

- `Open()` (BlobServiceClient) — Returns null for missing containers/blobs
- `Find()` — Returns null if container doesn't exist

### Exception Propagation

- `Open()` (BlobClient) — Exceptions propagate to caller
- `SaveTo()` — Exceptions propagate to caller
- `GetBlobsAsync()` — Catches, logs, and re-throws `RequestFailedException`

---

## Performance Characteristics

| Method | Blocking | Memory Usage | Network Calls |
|--------|----------|--------------|---------------|
| `MoveTo` | Yes (polling) | Low | 2+ (copy status checks + delete) |
| `CopyTo` | Yes (polling) | Low | 2+ (copy status checks) |
| `SaveTo` | Yes | Stream size | 1 (upload) |
| `Open` (Service) | Yes | Blob size | 1 (download) |
| `Open` (Client) | Yes | Blob size | 1 (download) |
| `Find` | No | Minimal | 1 (container check only) |
| `GetListing` | Yes (Task.Result) | Result set size | 1+ (paged requests) |

---

## Usage Recommendations

### When to Use Each Method

**MoveTo:**

- Moving files between processing stages
- Implementing file lifecycle (processing → completed → archive)
- Cleanup operations after successful processing

**CopyTo:**

- Creating backups without removing originals
- Duplicating files for parallel processing
- Archiving with source retention

**SaveTo:**

- Saving processed streams to storage
- Creating new blobs from generated content
- Backup operations for in-memory data

**Open (Service):**

- Loading files when you need null-safety checks
- Optional file loading (template may not exist)
- Validating file existence before processing

**Open (Client):**

- Loading files when blob client already available
- Processing files in memory (EPPlus, validation)
- Files under 100MB (memory constraint)

**Find:**

- Locating files across processing stages
- Download endpoints (file could be anywhere)
- Debugging and audit operations

**GetListing:**

- Status page blob listings
- Monitoring container contents
- Debugging storage state

---

## Known Issues and Technical Debt

1. **Memory Usage in Open()**
   - Loads entire blob into memory
   - Not suitable for large files (>100MB)
   - **Recommendation:** Add streaming alternative or size check

2. **Polling Interval in CopyTo()**
   - Fixed 1-second interval may be inefficient for small files
   - No exponential backoff
   - **Recommendation:** Implement adaptive polling or configurable interval

3. **Error Handling Inconsistency**
   - Some methods log errors, others don't
   - Some return null, others return false
   - **Recommendation:** Standardize error handling strategy

4. **Find() Return Behavior**
   - Returns blob client even if blob doesn't exist in final location
   - Caller must check `.Exists()` on returned client
   - **Recommendation:** Return null if blob not found, or add boolean out parameter

5. **Task.Result Blocking**
   - `GetListing()` uses blocking `.Result` pattern
   - Can cause deadlocks in async contexts
   - **Recommendation:** Provide async alternatives or use ConfigureAwait(false)

6. **No Async Alternatives**
   - All methods are synchronous wrappers
   - Limits scalability for high-throughput scenarios
   - **Recommendation:** Add async method overloads

7. **Console Logging in GetBlobsAsync()**
   - Uses `Console.WriteLine()` instead of ILogger
   - Not integrated with Application Insights
   - **Recommendation:** Accept ILogger parameter

8. **Missing Directory in Find()**
   - Does not search `importing/` directory
   - May fail to find files in transit between processing stages
   - **Recommendation:** Add `importing/` to search path

9. **No Cancellation Token Support**
   - Long-running operations cannot be cancelled
   - Blocks until completion or timeout
   - **Recommendation:** Add CancellationToken parameters

10. **SaveTo Error Handling**
    - No exception handling (exceptions propagate)
    - No logging for failures
    - **Recommendation:** Add try-catch with logging

---

## Dependencies

### Azure SDK

- `Azure.Storage.Blobs` — Blob storage client library
- `Azure.Storage.Blobs.Models` — Blob metadata models
- `Azure` — Core Azure types (Page, RequestFailedException)

### .NET Framework

- `System.IO` — Stream handling
- `System.Data` — DataTable for HTML generation
- `System.Threading.Tasks` — Async/await support

### External Methods

- `TableToHtml()` — DataTable extension (defined in StatusPage.cs)
