# Component: HtmlDiff (BatchProcessorService)

**One-line purpose**  
HTML-aware text differencing engine that generates redline markup (insertions/deletions) by tokenizing content into words and tags, matching common sequences, and wrapping changes in semantic HTML markup for document comparison workflows.

---

## Files analyzed

* `BatchProcessorService/HtmlDiff.cs`

---

## What this code contains (high level)

1. **Text Sanitization Pipeline** — Standardizes HTML tags, escapes malformed markup, removes adjacent empty tags
2. **Word-Level Tokenization** — Splits HTML content into arrays of words, whitespace, and complete tag tokens
3. **Longest Common Subsequence (LCS) Matching** — Recursive block-matching algorithm to identify unchanged content
4. **Diff Operation Generation** — Determines insert/delete/replace/equal operations from matched blocks
5. **Semantic Markup Output** — Wraps changed content in `<ins>` and `<del>` tags while preserving HTML structure

This class serves as the core differencing engine for document comparison, enabling redline/blackline functionality between original and modified versions of HTML content.

---

## Classes, properties and methods (developer reference)

### `HtmlDiff` - Main Differencing Class

**Purpose:** Compares two HTML text strings and generates markup highlighting insertions, deletions, and modifications.

#### Public Static Methods

##### Primary Entry Point

* `string redlineText(string oldText, string newText, bool stripHtml)` — Static factory method for generating redline comparison
  * **Parameters:**
    * `oldText` — Original version of content
    * `newText` — Modified version of content
    * `stripHtml` — **IGNORED** parameter (legacy signature)
  * **Returns:** HTML string with `<ins>` and `<del>` tags marking changes
  * **Usage Pattern:** Primary public API for external callers

##### Text Preparation

* `string sanitizeText(string text)` — Normalizes non-breaking spaces and special characters
  * **Replacements:**
    * `&nbsp;` → standard space
    * `&#160;` → standard space
    * Character 160 → standard space
  * **Purpose:** Ensures consistent whitespace handling across different HTML encodings

* `string standardizeTags(string text)` — Converts similar tags to canonical forms
  * **Transformations:**
    * `<b>` → `<strong>`
    * `<i>` → `<em>`
    * `<*>` → `&lt;*&gt;` (escapes client-specific weird tags)
  * **Rationale:** Reduces tag variation to improve matching accuracy

#### Private Core Properties

##### Source Content

* `string oldText` — Sanitized original text after preprocessing
* `string newText` — Sanitized modified text after preprocessing

##### Tokenized Content

* `string[] oldWords` — Original text split into word/tag/whitespace tokens
* `string[] newWords` — Modified text split into word/tag/whitespace tokens

##### Matching Infrastructure

* `Dictionary<string, List<int>> wordIndices` — Index mapping newWords tokens to their positions for fast lookup
* `StringBuilder content` — Accumulator for final output markup

##### Special Case Handling

* `string[] specialCaseOpeningTags` — Regex patterns for format tags: `<strong>`, `<b>`, `<i>`, `<big>`, `<small>`, `<u>`, `<sub>`, `<sup>`, `<strike>`, `<s>`
* `string[] specialCaseClosingTags` — Corresponding closing tags
* `string[] adjacentTagsNotToRemove` — Tags preserved even when empty: `p`, `br`

#### Constructor

* `HtmlDiff(string oldText, string newText)` — Initializes diff engine with preprocessed text
  * **Preprocessing Pipeline:**
    1. `standardizeTags()` — Normalize tag variations
    2. `escapeUnclosedAngleBrackets()` — Fix malformed HTML
    3. `removeAdjacentTags()` — Clean empty tag pairs
    4. `sanitizeText()` — Normalize whitespace
  * **Initialization:** Creates empty StringBuilder for output

#### HTML Preprocessing Methods

##### Tag Cleanup

* `string removeAdjacentTags(string text)` — Removes empty tag pairs using HtmlAgilityPack
  * **Configuration:**
    * Allows `<br />` empty tags
    * Disables auto-close and syntax checking
    * Preserves empty node structure
  * **Processing:** Recursive removal via `RemoveEmptyNodes()`

* `void RemoveEmptyNodes(HtmlAgilityPack.HtmlNode containerNode)` — Recursively removes empty HTML nodes
  * **Preservation Rules:**
    * Keep nodes with attributes
    * Keep `p` and `br` tags (in `adjacentTagsNotToRemove`)
    * Keep nodes with inner text
  * **Algorithm:** Bottom-up traversal (processes children first)

##### Malformed HTML Handling

* `string escapeUnclosedAngleBrackets(string htmlInput)` — Escapes unclosed `<` characters
  * **Algorithm:**
    1. Track state of last `<` character
    2. Collect positions of all unclosed `<` chars
    3. Replace unclosed `<` with `&lt;`
  * **Purpose:** Prevents HTML parsing failures from malformed input

#### Tokenization Methods

##### Word Splitting Pipeline

* `void SplitInputsToWords()` — Tokenizes both old and new text
  * **Process:** `Explode()` → `ConvertHtmlToListOfWords()`
  * **Result:** Populates `oldWords` and `newWords` arrays

* `string[] Explode(string value)` — Splits string into character array
  * **Implementation:** `Regex.Split(value, @"")`
  * **Purpose:** Converts string to char sequence for state machine processing

* `string[] ConvertHtmlToListOfWords(string[] characterString)` — State machine tokenizer
  * **States:** `Mode.character`, `Mode.tag`, `Mode.whitespace`
  * **Token Types:**
    * **Words:** Alphanumeric sequences (`[\w\#@]+`)
    * **Tags:** Complete HTML tags from `<` to `>`
    * **Whitespace:** Consecutive whitespace sequences
    * **Punctuation:** Individual non-word characters
  * **State Transitions:**
    * `character` → `tag` on `<`
    * `character` → `whitespace` on whitespace
    * `tag` → `character`/`whitespace` on `>`
    * `whitespace` → `tag` on `<`, `character` on non-whitespace

#### Matching Algorithm Methods

##### Index Building

* `void IndexNewWords()` — Builds reverse index of newWords for fast lookups
  * **Optimization:** Tags stripped of attributes for better matching
  * **Data Structure:** `Dictionary<string, List<int>>` mapping tokens to positions
  * **Purpose:** O(1) lookup during match finding instead of O(n) search

* `string StripTagAttributes(string word)` — Removes tag attributes for matching
  * **Pattern:** Extracts `<tagname` and appends `>` or `/>`
  * **Example:** `<div class="foo">` → `<div>`
  * **Rationale:** Attribute changes not currently supported in diff

##### Block Matching (LCS Algorithm)

* `List<Match> MatchingBlocks()` — Finds all matching blocks between old and new text
  * **Algorithm:** Recursive divide-and-conquer
  * **Entry Point:** Calls `FindMatchingBlocks()` with full text ranges

* `void FindMatchingBlocks(int startInOld, int endInOld, int startInNew, int endInNew, List<Match> matchingBlocks)` — Recursive block finder
  * **Algorithm:**
    1. Find longest common subsequence in range
    2. Recursively process text before match
    3. Add match to results
    4. Recursively process text after match
  * **Base Case:** No match found in range

* `Match FindMatch(int startInOld, int endInOld, int startInNew, int endInNew)` — Finds longest matching sequence
  * **Algorithm:** Dynamic programming with sliding window
  * **Data Structure:** `matchLengthAt` dictionary tracks match lengths
  * **Optimization:** Uses `wordIndices` for O(1) position lookup
  * **Result:** Returns longest match or null if no match found

#### Operation Generation Methods

##### Operation Builder

* `List<Operation> Operations()` — Converts matches into diff operations
  * **Algorithm:**
    1. Iterate through matches in sequence
    2. Generate operations for gaps between matches
    3. Generate equal operations for match ranges
  * **Operation Types:**
    * `Action.replace` — Both positions have changed
    * `Action.insert` — Only new position advanced
    * `Action.delete` — Only old position advanced
    * `Action.equal` — Match found

#### Output Generation Methods

##### Build Pipeline

* `string Build()` — Main orchestration method
  * **Pipeline:**
    1. `SplitInputsToWords()` — Tokenize inputs
    2. `IndexNewWords()` — Build lookup index
    3. `Operations()` — Generate diff operations
    4. `PerformOperation()` — Execute each operation
  * **Returns:** Complete HTML diff markup

##### Operation Processors

* `void PerformOperation(Operation operation)` — Dispatches operation to handler
  * **Routing:**
    * `Action.equal` → `ProcessEqualOperation()`
    * `Action.delete` → `ProcessDeleteOperation()`
    * `Action.insert` → `ProcessInsertOperation()`
    * `Action.replace` → `ProcessReplaceOperation()`

* `void ProcessEqualOperation(Operation operation)` — Outputs unchanged content
  * **Processing:** Extracts word range and appends to content
  * **No Markup:** Unchanged content has no wrapping tags

* `void ProcessDeleteOperation(Operation operation, string cssClass)` — Wraps deletions
  * **Markup:** `<del>` tags around removed content
  * **CSS Classes:** `diffdel` or `diffmod` (for replacements)

* `void ProcessInsertOperation(Operation operation, string cssClass)` — Wraps insertions
  * **Markup:** `<ins>` tags around added content
  * **CSS Classes:** `diffins` or `diffmod` (for replacements)

* `void ProcessReplaceOperation(Operation operation)` — Handles modifications
  * **Processing:** Calls both delete and insert with `diffmod` class
  * **Result:** Shows old content deleted, new content inserted

##### Smart Tag Insertion

* `void InsertTag(string tag, string cssClass, List<string> words)` — Complex tag wrapping with HTML preservation
  * **Algorithm:**
    1. Extract consecutive non-tag words → wrap in `<ins>`/`<del>`
    2. Extract consecutive tag words → output without wrapping
    3. Repeat until all words processed
  * **Special Handling:**
    * Tags in deletions are removed entirely
    * Tags in insertions are preserved
  * **Purpose:** Prevents nested `<ins>`/`<del>` tags inside HTML tags

* `string[] ExtractConsecutiveWords(List<string> words, Func<string, bool> condition)` — Extracts and removes matching prefix
  * **Processing:**
    1. Find first non-matching word
    2. Extract all words before it
    3. Remove extracted words from list
  * **Returns:** Extracted word array

* `string WrapText(string text, string tagName, string cssClass)` — Simple tag wrapper
  * **Format:** `<{tag}>{text}</{tag}>`
  * **Note:** CSS class parameter currently unused in output

#### Utility Methods

##### Tag Detection

* `bool IsTag(string item)` — Checks if token is HTML tag
* `bool IsOpeningTag(string item)` — Pattern: `^\s*<[^>]+>\s*$`
* `bool IsClosingTag(string item)` — Pattern: `^\s*</[^>]+>\s*$`
* `bool IsStartOfTag(string val)` — Checks if character is `<`
* `bool IsEndOfTag(string val)` — Checks if character is `>`
* `bool IsWhiteSpace(string value)` — Regex whitespace check

---

### `Match` - Internal Matching Block Class

**Purpose:** Represents a continuous sequence of matching tokens between old and new text.

#### Properties

* `int StartInOld` — Starting position in old text array
* `int StartInNew` — Starting position in new text array
* `int Size` — Number of consecutive matching tokens

#### Computed Properties

* `int EndInOld` — `StartInOld + Size` (exclusive end position)
* `int EndInNew` — `StartInNew + Size` (exclusive end position)

#### Constructor for `Match` - Internal Matching Block Class

* `Match(int startInOld, int startInNew, int size)` — Initializes match block

---

### `Operation` - Internal Diff Operation Class

**Purpose:** Represents a single diff operation (insert/delete/equal/replace) with position ranges.

#### Properties `Operation` - Internal Diff Operation Class

* `Action Action` — Type of operation (enum: `equal`, `delete`, `insert`, `none`, `replace`)
* `int StartInOld` — Start position in old text array
* `int EndInOld` — End position in old text array (exclusive)
* `int StartInNew` — Start position in new text array
* `int EndInNew` — End position in new text array (exclusive)

#### Constructor for `Operation` - Internal Diff Operation Class

* `Operation(Action action, int startInOld, int endInOld, int startInNew, int endInNew)` — Initializes operation with ranges

---

### `Mode` - Tokenization State Enum

**Purpose:** Defines state machine states for word tokenization.

#### Values

* `character` — Processing regular text characters
* `tag` — Inside HTML tag (between `<` and `>`)
* `whitespace` — Processing consecutive whitespace

---

### `Action` - Diff Operation Type Enum

**Purpose:** Defines types of diff operations generated during comparison.

#### Values of `Action` - Diff Operation Type Enum

* `equal` — Content unchanged in both versions
* `delete` — Content removed from old version
* `insert` — Content added in new version
* `none` — No operation (placeholder/gap)
* `replace` — Content modified (combination of delete + insert)

---

## Business logic and integration patterns

### Document Comparison Workflow

The class supports standard document redlining workflows:

* **Legal/Regulatory Review:** Track changes between document versions
* **Content Approval:** Show modifications for approval processes
* **Version Control:** Visualize differences between stored versions

### HTML Structure Preservation

Critical for document integrity:

* **Tag Matching:** Tags treated as atomic units during comparison
* **Attribute Ignoring:** Attribute changes don't trigger diffs (limitation)
* **Structure Preservation:** Nesting and hierarchy maintained in output
* **Empty Tag Cleanup:** Removes meaningless empty tag pairs

### Whitespace Normalization

Consistent handling across HTML sources:

* **Non-Breaking Spaces:** All variants converted to regular spaces
* **Whitespace Tokens:** Preserved as separate tokens for accurate positioning
* **Cultural Differences:** Handles different HTML encoding standards

### CSS Class Integration

Output designed for styled rendering:

* **Delete Markup:** `<del>` tags for removed content
* **Insert Markup:** `<ins>` tags for added content
* **Modification Markup:** Both tags with `diffmod` class (currently unused)
* **Stylesheet Integration:** Output expects CSS definitions for visual presentation

---

## Technical implementation considerations

### Performance Characteristics

* **Tokenization:** O(n) where n = character count
* **Index Building:** O(m) where m = token count in new text
* **Match Finding:** O(n × m) worst case for dynamic programming
* **Output Generation:** O(m) for operation processing
* **Memory:** O(n + m) for token arrays and index dictionary

### Algorithm Analysis

* **LCS Variant:** Uses longest common subsequence with recursive divide-and-conquer
* **Optimization:** Word indexing provides O(1) lookup instead of O(n) search
* **Match Quality:** Finds longest matches first, then recursively processes gaps
* **Greedy Approach:** May not find globally optimal diff in complex cases

### Edge Case Handling

* **Malformed HTML:** Escapes unclosed angle brackets
* **Empty Tags:** Removes meaningless adjacent tag pairs
* **Tag Attributes:** Stripped during matching (limitation)
* **Special Characters:** Non-breaking space normalization
* **Nested Changes:** Smart tag insertion prevents invalid HTML nesting

### HtmlAgilityPack Integration

External dependency for HTML parsing:

* **Empty Node Handling:** Configured to preserve `<br />` tags
* **Syntax Checking:** Disabled for lenient parsing
* **Tree Traversal:** Recursive bottom-up processing

---

## Integration with broader PDI system

### BatchProcessorService Context

HtmlDiff operates within the batch processing service:

* Likely used during document transformation/comparison workflows
* Processes content extracted from Publisher documents
* Generates redline output for regulatory compliance documents
* May integrate with approval workflows for fund documentation

### Potential Integration Points

Based on system architecture:

* **Document Transformation:** Compare before/after template application
* **Approval Workflow:** Show changes for legal review
* **Audit Trail:** Document modifications for compliance
* **Version Control:** Track document evolution over time

### Output Format Expectations

Generated HTML designed for:

* **Web Display:** CSS-styled diff visualization
* **Email Notifications:** Embedded HTML in change alerts
* **Report Generation:** Included in change reports
* **Archive Storage:** Permanent record of document changes

---

## Limitations and known issues

### Current Limitations

#### Attribute Changes Not Detected

* **Issue:** `<div class="old">` vs `<div class="new">` shows as unchanged
* **Reason:** Tags stripped to base form during matching
* **Workaround:** None implemented

#### CSS Class System Unused

* **Issue:** `cssClass` parameter generated but not output in markup
* **Code Comment:** Line 522 shows class param but format doesn't use it
* **Impact:** Cannot distinguish `diffmod` from `diffdel`/`diffins` in output

#### Special Case Code Commented Out

* **Location:** Lines 500-520 in `InsertTag()`
* **Purpose:** Handling of `<strong>`, `<b>`, `<i>` tags in changes
* **Status:** Multiple sections commented out with "NEW CODE" / "OLD CODE" markers
* **Risk:** Unclear which behavior is intended

#### StripHtml Parameter Ignored

* **Method:** `redlineText()` parameter `stripHtml`
* **Status:** Accepted but never used
* **Impact:** Misleading API signature

### Potential Issues

#### Performance on Large Documents

* **Concern:** O(n × m) complexity could be slow for very large texts
* **Risk Area:** `FindMatch()` dynamic programming
* **Mitigation:** None implemented (no size limits or timeout)

#### Memory Usage

* **Concern:** Full text tokenization stores all words in memory
* **Risk Area:** Large HTML documents with many tags
* **Mitigation:** None implemented

#### Invalid HTML Output

* **Concern:** Complex nested changes might produce invalid HTML
* **Note:** Comment line 489 acknowledges this: "this still doesn't guarantee valid HTML"
* **Example:** Diffing text containing `<ins>` or `<del>` tags

---

## Action items for system maintenance

1. **Code Cleanup Priority**
   * **CRITICAL:** Resolve commented code sections (lines 492-520)
   * **HIGH:** Remove unused `stripHtml` parameter or implement functionality
   * **MEDIUM:** Add XML documentation to public API

2. **Performance Monitoring**
   * Profile with realistic document sizes
   * Monitor memory usage patterns
   * Identify bottlenecks in match finding

3. **Bug Investigation**
   * Test attribute change scenarios
   * Verify CSS class output functionality
   * Validate HTML output correctness

4. **Documentation Needs**
   * Usage examples for common scenarios
   * Limitations documentation for users
   * Algorithm explanation for maintainers

5. **Testing Requirements**
   * Create comprehensive test suite
   * Document test scenarios and expected outputs
   * Add regression tests for bug fixes

---

## Developer notes

### Design Pattern Recognition

* **Factory Method:** Static `redlineText()` entry point
* **Builder Pattern:** Incremental content StringBuilder
* **State Machine:** Tokenization mode transitions
* **Strategy Pattern:** Different operation processors

### Code Evolution Evidence

Multiple commented sections and "NEW CODE" markers suggest:

* Active development/experimentation
* Incomplete feature implementation
* Need for code review and cleanup decision

### HTML Agility Pack Usage

External dependency requires:

* NuGet package reference
* Understanding of HtmlAgilityPack API
* Maintenance of compatibility across versions
