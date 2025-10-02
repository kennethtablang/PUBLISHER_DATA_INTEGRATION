# Tests — Helper Merge Tables Tests (analysis)

## One-line purpose

Comprehensive test suite for MergeTables and MergeTablesSettings classes validating complex financial table merging functionality including historical data integration, interim period handling, column management, and regulatory table format processing for multi-period financial document generation.

---

## Files analyzed

* `Publisher_Data_Operations_Tests/Helper/MergeTablesTests.cs`

---

## What this code contains (high level)

1. **Configuration Management Testing** — Validation of MergeTablesSettings class for table merge configuration with row/column specifications and interim period handling
2. **Historical Data Integration** — Testing complex merging of current financial data with historical records for multi-period reporting
3. **Sophisticated Test Scenarios** — Extensive real-world financial table merging scenarios including HSBC fund data, ratios, and supplemental data
4. **Interim Period Processing** — Advanced handling of interim vs annual reporting periods with automatic column adjustment and data positioning
5. **Regulatory Table Format Support** — Testing specific financial regulatory table formats (M11a, M12a, M15i, M15m) with proper column management

This test suite validates the most complex table processing functionality in the system, enabling sophisticated multi-period financial reporting with historical data preservation and regulatory compliance.

---

## Classes, properties and methods (developer reference)

### `MergeTablesSettingsTests` - Configuration Test Class

**Purpose:** Validates MergeTablesSettings class functionality for configuring table merge operations.

#### Test Infrastructure Setup

##### Constructor and Configuration

```csharp
public MergeTablesSettingsTests()
{
    _checkRow = 1457362;
    _checkCol = 1518743494;
    _desiredRows = 62023840;
    _desiredColumns = 1089103889;
    _interimIndicator = "Something";
    _interimOffset = -1;
    _testClass = new MergeTablesSettings(_checkRow, _checkCol, _desiredRows, _desiredColumns, _interimIndicator, _interimOffset);
}
```

**Configuration Parameters:**

* **CheckRow/CheckCol:** Large integers (1457362, 1518743494) for validation positions
* **DesiredRows/DesiredColumns:** Target dimensions (62023840, 1089103889) for merged tables
* **InterimIndicator:** String identifier ("Something") for interim period detection
* **InterimOffset:** Integer (-1) for positioning adjustments

#### Property Management Tests

**CheckField Property:**

* **Tuple<int, int> Structure:** Encapsulates row/column pair for validation checks
* **Runtime Modification:** Supports dynamic configuration changes during processing

**Dimensional Configuration:**

* **DesiredRows/DesiredColumns:** Target table dimensions after merging
* **Initialization Validation:** Constructor values properly assigned to properties
* **Runtime Updates:** Properties support modification after instantiation

**Interim Processing Configuration:**

* **InterimIndicator:** String pattern for detecting interim periods
* **InterimOffset:** Numeric offset for interim period positioning adjustments

---

### `MergeTablesTests` - Main Functionality Test Class

**Purpose:** Tests complex table merging operations with historical data integration and regulatory format compliance.

#### Advanced Test Data Provider

##### `MergeTablesTestsData` - Sophisticated Financial Table Scenarios

**Purpose:** Provides extensive real-world financial table merging scenarios using production-like data structures.

**Test Scenario Categories:**

##### 1. Ratios and Supplemental Data (M12a Format)

```csharp
yield return new object[] {
    // Current data: 2022 values with historical periods showing $–/0.00%
    "<table><row><cell>Ratios and Supplemental Data</cell>...",
    "M12a",
    // Expected: Historical data populated with real values
    "<table><row><cell>Ratios and Supplemental Data</cell>..."
};
```

**Business Context:**

* **Fund Performance Metrics:** Total net asset value, number of units, MER, trading expense ratio
* **Multi-Year Display:** 2022-2017 period coverage (6 years)
* **Historical Integration:** Current period ($113,134, 1.14%) merged with historical data
* **Column Management:** Drops excess columns, maintains proper period alignment

**Data Transformation Logic:**

* **Current Input:** Recent period data with placeholder historical values ($–, 0.00%)
* **Historical Source:** Database-stored historical values for previous periods
* **Merged Output:** Complete multi-period table with all historical values populated

##### 2. Net Assets per Unit Tables (M11a Format)

```csharp
// Complex fund performance table with distributions and operations
"<table><row><cell>HSBC Canadian Bond Fund - Investor Series - Net Assets per Unit<sup>(1)</sup></cell>..."
```

**Advanced Processing Features:**

* **Fund-Specific Data:** HSBC Canadian Bond Fund Investor Series
* **Complex Calculations:** Net assets, operations, distributions per unit
* **Period Handling:** Year-end vs interim period processing
* **Column Alignment:** "2018 DROPPED", "2017 DROPPED" indicating column management
* **Header Preservation:** Maintains complex fund naming and superscript formatting

**Merge Logic Complexity:**

* **Header Row Management:** Multi-row headers with fund name and period descriptions
* **Data Row Processing:** Financial calculations (revenues, expenses, gains/losses)
* **Distribution Processing:** Multiple distribution categories per period
* **Column Dropping:** Excess historical columns removed based on configuration

##### 3. Interim Period Processing (M15i, M15m, M15x8)

```csharp
"<table><row><cell>Mar.&lt;br&gt;2023</cell><cell>8.05</cell></row></table>"
```

**Interim vs Annual Logic:**

* **M15i (Interim):** Quarterly/semi-annual data processing
* **M15m (Monthly):** Monthly performance data
* **M15x8 (Series-Specific):** Series X8 specific processing with attributes

**Date Format Processing:**

* **HTML Entities:** `&lt;br&gt;` for line breaks in date display
* **Date Patterns:** "Mar.&lt;br&gt;2023", "Sep. 31&lt;br&gt;2022"
* **Superscript Handling:** `<sup>†</sup>` for footnote indicators
* **Row Type Processing:** `rowtype="Level1.SubHeader"` for hierarchical display

##### 4. Edge Case Scenarios

```csharp
// No history field scenarios
yield return new object[] {
    "<table><row><cell>Mar.&lt;br&gt;2023</cell><cell>8.05</cell></row></table>",
    "NR999",    // No matching configuration
    null        // Expected null result
};
```

**Error Handling Validation:**

* **Unknown Field Names:** "NR999" returns null (no configuration found)
* **Missing Historical Data:** Fields without history return current data unchanged
* **Configuration Mismatches:** Graceful handling of unconfigured field types

#### Core Method Testing

##### Historical Data Loading Tests

* `void CannotCallLoadHistoricFieldTableWithNullRowIdentity()`
* `void CannotCallLoadHistoricFieldTableWithNullConObject()`  
* `void CannotCallLoadHistoricFieldTableWithInvalidFieldName(string value)`

**Parameter Validation Pattern:**

* **RowIdentity:** Required for data context and filtering
* **DBConnection:** Required for database access to historical data
* **FieldName:** Must be non-null, non-empty, non-whitespace

##### Table Merging Tests

* `[Theory] void CanCallMergeTableDataWithCurrentDataTableAndFieldName(string currentDataTable, string fieldName, string expected)`

**Complex Integration Test:**

```csharp
MergeTables mt = new MergeTables();
mt.HistoricTableData = SetupTable();        // Load historical data from XML resources
mt.MergeTableSettings = SetupTableSettings(); // Load merge configuration
var result = mt.MergeTableData(currentDataTable, fieldName);
```

**Processing Pipeline:**

1. **Setup:** Load historical data and merge configuration from XML resources
2. **Input Processing:** Parse current data XML table structure
3. **Historical Lookup:** Find matching historical data by field name
4. **Merge Logic:** Apply field-specific merging rules (M11a, M12a, M15i, etc.)
5. **Output Generation:** Create merged XML table with proper formatting

##### Configuration Testing

* `[Theory] void GetMergeFieldNamePrefixTests(string fieldName, bool exists)`

**Field Name Prefix Validation:**

```csharp
[InlineData("M11", true)]   // M11a, M11b, etc. supported
[InlineData("C1", false)]   // C1 prefix not configured for merging
[InlineData("", false)]     // Empty field names not supported
```

**Business Logic:** Only configured field name prefixes support historical data merging.

---

## Complex Test Data Infrastructure

### Historical Data Setup

```csharp
public static DataTable SetupTable()
{
    DataTable dt = new DataTable();
    dt.Columns.Add("Field_Name", Type.GetType("System.String"));
    dt.Columns.Add("Content", Type.GetType("System.String"));
    dt.Columns.Add("Header_Field_Name", Type.GetType("System.String"));
    dt.Columns.Add("Header_Content", Type.GetType("System.String"));
    dt.Columns.Add("DOCUMENT_NUMBER", Type.GetType("System.String"));
    dt.Columns.Add("FEED_COMPANY_ID", Type.GetType("System.Int32"));
    
    return TestHelpers.LoadXMLTable(Resources.PUB_Data_Integration_DEV.pub_Historic_Field_Data, dt);
}
```

**Schema Analysis:**

* **Field_Name:** Primary key for historical data lookup
* **Content:** XML table content for historical periods
* **Header Fields:** Separate header content management
* **Document Context:** Document number and company ID for data correlation

### Merge Configuration Setup

```csharp
public static DataTable SetupTableSettings()
{
    // Configuration for different field types (M11a, M12a, M15i, etc.)
    return TestHelpers.LoadXMLTable(Resources.PUB_Data_Integration_DEV.pdi_Data_Type_Merge_Table, dt);
}
```

**Configuration Schema:**

* **Field_Name_Prefix:** Pattern matching for field types (M11, M12, M15)
* **Check_Field_Row/Column:** Validation positions for data integrity
* **Desired_Rows/Columns:** Target dimensions for merged output
* **Interim_Indicator/Offset:** Interim period processing configuration

---

## Business logic and integration patterns

### Multi-Period Financial Reporting

The tests validate sophisticated financial reporting requirements:

**Regulatory Compliance:**

* **Multi-Year Tables:** 5-6 year historical coverage for regulatory disclosure
* **Period Consistency:** Proper alignment of annual vs interim periods
* **Data Completeness:** Historical data integration prevents gaps in reporting

**Fund Performance Reporting:**

* **Net Asset Value Tracking:** Per-unit calculations across multiple periods
* **Operations Breakdown:** Revenue, expenses, gains/losses by period
* **Distribution Reporting:** Multiple distribution categories per period

### Historical Data Management

Complex data preservation and integration:

**Data Source Integration:**

* **Current Processing:** Real-time data from current document processing
* **Historical Database:** Stored historical data from previous periods
* **Merge Logic:** Intelligent combination based on field type and configuration

**Column Management:**

* **Dynamic Sizing:** Adjusts table columns based on available historical data
* **Excess Column Handling:** Removes unnecessary columns while preserving data integrity
* **Header Alignment:** Maintains proper column headers across merged periods

### Interim Period Processing

Advanced handling of different reporting periods:

**Period Type Detection:**

* **Annual Reports:** Full year financial data with complete period coverage
* **Interim Reports:** Quarterly/semi-annual data with different column structures
* **Series-Specific:** Fund series-specific reporting requirements

**Date Format Management:**

* **HTML Entity Processing:** Proper handling of formatted dates with line breaks
* **Superscript Footnotes:** Preservation of regulatory footnote indicators
* **Period Alignment:** Consistent date formatting across historical periods

---

## Technical implementation considerations

### Performance Characteristics

* **Database Integration:** Historical data loaded from database resources
* **XML Processing:** Complex XML table parsing and generation
* **Memory Management:** Large DataTable structures for multi-period data
* **Configuration Caching:** Merge settings loaded once per test session

### Data Integrity Management

* **Validation Points:** Check row/column positions for data integrity verification
* **Error Handling:** Graceful handling of missing historical data or configuration
* **Type Safety:** Strong typing for financial data processing
* **Resource Management:** Proper cleanup of database connections and large data structures

### Integration Architecture

* **Database Layer:** Integration with historical data storage systems
* **Configuration Management:** Flexible configuration system for different field types
* **Transform Pipeline:** Integration with broader document transformation workflow
* **Resource Loading:** XML-based test data loading from embedded resources

---

## Test coverage and quality assurance

### Financial Table Format Coverage

**Regulatory Forms:**

* **M11a:** Net Assets per Unit tables with complex operations breakdown
* **M12a:** Ratios and Supplemental Data with performance metrics
* **M15i/M15m/M15x8:** Performance data with various period types

**Processing Scenarios:**

* **New Fund Data:** Current period with no historical data (placeholder values)
* **Established Funds:** Full historical data integration across multiple periods
* **Interim Processing:** Quarterly and semi-annual data handling
* **Series-Specific:** Fund series-specific processing requirements

### Error Handling Coverage

**Parameter Validation:**

* **Null Parameter Rejection:** Comprehensive null checking for all critical parameters
* **Invalid Input Handling:** Empty, whitespace, and malformed input processing
* **Database Connection Management:** Proper error handling for database operations

**Configuration Coverage:**

* **Unknown Field Types:** Graceful handling of unconfigured field names
* **Missing Historical Data:** Appropriate fallback to current data only
* **Invalid Configuration:** Error handling for malformed merge settings

### Data Quality Coverage

**Financial Data Accuracy:**

* **Precision Preservation:** Proper handling of decimal places in financial data
* **Currency Formatting:** Consistent currency symbol and formatting across periods
* **Percentage Display:** Proper percentage formatting with appropriate precision

**Table Structure Integrity:**

* **Column Alignment:** Proper alignment of data across historical periods
* **Header Preservation:** Complex multi-row headers maintained correctly
* **Row Type Management:** Hierarchical row types preserved during merging

---

## Potential enhancements

### Performance Optimizations

1. **Caching Strategy:** Cache frequently accessed historical data and configurations
2. **Parallel Processing:** Process multiple table merges concurrently
3. **Memory Optimization:** Optimize large DataTable operations for better memory usage
4. **Database Optimization:** Optimize historical data queries for better performance

### Functionality Extensions

1. **Additional Table Types:** Support for new regulatory table formats
2. **Dynamic Period Ranges:** Configurable historical period coverage
3. **Advanced Interim Logic:** More sophisticated interim vs annual period detection
4. **Custom Merge Rules:** Client-specific table merging logic

### Testing Enhancements

1. **Performance Testing:** Validate performance with large historical datasets
2. **Integration Testing:** End-to-end testing with actual database systems
3. **Error Recovery Testing:** Test recovery from various database error conditions
4. **Concurrency Testing:** Multi-threaded table merging scenarios

---

## Action items for system maintenance

1. **Historical Data Quality:** Monitor historical data integrity and completeness
2. **Configuration Management:** Keep merge configuration current with new regulatory requirements
3. **Performance Monitoring:** Track table merge performance as historical data grows
4. **Database Optimization:** Optimize historical data storage and retrieval patterns
5. **Regulatory Updates:** Ensure table formats meet current regulatory requirements
6. **Integration Testing:** Validate merge functionality with production data scenarios
