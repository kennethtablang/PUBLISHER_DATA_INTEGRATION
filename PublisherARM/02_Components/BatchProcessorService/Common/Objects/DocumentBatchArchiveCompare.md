# DocumentBatchArchiveCompare — Archive Comparison Batch Processor (analysis)

## One-line purpose

Specialized batch processor that retrieves archived document versions from Azure Storage, generates comparison reports (redline/side-by-side) using CompareDocs library, and packages results as downloadable ZIP files for regulatory compliance and audit purposes.

---

## Files analyzed

* `BatchProcessorService/DocumentBatchArchiveCompare.cs`

---

## What this code contains (high level)

1. **Archive Version Retrieval** — Fetches two document versions from Azure Storage based on archive dates
2. **Document Comparison Engine** — Generates comparison reports using CompareDocs third-party library
3. **File System Management** — Creates temporary directories, downloads files, and performs cleanup
4. **ZIP Package Creation** — Bundles all comparison PDFs into single downloadable archive
5. **Validation Layer** — Verifies archive documents exist before comparison
6. **Progress Tracking** — Phased progress reporting (70% processing, 80% zipping, 90% cleanup, 100% complete)
7. **Comparison Mode Support** — Supports both consolidated (redline) and side-by-side comparison formats

This class handles complex multi-version document comparison workflows for compliance and audit reporting, integrating with Azure Storage and third-party comparison tools.

---

## Classes, properties and methods (developer reference)

### `DocumentBatchArchiveCompare : DocumentBatch` - Archive Comparison Processor

**Purpose:** Orchestrates retrieval, comparison, and packaging of archived document versions for regulatory compliance and change tracking.

#### Constructors

##### Default Constructor

* `public DocumentBatchArchiveCompare() : base()` — Creates new comparison batch instance

**Initialization:**

```csharp
this.batchType = BatchType.ArchiveCompare;
```

##### Database and File System Constructor

* `public DocumentBatchArchiveCompare(int batchID, string archiveFileTempPath) : base(batchID)` — Loads batch with file system setup

**Multi-Phase Initialization:**

## Phase 1: Base Loading

```csharp
base(batchID)  // Loads core batch properties
```

## Phase 2: Comparison-Specific Data

```csharp
using (DatabaseConnector databaseConnector = new DatabaseConnector(Constants.connectionString))
{
    SqlDataReader batchInfo = databaseConnector.Execute("sp_SELECT_BATCH_ARCHIVE_COMPARE", databaseParameters);
    
    if (batchInfo.Read())
    {
        this.compareStartDate = HandyFunctions.assumeDefault(batchInfo["START_DATE"], DateTime.MinValue);
        this.compareEndDate = HandyFunctions.assumeDefault(batchInfo["END_DATE"], DateTime.MinValue);
        this.isFinal = HandyFunctions.convertToBool(batchInfo["MARK_AS_FINAL"]);
        this.SideBySide = HandyFunctions.convertToBool(batchInfo["SIDE_BY_SIDE"]);
        this.CompletedBatchFileName = batchInfo["COMPLETED_BATCH_FILENAME"].ToString();
    }
}
```

## Phase 3: File System Setup

```csharp
ArchiveFileTempPath = archiveFileTempPath;  // Base temp directory
ArchiveFileTempZipPath = ArchiveFileTempPath + this.CompletedBatchFileName.Replace(".zip", string.Empty);
System.IO.Directory.CreateDirectory(ArchiveFileTempZipPath);  // Create working directory
```

**Directory Structure Created:**

```note
{ArchiveFileTempPath}/
└── {CompletedBatchFileName without .zip}/
    ├── downloaded archive PDFs
    └── generated comparison PDFs
```

---

## Property Accessors

### Comparison Configuration

* `public bool isFinal { get; set; }` — Use final archives for comparison (regulatory versions)
* `public DateTime compareStartDate { get; set; }` — Start date for comparison range
* `public DateTime compareEndDate { get; set; }` — End date for comparison range
* `public bool SideBySide { get; set; }` — Comparison format (true = side-by-side, false = consolidated redline)

### Processing Context

* `public List<ArchiveCompareDocs> archiveCompareDocs { get; set; }` — Collection of document comparison metadata
* `public string CompletedBatchFileName { get; set; }` — Output ZIP filename
* `private string ArchiveFileTempPath { get; set; }` — Base temporary directory path
* `private string ArchiveFileTempZipPath { get; set; }` — Working directory for current batch

---

## Core Processing Methods

### Archive Validation

* `public List<ArchiveCompareDocs> ValidateArchiveDocument(List<Document> selectedDocuments)` — Validates archive existence for document list

**Validation Process:**

```csharp
foreach (Document document in selectedDocuments)
{
    docs.Add(new ArchiveCompareDocs(
        document.documentName,
        RetrieveArchiveDocument(document, this.compareStartDate, isFinal),  // First version
        RetrieveArchiveDocument(document, this.compareEndDate, isFinal)      // Second version
    ));
}
```

**Returns:** List of `ArchiveCompareDocs` with document name and two archive IDs (GUIDs)

**Use Case:** Pre-flight validation before batch processing to identify missing archives.

### Archive Retrieval

* `public Guid RetrieveArchiveDocument(Document document, DateTime archiveDate, bool isFinal)` — Retrieves archive ID for specific document and date

**Query Logic:**

```csharp
DatabaseParameter[] databaseParameters = new DatabaseParameter[4];
databaseParameters[0] = new DatabaseParameter("@documentID", SqlDbType.Int, 4, document.documentID);
databaseParameters[1] = new DatabaseParameter("@cultureCode", SqlDbType.VarChar, 10, document.documentCultureCode);
databaseParameters[2] = new DatabaseParameter("@archiveDate", SqlDbType.DateTime, 8, archiveDate);
databaseParameters[3] = new DatabaseParameter("@isFinal", SqlDbType.Bit, isFinal);

SqlDataReader ArchiveDocumentsReader = databaseConnector.Execute("sp_SELECT_DOCUMENT_ARCHIVE_COMPARE", databaseParameters);
```

**Archive Selection Logic:**

* Finds archive closest to specified date
* Filters by final/non-final designation
* Returns GUID for Azure Storage blob reference

**Returns:** Archive GUID (Guid.Empty if not found)

**Side Effect:** Updates `document.documentName` from archive metadata (questionable practice - modifies input parameter)

---

## Batch Processing Methods

### Main Comparison Orchestration

* `public bool ArchiveCompareBatch()` — Primary batch processing method

**Processing Pipeline:**

## Phase 1: Initialization (0%)

```csharp
string originalThreadCultureCode = HandyFunctions.getCurrentCultureCode();  // Captured but unused
float percentIncrementPerDocument = 70f / this.documents.Count;  // 70% allocated to document processing
float percentComplete = 0f;
```

## Phase 2: Document Processing Loop (0-70%)

```csharp
foreach (Document document in this.documents)
{
    this.updateProgress(percentComplete, "Processing Document: " + document.documentID);
    
    // Retrieve archive IDs for comparison
    string compareFile1 = RetrieveArchiveDocument(document, this.compareStartDate, this.isFinal).ToString().ToUpper() + ".pdf";
    string compareFile2 = RetrieveArchiveDocument(document, this.compareEndDate, this.isFinal).ToString().ToUpper() + ".pdf";
    
    // Download both versions from Azure Storage
    if (!HandyFunctions.downloadAzureStorageFile(Constants.DocumentArchiveContainerName, compareFile1, ArchiveFileTempZipPath))
    {
        Log.error("DocumentBatchArchiveCompare.ArchiveCompareBatch()", "Missing archive file1: " + compareFile1);
        IsFileDownloadedForComparision = false;
    }
    
    if (!HandyFunctions.downloadAzureStorageFile(Constants.DocumentArchiveContainerName, compareFile2, ArchiveFileTempZipPath))
    {
        Log.error("DocumentBatchArchiveCompare.ArchiveCompareBatch()", "Missing archive file2: " + compareFile2);
        IsFileDownloadedForComparision = false;
    }
    
    // Generate comparison document
    CompareFiles compareDocs = new CompareFiles(
        ArchiveFileTempZipPath,
        compareFile1,
        compareFile2,
        this.SideBySide,
        ComparisonMode.pdf,
        Constants.COMPARE_DOCS_LIC_NAME,
        Constants.COMPARE_DOCS_LIC_KEY,
        HandyFunctions.RemoveDiacritics(document.compareDocumentFileName)
    );
    
    if (IsFileDownloadedForComparision)
    {
        if (compareDocs.Compare())
        {
            percentComplete += percentIncrementPerDocument;
        }
        else
        {
            Log.error("DocumentBatchArchiveCompare.ArchiveCompareBatch()", "CompareDocs failed for Document ID: " + document.documentID);
        }
    }
    
    compareDocs.DeleteTempInputFiles();  // Cleanup downloaded archives
}
```

## Phase 3: ZIP Creation (70-100%)

```csharp
return CreateZipFile();
```

**Error Handling:** Logs errors but continues processing remaining documents (no fail-fast behavior)

### ZIP Package Creation

* `private bool CreateZipFile()` — Bundles comparison results into downloadable ZIP

**Packaging Pipeline:**

## Step 1: Validation (70%)*

```csharp
this.updateProgress(70f, "Generating Zip File");

if (System.IO.Directory.GetFiles(ArchiveFileTempZipPath).Length == 0)
{
    throw new Exception("zip folder is empty");
}
```

## Step 2: ZIP Creation (70-80%)

```csharp
string absoluteFinalZipFile = ArchiveFileTempPath + this.CompletedBatchFileName;
ZipFile.CreateFromDirectory(ArchiveFileTempZipPath, absoluteFinalZipFile);
```

## Step 3: Azure Upload (80-90%)

```csharp
this.updateProgress(80f, "Uploading Zip File To Storage");

System.IO.MemoryStream finalZipFileAsStream = HandyFunctions.GetFileInMemoryStream(absoluteFinalZipFile);
HandyFunctions.saveToAzureStorage(
    Constants.BATCH_PROCESSING_PACKAGE_CONTAINER_NAME,
    finalZipFileAsStream,
    this.CompletedBatchFileName
);
```

## Step 4: Cleanup (90-100%)

```csharp
this.updateProgress(90f, "Deleting Temp Files");

HandyFunctions.DeleteFilesAndSubDirsRecursively(ArchiveFileTempZipPath);  // Remove working directory
System.IO.File.Delete(absoluteFinalZipFile);  // Remove local ZIP file

this.updateProgress(100f, "Complete");
```

**Returns:** `true` if successful, `false` if error occurred

---

## Persistence Methods

### Batch Saving

* `public override int saveBatch()` — Persists comparison batch with metadata

**Two-Phase Save:**

```csharp
int batchId = base.saveBatch();  // Save base batch data

// Save comparison-specific data
DatabaseParameter[] databaseParameters = new DatabaseParameter[6];
databaseParameters[0] = new DatabaseParameter("@batchID", SqlDbType.Int, 4, batchId);
databaseParameters[1] = new DatabaseParameter("@compareStartDate", SqlDbType.DateTime, 8, this.compareStartDate);
databaseParameters[2] = new DatabaseParameter("@compareEndDate", SqlDbType.DateTime, 8, this.compareEndDate);
databaseParameters[3] = new DatabaseParameter("@completedBatchFilename", SqlDbType.VarChar, 60, this.CompletedBatchFileName);
databaseParameters[4] = new DatabaseParameter("@isFinal", SqlDbType.Bit, 1, this.isFinal, false);
databaseParameters[5] = new DatabaseParameter("@sideBySide", SqlDbType.Bit, 1, this.SideBySide, false);

databaseConnector.Execute("sp_SAVE_BATCH_ARCHIVE_COMPARE", databaseParameters);
```

**Database Tables:**

* **BATCH:** Core batch metadata
* **BATCH_ARCHIVE_COMPARE:** Comparison-specific configuration

---

## Supporting Classes

### `ArchiveCompareDocs` - Comparison Metadata Container

**Purpose:** Encapsulates document comparison metadata for validation and processing.

#### Constructor

* `public ArchiveCompareDocs(string documentName, Guid firstArchiveCompareId, Guid secondArchiveCompareId)` — Creates comparison metadata instance

#### Properties

* `public string documentName { get; set; }` — Document display name
* `public Guid firstArchiveCompareId { get; set; }` — Archive ID for start date version
* `public Guid secondArchiveCompareId { get; set; }` — Archive ID for end date version

**Usage Pattern:**

```csharp
ArchiveCompareDocs metadata = new ArchiveCompareDocs(
    "Annual Report 2023",
    Guid.Parse("12345..."),  // January 2023 archive
    Guid.Parse("67890...")   // December 2023 archive
);
```

---

## Business logic and integration patterns

### Archive Comparison Workflow

Complex multi-stage workflow for regulatory document comparison:

## Stage 1: Batch Setup

* User selects documents and date range
* System validates archives exist for both dates
* Batch created with comparison parameters

## Stage 2: File Retrieval

* Archives downloaded from Azure Storage
* Stored in temporary working directory
* Both versions required for comparison

## Stage 3: Comparison Generation

* CompareDocs library generates comparison PDFs
* Supports consolidated (redline) or side-by-side formats
* Output saved to working directory

## Stage 4: Packaging

* All comparison PDFs bundled into ZIP
* ZIP uploaded to Azure Storage
* Local files cleaned up

## Stage 5: Delivery

* ZIP available for download via web interface
* Contains all comparison documents for batch

### CompareDocs Integration

Third-party document comparison library integration:

**Library:** CompareDocs (commercial PDF comparison tool)
**License:** Requires license name and key from Constants
**Comparison Modes:**

* **Consolidated (Redline):** Traditional legal blackline format with insertions/deletions marked
* **Side-by-Side:** Two versions displayed side-by-side with synchronized scrolling

**File Naming:**

```csharp
HandyFunctions.RemoveDiacritics(document.compareDocumentFileName)
```

Removes diacritical marks (accents) for file system compatibility.

### Azure Storage Architecture

Multi-container storage strategy:

**Archive Container:** `Constants.DocumentArchiveContainerName`

* Stores original archived document versions
* Files named as `{ARCHIVE_GUID}.pdf`

**Package Container:** `Constants.BATCH_PROCESSING_PACKAGE_CONTAINER_NAME`

* Stores downloadable ZIP packages
* Files named according to user-specified pattern

### File System Management

Comprehensive temporary file handling:

**Creation:** Working directories created per batch
**Processing:** Files downloaded, processed, and stored temporarily
**Cleanup:** Complete removal of temporary files and directories
**Error Handling:** Cleanup occurs even if comparison fails

---

## Technical implementation considerations

### Performance Characteristics

**Sequential Processing:** Documents processed one-by-one (no parallelization)
**Network I/O:** Multiple Azure Storage downloads per document (2 archives + 1 upload)
**Disk I/O:** Files written to disk for CompareDocs processing
**Memory Usage:** ZIP file loaded into memory stream before upload

**Bottlenecks:**

* Azure Storage download latency
* CompareDocs comparison processing time
* ZIP compression for large batches

### Error Handling Strategy

**Partial Failure Tolerance:**

* Logs errors but continues processing
* Accumulates failed documents
* Final ZIP may contain subset of requested comparisons

**Potential Issues:**

* No indication to user which documents failed
* Empty ZIP created if all comparisons fail
* Failed downloads still trigger comparison attempts

### File System Dependencies

**Temporary Directory Requirements:**

* Write access to configured temp path
* Sufficient disk space for:
  * Downloaded archive PDFs (2 per document)
  * Generated comparison PDFs (1 per document)
  * Final ZIP file
* Proper cleanup to prevent disk exhaustion

### Third-Party Library Integration

**CompareDocs Dependency:**

* Commercial license required
* License validation on every comparison
* Potential failure point if license expires
* Limited error information from library

---

## Integration with broader Publisher system

### Program.cs Integration

Called by batch processing service:

```csharp
// From Program.cs
private static bool processBatchArchiveCompare(DocumentBatch batch)
{
    try
    {
        DocumentBatchArchiveCompare batchArchiveCompare = new DocumentBatchArchiveCompare(
            batch.batchID,
            Constants.ARCHIVE_FILE_TEMP_PATH
        );
        batchArchiveCompare.indicateBatchProcessingStart();

        if (batchArchiveCompare.ArchiveCompareBatch())
        {
            batchArchiveCompare.indicateBatchProcessingEnd();
            return true;
        }

        return false;
    }
    catch (Exception exception)
    {
        Log.error("WorkerRole.processBatchArchiveCompare()", exception, "Batch ID: " + batch.batchID);
        return false;
    }
}
```

### HtmlDiff Integration

May use HtmlDiff class for comparison enhancement:

* HTML-based comparison for specific scenarios
* Alternative to CompareDocs for certain document types
* Complementary comparison capabilities

### Constants Configuration

Multiple configuration dependencies:

* `ARCHIVE_FILE_TEMP_PATH` — Temporary file processing location
* `DocumentArchiveContainerName` — Archive storage container
* `BATCH_PROCESSING_PACKAGE_CONTAINER_NAME` — Package output container
* `COMPARE_DOCS_LIC_NAME` — CompareDocs license name
* `COMPARE_DOCS_LIC_KEY` — CompareDocs license key

---

## Potential enhancements

### Performance Improvements

1. **Parallel Processing:** Process multiple documents concurrently
2. **Streaming Comparisons:** Avoid disk writes for comparison input
3. **Incremental ZIP Creation:** Build ZIP as documents complete
4. **Azure Storage Optimization:** Batch download operations

### Reliability Improvements

1. **Failed Document Tracking:** Report which documents failed in ZIP package
2. **Retry Logic:** Automatic retry for transient Azure Storage failures
3. **Checkpoint/Resume:** Support resuming failed batches
4. **Validation Reporting:** Generate manifest of successful/failed comparisons

### Feature Enhancements

1. **Comparison Summary:** Include summary report in ZIP
2. **Multiple Date Ranges:** Support comparing more than two versions
3. **Incremental Comparison:** Only compare changed documents
4. **Format Options:** Support additional comparison output formats

### Operational Improvements

1. **Disk Space Monitoring:** Check available space before processing
2. **License Validation:** Pre-validate CompareDocs license
3. **Async Operations:** Convert to async/await pattern
4. **Progress Granularity:** More detailed progress reporting

---

## Action items for system maintenance

1. **Code Cleanup:** Remove unused `originalThreadCultureCode` variable
2. **Error Handling Review:** Improve partial failure reporting
3. **License Management:** Implement CompareDocs license expiration monitoring
4. **Disk Space Monitoring:** Add disk space checks before batch processing
5. **Performance Profiling:** Identify and optimize bottlenecks
6. **Cleanup Verification:** Ensure temporary files always removed
7. **Azure Storage Optimization:** Review download/upload patterns for efficiency

---

## Critical considerations

### Side Effect in RetrieveArchiveDocument

The method modifies the input `Document` parameter:

```csharp
document.documentName = ArchiveDocumentsReader["DocumentName"].ToString();
```

**Issues:**

* Violates single responsibility principle
* Unexpected side effect not indicated by method signature
* Could cause subtle bugs if caller expects original document name
* Not thread-safe if document shared across threads

**Recommendation:** Return archive metadata separately or use output parameter.

### Partial Failure Handling

The batch continues processing after individual document failures:

**Problems:**

* Final ZIP may contain incomplete set
* No clear indication to user which documents failed
* User might assume all documents succeeded

**Recommendation:** Generate manifest file in ZIP listing successful/failed documents.

### Unused Culture Code Variable

Same issue as other batch types - captured but never used:

```csharp
string originalThreadCultureCode = HandyFunctions.getCurrentCultureCode();
```

**Recommendation:** Remove or document intended use.

### Disk Space Management

No verification of available disk space before processing:

**Risks:**

* Large batches could fill disk
* Partial processing leaves orphaned files
* ZIP creation could fail after all comparisons complete

**Recommendation:** Check available space before batch processing and implement disk space monitoring.
