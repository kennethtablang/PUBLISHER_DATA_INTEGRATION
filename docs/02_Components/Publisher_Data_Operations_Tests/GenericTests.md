# Tests — Generic Tests (analysis)

## One-line purpose

Test suite for bilingual text processing functionality including date formatting, French translation services, and multilingual document content management with missing translation tracking.

---

## Files analyzed

* `Publisher_Data_Operations_Tests/GenericTests.cs`

---

## What this code contains (high level)

1. **Date Formatting Tests** — Validation of bilingual date formatting with error handling for invalid dates
2. **French Translation Testing** — Comprehensive testing of English-to-French translation services with fallback mechanisms
3. **Test Data Infrastructure** — Setup of translation dictionaries and missing translation tracking tables
4. **Multilingual Content Processing** — Tests for financial data formatting including currency and percentage localization
5. **Missing Translation Management** — Framework for tracking and reporting untranslated content

This test suite validates the multilingual capabilities essential for Canadian financial document processing, ensuring proper English-French translation and cultural formatting compliance.

---

## Classes, properties and methods (developer reference)

### `GenericTests` - Main Test Class

**Purpose:** Tests the Generic class functionality for multilingual document processing, particularly English-French translation and date formatting capabilities.

#### Test Infrastructure Properties

##### Core Test Objects

* `Generic genericTests` — System under test instance
* `RowIdentity row` — Test row identity with client/document context

#### Constructor Setup

* `GenericTests()`

**Initialization Logic:**

##### System Under Test Setup

* **Generic Instance:** Creates Generic class with empty parameters `new Generic("", null)`
* **Row Identity:** Creates test row with `ClientID: 1003`, `DocumentTypeID: 4`, `LOBID: 13`, Document: `"53453"`
* **Property Assignment:** Maps row identity properties to genericTests instance

##### Translation Data Setup

* **Client Translation Loading:**

  ```csharp
  genericTests.ClientTranslation = TestHelpers.LoadTestData(PUB_Data_Integration_DEV.pdi_Client_Translation_Language, "pdi_Client_Translation_Language");
  ```

* **Global Text Loading:**

  ```csharp
  genericTests.GlobalTextLanguage = TestHelpers.LoadTestData(PUB_Data_Integration_DEV.pdi_Global_Text_Language, "pdi_Global_Text_Language");
  ```

##### Missing French Tracking Setup

**Primary Missing Translation Table:**

```csharp
DataTable MissingFrench = new DataTable("MissingFrench");
// Schema: Missing_ID (Guid), Client_ID (int), LOB_ID (int), Document_Type_ID (int), en-CA (string)
// Test Data: One dummy record with "Dummy Value"
```

**Missing Translation Details Table:**

```csharp
DataTable MissingFrenchDetails = new DataTable("MissingFrenchDetails");
// Schema: Missing_ID (Guid), Job_ID (int), Document_Number (string), Field_Name (string)
// Test Data: One test record linked to MissingFrench via GUID
```

---

## Test Methods (detailed analysis)

### Date Formatting Tests

#### `[Theory] isLongDateFormat(string date, string expectedEnglish, string expectedFrench)`

**Purpose:** Validates bilingual date formatting with comprehensive error handling.

**Test Cases:**

##### Valid Date Formats

1. **Standard Format:** `"01/10/2019"` → `"October 1, 2019"` / `"1er octobre 2019"`
   * **English:** Full month name with day, year
   * **French:** Ordinal indicator for 1st (`1<sup>er</sup>`) with French month name

2. **Leading Zero Variations:**
   * `"08/07/2020"` → `"July 8, 2020"` / `"8 juillet 2020"`
   * `"8/07/2020"` → `"July 8, 2020"` / `"8 juillet 2020"`
   * **Behavior:** Handles leading zeros consistently

##### Invalid Date Handling

1. **Invalid Day Format:** `"08/7/2020"` → `"Invalid Date Format"` / `"Format de date non valide"`
2. **Invalid Day Value:** `"33/07/2020"` → `"Invalid Date Format"` / `"Format de date non valide"`

**Technical Implementation:**

* **Method Called:** `genericTests.longFormDate(date)`
* **Return Type:** `string[]` array with English [0] and French [1] elements
* **Error Handling:** Graceful fallback to localized error messages

**Cultural Formatting Notes:**

* **French Ordinal:** Uses HTML superscript for "1er" (first)
* **French Month Names:** Properly localized (octobre, juillet)
* **Error Messages:** Bilingual error reporting

---

### French Translation Tests

#### `[Theory] searchFrenchTests(string en, string expectedFrench)`

**Purpose:** Validates English-to-French translation with financial formatting, missing translation handling, and cultural number formatting.

**Test Cases:**

##### Successful Translations

1. **Financial Terminology with Percentages:**

   ```csharp
   "OTHER ASSETS LESS LIABILITIES (-2.15%)" → "AUTRES ACTIFS, MOINS LES PASSIFS (-2,15\u00A0%)"
   ```

   * **Translation:** Complete financial term translation
   * **Number Format:** Decimal comma instead of period (2,15)
   * **Typography:** Non-breaking space before percentage symbol (\u00A0)

2. **Date Translation:**

   ```csharp
   "July 8, 2020" → "8 juillet 2020"
   ```

   * **Format Change:** Removes comma, reorders day/month
   * **Month Translation:** July → juillet

3. **Complex Financial Terms with Currency:**

   ```csharp
   "Information Technology –  $0.73" → "TECHNOLOGIES DE L'INFORMATION –  0,73\u00A0$"
   ```

   * **Industry Translation:** Information Technology → TECHNOLOGIES DE L'INFORMATION
   * **Currency Format:** Dollar sign moves to end, decimal comma, non-breaking space

##### Missing Translation Handling

1. **Missing Translation with Percentage:**

   ```csharp
   "ASSET-BACKED SECURITIES (14.28%)" → "MISSING FRENCH: ASSET-BACKED SECURITIES (14,28\u00A0%)"
   ```

   * **Fallback Pattern:** Prefixes "MISSING FRENCH: " to original text
   * **Number Formatting:** Still applies French formatting (comma decimal, non-breaking space)

2. **Invalid Percentage Format:**

   ```csharp
   "SHORT-TERM INVESTMENTS (1,01 %)" → "MISSING FRENCH: SHORT-TERM INVESTMENTS (ERROR\u00A0%)"
   ```

   * **Error Handling:** Detects invalid format (comma in input), returns "ERROR %"
   * **Missing Translation:** Still flagged as missing French

**Technical Implementation:**

* **Method Called:** `genericTests.SearchFrench(row, en, 888, "fieldname")`
* **Parameters:**
  * `row` — RowIdentity for context
  * `en` — English text to translate
  * `888` — Job ID for tracking
  * `"fieldname"` — Field name for missing translation logging
* **Return Type:** `Tuple<string, string>` with English and French
* **Test Assertion:** Validates `fr.Item2` (French translation)

---

## Translation System Integration Patterns

### Missing Translation Tracking

The test setup demonstrates sophisticated missing translation management:

**Database Schema Integration:**

* **Primary Missing Table:** Links missing translations to client, LOB, and document type
* **Details Table:** Tracks specific job, document, and field where translation is missing
* **GUID Linking:** Maintains referential integrity between missing translation records

**Tracking Workflow:**

1. **Detection:** SearchFrench method detects untranslated content
2. **Logging:** Records missing translation with context (job, document, field)
3. **Fallback:** Provides formatted fallback text with "MISSING FRENCH:" prefix
4. **Formatting:** Still applies French number/currency formatting even for missing translations

### Cultural Formatting Standards

#### French Canadian Formatting Rules

1. **Decimal Separators:** Period (.) → Comma (,)
2. **Percentage Formatting:** Non-breaking space before % symbol (\u00A0)
3. **Currency Formatting:** Dollar sign after amount with non-breaking space
4. **Date Formatting:** Day Month Year without commas

#### Typography Standards

* **Non-Breaking Spaces:** Used before currency symbols and percentage signs
* **HTML Formatting:** Supports superscript tags for ordinal indicators
* **Case Conventions:** Financial terms often in uppercase in French

### Translation Data Management

#### Test Data Sources

1. **Client Translation Language:** Client-specific translation overrides
2. **Global Text Language:** System-wide translation dictionary
3. **Missing Translation Tracking:** Runtime detection and logging

#### Fallback Hierarchy

1. **Client-Specific Translations:** First priority for translation lookup
2. **Global Translations:** System-wide fallback
3. **Missing Translation Handling:** Graceful degradation with formatting preservation
4. **Error Handling:** Invalid format detection with error indicators

---

## Business logic and integration patterns

### Multilingual Document Processing

The tests validate critical functionality for Canadian financial documents:

**Regulatory Compliance:**

* **Official Languages:** English-French bilingual requirements
* **Financial Formatting:** Cultural number formatting standards
* **Date Standards:** Proper localization of date formats

**Content Management:**

* **Translation Tracking:** Systematic identification of missing translations
* **Quality Assurance:** Validation of translation accuracy and formatting
* **Client Customization:** Client-specific translation overrides

### Error Handling and Quality Control

Comprehensive error handling demonstrates production-ready approach:

**Date Validation:**

* **Format Validation:** Detects invalid date formats
* **Value Validation:** Identifies impossible dates (day 33)
* **Bilingual Errors:** Localized error messages

**Translation Quality:**

* **Missing Translation Detection:** Systematic identification
* **Format Preservation:** Maintains formatting even for missing translations
* **Error Indication:** Clear marking of problematic content

---

## Technical implementation considerations

### Performance Characteristics

* **Dictionary Lookups:** Translation services use DataTable-based lookups
* **Test Data Loading:** XML deserialization for translation dictionaries
* **String Processing:** Cultural formatting applied to numeric content
* **Missing Translation Tracking:** Database operations for logging

### Cultural Formatting Complexity

* **Unicode Support:** Non-breaking spaces (\u00A0) for typography
* **Number Parsing:** Decimal format detection and conversion
* **Currency Handling:** Position and spacing rules for different currencies
* **Date Parsing:** Multiple date format support with error handling

### Test Data Management

* **XML Resources:** Translation dictionaries loaded from embedded XML
* **In-Memory Tables:** DataTable structures for translation lookups
* **GUID Tracking:** Referential integrity for missing translation records
* **Row Identity Context:** Client/document context for translation requests

---

## Integration with broader PDI system

### Generic Class Integration

The tests validate the Generic class which appears to be a core multilingual service:

* **Translation Services:** Primary interface for English-French translation
* **Date Formatting:** Centralized date localization service
* **Cultural Formatting:** Number and currency formatting services

### Document Processing Integration

Translation services integrate with document processing workflow:

* **Field-Level Translation:** Individual field translation with tracking
* **Document Context:** Client/LOB/DocumentType context for translation decisions
* **Job Tracking:** Missing translations tracked by processing job

### Database Integration

Missing translation tracking integrates with persistent storage:

* **Missing French Tables:** Database tables for translation gap tracking
* **Translation Dictionaries:** Database-backed translation lookup services
* **Audit Trail:** Complete tracking of translation requests and results

---

## Test coverage and quality assurance

### Date Formatting Coverage

**Valid Date Scenarios:**

* Standard MM/dd/yyyy format
* Leading zero variations
* Ordinal indicators in French

**Error Scenarios:**

* Invalid date formats
* Impossible date values
* Graceful error message localization

### Translation Coverage

**Financial Domain:**

* Asset and liability terminology
* Investment categories and percentages
* Currency amounts and formatting

**Missing Translation Scenarios:**

* Untranslated financial terms
* Invalid percentage formats
* Fallback text generation

**Cultural Formatting:**

* French decimal comma formatting
* Non-breaking space typography
* Currency symbol positioning

---

## Potential enhancements

### Test Coverage Expansion

1. **Edge Cases:** More invalid date formats and boundary conditions
2. **Currency Varieties:** Different currency types and formatting rules
3. **Complex Financial Terms:** Multi-clause financial expressions
4. **Translation Context:** Different translation contexts affecting results

### Translation System Improvements

1. **Translation Quality Metrics:** Automated validation of translation accuracy
2. **Caching Strategy:** Performance optimization for frequently translated terms
3. **Context Awareness:** More sophisticated context-driven translation selection
4. **Bulk Translation:** Batch processing capabilities for large document sets

### Error Handling Enhancements

1. **Detailed Error Messages:** More specific error information for debugging
2. **Recovery Strategies:** Automatic retry or alternative translation sources
3. **Quality Scoring:** Confidence metrics for translation quality
4. **User Feedback Integration:** Mechanism for improving translations based on feedback

---

## Action items for system maintenance

1. **Translation Dictionary Updates:** Regular updates to translation databases
2. **Cultural Formatting Review:** Periodic review of formatting rules for compliance
3. **Error Pattern Analysis:** Analysis of missing translation patterns for improvement
4. **Performance Monitoring:** Monitor translation lookup performance as dictionary grows
5. **Test Data Maintenance:** Keep test translations current with production dictionaries
6. **Regulatory Compliance:** Ensure formatting rules meet current regulatory standards
