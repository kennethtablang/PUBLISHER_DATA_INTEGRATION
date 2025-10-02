# ExcelHelper — Component Analysis

## One-line purpose

Comprehensive Excel file validation engine that performs business rule validation, data integrity checks, and regulatory compliance verification for financial documents in the Publisher Data Integration pipeline.

---

## Files analyzed

* `Publisher_Data_Operations/Helper/ExcelHelper.cs`

---

## What this code contains (high level)

1. **Template-Driven Validation System** — Uses Excel template comments to define validation rules and applies them to input worksheets
2. **Financial Business Rule Engine** — Implements complex financial validation rules (MER/TER calculations, fund validation, calendar year requirements)
3. **Multi-Sheet Cross-Referencing** — Validates data consistency across multiple worksheets within a workbook
4. **Regulatory Compliance Checks** — Ensures documents meet specific financial regulatory requirements (22b tables, proforma handling)
5. **Data Quality Assurance** — Performs comprehensive data type, format, and content validation with detailed error reporting

This component serves as the primary validation orchestrator for Excel-based financial documents, ensuring data meets both technical and business requirements before Publisher processing.

---

## Classes, properties and methods (developer reference)

### `ExcelHelper` - Main Validation Orchestrator

**Purpose:** Manages comprehensive Excel file validation using template-driven rules, business logic, and cross-sheet referencing for financial document processing.

#### Core Properties

##### Validation State

* `bool IsValidData { get; private set; }` — Overall validation status (false if any validation fails)
* `Logger Log` — Error logging and tracking system
* `PDIStream ProcessStream { get; private set; }` — Stream context containing file and processing metadata

##### Database Integration

* `DBConnection dBConnection` — Database connection for validation queries and lookups
* `int FileID` — Global file identifier for validation context
* `ParameterValidator paramValidator` — XML and scenario validation utility

#### Constructor

* `ExcelHelper(PDIStream fileStream, object conn, Logger log = null)`
  * **Connection Handling:** Accepts DBConnection or connection parameters
  * **Context Setup:** Initializes FileID, validation state, and logging
  * **Parameter Validation:** Creates ParameterValidator if DocumentTypeID available
  * **Logger Management:** Uses provided logger or creates new instance

---

## Core Validation Methods

### Worksheet Structure Validation

#### Basic Worksheet Checks

* `bool WorksheetExist(Excel.Workbook workbook, string WSName)` — Verifies worksheet existence
* `void FindHiddenRowInWS(Excel.Worksheet worksheet)` — Detects hidden rows (validation rule 2AII6)
* `void FindHiddenColumnInWS(Excel.Worksheet worksheet)` — Detects hidden columns (validation rule 2AII6)
* `void LogHiddenSheetRowsColumns(Excel.Worksheet worksheet)` — Combined visibility validation
* `void LogFilters(Excel.Worksheet worksheet)` — Detects autofilters (validation rule 2AII5)

##### Formula and Content Validation

* `void LogFormulas(Excel.Worksheet worksheet)` — Identifies formulas in worksheets (validation rule 2AI2)
  * **Find Strategy:** Uses Aspose Find with formula-specific options
  * **Performance:** Efficient cell-by-cell scanning for formula detection

* `void CheckCellDataType(string cellFormat, Excel.Worksheet worksheet)` — Validates cell formatting (validation rule 2AII3)
  * **Performance Optimization:** Uses style-based searching (97% faster than iteration)
  * **Format Validation:** Ensures cells match expected text format
  * **Duplicate Prevention:** HashSet prevents duplicate error reporting

##### Header and Structure Validation

* `void ValidateWorkSheetHeader(WorksheetValidation wv)` — Template header comparison (validation rule 2AII2A/B)
  * **Cell-by-Cell Comparison:** Compares input headers against template
  * **Position Accuracy:** Validates exact cell positioning and content

---

### Business Rule Validation Methods

#### Fund Code and Cross-Sheet Validation

* `void MatchFundCodeToDocumentDataTab(WorksheetValidation mainWorksheet, WorksheetValidation worksheet)` — Fund code consistency (validation rule 2AII8)
* `void MatchFundCodesOfTwoSheets(WorksheetValidation firstWorksheet, WorksheetValidation secondWorksheet)` — Cross-sheet fund code validation (validation rule 3C)
* `void ValidateIsProforma(ValidationList validList)` — Proforma fund validation (validation rules 2AVI3/2AVI4)
  * **Business Logic:** Validates fund data presence based on proforma status
  * **Exception Handling:** Brand new funds exempt from certain requirements

##### Account and Document Validation

* `void ValidateAccountDocumentTypeAndLOB(WorksheetValidation worksheet)` — File parameter validation (validation rule 2AIII1)
  * **Template Verification:** Checks Publisher template availability
  * **Parameter Consistency:** Validates ClientCode, DocumentType, LineOfBusiness

##### Data Quality and Content Validation

* `void CheckBlanksAndNAInWorksheet(WorksheetValidation worksheet)` — Empty cell detection (validation rule 2AII7)
  * **Smart Row Detection:** Skips entirely blank rows to reduce noise
  * **N/A Value Handling:** Identifies unacceptable N/A entries

* `void CheckElementsInColumn(WorksheetValidation worksheet, int[] columnList, string[] elementList, ValidationType vt)` — Element validation (validation rules 2AVI2/2AIII6)
  * **Flexible Logic:** Supports both acceptable and unacceptable element lists
  * **String Matching:** Uses extension methods for word-based matching

---

### Advanced Financial Validation

#### Row Sequence Validation

* `void ValidateRowNumbers(WorksheetValidation worksheet, int[] columns)` — Row numbering validation (validation rule 2AVI1)
  * **Fund Grouping:** Maintains separate counters per fund code
  * **Sequential Logic:** Ensures continuous row numbering within fund groups

* `void ValidateRowWithDateNumbers(WorksheetValidation worksheet, int[] columns)` — Date sequence validation (validation rules 2AVI1B/C)
  * **Date Progression:** Validates chronological date ordering
  * **Combined Validation:** Row numbers and date sequences together

##### MER/TER Financial Calculations

* `void MerValidation(WorksheetValidation workSheet)` — Management expense ratio validation
* `void MerValidation(WorksheetValidation workSheet, int row, (Excel.Cell merPercentageCell, Excel.Cell terPercentageCell, Excel.Cell totalFundsPercentCell, Excel.Cell totalFundsAmountCell) cellMerTer, bool isSwitch = false)` — Detailed MER validation (validation rules 3G1I1A/B/C)
  * **Mathematical Precision:** Uses Math.Round to handle floating-point arithmetic
  * **Switching Logic:** Handles both regular and switch MER calculations
  * **Date-Based Requirements:** 60-day timespan threshold for validation requirements

##### Time-Based Validations

* `void Validate12Months(WorksheetValidation workSheet, List<int> columnList)` — Twelve-month validation (validation rules 3EI1/3EII1)
  * **Date Range Calculation:** Uses multiple date sources (PerformanceResetDate, InceptionDate, FirstOfferingDate)
  * **Performance Reset Integration:** Prioritizes PerformanceResetDate over inception dates
  * **Proforma Exclusion:** Skips validation for proforma records

* `void ValidateCalendarYears(WorksheetValidation workSheet, List<int> columnList)` — Calendar year validation (validation rules 3FI1/3FII1)
  * **22b Table Integration:** Validates historical return data tables
  * **Column Count Matching:** Ensures table columns match calculated calendar years
  * **Best/Worst Return Requirements:** Validates required return statistics

---

### Template-Driven Validation System

#### Column Discovery Methods

* `int FindColumnByName(WorksheetValidation workSheet, string columnName, Excel.LookAtType lookAt = Excel.LookAtType.Contains)` — Single column lookup
* `List<int> FindColumnsByName(WorksheetValidation workSheet, string columnName, Excel.LookAtType lookAt = Excel.LookAtType.Contains)` — Multiple column lookup
* `List<int> FindColumnsInList(WorksheetValidation workSheet, List<int> columnList, string[] findValues, Excel.LookAtType lookAt = Excel.LookAtType.Contains)` — Filtered column search
* `void FindColumnsInList(WorksheetValidation workSheet, List<int> columnList, Dictionary<string, int> findValues, Excel.LookAtType lookAt = Excel.LookAtType.Contains)` — Dictionary-based column mapping

##### Validation Type Handlers

* `void ValidateFlagRow(WorksheetValidation workSheet, List<int> columnList, string flagValue = "Flag")` — Flag-based conditional validation
* `void ValidateOneRequired(WorksheetValidation workSheet, List<int> columnList)` — At-least-one requirement validation (validation rule 3D)
* `void ValidateAllOrNone(WorksheetValidation workSheet, List<int> columnList)` — All-or-none validation (validation rule 3DII)

##### Content Validation Methods

* `void ValidateUniqueColumnContents(WorksheetValidation workSheet, List<int> foundColumns)` — Uniqueness validation (validation rule 2AV)
* `void ValidateXMLColumnContents(WorksheetValidation workSheet, List<int> foundColumns)` — XML content validation
* `void ValidateScenarioColumnContents(WorksheetValidation workSheet, List<int> foundColumns)` — Scenario identifier validation
* `void ValidateFieldAttributes(WorksheetValidation workSheet, List<int> foundColumns, bool restricted = true)` — Field attribute validation
* `void ValidateSheetName(WorksheetValidation workSheet, List<int> foundColumns)` — Sheet reference validation

---

### Template Processing and Orchestration

#### Main Validation Controller

* `void TemplateValidation(WorksheetValidation currentValidation, WorksheetValidation dataSheet = null)` — Primary template validation orchestrator
  * **Comment-Based Discovery:** Scans Excel comments for validation type instructions
  * **Enum-Driven Processing:** Maps comment text to ValidationType enum values
  * **Dynamic Column Discovery:** Identifies columns for each validation type
  * **Validation Routing:** Dispatches to appropriate validation methods

##### Data Type Validation

* `void ValidateByType(Excel.Worksheet worksheet, List<int> columnList, int startRow, ValidationType vt)` — Type-specific validation
  * **DateFormat:** Date parsing and format validation
  * **MaxTwoDecimals/MaxFourDecimals/MaxOneDecimal:** Decimal precision validation
  * **WholeNumbers:** Integer validation with special character handling
  * **RequiredValues:** Non-null requirement validation

##### Utility Methods

* `int CountHeaderRows(Excel.Worksheet templateSheet)` — Dynamic header row counting
* `List<int> LoadValidationFields(Excel.Worksheet workSheet, string validationName)` — Comment-based field discovery

---

### Error Reporting and Logging

#### Error Logging Integration

* `void LogFileValidationIssue(string errorMessage, Excel.Cell validationCell = null)` — Single cell error logging
* `void LogFileValidationIssue(string errorMessage, List<Excel.Cell> validationCellList)` — Multi-cell error logging
  * **Comment Integration:** Adds validation errors as Excel cell comments
  * **Comment Management:** Handles existing comments and formatting
  * **Validation State:** Updates IsValidData flag on errors

#### Database Integrations

* `void UpdateFileStatusAndDcoumentCountToLogTable(int dataID, int documentCount)` — File status persistence
* `bool IsBrandNewFund(string docCode, string fundCode, int? docTypeID, string filingID, int? clientID)` — Brand new fund determination
* `static bool IsBrandNewFund(DataTable dt, string filingID)` — DataTable-based fund analysis

---

### Date and Time Calculations

#### Date Calculation Utilities

* `bool TwelveMonthsDiff(List<DateTime> dates)` — Twelve-month span validation
* `int AgeInCalendarYears(List<DateTime> dates)` — Calendar year age calculation
* `double DaysDiff(List<DateTime> dates)` — Fractional day difference calculation
* `int YearsDiff(List<DateTime> dates)` — Year difference calculation

These methods support complex business rules requiring precise date calculations for financial regulatory compliance.

---

### Static Update Validation

* `void CheckStaticUpdate(Excel.Workbook workbook)` — Static table update validation
  * **Update Sheet Processing:** Validates "Table UPDATE" sheet instructions
  * **Cross-Reference Verification:** Ensures referenced sheets exist for updates

---

## Business logic and integration patterns

### Template-Driven Architecture

The ExcelHelper implements a sophisticated template-driven validation system:

* **Comment-Based Configuration:** Validation rules embedded in Excel template comments
* **Dynamic Rule Discovery:** Runtime parsing of validation instructions
* **Flexible Column Mapping:** Automatic column identification and mapping
* **Extensible Validation Types:** Enum-driven validation type system

### Financial Domain Integration

Specialized support for financial document requirements:

* **MER/TER Calculations:** Complex expense ratio validations with mathematical precision
* **Regulatory Compliance:** 22b table validation, proforma handling, calendar year requirements  
* **Fund Lifecycle Management:** Brand new fund detection and special handling
* **Cross-Sheet Consistency:** Fund code validation across multiple worksheets

### Performance Optimization Strategies

* **Style-Based Cell Scanning:** 97% performance improvement for format validation
* **HashSet Deduplication:** Prevents duplicate error reporting
* **Consecutive Range Optimization:** Efficient cell area searching
* **Smart Row Detection:** Skips blank rows to reduce processing overhead

### Error Context Preservation

* **Cell-Level Error Attribution:** Precise error location tracking
* **Comment Integration:** Validation errors embedded in Excel comments
* **Hierarchical Error Codes:** Structured error code system for regulatory traceability
* **Multi-Cell Error Groups:** Related validation failures grouped together

---

## Technical implementation considerations

### Memory and Performance

* **Large File Handling:** Optimized for files with 13,000+ rows (3.5s vs 138s processing time)
* **Style Pool Optimization:** Efficient workbook style processing
* **Find Operation Efficiency:** Aspose.Cells Find operations preferred over iteration
* **Connection Reuse:** Single database connection throughout validation lifecycle

### Data Type Handling

* **Floating Point Precision:** Math.Round usage for financial calculations
* **Cultural Formatting:** Proper handling of different date and number formats
* **Null Safety:** Comprehensive null checking throughout validation logic
* **Type Conversion Safety:** TryParse patterns with fallback handling

### Excel Integration Complexity

* **Aspose.Cells Mastery:** Advanced usage of Aspose.Cells API features
* **Cell Reference Management:** Proper handling of cell names and coordinates
* **Workbook State Management:** In-memory comment and formatting changes
* **Template Compatibility:** Support for various template formats and structures

### Error Resilience

* **Validation Continuation:** Individual validation failures don't stop overall processing
* **Exception Isolation:** Try-catch blocks prevent validation cascade failures  
* **Graceful Degradation:** Missing columns or templates handled appropriately
* **Database Failure Recovery:** Validation continues even with database connectivity issues

---

## Integration with broader PDI system

### Pipeline Integration Points

* **PDIStream Integration:** Deep integration with file processing streams
* **Database Schema Dependency:** Relies on pdi_Publisher_Documents and related tables
* **Logger Integration:** Comprehensive error logging with database persistence
* **AsposeLoader Coordination:** Works with AsposeLoader for data extraction

### Validation Rule Traceability

* **Azure DevOps Integration:** Validation rule numbers correspond to work items
* **Regulatory Mapping:** Error codes map to specific regulatory requirements
* **Documentation Integration:** Comment-based validation rules provide audit trail

### Template Management System

* **Template Versioning:** Support for different template versions per document type
* **Dynamic Rule Loading:** Runtime loading of validation rules from templates
* **Client-Specific Rules:** Support for client-specific validation requirements

### Publisher Integration Preparation

* **Data Quality Assurance:** Ensures data meets Publisher import requirements
* **Format Standardization:** Prepares data in Publisher-compatible formats
* **Error Prevention:** Catches issues before expensive Publisher processing

---

## Potential enhancements

### Performance Optimizations

1. **Parallel Validation:** Multi-threaded validation for large files
2. **Validation Caching:** Cache validation results for repeated patterns
3. **Incremental Validation:** Only validate changed sections in updated files
4. **Memory Streaming:** Process large files without full memory loading

### Validation Rule Enhancements

1. **Custom Rule Engine:** Pluggable validation rule system
2. **Machine Learning Integration:** Pattern-based validation anomaly detection
3. **Cross-File Validation:** Validation across multiple related files
4. **Real-Time Validation:** Live validation during file editing

### Error Reporting Improvements

1. **Interactive Error Resolution:** Guided error correction workflows
2. **Validation Summary Reports:** Executive summary of validation results
3. **Trend Analysis:** Historical validation error pattern analysis
4. **Integration with BI Tools:** Validation metrics in business intelligence dashboards

### Template System Evolution

1. **Visual Template Designer:** GUI-based template and rule creation
2. **Version Control Integration:** Template versioning with change tracking
3. **Rule Testing Framework:** Unit testing for validation rules
4. **Template Inheritance:** Base template with client-specific overrides

---

## Action items for system maintenance

1. **Performance Monitoring:** Track validation processing times and memory usage patterns
2. **Validation Rule Audit:** Regular review of validation rule effectiveness and accuracy
3. **Template Maintenance:** Keep templates synchronized with regulatory requirement changes
4. **Error Pattern Analysis:** Monitor common validation failures for process improvement opportunities
5. **Database Performance:** Monitor validation query performance and optimization opportunities
6. **Aspose.Cells Updates:** Stay current with Aspose.Cells API updates and optimizations
