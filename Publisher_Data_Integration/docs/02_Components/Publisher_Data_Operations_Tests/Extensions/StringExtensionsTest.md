# Tests — Extensions String Extensions Tests (analysis)

## One-line purpose

Comprehensive test suite for string extension methods providing financial data formatting, XML manipulation, date calculations, cultural localization, and specialized string processing functionality for document generation workflows.

---

## Files analyzed

* `Publisher_Data_Operations_Tests/Extensions/StringExtensionsTests.cs`

---

## What this code contains (high level)

1. **Financial Data Formatting** — Currency, percentage, and decimal formatting with cultural localization (English-Canadian, French-Canadian)
2. **Date and Time Processing** — Date parsing, age calculations, filing year determination, and consecutive year validation
3. **XML Content Manipulation** — XML cleaning, cell content processing, row insertion, and markup removal
4. **Data Validation and Conversion** — Boolean conversion, numeric validation, N/A detection, and type-safe string processing
5. **Specialized Business Logic** — Base26 encoding, field letter incrementing, and document-specific calculations

This test suite validates the extensive string processing infrastructure that enables sophisticated financial document generation with proper cultural formatting, regulatory compliance, and content quality assurance.

---

## Test Method Categories and Analysis

### Data Quality and Validation Tests

#### N/A and Blank Detection

* `[Theory] void isNaOrBlankTests(string value, bool expected)`

**Purpose:** Validates detection of N/A values and blank content across various formats and edge cases.

**Test Cases Analysis:**

* **Standard N/A Formats:** `"n/a "`, `"N/A"` → `true` (case-insensitive, whitespace tolerant)
* **Whitespace Variations:** `"\t \n\u00A0"` → `true` (tabs, newlines, non-breaking spaces)
* **Null Handling:** `null` → `true` (graceful null processing)
* **False Positives Prevention:** `"anything"`, `"anything N/A"` → `false` (partial matches rejected)
* **Special Whitespace:** `"\u00A0 n/A"` → `true` (non-breaking space + case insensitivity)
* **Quoted Values:** `"'N/A'"`, `"\"N/A\""` → `false` (quoted N/A values treated as content)

**Business Logic:** Enables consistent N/A detection across financial documents where missing data must be identified and handled appropriately.

#### String-to-Boolean Conversion

* `[Theory] void stringToBoolTests(string value, bool expected)`

**Test Cases Analysis:**

* **N/A Handling:** `"n/a"`, `"N/A"`, `""`, `null` → `false` (default to false for missing values)
* **Positive Indicators:** `"Y"`, `"Yes"`, `"y"`, `"1"`, `"True"`, `"true"` → `true`
* **Negative Indicators:** `"false"` → `false`
* **Whitespace Tolerance:** `" y"`, `" 1"` → `true` (leading whitespace trimmed)

**Pattern:** Comprehensive boolean parsing supporting multiple common formats used in financial data.

---

### Date and Time Processing Tests

#### Date Parsing and Validation

* `[Theory] void stringToDateTests(string value, int year, int month, int day)`

**Test Cases Analysis:**

* **Standard Format:** `"12/12/2019"` → `DateTime(2019, 12, 12)` (MM/dd/yyyy)
* **Leading Zeros:** `"01/01/2020"` → `DateTime(2020, 1, 1)` (zero-padded months/days)
* **Invalid Format Detection:** `" 01/03/2013"` → `DateTime(9999, 12, 31)` (leading space causes failure → MaxValue.Date)
* **ISO Format Support:** `"2013-10-25"` → `DateTime(2013, 10, 25)` (FSMRFP BAU/STATIC requirement)
* **Invalid Date Handling:** `"2021-09-31"` → `DateTime(9999, 12, 31)` (September 31st doesn't exist)

**Error Handling Pattern:** Invalid dates return `DateTime.MaxValue.Date` (9999-12-31) as sentinel value.

#### Age Calculations

* `[Theory] void stringAgeInCalendarYearsTests(string value, string inceptionDate, int expected)`

**Complex Business Logic Testing:**

* **Basic Calculation:** `"30/09/2020"` vs `"13/02/2009"` → `10` years (capped at maximum)
* **Same Date:** `"10/10/2020"` vs `"10/10/2020"` → `0` years
* **Maximum Age Cap:** `"30/09/2020"` vs `"30/06/2005"` → `10` years (15-year difference capped at 10)
* **Year Boundary Logic:** `"31/12/2020"` vs `"01/01/2020"` → `1` year
* **Partial Year Handling:** Various edge cases around year boundaries and partial years

**Business Context:** Appears to implement regulatory age calculation rules with maximum age limits for fund performance reporting.

#### Filing Year Calculation

* `[Theory] void stringFilingYearTests(string value, int expected)`

**Test Cases:**

* **Quarter 3:** `"30/09/2020"` → `2020` (filing year same as calendar year)
* **Quarter 4:** `"31/12/2020"` → `2021` (filing year advances to next year)

**Business Logic:** Regulatory filing year determination based on reporting periods.

---

### Financial Formatting Tests

#### Currency Formatting with Cultural Support

* `[Theory] void stringToCurrencyTests(string value, string culture, string expected)`

**French-Canadian (fr-CA) Formatting:**

* **Positive Values:** `"1000"` → `"1 000 $"` (space thousands separator, dollar sign after)
* **Negative Values:** `"-450"` → `"(450 $)"` (parentheses for negatives)

**English-Canadian (en-CA) Formatting:**

* **Standard Format:** `"999"` → `"$999"` (dollar sign before, no thousands separator)
* **Large Values:** `"2543.54"` → `"$2,544"` (rounded up, comma thousands separator)
* **Rounding Logic:** `"2543.45"` → `"$2,543"` (standard rounding rules)
* **Negative Values:** `"-2500"` → `"-$2,500"` (minus sign with dollar sign)

#### Decimal Currency Formatting

* `[Theory] void stringToCurrencyDecimalTests(string value, string culture, string expected)`

**Preserves Decimal Places:**

* **English:** `"11.59"` → `"$11.59"`
* **French:** `"1001.65"` → `"1 001,65 $"` (decimal comma, space separator)
* **French Negatives:** `"-1001.65"` → `"(1 001,65 $)"` (parentheses)

#### Percentage Formatting

* `[Theory] void stringToPercentTests(string value, string culture, int scale, string expected)`

**Cultural Formatting Rules:**

* **French Spacing:** `"0.856"` → `"0,856 %"` (decimal comma, space before %)
* **English No Space:** `"0.10"` → `"0.10%"` (decimal point, no space)
* **Large Numbers:** `"20000"` → `"20,000%"` (en-CA) vs `"20 000 %"` (fr-CA)
* **Scale Control:** `"11.65"` with scale `1` → `"11.7%"` (decimal place control)
* **Fixed Scale:** `"18"` with scale `1` → `"18.0%"` (force decimal places)

**Non-Breaking Spaces:** All tests replace regular spaces with `\u00A0` (non-breaking spaces) for proper typography.

---

### Advanced Date Logic Tests

#### Consecutive Years Validation

* `[Theory] void consecutiveYearsTests(string startdate, string endDate, int yearDif, bool expected)`

**Complex Business Logic:**

* **Exact Match:** `"15/05/2010"` to `"15/05/2020"` for `10` years → `true`
* **Day Precision:** `"15/05/2010"` to `"14/05/2020"` for `10` years → `false` (one day short)
* **Multi-Year Validation:** Various scenarios testing precise date arithmetic for regulatory compliance

**Business Context:** Likely validates fund performance periods for regulatory reporting requirements.

---

### XML Processing and Manipulation Tests

#### XML Content Cleaning

* `[Theory] void cleanXMLTests(string value, string expected)`

**Complex XML Normalization:**

* **Entity Encoding:** Converts special characters to HTML entities (`&` → `&amp;`)
* **Self-Closing Tags:** Converts `<br>` to `<br />`
* **Quote Handling:** `&#39;` for apostrophes
* **Space Normalization:** `&nbsp;` → `&#160;` (non-breaking space entity)

#### Cell Content Processing

* `[Theory] void cleanCellContentsTest(string value, string expected)`

**XML Cell Content Sanitization:**

* **Ampersand Encoding:** `"Total Number & Investments"` → `"Total Number &amp; Investments"`
* **Tag Encoding:** `<sup>4</sup>` → `&lt;sup&gt;4&lt;/sup&gt;`

#### Table Row Insertion

* `[Theory] void InsertHeaderRowTest(string value, string header, int row, string expected)`
* `[Theory] void InsertHeaderRowNewTest(string value, string header, DataRowInsert dri, string expected)`

**Advanced Table Manipulation:**

* **Position-Based Insertion:** Insert headers at specific row positions
* **Enum-Driven Insertion:** `DataRowInsert.AfterDescRepeat`, `DataRowInsert.FirstRow`, `DataRowInsert.ClearExtraColumns`
* **Complex Financial Tables:** Real fund data with subsidiary transactions and balance information

**Business Applications:** Dynamic table structure modification for regulatory reporting requirements.

---

### Specialized String Processing Tests

#### Base26 Encoding (Excel Column-Style)

* `[Theory] void ToBase26Tests(int number, string expected)`
* `[Theory] void FromBase26Tests(string number, int expected)`

**Excel Column Reference System:**

* **Basic Conversion:** `1` ↔ `"a"`, `26` ↔ `"z"`
* **Extended Range:** `27` ↔ `"aa"`, `18252` ↔ `"zyz"`
* **Error Handling:** `0`, `-1` → `null` (invalid inputs)

#### Field Letter Management

* `[Theory] void IncrementFieldLetterTests(string current, string first, string last, string expected)`

**Sequential Field Generation:**

* **Simple Increment:** `"a"` (range `"a"` to `"z"`) → `"b"`
* **Boundary Handling:** `"z"` (range `"a"` to `"z"`) → `null` (end of range)
* **Multi-Character Fields:** `"gg"` (range `"ii"` to `"iig"`) → `"iii"`

**Business Context:** Appears to support dynamic form field generation or template parameter management.

---

### Content Validation and Analysis Tests

#### Numeric Content Detection

* `[Theory] void isNumericTests(string value, bool expected)`

**Complex Numeric Pattern Recognition:**

* **Simple Cases:** `"Classic123"`, `"123Classic"` → `false` (mixed content)
* **Financial Formats:** `"#(123,000.67)"`, `"-123,000.67"`, `"+123,000.67"` → `true`
* **Formatted Numbers:** `"5,123,000.67"` → `true` (thousands separators)
* **Embedded Markup:** `"<b>$(42,002.23)</b>"` → `true` (financial data in markup)
* **Securities Format:** `"4.250%, 2025-02-15"` → `true` (special SOI format with space)

#### Word Containment Testing

* `[Theory] void containsWordsTests(string check, string[] elements, bool expected)`

**Intelligent Word Matching:**

* **Partial Matches:** `"Classic"` contains `"Class"` → `false` (partial word match rejected)
* **Exact Matches:** `"Class"` contains `"Class"` → `true`
* **Multi-Word Content:** `"Classic Series"` contains `["Class", "Series"]` → `true` (contains "Series")
* **Non-Matches:** `"Classic Tests"` vs `["Class", "Series", "Test"]` → `false`

---

## Integration patterns and business logic

### Cultural Localization Framework

The tests validate comprehensive bilingual support:

**French-Canadian Formatting:**

* Decimal comma instead of decimal point
* Space as thousands separator
* Dollar sign after amount
* Non-breaking spaces in formatting
* Parentheses for negative currency values

**English-Canadian Formatting:**

* Decimal point notation
* Comma as thousands separator
* Dollar sign before amount
* No space before percentage symbol
* Minus sign for negative values

### Regulatory Compliance Support

Multiple test categories support financial regulatory requirements:

**Date Calculations:** Age calculations with maximum limits, consecutive year validation, filing year determination
**Financial Formatting:** Consistent currency and percentage formatting across cultures
**Document Processing:** XML manipulation for regulatory document generation

### Document Generation Infrastructure

The tests validate infrastructure supporting complex document workflows:

**Template Processing:** Parameter replacement, field letter management, content validation
**Table Management:** Dynamic row insertion, column analysis, content cleaning
**Quality Assurance:** N/A detection, numeric validation, content sanitization

---

## Technical implementation considerations

### Performance Characteristics

* **String Processing:** Heavy use of regex patterns and string manipulation
* **Cultural Formatting:** .NET CultureInfo-based formatting with custom rules
* **XML Processing:** DOM-based manipulation for table structure changes
* **Date Arithmetic:** Complex date calculations with business rule enforcement

### Error Handling Strategies

* **Graceful Degradation:** Invalid dates return sentinel values (DateTime.MaxValue)
* **Null Safety:** Consistent null handling across all extension methods
* **Boundary Conditions:** Proper handling of edge cases (end of ranges, invalid formats)
* **Type Safety:** Safe conversion methods with fallback values

### Integration Points

* **Cultural Services:** Integration with .NET localization framework
* **XML Processing:** Coordination with XML table generation systems
* **Business Rules:** Implementation of regulatory and business logic requirements
* **Template Systems:** Support for dynamic content generation and parameter replacement

---

## Potential enhancements

### Performance Optimizations

1. **Regex Compilation:** Pre-compile frequently used regex patterns
2. **String Interning:** Cache commonly formatted values
3. **Culture Caching:** Cache CultureInfo instances for repeated formatting
4. **StringBuilder Usage:** Use StringBuilder for complex string building operations

### Functionality Extensions

1. **Additional Cultures:** Support for other Canadian French variants
2. **Currency Symbols:** Support for multiple currency types beyond CAD
3. **Date Format Flexibility:** Additional date parsing formats
4. **Validation Enhancement:** More sophisticated numeric and date validation

### Error Handling Improvements

1. **Detailed Error Messages:** More specific error information for debugging
2. **Exception Handling:** Structured exception handling for edge cases
3. **Validation Results:** Return validation results with error details
4. **Recovery Strategies:** Automatic correction of common formatting issues

---

## Action items for system maintenance

1. **Cultural Rule Updates:** Monitor changes to Canadian financial formatting requirements
2. **Performance Monitoring:** Track string processing performance on large documents
3. **Regex Pattern Review:** Validate and optimize regex patterns for accuracy and performance
4. **Business Rule Validation:** Ensure age calculation and filing year logic meets current regulations
5. **Unicode Handling:** Validate proper handling of special characters and entities
6. **Integration Testing:** Test string extensions with complete document generation workflows
