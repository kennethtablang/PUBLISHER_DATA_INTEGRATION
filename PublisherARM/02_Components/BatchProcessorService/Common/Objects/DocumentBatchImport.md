# DocumentBatchImport — Excel Import Batch Processor (analysis)

## One-line purpose

Specialized batch processor that imports structured Excel files with validation, comprehensive error reporting, asynchronous database operations, staging table architecture, and automatic cleanup of malformed data for bulk document content updates.

---

## Files analyzed

* `BatchProcessorService/DocumentBatchImport.cs`

---

## What this code contains (high level)

1. **Multi-Sheet Excel Import** — Processes three worksheets (Text, Table, Chart fields) with different validation rules
2. **Comprehensive Validation** — Validates document codes, field names, culture codes, and column counts
3. **Error Reporting System** — Generates detailed Excel error reports with timestamp, location, and description
4. **Staging Table Architecture** — Two-phase import (staging tables → production tables)
5. **Asynchronous Processing** — Uses async patterns with ManualResetEvent for long-running operations
6. **IDisposable Implementation** — Proper resource management for SQL connections and Excel documents
7. **Selective Data Removal** — Optional removal of existing data before import

This class represents the data ingestion pipeline for bulk content updates, with sophisticated error handling and user feedback mechanisms.

---

## Classes, properties and methods (developer reference)

### `DocumentBatchImport : DocumentBatch, IDisposable` - Excel Import Processor

**Purpose:** Imports document field data from Excel files with validation, error reporting, and staged database updates.

#### Constants

##### Error Report Configuration

* `private const string EXCEL_SHEET_IMPORT_ERROR = "Error Report"` — Error report worksheet name
* `private const string EXCEL_SHEET_ERROR_COLUMN_DATE = "Date"` — Error timestamp column
* `private const string EXCEL_SHEET_ERROR_COLUMN_FILE = "File/Sheet"` — Error location column
* `private const string EXCEL_SHEET_ERROR_COLUMN_DESCRIPTION = "Error Description"` — Error detail column
* `private const string EXCEL_SHEET_ERROR_COLUMN_ROW_NO = "Row Number"` — Row identifier column
* `private const string EXCEL_FILE_NAME_IMPORT_ERROR = "Error Report"` — Error file base name
* `private const string BATCH_NAME_DATE_FORMAT = "d-MMM-yyyy h:mm tt"` — Error timestamp format

##### Error Message Constants

* `private const string IMPORT_ROW_ERROR = "Import file row does not include mandatory columns..."` — Column validation error
* `private const string IMPORT_ROW_CULTURE_CODE_ERROR = "Import file row has wrong culture code..."` — Invalid culture code
* `private const string IMPORT_ROW_DOCUMENT_CODE_ERROR = "Import file row missing the document code"` — Missing document code
* `private const string IMPORT_ROW_FIELD_NAME_ERROR = "Import file row missing the field name..."` — Missing field name
* `private const string IMPORT_ROW_ERROR_DB = "Error occured while trying to import the row in Database : "` — Database error prefix
* `private const string IMPORT_ROW = "Row"` — Row identifier prefix

#### Private Fields

##### Resource Management

* `private SqlConnection _connection = null` — Connection for async import operation
* `ExcelDocument excelDocImportError = null` — Error report Excel document
* `string pathToExcelFileError = string.Empty` — Error report file path

##### Threading Synchronization

* `private readonly ManualResetEvent _reset = new ManualResetEvent(false)` — Synchronization primitive for async operations
* `private bool _isProgresCompletedError = false` — Tracks async operation error state

#### Constructors

##### Default Constructor

* `public DocumentBatchImport() : base()` — Creates new import batch instance

##### Database Constructor

* `public DocumentBatchImport(int batchID) : base(batchID)` — Loads existing import batch

**Configuration Loading:**

```csharp
SqlDataReader batchInfo = databaseConnector.Execute("sp_SELECT_BATCH_IMPORT", databaseParameters);

if (batchInfo.Read())
{
    this.importFileName = batchInfo["IMPORT_FILENAME"].ToString();
    this.documentTypeID = Convert.ToInt32(batchInfo["DOCUMENT_TYPE_ID"].ToString());
    this.removeExisting = HandyFunctions.convertToBool(batchInfo["REMOVE_EXISTING"]);
}
```

#### Destructor and Disposal

##### Finalizer

* `~DocumentBatchImport()` — Ensures cleanup even if Dispose not called

**Pattern:**

```csharp
~DocumentBatchImport()
{
    Dispose(false);  // Finalizer calls without suppressing finalization
}
```

##### IDisposable Implementation

* `public void Dispose()` — Public disposal method
* `protected virtual void Dispose(bool disposing)` — Protected disposal implementation

**Disposal Logic:**

```csharp
protected virtual void Dispose(bool disposing)
{
    if (disposing)
    {
        if (_connection != null)
            _connection.Close();
        
        if (excelDocImportError != null)
            excelDocImportError.Dispose();
        
        if (_reset != null)
            _reset.Dispose();
    }
}
```

**Resources Managed:**

* SQL connection for async operations
* Excel error report document
* ManualResetEvent synchronization primitive

---

## Helper Methods

### Staging Table Cleanup

* `private void removeImportRecords()` — Clears staging tables before import

**Purpose:** Ensures clean state for new import, removes leftover data from failed imports.

### Error Record Deletion

* `private void deleteImportErrorRecords(string importErrorDocList)` — Removes data for documents with errors

**Cleanup Strategy:** If any row for a document fails validation, all data for that document removed from staging.

### Error Logging

* `private void logImportError(string sheetName, string errorType, int rowNumber)` — Records validation errors

**Error Report Creation:**

```csharp
if (excelDocImportError == null)
{
    // First error - create error report file
    string importErrorFileNameToCreate = EXCEL_FILE_NAME_IMPORT_ERROR + "_" + this.batchID + "_" + this.importFileName;
    this.importErrorFileName = importErrorFileNameToCreate;
    pathToExcelFileError = Path.Combine(Constants.EXCEL_TEMP_PATH, importErrorFileName);
    
    excelDocImportError = new ExcelDocument(pathToExcelFileError, EXCEL_SHEET_IMPORT_ERROR, false);
    
    // Set metadata
    excelDocImportError.company = App_GlobalResources.ExcelCopy.ImportErrorDocument_MetaData_Creator;
    // ... additional metadata
    
    // Add header row
    excelDocImportError.appendRow(EXCEL_SHEET_IMPORT_ERROR, 
        new List<string> { EXCEL_SHEET_ERROR_COLUMN_DATE, EXCEL_SHEET_ERROR_COLUMN_FILE, 
                          EXCEL_SHEET_ERROR_COLUMN_DESCRIPTION, EXCEL_SHEET_ERROR_COLUMN_ROW_NO });
    
    // Add first error
    excelDocImportError.appendRow(EXCEL_SHEET_IMPORT_ERROR, 
        new List<string> { HandyFunctions.convertToLocalTimeZone(DateTime.UtcNow).ToString(BATCH_NAME_DATE_FORMAT), 
                          this.importFileName + "/" + sheetName, errorType, IMPORT_ROW + " " + Convert.ToString(rowNumber) });
}
else
{
    // Subsequent errors - append to existing report
    excelDocImportError.appendRow(EXCEL_SHEET_IMPORT_ERROR, 
        new List<string> { /* error details */ });
}
```

### Culture Code Validation

* `private bool isValidCultureCode(string cultureCodeImport)` — Validates culture code against supported codes

**Validation:**

```csharp
return Constants.DOCUMENT_CULTURE_CODES.Contains(cultureCodeImport, StringComparer.InvariantCultureIgnoreCase);
```

**Supported Codes:** Defined in `Constants.DOCUMENT_CULTURE_CODES` (typically "en-CA", "fr-CA")

---

## Persistence Methods

### Import Batch Saving

* `public override int saveBatch()` — Persists import configuration

**Database Storage:**

```csharp
DatabaseParameter[] databaseParameters = new DatabaseParameter[4];
databaseParameters[0] = new DatabaseParameter("@batchID", SqlDbType.Int, 4, batchId);
databaseParameters[1] = new DatabaseParameter("@documentTypeID", SqlDbType.Int, 4, this.documentTypeID);
databaseParameters[2] = new DatabaseParameter("@removeExisting", SqlDbType.Bit, 1, Convert.ToInt32(this.removeExisting));
databaseParameters[3] = new DatabaseParameter("@importFilename", SqlDbType.VarChar, 200, this.importFileName);
```

### Completion Indication Override

* `public override void indicateBatchProcessingEnd()` — Records completion with metrics and error report

**Metrics Recording:**

```csharp
DatabaseParameter[] databaseParameters = new DatabaseParameter[5];
databaseParameters[0] = new DatabaseParameter("@batchID", SqlDbType.Int, 4, this.batchID);
databaseParameters[1] = new DatabaseParameter("@importFileName", SqlDbType.VarChar, 200, this.importFileName);
databaseParameters[2] = new DatabaseParameter("@errorReportFileName", SqlDbType.VarChar, 200, this.importErrorFileName, string.IsNullOrEmpty(this.importErrorFileName));
databaseParameters[3] = new DatabaseParameter("@documentCount", SqlDbType.Int, 4, this.importedDocuments.Distinct().Count());
databaseParameters[4] = new DatabaseParameter("@fieldCount", SqlDbType.Int, 4, this.importedFieldCount);
```

**Records:**

* Import filename
* Error report filename (if errors occurred)
* Distinct document count
* Total field count

---

## Property Accessors

### Import Configuration

* `public int documentTypeID { get; set; }` — Document type filter
* `public string importFileName { get; set; }` — Excel file to import
* `public bool removeExisting { get; set; }` — Remove existing content before import

### Processing Metrics (Private)

* `private List<string> importedDocuments { get; set; }` — Unique document codes imported
* `private int importedFieldCount { get; set; }` — Total field count
* `private string importErrorFileName { get; set; }` — Error report filename

---

## Business logic and integration patterns

### Staging Table Architecture

Two-phase import strategy minimizes production data corruption:

## Phase 1: Staging

* Data imported into temporary staging tables
* Validation occurs during staging
* Errors identified and logged
* Failed documents removed from staging

## Phase 2: Production

* Async stored procedure moves data to production tables
* Optional removal of existing data
* Atomic operation via database transaction

**Benefits:**

* Production data protected during import
* Failed imports don't corrupt production
* Easy rollback if issues detected

### Comprehensive Error Reporting

User-friendly error reporting system:

**Error Report Contents:**

* Timestamp of error
* File and sheet location
* Error description (validation type)
* Row number

**Error Report Delivery:**

* Uploaded to same Azure container as source file
* Referenced in batch completion record
* Downloadable via web interface

**User Experience:** Users can fix errors in Excel, re-upload, and retry import.

### Validation Strategy

Multi-level validation cascade:

**Level 1: Structural** - Minimum column count
**Level 2: Mandatory Fields** - Document code, field name
**Level 3: Business Rules** - Valid culture code
**Level 4: Database** - Successful insertion

**Fail-Fast Per Document:** If any row for a document fails, all rows for that document rejected.

### Remove Existing Functionality

Configurable data replacement strategy:

**removeExisting = true:**

* Existing content deleted before import
* Use case: Complete data refresh

**removeExisting = false:**

* Existing content preserved
* New data merged with existing
* Use case: Incremental updates

---

## Technical implementation considerations

### Asynchronous Processing Pattern

Legacy async pattern (pre-async/await):

**Pattern:** `BeginExecuteNonQuery` / `EndExecuteNonQuery` with callback
**Synchronization:** `ManualResetEvent` for thread coordination
**Modern Alternative:** async/await would simplify this significantly

**Current Implementation:**

```csharp
command.BeginExecuteNonQuery(callback, command);
_reset.WaitOne();  // Blocks until callback completes
```

**Potential Issue:** Blocking wait defeats purpose of async operation.

### Resource Managements

Proper IDisposable implementation:

**Resources:**

* SQL connection (for async operation)
* Excel error document
* ManualResetEvent

**Pattern:** Finalizer + Dispose(bool) for deterministic cleanup

### Memory Efficiency

Row-by-row processing prevents memory issues:

* Doesn't load entire Excel file into memory
* Processes one row at a time
* Immediately writes to database

### Error Handling Strategy

**Continue On Error:** Individual row failures don't halt import
**Document-Level Cleanup:** Failed documents completely removed
**User Notification:** Detailed error report for troubleshooting

---

## Integration with broader Publisher system

### ExcelDataDocument Integration

Reuses export format for symmetry:

* Same three-worksheet structure
* Same column definitions
* Validation against export template

### Azure Storage Integration

**DATA_IMPORT_CONTAINER_NAME:** Dedicated container

* Source Excel files uploaded by users
* Error reports written back
* Provides audit trail of imports

### Stored Procedure Architecture

**Staging Procedures:**

* `sp_SAVE_IMPORT_TEXT`
* `sp_SAVE_IMPORT_TABLE`
* `sp_SAVE_IMPORT_CHART`

**Production Procedure:**

* `sp_IMPORT_CONTENT` - Async operation moving staging to production

---

## Potential enhancements

### Modern Async Pattern

1. **Async/Await:** Replace legacy async pattern with modern async/await
2. **Streaming Processing:** True streaming without blocking wait
3. **Cancellation Support:** Support for cancelling long-running imports

### Performance Improvements

1. **Bulk Insert:** Use SqlBulkCopy instead of row-by-row
2. **Parallel Processing:** Process sheets concurrently
3. **Batch Database Operations:** Reduce database round trips

### Validation Enhancements

1. **Pre-Import Validation:** Validate entire file before database operations
2. **Business Rule Validation:** Validate against document templates
3. **Data Type Validation:** Validate data types beyond presence

### Error Reporting Improvements

1. **Detailed Error Context:** Include cell values in error report
2. **Suggested Fixes:** Provide correction suggestions
3. **Partial Success:** Option to import successful rows despite errors

---

## Critical considerations

### Async Pattern Misuse

The async implementation blocks the calling thread:

```csharp
command.BeginExecuteNonQuery(callback, command);
_reset.WaitOne();  // Blocks until complete
```

**Problem:** Defeats purpose of async operation - thread still blocked
**Impact:** No performance benefit, added complexity
**Modern Solution:** Use `await command.ExecuteNonQueryAsync()`

**Recommendation:** Refactor to true async/await pattern or remove async entirely.

### Error Document Disposal

Error document potentially not disposed on exceptions:

```csharp
if (excelDocImportError != null)
    excelDocImportError.Dispose();
```

Called in multiple places but not guaranteed if exception occurs.

**Recommendation:** Use using statement for automatic disposal.

### Row-by-Row Database Operations

Each row requires separate database call:

**Performance Impact:** Hundreds/thousands of database round trips
**Network Latency:** Significant overhead for each call

**Recommendation:** Batch operations or use SqlBulkCopy for better performance.
