# DataTransferObject.cs

## Overview

**Namespace:** `PDI_Azure_Function`  
**Type:** Class (POCO - Plain Old CLR Object)  
**Primary Function:** Data Transfer Object (DTO) / Message Envelope

## Purpose

The `DataTransferObject` class serves as the standardized message payload for Azure Queue Storage communication throughout the PDI processing pipeline. It encapsulates file processing metadata, batch information, status tracking, and error details required for asynchronous processing across multiple Azure Functions.

---

## Business Context

### What It Does

- **Encapsulates** all necessary context for file processing across function boundaries
- **Serializes** to JSON and Base64 encoding for Azure Queue message compatibility
- **Maintains** processing state including retry counts and error messages
- **Carries** batch and job identifiers for traceability
- **Supports** multiple construction patterns for different processing stages

### When It's Used

- Created in `BatchBlob` when files are initially extracted/registered
- Passed through `PDI_QueueName` to `QueueProcess` function
- Transformed and passed to `PUB_QueueName` for Publisher import
- Reconstructed from queue messages at each processing stage
- Used in HTTP endpoints for status reporting and file operations

---

## Class Structure

### Properties

| Property | Type | Description | Purpose |
|----------|------|-------------|---------|
| `File_ID` | `string` | Unique file identifier from database | Tracks individual file records in PDI system |
| `FileName` | `string` | Original filename (with extension) | Identifies the file being processed |
| `FileRunID` | `string` | Execution/run identifier | Correlates processing attempts for same file |
| `FileStatus` | `bool?` | Processing success indicator | `true` = success, `false` = failure, `null` = pending |
| `Job_ID` | `string` | Publisher job identifier | Links to Publisher import job |
| `Batch_ID` | `string` | Batch identifier from PDI system | Groups related files from same upload |
| `ProcessStatus` | `ProcessStatusObject` | Detailed processing status | Contains granular status information |
| `FileDetails` | `FileDetailsObject` | File metadata and details | Stores file-specific information |
| `ErrorMessage` | `string` | Processing error description | Captures failure reasons for troubleshooting |
| `RetryCount` | `int` | Number of retry attempts | Tracks retry logic and circuit breaking |
| `NotificationEmailAddress` | `string` | Recipient for notifications | Email address for batch completion/failure alerts |

---

## Constructor Overloads

### 1. Default Constructor

```csharp
public DataTransferObject()
```

**Purpose:** Creates empty DTO with all properties initialized to `null` or default values

**Initialization:**

- All string properties → `null`
- `FileStatus` → `null`
- `RetryCount` → `0`
- Complex objects (`ProcessStatus`, `FileDetails`) → `null`

**Usage Scenario:**

- Manual property population
- Placeholder/empty message creation
- Testing scenarios

---

### 2. JSON Deserialization Constructor

```csharp
public DataTransferObject(string jsonString)
```

**Purpose:** Reconstructs DTO from JSON-encoded queue message

**Parameters:**

- `jsonString` - JSON representation of a `DataTransferObject`

**Process:**

1. Deserializes JSON string to dynamic object using `Newtonsoft.Json`
2. Uses null-conditional operators (`?.`) to safely extract properties
3. Applies null-coalescing for `RetryCount` to default to `0`

**Usage Scenario:**

- Azure Queue message consumption
- Receiving messages from `PDI_QueueName` or `PUB_QueueName`
- HTTP POST endpoint payload deserialization

**Error Handling:**

- Silently handles missing properties (assigns `null`)
- Uses `??` operator to provide fallback for `RetryCount`

---

### 3. JSON + Form Values Constructor

```csharp
public DataTransferObject(string jsonString, NameValueCollection values)
```

**Purpose:** Reconstructs DTO from JSON with form/query parameter overrides

**Parameters:**

- `jsonString` - Base JSON representation
- `values` - Name-value collection (HTTP form data or query parameters)

**Process:**

1. Deserializes JSON to base object
2. For each property, prioritizes form value over JSON value
3. Uses cascading null-coalescing: `formValue ?? jsonValue ?? defaultValue`

**Usage Scenario:**

- HTTP endpoints that accept both JSON body and form parameters
- StatusPage file upload with embedded metadata
- Testing/debugging with parameter overrides

**Priority Chain:**

```note
Form/Query Parameter → JSON Property → null/default
```

---

### 4. Minimal Constructor (Batch Initial)

```csharp
public DataTransferObject(string fileName, int batchID, int retryCount)
```

**Purpose:** Creates lightweight DTO for initial batch processing

**Parameters:**

- `fileName` - Name of the file to process
- `batchID` - Batch identifier from `PDIBatch`
- `retryCount` - Initial retry count (usually from batch configuration)

**Initialization:**

- `FileName` → provided value
- `Batch_ID` → `batchID.ToString()`
- `RetryCount` → provided value
- All other properties → `null` (default)

**Usage Scenario:**

- `BatchBlob` after extracting ZIP entries or registering single XLSX
- Initial queue message creation when minimal context is known
- First stage of processing pipeline

**Example:**

```csharp
var dto = new DataTransferObject("Report_2024.xlsx", 12345, 3);
// Result: FileName="Report_2024.xlsx", Batch_ID="12345", RetryCount=3
```

---

### 5. Orchestration Constructor

```csharp
public DataTransferObject(Orchestration orch)
```

**Purpose:** Builds comprehensive DTO from `Orchestration` processing result

**Parameters:**

- `orch` - `Orchestration` object containing complete processing context

**Process:**

1. Null-checks `orch` parameter
2. Extracts properties from `orch.GetFile` (null-safe with conditional checks)
3. Converts numeric IDs to strings
4. Copies status, details, and error information

**Property Mapping:**

```diagram
File_ID ← orch.GetFile.FileID
Batch_ID ← orch.GetFile.BatchID
FileName ← orch.GetFile.OnlyFileName
FileStatus ← orch.FileStatus
FileRunID ← orch.FileRunID
ProcessStatus ← orch.GetProcessStatus
FileDetails ← orch.GetFileDetails
Job_ID ← orch.GetFile.JobID
ErrorMessage ← orch.ErrorMessage
RetryCount ← orch.RetryCount
NotificationEmailAddress ← orch.NotificationEmailAddress
```

**Usage Scenario:**

- After `Orchestration.ProcessFile()` completes in `QueueProcess`
- Before enqueuing to `PUB_QueueName` for Publisher import
- Creating response payload after processing operations

**Commented Fields:**

```csharp
// DocumentType = orch.GetFile.DocumentType;        // Future use
// Company_ID = orch.GetFile.CompanyID.ToString();  // Future use
// Document_Type_ID = orch.GetFile.DocumentTypeID.ToString(); // Future use
```

---

## Methods

### `Base64Encode()`

**Signature:**

```csharp
public string Base64Encode()
```

**Purpose:** Serializes DTO to Base64-encoded JSON for Azure Queue message

**Process:**

```note
DataTransferObject
    ↓
JsonConvert.SerializeObject(this)  → JSON string
    ↓
Encoding.UTF8.GetBytes(...)        → Byte array
    ↓
Convert.ToBase64String(...)        → Base64 string
```

**Returns:** Base64-encoded string suitable for `QueueClient.SendMessage()`

**Usage Example:**

```csharp
DataTransferObject dto = new DataTransferObject("file.xlsx", 123, 0);
string message = dto.Base64Encode();
queueClient.SendMessage(message);
```

**Why Base64?**

- Azure Queue messages must be UTF-8 encoded or Base64-encoded
- JSON may contain characters incompatible with queue message format
- Base64 ensures safe transport through queue infrastructure

---

## Processing Pipeline Flow

### Stage 1: Initial Batch Processing

```note
[BatchBlob] → DataTransferObject(fileName, batchID, retryCount)
    ↓
Base64Encode()
    ↓
QueueClient.SendMessage() → PDI_QueueName
```

### Stage 2: Queue Processing

```note
QueueClient.Receive() → Base64 message string
    ↓
Base64Decode()
    ↓
DataTransferObject(jsonString)
    ↓
[Processing Logic]
    ↓
Orchestration object
    ↓
DataTransferObject(orch)
    ↓
Base64Encode()
    ↓
QueueClient.SendMessage() → PUB_QueueName
```

### Stage 3: Publisher Import

```note
QueueClient.Receive() → Base64 message string
    ↓
Base64Decode()
    ↓
DataTransferObject(jsonString)
    ↓
[Publisher Import Logic]
    ↓
Final Status Update
```

---

## Data Type Details

### `ProcessStatusObject`

**Source:** `Publisher_Data_Operations.ProcessStatusObject`

**Purpose:** Contains granular processing status information

**Typical Properties:** *(Inferred from usage)*

- Processing stage indicators
- Validation results
- Transformation status
- Import status

### `FileDetailsObject`

**Source:** `Publisher_Data_Operations.FileDetailsObject`

**Purpose:** Stores file-specific metadata and details

**Typical Properties:** *(Inferred from usage)*

- File size
- Record counts
- Schema information
- Validation metrics

---

## Dependencies

### External Libraries

- `Newtonsoft.Json` - JSON serialization/deserialization
- `System.Collections.Specialized.NameValueCollection` - Form/query parameter handling

### Internal Dependencies

- `Publisher_Data_Operations.ProcessStatusObject` - Processing status container
- `Publisher_Data_Operations.FileDetailsObject` - File metadata container
- `Publisher_Data_Operations.Helper.Orchestration` - Processing orchestration class

---

## Design Patterns

### 1. Data Transfer Object (DTO) Pattern

**Intent:** Encapsulate data for cross-boundary communication

**Benefits:**

- Decouples services (Azure Functions)
- Provides serialization boundary
- Centralizes message structure

### 2. Builder Pattern (via Constructors)

**Intent:** Flexible object construction for different scenarios

**Benefits:**

- Multiple initialization strategies
- Clear intent per constructor
- Incremental property population

### 3. Null Object Pattern

**Intent:** Safe property access with null-conditional operators

**Benefits:**

- Prevents `NullReferenceException`
- Graceful handling of incomplete data
- Fail-safe deserialization

---

## Serialization Details

### JSON Serialization

**Library:** `Newtonsoft.Json` (Json.NET)

**Default Behavior:**

- Property names match C# convention (PascalCase)
- `null` properties included in JSON
- Circular references not handled (should be avoided in DTO design)

**Example JSON:**

```json
{
  "File_ID": "67890",
  "FileName": "Report_2024.xlsx",
  "FileRunID": "run-12345",
  "FileStatus": true,
  "Job_ID": "54321",
  "Batch_ID": "12345",
  "ProcessStatus": null,
  "FileDetails": null,
  "ErrorMessage": null,
  "RetryCount": 0,
  "NotificationEmailAddress": "admin@example.com"
}
```

### Base64 Encoding

**Purpose:** Safe queue message transport

**Process:**

```note
JSON → UTF-8 Bytes → Base64 String
```

**Example:**

```note
Input:  {"FileName":"test.xlsx","Batch_ID":"123","RetryCount":0}
Output: eyJGaWxlTmFtZSI6InRlc3QueGxzeCIsIkJhdGNoX0lEIjoiMTIzIiwiUmV0cnlDb3VudCI6MH0=
```

---

## Property Nullability Strategy

### Nullable Properties

| Property | Why Nullable | Default Value |
|----------|--------------|---------------|
| `File_ID` | May not exist at batch stage | `null` |
| `FileRunID` | Generated during processing | `null` |
| `FileStatus` | Unknown until processed | `null` |
| `Job_ID` | Created during Publisher import | `null` |
| `ProcessStatus` | Populated during orchestration | `null` |
| `FileDetails` | Populated during file inspection | `null` |
| `ErrorMessage` | Only present on failure | `null` |
| `NotificationEmailAddress` | Optional feature | `null` |

### Non-Nullable Properties

| Property | Why Not Nullable | Default Value |
|----------|------------------|---------------|
| `FileName` | Required for processing | `null` (but should always be set) |
| `Batch_ID` | Required for traceability | `null` (but should always be set) |
| `RetryCount` | Always has value | `0` |

---

## Usage Patterns

### Pattern 1: Initial Batch Message

```csharp
// In BatchBlob.cs
var dto = new DataTransferObject(fileName, batch.BatchID, batch.RetryCount);
queueClient.SendMessage(dto.Base64Encode());
```

### Pattern 2: Queue Message Consumption

```csharp
// In QueueProcess.cs
string base64Message = queueMessage.MessageText;
string json = Encoding.UTF8.GetString(Convert.FromBase64String(base64Message));
var dto = new DataTransferObject(json);
```

### Pattern 3: HTTP Endpoint with Overrides

```csharp
// In StatusPage.cs
var dto = new DataTransferObject(
    requestBody,                    // JSON from body
    HttpUtility.ParseQueryString(query) // Query parameters
);
```

### Pattern 4: Enriched Message After Processing

```csharp
// In QueueProcess.cs after orchestration
Orchestration orch = new Orchestration(fileStream, dto);
orch.ProcessFile();
var enrichedDto = new DataTransferObject(orch);
publisherQueue.SendMessage(enrichedDto.Base64Encode());
```

---

## Error Handling Considerations

### Deserialization Failures

**Scenario:** Malformed JSON in queue message

**Behavior:**

- `JsonConvert.DeserializeObject` throws `JsonException`
- Should be caught by calling function
- May result in poison message in queue

**Mitigation:**

```csharp
try {
    var dto = new DataTransferObject(jsonString);
} catch (JsonException ex) {
    log.LogError(ex, "Failed to deserialize DTO from queue message");
    // Move to poison queue or dead letter
}
```

### Missing Properties

**Scenario:** JSON missing expected properties

**Behavior:**

- Null-conditional operators (`?.`) return `null`
- Properties assigned `null` values
- Processing continues (may fail later if required)

**Validation:**

```csharp
var dto = new DataTransferObject(jsonString);
if (string.IsNullOrEmpty(dto.FileName)) {
    log.LogError("DTO missing required FileName property");
    return;
}
```

---

## Best Practices

### When Creating DTOs

1. **Use Appropriate Constructor**
   - Minimal constructor for initial stages
   - Orchestration constructor for enriched context
   - JSON constructor for queue consumption

2. **Always Validate Critical Properties**

   ```csharp
   if (string.IsNullOrEmpty(dto.FileName) || string.IsNullOrEmpty(dto.Batch_ID)) {
       throw new ArgumentException("DTO missing required properties");
   }
   ```

3. **Set RetryCount Appropriately**
   - Initial: Use batch configuration value
   - Retry: Increment from previous DTO

4. **Populate NotificationEmailAddress Early**
   - Set at batch creation if email notifications enabled
   - Carries through entire pipeline

### When Consuming DTOs

1. **Always Decode from Base64**

   ```csharp
   string json = Encoding.UTF8.GetString(Convert.FromBase64String(message));
   ```

2. **Handle Null Properties Gracefully**

   ```csharp
   int fileId = int.TryParse(dto.File_ID, out var id) ? id : -1;
   ```

3. **Log Reconstruction Failures**
   - Capture original message for debugging
   - Log exception details

---

## Testing Considerations

### Unit Test Scenarios

1. **Default Constructor**

   ```csharp
   [Test]
   public void DefaultConstructor_InitializesAllPropertiesToNull() {
       var dto = new DataTransferObject();
       Assert.IsNull(dto.FileName);
       Assert.AreEqual(0, dto.RetryCount);
   }
   ```

2. **JSON Roundtrip**

   ```csharp
   [Test]
   public void JsonSerialization_Roundtrip_PreservesData() {
       var original = new DataTransferObject("test.xlsx", 123, 1);
       string json = JsonConvert.SerializeObject(original);
       var deserialized = new DataTransferObject(json);
       Assert.AreEqual(original.FileName, deserialized.FileName);
   }
   ```

3. **Base64 Encoding**

   ```csharp
   [Test]
   public void Base64Encode_ReturnsValidBase64String() {
       var dto = new DataTransferObject("test.xlsx", 123, 0);
       string encoded = dto.Base64Encode();
       Assert.DoesNotThrow(() => Convert.FromBase64String(encoded));
   }
   ```

4. **Null Handling**

   ```csharp
   [Test]
   public void JsonConstructor_HandlesNullProperties_Gracefully() {
       string json = "{}"; // Empty JSON object
       var dto = new DataTransferObject(json);
       Assert.IsNull(dto.FileName);
       Assert.AreEqual(0, dto.RetryCount);
   }
   ```

---

## Performance Considerations

### Serialization Overhead

- JSON serialization is CPU-intensive for large objects
- `ProcessStatusObject` and `FileDetailsObject` size impacts performance
- Consider compression for very large DTOs (not currently implemented)

### Memory Allocation

- Each constructor creates new object instance
- String concatenation in Base64 encoding allocates memory
- Use object pooling for high-throughput scenarios (not currently implemented)

### Queue Message Size Limits

- Azure Queue messages limited to **64 KB**
- Base64 encoding increases size by ~33%
- Ensure DTO doesn't exceed ~48 KB JSON size
- Consider splitting large payloads or using blob references

---

## Security Considerations

### Sensitive Data

- DTO may transit through multiple queues
- Email addresses are PII (Personally Identifiable Information)
- Error messages may contain sensitive path information

**Recommendations:**

- Avoid including credentials or tokens
- Sanitize error messages before assignment
- Consider encryption for sensitive properties

### Injection Risks

- JSON deserialization from untrusted sources
- Use `JsonSerializerSettings` with type restrictions if needed
- Validate file names for path traversal attempts

---

## Maintenance Notes

### Commented Properties

The constructor from `Orchestration` contains commented-out property assignments:

```csharp
// DocumentType = orch.GetFile.DocumentType;
// Company_ID = orch.GetFile.CompanyID.ToString();
// Document_Type_ID = orch.GetFile.DocumentTypeID.ToString();
```

**Implications:**

- These properties may be added in future versions
- Currently not part of message contract
- Schema evolution should be backward-compatible

### String-Based IDs

All ID properties are `string` type (not `int` or `long`)

**Rationale:**

- Simplifies serialization (no type conversion errors)
- Supports future non-numeric ID schemes
- Nullable via `null` instead of sentinel values

**Trade-offs:**

- Requires parsing when used as numeric IDs
- Larger JSON payload size
- Potential for invalid ID formats

---

## Related Components

### Producers (Create DTOs)

- `BatchBlob.cs` - Creates initial batch messages
- `QueueProcess.cs` - Creates enriched messages after processing
- `StatusPage.cs` - Creates DTOs from HTTP requests

### Consumers (Read DTOs)

- `QueueProcess.cs` - Reads from `PDI_QueueName`
- `QueueImport.cs` - Reads from `PUB_QueueName`
- `StatusPage.cs` - Reads from HTTP request bodies

### Related Types

- `Orchestration` - Business logic processor
- `ProcessStatusObject` - Status container
- `FileDetailsObject` - File metadata container
- `PDIBatch` - Batch management
