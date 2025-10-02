# Tests — Extensions Table Tests (analysis)

## One-line purpose

Comprehensive test suite for TableList functionality validating financial table generation across multiple regulatory formats including 10-year performance tables, investment allocation tables, distribution schedules, and multilingual currency formatting with axis label marking.

---

## Files analyzed

* `Publisher_Data_Operations_Tests/Extensions/TableTests.cs`

---

## What this code contains (high level)

1. **Regulatory Table Generation** — Testing 10-year performance tables with filing year calculations and negative return counting
2. **Investment Allocation Tables** — Validation of Schedule 16/17 tables with percentage formatting and hierarchical row levels
3. **Distribution Tables** — Multi-column distribution data processing with N/A handling and temporal organization
4. **Currency Time Series** — 10K value tables with axis label marking and month name localization
5. **Multilingual Financial Formatting** — Comprehensive English-French formatting with cultural number conventions

This test suite validates the TableList class functionality that generates XML tables for Publisher document integration, ensuring proper regulatory compliance and cultural formatting standards.

---

## Test Method Categories and Analysis

### Regulatory Performance Table Tests

#### Ten-Year Performance Table Generation

* `[Theory] void assembleTenYearTableTests(int filingYear, string[] yearData, string expectedString, int expectedYears, int expectedNegative)`

**Purpose:** Validates 22b-style regulatory table generation with calendar year calculations and negative return counting.

**Test Case Analysis:**

##### 1. Partial 10-Year Data (5 Years with Gaps)

```csharp
[InlineData(2020, new string[] { "", "", "", "", "", "2.5", "6.2", "3.2", "-0.5", "-6.5" }, 
"<table><row><cell>2015</cell><cell>2.5%</cell></row><row><cell>2016</cell><cell>6.2%</cell></row><row><cell>2017</cell><cell>3.2%</cell></row><row><cell>2018</cell><cell>-0.5%</cell></row><row><cell>2019</cell><cell>-6.5%</cell></row></table>", 5, 2)]
```

**Business Logic Validation:**

* **Filing Year Calculation:** 2020 filing year generates 2015-2019 calendar years (10-year lookback)
* **Empty Data Handling:** Leading empty strings ignored, only non-empty values generate rows
* **Year Generation:** Years calculated backwards from filing year (2020-1=2019, 2020-2=2018, etc.)
* **Calendar Year Count:** 5 calendar years with data
* **Negative Return Count:** 2 years with negative returns ("-0.5", "-6.5")
* **Percentage Formatting:** Values formatted with "%" suffix and proper decimal places

##### 2. Complete 10-Year Data Set

```csharp
[InlineData(2020, new string[] { "2.1", "2.5", "-1.6", "6.5", "3.7", "2.9", "-4.21", "5.7", "7.2", "3.5" }, 
"<table><row><cell>2010</cell><cell>2.10%</cell></row>...<row><cell>2019</cell><cell>3.50%</cell></row></table>", 10, 2)]
```

**Complete Dataset Processing:**

* **Full Decade:** 2010-2019 (10 complete calendar years)
* **Decimal Formatting:** Values formatted to 2 decimal places ("2.1" → "2.10%")
* **Negative Returns:** "-1.6" and "-4.21" counted as negative years
* **Sequential Years:** Proper chronological ordering from earliest to latest

##### 3. All-Empty Data Handling

```csharp
[InlineData(2020, new string[] { "", "", "", "", "", "", "", "", "", "" }, "<table></table>", 0, 0)]
```

**Edge Case Validation:**

* **Empty Table Generation:** All empty values result in empty `<table></table>` XML
* **Zero Counts:** Both calendar years and negative years return 0
* **Graceful Handling:** No errors thrown for all-empty datasets

**Constructor and Processing Pattern:**

```csharp
TableList tl = new TableList(filingYear);  // 22b-style table with filing year context
foreach (string s in yearData)
    tl.AddValidation(s);                   // Add values, empty strings filtered out
string result = tl.GetTableString(out int calcCalYears, out int calcNegYears);
```

**Regulatory Compliance:** The out parameters (`calcCalYears`, `calcNegYears`) suggest regulatory reporting requirements where the number of calendar years and negative return years must be tracked and reported.

---

### Investment Allocation Table Tests

#### Schedule 16 Table Format (Top Holdings)

* `[Theory] void assemble16TableTests(string[] enData, string[] frData, string[] valData, string expectedEnglishString, string expectedFrenchString)`

**Purpose:** Validates top holdings table generation with bilingual company names and percentage allocations.

**Test Data Analysis:**

```csharp
[InlineData(new string[] { "Power Financial Corp. ", "Bank of Nova Scotia ", "Loblaw Companies Ltd. ", ... },
           new string[] { "Corporation Financière Power ", "Banque de Nouvelle-Écosse ", "Les Compagnies Loblaw ltée ", ... },
           new string[] { "3.5", "3.2", "3.17", "3.14", "3.04" }, ...)]
```

**Business Context:**

* **Canadian Companies:** Real Canadian corporate names (Power Financial, Bank of Nova Scotia, Loblaw)
* **Official Translations:** Proper French corporate name translations
* **Portfolio Percentages:** Holdings percentages in descending order (3.5% to 3.04%)

**Cultural Formatting Differences:**

* **English Output:** `"Power Financial Corp."` with `"3.50%"` (decimal point, no space)
* **French Output:** `"Corporation Financière Power"` with `"3,50\u00A0%"` (decimal comma, non-breaking space)

**Processing Pattern:**

```csharp
TableList tl = new TableList();  // Default percentage table
for (int i = 0; i < enData.Length; i++)
    tl.AddValidation(valData[i], enData[i], frData[i]);  // value, English, French
```

#### Schedule 17 Table Format (Investment Mix with Levels)

* `[Theory] void assemble17TableTests(string[] enData, string[] frData, string[] valData, string[] levelData, string expectedEnglishString, string expectedFrenchString)`

**Complex Hierarchical Structure:**

```csharp
[InlineData(new string[] { "Money Market Securities", "Corporations", "Municipalities and Semi-Public Institutions", ... },
           new string[] { "Titres de marché monétaire", "Sociétés", "Municipalités et institutions parapubliques", ... },
           new string[] { "82.2", "74", "4.4", "3.8", "17.8" },
           new string[] { "1", "2", "2", "2", "1" }, ...)]
```

**Row Level Logic:**

* **Level 1:** Primary categories ("Money Market Securities", "Canadian Bonds")
* **Level 2:** Subcategories ("Corporations", "Municipalities")
* **Column Positioning:** Level determines which column receives the percentage value

**Expected Output Structure:**

```xml
<table>
    <row><cell>Money Market Securities</cell><cell>82.2%</cell><cell /></row>      <!-- Level 1: First column -->
    <row><cell>Corporations</cell><cell /><cell>74.0%</cell></row>                <!-- Level 2: Second column -->
    <row><cell>Canadian Bonds</cell><cell>17.8%</cell><cell /></row>              <!-- Level 1: First column -->
</table>
```

**Business Logic:** Represents investment allocation hierarchy where main categories and subcategories are displayed in different columns to show portfolio structure.

---

### Specialized Table Format Tests

#### Number of Investments Table

* `[Theory] void assembleNumberOfInvestmentsTableTests(...)`

**Unique Characteristics:**

* **TableTypes.Number:** Non-percentage formatting (no "%" suffix)
* **Row Level Sorting:** Level 1 items appear first, then Level 2 items
* **Reverse Order Display:** Within levels, items displayed in reverse order

**Expected Output:**

```xml
<table>
    <row><cell>Total Number of Investments</cell><cell>79</cell></row>    <!-- Level 1 -->
    <row><cell>Equity Assets</cell><cell>41</cell></row>                  <!-- Level 2 -->
    <row><cell>Fixed Income</cell><cell>38</cell></row>                   <!-- Level 2 -->
</table>
```

#### Distribution Table Format

* `[Theory] void assembleDistributionsTableTests(...)`

**Multi-Column Distribution Processing:**

```csharp
[InlineData(new string[] { "English Header", "October 2020", "November 2020", "December 2020", "English Header", "October 2020", "November 2020", "December 2020" },
           new string[] { "French Header", "october 2020", "november 2020", "december 2020", "French Header", "october 2020", "november 2020", "december 2020" },
           new string[] { "F", "N/A", "0.249", "N/A", "F5", "0.033", "0.033", "0.033" },
           new string[] { "-1", "2", "1", "0", "-1", "2", "1", "0" }, ...)]
```

**Complex Data Organization:**

* **Row Numbers:** "-1" indicates header rows, positive numbers indicate data rows
* **Column Grouping:** Multiple series (F and F5) with same month structure
* **N/A Handling:** "N/A" values converted to "-" in output
* **Multi-Value Processing:** Same month names consolidate into single rows with multiple value columns

**Expected Structure:**

```xml
<table>
    <row><cell>English Header</cell><cell>F</cell><cell>F5</cell></row>
    <row><cell>October 2020</cell><cell>-</cell><cell>0.033</cell></row>
    <row><cell>November 2020</cell><cell>0.249</cell><cell>0.033</cell></row>
    <row><cell>December 2020</cell><cell>-</cell><cell>0.033</cell></row>
</table>
```

**Business Context:** Represents fund distribution schedules where different series (F, F5) have distributions in different months.

---

### Time Series Currency Table Tests

#### 10K Value Tables with Axis Marking

* `[Theory] void assemble10KTableTests(string[] valData, string[] dateData, string expectedEnglishString, string expectedFrenchString)`

**Sophisticated Date and Currency Processing:**

```csharp
[InlineData(new string[] { "10112", "10005", "9987", ... },
           new string[] { "02/2010", "03/2010", "04/2010", ... },
           "<table><row><cell>Feb 2010</cell><cell>$10,112</cell><cell>Y</cell></row>...",
           "<table><row><cell>févr. 2010</cell><cell>$10,112</cell><cell>Y</cell></row>...")]
```

**Advanced Processing Features:**

##### Axis Label Marking System

* **First/Last Marking:** First and last entries always marked with "Y"
* **December Marking:** December entries marked for chart axis labels
* **Chart Integration:** "Y" markers indicate chart axis points for Publisher chart generation

##### Multilingual Month Names

**English Month Abbreviations:**

* "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec", "Jan"

**French Month Abbreviations:**

* "févr.", "mars", "avr.", "mai", "juin", "juill.", "août", "sept.", "oct.", "nov.", "déc.", "janv."

**Cultural Differences:**

* **French Periods:** Most French abbreviations include periods (févr., avr., juill.)
* **No-Period Exceptions:** "mars", "mai", "juin", "août" don't use periods
* **Accent Handling:** Proper French accents (févr., août)

##### Month Dictionary Helper

```csharp
public Dictionary<string, string[]> getShortMonths()
{
    Dictionary<string, string[]> shortMonths = new Dictionary<string, string[]>();
    shortMonths.Add("SF01", new string[2] {"Jan", "janv." });
    shortMonths.Add("SF02", new string[2] { "Feb", "févr." });
    // ... complete month mapping
}
```

**Integration Pattern:**

```csharp
TableList tl = new TableList(TableTypes.Currency);  // Currency formatting with $ symbols
tl.ShortMonths = getShortMonths();                  // Bilingual month name dictionary
```

---

## Business logic and integration patterns

### Regulatory Compliance Framework

The tests validate specific regulatory requirements:

**22b Tables (Ten-Year Performance):**

* Filing year-based calendar year calculations
* Negative return year counting for risk disclosure
* Proper historical period coverage (up to 10 years)
* Empty data handling for funds with limited history

**Investment Allocation Tables (Schedule 16/17):**

* Top holdings disclosure with percentage thresholds
* Investment mix categorization with hierarchical levels
* Bilingual corporate name accuracy for Canadian companies

### Multilingual Financial Formatting

Comprehensive cultural formatting validation:

**English-Canadian Standards:**

* Dollar sign before amount ($10,112)
* Decimal point for percentages (3.50%)
* No space before percentage symbol
* Comma thousands separators

**French-Canadian Standards:**

* Dollar sign after amount (10 112 $)
* Decimal comma for percentages (3,50 %)
* Non-breaking space before percentage symbol (\u00A0)
* Space thousands separators

### Chart Integration Infrastructure

The axis marking system supports Publisher chart generation:

**Intelligent Axis Selection:**

* Always mark first and last data points
* Mark December dates for annual intervals
* Limit to 10 total axis labels for chart readability
* Support Publisher chart XML generation

### Data Quality and Processing

Sophisticated data handling patterns:

**Empty Data Management:**

* Skip empty string values in processing
* Generate empty tables for all-empty datasets
* Maintain data integrity across multilingual processing

**Type-Specific Formatting:**

* Percentage tables: Add "%" suffix with proper decimal places
* Currency tables: Add "$" prefix with comma separators
* Number tables: Plain numeric formatting without symbols

---

## Technical implementation considerations

### Performance Characteristics

* **Sequential Processing:** Linear processing of table data arrays
* **Cultural Formatting:** .NET CultureInfo-based number formatting
* **XML Generation:** Efficient string building for table XML
* **Memory Usage:** Minimal memory overhead with direct array processing

### Integration Points

* **Publisher XML:** Direct XML table generation for Publisher import
* **Chart Services:** Axis marking coordinates with chart generation
* **Cultural Services:** Integration with multilingual formatting infrastructure
* **Regulatory Services:** Compliance calculations (calendar years, negative returns)

### Data Validation Strategy

* **Input Sanitization:** Empty string filtering and null handling
* **Format Validation:** Proper decimal and percentage formatting
* **Cultural Compliance:** Accurate bilingual formatting standards
* **Business Rule Enforcement:** Regulatory calculation requirements

---

## Test coverage and quality assurance

### Table Format Coverage

**Regulatory Tables:**

* 10-year performance tables (22b format)
* Investment allocation tables (Schedule 16/17)
* Distribution schedules with multi-column data
* Time series currency tables (10K values)

**Data Scenarios:**

* Complete datasets with all values
* Partial datasets with empty values
* All-empty datasets for edge case handling
* Multi-series data for complex distributions

### Cultural Formatting Coverage

**Language Support:**

* English-Canadian financial formatting
* French-Canadian financial formatting
* Bilingual month name handling
* Cultural number format compliance

**Formatting Scenarios:**

* Currency amounts with proper symbols
* Percentage values with cultural conventions
* Date formatting with month abbreviations
* Non-breaking space typography

### Business Logic Coverage

**Regulatory Calculations:**

* Filing year to calendar year mapping
* Negative return year counting
* Historical period coverage validation
* Empty data period handling

**Hierarchical Processing:**

* Row level-based column positioning
* Multi-level investment categorization
* Priority-based sorting and display
* Cross-series data consolidation

---

## Potential enhancements

### Functionality Extensions

1. **Additional Table Types:** Support for new regulatory table formats
2. **Dynamic Axis Marking:** Configurable axis labeling algorithms
3. **Extended Cultural Support:** Additional French variants and locales
4. **Validation Enhancement:** More sophisticated data validation rules

### Performance Optimizations

1. **Batch Processing:** Optimize multi-table generation workflows
2. **Format Caching:** Cache formatted values for repeated data
3. **XML Optimization:** Streamlined XML generation for large tables
4. **Memory Management:** Optimize memory usage for large datasets

### Integration Improvements

1. **Chart Integration:** Enhanced chart data export capabilities
2. **Template Integration:** Direct template parameter integration
3. **Validation Services:** Integration with data validation frameworks
4. **Audit Services:** Processing audit trail and logging

---

## Action items for system maintenance

1. **Regulatory Updates:** Monitor changes to Canadian financial reporting requirements
2. **Cultural Standards:** Validate French-Canadian formatting compliance with current standards
3. **Chart Compatibility:** Ensure axis marking remains compatible with Publisher chart generation
4. **Performance Monitoring:** Monitor table generation performance with large datasets
5. **Month Name Accuracy:** Validate French month abbreviations meet current linguistic standards
6. **Integration Testing:** Test table generation with complete document workflows
