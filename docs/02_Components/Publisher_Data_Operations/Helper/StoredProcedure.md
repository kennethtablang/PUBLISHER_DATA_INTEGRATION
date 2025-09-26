# StoredProcedure Helper — Component Analysis

## One-line purpose

SQL stored procedure container that provides complex Publisher database import logic with hierarchical data structure creation, transactional integrity, and dynamic retry mechanisms for ETL data loading operations.

---

## Files analyzed

* `Publisher_Data_Operations/Helper/StoredProcedure.cs`

---

## What this code contains (high level)

1. **Dynamic Stored Procedure Creation** — Generates complex SQL stored procedures as strings for runtime deployment
2. **Publisher Database Import Logic** — Complete ETL import procedure handling documents, sections, fields, and content
3. **Hierarchical Data Structure Management** — Creates nested document structures (Document → Section → Field → Value)
4. **Transaction Management** — Multi-level transaction handling with serializable isolation for data integrity
5. **Retry Logic and Error Handling** — Sophisticated retry mechanisms for deadlock resolution and error recovery

This component bridges the gap between PDI ETL processing and Publisher database persistence with enterprise-grade data integrity and performance optimization.

---

## Classes, properties and methods (developer reference)

### `StoredProcedure` - Static SQL Generator Class

**Purpose:** Provides static methods that generate complex SQL stored procedures as strings for dynamic deployment in the PDI-to-Publisher ETL pipeline.

#### Static Methods

##### Primary Import Procedure

* `static string ImportDataSP()` — Generates the main data import stored procedure
  * **Procedure Name:** `sp_pdi_IMPORT_DATA_temp`
  * **Purpose:** ETL data import from temp table to Publisher schema
  * **Complexity:** 200+ lines of complex SQL with cursors, transactions, and error handling
  * **Author Attribution:** Scott Konkle, created 03/01/2022, updated 06/16/2022

##### Cleanup Utility

* `static string ClearSPByDate()` — Generates cleanup procedure for date-based stored procedure removal
  * **Purpose:** Removes stored procedures older than one day
  * **Safety Mechanism:** Prevents accumulation of temporary procedures
  * **Date Logic:** Uses `datediff(day, @create, getdate()) > 0` for cleanup trigger

---

## Import Stored Procedure Analysis (`sp_pdi_IMPORT_DATA_temp`)

### Data Structure and Flow

#### Temp Table Schema

```sql
CREATE TABLE #ImportData (
    [DOCUMENT_NUMBER] NVARCHAR(50) NOT NULL,
    [FIELD_NAME] NVARCHAR(50) NOT NULL,
    [BUSINESS_ID] INT NOT NULL,
    [XMLCONTENT] XML NULL,
    [CONTENT] NVARCHAR(MAX) NULL,
    [DATA_TYPE] VARCHAR(50) NULL,
    [IsTextField] BIT NOT NULL,
    [IsTableField] BIT NOT NULL,
    [IsChartField] BIT NOT NULL,
    [LANGUAGE_ID] INT NOT NULL,
    [DOCUMENT_TYPE_ID] INT NOT NULL,
    -- ... additional metadata fields
)
```

**Schema Design Analysis:**

* **Document Context:** Links content to specific documents and business entities
* **Content Types:** Supports TEXT, TABLE, and CHART data types with XML and text storage
* **Multilingual:** LANGUAGE_ID enables multiple language content support
* **Hierarchical IDs:** Template, Section, and Field IDs for nested structure
* **File Metadata:** Document filenames and sort order information

### Complex Source Data Integration

#### Multi-Table JOIN Logic

The procedure performs complex JOINs to resolve business context:

```sql
FROM #pdi_import_source PDI
INNER JOIN DOCUMENT_TYPE DocT ON DocT.FEED_TYPE_NAME = PDI.Feed_Type_Name
INNER JOIN LINE_OF_BUSINESS LOB ON LOB.BUSINESS_NAME = PDI.Business_Name
INNER JOIN [LANGUAGE] L ON L.CULTURE_CODE = PDI.CULTURE_CODE
LEFT OUTER JOIN (...) FI -- Field Information lookup
LEFT OUTER JOIN (...) DefaultDT -- Default Document Template
LEFT OUTER JOIN DOCUMENT D -- Existing document lookup
LEFT OUTER JOIN (...) DefaultDS -- Default Document Section
```

**Integration Complexity:**

* **Business Context Resolution:** Maps feed data to internal business entities
* **Template Resolution:** Finds specific or default document templates
* **Field Matching:** Attempts to match fields to existing schema
* **Fallback Logic:** Uses default templates and sections when specific matches unavailable

### Hierarchical Data Creation Process

#### Stage 1: Document Creation

```sql
DECLARE documentDataCursor CURSOR LOCAL FAST_FORWARD FOR 
SELECT DISTINCT document_number, BUSINESS_ID, DOCUMENT_TEMPLATE_ID 
FROM #ImportData WHERE DOCUMENT_ID IS NULL
```

**Process Flow:**

1. **Cursor-Based Processing:** Iterates through distinct documents needing creation
2. **Serializable Transactions:** Prevents concurrent document creation conflicts
3. **Document Creation:** Uses `sp_SAVE_DOCUMENT` stored procedure
4. **ID Population:** Updates temp table with generated document IDs

#### Stage 2: Document Metadata Updates

Separate UPDATE statements for different document attributes:

* **Document Filename (EN):** English filename assignment
* **Document Filename (FR):** French filename assignment  
* **Sort Order:** Document ordering information

**Optimization Pattern:** Separate updates with NULL filtering for performance

#### Stage 3: Section Management

```sql
SELECT TOP 1 @documentSectionID = DOCUMENT_SECTION_ID 
FROM DOCUMENT_SECTION 
WHERE IS_ACTIVE = 1 AND DOCUMENT_TEMPLATE_ID = @documentTemplateID 
ORDER BY SORT_ORDER
```

**Section Logic:**

* **Existing Section Reuse:** Attempts to use first active section
* **Dynamic Section Creation:** Creates new sections using `sp_SAVE_DOCUMENT_SECTION`
* **Template Association:** Links sections to appropriate document templates

#### Stage 4: Field Definition Management

```sql
exec sp_SAVE_DOCUMENT_FIELD @documentSectionID, @documentTemplateID, @fieldName, 
null, null, @isTextField, @isTableField, @isChartField, @documentFieldID out
```

**Field Creation Logic:**

* **Type-Aware Creation:** Creates fields with specific type flags (Text/Table/Chart)
* **Template Association:** Links fields to templates and sections
* **Dynamic Schema:** Creates new fields as needed for imported data

### Content Data Management

#### Primary Content Updates

```sql
UPDATE DFV 
SET CONTENT = ID.[Content], XMLCONTENT = ID.XMLCONTENT, IS_ACTIVE = 1, DATA_TYPE = ID.DATA_TYPE
FROM #ImportData ID
INNER JOIN DOCUMENT_FIELD_VALUE DFV ON ...
```

#### New Content Insertion

```sql
INSERT INTO DOCUMENT_FIELD_VALUE (DOCUMENT_ID, DOCUMENT_FIELD_ID, LANGUAGE_ID, IS_ACTIVE, CONTENT, XMLCONTENT, DATA_TYPE)
SELECT ... FROM #ImportData ID
LEFT OUTER JOIN DOCUMENT_FIELD_VALUE DFV ON ...
WHERE DFV.DOCUMENT_ID IS NULL
```

**Content Management Strategy:**

* **Update Existing:** Modifies existing field values with new content
* **Insert New:** Creates new field values where none exist
* **Type Handling:** Manages both XML and text content appropriately
* **Language Support:** Handles multilingual content insertion

### Historical Data Archiving

#### Retry Logic for History Table

```sql
DECLARE @RetryNo Int = 1, @RetryMaxNo Int = 5;
WHILE @RetryNo < @RetryMaxNo
BEGIN
    BEGIN TRY 
        INSERT INTO DOCUMENT_FIELD_VALUE_HISTORY ...
        SET @RetryNo = @RetryMaxNo;
    END TRY
    BEGIN CATCH
        IF ERROR_NUMBER() IN (-1, -2, 701, 1204, 1205, 1222, 8645, 8651, 30053)
        BEGIN
            SET @RetryNo += 1;
            WAITFOR DELAY '00:00:10';
        END 
        ELSE
            THROW;
    END CATCH
END
```

**Retry Mechanism Analysis:**

* **Specific Error Codes:** Targets deadlock and resource-related errors
* **Exponential Backoff:** 10-second delay between retry attempts
* **Maximum Attempts:** 5 retry attempts before failure
* **Performance Note:** Historical insertion identified as 75% of processing time

---

## Transaction Management and Data Integrity

### Isolation Level Strategy

The procedure uses `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE` for critical sections:

* **Document Creation:** Prevents duplicate document creation
* **Section Creation:** Prevents concurrent section conflicts
* **Field Creation:** "Most troublesome" according to comments

### Transaction Boundaries

* **Short Transactions:** Commits quickly after serializable operations
* **Performance Optimization:** Minimizes lock duration
* **Deadlock Prevention:** Reduces likelihood of transaction conflicts

### Error Handling Pattern

```sql
BEGIN TRY
    -- Complex processing logic
    SELECT 'Complete';
END TRY
BEGIN CATCH
    SELECT ERROR_MESSAGE() AS ErrorMessage;
END CATCH
```

**Error Strategy:**

* **Comprehensive Wrapping:** Entire procedure wrapped in TRY-CATCH
* **Simple Error Reporting:** Returns error message as result set
* **No Rollback Logic:** Relies on individual transaction management

---

## Performance Considerations and Optimizations

### Cursor Usage Justification

The procedure uses cursors for hierarchical data creation:

* **Document Cursor:** Processes distinct documents requiring creation
* **Section Cursor:** Handles section creation per template
* **Field Cursor:** Manages field definition creation

**Cursor Characteristics:**

* **LOCAL FAST_FORWARD:** Optimized for performance
* **Minimal Data:** Cursors process only essential data
* **Transaction Scope:** Cursors used within appropriate transaction boundaries

### Performance Bottlenecks

Comments identify specific performance issues:

* **History Table:** 75% of processing time for 1000 records
* **Bulk Operations:** Uses UPDATE and INSERT patterns for efficiency
* **Index Dependencies:** Performance relies on proper indexing

### Memory Management

* **Temp Table Cleanup:** `DROP TABLE IF EXISTS #ImportData`
* **Cursor Cleanup:** Explicit DEALLOCATE statements
* **Resource Management:** Proper cleanup of database resources

---

## Technical implementation considerations

### SQL Code Generation Approach

Storing SQL as C# strings presents several challenges:

* **Maintenance Difficulty:** Complex SQL hard to maintain in string format
* **IDE Support:** Limited IntelliSense and syntax highlighting
* **Version Control:** Difficult to track SQL changes in version control
* **Testing:** SQL logic requires database testing environment

### Dynamic Procedure Deployment

The approach of generating procedures at runtime:

* **Flexibility:** Allows procedure customization per deployment
* **Complexity:** Adds deployment complexity
* **Security:** Generated SQL needs proper validation
* **Debugging:** Runtime SQL generation complicates debugging

### Database Schema Dependencies

The procedure assumes specific Publisher schema:

* **Table Structure:** Depends on exact Publisher table schemas
* **Stored Procedures:** Calls other procedures (`sp_SAVE_DOCUMENT`, etc.)
* **Data Types:** Assumes specific column types and constraints
* **Relationships:** Relies on foreign key relationships

---

## Integration with broader PDI system

### ETL Pipeline Role

This stored procedure represents the final stage of the ETL process:

* **Extract:** Data extracted from Excel files by other components
* **Transform:** Data transformed and validated by ExcelHelper and related components
* **Load:** This procedure handles the final load into Publisher

### Temp Table Integration

The procedure assumes data provided via `#pdi_import_source` temp table:

* **Data Contract:** Specific column names and types expected
* **Processing Context:** Assumes single job's data in temp table
* **Performance:** Temp table approach enables bulk processing

### Error Reporting Integration

The procedure's error handling integrates with PDI error tracking:

* **Simple Error Format:** Returns error message as result set
* **Processing Integration:** Calling code handles error messages
* **Logging:** External logging handled by PDI components

---

## Date-based cleanup mechanism

### Cleanup Procedure Logic

The `ClearSPByDate()` method provides automated cleanup:

```sql
IF datediff(day, @create, getdate()) > 0
BEGIN 
    DROP PROCEDURE IF EXISTS [dbo].[sp_pdi_IMPORT_DATA_temp]
    PRINT 'sp_pdi_IMPORT_DATA_temp different create and current dates - deleted'
END
```

**Cleanup Strategy:**

* **Daily Cleanup:** Removes procedures older than one day
* **Safety Mechanism:** Prevents accumulation of temp procedures
* **Automated Management:** Reduces manual procedure management overhead

---

## Potential enhancements

### Performance Improvements

1. **Bulk Operations:** Replace cursors with set-based operations where possible
2. **Parallel Processing:** Enable parallel execution for independent operations
3. **Index Optimization:** Optimize temp table indexing for JOIN performance
4. **History Table Optimization:** Address the 75% performance bottleneck

### Error Handling Enhancements

1. **Detailed Error Reporting:** Provide more context in error messages
2. **Partial Success Handling:** Handle scenarios where some data succeeds
3. **Rollback Logic:** Implement proper rollback strategies for failures
4. **Error Logging:** Integration with PDI logging infrastructure

### Maintenance Improvements

1. **SQL File Externalization:** Move SQL to external files for better maintenance
2. **Parameter Validation:** Add parameter validation and sanitization
3. **Schema Validation:** Validate schema assumptions before execution
4. **Documentation:** Enhance inline documentation for complex logic

### Scalability Enhancements

1. **Batch Processing:** Support for larger data volumes
2. **Memory Optimization:** Reduce memory footprint for large imports
3. **Connection Pooling:** Optimize database connection usage
4. **Monitoring Integration:** Add performance monitoring capabilities

---

## Action items for system maintenance

1. **Performance Analysis:** Profile the stored procedure execution to identify additional bottlenecks beyond the history table
2. **SQL Externalization:** Consider moving complex SQL to external files for better maintainability
3. **Schema Evolution:** Plan for Publisher schema changes and their impact on this procedure
4. **Error Handling Enhancement:** Improve error reporting and handling mechanisms
5. **Testing Strategy:** Develop comprehensive testing approach for complex SQL logic
6. **Documentation:** Document the expected data contract for the temp table input
