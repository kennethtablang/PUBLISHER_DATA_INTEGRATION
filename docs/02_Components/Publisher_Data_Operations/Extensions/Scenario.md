# Extensions — Scenario (analysis)

## One-line purpose

Implements a sophisticated conditional content system using flag-based scenario matching to select appropriate English/French text based on document field values, enabling dynamic content generation for regulatory documents.

---

## Files analyzed

* `Publisher_Data_Operations/Extensions/Scenario.cs`

---

## What this code contains (high level)

1. **Flag-Based Conditional Logic** — Complex parsing and matching system for conditional content rules
2. **Scenario Matching Engine** — Intelligent selection of content based on document field values
3. **Ranking and Prioritization** — Sophisticated ordering system to select most specific matching scenario
4. **Bilingual Content Management** — Parallel English/French content with conditional logic
5. **Dynamic Content Selection** — Runtime content selection based on document characteristics

This system enables regulatory documents to dynamically adapt their content based on fund characteristics, document types, and other contextual factors while maintaining compliance requirements.

---

## Classes, properties and methods (developer reference)

### `Flag` - Individual Condition Class

**Purpose:** Represents a single conditional flag with associated values for scenario matching.

#### Properties & Constructor

* `string FieldName { get; set; }` — Field name to match against document properties
* `List<string> Values { get; set; }` — List of acceptable values for this field
* `Flag(string fieldName, List<string> values)` — Constructor initializing field name and values

**Usage Pattern:**

```csharp
var flag = new Flag("DocumentType", new List<string> { "FF", "ETF", "MRFP" });
```

### `FlagEqualityComparer : IEqualityComparer<Flag>` - Comparison Logic

**Purpose:** Enables efficient Flag comparison and deduplication based on field names.

#### Methods

* `bool Equals(Flag x, Flag y)` — Flags equal if FieldName properties match
* `int GetHashCode(Flag obj)` — Hash based on FieldName for efficient collections

### `Flags : List<Flag>` - Flag Collection and Parsing

**Purpose:** Collection of flags with parsing capabilities and scenario string generation.

#### Constants (Scenario Syntax Definition)

* `const string ValStart = "{"` — Value list start delimiter
* `const string ValEnd = "}"` — Value list end delimiter  
* `const string ValSeperator = ","` — Separator between values within a flag
* `const string FlagSeparator = "&"` — Separator between different flags
* `static string ScenarioMatch` — Regex pattern for parsing scenario syntax

**Scenario Syntax Examples:**

```csharp
DocumentType{FF,ETF}&IsProforma{true}
ClientType{Institutional}&AssetClass{Equity,Bond}
default  // Special case for default scenarios
```

#### Constructors

* `Flags()` — Empty collection constructor
* `Flags(string scenarioText)` — Parse scenario string into flag collection

#### Core Methods

##### Parsing Operations

* `void ParseScenario(string scenarioText)` — Main parsing method using regex to extract flags and values
  * Handles complex scenario strings with multiple flags
  * Creates Flag objects from parsed components
  * Falls back to single flag without values for simple scenarios

##### Utility Methods  

* `bool IsDefault()` — Checks if scenario contains "default" (case-insensitive)
* `List<string> DistinctFieldNames()` — Returns unique field names from all flags
* `Flag Find(string fieldName)` — Locates flag by field name
* `override string ToString()` — Reconstructs scenario string from flag collection

**ToString() Output Format:**

* Flags with values: `FieldName{value1,value2,value3}&`
* Flags without values: `FieldName&`
* Removes trailing separator

### `Scenario` - Content Container Class

**Purpose:** Combines conditional logic (Flags) with bilingual content and metadata.

#### Properties

* `string EnglishText { get; set; }` — English version of conditional content
* `string FrenchText { get; set; }` — French version of conditional content  
* `DateTime LastUpdated { get; set; }` — Timestamp for sorting scenarios with same rank
* `Flags FlagList { get; set; }` — Collection of conditional flags for matching

#### Constructor

* `Scenario()` — Default empty constructor
* `Scenario(string scenarioText, string english, string french, DateTime lastUpdated)` — Full initialization constructor

#### Core Method

##### Matching Logic

* `bool MatchFields(Dictionary<string, string> docFields)` — Primary matching algorithm
  
**Matching Rules:**

1. **Default Scenarios:** Always match if `FlagList.IsDefault()` returns true
2. **Field Existence:** All flag field names must exist in document fields
3. **Value Matching:** Document field values must match at least one flag value (case-insensitive)
4. **N/A Handling:** Special logic for N/A, blank, and null value matching
5. **NotN/A Logic:** Support for "not N/A" matching conditions

**Matching Algorithm:**

```csharp
foreach (Flag f in FlagList) {
    val = docFields[f.FieldName];
    if (!f.Values.Any(item => 
        item.ToUpper() == val.ToUpper() || 
        (item.IsNaOrBlank() && val.IsNaOrBlank()) || 
        (item.IsNotNaOrBlank() && !val.IsNaOrBlank()))) {
        return false; // Fail if any flag doesn't match
    }
}
```

##### Ranking and Comparison

* `int RankCount()` — Returns flag count for ranking (-1 for default scenarios)
* `int CompareTo(Scenario other)` — Custom comparison for sorting
  * Primary sort: Flag count (more specific scenarios first)
  * Secondary sort: LastUpdated timestamp for tie-breaking

### `Scenarios : List<Scenario>` - Scenario Collection

**Purpose:** Collection of all scenarios with ranking and analysis capabilities.

#### Method

##### Ordering Operations

* `void RankOrder()` — Sorts scenarios by rank count (highest to lowest)
  * More specific scenarios (more flags) appear first
  * Default scenarios appear last
  * Recent updates break ties

##### Analysis Methods

* `List<string> AllDistinctFieldNames(Dictionary<string, string> stageFields = null)` — Extract all unique field names
  * Collects field names from all scenario flags
  * Optionally excludes fields already provided in stageFields
  * Used for determining required document fields

---

## Business logic and integration patterns

### Content Selection Strategy

The system implements a sophisticated content selection algorithm:

1. **Rank Ordering:** More specific scenarios (more flags) have priority
2. **Best Match Selection:** First matching scenario in ranked order is selected
3. **Default Fallback:** Default scenarios provide fallback content
4. **Temporal Precedence:** Recent updates take priority for equal-rank scenarios

### Document Field Integration

Scenarios integrate with document processing pipeline:

* **Field Discovery:** `AllDistinctFieldNames()` identifies required document properties
* **Dynamic Matching:** Runtime evaluation against actual document field values
* **Validation Support:** Missing required fields can be detected before processing

### Regulatory Compliance Support

The system enables compliance-driven content variation:

* **Document Type Variations:** Different content for FF vs MRFP vs ETF documents
* **Client-Specific Content:** Institutional vs Retail investor content
* **Jurisdiction-Specific Rules:** Regional regulatory requirement handling
* **Fund Characteristic Adaptation:** Content varies by asset class, structure, etc.

---

## Technical implementation considerations

### Performance Characteristics

* **Regex Parsing:** Complex regex for scenario parsing may impact performance on large scenario sets
* **Linear Matching:** Scenario matching is O(n) where n is number of scenarios
* **String Comparison:** Case-insensitive matching throughout system
* **Memory Usage:** Each scenario maintains full English/French content strings

### Error Handling Strategy

* **Missing Fields:** Logs errors but continues processing other flags
* **Parse Failures:** Graceful fallback to simple flag creation
* **Invalid Scenarios:** System continues with remaining valid scenarios

### Extensibility Patterns

The current design could be extended to support:

* **Numeric Comparisons:** Greater than, less than operations for numeric fields
* **Regular Expression Values:** Pattern matching beyond exact string matching
* **Nested Logic:** AND/OR combinations within individual flags
* **Date-Based Conditions:** Time-sensitive content variations

---

## Integration with broader PDI system

### Template Processing Integration

Scenarios work with template parameter replacement:

* Scenario text can contain template parameters
* Parameter replacement occurs after scenario selection
* Enables dynamic content with variable substitution

### Database Integration  

Scenarios are typically stored in database tables:

* `pdi_Data_Staging_STATIC_Content_Scenario` table structure
* Client/Document Type specific scenario configurations  
* Version control through LastUpdated timestamps

### Translation System Integration

Parallel English/French content supports:

* Regulatory bilingual requirements (Canadian market)
* Consistent terminology across languages
* Quality assurance for translation accuracy

---

## Potential system enhancements

### Performance Optimizations

1. **Compiled Regex:** Pre-compile regex patterns for better performance
2. **Indexed Matching:** Create field-based indices for faster scenario lookup
3. **Caching Layer:** Cache compiled scenarios and matching results
4. **Lazy Evaluation:** Parse scenarios only when needed

### Functionality Extensions

1. **Numeric Comparisons:** `AssetValue{>1000000}` for threshold-based content
2. **Date Ranges:** `AsAtDate{2023-01-01:2023-12-31}` for time-based content
3. **Nested Logic:** `(DocumentType{FF}&ClientType{Retail})|(DocumentType{ETF})`
4. **Validation Rules:** Built-in validation for scenario syntax and field references

### Monitoring and Debugging

1. **Scenario Usage Tracking:** Monitor which scenarios are selected most frequently
2. **Match Debugging:** Detailed logging of scenario evaluation process
3. **Performance Metrics:** Track scenario parsing and matching performance
4. **Content Analytics:** Analyze content variation patterns

---

## Action items for system maintenance

1. **Performance Profiling:** Monitor regex parsing performance on large scenario sets
2. **Scenario Optimization:** Review scenario specificity to minimize evaluation overhead  
3. **Error Handling Enhancement:** Improve error reporting for scenario parsing failures
4. **Documentation Standards:** Create formal documentation for scenario syntax rules
5. **Testing Coverage:** Ensure comprehensive testing of edge cases in matching logic
6. **Version Management:** Implement scenario versioning for content change tracking
