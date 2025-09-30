# Tests — Extensions Data Tests (analysis)

## One-line purpose

Test suite for XML-to-DataTable conversion functionality and PDI_DataTable custom implementation, focusing on bidirectional XML transformation, edge case handling, and stress testing with random data generation.

---

## Files analyzed

* `Publisher_Data_Operations_Tests/Extensions/DataTests.cs`

---

## What this code contains (high level)

1. **XML Conversion Testing** — Comprehensive testing of XMLtoDataTable extension method with real financial document XML samples
2. **Bidirectional XML Transformation** — Round-trip testing to ensure XML-to-DataTable-to-XML consistency
3. **PDI_DataTable Validation** — Testing custom DataTable implementation with extended properties and row type support
4. **Edge Case Handling** — Validation of null, empty, and malformed XML input scenarios
5. **Stress Testing Infrastructure** — Integration with TestHelpers for random data generation and comparison utilities

This test suite validates the core XML processing capabilities that enable the Publisher Data Integration system to handle complex financial table structures with metadata preservation.

---

## Classes, properties and methods (developer reference)

### `DataTests` - Static Test Class (XML Processing Focus)

**Purpose:** Tests extension methods for XML-DataTable conversion, particularly the critical XMLtoDataTable functionality used throughout the financial document processing pipeline.

#### Test Infrastructure Pattern

##### Extensive Commented-Out Tests

The class contains a comprehensive set of commented-out tests covering DataRow extension methods:

**DataRow String Value Methods:**

* `GetPartialColumnStringValue()` — Column matching with partial names
* `GetExactColumnStringValue()` — Exact column name matching  
* `GetStringValue(int col)` — Index-based string retrieval

**DataRow Numeric Value Methods:**

* `GetPartialColumnIntValue()` — Integer parsing with partial column matching
* `GetExactColumnIntValue()` — Integer parsing with exact column matching
* `GetIntValue(int col)` — Index-based integer retrieval

**Specialized Data Type Methods:**

* `GetExactColumnFFDocAge()` — Custom FFDocAge enum retrieval
* `GetPartialColumnBoolValue()` — Boolean parsing with partial column matching
* `GetBoolValue(int col)` — Index-based boolean retrieval

**Column Discovery Methods:**

* `FindDataRowColumn()` — Dynamic column discovery in DataRow
* `FindDataTableColumn()` — Dynamic column discovery in DataTable

**Change Tracking Methods:**

* `GetChangedColumns()` — Multiple overloads for DataRow, DataRow[], and DataTable
* `RowHasChanged()` — Individual row change detection

**Data Cleaning Methods:**

* `RemoveBlankColumns()` — Column cleanup with minimum column preservation
* `RemoveBlankColumnsAtEnd()` — End-column cleanup with row type consideration
* `RemoveBlankRows()` — Row cleanup functionality

**Integration Methods:**

* `ValidateXML()` — XML structure validation with Logger integration
* `ToCSV()` — DataTable to CSV conversion
* `GetDataRowDictionary()` — DataRow to Dictionary conversion
* `ReplaceByDataRow()` — Template replacement using DataRow values

**Pattern Analysis:** Each commented test follows consistent structure:

* Null parameter validation with `ArgumentNullException`
* Invalid parameter validation with Theory/InlineData for edge cases
* Basic functionality validation (though marked as "Create or modify test")

---

#### Active XML Processing Tests

##### Basic XML Conversion Test

* `[Fact] static void CanCallXMLtoDataTable()`

**Purpose:** Validates basic XML-to-DataTable conversion functionality

**Test Case:**

```csharp
string xmlString = "<table><row><cell>value</cell></row></table>";
DataTable dt = xmlString.XMLtoDataTable();
Assert.True(dt.Rows.Count > 0);
```

**Validation:** Ensures non-empty DataTable result from simple XML structure

##### Comprehensive Round-Trip Testing

* `[Theory] static void XMLtoDataTableAndBackTests(string xmlString)`

**Purpose:** Validates bidirectional XML transformation with production-like financial data

**Test Data Categories:**

###### 1. Simple Table Structure

```xml
<table testAttrib="Test"><row testRowAttrib="TestRow">
    <cell>Fund</cell><cell>Subsidiary</cell><cell>Ownership</cell><cell>Principal Place of Business</cell>
</row></table>
```

* **Metadata Preservation:** Tests attribute handling (testAttrib, testRowAttrib)
* **Column Structure:** Basic fund subsidiary data structure

###### 2. Financial Performance Tables

```xml
<table><row>
    <cell>Investor Series - Net Assets per Unit(1)</cell>
    <cell></cell><cell></cell>...
</row><row>
    <cell>Period ended June 30 2022, and the year(s) ended December 31</cell>
    ...
</row></table>
```

* **Financial Reporting:** Net assets per unit data over multiple periods
* **Date Formatting:** HTML line breaks in dates (`&lt;br /&gt;`)
* **Currency Data:** Dollar amounts with proper formatting
* **Empty Cell Handling:** Mixed empty and populated cells

###### 3. Multilingual Content

```xml
<table><row rowtype="Level1.Header">
    <cell>Actif net par action du Fonds ($)&lt;sup&gt;1&lt;/sup&gt;</cell>
    <cell>03/31</cell><cell>03/31</cell>...
</row></table>
```

* **French Content:** French financial terminology
* **HTML Formatting:** Superscript tags, strong tags, HTML entities
* **Row Type Metadata:** Hierarchical row classification (Level1.Header, Level1.Subheader, etc.)
* **Missing Translation Indicators:** "MISSING FRENCH:" prefixes

###### 4. Complex Financial Tables

```xml
<table><row sourceDocument="50428219" timeStamp="2022-07-29">
    <cell>&lt;strong&gt;As at June 30, 2022&lt;/strong&gt;</cell>
    ...
</row></table>
```

* **Source Document Tracking:** sourceDocument and timeStamp attributes
* **Multi-Currency Data:** CAD, USD, GBP, JPY currency handling
* **Complex Financial Data:** Forward contracts, derivatives, fund allocations
* **Hierarchical Grouping:** Fund-level aggregation rows

###### 5. Edge Cases

```xml
<table><row/></table>
<table><row rowType="Level1.SubHeader" /><row rowType="Level1.Header"><cell/><cell> </cell></row></table>
<table/>
```

* **Empty Structures:** Self-closing rows and tables
* **Mixed Case Attributes:** "rowType" vs "rowtype"
* **Whitespace Handling:** Empty cells vs cells with spaces

**Round-Trip Validation Logic:**

```csharp
string xmlTestString = xmlString.ReplaceCI("<cell></cell>", "<cell />"); 
xmlTestString = xmlTestString.ReplaceCI("<row/>", "<row />").ReplaceCI("<cell/>", "<cell />");
PDI_DataTable dt = xmlString.XMLtoDataTable();
string xml = dt.DataTabletoXML();
Assert.Equal(xmlTestString, xml);
```

**Normalization Strategy:**

* **Self-Closing Tag Conversion:** Converts `<cell></cell>` to `<cell />` for consistency
* **Extension Method Usage:** Uses `.ReplaceCI()` (case-insensitive replace) extension method
* **Round-Trip Assertion:** Ensures XML → DataTable → XML produces identical result

##### Error Handling Tests

* `[Theory] static void XMLtoDataTableWithInvalidXmlStringReturnsNull(string value)`

**Test Cases:**

* `null` input
* Empty string `""`
* Whitespace-only `"   "`

**Expected Behavior:** Returns `null` for invalid XML input, providing graceful error handling

---

### `PDI_DataTableTests` - Custom DataTable Implementation Tests

**Purpose:** Validates the custom PDI_DataTable class that extends standard DataTable functionality with row type support and extended properties.

#### Test Infrastructure Setup

##### Constructor Setup

```csharp
public PDI_DataTableTests()
{
    _tableName = "TestValue19918901";
    _tableNamespace = "TestValue1969199446";
    _dt = new DataTable();
    _testClass = new PDI_DataTable(_tableName, _tableNamespace);
}
```

**Test Context:** Creates test instances with randomized names to ensure isolation

#### Constructor Validation Tests

##### Multiple Constructor Patterns

* `[Fact] void CanConstruct()`

**Validated Constructors:**

```csharp
var instance = new PDI_DataTable();                           // Default constructor
var instance = new PDI_DataTable(_tableName);                 // Name-only constructor  
var instance = new PDI_DataTable(_tableName, _tableNamespace); // Name + namespace constructor
var instance = new PDI_DataTable(_dt);                        // DataTable conversion constructor
```

**Validation:** Ensures all constructor overloads produce non-null instances

##### TableName Preservation Test

* `[Theory] void ConstructWithTableNameReturnsTableName(string value)`

**Test Cases:**

* Empty string `""`
* Whitespace `"   "`
* Actual table name `"TableName"`

**Validation:** Confirms table name is properly preserved during construction

#### Stress Testing Integration

##### Random Data Conversion Test

* `[Fact] void CanConvertDataTable()`

**Test Strategy:**

```csharp
for (int i = 0; i < 1000; i++)
{
    DataTable dt = TestHelpers.RandomTable(i % 2 == 0);
    PDI_DataTable pd1 = new PDI_DataTable(dt);
    Assert.True(TestHelpers.TablesEqual(dt, pd1));
}
```

**Stress Testing Approach:**

* **1000 Iterations:** Comprehensive stress testing
* **Random Table Generation:** Uses TestHelpers.RandomTable() for varied data
* **Row Type Variation:** Alternates between tables with/without row type columns (`i % 2 == 0`)
* **Equality Validation:** Uses TestHelpers.TablesEqual() for deep comparison

**Integration Points:**

* **TestHelpers.RandomTable():** Generates random tables with optional row type columns
* **TestHelpers.TablesEqual():** Handles comparison including extended properties
* **Row Type Handling:** Tests the critical row type column functionality

---

## Business logic and integration patterns

### XML Processing in Financial Documents

The tests validate XML processing capabilities essential for financial document workflows:

**Publisher XML Format Support:**

* **Table Structure:** Standard table/row/cell hierarchy
* **Metadata Attributes:** Row types, source documents, timestamps
* **Financial Content:** Currency amounts, percentages, dates
* **Multilingual Content:** English-French bilingual processing

**Row Type Classification System:**

* **Level1.Header:** Primary section headers
* **Level1.Subheader:** Secondary section headers  
* **Level1.Total:** Summary/total rows
* **Level2.Subheader:** Tertiary classification
* **Business Logic:** Enables document structure understanding and formatting

### Extended Properties Integration

PDI_DataTable provides enhanced functionality beyond standard DataTable:

**Row Type Storage:**

* Standard DataTable: Row type stored as column
* PDI_DataTable: Row type stored in ExtendedProperties
* **Advantage:** Cleaner table structure while preserving metadata

**Comparison Logic:**

* **TestHelpers.TablesEqual():** Handles both column-based and ExtendedProperty-based row types
* **Seamless Conversion:** Standard DataTable ↔ PDI_DataTable with metadata preservation

### Production Data Validation

Test data represents real-world financial document scenarios:

**Canadian Fund Reporting:**

* Net assets per unit calculations
* Multi-period financial performance
* Distribution and dividend reporting
* Forward contract and derivative data

**Regulatory Compliance:**

* Source document tracking (sourceDocument attributes)
* Timestamp preservation (timeStamp attributes)
* Audit trail capabilities

---

## Technical implementation considerations

### XML Processing Performance

The tests indirectly validate performance characteristics:

**Round-Trip Efficiency:**

* Large XML documents (complex financial tables) processed successfully
* Memory management during conversion cycles
* String normalization overhead (ReplaceCI operations)

**Error Handling Strategy:**

* Graceful null handling for invalid XML
* No exceptions thrown for malformed input
* Consistent return values (null for failures)

### Data Structure Complexity

Tests handle complex data scenarios:

**HTML Entity Encoding:**

* `&lt;` `&gt;` for angle brackets in content
* `&#160;` for non-breaking spaces
* `&lt;sup&gt;` `&lt;/sup&gt;` for superscript formatting

**Multi-Currency Handling:**

* CAD, USD, GBP, JPY currency codes
* Numeric formatting preservation
* Negative value handling with parentheses

**Hierarchical Data:**

* Row type classification systems
* Source document grouping
* Fund-level aggregation patterns

### Integration with Extension Methods

Heavy reliance on custom extension methods:

**String Extensions:**

* `.ReplaceCI()` — Case-insensitive string replacement
* `.XMLtoDataTable()` — Core XML parsing functionality

**DataTable Extensions:**

* `.DataTabletoXML()` — Reverse conversion to XML
* Integration with broader extension method ecosystem

---

## Test coverage and quality assurance

### Comprehensive XML Scenario Coverage

Test data covers extensive real-world scenarios:

**Financial Document Types:**

* Fund performance reports (Net Assets per Unit)
* Forward contract tables
* Distribution schedules
* Multi-period comparative data

**Content Complexity:**

* Multilingual content (English-French)
* HTML formatting preservation
* Missing translation handling
* Currency and date formatting

**Structural Variations:**

* Self-closing tags vs explicit closing
* Empty cells and rows
* Attribute variations (case sensitivity)
* Nested content with formatting

### Edge Case Validation

Thorough testing of boundary conditions:

**Input Validation:**

* Null, empty, and whitespace-only XML
* Malformed XML structures
* Mixed case attribute names

**Output Consistency:**

* Round-trip transformation accuracy
* Self-closing tag normalization
* Metadata preservation

**Error Scenarios:**

* Graceful null returns for invalid input
* No exception throwing for edge cases

### Stress Testing Integrations

Random data generation provides comprehensive validation:

**Scale Testing:**

* 1000 iteration loops for statistical confidence
* Variable table structures (2-20 columns, 1-35 rows)
* Random content generation

**Row Type Testing:**

* Alternating tables with/without row type columns
* Extended properties vs column-based storage
* Conversion accuracy validation

---

## Potential enhancements

### Test Coverage Expansion

1. **Performance Testing:** Add timing assertions for large XML documents
2. **Memory Testing:** Validate memory usage during conversion cycles
3. **Concurrent Testing:** Multi-threaded XML processing validation
4. **Error Detail Testing:** More specific error condition validation

### XML Processing Improvements

1. **Schema Validation:** XSD-based XML structure validation
2. **Encoding Testing:** Different character encodings and special characters
3. **Namespace Testing:** XML namespace handling validation
4. **Large Document Testing:** Multi-megabyte XML document handling

### Integration Testing Enhancements

1. **End-to-End Testing:** Full Publisher pipeline integration tests
2. **Database Integration:** PDI_DataTable with Entity Framework testing
3. **Template Integration:** XML processing with template parameter replacement
4. **Multilingual Testing:** Comprehensive bilingual content validation

---

## Action items for system maintenance

1. **Commented Code Review:** Evaluate whether commented-out extension method tests should be implemented or removed
2. **Performance Monitoring:** Monitor XML processing performance as document complexity increases
3. **Test Data Updates:** Keep test XML samples current with production document formats
4. **Extension Method Validation:** Ensure ReplaceCI and other extension methods remain stable
5. **Error Handling Review:** Validate XML error handling meets production requirements
6. **Documentation Updates:** Document XML format requirements and supported features
