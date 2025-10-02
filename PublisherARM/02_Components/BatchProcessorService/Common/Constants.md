# Constants — Configuration and System Settings (analysis)

## One-line purpose

Centralized configuration management class that provides system-wide constants and settings loaded from app.config, including Azure services configuration, batch processing parameters, cultural localization, file paths, and integration endpoints.

---

## Files analyzed

* `BatchProcessorService/Constants.cs`

---

## What this code contains (high level)

1. **Cultural Localization Settings** — Bilingual support configuration for English/French Canadian locales
2. **Azure Storage Configuration** — Queue names, container names, connection strings, and retry policies
3. **Batch Processing Parameters** — Thread limits, timeouts, retry counts, and processing intervals
4. **File System Paths** — Temporary storage locations and template file paths
5. **External Service Integration** — Composition engine, mail service, and BlueID authentication settings
6. **System Operational Constants** — MIME types, XML attributes, timezone settings, and pagination

This class serves as the single source of truth for all configurable system parameters and acts as the bridge between app.config settings and application code.

---

## Classes, properties and methods (developer reference)

### `Constants` - Static Configuration Container

**Purpose:** Provides centralized access to all system configuration with automatic type conversion and default value handling.

#### Cultural Localization Constants

##### Supported Cultures

* `public readonly static string[] DOCUMENT_CULTURE_CODES = { "en-CA", "fr-CA" }` — Active culture support array
* `public const string DEFAULT_CULTURE_CODE = "en-CA"` — System fallback culture
* `public const string ENGLISH_CULTURE_CODE = "en-CA"` — English Canadian locale
* `public const string FRENCH_CULTURE_CODE = "fr-CA"` — French Canadian locale

**Historical Note:**

```csharp
// Commented out Chinese support
//public static string[] DOCUMENT_CULTURE_CODES = { "en-CA", "fr-CA", "zh-CHS" };
//public const string CHINESE_CULTURE_CODE = "zh-CHS";
```

Indicates previous or planned Chinese localization support that has been disabled.

##### Regional Settings

* `public readonly static System.TimeZoneInfo DEFAULT_TIMEZONE = System.TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time")` — EST with DST handling

#### Azure Infrastructure Configuration

##### Storage Account Settings

* `public readonly static string StorageAccountConnectionString = ConfigurationManager.AppSettings["StorageAccount.ConnectionString"]` — Primary Azure Storage connection
* `public readonly static Microsoft.WindowsAzure.Storage.RetryPolicies.IRetryPolicy StorageAccountRetryPolicy = new Microsoft.WindowsAzure.Storage.RetryPolicies.ExponentialRetry()` — Default retry policy

##### Queue Names (Lowercase Required)

* `public const string BATCH_IMPORT_EXPORT_PROCESSING_QUEUE_NAME = "batch-processing-import-export-queue"` — Import/Export operations queue
* `public const string BATCH_NON_IMPORT_EXPORT_PROCESSING_QUEUE_NAME = "batch-processing-non-import-export-queue"` — Other batch operations queue
* `public readonly static string BATCH_PROCESSING_QUEUE_NAME = ConfigurationManager.AppSettings["StorageAccount.Quename"]` — Primary processing queue (configurable)

**Queue Architecture Pattern:**
The system uses separate queues for different batch types, aligning with the threading strategy observed in Program.cs where Import/Export get unlimited threading while others use limited concurrency.

##### Container Names (Lowercase Required)

* `public const string BATCH_PROCESSING_PACKAGE_CONTAINER_NAME = "batch-processing-packages"` — Package batch storage
* `public const string BOOK_PROCESSING_PACKAGE_CONTAINER_NAME = "book-processing-packages"` — Book batch storage
* `public const string DocumentArchiveContainerName = "document-archive"` — Archive storage
* `public const string DATA_EXPORT_CONTAINER_NAME = "data-export"` — Export result storage
* `public const string DATA_IMPORT_CONTAINER_NAME = "data-import"` — Import source storage
* `public const string DATA_REPORT_CONTAINER_NAME = "document-report"` — Report output storage
* `public const string REQUEST_LOG_CONTAINER_NAME = "request-logs"` — Diagnostic logs storage

#### Batch Processing Configuration

##### Performance Parameters

* `public readonly static int MAX_BATCH_PROCESSING_THREADS = HandyFunctions.assumeDefault(ConfigurationManager.AppSettings["BatchProcessing.MaxBatchProcessingThreads"], 5)` — Concurrency limit (default: 5)
* `public readonly static int BatchProcessingQueueCheckInterval = HandyFunctions.assumeDefault(ConfigurationManager.AppSettings["BatchProcessing.QueueCheckInterval"], 10)` — Polling interval in seconds (default: 10)
* `public readonly static int BATCH_PROCESSING_TIMEOUT = HandyFunctions.assumeDefault(ConfigurationManager.AppSettings["BatchProcessing.Timeout"], 7200)` — Processing timeout in seconds (default: 2 hours)

##### Reliability Parameters

* `public const int BATCH_PROCESSING_MAX_RETRIES = 5` — Maximum retry attempts per batch
* `public const int BATCH_START_OFFSET_HOURS = 1` — Scheduling offset in hours
* `public readonly static TimeSpan MAX_QUEUE_ITEM_LIFETIME = new TimeSpan(1, 0, 0, 0, 0)` — 1-day queue message TTL

##### Operational Settings

* `public readonly static bool BatchProcessingLogRequests = HandyFunctions.convertToBool(ConfigurationManager.AppSettings["BatchProcessing.LogRequests"])` — Request logging toggle

#### File System Configuration

##### Temporary Storage Paths

* `public readonly static string ARCHIVE_FILE_TEMP_PATH = AppDomain.CurrentDomain.BaseDirectory + ConfigurationManager.AppSettings["ArchiveFileStorage"]` — Archive processing temporary location
* `public readonly static string EXCEL_TEMP_PATH = AppDomain.CurrentDomain.BaseDirectory + ConfigurationManager.AppSettings["TempExcelFileStorage"]` — Excel processing temporary location

##### Template Paths

* `public readonly static string EXCEL_EXPORT_TEMPLATE_PATH = AppDomain.CurrentDomain.BaseDirectory + "Resources\\DataExportTemplate.xlsx"` — Export template file location

**Path Construction Pattern:** All paths combine `AppDomain.CurrentDomain.BaseDirectory` with configuration values for deployment flexibility.

#### External Service Integration

##### Composition Engine Settings

* `public readonly static int COMPOSITION_WEB_SERVICE_TIMEOUT = HandyFunctions.assumeDefault(ConfigurationManager.AppSettings["CompositionEngine.Timeout"], 120) * 1000` — Timeout in milliseconds (default: 2 minutes)
* `public readonly static bool CompositionEngineLogRequests = HandyFunctions.convertToBool(ConfigurationManager.AppSettings["CompositionEngine.LogRequests"])` — Request logging toggle

##### BlueID Authentication Integration

* `public readonly static string blueIDCompositionEngineWebserviceURL = ConfigurationManager.AppSettings["BlueIDCompositionEngineWebservice.URL"]` — Service endpoint
* `public readonly static string blueIDCompositionEngineWebserviceUsername = ConfigurationManager.AppSettings["BlueIDCompositionEngineWebservice.Username"]` — Authentication username
* `public readonly static string blueIDCompositionEngineWebservicePassword = ConfigurationManager.AppSettings["BlueIDCompositionEngineWebservice.Password"]` — Authentication password

#### Email Configuration

##### SMTP Settings

* `public readonly static string MAIL_HOST = ConfigurationManager.AppSettings["smtp.sendgrid.net"]` — SendGrid SMTP host
* `public readonly static int MAIL_PORT = HandyFunctions.assumeDefault(ConfigurationManager.AppSettings["Mail.Port"], 1025)` — SMTP port (default: 1025)
* `public readonly static string MAIL_USERNAME = ConfigurationManager.AppSettings["Mail.Username"]` — SMTP authentication username
* `public readonly static string MAIL_PASSWORD = ConfigurationManager.AppSettings["Mail.Password"]` — SMTP authentication password

##### Error Notification Settings

* `public readonly static List<string> errorEmailRecipients = ConfigurationManager.AppSettings["ErrorEmail.RecipientList"].Split(',').ToList<string>()` — Error notification recipients
* `public readonly static string errorEmailSender = ConfigurationManager.AppSettings["ErrorEmail.Sender"]` — Error notification sender address

#### Database Configuration

##### Connection Settings

* `public readonly static string connectionString = ConfigurationManager.AppSettings["FDPublisher.DbConnection"]` — Primary database connection string
* `public readonly static int LONG_RUNNING_QUERY_COMMAND_TIMEOUT = HandyFunctions.assumeDefault(ConfigurationManager.AppSettings["LongRunningQueryCommand.Timeout"], 600)` — Extended query timeout in seconds (default: 10 minutes)

#### System Operational Constants

##### MIME Type Definitions

* `public const string MIME_TYPE_PDF = "application/pdf"` — PDF document type
* `public const string MIME_TYPE_XML = "text/xml"` — XML document type
* `public const string MIME_TYPE_EXCEL = "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"` — Excel document type
* `public const string MIME_TYPE_DEFAULT = "application/octet-stream"` — Generic binary type
* `public const string MIME_TYPE_ZIP = "application/octet-stream"` — ZIP archive type

##### XML Processing Constants

* `public const string XML_ROW_ORDER_ATTRIBUTE = "roworder"` — Table row ordering attribute
* `public const string XML_ROW_TYPE_ATTRIBUTE = "rowtype"` — Table row type attribute
* `public const string XML_EMPTY_TABLE = "<table><row /></table>"` — Empty table XML structure

##### User Interface Constants

* `public const int ITEMS_PER_PAGE = 20` — Pagination size
* `public const int VISIBLE_PAGE_LINKS = 10` — Navigation link count
* `public const bool IS_AUDITREPORT_VISIBLE = true` — Audit report feature toggle

##### Application Monitoring

* `public readonly static string APPLICATION_INSIGHTS_INSTRUMENTATION_KEY = ConfigurationManager.AppSettings["ApplicationInsights.InstrumentationKey"]` — Azure Application Insights key
* `public const string EventLogTableName = "EventLog"` — Azure Storage logging table name

##### Specialized Processing Constants

* `public static List<string> AdditionalValuesToIncludeInTableSync = new List<string>(new string[] { "-", "−", "—" })` — Unicode dash characters for table synchronization (Unicode: 002D, 2212, 2014)

##### Document Comparison Integration

* `public readonly static string COMPARE_DOCS_LIC_NAME = ConfigurationManager.AppSettings["CompareDocsLicName"]` — Document comparison license name
* `public readonly static string COMPARE_DOCS_LIC_KEY = ConfigurationManager.AppSettings["CompareDocsLicKey"]` — Document comparison license key

##### Environment Configuration

* `public readonly static string environmentName = ConfigurationManager.AppSettings["env"]` — Environment identifier (dev/staging/prod)
* `public readonly static Uri BASE_URL = new Uri(ConfigurationManager.AppSettings["BaseURL"])` — Application base URL

---

## Business logic and integration patterns

### Configuration Management Strategy

The class employs a hybrid approach combining compile-time constants and runtime configuration:

**Compile-Time Constants:** Used for values that never change across environments

* MIME types
* XML attribute names
* Cultural codes
* Retry limits

**Runtime Configuration:** Used for environment-specific values

* Connection strings
* Service URLs
* File paths
* Performance parameters

### Default Value Handling Pattern

Extensive use of `HandyFunctions.assumeDefault()` provides graceful degradation:

```csharp
public readonly static int BATCH_PROCESSING_TIMEOUT = HandyFunctions.assumeDefault(
    ConfigurationManager.AppSettings["BatchProcessing.Timeout"], 7200);
```

**Benefits:**

* Systems remain functional with missing configuration
* Clear documentation of expected values
* Consistent behavior across environments

### Cultural Localization Architecture

Bilingual support is embedded throughout the system:

* **Primary Languages:** English Canadian (en-CA) and French Canadian (fr-CA)
* **Default Fallback:** English Canadian for missing translations
* **Timezone Consistency:** Eastern Standard Time with automatic DST handling

### Azure Services Integration

Comprehensive Azure cloud integration strategy:

**Storage Queues:** Separate queues for different processing types
**Blob Storage:** Multiple containers for different data types
**Application Insights:** Centralized telemetry and monitoring
**Retry Policies:** Built-in resilience with exponential backoff

### Email Notification System

SendGrid integration for operational notifications:

* **Error Notifications:** Automatic error alerting to administrator list
* **SMTP Configuration:** Flexible SMTP settings for different environments
* **Recipient Management:** Comma-separated configuration for easy updates

---

## Technical implementation considerations

### Configuration Dependency Management

Heavy reliance on ConfigurationManager creates several considerations:

**Initialization Timing:** Static readonly fields are initialized when class is first accessed
**Error Handling:** Missing configuration values may cause runtime exceptions
**Testing Challenges:** Configuration dependencies make unit testing more complex

### Thread Safety Considerations

Static readonly fields provide thread-safe lazy initialization:

```csharp
public readonly static List<string> errorEmailRecipients = 
    ConfigurationManager.AppSettings["ErrorEmail.RecipientList"].Split(',').ToList<string>();
```

**Thread Safety:** One-time initialization is inherently thread-safe
**Performance:** No synchronization overhead after initialization

### Memory Usage Patterns

Configuration values loaded once and cached for application lifetime:

* **Memory Efficiency:** Single instance of each configuration value
* **Performance:** No repeated configuration reads
* **Memory Footprint:** Minimal - mostly string and primitive values

### Error Handling Gaps

Current implementation has limited error handling for configuration issues:

* **Missing Keys:** May throw exceptions at startup
* **Invalid Formats:** Type conversion failures not handled
* **Connection String Issues:** Database connectivity problems not validated at startup

---

## Integration with broader Publisher system

### Batch Processing Integration

Constants directly support the batch processing architecture observed in Program.cs:

* **Thread Limits:** `MAX_BATCH_PROCESSING_THREADS` controls concurrency
* **Queue Names:** Support for separate Import/Export vs other batch queues
* **Timeout Values:** `BATCH_PROCESSING_TIMEOUT` prevents runaway processes

### Document Processing Pipeline

File processing constants support the document workflow:

* **Temporary Paths:** Staging areas for file processing
* **Template Locations:** Excel export template integration
* **MIME Types:** Content type detection and handling

### Cultural Compliance

Bilingual support meets Canadian regulatory requirements:

* **Official Languages:** English and French Canadian locales
* **Timezone Handling:** Eastern time zone for Canadian financial markets
* **Cultural Formatting:** Support for Canadian-specific formatting rules

### Monitoring and Diagnostics

Comprehensive monitoring configuration:

* **Application Insights:** Cloud-based telemetry
* **Request Logging:** Configurable detailed logging
* **Error Notifications:** Automated alerting system

---

## Potential enhancements

### Configuration Management Improvements

1. **Validation Layer:** Validate all configuration values at startup
2. **Configuration Objects:** Strongly-typed configuration sections
3. **Environment Detection:** Automatic environment-specific configuration
4. **Configuration Reloading:** Support for runtime configuration updates

### Error Handling Enhancements

1. **Startup Validation:** Comprehensive configuration validation during startup
2. **Graceful Degradation:** Fallback values for non-critical settings
3. **Configuration Monitoring:** Alert on configuration-related errors

### Security Improvements

1. **Credential Encryption:** Encrypt sensitive configuration values
2. **Configuration Auditing:** Log configuration access and changes
3. **Least Privilege:** Separate read-only configuration access

### Maintainability Enhancements

1. **Configuration Documentation:** Self-documenting configuration schema
2. **Dependency Injection:** Replace static access with dependency injection
3. **Configuration Categorization:** Group related settings into logical sections

---

## Action items for system maintenance

1. **Configuration Validation:** Implement startup validation for all required configuration values
2. **Security Audit:** Review and secure sensitive configuration values (passwords, connection strings)
3. **Dead Configuration Cleanup:** Remove unused configuration keys (e.g., Chinese culture support)
4. **Environment Parity:** Ensure configuration consistency across dev/staging/prod environments
5. **Performance Monitoring:** Monitor the impact of current timeout and thread limit settings
6. **Documentation Updates:** Document the purpose and valid values for each configuration setting
7. **Migration Planning:** Consider migrating to more modern configuration systems (.NET Core's IConfiguration)

---

## Critical considerations

### Configuration Security

Several security-sensitive values are stored in plaintext configuration:

* Database connection strings
* SMTP passwords  
* BlueID service credentials
* License keys

**Recommendation:** Consider implementing configuration encryption or moving sensitive values to Azure Key Vault.

### Environment Configuration Management

The system relies heavily on app.config for environment-specific settings, which can lead to:

* **Configuration Drift:** Different values across environments
* **Deployment Complexity:** Manual configuration management
* **Error Prone:** Risk of incorrect configuration in production

**Recommendation:** Consider implementing environment-specific configuration files or cloud-based configuration management.
