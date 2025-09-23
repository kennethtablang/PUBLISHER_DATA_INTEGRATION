# Extensions — Table (analysis)

## One-line purpose

Comprehensive table data structure and formatting system that handles multi-type financial data with bilingual support, specialized formatting rules, and XML output generation for Publisher document processing.

---

## Files analyzed

* `Publisher_Data_Operations/Extensions/Table.cs`

---

## What this code contains (high level)

1. **TableTypes Enumeration** — Defines different data formatting and processing types
2. **TableItem Class** — Individual table row item with value parsing, formatting, and bilingual content
3. **TableList Class** — Collection of table items with specialized processing methods for different business scenarios
4. **Multi-Format Support** — Handles various data types from percentages to distributions with cultural formatting
5. **XML Table Generation** — Converts structured data into Publisher-compatible XML table format

This system serves as the primary data structure for tabular content in financial documents, handling complex formatting requirements and regulatory compliance needs.

---

## Classes, properties and methods (developer reference)

### `TableTypes` - Enumeration

**Purpose:** Defines the various data types and processing methodologies for table content formatting.

#### Enumeration Values

* `Percent` — Percentage values with cultural formatting
* `Currency` — Monetary values with currency symbols
* `Number` — Numeric values with decimal formatting
* `Distribution` — Multi-column distribution data (fund distributions)
* `Pivot` — Pivot table format with years as columns
* `Date` — Date values with minimal formatting
* `MultiDecimal` — Multi-column decimal data
* `MultiText` — Multi-column text data
* `MultiPercent` — Multi-column percentage data
* `MultiYear` — Multi-column year-based data

**Usage Pattern:** Determines formatting logic and cell generation methods throughout the system.

---

### `TableItem` - Individual Row Data Class

**Purpose:** Represents a single table row with value parsing, multilingual content, and specialized formatting capabilities.

#### Core Properties

##### Content Properties

* `string EnglishText { get; set; }` — English description/label for the row
* `string FrenchText { get; set; }` — French description/label for the row
* `string ValueText { get; set; }` — Primary value with automatic decimal parsing
* `List<string> EnglishValueList { get; set; }` — Multi-value English data (distributions)
* `List<string> FrenchValueList { get; set; }` — Multi-value French data (distributions)

##### Metadata Properties

* `string RowLevel { get; set; }` — Hierarchical level for nested table structures
* `string Row { get; set; }` — Row identifier/number
* `string AllocationType { get; set; }` — Type classification for allocation tables
* `DateTime DistributionDate { get; set; }` — Date for distribution/temporal data
* `string MarkDate { get; set; }` — Marker for axis labels in charts

##### Computed Properties

* `int ValueScale { get; private set; }` — Automatic decimal scale detection
* `decimal? ValueDecimal { get; }` — Parsed decimal value (null if non-numeric)
* `int RowInt { get; }` — Row number as integer (-1 if invalid)
* `int RowLevelInt { get; }` — Row level as integer (-1 if invalid)

#### Constructors

##### Primary Constructor

* `TableItem(string value, string english = null, string french = null, string rownumber = null, string rowlevel = null, string allocationOrDate = null)`
  * **Value Processing:** Trims and parses value for decimal conversion
  * **Date Detection:** Attempts to parse `allocationOrDate` as "MM/yyyy" format
  * **Allocation Fallback:** Uses `allocationOrDate` as allocation type if date parsing fails

##### Multi-Value Constructors

* `TableItem(List<string> valueListEN, string english = null, string french = null, string rownumber = null)`
* `TableItem(List<string> valueListEN, List<string> valueListFR, string english = null, string french = null, string rownumber = null)`
  * **Scale Calculation:** Automatically determines maximum scale from value lists
  * **Bilingual Support:** Handles paired English/French value collections

#### Core Methods

##### Value Management

* `void AddToList(string valueEN, string valueFR = null)` — Adds values to existing lists
  * **Dynamic Scale Update:** Recalculates scale when new values added
  * **List Management:** Handles both English-only and bilingual additions

##### Content Validation

* `bool IsRowLevel()` — Checks if item has valid row level
* `bool IsAllocationType()` — Checks if item has allocation type classification

##### Cell Generation Methods

###### Basic Cell Formatting

* `string GetCellValue(int scale, string append = "%")` — Formats value as XML cell
  * **Decimal Formatting:** Uses specified scale for numeric precision
  * **Row Level Handling:** Adds empty cells for hierarchical structure
  * **Default Append:** Adds percentage symbol by default

###### Type-Specific Cell Formatting

* `string GetCellValue(int scale, TableTypes tableType, bool getFrench)` — Advanced type-based formatting

**Type-Specific Processing:**

* **TableTypes.Percent:** Cultural percentage formatting with proper symbols
* **TableTypes.Currency:** Currency formatting with cultural symbols
* **TableTypes.Number:** Plain numeric formatting with specified scale
* **TableTypes.Distribution:** Multi-column decimal formatting without symbols
* **TableTypes.MultiText/MultiPercent/MultiYear:** Complex multi-cell processing
* **TableTypes.Date:** Simple text formatting

###### Text Cell Generation

* `string GetCellText(TableTypes tableType, bool getFrench = false)` — Generates text cells
* `string GetCellText(Dictionary<string, string[]> shortMonths, TableTypes tableType, bool getFrench = false)` — Date-aware text with month formatting

#### Advanced Processing Methods

##### Private Cell Builders

* `string makeCell(string value)` — Basic XML cell wrapper with null handling
* `string makeCell(int scale, string append = "", bool getFrench = false)` — Multi-value cell generation from value lists
* `string makeMultiCell(int scale, bool getFrench = false, TableTypes tableType = TableTypes.MultiText)` — Complex multi-type cell processing

**Multi-Cell Processing Logic:**

1. **Date Detection:** Processes dates first, extracting years for MultiYear type
2. **Numeric Processing:** Handles percentages and decimals with proper formatting
3. **Text Processing:** Trims quotes and handles non-numeric content
4. **Cultural Formatting:** Applies French/English formatting rules

---

### `TableList : List<TableItem>` - Table Collection Class

**Purpose:** Manages collections of table items with specialized business logic for different table types and financial reporting requirements.

#### Properties

* `int FilingYear { get; set; }` — Filing year for 22b-style tables (private)
* `TableTypes TableType { get; set; }` — Determines processing and formatting behavior
* `Dictionary<string, string[]> ShortMonths { get; set; }` — Month name translations for date formatting
* `string NAValue { get; set; }` — Replacement value for N/A or blank entries

#### Constructor

##### Specialized Constructors

* `TableList()` — Default percentage table
* `TableList(int filingYear)` — 22b table with filing year context
* `TableList(int filingYear, string naValue)` — 22b pivot table with N/A handling
* `TableList(TableTypes valueType)` — Type-specific table
* `TableList(string naValue)` — Distribution table with N/A replacement

#### Data Management Methods

##### Item Addition

* `bool AddValidation(string value, string english = null, string french = null, string rownumber = null, string rowlevel = null, string allocation = null)` — Primary item addition with validation

**Processing Logic by Type:**

* **Distribution:** Finds existing item by English text and row, adds to value list
* **Pivot:** Adds with N/A replacement handling
* **Default:** Skips N/A or blank values entirely

* `bool AddValidationDistrib(string valueEN, string valueFR, string english = null, string french = null, string rownumber = null)` — Bilingual distribution addition
* `void AddMultiCell(string rownumber, string valueEN, string valueFR, string english = null, string french = null)` — Multi-cell data addition

#### Processing and Analysis Methods

##### Scale Management

* `int GetMaxScale()` — Determines maximum decimal precision across all items
  * **Type-Specific Limits:** Distribution (3), MultiText (4), others (2)
  * **Performance:** Prevents excessive decimal precision

##### Date Processing

* `bool MarkByDate(DateTime date, string mark = "Y")` — Marks specific distribution dates for axis labels
* `bool MarkDates()` — Intelligent axis label marking for 10K tables

**Axis Label Algorithm:**

1. Always marks first and last dates
2. For longer periods, marks December dates only (yearly intervals)
3. Calculates optimal spacing using gap analysis
4. Limited to 10 total axis labels for Publisher compatibility

#### Table Generation Methods

##### Specialized Table Formats

###### 22b Table Generation

* `string GetTableString(out int calendarYears, out int negativeYears)` — Generates 22b regulatory table
  * **Calendar Year Calculation:** Counts total and negative return years
  * **Filing Year Context:** Uses filing year to calculate historical years
  * **Regulatory Compliance:** Specific format for financial regulatory reporting

###### Pivot Table Generation

* `string GetTablePivotString()` — Horizontal pivot table with years as columns
  * **Structure:** First row contains years, second row contains values
  * **Scale Management:** Minimum scale of 1 for pivot tables

###### Standard Table Generation

* `string GetTableString(bool getFrench = false)` — Primary table generation method
  * **Sorting Logic:** Numbers tables sorted by row level then row number
  * **Date Marking:** Automatically applies axis label marking
  * **Cell Integration:** Combines text, value, and mark cells

* `string GetTableStringFrench()` — Convenience wrapper for French table generation

---

## Business logic and integration patterns

### Regulatory Compliance Integration

The system handles specific regulatory reporting requirements:

* **22b Tables:** Historical return data with negative year counting
* **Filing Year Logic:** Proper year boundary handling for compliance periods
* **Calendar Year Calculations:** Specific algorithms for age and return period determination

### Multilingual Financial Reporting

Comprehensive bilingual support throughout:

* **Cultural Formatting:** French-Canadian currency and percentage patterns
* **Month Names:** Short month translation system for date display
* **N/A Handling:** Consistent null value replacement across languages

### Distribution Table Processing

Specialized handling for fund distribution data:

* **Multi-Column Data:** Supports multiple distribution periods per row
* **Scale Preservation:** Maintains precision across all distribution values
* **Date-Based Sorting:** Temporal organization of distribution information

### Chart Integration

Axis label marking system for Publisher chart generation:

* **Intelligent Marking:** Selects optimal dates for chart axes
* **Publisher Limits:** Respects 10-label limit for chart compatibility
* **Temporal Distribution:** Ensures even distribution of labels across time periods

---

## Technical implementation considerations

### Performance Characteristics

* **Scale Calculation:** O(n) operation across all items when adding values
* **Sorting Logic:** Built-in List sorting for row level and number organization
* **Memory Usage:** Efficient with list-based value storage for distributions

### Data Type Handling

* **Automatic Parsing:** Decimal conversion with fallback to text
* **Scale Preservation:** Maintains original decimal precision
* **Cultural Formatting:** Proper numeric formatting per locale

### XML Generation Strategy

* **StringBuilder Usage:** Efficient string building for table generation
* **Cell Wrapping:** Consistent XML cell structure throughout
* **Null Handling:** Safe processing of missing or null values

---

## Integration with broader PDI system

### AsposeLoader Integration

TableList objects are commonly created and populated by AsposeLoader methods:

* `AddGenericTable` creates and populates TableList instances
* `BuildAllocationTable` uses TableList for allocation processing
* Short months dictionary integration for date formatting

### Transform Pipeline Integration

TableList serves as intermediate format between extraction and final output:

* Raw Excel data → TableList → XML table format → Publisher import
* Handles data cleaning and validation during transformation
* Provides consistent formatting across document types

### Template System Integration

Generated XML integrates with template parameter replacement:

* Table XML can contain template tokens
* Cultural formatting supports template localization
* N/A value replacement works with template parameter system

---

## Potential enhancements

### Performance Optimizations

1. **Scale Caching:** Cache max scale calculation to avoid repeated computation
2. **Sorted Insertion:** Maintain sort order during addition instead of sorting afterward
3. **String Pooling:** Consider string interning for commonly used values

### Functionality Extensions

1. **Additional TableTypes:** Support for new financial data types as requirements evolve
2. **Custom Formatting Rules:** Configurable formatting patterns per client
3. **Validation Enhancement:** More sophisticated data validation during addition
4. **Chart Integration:** Direct chart data export formats

### Maintainability Improvements

1. **Method Decomposition:** Break down large methods like `GetCellValue` into focused helpers
2. **Configuration Externalization:** Move hard-coded limits and patterns to configuration
3. **Error Handling:** More robust error handling for edge cases
4. **Unit Testing:** Comprehensive test coverage for formatting edge cases

---

## Action items for system maintenance

1. **Performance Monitoring:** Monitor scale calculation performance on large datasets
2. **Formatting Validation:** Ensure cultural formatting meets regulatory requirements
3. **Date Logic Testing:** Verify axis marking algorithm works across various time periods
4. **Memory Usage Analysis:** Review memory usage patterns for distribution tables
5. **Business Rule Documentation:** Document the complex formatting rules for future maintenance
