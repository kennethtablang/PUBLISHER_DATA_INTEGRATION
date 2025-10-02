# Extensions — StringExtensions (analysis)

## One-line purpose

Comprehensive collection of string manipulation utilities providing data type conversion, validation, formatting, XML processing, and specialized business logic for financial document processing in the PDI system.

---

## Files analyzed

* `Publisher_Data_Operations/Extensions/StringExtensions.cs`

---

## What this code contains (high level)

1. **Data Type Conversion** — Safe conversion methods for boolean, date, numeric, and currency values
2. **XML Processing Utilities** — Comprehensive XML validation, cleaning, and manipulation methods  
3. **Financial Formatting** — Currency, percentage, and decimal formatting with cultural localization
4. **Business Logic Helpers** — Specialized methods for age calculations, filing years, and document processing
5. **Text Processing** — Advanced string manipulation including base-26 conversion, field name processing, and content cleaning
6. **Validation Utilities** — Methods for checking data validity, format compliance, and content analysis

This extension class serves as the foundational utility layer for data transformation, validation, and formatting throughout the PDI pipeline.

---

## Classes, properties and methods (developer reference)

### `StringExtensions` - Static Extension Class

**Purpose:** Provides comprehensive string manipulation capabilities optimized for financial document processing and data transformation workflows.

---

## Data Type Conversion Methods

### Boolean Conversion

* `static bool ToBool(this string value)` — Flexible boolean conversion with multiple input formats
  * **Supported Values:** "Y*", "1", "True", standard boolean strings
  * **Default Behavior:** Returns `false` for null, empty, or unrecognized values
  * **Case Handling:** Case-insensitive comparison throughout

### Date Conversion Methods

* `static DateTime ToDate(this string value, DateTime nullValue)` — Multi-format date parsing
  * **Supported Formats:** "dd/MM/yyyy", "d/MM/yyyy", "yyyy-MM-dd" (ISO 8601)
  * **Culture:** Uses InvariantCulture for consistent parsing
  * **Fallback:** Returns specified nullValue for invalid dates

* `static DateTime ToDateUS(this string value, DateTime nullValue)` — US date format parsing
  * **Supported Formats:** "M/d/yyyy", "yyyy-MM-dd"
  * **Usage Context:** Specifically for BNY data processing

* `static bool IsDate(this string value)` — Date validation helper
  * Returns true if string can be parsed as valid date

### Numeric and Currency Formatting

#### Currency Formatting

* `static string ToCurrency(this string amount, string culture = "en-CA", string pattern = "C0")` — Cultural currency formatting
  * **Default:** Canadian English with no decimals
  * **Special Handling:** French-Canadian negative pattern correction (pattern 15 vs 8)
  * **Non-breaking Spaces:** Replaces regular spaces with `\u00A0`

* `static string ToCurrencyDecimal(this string amount, string culture = "en-CA")` — Currency with 2 decimal places

#### Percentage Formatting  

* `static string ToPercent(this string value, string culture = "en-CA", int scale = -1)` — Percentage formatting
  * **Assumption:** Input value is already percentage (not decimal)
  * **Scale Handling:** Preserves original decimal places or uses specified scale
  * **Cultural Patterns:** Forces pattern 1 for English locales
  * **Processing:** Divides by 100 to compensate for percentage vs decimal representation

#### Decimal Formatting

* `static string ToDecimal(this string value, int minScale = -1, int fixedScale = -1, string culture = "en-CA")` — Flexible decimal formatting
  * **Scale Options:** Minimum scale, fixed scale, or preserve original
  * **Cultural Support:** Localized number formatting
  * **Overloads:** Support for decimal and nullable decimal types

---

## Validation and Analysis Methods

### Content Validation

* `static bool IsNaOrBlank(this string value)` — Null, empty, or N/A detection
* `static bool IsNotNaOrBlank(this string value)` — Special "!N/A" pattern matching for scenarios
* `static bool IsReq(this string value)` — "Req" (Required) field detection
* `static bool IsNumeric(this string value)` — Numeric content validation with markup stripping

### XML Processing

* `static bool IsValidXML(this string value, bool addRoot = true)` — XML validation wrapper
* `static bool ContainsXML(this string value)` — XML content detection (checks for angle brackets)

---

## XML Processing and Cleanup Methods

### XML Content Manipulation

* `static string CleanXML(this string rawXML)` — Comprehensive XML cleaning
  * **Entity Encoding:** Handles &, <, >, ", ' characters
  * **Special Characters:** Non-breaking spaces, non-breaking hyphens
  * **Line Breaks:** Converts various BR tag formats to `<br />`
  * **Smart Ampersand Handling:** Only encodes & when not already escaped

* `static string ExcelTextClean(this string value)` — Excel-specific content cleaning
  * **Optimized for Excel Import:** Handles Excel-specific formatting quirks
  * **Line Break Normalization:** Converts newlines to `<br />` tags

### HTML/XML Conversion

* `static string IncomingHTMLtoXML(this string rawHTML)` — Convert HTML to XML-compatible format
* `static string OutgoingHTMLtoXML(this string rawHTML)` — Comprehensive HTML to XML conversion
* `static string IncomingXMLtoHTML(this string rawXML)` — Reverse XML to HTML conversion

---

## Text Manipulation and Processing

### Content Cleaning

* `static string RemoveMarkup(this string value, string tag = null)` — HTML/XML tag removal
* `static string RemoveHTML(this string value)` — Simple HTML tag stripping
* `static string StripTagsExceptAlpha(this string source)` — Extract alphabetic characters only
* `static string RemoveExceptAlpha(this string value)` — Alphabetic character extraction
* `static string RemoveExceptAlphaNumeric(this string value)` — Alphanumeric extraction

### String Replacement and Search

* `static string ReplaceCI(this string input, string search, string replacement)` — Case-insensitive replacement
* `static string ReplaceByDictionary(this string input, Dictionary<string, string> replace, bool appendBrackets = true)` — Dictionary-based replacement
  * **Template Processing:** Supports `<TOKEN>` style replacement
  * **Case Insensitive:** Handles case variations in token matching
  * **Regex-Based:** Uses compiled regex for efficient replacement

---

## Business Logic Methods

### Date and Age Calculations

* `static int AgeInCalendarYears(this string dataAsDate, string inceptionDate)` — Fund age calculation
  * **Business Logic:** Calendar year calculation for regulatory compliance
  * **Special Rules:** Year-end adjustment logic for December 31 dates
  * **Maximum Cap:** Returns maximum of 10 years

* `static int AgeInYearsBNY(this string dataAsDate, string inceptionDate)` — BNY-specific age calculation
  * **Different Logic:** Uses days-per-year calculation vs calendar years
  * **US Date Format:** Uses `ToDateUS` for date parsing

* `static int FilingYear(this string filingDate)` — Filing year determination
  * **Year-End Rule:** December 31 dates return following year
  * **Regulatory Compliance:** Matches filing year business rules

---

## Advanced Text Processing

### Base-26 Conversion (Excel Column Style)

* `static string ToBase26(this int number, char startChar = 'a')` — Convert integer to base-26 string
* `static int FromBase26(this string value, char startChar = 'a')` — Convert base-26 string to integer
* `static string Wrap26(this string current, char startChar = 'a')` — Handle character wrapping
* `static string UnWrap26(this string current, char startChar = 'a')` — Reverse character wrapping

### Field Name Processing

* `static string IncrementFieldLetter(this string current, string first = "a", string last = "zzz", bool restricted = true, char startLetter = 'a')` — Complex field name generation
  * **Restriction Logic:** Avoids 'f' and 'h' characters (business rule)
  * **Range Support:** Supports complex field name sequences
  * **Wrapping Logic:** Handles character sequence wrapping

### Number to Words Conversion

* `static string ConvertNumberToWords(int pValue, string culture = "en-CA")` — Bilingual number-to-words conversion
  * **English Support:** Full English number names
  * **French Support:** Complete French number names with complex rules
  * **Cultural Rules:** Handles French-specific numbering patterns (soixante-dix, quatre-vingt, etc.)

---

## XML Table Processing Methods

### Table Analysis and Manipulation

* `static string InsertHeaderRow(this string table, string headerRow, DataRowInsert dri = DataRowInsert.FirstRow)` — Advanced header insertion
  * **Multiple Modes:** FirstRow, AfterDescRepeat, AfterColumnChange, ClearExtraColumns
  * **Smart Logic:** Analyzes table structure for optimal header placement
  * **Column Matching:** Ensures header and data column alignment

### XML Content Analysis

* `static int XMLColumnCount(this string text)` — Count columns in XML table
* `static string XMLColumnValueByIndex(this string rowText, int column)` — Extract column value by position
* `static bool IsPositiveXMLColumnValueByIndex(this string rowText, int oneBasedColumn)` — Numeric sign detection
* `static int FindLastPositiveXMLRowIndex(this string tableText, int checkColumn, int removeRows, int requiredColumn = -1, int startPos = 0)` — Complex table analysis for positive/negative value separation

### Advanced Table Processing

* `static string ExtractXMLRows(this string tableText, int checkColumn, int removeRows, int direction = 1, int requiredColumn = -1, int startPos = 0)` — Conditional row extraction
  * **Direction Control:** Extract positive (direction = 1) or negative (direction = -1) values
  * **Required Column Logic:** Additional filtering based on required field presence
  * **Complex Business Rules:** Supports sophisticated table processing requirements

---

## Stream Processing Extensions

### Stream Utilities

* `static Stream ToStream(this string str)` — Convert string to UTF-8 memory stream
* `static string ToString(this Stream stream)` — Read stream content to string
* `static void CopyTo(this Stream fromStream, Stream toStream)` — Stream copying utility

---

## Integration patterns and performance considerations

### Cultural Localization Architecture

The extension methods provide comprehensive bilingual support:

* **Canadian English/French:** Primary focus for regulatory compliance
* **Cultural Formatting:** Currency, percentage, date formatting per locale
* **Number Conversion:** Full bilingual number-to-words conversion

### XML Processing Strategy

Multi-layered approach to XML handling:

* **Validation:** Integration with ParameterValidator for XML checking
* **Cleaning:** Multiple cleaning methods for different input sources
* **Analysis:** Advanced table structure analysis for business logic

### Performance Optimizations

* **Regex Caching:** Consider compiling frequently-used patterns
* **String Builder Usage:** Used in complex string building operations
* **Early Returns:** Validation methods use early return patterns
* **Memory Streams:** Efficient stream processing for large content

---

## Business rule complexity

### Financial Document Requirements

Many methods reflect specific financial industry requirements:

* **Age Calculations:** Multiple algorithms for different regulatory contexts
* **Filing Year Logic:** Year-end boundary handling for compliance
* **Currency Formatting:** Specific patterns for financial reporting
* **Percentage Handling:** Assumes percentage vs decimal input format

### Field Name Management

Complex field naming system for template processing:

* **Base-26 Logic:** Excel-style column naming system
* **Character Restrictions:** Business rule avoiding 'f' and 'h' characters
* **Range Expansion:** Support for complex field name sequences

### XML Table Processing

Sophisticated table analysis for financial data:

* **Positive/Negative Separation:** Common requirement in financial reporting
* **Header Insertion Logic:** Multiple strategies for table structure
* **Column Analysis:** Position-based data extraction and validation

---

## Action items for system maintenance

### Performance Optimization

1. **Regex Compilation:** Pre-compile frequently used regex patterns
2. **String Interning:** Consider interning for commonly used strings
3. **Memory Management:** Review memory usage in large text processing operations

### Code Organization

1. **Method Grouping:** Consider splitting into multiple focused extension classes
2. **Documentation:** Add XML documentation for complex business logic methods
3. **Unit Testing:** Ensure comprehensive test coverage for edge cases

### Maintenance Considerations

1. **Cultural Updates:** Monitor for changes in .NET cultural formatting patterns
2. **Regex Security:** Review regex patterns for potential ReDoS vulnerabilities
3. **Business Rule Evolution:** Track changes in financial reporting requirements
