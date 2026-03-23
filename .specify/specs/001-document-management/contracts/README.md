# Service Contracts: Document Upload and Management

**Feature**: 001-document-management  
**Date**: 2026-03-23  
**Status**: Specification Reference

This directory documents the service layer interfaces that define the contracts between the UI layer (Blazor components), business logic layer (services), and infrastructure layer (file storage, virus scanning).

---

## Interface Specifications

### 1. IDocumentService

**Purpose**: Core business logic for document lifecycle management

**Location**: `ContosoDashboard/Services/IDocumentService.cs`

**Methods**:

```csharp
public interface IDocumentService
{
    /// <summary>
    /// Upload a new document
    /// </summary>
    /// <param name="userId">Current user ID (from claims)</param>
    /// <param name="fileStream">File content stream</param>
    /// <param name="fileName">Original filename</param>
    /// <param name="contentType">MIME type</param>
    /// <param name="title">Document title (from form)</param>
    /// <param name="category">Document category</param>
    /// <param name="projectId">Optional project association</param>
    /// <returns>Created Document entity</returns>
    /// <throws>ArgumentException on validation failure</throws>
    /// <throws>SecurityException if virus scan fails</throws>
    /// <throws>UnauthorizedAccessException if user cannot upload to project</throws>
    Task<Document> UploadAsync(
        int userId,
        Stream fileStream,
        string fileName,
        string contentType,
        string title,
        string category,
        int? projectId = null);
    
    /// <summary>
    /// Download a document file that user has access to
    /// </summary>
    /// <param name="userId">Current user ID (from claims)</param>
    /// <param name="documentId">Document to download</param>
    /// <returns>File stream ready for download</returns>
    /// <throws>KeyNotFoundException if document not found</throws>
    /// <throws>UnauthorizedAccessException if user lacks access (IDOR protection)</throws>
    Task<Stream> DownloadAsync(int userId, int documentId);
    
    /// <summary>
    /// Delete a document (hard delete with cascade)</summary>
    /// <param name="userId">Current user ID</param>
    /// <param name="documentId">Document to delete</param>
    /// <throws>KeyNotFoundException if not found</throws>
    /// <throws>UnauthorizedAccessException if user cannot delete</throws>
    Task DeleteAsync(int userId, int documentId);
    
    /// <summary>
    /// Get all documents accessible to user
    /// </summary>
    /// <param name="userId">Current user</param>
    /// <returns>List of accessible documents</returns>
    Task<List<Document>> GetUserDocumentsAsync(int userId);
    
    /// <summary>
    /// Get documents associated with a project
    /// </summary>
    /// <param name="userId">Current user (for access check)</param>
    /// <param name="projectId">Project ID</param>
    /// <returns>Project documents if user is member</returns>
    /// <throws>UnauthorizedAccessException if not project member</throws>
    Task<List<Document>> GetProjectDocumentsAsync(int userId, int projectId);
    
    /// <summary>
    /// Search documents by title, description, tags
    /// </summary>
    /// <param name="userId">Current user (scope to accessible docs)</param>
    /// <param name="query">Search term</param>
    /// <param name="category">Optional filter</param>
    /// <param name="fromDate">Optional date range start</param>
    /// <param name="toDate">Optional date range end</param>
    /// <returns>Matching documents (limited to 2-second SLA)</returns>
    Task<List<Document>> SearchAsync(
        int userId,
        string query,
        string? category = null,
        DateTime? fromDate = null,
        DateTime? toDate = null);
    
    /// <summary>
    /// Share document with users and/or project teams
    /// </summary>
    /// <param name="userId">Document owner</param>
    /// <param name="documentId">Document to share</param>
    /// <param name="shareWithUserIds">Individual user IDs to share with</param>
    /// <param name="shareWithProjectIds">Project IDs to share with (all members)</param>
    /// <returns>Newly created DocumentShare records</returns>
    /// <throws>UnauthorizedAccessException if user doesn't own document</throws>
    Task<List<DocumentShare>> ShareAsync(
        int userId,
        int documentId,
        int[]? shareWithUserIds = null,
        int[]? shareWithProjectIds = null);
    
    /// <summary>
    /// Update document metadata
    /// </summary>
    /// <param name="userId">Current user</param>
    /// <param name="documentId">Document to update</param>
    /// <param name="title">New title</param>
    /// <param name="description">New description</param>
    /// <param name="category">New category</param>
    /// <returns>Updated document</returns>
    /// <throws>UnauthorizedAccessException if user doesn't own document</throws>
    Task<Document> UpdateMetadataAsync(
        int userId,
        int documentId,
        string title,
        string description,
        string category);
    
    /// <summary>
    /// Replace document file with new version
    /// </summary>
    /// <param name="userId">Current user</param>
    /// <param name="documentId">Document to replace</param>
    /// <param name="fileStream">New file content</param>
    /// <param name="fileName">New file name</param>
    /// <param name="contentType">New MIME type</param>
    /// <returns>Updated document</returns>
    /// <throws>UnauthorizedAccessException if user doesn't own document</throws>
    Task<Document> ReplaceFileAsync(
        int userId,
        int documentId,
        Stream fileStream,
        string fileName,
        string contentType);
}
```

**Implementation Notes**:
- All methods check authorization at service layer (not just controller)
- IDOR protection by verifying user access before operations
- Async/await throughout for non-blocking operations
- Logging at key points (upload, delete, share, access violations)

---

### 2. IFileStorageService

**Purpose**: Abstract file storage to enable local (training) → cloud (production) migration

**Location**: `ContosoDashboard/Services/IFileStorageService.cs`

**Methods**:

```csharp
public interface IFileStorageService
{
    /// <summary>
    /// Upload a file to storage
    /// </summary>
    /// <param name="fileStream">File content</param>
    /// <param name="fileName">Original filename</param>
    /// <param name="contentType">MIME type</param>
    /// <returns>Storage path (relative for local, blob URI for Azure)</returns>
    Task<string> UploadAsync(Stream fileStream, string fileName, string contentType);
    
    /// <summary>
    /// Download a file from storage
    /// </summary>
    /// <param name="filePath">Storage path returned from Upload</param>
    /// <returns>File stream</returns>
    Task<Stream> DownloadAsync(string filePath);
    
    /// <summary>
    /// Delete a file from storage
    /// </summary>
    /// <param name="filePath">Storage path</param>
    Task DeleteAsync(string filePath);
    
    /// <summary>
    /// Get a public URL for the file (for direct download links)
    /// </summary>
    /// <param name="filePath">Storage path</param>
    /// <param name="expiration">URL validity duration</param>
    /// <returns>Public URL</returns>
    Task<string> GetUrlAsync(string filePath, TimeSpan expiration);
}
```

**Training Implementation** (`LocalFileStorageService`):
- Stores files in `AppData/uploads/` directory outside wwwroot
- File paths: `{userId}/{projectId or "personal"}/{guid}.{extension}`
- Uses System.IO for file operations

**Production Implementation** (Future `AzureBlobStorageService`):
- Stores files in Azure Blob Storage container
- Blob names: same path pattern as local
- Uses Azure.Storage.Blobs SDK
- Supports SAS token generation for expiring URLs

---

### 3. IVirusScanService

**Purpose**: Abstract virus scanning to support mock (training) and real implementations

**Location**: `ContosoDashboard/Services/IVirusScanService.cs`

**Methods**:

```csharp
public interface IVirusScanService
{
    /// <summary>
    /// Scan a file for viruses/malware
    /// </summary>
    /// <param name="fileStream">File content</param>
    /// <param name="fileName">Filename for logging</param>
    /// <returns>Tuple: (IsSafe=true if passed, Message describing result)</returns>
    /// <remarks>
    /// Training implementation: Always returns (true, "OK")
    /// Production implementation: Real ClamAV or cloud scanning
    /// </remarks>
    Task<(bool IsSafe, string Message)> ScanAsync(Stream fileStream, string fileName);
}
```

**Training Implementation** (`MockVirusScanService`):
```csharp
public class MockVirusScanService : IVirusScanService
{
    public async Task<(bool IsSafe, string Message)> ScanAsync(Stream fileStream, string fileName)
    {
        // Simulate scan delay for realism
        await Task.Delay(100);
        
        _logger.LogInformation($"Mock virus scan: {fileName} - PASSED");
        return (true, "Scan passed (mock implementation - training only)");
    }
}
```

**Production Implementation** (Future):
- Integration with ClamAV daemon or cloud scanning service
- Real antivirus detection
- Audit logging of scan results

---

### 4. ICurrentUserService (Existing)

**Purpose**: Extract current user from ASP.NET Core context

**Usage in DocumentService**:

```csharp
public class DocumentService : IDocumentService
{
    private readonly ICurrentUserService _currentUserService;
    
    public async Task<Document> UploadAsync(...)
    {
        var userId = _currentUserService.GetCurrentUserId();
        // ... rest of upload logic
    }
}
```

**Assumption**: Current application provides ICurrentUserService with these methods:
- `int GetCurrentUserId()` - Returns user ID from claims
- `User? GetCurrentUser()` - Returns User entity
- `string[] GetRoles()` - Returns user roles

---

### 5. Background Processing Contracts

**Purpose**: Async virus scanning using Azure Functions with Queue Storage triggers

#### 5.1 Azure Queue Message Format

**Location**: `ContosoDashboard/Models/ScanRequest.cs`

```csharp
public class ScanRequest
{
    public int DocumentId { get; set; }
    public string BlobUrl { get; set; } = string.Empty;
    public int UserId { get; set; }
    public string FileName { get; set; } = string.Empty;
    public string ContentType { get; set; } = string.Empty;
    public DateTime UploadedAt { get; set; }
    public string CorrelationId { get; set; } = string.Empty;
}
```

#### 5.2 Azure Function Contract

**Location**: `ContosoDashboard.AzureFunctions/ProcessVirusScan.cs`

```csharp
[FunctionName("ProcessVirusScan")]
public async Task Run(
    [QueueTrigger("virus-scan-queue", Connection = "AzureWebJobsStorage")] 
    ScanRequest request,
    ILogger log)
{
    // Implementation details in plan.md
}
```

#### 5.3 Document Status Enum

**Location**: `ContosoDashboard/Models/DocumentStatus.cs`

```csharp
public enum DocumentStatus
{
    Uploading,    // Initial state during upload
    Scanning,     // Queued for virus scan (default)
    Clean,        // Scan passed, available for use
    Infected,     // Scan failed, quarantined
    Deleted       // Hard deleted
}
```

---

## Dependency Injection Configuration

**In Program.cs**:

```csharp
// Register document management services
builder.Services.AddScoped<IDocumentService, DocumentService>();
builder.Services.AddScoped<IFileStorageService, LocalFileStorageService>();  // Training
builder.Services.AddScoped<IVirusScanService, MockVirusScanService>();  // Training

// For production, swap implementations:
// builder.Services.AddScoped<IFileStorageService, AzureBlobStorageService>();
// builder.Services.AddScoped<IVirusScanService, ClamAvVirusScanService>();
```

---

## Contract Testing Strategy

### Unit Tests (Services)

```csharp
[TestClass]
public class DocumentServiceTests
{
    private Mock<IFileStorageService> _mockFileStorage;
    private Mock<IVirusScanService> _mockVirusScan;
    private Mock<ApplicationDbContext> _mockDbContext;
    private IDocumentService _sut;
    
    [TestInitialize]
    public void Setup()
    {
        _mockFileStorage = new Mock<IFileStorageService>();
        _mockVirusScan = new Mock<IVirusScanService>();
        _sut = new DocumentService(_mockFileStorage.Object, _mockVirusScan.Object, ...);
    }
    
    [TestMethod]
    public async Task UploadAsync_WithValidFile_CreatesDocument()
    {
        // Arrange
        var fileStream = new MemoryStream(Encoding.UTF8.GetBytes("test content"));
        
        // Act
        var result = await _sut.UploadAsync(
            userId: 1,
            fileStream: fileStream,
            fileName: "test.pdf",
            contentType: "application/pdf",
            title: "Test Document",
            category: "Reports");
        
        // Assert
        Assert.IsNotNull(result);
        Assert.AreEqual("Test Document", result.Title);
        _mockFileStorage.Verify(x => x.UploadAsync(It.IsAny<Stream>(), "test.pdf", "application/pdf"), Times.Once());
    }
    
    [TestMethod]
    public async Task DownloadAsync_WithUnauthorizedUser_ThrowsException()
    {
        // Arrange
        var documentId = 1;
        var unauthorizedUserId = 999;
        
        // Act & Assert
        await Assert.ThrowsExceptionAsync<UnauthorizedAccessException>(
            () => _sut.DownloadAsync(unauthorizedUserId, documentId));
    }
    
    [TestMethod]
    public async Task DeleteAsync_RemovesFileAndDatabase()
    {
        // Arrange
        var documentId = 1;
        
        // Act
        await _sut.DeleteAsync(userId: 1, documentId: documentId);
        
        // Assert
        _mockFileStorage.Verify(x => x.DeleteAsync(It.IsAny<string>()), Times.Once());
        // Verify DB remove called
    }
}
```

### Integration Tests

```csharp
[TestClass]
public class DocumentServiceIntegrationTests
{
    private DocumentService _documentService;
    private ApplicationDbContext _dbContext;
    private TestDatabaseFixture _fixture;
    
    [TestInitialize]
    public void Setup()
    {
        _fixture = new TestDatabaseFixture();
        _dbContext = _fixture.CreateDbContext();
        _documentService = new DocumentService(
            new LocalFileStorageService(...),
            new MockVirusScanService(),
            _dbContext);
    }
    
    [TestMethod]
    public async Task Upload_And_Download_Roundtrip()
    {
        // Create test file
        var content = Encoding.UTF8.GetBytes("Test PDF content");
        using var fileStream = new MemoryStream(content);
        
        // Upload
        var doc = await _documentService.UploadAsync(
            userId: 1,
            fileStream: fileStream,
            fileName: "test.pdf",
            contentType: "application/pdf",
            title: "Test",
            category: "Reports");
        
        // Download
        using var downloaded = await _documentService.DownloadAsync(userId: 1, documentId: doc.DocumentId);
        
        // Verify content matches
        var downloadedContent = new StreamReader(downloaded).ReadToEnd();
        Assert.AreEqual("Test PDF content", downloadedContent);
    }
}
```

---

## Versioning & Compatibility

| Version | Date | Changes | Backward Compatible |
|---------|------|---------|-------------------|
| 1.0 | 2026-03-23 | Initial contracts | N/A |
| 1.1 (planned) | TBD | Sharing with teams | Yes - new parameter |
| 2.0 (planned) | TBD | Version history | No - new method |

---

## References

- [Data Model](../data-model.md) - Database schema and relationships
- [Quickstart](../quickstart.md) - Implementation patterns and examples
- [Plan](../plan.md) - Full architecture and design decisions

---

**Status**: ✅ Contracts Defined
**Last Updated**: 2026-03-23