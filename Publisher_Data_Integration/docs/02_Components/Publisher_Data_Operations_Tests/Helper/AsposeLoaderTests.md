# Tests — Helper AsposeLoader Tests (analysis)

## One-line purpose

Test suite for AsposeLoader utility methods that handle Excel column name processing, row number extraction, and Excel address conversion for financial document template field mapping and data extraction workflows.

---

## Files analyzed

* `Publisher_Data_Operations_Tests/Helper/AsposeLoaderTests.cs`

---

## What this code contains (high level)

1. **Column Name Processing** — Validation of Excel column header normalization and cleanup for template field mapping
2. **Row Number Extraction** — Testing numeric value extraction from complex field names for data positioning
3. **Excel Address Conversion** — Bidirectional conversion between Excel addresses (A3, I7) and zero-based row/column indices
4. **Template Field Processing** — Validation of field name processing for template parameter extraction and mapping

This test suite validates the utility methods that enable AsposeLoader to process Excel templates and extract structured data for financial document generation workflows.

---

## Classes, properties and methods (developer reference)

### `AsposeLoaderTests` - Test Class

**Purpose:** Validates AsposeLoader static utility methods for Excel template processing and address manipulation.

#### Test Infrastructure Setup

##### Constructor

```csharp
public AsposeLoaderTests()
{
    asposeLoader = new AsposeLoader("");
}
```

**Instance Creation:** Creates AsposeLoader instance with empty parameter, suggesting the constructor accepts a parameter (likely file path or configuration) that isn't needed for utility method testing.

---

## Core Utility Method Testing

### Column Name Processing Tests

#### `[Theory] void getColumnNameTests(string value, string expected)`

**Purpose:** Validates Excel column header normalization for template field mapping and parameter extraction.

**Test Case Analysis:**

##### 1. Multi-Line Header with Row Reference

```csharp
[InlineData("Trailing Commission Table\n(Header)\nRow 1_E30b", "TrailingCommissionTable(Header)Row01_E30b")]
```

* **Newline Removal:** `\n` characters stripped from multi-line headers
* **Whitespace Elimination:** Spaces removed entirely
* **Number Formatting:** "Row 1" → "Row01" (zero-padded to 2 digits)
* **Template Code Preservation:** "_E30b" suffix maintained for field identification

##### 2. Double-Digit Row Numbers

```csharp
[InlineData("Trailing Commission Table\n(Header)\nRow 10_E30b", "TrailingCommissionTable(Header)Row10_E30b")]
```

* **Consistent Formatting:** "Row 10" remains "Row10" (already 2 digits)
* **Same Processing Logic:** Newlines removed, spaces eliminated, template code preserved

##### 3. French Field Processing

```csharp
[InlineData("Custom Text French\nField 9_E209c_FR", "CustomTextFrenchField09_E209c_FR")]
```

* **Language Handling:** French field names processed consistently
* **Number Padding:** "Field 9" → "Field09" (zero-padded)
* **Language Code Preservation:** "_FR" suffix maintained for localization

##### 4. Complex Descriptive Names

```csharp
[InlineData("Current year\nminus 8 years\nreturn_FP9", "CurrentYearMinus08YearsReturn_FP9")]
```

* **Multi-Word Processing:** Complex descriptions converted to camelCase-style
* **Number Extraction:** "minus 8 years" → "Minus08Years"
* **Context Preservation:** Maintains semantic meaning while normalizing format

##### 5. Template Parameter Removal

```csharp
[InlineData("FundCode_<FundCode>", "FundCode")]
```

* **Template Parameter Stripping:** `<FundCode>` template parameter removed entirely
* **Clean Field Names:** Results in simple field name without template syntax

**Normalization Rules Inferred:**

1. Remove all `\n` (newline) characters
2. Remove all spaces
3. Zero-pad single-digit numbers to 2 digits (1 → 01, 9 → 09)
4. Remove template parameters in angle brackets (`<...>`)
5. Preserve underscore separators and suffix codes
6. Maintain camelCase-style capitalization

---

### Row Number Extraction Tests

#### `[Theory] void getRowNumberTests(string value, int expected)`

**Purpose:** Validates numeric value extraction from complex field names for data positioning and template mapping.

**Test Case Analysis:**

##### 1. Single-Digit Row Extraction

```csharp
[InlineData("Trailing Commission Table\n(Header)\nRow 1_E30b", 1)]
```

* **Pattern Recognition:** Identifies "Row 1" pattern and extracts integer value
* **Consistent with Column Test:** Same input as column name test

##### 2. Double-Digit Row Extraction

```csharp
[InlineData("Trailing Commission Table\n(Header)\nRow 10_E30b", 10)]
```

* **Multi-Digit Handling:** Properly extracts "10" from "Row 10" pattern

##### 3. Field Number Extraction

```csharp
[InlineData("Custom Text French\nField 9_E209c_FR", 9)]
```

* **Pattern Variation:** Extracts number from "Field 9" pattern (not just "Row")
* **Language Agnostic:** Works with French field descriptions

##### 4. Complex Text Number Extraction

```csharp
[InlineData("Current year\nminus 8 years\nreturn_FP9", 8)]
```

* **Context-Aware Extraction:** Identifies "8" from "minus 8 years" phrase
* **Multiple Number Handling:** Correctly selects relevant number (8, not 9 from FP9)

##### 5. No Number Scenario

```csharp
[InlineData("FundCode_<FundCode>", 0)]
```

* **Default Value:** Returns 0 when no recognizable number pattern found
* **Graceful Handling:** No exception thrown for non-numeric field names

**Number Extraction Logic:**

* Searches for "Row X", "Field X", or "minus X years" patterns
* Extracts the numeric value from recognized patterns
* Returns 0 as default when no pattern matches
* Handles both single and multi-digit numbers

---

### Excel Address Conversion Tests

#### `[Theory] void convertAddressTests(string address, bool success, int row, int col)`

**Purpose:** Validates conversion from Excel address notation (A3, I7) to zero-based row/column indices.

**Test Case Analysis:**

##### 1. Valid Single-Character Column Address

```csharp
[InlineData("A3", true, 2, 0)]
```

* **Column Conversion:** "A" → column index 0 (zero-based)
* **Row Conversion:** "3" → row index 2 (zero-based, so 3-1=2)
* **Success Flag:** Returns true for valid address format

##### 2. Valid Multi-Character Column Address

```csharp
[InlineData("I7", true, 6, 8)]
```

* **Column Conversion:** "I" → column index 8 (A=0, B=1, ..., I=8)
* **Row Conversion:** "7" → row index 6 (zero-based, so 7-1=6)
* **Extended Range:** Handles columns beyond basic A-Z range

##### 3. Invalid Format - Number First

```csharp
[InlineData("7I", false, -1, -1)]
```

* **Format Validation:** Rejects addresses with number before letter
* **Error Indicators:** Returns false with -1 values for invalid format

##### 4. Invalid Format - Range Reference

```csharp
[InlineData("A3:I7", false, -1, -1)]
```

* **Range Rejection:** Does not handle range references (A3:I7)
* **Single Cell Only:** Designed for individual cell addresses only

##### 5. Invalid Format - Mixed Characters

```csharp
[InlineData("A1C1", false, -1, -1)]
```

* **Complex Format Rejection:** Does not handle R1C1-style references
* **Standard Excel Only:** Expects standard A1-style notation

**Address Conversion Rules:**

* Accepts only standard Excel address format (letter(s) followed by number)
* Converts to zero-based indexing (A1 → row 0, col 0)
* Returns success flag with actual indices or failure flag with -1 values
* Does not handle range references or alternative notation systems

#### `[Theory] void convertIndextoAddress(int row, int col, string address)`

**Purpose:** Validates reverse conversion from zero-based indices to Excel address notation.

**Test Case Analysis:**

##### 1. Valid Row/Column Combination

```csharp
[InlineData(2, 0, "A3")]
```

* **Reverse Conversion:** row 2, col 0 → "A3" (adding 1 for one-based Excel rows)
* **Column Mapping:** column 0 → "A"

##### 2. Column-Only Conversion

```csharp
[InlineData(-1, 0, "A")]
```

* **Row -1 Handling:** Invalid row (-1) results in column-only address
* **Partial Address:** Returns just the column letter when row is invalid

##### 3. Row-Only Conversion  

```csharp
[InlineData(1, -1, "2")]
```

* **Column -1 Handling:** Invalid column (-1) results in row-only address
* **Partial Address:** Returns just the row number when column is invalid

##### 4. Both Invalid Indices

```csharp
[InlineData(-1, -1, null)]
```

* **Complete Failure:** Both invalid indices result in null return
* **Error Handling:** Graceful failure with null rather than exception

**Reverse Conversion Logic:**

* Converts zero-based indices back to Excel address format
* Handles partial conversions (column-only or row-only)
* Returns null for completely invalid input
* Adds 1 to row indices for Excel's one-based row numbering

---

## Integration patterns and business logic

### Excel Template Processing

The utility methods support Excel template processing workflows:

**Field Name Normalization:**

* Converts complex multi-line headers to clean field names
* Enables consistent template parameter mapping
* Supports multilingual field processing (English/French)

**Data Positioning:**

* Row number extraction enables precise data placement
* Address conversion supports cell-level data manipulation
* Zero-based indexing integrates with programming APIs

### Template Parameter Management

Field processing supports template parameter workflows:

**Parameter Extraction:**

* Removes template syntax (`<FundCode>`) from field names
* Maintains field identification codes (_E30b,_FR, _FP9)
* Preserves semantic field names for mapping

**Localization Support:**

* Handles French field names consistently
* Maintains language-specific suffixes (_FR)
* Enables bilingual template processing

### Excel Integration Framework

Address conversion supports Aspose.Cells integration:

**API Compatibility:**

* Zero-based indexing matches Aspose.Cells API requirements
* Bidirectional conversion enables flexible cell access
* Error handling prevents API exceptions from invalid addresses

**Template Mapping:**

* Enables dynamic cell address generation
* Supports programmatic Excel manipulation
* Facilitates data extraction from template structures

---

## Technical implementation considerations

### String Processing Performance

* **Regex Usage:** Likely uses regular expressions for number extraction and pattern matching
* **String Manipulation:** Multiple string operations for normalization and cleanup
* **Memory Usage:** Efficient string processing for template field names

### Error Handling Strategy

* **Graceful Degradation:** Invalid addresses return failure flags rather than exceptions
* **Default Values:** Meaningful defaults (0, null, -1) for error conditions
* **Validation Logic:** Input validation before processing to prevent errors

### Integration Requirements

* **Aspose.Cells Compatibility:** Zero-based indexing matches library requirements
* **Template System Integration:** Clean field names enable template parameter mapping
* **Multilingual Support:** Consistent processing across language variants

---

## Test coverage and quality assurance

### Field Name Processing Coverage

**Format Variations:**

* Multi-line headers with newlines
* Complex descriptive field names
* Numbered field sequences (Row/Field patterns)
* Template parameter syntax

**Language Support:**

* English field names
* French field names with _FR suffixes
* Mixed language processing consistency

### Address Conversion Coverage

**Valid Scenarios:**

* Single-character column addresses (A1-Z99)
* Multi-character column addresses beyond Z
* Standard Excel address format validation

**Error Scenarios:**

* Invalid format detection (7I, A3:I7, A1C1)
* Partial address handling (column-only, row-only)
* Complete failure scenarios with appropriate null returns

### Number Extraction Coverage

**Pattern Recognition:**

* "Row X" patterns with various row numbers
* "Field X" patterns for field identification  
* Complex text with embedded numbers ("minus 8 years")
* No-number scenarios with default values

---

## Potential enhancements

### Functionality Extensions

1. **Extended Column Support:** Handle column addresses beyond Z (AA, AB, etc.)
2. **Range Processing:** Support for range references (A1:C3)
3. **Alternative Notations:** R1C1 reference style support
4. **Validation Enhancement:** More sophisticated address format validation

### Performance Optimizations

1. **Regex Compilation:** Pre-compile regular expressions for better performance
2. **Caching:** Cache normalized field names for repeated processing
3. **String Pooling:** Optimize string creation for common field names
4. **Batch Processing:** Process multiple addresses/fields in single operations

### Error Handling Improvements

1. **Detailed Error Messages:** Specific error information for debugging
2. **Exception Types:** Structured exceptions for different error conditions
3. **Validation Results:** Return validation details with success/failure
4. **Recovery Suggestions:** Automatic correction for common formatting issues

---

## Action items for system maintenance

1. **Template Compatibility:** Ensure field name processing remains compatible with template changes
2. **Performance Monitoring:** Monitor string processing performance on large templates
3. **Address Range Testing:** Validate address conversion with extended Excel column ranges
4. **Localization Updates:** Keep multilingual field processing current with template requirements
5. **Integration Testing:** Validate utility methods with complete AsposeLoader workflows
6. **Error Pattern Analysis:** Analyze common processing errors for improvement opportunities
