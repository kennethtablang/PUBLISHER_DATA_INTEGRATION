# PDIFile Helper — Component Analysis

## One-line purpose

Comprehensive file identity parser and database integration manager that decodes structured filenames into business context, orchestrates multi-stage database record creation, and provides complete file lifecycle tracking for the Publisher Data Integration pipeline.

---

## Files analyzed

* `Publisher_Data_Operations/Helper/PDIFile.cs`

---

## What this code contains (high level)

1. **Structured Filename Parser** — Decodes standardized filename format into business entities (custodian, client, LOB, document type, etc.)
2. **Database Integration Orchestrator** — Manages complex multi-table database operations across receipt, file, and processing logs
3. **Validation and Error Management** — Comprehensive validation with detailed error tracking and logging integration
4. **Processing Lifecycle Coordinator** — Orchestrates file processing from receipt through completion with retry support
5. **Reporting and Parameter Services** — Provides parameter extraction and CSV generation for notifications and monitoring

This component serves as the foundational file processing coordinator, transforming filenames into rich business context and managing the complete processing lifecycle.

---

## Classes, properties and methods (developer reference)

### `FileDetailsObject` - Data Transfer Object

**Purpose:** Serializable container for file metadata, providing clean data transfer between processing stages and external systems.

#### Properties

* `string DataCustodian, DataCustodianID` — Data custodian identification
* `string CompanyName, CompanyID` — Company context
* `string DocumentType, DocumentTypeID` — Document classification
* `string DataType, DataTypeID` — Data type classification
* `DateTime? CreationDateTime` — File creation timestamp
* `string Version, Code, Note` — File versioning and annotation
* `bool FileNameIsValid, FileIsValid` — Validation status flags

#### Methods

* `FileDetailsObject()` — Default constructor
* `FileDetailsObject(PDIFile pdiFile)` — PDIFile conversion constructor
* `void SetValues(PDIFile pdiFile)` — Populates properties from PDIFile instance

---

### `PDIFile` - Main File Processing Class

**Purpose:** Central file processing orchestrator that manages filename parsing, database operations, validation, and processing lifecycle coordination.

#### Core Constants and Configuration

* `const char FILE_DELIMITER = '_'` — Filename section delimiter
* `private Dictionary<string, int?> IdValues` — Database ID cache for performance
* `private Dictionary<string, string> _parameters` — Processing parameters cache

#### State Management Properties

* `int? FileID { get; private set; }` — Primary file identifier with logger integration
* `int? DataID { get; private set; }` — Processing data identifier
* `int? JobID, BatchID` — Processing context identifiers
* `string FileRunID` — Unique execution identifier (GUID)

#### Database Integration

* `DBConnection dbConn` — Database connection for all operations
* `Logger _log` — Error logging and validation tracking
* `Processing _proc` — Processing stage management

#### Parsed Filename Components

* `string DataCustodian` — Data custodian name from filename
* `int? DataCustodianID` — Resolved custodian database ID
* `string ClientName` — Client name from filename
* `int? ClientID, CompanyID` — Resolved client database IDs
* `string LOB` — Line of business from filename
* `int? LOBID` — Resolved LOB database ID
* `string DataType, DocumentType` — Type classifications from filename
* `int? DataTypeID, DocumentTypeID` — Resolved type database IDs
* `string Code` — Optional document code (for FSMRFP documents)
* `string CreationDate, CreationTime, Version, Note` — Timestamp and versioning components

#### Computed Properties

* `DateTime? CreationDateTime` — Parsed creation date/time with validation
* `DataTypeID? GetDataType` — Enum conversion for data type
* `DocumentTypeID? GetDocumentType` — Enum conversion for document type
* `string Extension` — File extension extraction
* `bool IsValid` — Overall validation status
* `bool IsValidFileName` — Filename structure and content validation
* `FileDetailsObject GetFileDetails` — DTO conversion property

---

## Constructor Patterns and Initialization

### Primary Constructors

* `PDIFile(string fileName, object conn = null, bool loadOnly = false, int fileID = -1, Logger log = null)` — Full filename parsing constructor
  * **Filename Processing:** Parses filename in setter, triggers validation
  * **Database Integration:** Optional database connection for ID resolution
  * **Load-Only Mode:** Skip database operations for analysis-only scenarios
  * **Logger Integration:** Creates or uses provided logger

* `PDIFile(int fileID, object conn, string processPath = null, bool loadOnly = false, Logger log = null)` — Database-driven constructor
  * **File Lookup:** Retrieves filename from database using FileID
  * **Path Handling:** Optional process path combination
  * **Database Requirement:** Requires database connection for file lookup

* `PDIFile(string fileName)` — Lightweight parsing-only constructor

---

## Filename Parsing and Validation

### Filename Structure Analysis

The filename parser handles structured formats with 8-10 components:

```note

DataCustodian_ClientName_LOB_[Code_]DataType_DocumentType_CreationDate_CreationTime_Version[_Note]
```

**Parsing Logic in `OnlyFileName` setter:**

1. **Section Validation:** Ensures 8-10 underscore-delimited sections
2. **Component Extraction:** Sequential assignment to properties
3. **Format Detection:** Handles optional Code and Note components
4. **Date/Time Validation:** Validates creation timestamp format (yyyyMMdd HHmmss)
5. **Version Validation:** Ensures version is numeric
6. **Extension Validation:** Restricts to .xlsx and .zip files

### Advanced Validation Rules

* `bool IsValidFileName` — Comprehensive filename validation
  * **Database ID Requirements:** All major IDs must be resolvable
  * **Date Format Validation:** Creation date/time must be valid
  * **Version Format:** Version must be numeric
  * **FSMRFP Code Requirements:** Code required for specific FSMRFP document types
  * **Extension Restrictions:** Only .xlsx and .zip allowed

### Error Handling

* `void SetError(string errorMessage)` — Centralized error management
  * **Validation State:** Sets IsValid to false
  * **Logger Integration:** Creates logger if needed, updates context
  * **Error Accumulation:** Multiple errors can be tracked

---

## Database Operations and Lifecycle Management

### Multi-Stage Database Integration

The PDIFile orchestrates a complex multi-table database workflow:

#### Stage 1: Receipt Logging

* `int InsertReceiptLog()` — Creates initial receipt record
  * **Table:** `pdi_File_Receipt_Log`
  * **FileRunID Generation:** Creates GUID if not provided
  * **OUTPUT Clause:** Captures generated FileID
  * **Batch Integration:** Updates `pdi_Client_Batch_Files` if BatchID available

#### Stage 2: ID Resolution

* `bool LoadIDs()` — Resolves all business entity IDs
  * **Complex Query:** Multi-table lookup for all ID values
  * **Duplicate Detection:** Checks for existing files with same parameters
  * **Context Loading:** Loads DataID, JobID, BatchID if available
  * **Performance:** Single query for all ID resolution

**ID Resolution SQL Patterns:**

* **New Files:** Complex JOIN across custodian, client, LOB, type tables
* **Existing Files:** Simpler lookup using FileID
* **Duplicate Check:** Searches for matching business context

#### Stage 3: File Logging

* `void InsertFileLog()` — Creates processing record
  * **Table:** `pdi_File_Log`
  * **Duplicate Detection:** Checks for existing Data_ID
  * **Business Context:** Records all parsed filename components
  * **Error Integration:** Sets ProcessingStage.Duplicate if duplicate found

#### Stage 4: Processing Queue

* `void InsertProcessing()` — Initiates processing workflow
  * **Validation Requirement:** Only for valid files
  * **Processing Integration:** Creates `pdi_Processing_Queue_Log` record
  * **JobID Capture:** Captures assigned JobID for tracking

---

## Processing Lifecycle Methods

### Main Processing Orchestrator

* `void Process()` — Central processing coordinator
  **Workflow:**
  1. Insert receipt log (unless loadOnly)
  2. Validate filename structure and content
  3. Load database IDs if valid
  4. Update receipt log with validation status
  5. Create file log if valid (duplicate check)
  6. Create processing queue if valid
  7. Write accumulated errors to database

### Retry and Recovery Support

* `int Retry(int retryCount)` — Retry processing support
  * **Batch Integration:** Updates batch file associations
  * **Receipt Log Update:** Refreshes receipt timestamps
  * **File Log Creation:** Creates missing file log records
  * **Processing Queue:** Creates missing processing records
  * **Error Replay:** Re-processes filename for error capture

* `int ProcessAfterLoadOnly()` — Convert from loadOnly to full processing
  * **Mode Switch:** Disables loadOnly flag
  * **Full Processing:** Triggers complete Process() workflow

---

## Parameter and Reporting Services

### Parameter Loading and Caching

* `Dictionary<string, string> GetAllParameters()` — Complete parameter retrieval
  * **Smart Loading:** Uses JobID or FileID depending on availability
  * **Caching:** Results cached for performance
  * **Fallback Logic:** Multiple loading strategies

* `static Dictionary<string, string> LoadParameters(int jobID, DBConnection dbInternal)` — Static parameter loader
  * **Complex Query:** 7+ table JOIN for complete context
  * **Job-Centric:** Loads all job-related parameters
  * **Dictionary Conversion:** Uses extension methods for safe conversion

### Validation and Error Reporting

* `int CountValidationErrors()` — Count file validation errors
* `Stream GetValidationErrorsCSV()` — Generate validation error report
  * **CSV Generation:** Uses DataTable ToCSV() extension
  * **Memory Stream:** Seekable stream for downloads
  * **UTF-8 Encoding:** Proper encoding for international characters

### Translation Reporting

* `int CountMissingFrench()` — Count missing French translations
* `Stream GetMissingFrenchCSV()` — Generate missing translation report
  * **Complex Query:** Multi-table JOIN across translation tables
  * **Deduplication:** DISTINCT to avoid duplicates
  * **LOB Context:** Includes line of business information

---

## Advanced Database Query Patterns

### ID Resolution Strategy

The LoadIDs method demonstrates sophisticated database patterns:

**Dynamic Query Selection:**

* Uses different SQL based on whether FileID is known
* Optimizes JOIN strategy based on available information
* Handles both new file analysis and existing file lookup

**Complex JOIN Pattern:**

```sql

DECLARE @dataID INTEGER; DECLARE @fileID INTEGER;
SELECT @dataID = Data_ID, @fileID = [File_ID] FROM [pdi_File_Log] 
WHERE business_context_match;
SELECT calculated_ids FROM multiple_tables 
WHERE context_conditions;
```

### Parameter Aggregation Queries

The parameter loading demonstrates enterprise-grade query complexity:

* **7+ Table JOINs:** Complete business context aggregation
* **COALESCE Usage:** Handles NULL values gracefully
* **Batch Integration:** LEFT OUTER JOIN for batch information
* **Time Calculations:** MIN/MAX aggregations for processing times

---

## Business logic and integration patterns

### Filename Convention Enforcement

The PDIFile enforces strict business rules through filename structure:

* **Standardized Format:** Ensures consistent file naming across organization
* **Business Context Encoding:** Filename contains complete processing context
* **Validation Integration:** Filename validation prevents invalid processing
* **Extensibility:** Handles optional components (Code, Note) for different document types

### Multi-Tenant Architecture Support

* **Client Isolation:** All operations respect client boundaries
* **LOB Segmentation:** Line of business context maintained throughout
* **Custodian Association:** Data custodian relationships preserved
* **Company Hierarchy:** Multi-level organizational structure support

### Error Handling and Resilience

* **Graceful Degradation:** Processing continues despite individual failures
* **Error Accumulation:** Multiple errors collected and reported together
* **Database Transaction Safety:** Individual operations isolated
* **Retry Support:** Built-in support for processing retry scenarios

### Processing Stage Integration

* **Stage Awareness:** Integrates with Processing enum stages
* **Status Tracking:** Maintains processing status throughout lifecycle
* **Queue Integration:** Creates and manages processing queue entries
* **Completion Tracking:** Supports end-to-end processing monitoring

---

## Technical implementation considerations

### Performance Optimization

* **Single Query Strategy:** LoadIDs loads all IDs in single database call
* **Caching Strategy:** Parameters and IDs cached to avoid repeated queries
* **Lazy Loading:** Database operations only when needed
* **Bulk Operations:** Uses bulk patterns where appropriate

### Memory Management

* **Stream Handling:** Proper stream lifecycle for CSV generation
* **Dictionary Management:** Efficient key-value storage for parameters
* **Resource Cleanup:** Proper disposal patterns for database resources
* **String Processing:** Efficient string manipulation for filename parsing

### Database Connection Management

* **Connection Reuse:** Single connection for all operations
* **Transaction Boundaries:** Assumes caller manages transactions
* **Parameter Safety:** All queries use parameterized SQL
* **Error Handling:** Comprehensive database error capture and reporting

### Text Processing Robustness

* **Case Sensitivity:** Proper case handling for filename components
* **Extension Validation:** Strict file type enforcement
* **Date Parsing:** Robust date/time parsing with validation
* **Character Encoding:** Proper handling of international characters

---

## Integration with broader PDI system

### Pipeline Orchestration Role

PDIFile serves as the foundational component for file processing:

* **Entry Point:** First component to process incoming files
* **Context Provider:** Supplies business context to all downstream processing
* **Database Bridge:** Connects filename conventions to database structure
* **Error Gateway:** First line of validation and error detection

### Azure Function Integration

* **Batch Processing:** Integrates with PDIBatch for multi-file operations
* **Queue Integration:** Creates processing queue entries for Azure Functions
* **Status Reporting:** Provides status information for function monitoring
* **Parameter Supply:** Supplies parameters for function execution

### Logger Integration

* **Error Tracking:** Deep integration with Logger for error management
* **Context Sharing:** Shares FileID, BatchID, RunID with logger
* **Validation Logging:** All validation errors logged through Logger
* **Database Persistence:** Logger handles error persistence to database

---

## Potential enhancements

### Performance Improvements

1. **Async Operations:** Convert to async/await pattern for database operations
2. **Batch ID Loading:** Bulk loading of multiple file IDs
3. **Connection Pooling:** More sophisticated connection management
4. **Query Optimization:** Further optimize complex ID resolution queries

### Functionality Extensions

1. **Custom Filename Formats:** Support for client-specific filename patterns
2. **Metadata Extraction:** Extract additional metadata from file properties
3. **Content Validation:** Basic file content validation during processing
4. **Version Management:** Enhanced version tracking and comparison

### Error Handling Enhancements

1. **Error Categories:** Categorized error types for better handling
2. **Error Recovery:** Automatic recovery strategies for common errors
3. **Validation Rules Engine:** Configurable validation rule system
4. **Error Analytics:** Pattern analysis for error prevention

### Integration Improvements

1. **Event Publishing:** Publish processing events for external systems
2. **Webhook Support:** Callback support for processing status updates
3. **API Integration:** RESTful API for file processing operations
4. **Message Queue Integration:** Integration with enterprise message queues

---

## Action items for system maintenance

1. **Database Performance:** Monitor and optimize ID resolution query performance
2. **Filename Pattern Evolution:** Plan for filename format changes and backward compatibility
3. **Error Pattern Analysis:** Analyze error patterns for prevention opportunities
4. **Processing Time Monitoring:** Track processing times and identify bottlenecks
5. **Database Growth Management:** Plan for growth in file and processing logs
6. **Parameter Usage Analysis:** Monitor parameter usage patterns for optimization opportunities
