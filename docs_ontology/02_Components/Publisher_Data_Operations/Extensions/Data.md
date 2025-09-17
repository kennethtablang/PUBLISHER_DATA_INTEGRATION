# Extensions — Data (analysis)

## One-line purpose

Provides extended DataTable and DataRow functionality with custom XML serialization, column/row manipulation utilities, validation helpers, and enhanced data access patterns for the PDI pipeline.

---

## Files analyzed

* `Publisher_Data_Operations/Extensions/Data.cs`

---

## What this code contains (high level)

1. **PDI_DataTable** — Custom DataTable subclass that handles extended properties and specialized row types
2. **PDI_DataRow** — Custom DataRow with dictionary-based extended properties for metadata storage
3. **Data static class** — Extension methods for DataTable/DataRow operations including:
   * Column lookup with partial/exact matching
   * Type-safe data extraction (string, int, bool, DateTime, FFDocAge)
   * XML serialization/deserialization with custom table format
   * HTML table generation
   * Data validation and cleaning utilities
   * CSV export functionality

This serves as the core data manipulation layer for transforming Excel/source data into the structured format required by the Publisher pipeline.

---

## Classes, properties and methods (developer reference)

### `PDI_DataTable : DataTable` - Class inheriting from `DataTable` class

**Purpose:** Enhanced DataTable that preserves extended properties during operations and uses custom PDI_DataRow instances.

#### Constructors

* `PDI_DataTable()` — Default constructor
* `PDI_DataTable(string tableName)` — Constructor with table name
* `PDI_DataTable(string tableName, string tableNamespace)` — Constructor with name and namespace
* `PDI_DataTable(DataTable dt)` — Copy constructor that preserves extended properties and converts to PDI format

#### Key Methods

* `protected override Type GetRowType()` — Returns typeof(PDI_DataRow) to ensure custom row type usage
* `protected override DataRow NewRowFromBuilder(DataRowBuilder builder)` — Creates PDI_DataRow instances

#### Notes / Behaviour

* Copy constructor handles special `Generic.FSMRFP_ROWTYPE_COLUMN` by moving it to row extended properties
* Preserves all column and table extended properties during conversion
* Essential for maintaining metadata throughout the transformation pipeline

---

### `PDI_DataRow : DataRow` - Class

**Purpose:** Enhanced DataRow with dictionary-based extended properties for storing row-level metadata.

#### Properties

* `Dictionary<string, object> ExtendedProperties { get; }` — Dictionary for storing custom row metadata

#### Methods

* `void SetAttribute(string name, object value)` — Adds key-value pair to extended properties
* `PDI_DataRow()` — Default constructor (base is null)
* `PDI_DataRow(DataRowBuilder builder)` — Standard constructor using DataRowBuilder

#### Usage Pattern

```csharp
PDI_DataRow row = (PDI_DataRow)table.NewRow();
row.SetAttribute("rowtype", "header");
row.ExtendedProperties.Add("category", "allocation");
```

---

### `Data` - Static Extension Class

**Purpose:** Comprehensive extension methods for DataTable/DataRow operations used throughout the PDI pipeline.

#### Column Access Methods

##### String Value Extraction

* `string GetPartialColumnStringValue(this DataRow dr, string column)` — Finds column by partial name match
* `string GetExactColumnStringValue(this DataRow dr, string column)` — Exact column name lookup with `<column>` fallback
* `string GetStringValue(this DataRow dr, int col)` — Safe string extraction from column index

##### Integer Value Extraction

* `int GetPartialColumnIntValue(this DataRow dr, string column)` — Partial name match for integers
* `int GetExactColumnIntValue(this DataRow dr, string column)` — Exact name match, returns -1 for null/missing
* `int GetIntValue(this DataRow dr, int col)` — Safe integer parsing from column index

##### Other Type Extractions

* `DateTime GetExactColumnDateValue(this DataRow dr, string column)` — DateTime extraction, returns DateTime.MinValue on failure
* `FFDocAge? GetExactColumnFFDocAge(this DataRow dr, string column)` — Custom enum extraction for document age status
* `bool GetPartialColumnBoolValue(this DataRow dr, string column)` — Boolean extraction with partial name matching
* `bool GetExactColumnBoolValue(this DataRow dr, string column)` — Boolean extraction with exact name matching
* `bool GetBoolValue(this DataRow dr, int col)` — Safe boolean parsing from column index

#### Column Discovery Methods

* `int FindDataRowColumn(this DataRow row, string column)` — Wrapper for table column finding
* `int FindDataTableColumn(this DataTable tbl, string column)` — Case-insensitive partial column name matching

#### Data Manipulation Methods

##### Change Tracking

* `IEnumerable<DataColumn> GetChangedColumns(this DataRow row)` — Returns columns that have changed values
* `IEnumerable<DataColumn> GetChangedColumns(this IEnumerable<DataRow> rows)` — Changed columns across multiple rows
* `IEnumerable<DataColumn> GetChangedColumns(this DataTable table)` — Changed columns in entire table
* `bool RowHasChanged(this DataRow row)` — Quick check if any column has changed

##### Table Cleanup

* `bool RemoveBlankColumns(this DataTable dt, int minColumns = 0)` — Removes completely empty columns
* `bool RemoveBlankColumnsAtEnd(this DataTable dt, bool sourceHasRowType, string formatColumnName)` — Removes trailing empty columns
* `bool RemoveBlankRows(this DataTable dt)` — Removes entirely empty rows
* `bool IsBlankRow(this PDI_DataRow dr)` — Checks if row contains only null/empty values

#### Validation Methods

* `bool ValidateXML(this DataTable dt, Helper.Logger log = null)` — Validates XML content in table using ParameterValidator

#### Export Methods

* `string ToCSV(this DataTable dtTable)` — Exports DataTable to CSV format with proper escaping
* `string DataTabletoHTML(this DataTable dt, bool addHeader, bool processColors = false)` — Generates HTML table with optional color coding

#### Dictionary Conversion Methods

* `Dictionary<string, string> GetDataRowDictionary(this DataRow dr, Dictionary<string, string> docFields = null)` — Converts row to dictionary, excluding specified fields
* `Dictionary<string, string> GetDataRowDictionaryLocal(this DataRow dr)` — Converts row to dictionary with proper DateTime formatting

#### Template Replacement Method

* `string ReplaceByDataRow(this string input, DataRow dr, Generic gen, bool isFrench = false, string naValue = "-")` — Replaces XML tokens in templates using row data with localization support

#### XML Serialization Methods

##### XML to DataTable

* `PDI_DataTable XMLtoDataTable(this string xmlString, string tableName = null)` — Parses custom XML table format into PDI_DataTable

##### DataTable to XML

* `string DataTabletoXML(this DataTable dt, bool removeNA = false)` — Converts DataTable to custom XML format (wrapper)
* `string DataTabletoXML(this PDI_DataTable dt, bool removeNA = false, bool preserveColumnNames = false)` — Main XML serialization method

##### Custom XML Format Structure

```xml
<table attribute="value">
  <row rowattribute="value">
    <cell columnattribute="value">Cell Content</cell>
    <cell />
  </row>
  <row />
</table>
```

#### Helper Methods

* `DataRow XMLtoDataRow(this DataTable dt, string rowString)` — Parses single XML row into DataRow
* `int FindHeaderColumn(this DataTable dt, string headerText, int headerRow = 0)` — Locates header column by text content
* `void ConsolePrintTable(this DataTable tbl)` — Debug utility for console table output

---

#### Key Integration Points

##### XML Token Replacement

* `ReplaceByDataRow` method integrates with `Generic.loadValidParameters()` for template processing
* Supports bilingual content (English/French) with proper date formatting
* Handles currency formatting and N/A value substitution

##### Custom XML Format

* Preserves extended properties at table, row, and column levels
* Used throughout the pipeline for data serialization between processing stages
* Maintains metadata that standard DataTable XML would lose

##### Type Safety

* All extraction methods handle null values and type conversion gracefully
* Returns sensible defaults (-1 for integers, empty string for text, DateTime.MinValue for dates)
* Extensive logging when column lookups fail with suggestions

##### Performance Considerations

* Column finding uses case-insensitive partial matching which may be expensive on large tables
* XML parsing uses regex which could impact performance on large documents
* Change tracking methods iterate through all columns/rows

---

##### Usage Patterns in Pipeline

1. **Extract Phase:** Raw Excel data loaded into PDI_DataTable with extended properties
2. **Transform Phase:** Data manipulation using safe extraction methods and template replacement
3. **Load Phase:** XML serialization for storage in staging tables
4. **Validation Phase:** XML validation and data quality checks

##### Action Items

* Verify the exact schema expected by `AppendToDataTable` callers
* Check performance impact of regex-based XML parsing on large documents
* Confirm thread safety requirements for dictionary operations in extended properties

---
