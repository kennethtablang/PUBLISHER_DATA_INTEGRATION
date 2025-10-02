# ExcelDataDocument — Structured Excel Processing (analysis)

## One-line purpose

Specialized Excel document processor that handles standardized three-sheet Excel files (Text Fields, Table Fields, Chart Fields) for import/export operations with comprehensive validation, metrics calculation, and template cleaning capabilities.

---

## Files analyzed

* `BatchProcessorService/ExcelDataDocument.cs`

---

## What this code contains (high level)

1. **Structured Sheet Management** — Enforces specific three-sheet Excel format for document field data
2. **Import/Export Operations** — Dual-mode processing for both data import and export template generation
3. **Comprehensive Validation** — Multi-layer validation including sheet existence, column counts, and row requirements
4. **Template Cleaning** — Removes data rows from Excel templates while preserving structure
5. **Metrics Calculation** — Analyzes document and field uniqueness across all sheets
6. **Custom Exception Hierarchy** — Specialized exceptions for detailed error reporting
7. **OpenXML Integration** — Uses DocumentFormat.OpenXML for direct Excel file manipulation

This class extends the base ExcelDocument class to provide document-specific functionality for the Publisher system's data exchange operations.

---

## Classes, properties and methods (developer reference)

### `ExcelDataDocument : ExcelDocument` - Structured Excel Document Processor

**Purpose:** Handles standardized Excel files containing document field data across three predefined sheets with validation and metrics capabilities.

#### Constants and Enumerations

##### Operation Type

* `public enum Operation { Import, Export }` — Defines processing mode for the Excel document

##### Sheet Name Constants

* `public const string SHEET_NAME_TEXT = "Text Fields"` — Text field data sheet name
* `public const string SHEET_NAME_TABLE = "Table Fields"` — Table field data sheet name  
* `public const string SHEET_NAME_CHART = "Chart Fields"` — Chart field data sheet name

##### Validation Constants

* `private const int MIN_HEADER_COLUMNS_TEXT = 5` — Minimum columns required in Text Fields sheet
* `private const int MIN_HEADER_COLUMNS_TABLE = 6` — Minimum columns required in Table Fields sheet
* `private const int MIN_HEADER_COLUMNS_CHART = 5` — Minimum columns required in Chart Fields sheet
* `private const int MIN_ROWS_IN_IMPORT_FILE = 2` — Minimum rows including header (header + 1 data row)

##### Column Reference Constants

* `protected const string DOCUMENT_CODE_COLUMN = "A"` — Column A contains document codes
* `protected const string FIELD_NAME_COLUMN = "B"` — Column B contains field names

#### Private Fields

##### Cached Metrics

* `private int _uniqueDocumentCount = -1` — Cached count of unique documents (lazy loaded)
* `private int _uniqueFieldCount = -1` — Cached count of unique fields (lazy loaded)  
* `private int _totalRows = -1` — Cached total row count (lazy loaded)

#### Constructors

##### File Path Constructor

* `public ExcelDataDocument(string filePath, Operation operation) : base(filePath, operation == Operation.Import)`

**Initialization Logic:**

1. Calls base constructor with read-only flag based on operation
2. Sets `operation` property and `headerRowCount` to 2
3. For Export operations, automatically calls `cleanExcelFile()`

##### Stream Constructor  

* `public ExcelDataDocument(System.IO.Stream fileStream, Operation operation) : base(fileStream, operation == Operation.Import)`

## Same initialization pattern as file path constructor

### Validation Methods

#### Primary Validation

* `public bool validateDocument()` — Comprehensive validation returning true or throwing AggregateException

**Validation Sequence:**

1. **Sheet Existence Check:** Validates all three required sheets exist
2. **Header Column Validation:** Ensures minimum column counts per sheet
3. **Row Count Validation:** Ensures minimum rows including headers
4. **Exception Aggregation:** Collects all validation errors into single AggregateException

**Validation Logic Pattern:**

```csharp
List<Exception> exceptions = new List<Exception>();

// Check sheet existence
if (!sheetNames.Contains(SHEET_NAME_TEXT))
{
    exceptions.Add(new SheetNotFoundException(SHEET_NAME_TEXT));
    isSheetTextExist = false;
}

// Check header columns only if sheet exists
if (isSheetTextExist && this.getRowValues(SHEET_NAME_TEXT, 2).Count < MIN_HEADER_COLUMNS_TEXT)
{
    exceptions.Add(new HeaderColumnsMissingException(SHEET_NAME_TEXT, MIN_HEADER_COLUMNS_TEXT, this.getRowValues(SHEET_NAME_TEXT, 2).Count));
}

// Throw aggregate exception if any errors found
if (exceptions.Count > 0)
{
    throw new AggregateException(exceptions);
}
```

#### Template Cleaning Methods

##### Excel File Cleaning

* `protected new void cleanExcelFile()` — Removes all data rows while preserving headers

**Cleaning Process:**

1. Calls base class `cleanExcelFile()` method
2. Removes rows beyond `headerRowCount` (2) from all three sheets
3. Uses LINQ with `ToList()` to avoid modification during enumeration

**Row Removal Pattern:**

```csharp
foreach (var row in this.getSheetData(SHEET_NAME_TEXT).Elements<Row>().Where(r => r.RowIndex.Value > this.headerRowCount).ToList())
{
    row.Remove();
}
```

#### Property Accessors

##### Operation Mode

* `public Operation operation { get; set; }` — Gets/sets current processing mode

##### Metrics Properties (Lazy Loaded)

###### Unique Document Count

* `public int uniqueDocumentCount { get; }` — Count of distinct document codes across all sheets

**Lazy Loading Pattern:**

```csharp
if (_uniqueDocumentCount == -1)
{
    _uniqueDocumentCount = GetSheetMetrics(DOCUMENT_CODE_COLUMN);
}
return _uniqueDocumentCount;
```

###### Unique Field Count  

* `public int uniqueFieldCount { get; }` — Count of distinct field names across all sheets

###### Total Row Count

* `public int totalRows { get; }` — Total data rows across all sheets (excluding headers)

**Calculation Logic:**

```csharp
_totalRows = Math.Max(
    this.getRowCount(SHEET_NAME_TEXT) + 
    this.getRowCount(SHEET_NAME_TABLE) + 
    this.getRowCount(SHEET_NAME_CHART) - 
    (3 * this.headerRowCount), 0);
```

#### Helper Methods

##### Sheet Metrics Calculator

* `private int GetSheetMetrics(string columnName)` — Calculates unique values in specified column across all sheets

**Processing Steps:**

1. **Cell Collection:** Gathers cells from all three sheets beyond header rows
2. **Column Filtering:** Filters cells matching the specified column (A or B)
3. **Value Extraction:** Extracts cell values using base class methods
4. **Uniqueness Calculation:** Counts distinct non-empty values

**Union Operation Pattern:**

```csharp
IEnumerable<Cell> TextSheetValues = this.getSheetData(SHEET_NAME_TEXT).Descendants<Cell>().Where(cell => getCellReferenceComponents(cell.CellReference.Value).Item2 > this.headerRowCount);
IEnumerable<Cell> TableSheetValues = this.getSheetData(SHEET_NAME_TABLE).Descendants<Cell>().Where(cell => getCellReferenceComponents(cell.CellReference.Value).Item2 > this.headerRowCount);
IEnumerable<Cell> ChartSheetValues = this.getSheetData(SHEET_NAME_CHART).Descendants<Cell>().Where(cell => getCellReferenceComponents(cell.CellReference.Value).Item2 > this.headerRowCount);
var cellValues = TextSheetValues.Union(TableSheetValues).Union(ChartSheetValues);
```

##### Cell Reference Parser

* `private Tuple<string,int> getCellReferenceComponents(string cellReference)` — Parses Excel cell references (e.g., "A1" → ("A", 1))

**Parsing Algorithm:**

```csharp
int startIndex = cellReference.IndexOfAny("0123456789".ToCharArray());
return new Tuple<string, int>(cellReference.Substring(0, startIndex), Int32.Parse(cellReference.Substring(startIndex)));
```

##### Legacy Method (Commented)

* `private void updateMetrics()` — Original metrics calculation method (appears to have Union operation bug)

**Identified Issue:** The original Union operations may not work correctly due to IEnumerable deferred execution patterns.

---

## Custom Exception Hierarchy

### `SheetException` - Base Sheet Exception

**Purpose:** Base class for all sheet-related exceptions with serialization support

#### Properties

* `public string sheetName { get; private set; }` — Name of the problematic sheet

#### Serialization Support

* `public override void GetObjectData(SerializationInfo info, StreamingContext context)` — Custom serialization implementation

### `SheetNotFoundException : SheetException` - Missing Sheet Exception

**Purpose:** Thrown when required sheet is not found in Excel file

### `HeaderColumnsMissingException : SheetException` - Column Count Exception  

**Purpose:** Thrown when sheet has insufficient header columns

#### Properties for Column Count Exception  

* `public int columnsExpected { get; private set; }` — Required column count
* `public int columnsFound { get; private set; }` — Actual column count found

### `RowsMissingException : SheetException` - Row Count Exception

**Purpose:** Thrown when sheet has insufficient rows

#### Properties for Row Count Excception

* `public int rowsExpected { get; private set; }` — Required row count
* `public int rowsFound { get; private set; }` — Actual row count found

---

## Business logic and integration patterns

### Document Field Data Exchange

The class is designed specifically for exchanging document field data:

**Three Data Types:**

* **Text Fields:** Simple text content and labels
* **Table Fields:** Structured tabular data (requires 6 columns minimum)  
* **Chart Fields:** Chart configuration and data (5 columns minimum)

**Standard Format Enforcement:**

* Column A: Document codes for grouping
* Column B: Field names for identification
* Additional columns: Field-specific data

### Import/Export Operations

Dual-mode processing supports different workflows:

**Import Mode:**

* Validates incoming Excel files for compliance
* Calculates metrics on data volume and diversity
* Read-only access to preserve source files

**Export Mode:**  

* Cleans template files by removing existing data
* Prepares templates for population with fresh data
* Write access for template modification

### Validation Strategy

Multi-layer validation approach:

**Structural Validation:**

* Sheet existence and naming compliance
* Column count requirements per sheet type
* Minimum row requirements

**Data Validation:**

* Cell reference parsing and validation
* Non-empty value detection
* Uniqueness calculations

### Template Management

Excel template processing workflow:

**Template Cleaning Process:**

1. Preserve header rows (first 2 rows)
2. Remove all data rows from all sheets
3. Maintain Excel formatting and structure
4. Prepare for fresh data population

---

## Technical implementation considerations

### OpenXML Direct Manipulation

Uses DocumentFormat.OpenXml for direct Excel file access:

* **Performance:** Faster than COM automation
* **Server Compatibility:** Works without Excel installation  
* **Precision:** Direct XML manipulation of Excel structure
* **Memory Efficiency:** Streaming access to large files

### Lazy Loading Pattern

Metrics properties use lazy loading for performance:

* **Deferred Calculation:** Only calculated when accessed
* **Caching:** Results cached after first calculation
* **Memory Efficiency:** Avoids unnecessary processing

### Exception Aggregation

Validation uses exception aggregation pattern:

* **Complete Validation:** All errors found in single pass
* **Detailed Reporting:** Each error provides specific context
* **User Experience:** Users see all issues at once, not one-by-one

### Cell Reference Processing

Custom cell reference parsing:

* **Column Extraction:** Separates letter-based column identifiers
* **Row Extraction:** Separates numeric row identifiers  
* **Performance:** Avoids regex for simple parsing task

---

## Integration with broader Publisher system

### Batch Processing Integration

Integrates with DocumentBatchImport and DocumentBatchExport:

* **Import Operations:** Validates and processes incoming Excel files
* **Export Operations:** Generates clean templates for data export
* **Error Handling:** Provides detailed validation errors for batch processing

### Document Field Management

Supports the document field architecture:

* **Field Types:** Text, Table, and Chart fields as separate entities
* **Document Grouping:** Uses document codes for organization
* **Field Identification:** Uses field names for unique identification

### Excel File Storage Integration

Works with Constants-defined paths:

* **EXCEL_TEMP_PATH:** Temporary processing location
* **EXCEL_EXPORT_TEMPLATE_PATH:** Template file location
* **File Management:** Handles both file paths and streams

### Validation Error Integration

Custom exceptions integrate with system error handling:

* **Detailed Messages:** Specific error context for troubleshooting
* **Serialization Support:** Errors can be logged and transmitted
* **Aggregation Support:** Multiple errors handled gracefully

---

## Potential enhancements

### Performance Optimizations

1. **Streaming Validation:** Process large files without loading entire structure
2. **Parallel Processing:** Calculate metrics for different sheets concurrently
3. **Memory Management:** Dispose resources more aggressively

### Validation Enhancements  

1. **Data Type Validation:** Validate column data types beyond existence
2. **Business Rule Validation:** Enforce document-specific validation rules
3. **Cross-Reference Validation:** Validate relationships between sheets

### Template Management Improvements

1. **Selective Cleaning:** Clean only specified ranges or sheets
2. **Template Preservation:** Backup original templates before modification
3. **Dynamic Templates:** Support variable template structures

### Error Handling Enhancements

1. **Warning System:** Support warnings in addition to errors
2. **Error Recovery:** Automatic correction of common issues
3. **Contextual Messages:** More user-friendly error descriptions

---

## Action items for system maintenance

1. **Union Operation Bug Fix:** Review and fix the potential Union operation issue in commented `updateMetrics()` method
2. **Performance Testing:** Test metrics calculation performance on large Excel files
3. **Validation Coverage:** Ensure all business rules are covered by validation
4. **Memory Usage Analysis:** Monitor memory usage with large files
5. **Exception Message Review:** Ensure error messages are user-friendly and actionable
6. **Template Compatibility:** Verify template cleaning works with different Excel versions

---

## Critical considerations

### Excel File Format Dependencies

The class assumes specific Excel file structure:

* **Sheet Names:** Hard-coded English sheet names may not work with localized Excel
* **Column Layout:** Assumes documents codes in column A, field names in column B
* **Header Rows:** Assumes exactly 2 header rows across all sheets

**Risk:** Changes to Excel file format could break validation and processing.

### Performance Scalability

Metrics calculation processes all cells across all sheets:

* **Large Files:** May have performance issues with very large Excel files
* **Memory Usage:** Loads all cell data into memory for processing
* **Concurrent Access:** No thread safety for simultaneous access

**Recommendation:** Monitor performance with realistic file sizes and consider streaming approaches for very large files.

### Error Handling Completeness

While validation is comprehensive, some edge cases may not be covered:

* **Corrupted Files:** OpenXML exceptions may not be properly handled
* **Network Issues:** Stream-based constructors may not handle network interruptions
* **Concurrent Access:** Multiple processes accessing same file could cause issues

**Recommendation:** Add comprehensive exception handling for all OpenXML operations.
