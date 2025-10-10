# BatchBlob.cs

## Overview

**Namespace:** `PDI_Azure_Function`  
**Type:** Static Class  
**Primary Function:** Azure Function with Blob Trigger

## Purpose

The `BatchBlob` class serves as the entry point for the PDI (Publisher Data Integration) batch processing pipeline. It monitors an Azure Blob Storage container for incoming files, processes them based on their type (ZIP archives or XLSX files), and initiates the downstream processing workflow by enqueuing messages to Azure Storage Queues.

---

## Business Context

### What It Does

- **Monitors** the incoming batch container for new file uploads
- **Processes ZIP archives** by extracting individual entries and registering them for processing
- **Processes standalone XLSX files** by registering them as single-file batches
- **Maintains audit trail** through batch and file metadata recording
- **Initiates workflow** by placing messages onto processing queues
- **Archives or rejects** files based on processing outcomes

### When It Triggers

- Automatically triggers when a file is uploaded to the `IncomingBatchContainer`
- Processes files with extensions: `.zip`, `.xlsx`
- Rejects files with unsupported extensions

---

## Azure Function Details

### Function Metadata

```csharp
[FunctionName("Batch_Blob")]
```

- **Function Name:** `Batch_Blob`
- **Trigger Type:** `BlobTrigger`
- **Binding Path:** `%IncomingBatchContainer%/{name}`
- **Connection String:** `sapdi`

### Trigger Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `myBlob` | `Stream` | The blob content stream |
| `name` | `string` | The blob name/path |
| `uri` | `Uri` | The blob URI |
| `properties` | `BlobProperties` | Blob metadata including creation timestamp |
| `log` | `ILogger` | Logger instance for diagnostics |

---

## Method Documentation

### `Run` Method

#### Signature

```csharp
public static void Run(
    [BlobTrigger("%IncomingBatchContainer%/{name}", Connection = "sapdi")] Stream myBlob,
    string name,
    Uri uri,
    BlobProperties properties,
    ILogger log
)
```

#### Parameters

- **`myBlob`** - Input stream containing the uploaded file content
- **`name`** - Relative path/filename of the blob within the container
- **`uri`** - Full URI of the blob in Azure Storage
- **`properties`** - Azure Blob metadata, used primarily for `CreatedOn` timestamp
- **`log`** - ILogger instance for logging information, warnings, and critical errors

#### Return Type

`void` - Fire-and-forget processing model

---

## Processing Logic Flow

### 1. Initialization

```diagram
┌─────────────────────────────┐
│ Blob Trigger Activated      │
│ File: {name}                │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│ Load Environment Variables  │
│ - sapdi (connection string) │
│ - PDI_ConnectionString      │
│ - Container names           │
│ - Queue name                │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│ Extract Filename Only       │
│ Split path by '/'           │
└──────────┬──────────────────┘
           │
           ▼
    [File Type Check]
```

### 2. File Type Processing

#### A. ZIP File Processing

```diagram
┌─────────────────────────────────────┐
│ IF: File extension == ".zip"        │
└──────────┬──────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ PDIBatch.LoadBatch()                │
│ - Extracts ZIP entries              │
│ - Returns List<Tuple<int, string>>  │
│   (Entry ID, Entry Filename)        │
└──────────┬──────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ FOR EACH entry in ZIP:              │
│                                     │
│ 1. ExtractEntry(entryName)          │
│ 2. SaveTo("processing/{entry}")     │
│ 3. MarkEntryExtracted(entry)        │
│ 4. Create DTO & Enqueue             │
└──────────┬──────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ Move ZIP to "archive/{name}"        │
└─────────────────────────────────────┘
```

#### B. XLSX File Processing

```diagram
┌─────────────────────────────────────┐
│ ELSE IF: File extension == ".xlsx"  │
└──────────┬──────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ PDIBatch.RecordBatch()              │
│ - Creates batch record in DB        │
│ - Assigns Batch_ID                  │
└──────────┬──────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ PDIBatch.RecordSingle()             │
│ - Creates single file record        │
└──────────┬──────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ Move file to "processing/{name}"    │
└──────────┬──────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ MarkEntryExtracted(name)            │
└──────────┬──────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ Create DTO & Enqueue to PDI Queue   │
└─────────────────────────────────────┘
```

#### C. Unsupported File Processing

```diagram
┌─────────────────────────────────────┐
│ ELSE: Unsupported extension         │
└──────────┬──────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ Move to "rejected/{name}"           │
│ Log: Unrecognized file              │
└─────────────────────────────────────┘
```

### 3. Queue Message Creation

```diagram
┌─────────────────────────────────────┐
│ Create DataTransferObject           │
│ - FileName                          │
│ - Batch_ID                          │
│ - RetryCount                        │
└──────────┬──────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ Base64Encode DTO                    │
└──────────┬──────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ QueueClient.SendMessage()           │
│ Target: PDI_QueueName               │
└─────────────────────────────────────┘
```

---

## Dependencies

### External Libraries

- `Microsoft.Azure.WebJobs` - Azure Functions runtime and attributes
- `Microsoft.Extensions.Logging` - Structured logging
- `Azure.Storage.Blobs` - Blob storage operations
- `Azure.Storage.Queues` - Queue storage operations
- `Azure.Storage.Blobs.Models` - Blob metadata models

### Internal Dependencies

- `Publisher_Data_Operations.Helper.PDIBatch` - Batch processing business logic
- `Publisher_Data_Operations.Helper.PDIStream` - Stream wrapper for file processing
- `PDI_Azure_Function.Extensions.BlobServiceExtensions` - Extension methods for blob operations
- `PDI_Azure_Function.DataTransferObject` - Queue message payload structure

---

## Environment Variables

| Variable Name | Purpose | Usage |
|---------------|---------|-------|
| `sapdi` | Azure Storage connection string | Authenticates blob and queue operations |
| `PDI_ConnectionString` | SQL Database connection string | Used by `PDIBatch` for metadata persistence |
| `IncomingBatchContainer` | Source container name | Monitored container for incoming files |
| `MainContainer` | Processing container name | Destination for extracted/processing files |
| `PDI_QueueName` | Processing queue name | Queue for downstream processing triggers |

---

## Container Structure

### Blob Container Paths

```diagram
IncomingBatchContainer/
  └── {uploaded-files}         → Trigger point

MainContainer/
  ├── processing/              → Files ready for processing
  │   └── {filename}.xlsx
  ├── archive/                 → Successfully processed ZIP files
  │   └── {filename}.zip
  └── rejected/                → Unsupported/invalid files
      └── {filename}
```

---

## Error Handling

### Critical Errors (LogCritical)

1. **Failed to mark entry extracted** - Database update failure after successful blob operation
2. **Unable to create or access queue** - Queue availability issues
3. **Failed to extract/write from ZIP** - Blob write operation failure
4. **Failed to move blob** - Blob movement operation failure
5. **Failed to create file/batch record** - Database insert failure

### Exception Handling

```csharp
try {
    // Main processing logic
}
catch (Exception err) {
    log.LogCritical(err, err.Message);
}
```

- All exceptions are caught and logged with full stack trace
- Processing stops on exception; no retry logic at this level
- Failed files remain in `IncomingBatchContainer` for manual review

---

## Key Operations

### ZIP Processing Operations

1. `PDIBatch.LoadBatch(PDIStream, CreatedOn)` - Extracts ZIP and creates batch record
2. `PDIBatch.ExtractEntry(entryName)` - Returns stream of individual entry
3. `BlobServiceClient.SaveTo(stream, path, container)` - Saves extracted entry
4. `PDIBatch.MarkEntryExtracted(entryName)` - Updates entry status in database
5. `QueueClient.AddMessage(DTO)` - Enqueues processing message

### XLSX Processing Operations

1. `PDIBatch.RecordBatch(fileName, createdOn)` - Creates batch record
2. `PDIBatch.RecordSingle(fileName)` - Creates single file entry record
3. `BlobServiceClient.MoveTo(source, dest)` - Moves file to processing folder
4. `PDIBatch.MarkEntryExtracted(fileName)` - Updates file status
5. `QueueClient.SendMessage(DTO.Base64Encode())` - Enqueues processing message

---

## Business Rules

### File Acceptance Criteria

- **ZIP files** - Must contain valid XLSX entries
- **XLSX files** - Must be standalone Excel workbooks
- **All other extensions** - Rejected and moved to rejected container

### Batch ID Assignment

- ZIP files: Single `Batch_ID` assigned to the ZIP; all entries share this ID
- XLSX files: Each file receives its own `Batch_ID` as a single-entry batch

### Retry Logic

- `RetryCount` is read from `PDIBatch` and passed to `DataTransferObject`
- Actual retry logic handled by downstream queue processors
- Initial `RetryCount` value determined by batch configuration

---

## State Transitions

### File Lifecycle

```note
[Uploaded] → IncomingBatchContainer/{name}
    │
    ├─→ [ZIP] → Extracted → MainContainer/processing/{entry}
    │                     → IncomingBatchContainer → MainContainer/archive/{name}
    │
    ├─→ [XLSX] → Moved → MainContainer/processing/{name}
    │
    └─→ [Other] → Moved → MainContainer/rejected/{name}
```

### Database State

```note
[Upload Detected]
    │
    ├─→ [ZIP]
    │     └─→ Batch Record Created (Batch_ID assigned)
    │         └─→ Entry Records Created (per file in ZIP)
    │             └─→ Entry Status: Extracted
    │
    └─→ [XLSX]
          └─→ Batch Record Created (Batch_ID assigned)
              └─→ Single File Record Created
                  └─→ File Status: Extracted
```

---

## Logging Strategy

### Information Level

- Function start with blob name and size
- Successful queue message creation
- Directory creation events
- Unrecognized file handling

### Critical Level

- Entry extraction marking failures
- Queue access/creation failures
- Blob extraction/write failures
- Blob movement failures
- Database operation failures (batch/file creation)

---

## Performance Considerations

### Scalability

- Function instances scale based on blob trigger queue depth
- Each blob triggers independent function execution
- Large ZIP files processed sequentially (entries extracted one-by-one)

### Optimization Opportunities

- Parallel extraction of ZIP entries (currently sequential)
- Batch queue message creation (currently individual messages)
- Connection string caching (currently loaded per invocation)

---

## Security Considerations

### Authentication

- Uses connection string authentication (`sapdi`) for Azure Storage
- Connection strings stored in Application Settings (environment variables)

### Data Protection

- Files stored in Azure Blob Storage with encryption at rest
- No sensitive data logged (only filenames and metadata)

---

## Related Components

### Upstream

- **External Systems** - Fund managers/partners uploading files via SFTP, API, or portal

### Downstream

- **QueueProcess.cs** - Consumes messages from `PDI_QueueName`
- **PDIBatch** - Manages batch metadata and database operations
- **Publisher Import Pipeline** - Ultimate consumer of processed data

---

## Testing Considerations

### Test Scenarios

1. Upload valid ZIP with multiple XLSX entries
2. Upload single standalone XLSX file
3. Upload unsupported file type (e.g., .txt, .csv)
4. Upload ZIP with corrupted entries
5. Upload file when database is unavailable
6. Upload file when queue is unavailable
7. Upload empty ZIP file
8. Upload very large ZIP file (>100MB)

### Edge Cases

- Empty ZIP files
- ZIP files with nested directories
- Files with special characters in names
- Duplicate filenames across batches
- Network interruptions during processing
