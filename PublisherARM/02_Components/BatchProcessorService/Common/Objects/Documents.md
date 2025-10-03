# Document Class — Core Domain Entity (analysis)

## One-line purpose

Core domain entity representing a publishable document with comprehensive lifecycle management including document generation, archiving, comparison, streaming, and customizable filename pattern generation using the BlueID Composition Engine.

---

## Files analyzed

* `BatchProcessorService/Document.cs`

---

## What this code contains (high level)

1. **Document Entity Model** — Represents a document with metadata, template association, and cultural localization
2. **Document Generation Pipeline** — Creates PDF documents via BlueID Composition Engine integration
3. **Document Comparison System** — Generates blackline/redline comparison documents between date ranges
4. **Archive Management** — Handles document archiving to Azure Storage with versioning support
5. **Filename Pattern System** — Flexible token-based filename generation with length constraints
6. **Browser Streaming** — HTTP response streaming for document downloads and previews
7. **Lazy Loading Architecture** — Properties loaded on-demand from database with caching

This class represents the central business entity for document management in the Publisher system, orchestrating document creation, versioning, archiving, and delivery.

---

## Classes, properties and methods (developer reference)

### `Document : ObjectBase` - Core Document Domain Entity

**Purpose:** Encapsulates document metadata, lifecycle operations, and integration with the composition engine for PDF generation and management.

#### Constants

##### Filename Patterns

* `private const string DEFAULT_DOCUMENT_FILENAME_PATTERN = "<DOCUMENT_CODE>_<DOCUMENT_NAME>_<DOCUMENT_TYPE>.pdf"` — Default filename template for generated documents

**Pattern Tokens:**

* `<DOCUMENT_CODE>` — Document identifier code
* `<DOCUMENT_NAME>` — Human-readable document name
* `<DOCUMENT_TYPE>` — Document type/category

#### Private Fields

##### Core Properties

* `private int _documentID = -1` — Unique document identifier (database primary key)
* `private string _documentName = null` — Display name of the document
* `private string _documentCode = null` — Unique document code/number
* `private string _documentKeywords = null` — Search keywords for document discovery
* `private string _lastImportDate = null` — Timestamp of last data import operation
* `private int? _sortOrder = null` — Optional sort position for document ordering
* `private DocumentTemplate _documentTemplate = null` — Associated template for document rendering

##### Custom Filename Support

* `private string _documentFileName = null` — Customizable filename (added 2014-06-16)

**Historical Note:** Custom filename functionality was added mid-project, indicating evolving business requirements around document naming conventions.

#### Constructors

##### Single Culture Constructor

* `public Document(int documentID, string cultureCode) : this(documentID, cultureCode, cultureCode)` — Creates document with same site and document culture

##### Dual Culture Constructor

* `public Document(int documentID, string siteCultureCode, string documentCultureCode)` — Primary constructor supporting different cultures for site UI and document content

**Cultural Architecture:**

* `siteCultureCode` — Culture for site interface elements
* `documentCultureCode` — Culture for document content generation

**Lazy Loading Pattern:** Constructor only sets identifiers; properties loaded on first access via `ensurePropertiesAreSet()` (inherited from ObjectBase)

---

## Document Generation Methods

### Browser Streaming Operations

#### Standard Document Streaming

* `public void streamDocumentToBrowser(CompositionDocument.Quality quality)` — Streams generated PDF to browser
* `public void streamXmlDocumentToBrowser(CompositionDocument.Quality quality)` — Streams composition XML to browser for debugging

**Quality Parameter:** Enum controlling PDF generation quality (likely Draft/Final/Print quality levels)

#### Comparison Document Streaming

* `public void streamDocumentComparisonToBrowser(CompositionDocument.Quality quality, DateTime startDate, DateTime endDate)` — Streams comparison PDF to browser
* `public void streamXmlDocumentComparisonToBrowser(CompositionDocument.Quality quality, DateTime startDate, DateTime endDate)` — Streams comparison XML for debugging

**Date Range Pattern:** Compares document state between `startDate` and `endDate` to generate redline markup

### Core Document Generation

#### Primary Document Creation

* `public string createDocument(CompositionDocument.Quality quality, bool showXml, bool streamResponseToBrowser, bool memberOfBatch, bool stackPDF)` — Creates PDF document with flexible output options

**Parameters:**

* `quality` — PDF quality level for generation
* `showXml` — Debug flag to output composition XML instead of PDF
* `streamResponseToBrowser` — Stream result to HTTP response
* `memberOfBatch` — Flag indicating document is part of batch processing
* `stackPDF` — Stack multiple PDFs into single file

**Returns:** Request ID from composition engine (null if creation fails)

**Processing Flow:**

1. **Existence Check:** Validates document exists before processing
2. **Data Preparation:** Creates `CompositionDocumentData` wrapper
3. **Composition Engine Invocation:** Uses `BlueIDCompositionDocument` for PDF generation
4. **Output Handling:** Either XML debug output or PDF streaming
5. **Logging:** Records document downloads for audit trail

**Error Handling Pattern:**

```csharp
if (this.exists)
{
    // Process document
}
else
{
    Log.error("Document.createDocument()", "Request to create inactive document. documentID: " + this.documentID);
    return null;
}
```

#### Comparison Document Creation

* `public string createComparisonDocument(CompositionDocument.Quality quality, DateTime startDate, DateTime endDate, bool showXml, bool streamResponseToBrowser, bool memberOfBatch)` — Generates blackline comparison document

**Critical Business Logic - Date Adjustment:**

```csharp
// Nirav Vyas - 2016-10-06
// Changing from endDate to endDate.AddMinutes(1)
// If change made to field and blackline generated right away, will not pick up the change
// The reason is that field saves with seconds (e.g., 2016-10-06 2:20:40)
// But UI has date/time picker without second picker
// So it sets second portion as 00 (e.g., 2016-10-06 2:20:00)
ComparisonDocumentData documentData = new ComparisonDocumentData(this, startDate, endDate.AddMinutes(1));
```

**Business Problem Solved:** UI date picker precision mismatch with database timestamp precision causing missed changes in comparison documents.

---

## Archive Management Methods

### Archive Operations

#### Last Archive Date Query

* `public DateTime lastArchivedDate(DateTime priorTo, bool finalOnly)` — Retrieves last archive date before specified date

**Parameters:**

* `priorTo` — DateTime (UTC) to search backward from
* `finalOnly` — Only consider archives marked as "final"

**Returns:** Last archive date in UTC, or `DateTime.MinValue` if never archived

**Use Case:** Determines if document has been archived for compliance or audit purposes.

#### Document Archiving

* `public void archiveDocument(User user, bool isFinal, CompositionDocument.Quality quality)` — Archives document with user context
* `public void archiveDocument(Guid userKey, bool isFinal, CompositionDocument.Quality quality)` — Archives document with user key

**Archive Process:**

1. **PDF Generation:** Creates document at specified quality level
2. **Database Record:** Inserts archive metadata via `sp_SAVE_DOCUMENT_ARCHIVE`
3. **Azure Storage:** Saves PDF to blob storage in `DocumentArchiveContainerName`
4. **Naming Convention:** Archives stored as `{archiveID}.pdf`

**Final vs. Non-Final Archives:**

* **Final:** Permanent regulatory compliance archives
* **Non-Final:** Working versions or draft archives

**Storage Integration:**

```csharp
HandyFunctions.saveToAzureStorage(
    Constants.DocumentArchiveContainerName, 
    compositionDocument.documentStream, 
    newArchiveID.ToUpper() + ".pdf"
);
```

---

## Filename Pattern System

### Pattern Generation Methods

#### Standard Filename Generation

* `public string documentFileNameWithPattern(string filenamePattern)` — Generates filename from pattern
* `public string documentFileNameWithPattern(string filenamePattern, List<KeyValuePair<string, string>> replacementTokens, bool stackedPDF)` — Advanced pattern with custom tokens

**Supported Pattern Tokens:**

* `<DOCUMENT_NUMBER>` — Document code (deprecated, use DOCUMENT_CODE)
* `<DOCUMENT_CODE>` — Current document code identifier
* `<TEMPLATE_NAME>` — Name of document template
* `<DOCUMENT_TYPE>` — Document type from template
* `<CULTURE_CODE>` — Culture code (e.g., "en-CA", "fr-CA")
* `<UNIQUE_ID>` — Generated GUID for uniqueness
* `<YEAR>`, `<MONTH>`, `<DAY>` — Current date components
* `<DOCUMENT_NAME(length)>` — Document name truncated to specified length
* `<LANGUAGE(length)>` — Language name truncated to specified length

**Token Replacement Logic:**

```csharp
filename = Regex.Replace(filename, "<DOCUMENT_CODE>", this.documentCode, RegexOptions.IgnoreCase);
filename = Regex.Replace(filename, "<UNIQUE_ID>", Guid.NewGuid().ToString(), RegexOptions.IgnoreCase);
filename = Regex.Replace(filename, "<YEAR>", DateTime.UtcNow.Year.ToString(), RegexOptions.IgnoreCase);
```

**Stacked PDF Special Case:**
When `stackedPDF` is true, returns `this.documentFileName` directly without pattern processing.

#### Length-Constrained Token Replacement

* `private string replaceTokenToSpecifiedLength(string filename, string bareToken, string replacement)` — Replaces tokens with length constraints

**Pattern:** `<TOKEN(number)>` where number specifies maximum length

**Example:**

```note
<DOCUMENT_NAME(20)> with value "Very Long Document Name Here"
Result: "Very Long Document N"
```

**Recursive Processing:** Handles multiple instances of same token with different length constraints.

**Algorithm:**

1. **Regex Match:** Finds token with optional length parameter `(\(\d+\))?`
2. **Length Extraction:** Parses numeric length from parentheses
3. **Truncation:** Applies substring if replacement exceeds length
4. **Recursion:** Processes additional instances of same token

#### Comparison Document Filenames

* `public string compareDocumentFileNameWithPattern(string filenamePattern)` — Generates blackline document filename
* `public string compareDocumentFileNameWithPattern(string filenamePattern, List<KeyValuePair<string, string>> replacementTokens, bool stackedPDF)` — Advanced comparison filename

**Blackline Naming Convention:**

```csharp
// Adds "_Blackline" before file extension
List<string> filenamePatternPieces = new List<string>(filenamePattern.Split('.'));
filenamePatternPieces[Math.Max(0, filenamePatternPieces.Count - 2)] += "_" + App_GlobalResources.SiteCopy.Blackline;
```

**Example:** `Document_Code_Name.pdf` becomes `Document_Code_Name_Blackline.pdf`

---

## Property Accessors

### Database-Backed Properties (Lazy Loaded)

#### Core Properties of Document Object

* `public int documentID { get; set; }` — Unique document identifier
* `public string documentCode { get; set; }` — Document code/number
* `public string documentName { get; set; }` — Display name
* `public string documentKeywords { get; set; }` — Search keywords
* `public string documentFileName { get; }` — Custom or pattern-generated filename
* `public int? sortOrder { get; set; }` — Optional sort position
* `public string lastImportDate { get; set; }` — Last import timestamp

#### Relationship Properties

* `public DocumentTemplate documentTemplate { get; set; }` — Associated template

#### Computed Properties

* `public string compareDocumentFileName { get; }` — Filename with "_Blackline" suffix
* `public string compareDocumentName { get; }` — Document name with "Blackline" text

**Property Loading Pattern:**

```csharp
public string documentCode
{
    get
    {
        ensurePropertiesAreSet();  // Inherited from ObjectBase
        return _documentCode;
    }
    set { _documentCode = value; }
}
```

---

## Database Integration Methods

### Property Loading

* `protected override void setProperties()` — Loads document properties from database

**Data Loading Process:**

1. **Connection:** Uses `Constants.connectionString` for database access
2. **Stored Procedure:** Calls `sp_SELECT_DOCUMENT` with documentID and cultureCode
3. **Property Mapping:** Maps SqlDataReader columns to object properties
4. **Template Loading:** Instantiates `DocumentTemplate` from template ID
5. **Filename Logic:** Uses custom filename or generates from default pattern

**Filename Resolution:**

```csharp
this.documentFileName = (
    (string.IsNullOrWhiteSpace(documentInfo["DocumentFilename"].ToString()) || 
     documentInfo["DocumentFilename"].ToString().Trim().ToUpper() == "NULL") 
    ? this.documentFileNameWithPattern(DEFAULT_DOCUMENT_FILENAME_PATTERN) 
    : documentInfo["DocumentFilename"].ToString()
);
```

**Null Handling:** Checks for both null and string "NULL" from database for filename field.

---

## Business logic and integration patterns

### Document Lifecycle Management

The class orchestrates complete document lifecycle:

**Creation Phase:**

* Template-based document generation
* PDF creation via composition engine
* Custom filename assignment

**Working Phase:**

* Document editing and field updates
* Version comparison and blackline generation
* Quality variations (draft, final, print)

**Archive Phase:**

* Final document archiving to Azure Storage
* Historical version preservation
* Compliance and audit support

### BlueID Composition Engine Integration

Central integration point for document generation:

**Composition Pattern:**

```csharp
CompositionDocumentData documentData = new CompositionDocumentData(this);
using (BlueIDCompositionDocument compositionDocument = new BlueIDCompositionDocument(documentData))
{
    compositionDocument.quality = quality;
    compositionDocument.createDocument();
}
```

**Disposable Pattern:** Uses `using` statement for proper resource cleanup.

### Cultural Localization Architecture

Sophisticated dual-culture support:

**Site Culture:** Controls UI elements and system messages
**Document Culture:** Controls document content and formatting
**Fallback:** Can use same culture for both site and document

### Batch Processing Integration

Document generation aware of batch context:

**Batch Flags:**

* `memberOfBatch` — Indicates document is part of batch operation
* `stackPDF` — Enables PDF stacking for consolidated output

**Use Cases:**

* Bulk document generation
* Document package creation
* Automated publishing workflows

### Archive Compliance System

Archive functionality supports regulatory requirements:

**Audit Trail:** Records who archived, when, and whether final
**Historical Access:** Query historical archive dates for compliance
**Final Designation:** Distinguishes regulatory-required archives from working versions
**Azure Storage:** Scalable cloud storage for long-term retention

---

## Technical implementation considerations

### Performance Characteristics

* **Lazy Loading:** Properties loaded on first access, cached thereafter
* **Database Calls:** Single stored procedure call per document instance
* **Composition Engine:** External service call for PDF generation
* **Azure Storage:** Network I/O for archive operations

### Error Handling Strategy

Comprehensive exception handling with detailed logging:

**Pattern:**

```csharp
try
{
    // Operation
}
catch (Exception exception)
{
    Log.error("MethodName", exception, "context: " + contextInfo);
}
```

**Logging Context:** All errors include relevant identifiers and parameters for troubleshooting.

### Memory Management

**Disposable Resources:** Properly disposes `BlueIDCompositionDocument` and `DatabaseConnector`
**Stream Handling:** Uses `using` statements for stream cleanup
**Property Caching:** Caches loaded properties to avoid repeated database calls

### Date/Time Handling

**UTC Consistency:** Archive dates stored and retrieved in UTC
**Timezone Awareness:** Uses `Constants.DEFAULT_TIMEZONE` for localization
**Precision Issues:** Handles date picker precision mismatches with one-minute buffer

---

## Integration with broader Publisher system

### Template System Integration

Documents are always associated with templates:

* `DocumentTemplate` relationship for rendering rules
* Template determines document type and feed type
* Template name used in filename pattern generation

### Azure Services Integration

Multiple Azure service touchpoints:

* **Blob Storage:** Archive document storage
* **Container Management:** Uses configured container names
* **Retry Policies:** Leverages Azure storage retry policies

### Logging and Audit System

Comprehensive logging integration:

* **Document Downloads:** Tracked via `Log.documentDownload()`
* **Error Logging:** All exceptions logged with context
* **Archive Tracking:** Database records all archive operations

### Constants Configuration

Heavy reliance on system configuration:

* `connectionString` — Database connectivity
* `DocumentArchiveContainerName` — Azure storage container
* `ENGLISH_CULTURE_CODE` — Cultural defaults
* `DEFAULT_TIMEZONE` — Timezone for date operations

---

## Potential enhancements

### Performance Optimizations

1. **Bulk Loading:** Support loading multiple documents in single database call
2. **Property Caching:** Implement distributed cache for frequently accessed documents
3. **Async Operations:** Convert to async/await pattern for I/O operations
4. **Streaming Optimization:** Stream large PDFs without full memory load

### Feature Enhancements

1. **Version Control:** Built-in version tracking beyond archive dates
2. **Preview Generation:** Generate thumbnail previews for documents
3. **Metadata Enrichment:** Additional metadata fields for search and discovery
4. **Collaborative Features:** Check-in/check-out for concurrent editing

### Architecture Improvements

1. **Repository Pattern:** Extract data access to repository classes
2. **Dependency Injection:** Inject composition engine and storage services
3. **Event Sourcing:** Track all document state changes as events
4. **CQRS Pattern:** Separate read and write models for scalability

### Testing Improvements

1. **Mockable Dependencies:** Extract external dependencies to interfaces
2. **Unit Test Coverage:** Comprehensive test coverage for filename patterns
3. **Integration Tests:** Test end-to-end document generation workflows
4. **Performance Tests:** Benchmark document generation under load

---

## Action items for system maintenance

1. **Code Cleanup:** Remove or document commented historical code sections
2. **Performance Monitoring:** Track document generation times and identify bottlenecks
3. **Archive Storage Review:** Monitor Azure storage costs and implement retention policies
4. **Error Rate Analysis:** Review error logs for common failure patterns
5. **Filename Pattern Validation:** Ensure all pattern tokens are documented and tested
6. **Cultural Support Verification:** Validate bilingual functionality works correctly
7. **Composition Engine SLA:** Monitor composition engine availability and response times

---

## Critical considerations

### External Dependency Risk

Heavy reliance on BlueID Composition Engine creates single point of failure:

* **Availability:** System functionality depends on external service availability
* **Performance:** Document generation performance tied to external service
* **Error Handling:** Must gracefully handle composition engine failures

**Recommendation:** Implement circuit breaker pattern and fallback mechanisms.

### Date Precision Workaround

The one-minute buffer for comparison dates is a workaround for UI limitations:

* **Root Cause:** UI date picker doesn't include seconds
* **Workaround:** Adds one minute to end date
* **Risk:** Could include unintended changes in edge cases

**Recommendation:** Consider server-side timestamp rounding or enhanced UI date picker.

### Filename Pattern Complexity

Complex filename pattern system with many tokens and length constraints:

* **Maintenance:** Regex-based replacement logic difficult to test and debug
* **Documentation:** Token meanings not centrally documented
* **Validation:** No validation of pattern correctness before use

**Recommendation:** Create comprehensive filename pattern documentation and validation system.
