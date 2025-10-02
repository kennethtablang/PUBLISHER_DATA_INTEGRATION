# MergeTables Helper — Component Analysis

## One-line purpose

Intelligent table data merging system that combines current financial data with historical Publisher data using configurable merge strategies, rerun detection, and interim value handling for time-series financial document processing.

---

## Files analyzed

* `Publisher_Data_Operations/Helper/MergeTables.cs`

---

## What this code contains (high level)

1. **Configurable Merge Strategy Engine** — Database-driven merge settings for different field types and table structures
2. **Historical Data Integration** — Retrieves and processes historical data from Publisher database for continuity
3. **Intelligent Rerun Detection** — Identifies when data represents updates vs. new periods to determine merge behavior
4. **Interim Value Management** — Special handling for interim/preliminary data with configurable offset rules
5. **Multi-Dimensional Merging** — Supports both row-based (time-series) and column-based (data expansion) merging patterns

This component ensures financial tables maintain historical continuity while incorporating new data, critical for regulatory reporting and trend analysis.

---

## Classes, properties and methods (developer reference)

### `MergeTablesSettings` - Configuration Class

**Purpose:** Encapsulates merge configuration parameters for specific field types, providing serializable settings for table merging operations.

#### Properties

* `Tuple<int, int> CheckField { get; set; }` — Row/column coordinates for rerun detection and merge control
* `int DesiredRows { get; set; }` — Target number of rows after merge (0 = column-based merge)
* `int DesiredColumns { get; set; }` — Target number of columns after merge (0 = row-based merge)
* `string InterimIndicator { get; set; }` — Text pattern identifying interim/preliminary data
* `int InterimOffset { get; set; }` — Additional rows/columns allowed for interim data

#### Constructors

* `MergeTablesSettings(int checkRow, int checkCol, int desiredRows, int desiredColumns, string interimIndicator, int interimOffset)` — Direct parameter constructor
* `MergeTablesSettings(DataRow dr)` — Database row constructor using extension methods for safe column access

---

### `MergeTables` - Main Merging Engine

**Purpose:** Orchestrates table data merging between current and historical data using configurable strategies and intelligent detection algorithms.

#### Core Properties

##### Configuration and State

* `MergeTablesSettings MergeSettings { get; private set; }` — Current merge configuration
* `DataTable HistoricTableData` — Cached historical data from Publisher database
* `DataTable MergeTableSettings` — Cached merge configuration data
* `Logger _log` — Error logging and tracking

##### Context Tracking

* `string DocumentNumber` — Current document identifier for cache management
* `string FieldNamePrefix` — Field name prefix for efficient querying
* `int ClientID, DataTypeID, DocumentTypeID` — Context identifiers for cache invalidation

##### Computed Properties

* `int CheckRow` — Resolved check field row (supports negative indexing from end)
* `int CheckColumn` — Resolved check field column (supports negative indexing from end)
* `List<string> GetMergeFieldNamePrefix` — Available field name prefixes from settings

#### Constructor

##### Configuration Constructors

* `MergeTables()` — Default constructor
* `MergeTables(int checkRow, int checkCol, int desiredRows, int desiredColumns, string interimIndicator, int interimOffset)` — Direct settings
* `MergeTables(MergeTablesSettings mergeSettings)` — Settings object constructor

##### Database Integration Constructor

* `MergeTables(RowIdentity rowIdentity, int dataTypeID, DBConnection conPdi, Logger log = null)` — Database-driven initialization
  * **Validation:** Comprehensive null checking for all required parameters
  * **Auto-Loading:** Automatically loads merge settings from database

---

## Configuration Management Methods

### Settings Loading and Caching

* `int LoadMergeTableSettings(RowIdentity rowIdentity, int dataTypeID, DBConnection dbPdi)` — Load merge configurations from database
  * **Caching Strategy:** Only queries database when context changes (ClientID, DataTypeID, DocumentTypeID)
  * **Query Structure:** Targets `pdi_Data_Type_Merge_Table` with composite key filtering
  * **Return Value:** Number of loaded settings or -1 on error

* `bool GetMergeSettings(string fieldName, RowIdentity rowIdentity, int dataTypeID, DBConnection dbPdi)` — Load and apply settings for specific field
  * **Field Prefix Matching:** Uses field name prefix for configuration lookup
  * **Auto-Loading:** Calls `LoadMergeTableSettings` if needed
  * **Single Match Requirement:** Returns true only if exactly one configuration found

* `bool GetMergeSettings(string fieldName)` — Apply settings from cached data
  * **Cache Dependency:** Requires pre-loaded `MergeTableSettings`
  * **Field Prefix Logic:** Extracts prefix from field name for matching

---

## Historical Data Management

### Data Loading and Caching

* `int LoadHistoricalData(string fieldName, RowIdentity rowIdentity, DBConnection dbPub)` — Load historical data from Publisher
  * **Complex Query:** Joins multiple Publisher tables (DOCUMENT_FIELD_VALUE, DOCUMENT_FIELD_ATTRIBUTE, DOCUMENT, etc.)
  * **Bilingual Support:** Retrieves both English and French content
  * **Header Integration:** Includes header field data for complete table context
  * **Caching Logic:** Caches per field prefix + document number + client combination

**SQL Query Structure:**

* **Base Tables:** DOCUMENT_FIELD_VALUE (content), DOCUMENT_FIELD_ATTRIBUTE (metadata)
* **Context Tables:** DOCUMENT, LINE_OF_BUSINESS, COMPANY for hierarchy
* **Language Support:** LEFT OUTER JOIN for French content
* **Header Support:** LEFT OUTER JOIN for header fields (field name + 'h')
* **Filtering:** Active records only, specific client/document/field pattern

### Data Retrieval Methods

* `PDI_DataTable GetHistoricFieldTable(string fieldName, RowIdentity rowIdentity, DBConnection dbPub)` — Get historical table for field
  * **XML Processing:** Converts stored XML content to DataTable using extension methods
  * **Single Record Logic:** Returns table only if exactly one historical record found

* `string GetHistoricFieldString(string fieldName, bool getFrench = false)` — Get raw historical content
  * **Language Selection:** Returns French content if available and requested
  * **Column Safety:** Uses safe column access methods
  * **Fallback Logic:** Falls back to positional indexing if named columns unavailable

* `PDI_DataTable GetHistoricFieldTable(string fieldName, bool getFrench = false)` — Get historical table with language selection
* `string GetHistoricHeaderString(string fieldName, bool getFrench = false)` — Get historical header content

---

## Core Merging Logic

### Main Merge Orchestrator

* `string MergeTableData(string currentDataTable, string fieldName, RowIdentity rowIdentity = null, DBConnection dbPub = null, bool getFrench = false)` — Primary merge method
  * **Data Loading:** Loads historical data if database connection provided
  * **Settings Validation:** Ensures merge settings available for field
  * **Data Validation:** Verifies historical data availability and structure
  * **XML Processing:** Converts XML strings to DataTables for processing
  * **Result Formatting:** Returns merged data as XML string

**Process Flow:**

1. Load historical data (if needed)
2. Get merge settings for field
3. Validate historical data availability
4. Convert XML to DataTables
5. Perform table merging
6. Convert result back to XML

### Merge Strategy Implementation

* `PDI_DataTable CombineTables(PDI_DataTable newTable, PDI_DataTable oldTable)` — Core table combination logic
  * **Rerun Detection:** Determines if this is an update vs. new data
  * **Interim Handling:** Removes interim values before merging (unless rerun)
  * **Strategy Selection:** Row-based vs. column-based merging
  * **Data Preservation:** Maintains extended properties and formatting
  * **Size Management:** Applies `RemoveExtra` to maintain desired dimensions

---

## Detection and Analysis Methods

### Rerun Detection

* `bool IsRerun(PDI_DataTable table1, PDI_DataTable table2)` — Intelligent rerun detection
  * **Check Field Analysis:** Compares content at specified check field coordinates
  * **Text Normalization:** Uses comprehensive text cleaning for comparison
    * `RestoreAngleBrackets()` — Handles XML encoding
    * `RemoveHTML()` — Strips HTML formatting
    * `RemoveExceptAlphaNumeric()` — Normalizes to alphanumeric only
  * **Boundary Handling:** Supports negative indexing for last row/column
  * **Validation:** Throws ArgumentOutOfRangeException for invalid check fields

### Interim Value Detection

* `bool IsInterim(PDI_DataTable checkTable)` — Identifies interim/preliminary data
  * **Pattern Matching:** Searches for interim indicator in check field content
  * **Text Processing:** Same normalization as rerun detection
  * **Configuration-Driven:** Uses `InterimIndicator` from merge settings

---

## Data Manipulation Methods

### Cell and Row Operations

* `PDI_DataTable CopyCells(PDI_DataTable target, PDI_DataTable source, int offset = 1)` — Intelligent cell copying
  * **Dynamic Expansion:** Adds rows/columns as needed based on merge settings
  * **Selective Copying:** Copies cells based on check field position and data availability
  * **Property Preservation:** Maintains extended properties using `CopyExtendedProperties`
  * **Blank Value Logic:** Only overwrites blank target cells with non-blank source cells

* `void CopyExtendedProperties(PDI_DataRow target, PDI_DataRow source)` — Extended property management
  * **Property Merging:** Updates existing properties, adds new ones
  * **Key-Based Copying:** Maintains all extended property keys and values

### Table Size Management

* `PDI_DataTable RemoveExtra(PDI_DataTable checkTable)` — Table size enforcement
  * **Interim Awareness:** Considers interim offset when calculating limits
  * **Direction Handling:** Removes from end or beginning based on check field position
  * **Row vs Column Logic:** Applies appropriate dimension limits

---

## Advanced Merge Scenarios

### Row-Based Merging (Time Series)

**Use Case:** Financial data where new periods are added as rows

* **Column Consistency:** Ensures matching column count between tables
* **Temporal Insertion:** New data inserted at appropriate time position
* **Historical Preservation:** Maintains chronological data sequence
* **Rerun Handling:** Updates existing period data rather than adding new rows

### Column-Based Merging (Data Expansion)

**Use Case:** Financial data where new metrics are added as columns

* **Row Consistency:** Ensures matching row count between tables
* **Data Integration:** Combines new columns with historical columns
* **Position Control:** Uses check field to control insertion position
* **Extended Properties:** Preserves formatting and metadata across merge

### Interim Data Handling

**Special Logic:** Manages preliminary/interim data that may be replaced

* **Detection:** Identifies interim data using configurable indicator text
* **Offset Application:** Allows additional space for interim data
* **Replacement Logic:** Removes interim data when final data arrives (unless rerun)
* **Preservation:** Maintains interim data during rerun scenarios

---

## Business logic and integration patterns

### Financial Reporting Continuity

The MergeTables system supports critical financial reporting requirements:

* **Time Series Maintenance:** Preserves historical trends while adding current period data
* **Regulatory Compliance:** Maintains required historical periods for regulatory filings
* **Data Integrity:** Ensures consistent data structure across reporting periods
* **Preliminary Data Handling:** Manages interim/preliminary data replacement workflows

### Publisher Database Integration

Deep integration with Publisher data storage:

* **Multi-Language Support:** Handles English/French content consistently
* **Header Field Integration:** Maintains table headers alongside data
* **Document Hierarchy:** Respects company/LOB/document relationships
* **Active Record Filtering:** Only processes currently active data

### Caching and Performance

Intelligent caching strategy for production performance:

* **Context-Based Invalidation:** Cache invalidation based on business context changes
* **Prefix-Based Grouping:** Efficient querying using field name prefixes
* **Single Query Strategy:** Loads all related data in single database call
* **Memory Management:** Balances performance with memory usage

---

## Technical implementation considerations

### Error Handling and Resilience

* **Exception Types:** Specific exceptions for different failure modes (ArgumentNullException, ArgumentOutOfRangeException)
* **Data Validation:** Comprehensive validation before attempting merge operations
* **Graceful Degradation:** Falls back to current data when historical data unavailable
* **Logging Integration:** Comprehensive error logging for troubleshooting

### Text Processing Complexity

Advanced text normalization for reliable comparison:

* **XML Handling:** Proper handling of XML-encoded content
* **HTML Stripping:** Removes HTML formatting for comparison
* **Alphanumeric Normalization:** Reduces content to comparable form
* **Cultural Considerations:** Handles different text formats and encodings

### Database Connection Management

* **Connection Reuse:** Efficient use of provided database connections
* **Transaction Safety:** Assumes caller manages transaction boundaries
* **Query Optimization:** Complex queries optimized for Publisher schema
* **Connection Validation:** Validates connection availability before use

### Memory and Performance Characteristics

* **DataTable Operations:** Extensive use of DataTable for structured data manipulation
* **XML Conversion:** Multiple XML ↔ DataTable conversions per merge operation
* **Caching Strategy:** Reduces database calls through intelligent caching
* **Large Data Handling:** Designed for typical financial document table sizes

---

## Integration with broader PDI system

### Pipeline Position

MergeTables operates at a critical juncture in the PDI pipeline:

* **Post-Extraction:** Works with data after AsposeLoader extracts from Excel
* **Pre-Publisher:** Prepares data for Publisher import with historical context
* **Data Enrichment:** Adds historical context to current period data
* **Quality Assurance:** Ensures data continuity and completeness

### Database Schema Dependencies

* **PDI Database:** `pdi_Data_Type_Merge_Table` for configuration
* **Publisher Database:** Multiple tables for historical data retrieval
* **Schema Evolution:** Must adapt to changes in either database schema
* **Data Type Consistency:** Maintains consistency across database boundaries

### Extension Method Dependencies

Heavy reliance on PDI extension methods:

* **XML Processing:** `XMLtoDataTable()`, `DataTabletoXML()`
* **Safe Data Access:** `GetExactColumnStringValue()`, `GetExactColumnIntValue()`
* **Text Processing:** `IsNaOrBlank()`, `RestoreAngleBrackets()`, etc.
* **Field Processing:** `GetFieldNamePrefix()`, `EscapeSQL()`

---

## Potential enhancements

### Performance Optimizations

1. **Async Operations:** Asynchronous database operations for better scalability
2. **Connection Pooling:** More efficient database connection usage
3. **Parallel Processing:** Parallel merge operations for multiple fields
4. **Memory Streaming:** Stream processing for very large tables

### Functionality Extensions

1. **Custom Merge Strategies:** Pluggable merge algorithms for specific business needs
2. **Conflict Resolution:** Advanced conflict resolution when data differs
3. **Version Control:** Track merge history and support rollback operations
4. **Data Validation:** Built-in validation of merge results

### Configuration Enhancements

1. **UI Configuration:** Web-based interface for merge configuration management
2. **Template-Based Config:** Configuration templates for common merge patterns
3. **Dynamic Configuration:** Runtime configuration adjustment based on data patterns
4. **Configuration Versioning:** Track configuration changes over time

### Monitoring and Diagnostics

1. **Performance Metrics:** Detailed performance monitoring and alerting
2. **Merge Auditing:** Complete audit trail of merge operations
3. **Data Quality Metrics:** Metrics on merge success rates and data quality
4. **Visual Diagnostics:** Tools for visualizing merge operations and results

---

## Action items for system maintenance

1. **Performance Monitoring:** Monitor merge operation performance and identify bottlenecks
2. **Configuration Review:** Regular review of merge configurations for accuracy and effectiveness
3. **Database Schema Tracking:** Monitor changes to Publisher schema that may affect queries
4. **Cache Effectiveness:** Analyze cache hit rates and optimize caching strategies
5. **Error Pattern Analysis:** Monitor merge errors for patterns indicating systemic issues
6. **Data Quality Assessment:** Regular assessment of merge result quality and accuracy
