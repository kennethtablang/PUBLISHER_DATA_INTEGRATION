# Tests — TestHelpers (analysis)

## One-line purpose

Utility class providing XML-to-DataTable conversion services, type-safe data loading, random test data generation, and DataTable comparison functionality for test infrastructure support.

---

## Files analyzed

* `Publisher_Data_Operations_Tests/TestHelpers.cs`

---

## What this code contains (high level)

1. **XML Data Loading** — Robust XML-to-DataTable conversion with type-aware parsing and multi-table handling
2. **Type-Safe Data Conversion** — Intelligent type conversion supporting custom enums, dates, booleans, and numeric types
3. **Random Test Data Generation** — Dynamic test data creation for stress testing and edge case validation
4. **DataTable Comparison Utilities** — Sophisticated table comparison logic with extended property handling
5. **Resource Management** — Helper methods for loading embedded XML test data resources

This utility class serves as the foundation for test data management throughout the PDI testing framework, providing reliable and flexible data handling capabilities.

---

## Classes, properties and methods (developer reference)

### `TestHelpers` - Static Utility Class

**Purpose:** Provides essential utility methods for test data management, XML processing, and DataTable operations across the test suite.

---

## Core Utility Methods

### XML Data Loading and Conversion

#### `static DataTable LoadXMLTable(string xml, DataTable dt)`

**Purpose:** Converts XML data to strongly-typed DataTable with intelligent type conversion and multi-table XML handling.

**Parameters:**

* `xml` — XML string containing serialized data
* `dt` — Target DataTable with predefined schema and column types

**Processing Algorithm:**

##### 1. XML Parsing and DataSet Creation

```csharp
XmlDocument xmlDoc = new XmlDocument();
xmlDoc.LoadXml(xml);
XmlReader xmlReader = new XmlNodeReader(xmlDoc);
DataSet ds = new DataSet();
ds.ReadXml(reader);
```

##### 2. Multi-Table Processing

**Challenge Handling:** `ReadXml()` can split incoming data into multiple tables
**Solution:** Iterates through all DataSet tables to find matching columns

##### 3. Type-Aware Data Conversion

**Supported Types:**

**Integer Conversion:**

```csharp
case "System.Int32":
    if (int.TryParse(dtCur.Rows[r][dc].ToString(), out int resInt))
        newRow[dc.ColumnName] = resInt;
```

**Boolean Conversion:**

```csharp
case "System.Boolean":
    newRow[dc.ColumnName] = dtCur.Rows[r][dc].ToString().ToBool();
```

* **Extension Method Usage:** Uses custom `.ToBool()` extension method from `Publisher_Data_Operations.Extensions`

**DateTime Conversion:**

```csharp
case "System.DateTime":
    if (DateTime.TryParse(dtCur.Rows[r][dc].ToString(), out DateTime resDate))
        newRow[dc.ColumnName] = resDate;
    else if (dtCur.Rows[r][dc].ToString().Length > 0)
        newRow[dc.ColumnName] = dtCur.Rows[r][dc].ToString().ToDate(DateTime.MinValue);
```

* **Fallback Strategy:** Uses custom `.ToDate()` extension with `DateTime.MinValue` fallback

* **Empty String Handling:** Avoids processing empty date strings

**Custom Enum Conversion:**

```csharp
case "Publisher_Data_Operations.Extensions.FFDocAge":
    if (int.TryParse(dtCur.Rows[r][dc].ToString(), out int resFF))
        newRow[dc.ColumnName] = (FFDocAge)resFF;
```

* **Enum Support:** Handles custom FFDocAge enum by parsing integer values

**String and Default Conversion:**

```csharp
case "System.String":
default:
    newRow[dc.ColumnName] = dtCur.Rows[r][dc].ToString();
```

##### 4. Data Integrity

* **Column Existence Check:** `if (!dt.Columns.Contains(dc.ColumnName))`

* **Row Bounds Check:** `r >= dtCur.Rows.Count`
* **Change Acceptance:** `dt.AcceptChanges()` finalizes all changes

**Return Value:** Populated DataTable with type-converted data

---

#### `static DataTable LoadTestData(string xml, string tableName)`

**Purpose:** Simplified XML loading for resource-based test data with translation-specific handling.

**Parameters:**

* `xml` — XML formatted data string from resource files
* `tableName` — Source table name for context-specific processing

**Processing Logic:**

##### Standard XML Loading

```csharp
DataSet ds = new DataSet();
using (XmlReader reader = XmlReader.Create(new StringReader(xml)))
    ds.ReadXml(reader);
```

##### Translation Table Special Handling

```csharp
if (tableName.IndexOf("Translation", StringComparison.OrdinalIgnoreCase) >= 0)
    ds.Tables[0].CaseSensitive = true;
```

* **Case Sensitivity:** Translation tables require case-sensitive string comparisons

* **Reason:** Multilingual content may have case-sensitive translation keys

##### Error Handling

```csharp
catch (Exception e)
{
    Console.WriteLine(e.Message);
    throw;
}
```

* **Error Logging:** Console output for debugging

* **Exception Propagation:** Re-throws for proper test failure reporting

**Return Value:** First table from DataSet (`ds.Tables[0]`)

---

### Random Test Data Generation

#### `static DataTable RandomTable(bool includeRowType = false)`

**Purpose:** Generates random DataTable for stress testing, performance testing, and edge case validation.

**Parameters:**

* `includeRowType` — Optional flag to include special FSMRFP row type column

**Generation Algorithm:**

##### Table Structure Generation

```csharp
int cols = rand.Next(2, 20);    // 2-19 columns
int rows = rand.Next(1, 35);    // 1-34 rows
DataTable dt = new DataTable($"Random_{rows}_{cols}");
```

##### Column Creation

```csharp
for (int c = 0; c < cols; c++)
    dt.Columns.Add($"Column_{c + 1}");
```

* **Naming Pattern:** Sequential "Column_1", "Column_2", etc.

##### Special Column Handling

```csharp
if (includeRowType)
    dt.Columns.Add(Publisher_Data_Operations.Generic.FSMRFP_ROWTYPE_COLUMN);
```

* **Business Logic Integration:** Adds row type column used by Generic class for FSMRFP processing

##### Data Population

```csharp
for (int r = 0; r < rows; r++)
{
    DataRow dr = dt.NewRow();
    for (int c = 0; c < cols; c++)
        dr[c] = RandomString(rand.Next(0, 50));  // 0-49 character strings
}
```

##### Row Type Data

```csharp
if (dt.Columns.Contains(Publisher_Data_Operations.Generic.FSMRFP_ROWTYPE_COLUMN))
    dr[Publisher_Data_Operations.Generic.FSMRFP_ROWTYPE_COLUMN] = "Level1." + RandomString(10);
```

* **Pattern:** "Level1." prefix with 10-character random suffix

* **Business Context:** Simulates hierarchical row type structure

**Return Value:** Populated DataTable with random data

---

#### `static string RandomString(int length)`

**Purpose:** Generates random alphanumeric strings for test data population.

**Parameters:**

* `length` — Desired string length

**Implementation:**

```csharp
const string chars = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
return new string(Enumerable.Repeat(chars, length)
    .Select(s => s[rand.Next(s.Length)]).ToArray());
```

**Character Set:** Mixed case letters and digits (62 possible characters)
**Algorithm:** LINQ-based character selection with uniform distribution

---

### DataTable Comparison Utilities

#### `static bool TablesEqual(DataTable dt, PDI_DataTable pdi)`

**Purpose:** Compares standard DataTable with custom PDI_DataTable, handling extended properties and row type columns.

**Parameters:**

* `dt` — Standard ADO.NET DataTable
* `pdi` — Custom PDI_DataTable with extended functionality

**Comparison Algorithm:**

##### Structural Validation

```csharp
bool isRowType = dt.Columns.Contains(Publisher_Data_Operations.Generic.FSMRFP_ROWTYPE_COLUMN);
if (dt.Rows.Count != pdi.Rows.Count) return false;
if (dt.Columns.Count != pdi.Columns.Count + (isRowType ? 1 : 0)) return false;
if (dt.TableName != pdi.TableName) return false;
```

**Row Type Column Handling:** Accounts for row type column that may not exist in PDI_DataTable
**Count Adjustment:** `pdi.Columns.Count + (isRowType ? 1 : 0)`

##### Cell-by-Cell Comparison

```csharp
for (int row = 0; row < dt.Rows.Count; row++)
{
    for (int col = 0; col < dt.Columns.Count; col++)
    {
        // Special handling for row type column vs extended properties
        // Standard cell value comparison
    }
}
```

##### Extended Properties Handling

```csharp
if (isRowType && dt.Columns[col].ColumnName == Publisher_Data_Operations.Generic.FSMRFP_ROWTYPE_COLUMN)
{
    if (!((PDI_DataRow)pdi.Rows[row]).ExtendedProperties.ContainsValue(dt.Rows[row][col]))
        return false;
}
```

**Special Logic:** Row type data stored as column in DataTable but as ExtendedProperty in PDI_DataRow
**Validation:** Checks if ExtendedProperties contains the expected row type value

##### Standard Cell Comparison

```csharp
else if (dt.Rows[row][col] != pdi.Rows[row][col])
    return false;
```

**Return Value:** Boolean indicating table equality

---

## Technical implementation considerations

### Performance Characteristics

* **XML Processing:** DOM-based parsing for small to medium datasets

* **Type Conversion:** Reflection-free type switching for performance
* **Random Generation:** Efficient LINQ-based string generation
* **Table Comparison:** O(n×m) complexity for row/column iteration

### Memory Management

* **XML Loading:** Creates temporary XmlDocument and DataSet objects

* **Random Data:** Generates data in memory without disk I/O
* **String Generation:** Uses LINQ deferred execution for efficiency
* **Comparison:** No additional memory allocation beyond iteration variables

### Error Handling Strategy

* **XML Parsing:** Exception propagation with console logging

* **Type Conversion:** Graceful fallback for failed conversions
* **Null Handling:** Implicit null handling through ToString() calls
* **Bounds Checking:** Column and row existence validation

---

## Integration patterns

### Test Data Pipeline

The TestHelpers class serves as the foundation for test data management:

1. **Resource Loading:** `LoadTestData()` reads embedded XML resources
2. **Schema Mapping:** `LoadXMLTable()` converts to strongly-typed DataTables
3. **Type Conversion:** Automatic handling of custom types and enums
4. **Validation:** `TablesEqual()` enables result validation

### Extension Method Integration

Heavy reliance on custom extension methods:

* **`.ToBool()`** — String to boolean conversion
* **`.ToDate()`** — String to DateTime conversion with fallback
* **Integration Point:** Links test utilities with production extension methods

### Business Logic Integration

* **FSMRFP Row Types:** Special handling for financial document row classification

* **FFDocAge Enum:** Custom enum conversion for fund document age status
* **Translation Tables:** Case-sensitive handling for multilingual content
* **PDI_DataTable:** Custom table comparison supporting extended properties

---

## Business logic support

### Financial Document Testing

The utilities support specific financial document testing needs:

* **Fund Document Age:** FFDocAge enum conversion for regulatory compliance testing
* **Row Type Classification:** FSMRFP row type handling for document structure validation
* **Multilingual Content:** Case-sensitive translation table handling

### Data Quality Assurance

* **Type Safety:** Strong typing prevents data corruption during testing

* **Multi-Table Handling:** Robust handling of complex XML structures
* **Extended Properties:** Support for custom data attributes beyond standard columns

---

## Potential enhancements

### Performance Optimizations

1. **Streaming XML Processing:** XmlReader-based processing for large datasets
2. **Parallel Processing:** Multi-threaded table comparison for large datasets
3. **Type Conversion Caching:** Cache type information to avoid repeated reflection
4. **Memory Pooling:** Reuse DataTable and DataRow objects

### Functionality Extensions

1. **Schema Validation:** Automatic schema comparison before data loading
2. **Custom Type Handlers:** Pluggable type conversion system
3. **Comparison Options:** Configurable comparison rules (case sensitivity, precision)
4. **Bulk Operations:** Batch processing for multiple tables

### Error Handling Improvements

1. **Detailed Error Messages:** More specific error information with context
2. **Partial Success Handling:** Continue processing after individual row failures
3. **Validation Rules:** Pre-processing validation with detailed error reports
4. **Recovery Strategies:** Automatic correction of common data issues

---

## Action items for system maintenance

1. **Performance Monitoring:** Monitor XML processing performance on large test datasets
2. **Extension Method Dependencies:** Ensure extension methods remain stable across versions
3. **Type Conversion Testing:** Comprehensive testing of all supported data types
4. **Random Data Validation:** Ensure random data generation covers edge cases effectively
5. **Comparison Logic Review:** Validate PDI_DataTable comparison logic with production data
6. **Documentation Updates:** Document supported XML formats and type conversion rules
