# Helper — Worksheet Validation (analysis)

## One-line purpose

Excel worksheet validation framework that provides type-safe sheet identification, validation rule orchestration, and data quality enforcement for financial document processing workflows.

---

## Files analyzed

* `Publisher_Data_Operations/Helper/WorksheetValidation.cs`

---

## What this code contains (high level)

1. **ValidationType Enumeration** — Comprehensive validation rule definitions for financial data validation
2. **SheetType Enumeration** — Type-safe identification system for different worksheet categories and regulatory forms
3. **WorksheetValidation Class** — Container for worksheet pairs (input + template) with metadata and type detection
4. **ValidationList Class** — Orchestration engine that manages collections of worksheets and executes validation workflows

This system serves as the primary validation framework for Excel-based financial document processing, ensuring data quality, regulatory compliance, and proper sheet categorization before content transformation.

---

## Classes, properties and methods (developer reference)

### `ValidationType` - Validation Rule Enumeration

**Purpose:** Defines the complete set of validation rules applied to financial worksheets during processing.

#### Core Data Validation Types

* `DateFormat` — Validates date formatting consistency
* `MaxTwoDecimals` — Enforces decimal precision limits (2 places)
* `MaxOneDecimal` — Enforces decimal precision limits (1 place)  
* `MaxFourDecimals` — Enforces decimal precision limits (4 places)
* `WholeNumbers` — Ensures integer-only values
* `RequiredValues` — Validates mandatory field completion

#### Content Validation Types

* `UnacceptableElements` — Flags prohibited content or values
* `AcceptableElements` — Validates against allowed value sets
* `UniqueColumnContents` — Ensures column value uniqueness
* `XMLValidation` — Validates XML structure and format

#### Sequence and Structure Validation

* `RowSequenceCheck` — Validates row ordering and sequence integrity
* `RowWithDateSequenceCheck` — Date-aware sequence validation
* `AtLeastOneRequired` — Ensures minimum data presence
* `AllOrNone` — Enforces complete data set requirements

#### Financial-Specific Validation

* `FundAndInvestmentCheck` — Fund code and investment data consistency
* `InvestmentMix17` — Investment allocation validation (Schedule 17)
* `InvestmentMix40` — Investment allocation validation (Schedule 40)
* `TwelveMonthValidation` — 12-month period data validation
* `CalendarYearValidation` — Calendar year boundary validation
* `FlagValueValidation` — Binary flag field validation
* `ScenarioValidation` — Scenario-based data validation

#### Metadata and Structure Validation

* `FieldAttributeAllCheck` — Comprehensive field attribute validation
* `FieldAttributeCheck` — Individual field attribute validation
* `SheetNameCheck` — Worksheet naming convention validation
* `ActiveFilter` — Filter state validation
* `MatchingDataSheetRequired` — Cross-sheet data dependency validation

**Usage Pattern:** Template-driven validation where validation types are specified in template sheets to control validation behavior per worksheet type.

---

### `SheetType` - Worksheet Classification Enumeration

**Purpose:** Provides type-safe identification of worksheet categories based on regulatory forms, business functions, and processing requirements.

#### Primary Data Sheets

* `wsData` — Main document data sheet (DocumentData)
* `wsBNY` — Bank of New York specific data format

#### Schedule Forms (Regulatory)

* `ws16` — Schedule 16 (Investment portfolio data)
* `ws17` — Schedule 17 (Investment mix details)
* `ws40` — Schedule 40 (Investment allocation data)
* `ws10K` — 10K historical data
* `wsAllocation` — Fund allocation worksheets
* `wsDistribution` — Distribution data
* `wsNavPU` — Net Asset Value Per Unit

#### Fund Management Sheets

* `wsTheFund` — Fund details (Schedule 38)
* `wsTerminated` — Terminated funds (Schedule 94)
* `wsNewSeries` — New series data (Schedule 93)
* `wsNewFunds` — New fund data (SC93)
* `wsNoLongerOffered` — Discontinued offerings (SC94)

#### Fee and Commission Sheets

* `wsManagmentFees` — Management fee data (F42/M9a)
* `wsFixedAdminFees` — Fixed administration fees (Schedule 39)
* `wsBrokerageCommissions` — Brokerage commission data (F40)
* `wsSoftDollarComissions` — Soft dollar commissions (Schedule 27b)

#### Investment and Transfer Sheets

* `wsInvestmentsFund` — Investment fund data (Schedule 50)
* `wsSeriesRedesignation` — Series redesignation (Schedule 96)
* `wsSeriesMerger` — Series merger data (Schedule 95)
* `wsFundMerger` — Fund merger data (Schedule 92)
* `wsFundTransfer` — Fund transfer data (Schedule 123)

#### Financial Reporting Sheets

* `wsRemainingYears` — Remaining years data (Schedule 41)
* `wsIncomeTaxes` — Income tax data (Schedule 118)
* `wsFairValue` — Fair value data (Schedule 21)
* `wsComparisonOfNetAsset` — Net asset comparison (Schedule 150)
* `wsYearByYear` — Year-by-year performance (Schedule 15)
* `wsAnnualCompoundReturns` — Annual compound returns (M17)

#### Specialized Financial Sheets

* `wsSeedMoney` — Seed money data (163ca)
* `wsSubsidiarySummary` — Subsidiary summary (163da)
* `wsSubsidiaryDetail` — Subsidiary detail (Schedule 120)
* `wsCommitments` — Commitment data (Schedule 121)
* `wsUnfundedLoanCommitments` — Unfunded loan commitments (Schedule 90)

#### Administrative Sheets

* `wsFullName` — Full name data (Schedule 122)
* `wsNewlyOffered` — Newly offered data (Schedule 147)
* `wsChangeFundName` — Fund name changes (Schedule 91)
* `wsManagementPersonnel` — Management personnel data
* `wsKeyManagementPersonnelBrokerageCommissions` — Key management personnel brokerage (SC167/SC167b)
* `wsFundMergers` — Fund mergers data (Schedule 168)

#### System Configuration Sheets

* `wsBookFundMap` — Book to fund mapping
* `wsFundtoBookTableMap` — Fund to book table mapping
* `wsFieldUpdate` — Field update configurations
* `wsTableUpdate` — Table update configurations
* `wsClientTranslation` — Client-specific translations
* `wsStaticLanguage` — Static language content

#### Fallback

* `wsUnknown` — Unrecognized sheet type

**Usage Pattern:** Automatic sheet type detection drives validation rules, processing logic, and transformation behavior throughout the system.

---

### `WorksheetValidation` - Worksheet Container Class

**Purpose:** Encapsulates an input worksheet with its corresponding template, providing metadata, type identification, and data boundary detection.

#### Core Properties

##### Worksheet References

* `Excel.Worksheet InputSheet { get; set; }` — Source data worksheet from uploaded file
* `Excel.Worksheet TemplateSheet { get; set; }` — Corresponding template worksheet with validation rules
* `int FirstDataRow { get; set; }` — First row containing actual data (after headers)

##### Computed Properties

* `int DataRows { get; }` — Total number of data rows (MaxDataRow - FirstDataRow + 1)
* `SheetType TypeOfSheet { get; set; }` — Automatically determined sheet type

#### Constructor

* `WorksheetValidation(Excel.Worksheet inputSheet, Excel.Worksheet templateSheet, int firstDataRow)`
  * **Sheet Assignment:** Links input and template worksheets
  * **Data Boundary Setting:** Establishes first data row for processing
  * **Automatic Type Detection:** Calls `GetSheetType()` to classify the worksheet

#### Static Methods

##### Sheet Type Detection

* `static SheetType GetSheetType(string sheetName)` — Primary sheet classification logic

**Detection Algorithm:**

1. **Case-Insensitive Pattern Matching:** Uses `IndexOf` with `StringComparison.OrdinalIgnoreCase`
2. **Priority-Based Matching:** Specific patterns checked before generic ones
3. **Numeric Code Detection:** Recognizes regulatory form numbers (16, 17, 40, etc.)
4. **Special Case Handling:** Distinguishes between similar codes (F40 vs FF40)

**Key Pattern Examples:**

* `"Document"` → `wsData`
* `"BNY_Data"` → `wsBNY`
* `"Allocation"` → `wsAllocation`
* `"16"` → `ws16`
* `"F40"` (exact match) → `wsBrokerageCommissions`
* `"150"` (checked before "50") → `wsComparisonOfNetAsset`

##### Header Row Configuration

* `static int DefaultHeaderRows(SheetType sheetType)` — Returns default header row count per sheet type
  * **wsData, wsNavPU:** 2 header rows
  * **All others:** 1 header row

**Usage Pattern:** Used by `ExcelHelper.CountHeaderRows()` as fallback when template doesn't specify header count.

---

### `ValidationList : List<WorksheetValidation>` - Validation Orchestration Class

**Purpose:** Manages collections of WorksheetValidation objects and orchestrates the complete validation workflow for financial document processing.

#### Properties

##### Core Dependencies

* `ExcelHelper ExHelper { get; set; }` — Primary validation engine instance
* `PDIStream ProcessStream { get; private set; }` — Processing stream context from ExcelHelper

##### Computed Property

* `int TotalDocuments { get; }` — Document count from wsData sheet
  * **Error Handling:** Returns -1 if wsData sheet not found or calculation fails
  * **Data Source:** Uses `DataRows` property from wsData sheet

#### Constructors

* `ValidationList(ExcelHelper ex)`
  * **Dependency Injection:** Requires ExcelHelper instance for validation operations
  * **Stream Context:** Captures ProcessStream from ExcelHelper for document context

#### Collection Management Methods

##### Worksheet Addition

* `void AddWS(Excel.Worksheet inputSheet, Excel.Worksheet templateSheet)`
  * **Header Row Detection:** Uses `ExHelper.CountHeaderRows(templateSheet)` for first data row
  * **Automatic Classification:** Creates WorksheetValidation with automatic type detection
  * **Collection Addition:** Adds to internal list for batch processing

##### Sheet Lookup Operations

* `bool ContainsSheetByName(string inputSheetName)` — Linear search for sheet by name
* `WorksheetValidation GetSheetByType(SheetType st)` — Returns first sheet matching type (LINQ FirstOrDefault)

#### Primary Validation Orchestration

##### Main Validation Method

* `bool Validate()` — Executes complete validation workflow

**Validation Workflow:**

###### 1. Prerequisite Validation

* **Null Check:** Ensures ExHelper is not null
* **Document Data Validation:** For BAU files, ensures wsData sheet exists

###### 2. Universal Sheet Validation (All Sheets)

For non-BNY data types:

* **Formula Detection:** `LogFormulas()` - identifies and logs Excel formulas
* **Data Type Validation:** `CheckCellDataType("@")` - enforces text formatting (slow operation)
* **Blank/NA Detection:** `CheckBlanksAndNAInWorksheet()` - identifies data quality issues

For all sheet types:

* **Header Validation:** `ValidateWorkSheetHeader()` - validates column sequence
* **Hidden Content Logging:** `LogHiddenSheetRowsColumns()` - logs hidden rows/columns
* **Template-Driven Validation:** `TemplateValidation()` - executes template-specified validation rules

###### 3. Sheet-Specific Validation

**wsData (Document Data):**

* **Account Validation:** `ValidateAccountDocumentTypeAndLOB()` - verifies client, document type, line of business
* **MER Validation:** For FF/ETF BAU files, executes `MerValidation()`

**ws16, ws17, ws40 (Investment Schedules):**

* **Fund Code Matching:** `MatchFundCodeToDocumentDataTab()` - ensures consistency with wsData
* **Required Fund Validation:** Executes type-specific validation:
  * ws16: `ValidationType.FundAndInvestmentCheck`
  * ws17: `ValidationType.InvestmentMix17`
  * ws40: `ValidationType.InvestmentMix40`

###### 4. Cross-Sheet Validation

* **Fund Code Consistency:** If both ws16 and ws17 exist, validates fund code alignment
* **Proforma Validation:** `ValidateIsProforma()` - validates proforma data consistency

##### Return Value

* **Success Indicator:** Returns `ExHelper.IsValidData` - overall validation result

**Error Handling:** Throws `NullReferenceException` for missing critical components (ExHelper, wsData for BAU files).

---

## Business logic and integration patterns

### Template-Driven Validation Architecture

The system implements a sophisticated template-driven validation approach:

* **Template Specification:** Validation rules defined in template worksheets
* **Dynamic Rule Application:** ValidationType enum values drive validation logic
* **Sheet-Specific Rules:** Different validation sets applied based on SheetType
* **Extensible Framework:** New validation types can be added without code changes

### Regulatory Compliance Integration

Comprehensive support for financial regulatory requirements:

* **Schedule Identification:** Automatic recognition of regulatory form numbers
* **Cross-Reference Validation:** Fund codes validated across related schedules
* **Data Quality Enforcement:** Multiple validation layers ensure compliance-ready data
* **Document Type Awareness:** Validation rules adapt based on document type (FF, ETF, etc.)

### Multi-Format Support

Handles various Excel file formats and structures:

* **BAU Processing:** Business-as-usual document processing with full validation
* **BNY Integration:** Special handling for Bank of New York data format (allows formulas)
* **Template Matching:** Pairs input sheets with corresponding template sheets
* **Header Row Flexibility:** Adapts to different header row configurations per sheet type

### Data Quality Assurance

Multi-layered approach to data quality:

* **Formula Detection:** Identifies unexpected Excel formulas in data
* **Data Type Enforcement:** Ensures proper text formatting across cells
* **Blank/NA Detection:** Identifies missing or placeholder data
* **Hidden Content Logging:** Documents hidden rows/columns that may affect processing

---

## Technical implementation considerations

### Performance Characteristics

* **Sequential Processing:** Validates sheets one at a time for memory efficiency
* **Slow Operations:** `CheckCellDataType()` identified as performance bottleneck
* **LINQ Usage:** Efficient sheet lookup using FirstOrDefault
* **Linear Search:** Simple name-based sheet searches for small collections

### Memory Management

* **Worksheet References:** Holds references to Aspose.Cells worksheets
* **Template Caching:** Templates loaded once per validation session
* **Stream Context:** Maintains processing context without duplication

### Error Handling Strategy

* **Null Reference Protection:** Explicit null checks for critical dependencies
* **Graceful Degradation:** Returns -1 for document count calculation failures
* **Exception Propagation:** Throws exceptions for critical validation failures
* **Validation State Tracking:** Uses ExHelper.IsValidData for overall validation status

### Integration Points

* **ExcelHelper Dependency:** Primary validation logic delegated to ExcelHelper
* **PDIStream Context:** Accesses document type and processing metadata
* **Aspose.Cells Integration:** Works directly with Aspose Excel object model
* **Template System Integration:** Coordinates with template-driven validation framework

---

## Integration with broader PDI system

### AsposeLoader Integration

WorksheetValidation supports the Excel processing pipeline:

* ValidationList created during Excel file processing
* Sheet-template pairing during file loading
* Validation execution before content transformation
* Type detection drives processing logic selection

### Template System Integration

Coordinates with the template validation framework:

* Template sheets provide validation rule specifications
* ValidationType enumeration drives template-based validation
* Header row detection uses template metadata
* Validation results inform template parameter processing

### Document Processing Pipeline Integration

Serves as validation gate in the processing workflow:

* Excel file → ValidationList creation → Validation execution → Content transformation
* Validation failures prevent downstream processing
* Sheet type detection drives transformation logic
* Data quality metrics inform processing decisions

### Error Reporting Integration

Validation results integrate with broader error reporting:

* ExHelper aggregates validation messages
* Sheet-specific errors logged with context
* Cross-sheet validation failures documented
* Processing stream updated with validation status

---

## Potential enhancements

### Performance Optimizations

1. **Parallel Validation:** Process independent sheets concurrently
2. **Validation Caching:** Cache validation results for identical templates
3. **Selective Validation:** Skip expensive checks based on data type
4. **Batch Operations:** Group similar validation operations

### Functionality Extensions

1. **Custom Validation Rules:** Support for client-specific validation logic
2. **Validation Profiles:** Different validation sets per client or document type
3. **Interactive Validation:** Real-time validation feedback during data entry
4. **Validation Reporting:** Detailed validation reports with correction suggestions

### Maintainability Improvements

1. **Sheet Type Configuration:** Externalize sheet name patterns to configuration
2. **Validation Rule Engine:** More sophisticated rule definition and execution
3. **Error Message Enhancement:** More descriptive validation error messages
4. **Validation Metrics:** Performance and accuracy metrics for validation operations

### Integration Enhancements

1. **Database Integration:** Store validation rules and results in database
2. **API Integration:** Expose validation as web service for external systems
3. **Workflow Integration:** Integration with document workflow systems
4. **Audit Trail:** Comprehensive audit trail for validation decisions

---

## Action items for system maintenance

1. **Performance Monitoring:** Monitor CheckCellDataType performance on large files
2. **Sheet Type Coverage:** Regularly review and update sheet type patterns
3. **Validation Rule Testing:** Comprehensive testing of all ValidationType combinations
4. **Template Synchronization:** Ensure validation rules stay synchronized with templates
5. **Error Handling Review:** Review exception handling for edge cases
6. **Documentation Updates:** Keep sheet type documentation current with regulatory changes
