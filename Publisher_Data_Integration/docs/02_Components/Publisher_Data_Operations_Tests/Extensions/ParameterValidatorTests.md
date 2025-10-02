# Tests — Extensions Parameter Validator Tests (analysis)

## One-line purpose

Test suite for ParameterValidator functionality that validates XML content integrity and scenario-based parameter configurations for template processing and document generation workflows.

---

## Files analyzed

* `Publisher_Data_Operations_Tests/Extensions/ParameterValidatorTests.cs`

---

## What this code contains (high level)

1. **XML Validation Testing** — Comprehensive validation of XML content structure, formatting, and template parameter compatibility
2. **Scenario Configuration Testing** — Validation of complex parameter scenario strings with conditional logic and value sets
3. **Template Parameter Integration** — Testing template replacement tokens and parameter validation within financial document contexts
4. **Content Cleanup Testing** — Validation of XML cleanup and normalization functionality
5. **Business Rule Validation** — Testing parameter constraints and business logic validation for document generation

This test suite validates the critical parameter validation infrastructure that ensures template integrity and document generation quality in the Publisher Data Integration system.

---

## Classes, properties and methods (developer reference)

### `ParameterValidatorTests` - Test Class

**Purpose:** Validates ParameterValidator functionality for XML content validation and scenario-based parameter configuration testing.

#### Test Infrastructure Setup

##### Constructor and Test Data

```csharp
public ParameterValidatorTests()
{
    Dictionary<string, string> validParams = new Dictionary<string, string>();
    validParams.Add("TestVal", string.Empty);
    validParams.Add("AnotherVal", string.Empty);
    validParams.Add("FinalVal", string.Empty);
    validParams.Add("IsMERAvailable", string.Empty);
    validParams.Add("FFDocAgeStatusID", string.Empty);
    validParams.Add("SeriesLetter", string.Empty);
    validParams.Add("PortfolioCharacteristicsTemplate", string.Empty);
    validParams.Add("FF31b", string.Empty);

    xmlValidator = new ParameterValidator(validParams);
}
```

**Valid Parameter Registry:**

* **TestVal, AnotherVal, FinalVal:** Generic test parameters
* **IsMERAvailable:** Management Expense Ratio availability flag
* **FFDocAgeStatusID:** Fund Fact document age status identifier (links to FFDocAge enum)
* **SeriesLetter:** Fund series designation (A, L, HW, etc.)
* **PortfolioCharacteristicsTemplate:** Template identifier for portfolio characteristics
* **FF31b:** Specific fund fact form parameter

**Integration Pattern:** Parameter dictionary defines allowed template replacement tokens, establishing the validation context for XML content and scenario strings.

#### Cleanup and Disposal

* `[Fact] void Dispose()`
  * **Test Isolation:** Ensures clean test state by nullifying validator instance
  * **Resource Management:** Demonstrates proper test cleanup patterns

---

## Core Validation Methods Testing

### XML Content Validation Tests

#### `[Theory] void isXMLValidXML(string value, bool cleanup, bool expected)`

**Purpose:** Validates XML content structure, template parameter usage, and content cleanup functionality across diverse financial document scenarios.

**Test Parameters:**

* `value` — XML or text content to validate
* `cleanup` — Whether to attempt content cleanup/normalization
* `expected` — Expected validation result

**Test Case Analysis:**

##### 1. Basic Template Parameter Validation

```csharp
[InlineData("<table><row><cell>[Replacement in Square]</cell><cell></cell></row><row><cell></cell><cell></cell></row></table>", true, true)]
```

* **Square Bracket Parameters:** Tests non-standard `[Replacement in Square]` parameter format
* **Table Structure:** Valid XML table with empty cells
* **Expected Result:** `true` — Valid XML with recognized parameter pattern

##### 2. Invalid XML Structure Detection

```csharp
[InlineData("<table><row><cell></cell></row>", true, false)]
[InlineData("<table><row><cell></cell></row><table>", true, false)]
```

* **Unclosed Tags:** Missing closing `</table>` tag
* **Malformed Structure:** Invalid nested table structure
* **Expected Results:** `false` — Structural XML validation failure

##### 3. Standard Template Parameter Format

```csharp
[InlineData("<table><row><cell><SeriesLetter></cell></row></table>", true, true)]
```

* **Angle Bracket Parameters:** Standard `<SeriesLetter>` template parameter format
* **Valid Parameter:** SeriesLetter exists in validParams dictionary
* **Expected Result:** `true` — Valid XML with recognized parameter

##### 4. Plain Text Content Validation

```csharp
[InlineData("This is a normal string", true, true)]
[InlineData("This is a replace <AnotherVal> string with <TestVal> replacement strings", true, true)]
```

* **Non-XML Content:** Plain text validation capability
* **Mixed Content:** Text with embedded template parameters
* **Parameter Recognition:** `<AnotherVal>` and `<TestVal>` are valid parameters
* **Expected Results:** `true` — Valid content regardless of XML structure

##### 5. Complex Financial Document Content

```csharp
[InlineData("<p>You don't pay these expenses directly. They affect you because they reduce the fund's returns.</p><p>The fund's expenses are made up of the management fee, fixed administration fee, other fund costs and trading costs.The series' annual management fee is <TestVal> of the series' value. The series' annual fixed administration fee is <AdminFeePercent> of the series' value.</p><p>Because this series is new, it's fund costs and trading costs are not yet available.</p>", true, false)]
```

* **Financial Disclosure Content:** Real fund expense disclosure language
* **Valid Parameter:** `<TestVal>` is recognized
* **Invalid Parameter:** `<AdminFeePercent>` is not in validParams dictionary
* **Expected Result:** `false` — Invalid due to unrecognized parameter

##### 6. Successful Cleanup Validation

```csharp
[InlineData("<p>You don't pay these expenses directly. They affect you because they reduce the fund's returns.</p><p>The fund's expenses are made up of the management <br>fee, fixed administration fee, other fund costs and trading costs.The series' annual management fee is <TestVal> of the series' value. The series' annual fixed administration fee is <IsMERAvailable> of the series' value.</p><p>Because this series is new, it's fund costs and trading costs are not yet available.</p>", true, true)]
```

* **HTML Break Tags:** `<br>` tags within content
* **Valid Parameters:** Both `<TestVal>` and `<IsMERAvailable>` are recognized
* **Cleanup Enabled:** `cleanup = true` allows normalization
* **Expected Result:** `true` — Valid after cleanup

##### 7. Complex Table Structure Validation

```csharp
[InlineData("<table> <row> <cell></cell> <cell>Return</cell> <cell>3 months ending</cell> <cell>If you invested $1,000 at the beginning of the period</cell> </row> <row> <row><cell>Best return</cell> <cell>12.9%</cell> <cell>June 30, 2020</cell> <cell>Your investment would rise to $1,129</cell> </row> <row> <cell>Worst return</cell> <cell>-15.1%</cell> <cell>March 31, 2020</cell> <cell>Your investment would drop to $849</cell> </row> </table>", true, false)]
```

* **Financial Performance Table:** Best/worst return data with specific dates and values
* **Nested Row Structure:** Invalid nested `<row>` tags
* **No Template Parameters:** No parameter replacement required
* **Expected Result:** `false` — Invalid due to malformed XML structure

##### 8. HTML Entity Validation

```csharp
[InlineData("<table>\n\t<row>\n\t\t<cell>This is a normal string with a broken non breaking space &nbsp here</cell>\n\t</row>\n</table>", false, false)]
```

* **HTML Entities:** `&nbsp;` (non-breaking space) entity
* **No Cleanup:** `cleanup = false` prevents entity normalization
* **Whitespace Characters:** Newlines and tabs in XML
* **Expected Result:** `false` — Invalid entity handling without cleanup

##### 9. Complex HTML Structure with Issues

```csharp
[InlineData("<p>Investors:</p></p>\r\n<ul><li>seeking a core (or focused) fund concentrated in developed market stocks outside the U.S.</li>\r\n<li>planning to hold their investment for the medium to long term</li></ul>", true, false)]
```

* **Malformed HTML:** Extra closing `</p>` tag
* **Investment Description:** Real investment objective language
* **Line Breaks:** Windows-style `\r\n` line endings
* **Expected Result:** `false` — Invalid due to malformed HTML structure

##### 10. Valid Complex HTML Content

```csharp
[InlineData("<p>Investor's:</p>\r\n<ul><li>\"seeking\" a core (or focused) fund concentrated in developed market stocks outside the U.S.</li>\r\n<li>planning to hold their investment for the medium to long term</li></ul>", true, true)]
```

* **Correct HTML Structure:** Proper paragraph and list formatting
* **Quoted Content:** Embedded quotes within list items
* **Investment Language:** Professional investment description content
* **Expected Result:** `true` — Valid HTML structure with proper formatting

**Validation Logic Patterns:**

1. **XML Structure Validation:** Checks for well-formed XML/HTML
2. **Parameter Recognition:** Validates template parameters against known parameter dictionary
3. **Content Cleanup:** Attempts normalization when cleanup flag is enabled
4. **Entity Handling:** Processes HTML entities during validation
5. **Business Content:** Handles real-world financial disclosure and investment language

---

### Scenario Configuration Validation Tests

#### `[Theory] void isScenarioValid(string value, bool expected)`

**Purpose:** Validates complex parameter scenario strings that define conditional logic and parameter value sets for document generation workflows.

**Scenario String Format Analysis:**

##### 1. Invalid Trailing Separator

```csharp
[InlineData("IsMERAvailable{0}&FFDocAgeStatusID{0,1,2,3}&SeriesLetter{A,L,HW,}", false)]
```

* **Parameter Chain:** Multiple parameters connected with `&`
* **Value Sets:** Each parameter has defined allowed values in `{}`
* **Trailing Comma:** `SeriesLetter{A,L,HW,}` has invalid trailing comma
* **Expected Result:** `false` — Syntax error due to trailing comma

##### 2. Valid Multi-Parameter Scenario

```csharp
[InlineData("IsMERAvailable{0}&FFDocAgeStatusID{0,1,2,3}&SeriesLetter{A,L,HW}", true)]
```

* **Parameter Validation:** All parameters (IsMERAvailable, FFDocAgeStatusID, SeriesLetter) are in validParams
* **Value Set Syntax:** Proper `{value1,value2,value3}` format
* **Chain Syntax:** Proper `&` parameter separator
* **Expected Result:** `true` — Valid scenario configuration

##### 3. Simple Single Parameter Scenario

```csharp
[InlineData("FFDocAgeStatusID{1,3}", true)]
```

* **Single Parameter:** Only FFDocAgeStatusID specified
* **Subset Values:** `{1,3}` represents specific document age statuses
* **Expected Result:** `true` — Valid single parameter scenario

##### 4. Malformed Bracket Syntax

```csharp
[InlineData("FFDocAgeStatusID{1,3", false)]
```

* **Missing Closing Bracket:** Incomplete value set definition
* **Expected Result:** `false` — Syntax error

##### 5. Empty Value Set

```csharp
[InlineData("FFDocAgeStatusID{}", false)]
```

* **Empty Braces:** No values specified for parameter
* **Expected Result:** `false` — Invalid empty value set

##### 6. Duplicate Parameter Definition

```csharp
[InlineData("FFDocAgeStatusID{1,3}FFDocAgeStatusID{2}", false)]
```

* **Parameter Repetition:** FFDocAgeStatusID defined twice
* **Missing Separator:** No `&` between parameter definitions
* **Expected Result:** `false` — Invalid duplicate parameter definition

##### 7. Unknown Parameter Validation

```csharp
[InlineData("Unknown{1,3}&AlsoUnknown{1}", false)]
```

* **Unrecognized Parameters:** Neither "Unknown" nor "AlsoUnknown" exist in validParams
* **Expected Result:** `false` — Invalid parameters not in allowed set

##### 8. Complex Value Set with Special Characters

```csharp
[InlineData("PortfolioCharacteristicsTemplate{Distinction & Inhance, template2} & FFDocAgeStatusID{4}", true)]
```

* **Special Characters:** Ampersand `&` within value "Distinction & Inhance"
* **Mixed Value Types:** Template names and numeric IDs
* **Spaces in Values:** "Distinction & Inhance" contains spaces
* **Expected Result:** `true` — Valid complex scenario with special characters

##### 9. Invalid Character in Value Set

```csharp
[InlineData("FF31b{ABF╣F}", false)]
```

* **Invalid Character:** `╣` (extended ASCII character) in value
* **Expected Result:** `false` — Invalid character encoding

##### 10. Valid Underscore in Value

```csharp
[InlineData("FF31b{ABF_F}", true)]
```

* **Underscore Character:** `_` is valid in parameter values
* **Expected Result:** `true` — Valid value format

**Scenario String Grammar:**

```note
Scenario := Parameter ('&' Parameter)*
Parameter := ParameterName '{' ValueList '}'
ValueList := Value (',' Value)* [',']?  // Trailing comma invalid
Value := [A-Za-z0-9_ &]+ // Letters, numbers, underscore, space, ampersand
```

**Business Logic Integration:**

* **Conditional Document Generation:** Scenarios define when specific templates or content should be used
* **Parameter Value Constraints:** Value sets define allowed values for each parameter
* **Multi-Parameter Logic:** Complex scenarios can specify multiple parameter combinations
* **Template Selection:** Scenarios drive template selection logic in document generation

---

## Business logic and integration patterns

### Template Parameter Validation

The tests validate critical template processing functionality:

**Parameter Registry Pattern:**

* Valid parameters defined in constructor dictionary
* Template content validated against known parameter set
* Prevents invalid parameter injection into generated documents

**Parameter Format Support:**

* **Angle Bracket Format:** `<ParameterName>` — Standard template parameter format
* **Square Bracket Format:** `[Replacement in Square]` — Alternative parameter format
* **Mixed Content:** Parameters embedded within text and XML content

### Financial Document Content Validation

Real-world content validation scenarios:

**Fund Disclosure Content:**

* Expense ratio explanations with parameter replacement
* Investment objective descriptions
* Performance data tables with historical returns
* Legal disclaimer and regulatory language

**Content Quality Assurance:**

* XML structure validation prevents malformed output
* HTML entity handling ensures proper rendering
* Parameter validation prevents missing replacement values

### Scenario-Based Document Generation

Complex conditional logic support:

**Multi-Parameter Scenarios:**

* Fund age status combinations (FFDocAgeStatusID)
* Series letter variations (A, L, HW series)
* MER availability flags (IsMERAvailable)
* Template selection criteria (PortfolioCharacteristicsTemplate)

**Business Rule Enforcement:**

* Parameter value constraints
* Required parameter combinations
* Template compatibility validation

---

## Technical implementation considerations

### Performance Characteristics

* **Parameter Dictionary Lookup:** O(1) parameter existence validation
* **XML Parsing:** DOM-based validation for structure checking
* **String Pattern Matching:** Regular expression-based scenario parsing
* **Content Cleanup:** Optional normalization processing

### Error Handling Strategy

* **Graceful Validation:** Boolean return values rather than exceptions
* **Comprehensive Edge Cases:** Null, empty, malformed input handling
* **Business Rule Validation:** Parameter constraint enforcement
* **Content Normalization:** Optional cleanup for recoverable issues

### Integration Points

* **Template Processing:** Parameter validation feeds template replacement engine
* **Document Generation:** Scenario validation drives conditional content generation
* **Content Management:** XML validation ensures output quality
* **Business Logic:** Parameter constraints enforce business rules

---

## Test coverage and quality assurance

### XML Validation Coverage

**Structure Validation:**

* Well-formed XML/HTML checking
* Nested tag validation
* Self-closing tag handling
* Entity encoding validation

**Content Validation:**

* Template parameter recognition
* Mixed content handling (XML + text)
* Financial disclosure language
* Investment description content

**Error Scenarios:**

* Malformed XML structures
* Unknown template parameters
* Invalid HTML entity usage
* Unclosed tag detection

### Scenario Validation Coverage

**Syntax Validation:**

* Parameter chain syntax
* Value set bracket matching
* Separator validation (`&` between parameters)
* Trailing comma detection

**Business Rule Validation:**

* Parameter existence checking
* Value constraint enforcement
* Duplicate parameter detection
* Character encoding validation

**Complex Scenarios:**

* Multi-parameter combinations
* Special characters in values
* Template name handling
* Numeric parameter values

### Edge Case Testing

**Input Validation:**

* Empty and null scenario strings
* Malformed bracket syntax
* Invalid parameter names
* Character encoding edge cases

**Business Logic:**

* Parameter dependency validation
* Value set constraint checking
* Template compatibility verification
* Conditional logic validation

---

## Potential enhancements

### Validation Improvements

1. **Schema Validation:** XSD-based XML structure validation
2. **Parameter Dependencies:** Cross-parameter dependency checking
3. **Value Range Validation:** Numeric parameter range constraints
4. **Template Compatibility:** Parameter-template compatibility matrix

### Error Reporting Enhancements

1. **Detailed Error Messages:** Specific validation failure reasons
2. **Error Location:** Line/column position for XML validation errors
3. **Suggestion Engine:** Correction suggestions for common errors
4. **Validation Reports:** Comprehensive validation result reporting

### Performance Optimizations

1. **Validation Caching:** Cache validation results for repeated content
2. **Incremental Validation:** Validate only changed portions
3. **Parallel Validation:** Multi-threaded validation for large documents
4. **Compiled Regex:** Pre-compiled regex patterns for scenario parsing

### Business Logic Extensions

1. **Dynamic Parameters:** Runtime parameter registry updates
2. **Conditional Scenarios:** Complex conditional logic support
3. **Parameter Inheritance:** Hierarchical parameter relationships
4. **Validation Profiles:** Different validation rules per document type

---

## Action items for system maintenance

1. **Parameter Registry Management:** Maintain current list of valid template parameters
2. **Content Pattern Updates:** Keep validation patterns current with template changes
3. **Performance Monitoring:** Monitor validation performance on large documents
4. **Error Pattern Analysis:** Analyze common validation failures for improvement opportunities
5. **Business Rule Updates:** Update parameter constraints as business rules evolve
6. **Integration Testing:** Validate parameter validator integration with template processing pipeline
