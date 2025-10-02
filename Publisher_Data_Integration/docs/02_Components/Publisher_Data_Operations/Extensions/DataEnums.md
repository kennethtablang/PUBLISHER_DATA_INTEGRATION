# Extensions — DataEnums (analysis)

## One-line purposes

Defines core enumeration types that map to database reference tables and control data processing behavior throughout the PDI pipeline.

---

## Files analyzed

* `Publisher_Data_Operations/Extensions/DataEnums.cs`

---

## What this code contains (high level)

1. **FFDocAge** — Document lifecycle status enum mapping to `pdi_Document_Life_Cycle_Status` table
2. **DocumentTypeID** — Document type classification enum mapping to `pdi_Document_Type` table  
3. **DataTypeID** — Data processing type enum mapping to `pdi_Data_Type` table
4. **DataRowInsert** — Processing instruction enum for header row insertion behavior

These enums provide type-safe references to database lookup values and control processing logic flow throughout the system.

---

## Enums detailed reference

### `FFDocAge` - Enum

**Purpose:** Maps to `pdi_Document_Life_Cycle_Status` database table. Defines the regulatory lifecycle stage of fund documents for compliance and processing rules.

**Database Reference:** `pdi_Document_Life_Cycle_Status`

**Values:**

* `BrandNewFund = 0` — Completely new fund, requires full disclosure documentation
* `BrandNewSeries = 1` — New series of existing fund, partial documentation requirements
* `NewFund = 2` — New fund with some operational history
* `NewSeries = 3` — New series with established fund structure
* `TwelveConsecutiveMonths = 4` — Fund has 12+ months of performance data, standard reporting
* `NoSecuritiesIssued = 5` — Fund exists but has not issued any securities to investors

**Business Impact:**

* Determines validation rules and required fields during document processing
* Controls template selection and content requirements
* Affects compliance checks and approval workflows

---

### `DocumentTypeID` - Enum

**Purpose:** Maps to `pdi_Document_Type` database table. Categorizes different document formats and their processing requirements.

**Database Reference:** `pdi_Document_Type`

**Values:**

* `MRFP = 1` — Management Report of Fund Performance
* `FS = 2` — Financial Statements
* `FSMRFP = 3` — Combined FS and MRFP (zip file processing only, not a real document type)
* `QPD = 4` — Quarterly Portfolio Disclosure
* `FF = 5` — Fund Facts
* `FSBOOK = 6` — Financial Statements Book format
* `ETF = 11` — Exchange Traded Fund documents
* `SF = 12` — Summary Financials
* `FP = 13` — Fund Profile
* `EP = 14` — Enhanced Profile
* `SP = 15` — Simplified Profile
* `SFSBOOK = 16` — Summary Financial Statements Book
* `SFS = 17` — Summary Financial Statements
* `QPDBOOK = 19` — QPD Book format

**Key Notes:**

* `FSMRFP = 3` is a synthetic type used only for batch processing when zip files contain both FS and MRFP documents
* Book formats (6, 16, 19) indicate multi-document compilations
* ETF (11) has specialized processing requirements
* Profile types (13, 14, 15) are summary documents with simplified data requirements

**Business Impact:**

* Determines which extraction and transformation rules to apply
* Controls template selection and field validation
* Affects output formatting and publishing workflows

---

### `DataTypeID` - Enum

**Purpose:** Maps to `pdi_Data_Type` database table. Defines the processing methodology and data source type.

**Database Reference:** `pdi_Data_Type`

**Values:**

* `BAU = 1` — Business As Usual (standard operational processing)
* `STATIC = 2` — Static data processing (pre-defined content, minimal transformation)
* `FSMRFP = 3` — Combined FS/MRFP processing methodology
* `QPD = 4` — Quarterly Portfolio Disclosure processing
* `BNY = 5` — Bank of New York data source processing

**Business Impact:**

* Controls which transformation pipelines and validation rules to execute
* Determines data source connection and extraction methods
* Affects error handling and retry logic

---

### `DataRowInsert` - Enum

**Purpose:** Processing instruction enum that controls how header rows are inserted during data transformation operations.

**Usage Context:** Used during Excel extraction and table building operations to control data structure.

**Values:**

* `FirstRow` — Default behavior: adds header as the first row in the table
* `AfterDescRepeat` — Inserts header after description repetition logic
* `AfterColumnChange` — Inserts header after detecting column structure changes
* `ClearExtraColumns` — Uses header (scenario) to determine column count and removes extra columns while applying FirstRow behavior

**Business Impact:**

* Controls table structure during Excel-to-DataTable conversion
* Affects data alignment and validation during transformation
* Ensures consistent header placement across different document formats

---

## Integration patterns

### Database Mapping

All enums except `DataRowInsert` directly correspond to database lookup tables:

* Enum integer values match primary key values in reference tables
* Used throughout the codebase for type-safe database operations
* Enable foreign key relationships without magic numbers

### Processing Logic Control

```csharp
// Example usage patterns:
if (docType == DocumentTypeID.FSMRFP) {
    // Special handling for combined documents
}

switch (ffDocAge) {
    case FFDocAge.BrandNewFund:
        // Apply strictest validation rules
        break;
    case FFDocAge.TwelveConsecutiveMonths:
        // Standard processing
        break;
}
```

### Type Safety Benefits

* Prevents invalid ID values from being passed to database operations

* Enables IntelliSense support and compile-time validation
* Reduces runtime errors from typos or incorrect ID references
* Improves code readability compared to magic numbers

---

## Business vocabulary connections

These enums form the foundation for business rule processing:

## Document Lifecycle Management

* `FFDocAge` values determine regulatory compliance requirements
* Each stage has different mandatory field requirements
* Affects approval workflows and publishing timelines

## Document Classification System

* `DocumentTypeID` drives template selection and processing pipelines
* Book formats require different pagination and formatting rules
* Profile documents have simplified data requirements

## Processing Methodology Framework

* `DataTypeID` determines which transformation engine to use
* BAU vs STATIC affects validation strictness and field requirements
* External data sources (BNY) have specialized connection handling

---

## Action items for documentation

* Map each enum value to specific business rules and validation requirements  
* Document the relationship between `DocumentTypeID` and template selection logic
* Verify that all enum values are actively used (some may be legacy)
* Confirm database table relationships and foreign key constraints
* Document any planned additions or deprecations for future releases
