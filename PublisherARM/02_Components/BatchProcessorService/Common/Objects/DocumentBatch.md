# DocumentBatch — Batch Processing Entity (analysis)

## One-line purpose

Core domain entity representing a batch job for bulk document operations with comprehensive lifecycle management, dependency orchestration, progress tracking, and support for multiple batch types (Package, Archive, Book, Import, Export, Report, ArchiveCompare).

---

## Files analyzed

* `BatchProcessorService/DocumentBatch.cs`

---

## What this code contains (high level)

1. **Batch Type System** — Enumeration supporting seven distinct batch operation types
2. **Batch Lifecycle Management** — Start/stop tracking, progress reporting, and completion status
3. **Document Collection Management** — Lazy-loaded document list for batch processing
4. **Dependency Orchestration** — Support for batch dependencies and sequential execution
5. **Progress Reporting** — Real-time progress updates with percentage completion and messaging
6. **Database Integration** — Comprehensive stored procedure integration for batch persistence
7. **Bulk Operations** — SqlBulkCopy for efficient batch-document relationship storage
8. **Readiness Validation** — Complex business rules determining batch processing eligibility

This class represents the central orchestration entity for all batch processing operations in the Publisher system, directly integrated with the Program.cs batch processing service.

---

## Classes, properties and methods (developer reference)

### `DocumentBatch` - Batch Processing Domain Entity

**Purpose:** Encapsulates batch job metadata, lifecycle operations, and business logic for determining batch processing readiness and orchestrating document collections.

#### Constants

##### Progress Allocation

* `protected const int PERCENT_OF_TIME_DEVOTED_TO_BATCH_AGGREGATION = 10` — Progress percentage reserved for batch initialization phase

**Progress Tracking Strategy:** Allocates 10% of progress bar to batch setup, remaining 90% for actual processing.

#### Enumerations

##### Batch Type Definition

* `public enum BatchType { Package, Archive, Book, Import, Export, Unknown, Report, ArchiveCompare }` — Defines all supported batch operation types

**Batch Type Descriptions:**

* **Package:** Document packaging operations (PDF bundling)
* **Archive:** Document archiving to Azure Storage
* **Book:** Document book/publication creation
* **Import:** Content import from external sources (Excel, templates)
* **Export:** Content export to external formats (Excel, data dumps)
* **Report:** Report generation operations
* **ArchiveCompare:** Historical document comparison (blackline generation)
* **Unknown:** Invalid/unrecognized batch type

**Integration with Program.cs:** These types directly correspond to the batch processing dispatching logic in Program.cs `processBatchMessage()` method.

#### Private Fields

##### Document Collection

* `private List<Document> _documents = null` — Lazy-loaded collection of documents in batch

**Lazy Loading Pattern:** Documents loaded only when accessed to optimize memory usage.

#### Constructors

##### Default Constructor

* `public DocumentBatch()` — Creates uninitialized batch instance

**Initialization State:**

```csharp
this.exists = false;
this.batchID = -1;
```

##### Database Constructor

* `public DocumentBatch(int batchID)` — Loads batch from database by ID

**Loading Process:**

1. **Database Query:** Calls `sp_SELECT_BATCH` with batch ID parameter
2. **Property Mapping:** Maps all SqlDataReader columns to object properties
3. **Type Determination:** Calls `getBatchType()` with boolean flags to determine batch type
4. **Existence Flag:** Sets `exists = true` if batch found in database

**Property Initialization Pattern:**

```csharp
this.batchType = getBatchType(
    HandyFunctions.convertToBool(batchInfo["IS_PACKAGE_BATCH"]),
    HandyFunctions.convertToBool(batchInfo["IS_ARCHIVE_BATCH"]),
    HandyFunctions.convertToBool(batchInfo["IS_BOOK_BATCH"]),
    HandyFunctions.convertToBool(batchInfo["IS_EXPORT_BATCH"]),
    HandyFunctions.convertToBool(batchInfo["IS_IMPORT_BATCH"]),
    HandyFunctions.convertToBool(batchInfo["IS_AUDIT_REPORT"]),
    HandyFunctions.convertToBool(batchInfo["IS_ARCHIVE_COMPARE_BATCH"])
);
```

**Null Handling:** Uses `HandyFunctions.assumeDefault()` for nullable fields with sensible defaults.

---

## Lifecycle Management Methods

### Start/Stop Tracking

#### Batch Start Indication

* `public void indicateBatchProcessingStart()` — Marks batch as started in database

**Database Integration:** Calls `sp_SAVE_BATCH_PROCESSING_START` to record start timestamp

**Use Case:** Called by specialized batch classes (DocumentBatchPackage, DocumentBatchExport, etc.) before processing begins.

#### Batch Completion Indication

* `public virtual void indicateBatchProcessingEnd()` — Marks batch as completed

**Virtual Method:** Allows derived classes to override with custom completion logic

**Parameters:**

```csharp
databaseParameters[0] = new DatabaseParameter("@batchID", SqlDbType.Int, 4, this.batchID);
databaseParameters[1] = new DatabaseParameter("@completedBatchFilename", SqlDbType.VarChar, 200, null, true);
```

**Filename Parameter:** Accepts optional completed batch filename (null in base implementation)

---

## Persistence Methods

### Batch Saving

* `public virtual int saveBatch()` — Persists batch and associated documents to database

**Two-Phase Save Process:**

**Phase 1 - Batch Metadata:**

```csharp
DatabaseParameter[] databaseParameters = new DatabaseParameter[14];
// Batch identity and scheduling
databaseParameters[0] = new DatabaseParameter("@batchID", SqlDbType.Int, 4, this.batchID, this.batchID == -1);
databaseParameters[3] = new DatabaseParameter("@scheduledDate", SqlDbType.DateTime, 8, 
    DateTime.UtcNow.AddHours(-1 * Constants.BATCH_START_OFFSET_HOURS));

// Batch type flags (one will be true)
databaseParameters[6] = new DatabaseParameter("@isArchiveBatch", SqlDbType.Bit, 1, this.batchType == BatchType.Archive, false);
databaseParameters[7] = new DatabaseParameter("@isPackageBatch", SqlDbType.Bit, 1, this.batchType == BatchType.Package, false);
// ... additional batch type flags
```

**Scheduling Logic:** Subtracts `BATCH_START_OFFSET_HOURS` to handle clock drift between application and database servers.

**Phase 2 - Batch-Document Relationships:**
Uses SqlBulkCopy for efficient batch insert:

```csharp
DataTable dtBatchDocuments = new DataTable();
dtBatchDocuments.Columns.Add("BATCH_ID", typeof(int));
dtBatchDocuments.Columns.Add("DOCUMENT_ID", typeof(int));
dtBatchDocuments.Columns.Add("LANGUAGE_ID", typeof(int));
dtBatchDocuments.Columns.Add("SORT_ORDER", typeof(int));

// Populate DataTable from documents collection
foreach (Document document in this.documents)
{
    dtBatchDocuments.Rows.Add(this.batchID, document.documentID, 
        HandyFunctions.getLanguageID(document.documentCultureCode), document.sortOrder);
}

// Bulk insert to BATCH_DOCUMENT table
using (SqlBulkCopy bulkCopy = new SqlBulkCopy(destinationConnection))
{
    bulkCopy.DestinationTableName = "dbo.BATCH_DOCUMENT";
    bulkCopy.WriteToServer(dtBatchDocuments);
}
```

**Performance Optimization:** SqlBulkCopy provides efficient insertion for batches with many documents.

**Returns:** Batch ID (newly generated or existing)

---

## Progress Reporting Methods

### Progress Updates

#### Standard Progress Update

* `protected void updateProgress(float percentComplete, string message)` — Updates batch progress in database

**Validation Logic:**

```csharp
percentComplete = Math.Max(0, percentComplete);   // Floor at 0%
percentComplete = Math.Min(100, percentComplete); // Ceiling at 100%
```

**Database Update:** Calls `sp_UPDATE_BATCH_PROGRESS` with percentage and status message

#### Telemetry-Enhanced Progress Update

* `protected void updateProgress(float percentComplete, string message, TelemetryClient _client)` — Progress update with Application Insights telemetry

**Telemetry Events:**

* `DocumentBatch.updateProgress - top` — Method entry
* `DocumentBatch.updateProgress - inside using` — Database connection established
* `Inside DocumentBatch.updateProgress - calling Execute` — Before stored procedure call
* `Inside DocumentBatch.updateProgress - called Execute` — After stored procedure call
* `DocumentBatch.updateProgress - after using` — After database connection disposed
* `DocumentBatch.updateProgress - end` — Method exit

**Debugging Purpose:** Extensive telemetry helps diagnose batch processing issues and performance bottlenecks.

---

## Helper Methods

### Batch Type Resolution

* `private BatchType getBatchType(bool isPackageBatch, bool isArchiveBatch, bool isBookBatch, bool isExportBatch, bool isImportBatch, bool isAuditReport, bool isArchiveCompareBatch)` — Determines batch type from boolean flags

**Priority Order:**

1. Package → Archive → Book → Export → Import → Report → ArchiveCompare
2. Returns Unknown if no flags are true

**Database Schema Pattern:** Database uses separate boolean columns for each batch type rather than a single type column.

### Document Collection Loading

* `private List<Document> getBatchDocuments()` — Loads documents associated with batch

**Query Process:**

1. **Stored Procedure:** Calls `sp_SELECT_BATCH_DOCUMENT` with batch ID
2. **Document Instantiation:** Creates Document objects with culture codes
3. **Return:** List of Document instances

**Cultural Pattern:** Uses same culture code for both site and document culture.

---

## Property Accessors

### Core Properties

#### Identity and Metadata

* `public bool exists { get; set; }` — Indicates if batch exists in database
* `public int batchID { get; set; }` — Unique batch identifier
* `public string batchName { get; set; }` — User-friendly batch name
* `public string cultureCode { get; set; }` — Cultural context for batch
* `public int companyID { get; set; }` — Company/organization identifier

#### Scheduling and Dependencies

* `public int mustRunAfterBatchID { get; set; }` — Dependency on prior batch completion
* `public DateTime scheduledDate { get; set; }` — Earliest execution time
* `public bool dependantBatchComplete { get; set; }` — Flag indicating dependency satisfied

**Dependency Pattern:** Enables sequential batch execution where batch B must wait for batch A to complete.

#### Processing State

* `public bool processingComplete { get; set; }` — Batch completion flag
* `public DateTime processingStartDate { get; set; }` — Processing start timestamp
* `public bool isActive { get; set; }` — Batch enabled for processing
* `public int documentCount { get; set; }` — Number of documents in batch

#### User Context

* `public Guid batchCreatedByUserKey { get; set; }` — User who created the batch

#### Batch Configuration

* `public BatchType batchType { get; set; }` — Type of batch operation
* `public bool? isBatchAODA { get; set; }` — Accessibility compliance flag (Accessibility for Ontarians with Disabilities Act)

**AODA Support:** Indicates batch generates accessible documents meeting regulatory requirements.

#### Dynamic API Parameters

* `public string ctiParameter { get; set; }` — Crawford API payload parameter (Custom Type Information)
* `public string icfParameter { get; set; }` — Crawford API payload parameter (Integration Configuration)

**API Integration:** Supports dynamic parameters for external API integrations, likely added for specific client requirements.

### Computed Properties

#### Document Collection for creating Batches

* `public List<Document> documents { get; set; }` — Lazy-loaded document list

**Lazy Loading Implementation:**

```csharp
get
{
    if (_documents == null)
    {
        _documents = getBatchDocuments();
    }
    return _documents;
}
```

**Performance Benefit:** Avoids loading document collection unless explicitly needed.

#### Processing Readiness

* `public bool readyForProcessing { get; }` — Computed property determining if batch can be processed

**Readiness Criteria (All must be true):**

1. **Exists:** Batch record exists in database
2. **Valid Type:** Batch type is not Unknown
3. **Active:** Batch is marked as active
4. **Has Documents:** Document count > 0 OR batch is Export/Import/Report type
5. **Scheduled:** Scheduled date is not in future
6. **Dependencies Met:** No dependent batches or dependent batch is complete

**Complex Business Rule:**

```csharp
return
    this.exists &&
    this.batchType != BatchType.Unknown &&
    this.isActive &&
    (
        this.documentCount > 0 || 
        this.batchType == BatchType.Export ||
        this.batchType == BatchType.Import ||
        this.batchType == BatchType.Report
    ) &&
    this.scheduledDate <= DateTime.UtcNow &&
    this.dependantBatchComplete;
```

**Special Cases:** Export, Import, and Report batches don't require documents (they generate or consume data from external sources).

---

## Business logic and integration patterns

### Batch Type Architecture

The system supports seven distinct batch types with different behaviors:

**Document-Based Batches:**

* **Package:** Bundles documents into packages/portfolios
* **Archive:** Archives documents to long-term storage
* **Book:** Creates multi-document publications

**Data Exchange Batches:**

* **Import:** Imports content from Excel or other sources
* **Export:** Exports document data to Excel or other formats
* **Report:** Generates system reports

**Analysis Batches:**

* **ArchiveCompare:** Creates comparison documents against archived versions

### Dependency Orchestration

Sophisticated batch dependency system enables complex workflows:

**Use Cases:**

* Sequential processing where batch B needs batch A's output
* Coordinated multi-stage operations
* Dependency chains for complex document workflows

**Implementation:**

* `mustRunAfterBatchID` — References prerequisite batch
* `dependantBatchComplete` — Cached flag for performance
* Database-level tracking ensures consistency

### Progress Tracking System

Real-time progress reporting enables user monitoring:

**Progress Components:**

* **Percentage:** 0-100% completion with validation
* **Message:** Human-readable status description
* **Database Updates:** Stored for UI display and API access

**Allocation Strategy:** 10% reserved for batch aggregation, 90% for processing.

### Scheduling System

Time-based batch execution with clock drift compensation:

**Scheduling Features:**

* `scheduledDate` — Earliest execution time
* `BATCH_START_OFFSET_HOURS` — Clock drift compensation
* UTC timestamps for consistency

**Clock Drift Handling:**

```csharp
DateTime.UtcNow.AddHours(-1 * Constants.BATCH_START_OFFSET_HOURS)
```

Subtracts offset to ensure batches aren't delayed by minor clock differences.

---

## Technical implementation considerations

### Performance Optimization

**SqlBulkCopy Usage:** Efficient batch-document relationship insertion
**Lazy Loading:** Documents loaded only when needed
**Progress Caching:** Percentage validation prevents excessive database calls

### Thread Safety Considerations

**Instance Scope:** Each batch processing thread gets its own DocumentBatch instance
**No Static State:** All state is instance-level, avoiding concurrency issues
**Database Locking:** Relies on database-level locking for consistency

### Error Handling Strategy

**Try-Catch Blocks:** All database operations wrapped in exception handling
**Logging Integration:** Errors logged with batch ID for troubleshooting
**Graceful Degradation:** Progress update failures don't halt batch processing

### Database Schema Design

**Boolean Type Flags:** Separate columns for each batch type
**Nullable Fields:** Optional parameters (mustRunAfterBatchID, sortOrder) properly handled
**Bulk Operations:** Optimized for large document collections

---

## Integration with broader Publisher system

### Program.cs Integration

Direct integration with batch processing service:

**Processing Dispatch:** `BatchType` enum used to route batches to specialized processors:

```csharp
// From Program.cs
if (batch.batchType == DocumentBatch.BatchType.Package)
    processingComplete = processBatchPackage(batch);
else if (batch.batchType == DocumentBatch.BatchType.Export)
    processingComplete = processBatchExport(batch, _client);
// ... etc
```

**Readiness Check:** `readyForProcessing` property determines if batch should be processed.

### Specialized Batch Classes

Base class for specialized implementations:

* `DocumentBatchPackage`
* `DocumentBatchArchive`
* `DocumentBatchBook`
* `DocumentBatchExport`
* `DocumentBatchImport`
* `DocumentBatchReport`
* `DocumentBatchArchiveCompare`

**Inheritance Pattern:** Derived classes override `indicateBatchProcessingEnd()` and `saveBatch()` for custom behavior.

### Azure Queue Integration

Batches triggered via queue messages:

* Queue message contains batch ID
* `DocumentBatch` constructor loads batch details
* `readyForProcessing` validates batch before execution

### Application Insights Integration

Comprehensive telemetry for monitoring:

* Progress tracking events
* Performance measurement
* Error diagnostics

---

## Potential enhancements

### Architecture Improvements

1. **Repository Pattern:** Extract data access to separate repository classes
2. **Factory Pattern:** Create factory for instantiating specialized batch types
3. **Strategy Pattern:** Encapsulate batch type-specific logic in strategy classes
4. **Event Sourcing:** Track all batch state changes as events

### Feature Enhancements

1. **Retry Logic:** Built-in retry mechanisms for transient failures
2. **Cancellation Support:** Allow batch cancellation mid-processing
3. **Partial Completion:** Support resuming failed batches
4. **Priority Queuing:** Priority-based batch scheduling

### Performance Optimizations

1. **Async Operations:** Convert to async/await pattern
2. **Parallel Processing:** Process independent documents concurrently
3. **Caching Strategy:** Cache frequently accessed batch metadata
4. **Streaming Updates:** Real-time progress streaming via SignalR

---

## Action items for system maintenance

1. **Batch Type Analysis:** Analyze which batch types are most frequently used
2. **Performance Monitoring:** Track batch processing times by type
3. **Dependency Analysis:** Review batch dependency patterns for optimization opportunities
4. **Error Rate Tracking:** Monitor batch failure rates and common error patterns
5. **Progress Accuracy:** Validate progress reporting accuracy against actual processing time
6. **Database Schema Review:** Consider consolidating batch type flags into single enum column

---

## Critical considerations

### Batch Type Determination Logic

The `getBatchType()` method has priority ordering that could hide configuration errors:

* If multiple type flags are true, only the first match is returned
* No validation that exactly one flag is true
* Could lead to silent misconfiguration

**Recommendation:** Add validation to ensure mutually exclusive batch types.

### Clock Drift Compensation

The scheduled date offset assumes constant clock drift direction and magnitude:

```csharp
DateTime.UtcNow.AddHours(-1 * Constants.BATCH_START_OFFSET_HOURS)
```

This could cause issues if clocks drift in opposite directions or by varying amounts.

**Recommendation:** Use NTP synchronization instead of manual offset compensation.

### Readiness Logic Complexity

The `readyForProcessing` property has complex logic that may be difficult to troubleshoot:

* Multiple conditions with special cases
* No logging of which condition failed
* Could silently skip batches that should run

**Recommendation:** Add detailed logging of readiness check failures for diagnostics.
