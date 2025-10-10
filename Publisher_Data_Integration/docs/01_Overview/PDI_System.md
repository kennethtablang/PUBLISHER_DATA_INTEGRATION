# PDI System - File Location Reference

## Project Structure Overview

```file
Solution Root/
├── PDI_Azure_Function/              # Azure Functions project
│   ├── Extensions/                   # Helper extension methods
│   ├── Properties/                   # Publish profiles and settings
│   └── *.cs files                    # Function definitions
│
└── Publisher_Data_Operations/        # Core business logic library
    ├── Extensions/                   # Data processing extensions
    ├── Helper/                       # Helper classes
    └── *.cs files                    # Core processing classes
```

---

## PDI_Azure_Function Project

### Location: `PDI_Azure_Function/`

#### Main Function Files

| File | Path | Purpose |
|------|------|---------|
| BatchBlob.cs | `PDI_Azure_Function/` | Blob trigger for file uploads - handles ZIP and XLSX files |
| QueueProcess.cs | `PDI_Azure_Function/` | Queue trigger for file validation and transformation |
| QueueImport.cs | `PDI_Azure_Function/` | Queue trigger for Publisher database import |
| StatusPage.cs | `PDI_Azure_Function/` | HTTP trigger for web-based status monitoring |
| BlobProcess.cs | `PDI_Azure_Function/` | (Commented out) Original blob processing function |
| TimerCheck.cs | `PDI_Azure_Function/` | (Commented out) Scheduled maintenance function |

**Key Methods by File:**

**BatchBlob.cs:**

- `Run()` - Main blob trigger entry point
- Handles: File type detection, batch creation, ZIP extraction, queue message sending

**QueueProcess.cs:**

- `Run()` - Main queue trigger entry point
- Handles: File validation, data transformation, staging, queue forwarding

**QueueImport.cs:**

- `Run()` - Main queue trigger entry point  
- Handles: Publisher import, batch completion, email notifications

**StatusPage.cs:**

- `Status()` - Display status page with file listing
- `DownloadBlob()` - Download processed files
- `ValidationMessages()` - Display validation errors
- `FileUpload()` - Handle web-based file uploads

---

#### Extension Methods

**Location: `PDI_Azure_Function/Extensions/`**

| File | Purpose | Key Methods |
|------|---------|-------------|
| BlobServiceExtensions.cs | Azure Blob Storage helpers | MoveTo, CopyTo, SaveTo, Open, Find, GetListing |
| QueueServiceExtensions.cs | Azure Queue Service helpers | GetQueueLength, AddMessage |
| HttpExtensions.cs | HTTP request helpers | GetBaseURL |

**BlobServiceExtensions.cs:**

```file
PDI_Azure_Function/Extensions/BlobServiceExtensions.cs

Methods:
- MoveTo(source, dest) - Move blob between containers
- CopyTo(source, dest) - Copy blob with optional delete
- SaveTo(stream, path) - Save stream to blob
- Open(path) - Open blob as stream
- Find(filename) - Search for blob across folders
- GetListing() - List blobs in container
```

**QueueServiceExtensions.cs:**

```file
PDI_Azure_Function/Extensions/QueueServiceExtensions.cs

Methods:
- GetQueueLength() - Get approximate message count
- AddMessage(DTO) - Add DataTransferObject to queue
```

---

#### Data Transfer Object

**Location: `PDI_Azure_Function/DataTransferObject.cs`**

```file
PDI_Azure_Function/DataTransferObject.cs

Purpose: Serializable object passed between queue stages

Properties:
- File_ID, FileName, FileRunID
- Batch_ID, Job_ID
- FileStatus, ProcessStatus, FileDetails
- ErrorMessage, RetryCount
- NotificationEmailAddress

Methods:
- Base64Encode() - Serialize to Base64 JSON for queue
- Constructor overloads for different scenarios
```

---

#### Configuration Files

| File | Path | Purpose |
|------|------|---------|
| host.json | `PDI_Azure_Function/` | Function app settings (queue batch size, logging) |
| local.settings.json | `PDI_Azure_Function/` | Local development environment variables |
| .gitignore | `PDI_Azure_Function/` | Files excluded from source control |
| PDI_Azure_Function.csproj | `PDI_Azure_Function/` | Project file with NuGet package references |

**host.json:**

```json
PDI_Azure_Function/host.json

Configuration:
- Queue batch size: 4 (process 4 messages concurrently)
- Logging levels
- Application Insights settings
```

**local.settings.json:**

```json
PDI_Azure_Function/local.settings.json (NOT in source control)

Environment Variables:
- sapdi: Azure Storage connection string
- PDI_ConnectionString: PDI database connection
- PUB_ConnectionString: Publisher database connection
- IncomingBatchContainer: Name of upload container
- MainContainer: Name of main storage container
- PDI_QueueName: Processing queue name
- PUB_QueueName: Import queue name
- SMTP_Password: SendGrid API key
- SMTP_FromEmail: Sender email address
- SMTP_FromName: Sender display name
```

---

#### Publish Profiles

**Location: `PDI_Azure_Function/Properties/PublishProfiles/`**

| File | Environment | Purpose |
|------|-------------|---------|
| fapdidev - Zip Deploy.pubxml | Development | Publish to DEV environment |
| fapp-pdi-quality - Zip Deploy.pubxml | QA | Publish to QA environment |

---

## Publisher_Data_Operations Project

### Location: `Publisher_Data_Operations/`

#### Core Processing Classes

| File | Path | Purpose |
|------|------|---------|
| Orchestration.cs | `Publisher_Data_Operations/` | Main orchestration of file processing workflow |
| Transform.cs | `Publisher_Data_Operations/` | Data transformation and table processing |
| Processing.cs | `Publisher_Data_Operations/` | Processing status management |

**Orchestration.cs:**

```file
Publisher_Data_Operations/Orchestration.cs

Key Methods:
- ProcessFile(stream, template, retryCount) - Main processing pipeline
- PublisherImport(jobID) - Import to Publisher database
- HandleEventMessage(event) - Process Azure Event Grid messages

Properties:
- FileID, FileRunID, ErrorMessage
- FileStatus, ProcessStatus, FileDetails
- GetFile (PDIFile), GetJob, GetProcessStatus
```

**Transform.cs:**

```file
Publisher_Data_Operations/Transform.cs

Key Methods:
- ProcessTables(dataTable, docType) - Process table data
- ExtractData(sheet, template) - Extract data from Excel
- FormatValues(value, type) - Format values by type
- MergeTables(table1, table2) - Merge table data
```

---

#### Helper Classes

**Location: `Publisher_Data_Operations/Helper/`**

| File | Purpose |
|------|---------|
| Generic.cs | Generic helper methods, validation, staging |
| PDIFile.cs | File metadata and properties |
| PDIStream.cs | Stream wrapper with metadata |
| PDIBatch.cs | Batch file management |
| Logger.cs | Error and warning logging |
| PDISendGrid.cs | Email notification handling |
| RowIdentity.cs | Track document/fund identity |

**Generic.cs:**

```file
Publisher_Data_Operations/Helper/Generic.cs

Key Methods:
- ValidateFile(file, template) - Validate file structure
- StageData(jobID, dataTable) - Stage data for import
- GetStatusTable(offset, pageSize) - Get file status list
- verifyFrenchTableText(french, english) - Lookup/validate French translation
- loadValidParameters() - Load valid parameter list
- longFormDate(date) - Convert date to long form text
```

**PDIFile.cs:**

```file
Publisher_Data_Operations/Helper/PDIFile.cs

Properties:
- FileID, FileName, OnlyFileName
- ClientID, LOBID, DocumentTypeID, JobID
- BatchID, FileRunID
- UploadDate, ProcessDate
- FileStatus, ErrorMessage, RetryCount

Methods:
- GetDefaultTemplateName() - Template filename for document type
- GetDefaultStaticTemplateName() - Static template filename
```

**PDIStream.cs:**

```file
Publisher_Data_Operations/Helper/PDIStream.cs

Purpose: Wraps Stream with PDIFile metadata

Properties:
- PdiFile - Associated file metadata
- Stream - Underlying data stream

Methods:
- Constructor overloads for different scenarios
- Dispose() - Cleanup resources
```

**PDIBatch.cs:**

```file
Publisher_Data_Operations/Helper/PDIBatch.cs

Key Methods:
- LoadBatch(stream, createdDate) - Load ZIP batch
- RecordBatch(filename, createdDate) - Record batch in database
- RecordSingle(filename) - Record single file batch
- ExtractEntry(filename) - Extract file from ZIP
- MarkEntryExtracted(filename) - Mark file as extracted
- IsComplete() - Check if all batch files complete
- Count() - Number of files in batch
- SetComplete(filename) - Mark file as complete
```

**Logger.cs:**

```file
Publisher_Data_Operations/Helper/Logger.cs

Static Methods:
- AddError(logger, message) - Log error
- AddWarning(logger, message) - Log warning
- AzureError(logger, message) - Log Azure-specific error
- AzureWarning(logger, message) - Log Azure-specific warning
```

**PDISendGrid.cs:**

```file
Publisher_Data_Operations/Helper/PDISendGrid.cs

Key Methods:
- SendTemplateMessage(stream) - Send success email for single file
- SendBatchMessage(batch) - Send batch completion email
- SendErrorMessage(stream, error) - Send error notification
```

---

#### Extension Classes - Data Processing

**Location: `Publisher_Data_Operations/Extensions/`**

| File | Purpose | Key Functionality |
|------|---------|-------------------|
| StringExtensions.cs | Text processing and formatting | Currency, percentage, date formatting; number/date detection |
| Data.cs | DataTable extensions | XML conversion, validation, CSV export |
| Table.cs | Table data structures | TableList, TableItem for structured tables |
| Allocation.cs | Fund allocation management | AllocationList, AllocationTable, AllocationItem |
| Scenario.cs | Conditional content matching | Scenarios, Flags, scenario matching logic |
| ParameterValidator.cs | Content validation | XML validation, scenario syntax validation |
| DBConnection.cs | Database operations | Connection management, retry logic, bulk operations |
| SqlExtensions.cs | SQL helper methods | Parameter handling, column mapping |
| DataEnums.cs | Enumeration definitions | DocumentTypeID, DataTypeID, FFDocAge |
| RowIdentity.cs | Identity tracking | Track document type, client, LOB, fund code |
| ExcelExtensions.cs | Excel file helpers | Optional worksheet detection |

**StringExtensions.cs:**

```file
Publisher_Data_Operations/Extensions/StringExtensions.cs

Key Methods:
- ToBool(value) - Convert to boolean
- IsNaOrBlank(value) - Check if N/A or blank
- ToCurrency(value, culture) - Format as currency
- ToPercent(value, culture) - Format as percentage
- ToDecimal(value, scale, culture) - Format as decimal
- ToDate(value) - Parse date
- AgeInCalendarYears(asOfDate, inceptionDate) - Calculate fund age
- FilingYear(filingDate) - Determine filing year
- CleanXML(value) - Escape XML characters
- ReplaceByDictionary(input, replacements) - Token replacement
- ToBase26(number) - Convert to base-26 (a-z)
- IncrementFieldLetter(current) - Next field letter sequence
```

**Data.cs:**

```file
Publisher_Data_Operations/Extensions/Data.cs

Classes:
- PDI_DataTable - Enhanced DataTable with extended properties
- PDI_DataRow - Enhanced DataRow with extended properties

Key Methods:
- GetExactColumnStringValue(row, column) - Get string value
- GetExactColumnIntValue(row, column) - Get int value
- GetExactColumnDateValue(row, column) - Get date value
- XMLtoDataTable(xmlString) - Parse XML to DataTable
- DataTabletoXML(dataTable) - Convert DataTable to XML
- DataTabletoHTML(dataTable) - Convert to HTML
- RemoveBlankColumns(dataTable) - Remove empty columns
- RemoveBlankRows(dataTable) - Remove empty rows
- ValidateXML(dataTable) - Validate XML in cells
- ToCSV(dataTable) - Export to CSV
```

**Table.cs:**

```file
Publisher_Data_Operations/Extensions/Table.cs

Classes:
- TableList - List of TableItems with table-type-specific logic
- TableItem - Individual table row/cell

TableTypes Enum:
- Percent, Currency, Number, Distribution, Pivot, Date
- MultiDecimal, MultiText, MultiPercent, MultiYear

Key Methods (TableList):
- AddValidation(value, english, french) - Add item with validation
- GetMaxScale() - Calculate decimal precision
- GetTableString(getFrench) - Generate XML output
- MarkDates() - Mark dates for chart axis labels

Key Methods (TableItem):
- GetCellValue(scale, tableType) - Format value as XML cell
- GetCellText(getFrench) - Format text as XML cell
```

**Allocation.cs:**

```file
Publisher_Data_Operations/Extensions/Allocation.cs

Classes:
- AllocationList - List of AllocationTables for all funds
- AllocationTable - Allocations for single fund
- AllocationItem - Single allocation entry

Key Methods (AllocationList):
- AddCode(fundCode, allocation, english, french) - Add allocation
- OrderAll() - Sort all allocations
- AppendToDataTable(dataTable, jobID) - Export to staging

Key Methods (AllocationTable):
- ExistingOrder() - Sort allocations by field name
- MatchedQueue() - Get allocations with existing field names
- UnMatchedQueue() - Get new allocations

Properties (AllocationItem):
- Allocation, EnglishText, FrenchText
- FieldName, EnglishHeader, FrenchHeader
- MatchText (for comparison)
```

**Scenario.cs:**

```file
Publisher_Data_Operations/Extensions/Scenario.cs

Classes:
- Scenarios - List of all scenarios
- Scenario - Single scenario with content
- Flags - List of flag conditions
- Flag - Single field/values condition

Key Methods (Scenarios):
- RankOrder() - Sort by specificity
- AllDistinctFieldNames() - Get all unique field names

Key Methods (Scenario):
- MatchFields(docFields) - Check if scenario matches document
- RankCount() - Number of flags (specificity)
- CompareTo(other) - Compare scenarios for ranking

Key Methods (Flags):
- ParseScenario(text) - Parse scenario string into flags
- IsDefault() - Check if default scenario
- Find(fieldName) - Find flag by field name
```

**ParameterValidator.cs:**

```file
Publisher_Data_Operations/Extensions/ParameterValidator.cs

Key Methods:
- IsValidXML(xmlString) - Validate XML structure
- IsValidScenario(scenario) - Validate scenario syntax
- Static: IsValidXMLStatic(xmlString) - Static XML validation

Properties:
- LastError - Most recent validation error
```

**DBConnection.cs:**

```file
Publisher_Data_Operations/Extensions/DBConnection.cs

Key Methods:
- GetSqlConnection(dbName) - Get open connection with retry
- ExecuteNonQuery(sql, params) - Execute command
- ExecuteScalar(sql, params) - Execute scalar query
- LoadDataTable(sql, params, dataTable) - Fill DataTable
- UpdateDataTable(sql, params, dataTable) - Update with DataAdapter
- BulkCopy(destTable, dataTable) - Bulk insert
- ChangeDB(dbName) - Switch database

Properties:
- Transaction - Current SQL transaction
- LastError - Most recent database error
- GetServer, GetDatabase - Connection details

Static Methods:
- TestSQLConnectionString(connectionString, sql) - Test connection
```

---

## File Location Quick Reference

### When Working On

**File Upload Logic:**

- `PDI_Azure_Function/BatchBlob.cs`
- `PDI_Azure_Function/Extensions/BlobServiceExtensions.cs`
- `Publisher_Data_Operations/Helper/PDIBatch.cs`

**File Validation:**

- `PDI_Azure_Function/QueueProcess.cs`
- `Publisher_Data_Operations/Orchestration.cs`
- `Publisher_Data_Operations/Helper/Generic.cs`
- `Publisher_Data_Operations/Extensions/ParameterValidator.cs`

**Data Transformation:**

- `Publisher_Data_Operations/Transform.cs`
- `Publisher_Data_Operations/Extensions/StringExtensions.cs`
- `Publisher_Data_Operations/Extensions/Table.cs`
- `Publisher_Data_Operations/Extensions/Data.cs`

**Allocation Processing:**

- `Publisher_Data_Operations/Extensions/Allocation.cs`
- `Publisher_Data_Operations/Helper/Generic.cs` (French lookup)

**French Translation:**

- `Publisher_Data_Operations/Helper/Generic.cs` (verifyFrenchTableText)
- `Publisher_Data_Operations/Extensions/StringExtensions.cs` (formatting)

**Scenario Matching:**

- `Publisher_Data_Operations/Extensions/Scenario.cs`
- `Publisher_Data_Operations/Helper/Generic.cs` (scenario loading)

**Database Operations:**

- `Publisher_Data_Operations/Extensions/DBConnection.cs`
- `PDI_Azure_Function/QueueImport.cs` (import trigger)
- `Publisher_Data_Operations/Orchestration.cs` (PublisherImport method)

**Email Notifications:**

- `Publisher_Data_Operations/Helper/PDISendGrid.cs`
- `PDI_Azure_Function/QueueImport.cs` (calls SendGrid methods)
- `PDI_Azure_Function/QueueProcess.cs` (error emails)

**Status Monitoring:**

- `PDI_Azure_Function/StatusPage.cs`
- `Publisher_Data_Operations/Helper/Generic.cs` (GetStatusTable)

**Queue Management:**

- `PDI_Azure_Function/Extensions/QueueServiceExtensions.cs`
- `PDI_Azure_Function/QueueProcess.cs`
- `PDI_Azure_Function/QueueImport.cs`

---

## Database Objects

### PDI Database Tables

Located in SQL Server PDI Database:

**Core Tables:**

- `pdi_File` - File tracking
- `pdi_Batch_File` - Batch tracking
- `pdi_Batch_Entry` - Individual batch entries
- `pdi_Job` - Processing jobs
- `pdi_Data_Staging` - Staged data for import

**Reference Tables:**

- `pdi_Document_Type` - Document type definitions
- `pdi_Client` - Client/fund family definitions
- `pdi_Line_Of_Business` - LOB definitions
- `pdi_Data_Type` - Data type definitions
- `pdi_Document_Life_Cycle_Status` - Fund age status

**Translation & Content:**

- `pdi_Global_Text_Language` - Bilingual text library
- `pdi_Scenario` - Conditional content rules
- `pdi_Scenario_Flag` - Scenario conditions

**Supporting Tables:**

- `pdi_Validation_Messages` - Validation errors/warnings
- `pdi_File_Receipt_Log` - Audit log
- `pdi_Allocation` - Fund allocations
- `pdi_Table_Data` - Table data

### Publisher Database Objects

Located in SQL Server Publisher Database:

**Tables:**

- `Publisher_Jobs` - Import jobs
- `Publisher_Documents` - Published documents
- `Publisher_Document_Fields` - Document field values
- `Publisher_Clients` - Client definitions
- `Publisher_Document_Types` - Document type definitions

**Stored Procedures:**

- `sp_ImportDocument` - Main import procedure

---

## Azure Resources

### Blob Storage Containers

## Container: IncomingBatchContainer

- Purpose: Initial file upload location
- Trigger: BatchBlob function

## Container: MainContainer

- Folders:
  - `/processing` - Files being validated
  - `/importing` - Files being imported to Publisher
  - `/completed` - Successfully processed files
  - `/rejected` - Files with errors
  - `/archive` - Archived ZIP files
  - `/templates` - Validation templates

### Storage Queues

## Queue: PDI_Queue

- Purpose: File validation and transformation stage
- Trigger: QueueProcess function
- Message: DataTransferObject with Batch_ID, FileName, RetryCount

## Queue: PUB_Queue

- Purpose: Publisher database import stage
- Trigger: QueueImport function
- Message: DataTransferObject with Job_ID, File_ID

---

## Development Workflow File Paths

### Adding New Document Type

1. Add enum value: `Publisher_Data_Operations/Extensions/DataEnums.cs`
2. Add template: Upload to `MainContainer/templates/[DocumentType]_Template.xlsx`
3. Add validation: `Publisher_Data_Operations/Helper/Generic.cs` - ValidateFile method
4. Add transformation: `Publisher_Data_Operations/Transform.cs` - ProcessTables method

### Adding New Validation Rule

1. Add rule logic: `Publisher_Data_Operations/Helper/Generic.cs` - ValidateFile method
2. Add validation method: `Publisher_Data_Operations/Extensions/ParameterValidator.cs`
3. Add error messages: `Publisher_Data_Operations/Helper/Logger.cs`
4. Test in: `PDI_Azure_Function/QueueProcess.cs` - validation stage

### Adding New Number Format

1. Add format method: `Publisher_Data_Operations/Extensions/StringExtensions.cs`
2. Update table formatting: `Publisher_Data_Operations/Extensions/Table.cs`
3. Update transform logic: `Publisher_Data_Operations/Transform.cs`

### Adding New Table Type

1. Add enum value: `Publisher_Data_Operations/Extensions/Table.cs` - TableTypes enum
2. Add processing logic: `Publisher_Data_Operations/Extensions/Table.cs` - GetCellValue method
3. Add transformation: `Publisher_Data_Operations/Transform.cs` - ProcessTables method

### Modifying Email Templates

1. Email sending logic: `Publisher_Data_Operations/Helper/PDISendGrid.cs`
2. Trigger points:
   - Success: `PDI_Azure_Function/QueueImport.cs`
   - Error: `PDI_Azure_Function/QueueProcess.cs`, `PDI_Azure_Function/QueueImport.cs`
   - Batch complete: `PDI_Azure_Function/QueueImport.cs`

### Adding New Scenario Field

1. Database: Add field to `pdi_Scenario` table
2. Parsing logic: `Publisher_Data_Operations/Extensions/Scenario.cs` - ParseScenario method
3. Matching logic: `Publisher_Data_Operations/Extensions/Scenario.cs` - MatchFields method
4. Loading: `Publisher_Data_Operations/Helper/Generic.cs` - scenario loading

### Adding New Extension Method

**For Azure Functions:**

- Location: `PDI_Azure_Function/Extensions/`
- Add new .cs file or add to existing extension class
- Reference in function files as needed

**For Data Processing:**

- Location: `Publisher_Data_Operations/Extensions/`
- Add new .cs file or add to existing extension class
- Follow naming pattern: [Type]Extensions.cs

---

## Debugging File Locations

### When Debugging Upload Issues

**Check:**

1. `PDI_Azure_Function/BatchBlob.cs` - Blob trigger activation
2. `PDI_Azure_Function/Extensions/BlobServiceExtensions.cs` - SaveTo, MoveTo methods
3. `Publisher_Data_Operations/Helper/PDIBatch.cs` - Batch creation
4. Azure Portal: IncomingBatchContainer contents

**Common Issues:**

- File not triggering: Check blob connection string in `local.settings.json`
- File not moving: Check container permissions
- Batch not creating: Check database connection

### When Debugging Validation Errors

**Check:**

1. `Publisher_Data_Operations/Helper/Generic.cs` - ValidateFile method
2. `Publisher_Data_Operations/Extensions/ParameterValidator.cs` - Validation methods
3. `PDI_Azure_Function/StatusPage.cs` - ValidationMessages display
4. Database: `pdi_Validation_Messages` table

**Log Locations:**

- Azure Application Insights
- `Publisher_Data_Operations/Helper/Logger.cs` - Error logging

### When Debugging Transformation Issues

**Check:**

1. `Publisher_Data_Operations/Transform.cs` - ProcessTables method
2. `Publisher_Data_Operations/Extensions/StringExtensions.cs` - Format methods
3. `Publisher_Data_Operations/Extensions/Table.cs` - Table generation
4. `Publisher_Data_Operations/Extensions/Data.cs` - DataTable operations

**Test Data:**

- Input: Excel file in `/processing` folder
- Output: `pdi_Data_Staging` table
- Logs: Application Insights

### When Debugging French Translation Issues

**Check:**

1. `Publisher_Data_Operations/Helper/Generic.cs` - verifyFrenchTableText method
2. Database: `pdi_Global_Text_Language` table
3. `Publisher_Data_Operations/Extensions/Allocation.cs` - AddCode method (calls French lookup)

**Query Database:**

```sql
SELECT * FROM pdi_Global_Text_Language 
WHERE English_Text LIKE '%search term%'
ORDER BY Last_Updated DESC
```

### When Debugging Import Issues

**Check:**

1. `PDI_Azure_Function/QueueImport.cs` - Import trigger
2. `Publisher_Data_Operations/Orchestration.cs` - PublisherImport method
3. `Publisher_Data_Operations/Extensions/DBConnection.cs` - ExecuteScalar method
4. Publisher Database: `sp_ImportDocument` stored procedure
5. Publisher Database: `Publisher_Jobs`, `Publisher_Documents` tables

**Query Status:**

```sql
-- Check PDI staging
SELECT TOP 10 * FROM pdi_Data_Staging 
WHERE Job_ID = [your_job_id]

-- Check Publisher import
SELECT * FROM Publisher_Jobs WHERE Job_ID = [your_job_id]
SELECT * FROM Publisher_Documents WHERE Job_ID = [your_job_id]
```

### When Debugging Queue Issues

**Check:**

1. `PDI_Azure_Function/Extensions/QueueServiceExtensions.cs` - AddMessage method
2. `PDI_Azure_Function/QueueProcess.cs` - PDI_Queue trigger
3. `PDI_Azure_Function/QueueImport.cs` - PUB_Queue trigger
4. Azure Portal: Queue contents and poison queue

**Monitor:**

- `PDI_Azure_Function/StatusPage.cs` - GetQueueLength display
- Application Insights: Queue trigger executions
- Queue configuration: `PDI_Azure_Function/host.json`

---

## Configuration File Details

### host.json

```note
Location: PDI_Azure_Function/host.json

Purpose: Function app configuration

Key Settings:
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": { "isEnabled": true }
    }
  },
  "extensions": {
    "queues": {
      "batchSize": 4,  // Process 4 messages at once
      "newBatchThreshold": 1
    }
  }
}

Modify For:
- Queue batch size: Change "batchSize" value
- Logging levels: Modify "LogLevel" settings
```

### local.settings.json

```note
Location: PDI_Azure_Function/local.settings.json
Status: NOT in source control (in .gitignore)

Purpose: Local development environment variables

Required Variables:
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet",
    "sapdi": "[Azure Storage Connection String]",
    "PDI_ConnectionString": "[PDI Database Connection]",
    "PUB_ConnectionString": "[Publisher Database Connection]",
    "IncomingBatchContainer": "[Container Name]",
    "MainContainer": "[Container Name]",
    "PDI_QueueName": "[Queue Name]",
    "PUB_QueueName": "[Queue Name]",
    "SMTP_Password": "[SendGrid API Key]",
    "SMTP_FromEmail": "[Sender Email]",
    "SMTP_FromName": "[Sender Name]"
  }
}

Setup:
1. Copy local.settings.json.template to local.settings.json
2. Fill in actual connection strings and settings
3. Never commit to source control
```

### .gitignore

```note
Location: PDI_Azure_Function/.gitignore

Purpose: Exclude files from source control

Key Exclusions:
- local.settings.json (secrets)
- bin/, obj/ folders (build output)
- .vs/ folder (Visual Studio settings)
- *.user files (user-specific settings)

Add Custom Exclusions:
- Test data files
- Local database backups
- Personal configuration files
```

### PDI_Azure_Function.csproj

```note
Location: PDI_Azure_Function/PDI_Azure_Function.csproj

Purpose: Project file with dependencies

Key NuGet Packages:
- Azure.Storage.Blobs (v12.14.1)
- Azure.Storage.Queues (v12.12.0)
- Microsoft.Azure.WebJobs.Extensions.Storage (v5.0.1)
- Microsoft.NET.Sdk.Functions (v4.1.3)

Add New Package:
1. Open in Visual Studio
2. Right-click project > Manage NuGet Packages
3. Search and install
4. Or use: dotnet add package [PackageName]
```

---

## Testing File Locations

### Unit Test Locations (if implemented)

**Suggested Structure:**

```file
Solution Root/
├── PDI_Azure_Function.Tests/
│   ├── BatchBlobTests.cs
│   ├── QueueProcessTests.cs
│   └── ExtensionTests.cs
│
└── Publisher_Data_Operations.Tests/
    ├── OrchestrationTests.cs
    ├── TransformTests.cs
    └── ExtensionTests/
        ├── StringExtensionsTests.cs
        ├── TableTests.cs
        └── AllocationTests.cs
```

### Test Data Locations

**Excel Templates:**

```file
Azure Blob Storage: MainContainer/templates/
- MRFP_Template.xlsx
- FS_Template.xlsx
- FF_Template.xlsx
- QPD_Template.xlsx
etc.
```

**Sample Files:**

```file
Local development:
- [Solution]/TestData/SampleFiles/
  - Valid_MRFP.xlsx
  - Valid_FS.xlsx
  - Invalid_MissingSheet.xlsx
  - Invalid_MissingColumn.xlsx
```

---

## Deployment File Locations

### For Publish Profiles

**Development Environment:**

```note
Location: PDI_Azure_Function/Properties/PublishProfiles/fapdidev - Zip Deploy.pubxml

Target:
- Function App: fapdidev
- Resource Group: DEV-PDI-Publisher-ETL
- Region: Canada Central
- Subscription: [Dev Subscription ID]
```

**QA Environment:**

```note
Location: PDI_Azure_Function/Properties/PublishProfiles/fapp-pdi-quality - Zip Deploy.pubxml

Target:
- Function App: fapp-pdi-quality
- Resource Group: QA-PDI-Publisher-ETL
- Region: Canada Central
- Subscription: [QA Subscription ID]
```

### Deployment Steps

1. **Build Solution:**
   - Visual Studio: Build > Build Solution
   - CLI: `dotnet build`

2. **Publish to Azure:**
   - Visual Studio: Right-click project > Publish > Select profile
   - CLI: `dotnet publish -c Release`

3. **Verify Deployment:**
   - Check Azure Portal: Function App > Functions
   - Test: Upload sample file
   - Monitor: Application Insights > Live Metrics

---

## Common File Paths Reference

### Quick Navigation

**Main Function Entry Points:**

```file
📁 PDI_Azure_Function/BatchBlob.cs           → File upload processing
📁 PDI_Azure_Function/QueueProcess.cs        → File validation/transformation
📁 PDI_Azure_Function/QueueImport.cs         → Publisher import
📁 PDI_Azure_Function/StatusPage.cs          → Web status interface
```

**Core Business Logic:**

```file
📁 Publisher_Data_Operations/Orchestration.cs   → Main workflow orchestration
📁 Publisher_Data_Operations/Transform.cs       → Data transformation
📁 Publisher_Data_Operations/Helper/Generic.cs  → Validation & utilities
```

**Data Processing:**

```file
📁 Publisher_Data_Operations/Extensions/StringExtensions.cs  → Text formatting
📁 Publisher_Data_Operations/Extensions/Table.cs             → Table generation
📁 Publisher_Data_Operations/Extensions/Allocation.cs        → Allocation processing
📁 Publisher_Data_Operations/Extensions/Scenario.cs          → Scenario matching
```

**Database Operations:**

```file
📁 Publisher_Data_Operations/Extensions/DBConnection.cs  → Database access
💾 PDI Database                                          → Staging tables
💾 Publisher Database                                    → Final documents
```

**Helper Utilities:**

```file
📁 PDI_Azure_Function/Extensions/BlobServiceExtensions.cs    → Blob operations
📁 PDI_Azure_Function/Extensions/QueueServiceExtensions.cs   → Queue operations
📁 Publisher_Data_Operations/Helper/Logger.cs                → Error logging
📁 Publisher_Data_Operations/Helper/PDISendGrid.cs           → Email notifications
```

---

## Documentation Locations

### Code Documentation

**Inline Comments:**

- Found throughout all .cs files
- Method summaries using /// XML comments
- Complex logic explained with // inline comments

### External Documentation

**This Documentation Set:**

- System Architecture Diagram
- Entity Relationship Diagram
- Use Case Diagram
- Use Case Descriptions
- Business Documentation
- File Location Reference (this document)

**Azure Portal Documentation:**

- Application Insights dashboards
- Function execution logs
- Queue metrics
- Blob storage analytics

---

## File Organization Best Practices

### Naming Conventions

**Functions:**

- Pattern: `[Purpose][Type].cs`
- Examples: `BatchBlob.cs`, `QueueProcess.cs`, `StatusPage.cs`

**Extensions:**

- Pattern: `[Type]Extensions.cs`
- Examples: `BlobServiceExtensions.cs`, `StringExtensions.cs`

**Helpers:**

- Pattern: `PDI[Purpose].cs` or `[Purpose].cs`
- Examples: `PDIBatch.cs`, `PDIStream.cs`, `Generic.cs`

### Folder Structure

## PDI_Azure_Function/

```file
├── *.cs                    → Function definitions
├── Extensions/             → Extension methods
├── Properties/             → Configuration
│   ├── PublishProfiles/   → Deployment profiles
│   └── Service files       → Dependencies
├── host.json              → Function app config
└── local.settings.json    → Local environment (not in repo)
```

## Publisher_Data_Operations/

```file
├── *.cs                    → Core processing classes
├── Extensions/             → Data processing extensions
├── Helper/                 → Helper classes
└── Entities/              → Database entities (if using EF)
```

---

## Troubleshooting Guide by File

### Error: "Blob trigger not firing"

**Check Files:**

1. `PDI_Azure_Function/BatchBlob.cs` - Trigger attribute
2. `PDI_Azure_Function/local.settings.json` - Connection string
3. Azure Portal - Storage account connectivity

### Error: "French translation not found"

**Check Files:**

1. `Publisher_Data_Operations/Helper/Generic.cs` - verifyFrenchTableText
2. Database: `pdi_Global_Text_Language` table
3. `Publisher_Data_Operations/Extensions/Allocation.cs` - Translation calls

### Error: "Invalid XML in table"

**Check Files:**

1. `Publisher_Data_Operations/Extensions/ParameterValidator.cs` - IsValidXML
2. `Publisher_Data_Operations/Extensions/StringExtensions.cs` - CleanXML
3. `Publisher_Data_Operations/Extensions/Data.cs` - ValidateXML

### Error: "Staging data failed"

**Check Files:**

1. `Publisher_Data_Operations/Helper/Generic.cs` - StageData method
2. `Publisher_Data_Operations/Extensions/DBConnection.cs` - BulkCopy method
3. Database: `pdi_Data_Staging` table structure

### Error: "Import to Publisher failed"

**Check Files:**

1. `PDI_Azure_Function/QueueImport.cs` - Import trigger
2. `Publisher_Data_Operations/Orchestration.cs` - PublisherImport
3. `Publisher_Data_Operations/Extensions/DBConnection.cs` - ExecuteScalar
4. Publisher Database: `sp_ImportDocument` stored procedure

---

## Version Control Information

### Git Repository Structure

**Branches:**

- `main` or `master` - Production code
- `develop` - Development branch
- `feature/*` - Feature branches
- `bugfix/*` - Bug fix branches
- `release/*` - Release branches

### Files NOT in Source Control

Per `.gitignore`:

- `local.settings.json` (secrets)
- `bin/`, `obj/` (build output)
- `.vs/` (IDE settings)
- `*.user` files (personal settings)
- Test data files with sensitive information

### Files ALWAYS in Source Control

- All `.cs` source files
- `.csproj` project files
- `host.json` configuration
- `.gitignore` itself
- `README.md` documentation
- Publish profiles (`.pubxml`)

---

## Summary: Key Files by Function

| Function | Primary Files |
|----------|--------------|
| **File Upload** | BatchBlob.cs, BlobServiceExtensions.cs, PDIBatch.cs |
| **Validation** | QueueProcess.cs, Generic.cs, ParameterValidator.cs |
| **Transformation** | Transform.cs, StringExtensions.cs, Table.cs, Data.cs |
| **Allocation** | Allocation.cs, Generic.cs (French lookup) |
| **Scenarios** | Scenario.cs, Generic.cs (scenario loading) |
| **Staging** | Generic.cs, DBConnection.cs |
| **Import** | QueueImport.cs, Orchestration.cs, DBConnection.cs |
| **Email** | PDISendGrid.cs |
| **Status** | StatusPage.cs, Generic.cs |
| **Logging** | Logger.cs, Application Insights |

---

## Quick File Finder

**Need to modify currency formatting?**
→ `Publisher_Data_Operations/Extensions/StringExtensions.cs` - ToCurrency method

**Need to change queue batch size?**
→ `PDI_Azure_Function/host.json` - "batchSize" setting

**Need to add new document type?**
→ `Publisher_Data_Operations/Extensions/DataEnums.cs` - DocumentTypeID enum

**Need to change validation rules?**
→ `Publisher_Data_Operations/Helper/Generic.cs` - ValidateFile method

**Need to modify email templates?**
→ `Publisher_Data_Operations/Helper/PDISendGrid.cs` - Send methods

**Need to add new table type?**
→ `Publisher_Data_Operations/Extensions/Table.cs` - TableTypes enum + logic

**Need to troubleshoot blob operations?**
→ `PDI_Azure_Function/Extensions/BlobServiceExtensions.cs`

**Need to troubleshoot database operations?**
→ `Publisher_Data_Operations/Extensions/DBConnection.cs`
