# Class: StatusPage

**One-line purpose**  
HTTP-triggered Azure Functions class that provides a web-based monitoring dashboard, file upload interface, blob download capabilities, validation message viewing, and Event Grid integration for the PDI system.

---

## What this class does (business view)

- Provides a browser-accessible status dashboard showing system health, queue lengths, blob container contents, and recent processing jobs.
- Enables web-based file upload directly to the incoming batch container, bypassing external FTP or storage tools.
- Allows users to download processed files from blob storage via HTTP endpoints.
- Displays detailed validation messages for files that failed processing or encountered warnings.
- Implements Event Grid integration for handling external system events (e.g., notifications from Publisher or partner systems).
- Serves as the primary user interface for monitoring and interacting with the PDI processing pipeline.
- Implements pagination for job history browsing.
- Provides real-time system health checks for both PDI and Publisher databases.

---

## Key functions & triggers (what to look for)

- **Status Page (GET /api/Status/{offset?})** — Main dashboard with system overview, queue metrics, container listings, file upload form, and paginated job history.
- **File Upload (POST /api/FileUpload)** — Accepts multipart form file uploads and places them into the incoming batch container with `web/` prefix.
- **Blob Download (GET /api/DownloadBlob/{fileName})** — Streams blob content as downloadable file with proper content headers.
- **Validation Messages (GET /api/ValidationMessages/{fileName})** — Displays detailed validation errors and warnings for specific files.
- **Event Grid Handler (POST /api/EventGridHandlerHTTP)** — Receives Event Grid webhook notifications and processes them via Orchestration.
- **Authentication Context** — Captures Azure Easy Auth headers for user identity tracking (principal name, ID, IDP).
- **Client-Side JavaScript** — Implements AJAX file upload with feedback UI.

---

## Classes & Methods (detailed)

### `StatusPage` — file: `PDI_Azure_Function/StatusPage.cs`

**Purpose:** Static class containing HTTP-triggered Azure Functions that provide web UI, monitoring, and interactive capabilities for the PDI system. Acts as the primary human interface for system operations.

**Key Characteristics:**

- All functions use `AuthorizationLevel.Anonymous` (relies on Azure Easy Auth for security)
- Implements Bootstrap 5 for UI styling
- Uses `StringBuilder` pattern for HTML generation
- Captures authentication context from Azure Easy Auth headers
- Implements pagination with configurable page size (10 records)
- Uses extension methods for blob and queue operations

---

### Methods

#### `Run` (Status Page)

```csharp
public static async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "Status/{offSet?}")] HttpRequest req, 
    int? offSet, 
    ILogger log, 
    ClaimsPrincipal claimIdentity)
```

**Purpose:** Main HTTP endpoint that generates and returns the status dashboard HTML page with system metrics, queue information, and job history.

**Parameters:**

- `req` (HttpRequest) — Incoming HTTP request with headers and query parameters
- `offSet` (int?, optional) — Pagination offset for job history table (default: 0)
- `log` (ILogger) — Azure Functions logger
- `claimIdentity` (ClaimsPrincipal) — Azure Easy Auth claims principal

**Workflow:**

1. **Authentication Context Extraction**
   - Logs `claimIdentity.Identity.Name` from claims principal
   - Extracts Easy Auth headers:
     - `X-MS-CLIENT-PRINCIPAL-NAME` — User's display name
     - `X-MS-CLIENT-PRINCIPAL-ID` — User's unique ID
     - `X-MS-CLIENT-PRINCIPAL-IDP` — Identity provider (AAD, Google, etc.)
     - `X-MS-CLIENT-PRINCIPAL` — Base64-encoded claims JSON
   - Logs all authentication context for audit trail

2. **HTML Generation**
   - Calls `GetStatusHTMLPage(offSet, req)` to build complete HTML
   - Sets `ContentResult` with HTML content and `text/html` content type
   - Returns HTTP 200 with rendered page

**Security Notes:**

- Function is `Anonymous` but relies on Azure Easy Auth at platform level
- All user context captured in headers (not from token validation)
- Suitable for internal tools behind corporate authentication

**Return Value:** `IActionResult` with HTML content

---

#### `GetStatusHTMLPage`

```csharp
public static string GetStatusHTMLPage(int? offSet, HttpRequest req)
```

**Purpose:** Generates the complete HTML content for the status dashboard, including system health checks, queue metrics, blob listings, and paginated job history.

**Parameters:**

- `offSet` (int?, optional) — Pagination offset for job history
- `req` (HttpRequest) — Used to extract base URL for relative links

**Workflow:**

1. **HTML Header Generation**
   - Calls `GetHeader("Status Page - " + Guid.NewGuid())`
   - Includes Bootstrap 5 CSS via CDN
   - Adds favicon from investorcom.com domain

2. **Database Connection Initialization**
   - Creates `DBConnection` for PDI database
   - Creates `DBConnection` for Publisher database
   - Initializes `Logger` and `Generic` helper objects

3. **Environment Information Display**
   - Displays `SMTP_FromName` environment variable as page title
   - Shows PDI database server and database name
   - Shows Publisher database server and database name

4. **Database Health Checks**
   - **PDI Database:** Executes `SELECT Count(*) FROM [pdi_Global_Text_Language]`
   - **Publisher Database:** Executes `SELECT Count(*) FROM [Roles]`
   - Displays red alert banner if either connection fails
   - Uses `DBConnection.TestSQLConnectionString()` for validation

5. **Blob Container Listing**
   - Connects to `IncomingBatchContainer` via `BlobServiceClient`
   - Calls `batchBlob.GetListing(10)` extension method
   - Displays container name and top 10 blob entries

6. **Queue Length Metrics**
   - **PDI Queue:** Displays `PDI_QueueName` queue length
   - **Publisher Queue:** Displays `PUB_QueueName` queue length
   - Uses `queueClient.GetQueueLength()` extension method

7. **File Upload Form**
   - Renders Bootstrap-styled file input control
   - Includes hidden alert div for upload feedback
   - JavaScript handles AJAX submission (no page reload)

8. **Pagination Controls**
   - **Previous Button:**
     - Disabled if `offSet < PageSize`
     - Active with link to `Status/{offSet - PageSize}` otherwise
   - **Next Button:**
     - Disabled if result set < PageSize (last page)
     - Active with link to `Status/{offSet + PageSize}` otherwise

9. **Job History Table**
   - Calls `gen.GetStatusTable(localOffset, PageSize)` to fetch job records
   - Converts `DataTable` to HTML via `TableToHtml()` extension
   - Displays paginated results with clickable file names and validation links

10. **JavaScript and Footer**
    - Injects `GetJavaScript(baseURL)` for AJAX file upload
    - Closes HTML document with `GetFooter()`

**Return Value:** Complete HTML string ready for rendering

---

#### `GetHeader`

```csharp
public static string GetHeader(string title)
```

**Purpose:** Generates HTML document header with Bootstrap 5 CSS, favicon links, and page title.

**Parameters:**

- `title` (string) — Page title for browser tab

**Features:**

- HTML-encodes title for XSS protection
- Includes Bootstrap 5.1.3 from CDN with integrity hash
- Adds investorcom.com favicon and Apple touch icon
- Opens `<body>` tag for content

**Return Value:** HTML header string

---

#### `GetJavaScript`

```csharp
public static string GetJavaScript(string baseURL = "../")
```

**Purpose:** Generates client-side JavaScript for AJAX file upload functionality with visual feedback.

**Parameters:**

- `baseURL` (string, default: "../") — Base URL for API endpoints

**JavaScript Functionality:**

1. **Event Binding** (`window.onload`)
   - Attaches `change` event listener to `fileUpload` input element
   - Triggers `fileUploaded()` when user selects file

2. **File Upload Handler** (`fileUploaded`)
   - Retrieves selected file from input element
   - Creates `FormData` object and appends file
   - Constructs POST request to `{baseURL}FileUpload` endpoint
   - Calls `makeRequest()` with request properties

3. **AJAX Request Handler** (`makeRequest`)
   - Uses `fetch()` API for asynchronous HTTP POST
   - Checks response status code
   - **Success (200):**
     - Clears file input value
     - Displays "File sent for processing" message
     - Shows hidden alert div
   - **Failure (non-200):**
     - Displays "Something went wrong" message

**User Experience:**

- No page reload on file upload
- Immediate visual feedback
- File input clears after successful upload

**Return Value:** `<script>` block with embedded JavaScript

---

#### `GetFooter`

```csharp
public static string GetFooter()
```

**Purpose:** Closes HTML document body and tags.

**Return Value:** `</body></html>` string

---

#### `TableToHtml` (Extension Method)

```csharp
public static string TableToHtml(this DataTable dt, string baseURL = "../")
```

**Purpose:** Converts `DataTable` to HTML table with Bootstrap styling, special formatting for dates, and clickable links for files and validation messages.

**Parameters:**

- `dt` (DataTable) — Data to render as HTML table
- `baseURL` (string, default: "../") — Base URL for link generation

**Rendering Logic:**

1. **Empty Table Handling**
   - Returns "Empty" string if DataTable is null or has no rows

2. **Table Structure**
   - Generates `<table class='table'>` with Bootstrap styling
   - Creates `<thead>` with column headers
   - Excludes columns containing "_ID" from display (hidden primary keys)
   - Replaces underscores with spaces in column names

3. **Column Type Formatting**
   - **DateTime:** Converts to local time with format `yyyy-MM-dd h:mm:ss tt`
   - **DateTimeOffset:** Converts to local time with same format
   - **File_Name columns:** Creates download link to `DownloadBlob/{fileName}` with Job_ID in title attribute
   - **Max_Message columns:** Creates validation messages link to `ValidationMessages/{File_ID}` (opens in new tab)
   - **All other types:** HTML-encodes value

4. **Cell Styling**
   - Applies `word-wrap: break-word` and `max-width: 160px` for long text handling
   - Prevents table layout breaking from long file names or error messages

5. **XSS Protection**
   - All user content HTML-encoded via `HttpUtility.HtmlEncode()`
   - URL parameters HTML-encoded in href attributes

**Return Value:** HTML table string with Bootstrap classes

---

#### `DownloadBlob`

```csharp
public static async Task<HttpResponseMessage> DownloadBlob(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "DownloadBlob/{fileName}")] HttpRequest req, 
    string fileName, 
    ILogger log)
```

**Purpose:** HTTP endpoint that streams blob content as downloadable file with proper HTTP headers for browser download dialog.

**Parameters:**

- `req` (HttpRequest) — Incoming HTTP request
- `fileName` (string) — File name to download from blob storage
- `log` (ILogger) — Azure Functions logger

**Workflow:**

1. **Blob Client Initialization**
   - Creates `BlobServiceClient` with storage connection string
   - Uses `blobService.Find(fileName, MainContainer)` extension to locate blob
   - Searches across all subdirectories in MainContainer

2. **Blob Existence Check**
   - Calls `blobClient.Exists()` to verify blob presence
   - **If exists:**
     - Retrieves blob properties for metadata
     - Opens blob stream via `blobClient.Open()`
     - Creates `StreamContent` response
   - **If not exists:**
     - Returns HTTP 404 Not Found

3. **HTTP Response Headers**
   - `Content-Length`: Blob size in bytes
   - `Content-Type`: Original blob MIME type
   - `Content-Disposition`: attachment with original filename and size
   - `StatusCode`: 200 OK

**Security Considerations:**

- No path validation (potential directory traversal risk if fileName contains `../`)
- Anonymous access (relies on Azure Easy Auth)
- Searches entire MainContainer (no access control by subdirectory)

**Return Value:** `HttpResponseMessage` with blob stream or 404 status

---

#### `ValidationMessages`

```csharp
public static async Task<IActionResult> ValidationMessages(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "ValidationMessages/{fileName}")] HttpRequest req, 
    string fileName, 
    ILogger log)
```

**Purpose:** HTTP endpoint that displays detailed validation errors and warnings for a specific file in HTML table format.

**Parameters:**

- `req` (HttpRequest) — Incoming HTTP request
- `fileName` (string) — File ID or file name to query validation messages
- `log` (ILogger) — Azure Functions logger

**Workflow:**

1. **Database Connection**
   - Creates `DBConnection` to PDI database
   - Initializes `Logger` and `Generic` helper objects

2. **Validation Message Retrieval**
   - Calls `gen.GetValidationMessages(fileName)` to query database
   - Returns `DataTable` with validation records

3. **HTML Generation**
   - **If fileName has length:**
     - Generates HTML page with header: "Validation - {fileName}"
     - Converts DataTable to HTML table via `TableToHtml()`
     - Adds footer
     - Sets content type to `text/html`
   - **If fileName is empty:**
     - Returns HTTP 404

**Use Case:**

- Linked from status page when `Max_Message` column has content
- Opens in new browser tab for detailed error analysis
- Helps users understand why files failed processing

**Return Value:** `IActionResult` with HTML content or 404 status

---

#### `FileUpload`

```csharp
public static async Task<IActionResult> FileUpload(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = "FileUpload")] HttpRequest req, 
    ILogger log)
```

**Purpose:** HTTP endpoint that accepts multipart form file uploads and places them into the incoming batch container for processing.

**Parameters:**

- `req` (HttpRequest) — Incoming HTTP request with multipart form data
- `log` (ILogger) — Azure Functions logger

**Workflow:**

1. **File Retrieval**
   - Checks `req.Form.Files.Count > 0`
   - Extracts file from form field named "File"
   - Opens file stream via `file.OpenReadStream()`

2. **Blob Upload**
   - Creates `BlobContainerClient` for `IncomingBatchContainer`
   - Gets blob client for `web/{file.FileName}` path
   - **Critical:** Files uploaded via web UI are prefixed with `web/`
   - Calls `blob.UploadAsync(myBlob)` to upload stream

3. **Response Handling**
   - **Success:** Returns `OkObjectResult("File uploaded successfully")` (HTTP 200)
   - **No file:** Returns `NotFoundObjectResult("No file found")` (HTTP 404)
   - **Exception:** Returns `BadRequestObjectResult("An error occurred: " + e.Message)` (HTTP 400)

**File Processing Flow:**

- User uploads via web UI → File lands in `IncomingBatchContainer/web/{fileName}`
- BatchBlob trigger detects upload → Processes normally via standard pipeline
- `web/` prefix distinguishes web uploads from other upload methods (FTP, API, etc.)

**Security Considerations:**

- No file type validation (accepts any file)
- No file size validation (Azure Functions default: 100MB)
- No virus scanning
- Anonymous access (relies on Azure Easy Auth)

**Return Value:** `IActionResult` with success, error, or not found message

---

#### `EventGridHandlerHTTP`

```csharp
public static async Task<IActionResult> EventGridHandlerHTTP(
    [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = "EventGridHandlerHTTP")] HttpRequest req, 
    ILogger log)
```

**Purpose:** HTTP endpoint that receives Event Grid webhook notifications and processes them via the Orchestration layer for system integration.

**Parameters:**

- `req` (HttpRequest) — Incoming HTTP request with Event Grid JSON payload
- `log` (ILogger) — Azure Functions logger

**Workflow:**

1. **Request Body Reading**
   - Reads entire request body as string via `req.ReadAsStringAsync()`
   - Body contains Event Grid event JSON (potentially multiple events)

2. **Event Deserialization**
   - Creates `EventGridSubscriber` instance
   - Calls `DeserializeEventGridEvents(requestBody)` to parse JSON
   - Returns array of `EventGridEvent` objects

3. **Orchestration Initialization**
   - Creates `Orchestration` with both PDI and Publisher connection strings
   - Orchestration handles event processing logic

4. **Event Processing Loop**
   - Iterates through all events in array
   - Calls `orch.HandleEventMessage(eventGridEvent)` for each event
   - Processing logic encapsulated in Orchestration class

5. **Response**
   - Returns `OkObjectResult("File uploaded successfully")` (HTTP 200)
   - **Note:** Success message is misleading (should be "Events processed successfully")

**Event Grid Integration Use Cases:**

- Publisher system sends notifications when data updates occur
- Partner systems trigger PDI processing via events
- Workflow orchestration between multiple Azure services
- Asynchronous system integration

**Commented Code:**

- `CustomerEventHandlerEVG` function commented out (native Event Grid trigger)
- Replaced with HTTP-based handler (more flexible for debugging/testing)

**Return Value:** `IActionResult` with success message

---

## Related Components

- **BatchBlob.cs** — Processes files uploaded to `IncomingBatchContainer` (including `web/` prefixed uploads)
- **QueueProcess.cs** — Processes files through PDI pipeline
- **QueueImport.cs** — Imports files to Publisher database
- **Orchestration** — Handles Event Grid message processing
- **Generic** — Provides `GetStatusTable()` and `GetValidationMessages()` database queries
- **Extension Methods:** `BlobServiceClient.Find()`, `BlobContainerClient.GetListing()`, `QueueClient.GetQueueLength()`

---

## Configuration Dependencies

| Environment Variable | Purpose | Required |
|---------------------|---------|----------|
| `sapdi` | Azure Storage connection string | Yes |
| `PDI_ConnectionString` | PDI database connection | Yes |
| `PUB_ConnectionString` | Publisher database connection | Yes |
| `IncomingBatchContainer` | Blob container for uploads | Yes |
| `MainContainer` | Blob container for processing | Yes |
| `PDI_QueueName` | PDI queue name | Yes |
| `PUB_QueueName` | Publisher queue name | Yes |
| `SMTP_FromName` | Display name for page title | Yes |

---

## Security Considerations

### Authentication

- All functions use `AuthorizationLevel.Anonymous`
- Security relies on Azure Easy Auth (platform-level)
- Authentication context captured but not validated in code
- Suitable for internal tools behind corporate SSO

### Authorization

- No role-based access control implemented
- All authenticated users have full access
- No audit trail for user actions (only logged)

### Input Validation

- **File uploads:** No type, size, or content validation
- **File names:** No sanitization (potential path traversal in DownloadBlob)
- **Pagination offset:** No bounds checking (could cause database errors)

### XSS Protection

- HTML encoding applied consistently via `HttpUtility.HtmlEncode()`
- User-provided content safely rendered in HTML tables

### Recommendations

- Implement file type whitelist for uploads
- Add file size limits
- Sanitize fileName parameter in DownloadBlob
- Add role-based access control for upload functionality
- Implement audit logging for file operations

---

## UI/UX Features

### Dashboard Components

- **System Health:** Database connectivity checks with visual alerts
- **Queue Metrics:** Real-time queue lengths for monitoring backlog
- **Blob Listing:** Recent uploads in incoming container
- **Job History:** Paginated table with clickable links
- **File Upload:** Drag-and-drop capable input with AJAX submission

### Responsive Design

- Bootstrap 5 grid system for responsive layout
- Mobile-friendly navigation and tables
- Accessible form controls with labels

### User Feedback

- Success/error messages for file uploads
- Visual indicators for database errors
- Pagination controls with disabled state

---

## Constants and Configuration

```csharp
const int PageSize = 10;
```

**Purpose:** Defines number of job records displayed per page in status table.

**Customization:** Can be modified to show more/fewer records per page.

---

## Known Issues and Technical Debt

1. **Resource Disposal**
   - `DBConnection`, `Logger`, and `Generic` objects created in `GetStatusHTMLPage()` but only `intLog.Dispose()` called

2. **DownloadBlob Security**
   - No path validation on fileName parameter (potential directory traversal)
   - No access control (any authenticated user can download any file)

3. **FileUpload Validation**
   - No file type, size, or content validation
   - No virus scanning

4. **EventGridHandlerHTTP Response**
   - Returns misleading success message: "File uploaded successfully"
   - Should return "Events processed successfully"

5. **Error Handling**
   - Generic exception catching without specific error types
   - Limited error context in responses

6. **Async Pattern Inconsistency**
   - Methods marked `async` but don't await anything (except file upload and Event Grid handler)

7. **Hardcoded URLs**
   - Favicon and icons hardcoded to investorcom.com domain

8. **Pagination Bounds**
   - No validation that offset is within valid range
   - Could cause database errors or performance issues
