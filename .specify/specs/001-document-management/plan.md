# Implementation Plan: Document Upload and Management

**Branch**: `001-document-management` | **Date**: 2026-03-23 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `.specify/specs/001-document-management/spec.md`

## Summary

The Document Upload and Management feature enables Contoso employees to upload work documents (PDF, Office, images) up to 25MB to a centralized dashboard location, organized by category and project. Core functionality includes file upload with async progress modal, document browsing with search/filtering, role-based access control, document sharing with individual users and project teams, and metadata editing. Training implementation uses local file storage (GUID-based paths), SQLite database, mock virus scanning via `IVirusScanService` interface, hard-delete cascade behavior, and audit logging for compliance. Feature demonstrates offline-first architecture patterns and abstraction layers for cloud migration to Azure.

## Technical Context

**Language/Version**: C# with .NET 10.0 / ASP.NET Core 10.0  
**Primary Dependencies**: 
- Blazor Server (UI framework)
- Entity Framework Core 10.0.0 with SQLite provider
- Bootstrap 5.3 (UI styling)
- Microsoft.Identity.Web (future Azure AD readiness)

**Storage**: SQLite database (local development), Azure SQL ready via connection string swap  
**Testing**: xUnit for unit tests, integration tests with EF Core test database, Blazor UI tests  
**Target Platform**: Blazor Server web application running on .NET 10 (offline-capable, ARM64 compatible)
**Project Type**: Web service / Blazor Server application (training-focused)  
**Performance Goals**: 
- File uploads complete within 30 seconds for 25MB files
- Document list pages load within 2 seconds (500+ documents)
- Search returns results within 2 seconds
- 1000 concurrent users without degradation

**Constraints**:
- Offline-capable (no cloud dependencies during training)
- Mock authentication system (existing cookie-based auth)
- IDOR protection (service-level authorization required)
- No external virus scanning service (mock implementation)
- No version history (audit logging only)
- Hard delete cascade (no soft delete recovery)

**Scale/Scope**: 
- 4 user roles with hierarchical permissions (Employee, TeamLead, ProjectManager, Administrator)
- Document categories: Project Documents, Team Resources, Personal Files, Reports, Presentations, Other
- File types: PDF, Word (.docx), Excel (.xlsx), PowerPoint (.pptx), text files, images (JPEG, PNG)
- Initial 6 user stories (P1-P3), 26 functional requirements, 10 success criteria

## Constitution Check

### Gate 1: Alignment with Core Principles

| Principle | Status | Evidence |
|-----------|--------|----------|
| I. Training-First Development | ✅ PASS | Feature demonstrates file upload patterns, service abstractions, Blazor async patterns, EF Core relationships. Ideal teaching vehicle for DDD layers. |
| II. Mock Authentication | ✅ PASS | Uses existing role-based claims (Employee, TeamLead, ProjectManager, Administrator). No external auth required. |
| III. Offline-First Architecture | ✅ PASS | Local file storage with GUID paths, SQLite database, no cloud dependencies. `IFileStorageService` and `IVirusScanService` interfaces enable cloud migration. |
| IV. Clean Architecture | ✅ PASS | Service layer for DocumentService, FileStorageService, VirusScanService. UI (Blazor pages) separated from business logic (services). |
| V. Security by Design | ✅ PASS | Service-level authorization checks, IDOR protection via project membership verification, role-based policies, security headers. |
| VI. Comprehensive Testing | ✅ PASS | Unit tests on services (authorization, business logic). Integration tests for EF Core relationships. UI tests for critical upload/download/delete flows. |
| VII. Simplicity and Clarity | ✅ PASS | Start with upload + browse (P1), add sharing + editing (P2), dashboard integration (P3). Hard delete simplicity (no recovery). Audit trail over complex versioning. |

### Gate 2: Technology Stack Compliance

| Requirement | Status | Evidence |
|-------------|--------|----------|
| ASP.NET Core 8.0+ | ✅ PASS | Project targets .NET 10.0 (updated from .NET 8.0) |
| Blazor Server | ✅ PASS | UI implemented as Blazor component pages with async modal pattern |
| SQLite with EF Core | ✅ PASS | Document, DocumentShare, DocumentActivity entities in ApplicationDbContext |
| Bootstrap 5.3 | ✅ PASS | Responsive design using existing Bootstrap styling |
| Mock Authentication | ✅ PASS | Uses existing CustomAuthenticationStateProvider with role claims |

### Gate 3: Known Risk Mitigation

| Risk | Severity | Mitigation |
|------|----------|-----------|
| File storage path traversal attacks | High | GUID-based filenames prevent attack; strict whitelist of file extensions; files stored outside wwwroot |
| Unauthorized document access (IDOR) | High | Service-level checks verify user's project membership before granting access to project documents; individual documents checked by UserId |
| Orphaned database records | Medium | Upload sequence: save file → save DB record (not reverse); cascade delete on DocumentShare ensures cleanup |
| Concurrent upload conflicts | Low | GUID filenames ensure uniqueness; SQLite provides ACID guarantees |
| SQLite query performance with 500+ documents | Medium | Indexes on UserId, ProjectId, Status, Category; pagination implemented; search limited to 2-second SLA |
| Async upload modal UX issues | Low | Blazor best practices: Extract IBrowserFile metadata before stream, copy to MemoryStream, clear reference after |

**GATE RESULT**: ✅ **ALL GATES PASS** - Feature approved for Phase 0 research and Phase 1 design.

---

## Phase 0: Research & Clarification

### Research Topics (Resolved - No Unknowns)

All ambiguities resolved during `/speckit.clarify` workflow:

1. ✅ **Document Sharing Scope**: Individual users AND teams (project-based groups)
2. ✅ **Virus Scanning**: IVirusScanService interface (mock for training, real for production)
3. ✅ **Deletion Behavior**: Hard delete with cascade (DocumentShare records automatically deleted)
4. ✅ **File Replacement**: Audit trail only (simple overwrite, no version history)
5. ✅ **Upload UX**: Async background with modal and cancel button

### Output Artifact

**research.md**: [research.md](research.md) - Created during clarification workflow, documents all 5 clarifications and decision rationale.

---

## Phase 1: Design & Data Model

### Design Decisions Summary

#### 1. Data Model

**New Entities:**

```
Document
├── DocumentId (int, PK)
├── UserId (int, FK to User - uploaded by)
├── ProjectId (int?, FK to Project - optional)
├── Title (string, required)
├── Description (string, nullable)
├── Category (string, required) - "Project Documents", "Personal Files", etc.
├── FilePath (string) - GUID-based relative path
├── FileSize (long)
├── FileType (string, 255 chars) - MIME type
├── Status (string) - "Uploading", "Scanning", "Clean", "Infected", "Deleted"
├── UploadDate (DateTime)
├── UpdatedDate (DateTime)
└── IsDeleted (bool, false) - Hard delete flag

DocumentShare
├── ShareId (int, PK)
├── DocumentId (int, FK - cascade delete)
├── SharedWithUserId (int, FK to User)
├── SharedByUserId (int, FK to User - who shared it)
├── ShareDate (DateTime)
└── [No explicit soft delete - cascade means hard delete]

DocumentActivity (for audit logging)
├── ActivityId (int, PK)
├── DocumentId (int, FK, nullable)
├── UserId (int, FK)
├── Action (string) - "Upload", "Download", "Delete", "Replace", "Share"
├── Details (string, nullable)
└── Timestamp (DateTime)
```

**Relationships:**
- User → Document (one-to-many, uploaded by)
- Project → Document (one-to-many, optional)
- User → DocumentShare (one-to-many, SharedWithUserId)
- Document → DocumentShare (cascade delete on Document deletion)
- User → DocumentActivity (audit trail per user)

**Indexes:**
- (UserId) on Document - fast lookup of user's documents
- (ProjectId) on Document - fast lookup of project documents
- (DocumentId) on DocumentShare - fast lookup of document shares
- (UserId, IsDeleted) on Document - fast lookup of active documents by user

**Output Artifact**: [data-model.md](data-model.md)

---

#### 2. Service Layer Contracts

**IFileStorageService** (Infrastructure abstraction for file storage)
```csharp
public interface IFileStorageService
{
    Task<string> UploadAsync(Stream fileStream, string fileName, string contentType);
    Task<Stream> DownloadAsync(string filePath);
    Task DeleteAsync(string filePath);
    Task<string> GetUrlAsync(string filePath, TimeSpan expiration);
}
```
- **Training**: `LocalFileStorageService` using System.IO, files at `AppData/uploads/{userId}/{projectId or "personal"}/{guid}.{ext}`
- **Production**: `AzureBlobStorageService` using Azure.Storage.Blobs SDK

**IVirusScanService** (Infrastructure abstraction for virus scanning)
```csharp
public interface IVirusScanService
{
    Task<(bool IsSafe, string Message)> ScanAsync(Stream fileStream, string fileName);
}
```
- **Training**: `MockVirusScanService` always returns (true, "OK") with logging
- **Production**: Real scanning via ClamAV or cloud service

**IDocumentService** (Business logic)
```csharp
public interface IDocumentService
{
    Task<Document> UploadAsync(int userId, Stream fileStream, string fileName, string contentType, string title, string category, int? projectId);
    Task<Stream> DownloadAsync(int userId, int documentId);
    Task DeleteAsync(int userId, int documentId);
    Task<List<Document>> GetUserDocumentsAsync(int userId);
    Task<List<Document>> SearchAsync(int userId, string query, string? category, DateTime? fromDate, DateTime? toDate);
    Task ShareAsync(int ownerId, int documentId, int[] userIds);
    Task UpdateMetadataAsync(int userId, int documentId, string title, string description, string category);
}
```

**Output Artifact**: [contracts/file-storage-service.md](contracts/file-storage-service.md), [contracts/virus-scan-service.md](contracts/virus-scan-service.md), [contracts/document-service.md](contracts/document-service.md)

---

#### 2.1 Background Processing Architecture

**Async Virus Scanning with Azure Functions**

For production deployments, virus scanning will be handled asynchronously using Azure Functions triggered by Azure Queue Storage. This pattern enables:

- **Non-blocking uploads**: Users see immediate upload success while scanning happens in background
- **Scalable processing**: Functions scale automatically based on queue depth
- **Reliable delivery**: Queue Storage ensures messages aren't lost if functions are down
- **Cost efficiency**: Pay only for actual scanning operations

**Architecture Overview:**

```
Upload Flow:
1. User uploads file → DocumentService.UploadAsync()
2. File saved to Azure Blob Storage
3. Document record created in database (Status: "Scanning")
4. Scan request queued to Azure Queue Storage
5. Upload completes immediately (file available for download)
6. Azure Function processes queue message
7. Function scans file via ClamAV or cloud service
8. Function updates Document.Status to "Clean" or "Infected"
9. If infected: Function deletes file and marks document as quarantined
```

**Azure Queue Storage Message Format:**

```json
{
  "documentId": 123,
  "blobUrl": "https://contosostorage.blob.core.windows.net/documents/123/file.pdf",
  "userId": 456,
  "fileName": "quarterly-report.pdf",
  "contentType": "application/pdf",
  "uploadedAt": "2026-03-23T10:30:00Z",
  "correlationId": "upload-abc-123"
}
```

**Azure Function Implementation:**

```csharp
[FunctionName("ProcessVirusScan")]
public async Task Run(
    [QueueTrigger("virus-scan-queue", Connection = "AzureWebJobsStorage")] 
    ScanRequest request,
    ILogger log)
{
    log.LogInformation($"Processing virus scan for document {request.DocumentId}");
    
    try
    {
        // Download file from blob storage
        var blobClient = new BlobClient(request.BlobUrl);
        using var stream = await blobClient.DownloadStreamingAsync();
        
        // Perform virus scan
        var (isSafe, message) = await _virusScanner.ScanAsync(stream.Value.Content, request.FileName);
        
        // Update database status
        using var context = new ApplicationDbContext(_connectionString);
        var document = await context.Documents.FindAsync(request.DocumentId);
        
        if (isSafe)
        {
            document.Status = "Clean";
            log.LogInformation($"Document {request.DocumentId} passed virus scan");
        }
        else
        {
            document.Status = "Infected";
            // Optionally delete infected file
            await blobClient.DeleteIfExistsAsync();
            log.LogWarning($"Document {request.DocumentId} failed virus scan: {message}");
        }
        
        // Log activity
        context.DocumentActivities.Add(new DocumentActivity
        {
            DocumentId = request.DocumentId,
            UserId = request.UserId,
            Action = "VirusScan",
            Details = isSafe ? "Passed" : $"Failed: {message}",
            Timestamp = DateTime.UtcNow
        });
        
        await context.SaveChangesAsync();
    }
    catch (Exception ex)
    {
        log.LogError(ex, $"Virus scan failed for document {request.DocumentId}");
        // Could implement retry logic or dead letter queue
        throw; // Let Azure Functions handle retry
    }
}
```

**Queue Storage Configuration:**

```json
// host.json for Azure Functions
{
  "version": "2.0",
  "extensions": {
    "queues": {
      "maxPollingInterval": "00:00:10",
      "visibilityTimeout": "00:05:00",
      "batchSize": 16,
      "maxDequeueCount": 3
    }
  }
}
```

**Document Status Enum:**

```csharp
public enum DocumentStatus
{
    Uploading,    // Initial state during upload
    Scanning,     // Queued for virus scan
    Clean,        // Scan passed, available for use
    Infected,     // Scan failed, quarantined
    Deleted       // Hard deleted
}
```

**Database Schema Impact:**

Add `Status` column to Document entity:

```csharp
modelBuilder.Entity<Document>()
    .Property(d => d.Status)
    .HasDefaultValue(DocumentStatus.Scanning)
    .HasConversion<string>(); // Store as string in database
```

**Error Handling & Retry Logic:**

- **Function failures**: Azure Functions automatically retries failed executions (max 3 attempts)
- **Poison messages**: Failed messages move to poison queue after max retries
- **Timeout handling**: Functions have 10-minute timeout by default
- **Circuit breaker**: Consider implementing if external scan service is unreliable

**Monitoring & Alerting:**

- **Application Insights**: Track function execution times, success/failure rates
- **Alerts**: Notify when scan failure rate exceeds threshold (e.g., 5%)
- **Metrics**: Queue depth, processing latency, infection detection rate

**Migration Path from Training:**

Training implementation uses synchronous mock scanning. Production adds:

1. Add `Status` column to Document entity (with default "Scanning")
2. Update DocumentService to queue scan requests instead of calling IVirusScanService directly
3. Deploy Azure Function with Queue Storage trigger
4. Update UI to show document status (scanning/clean/infected)
5. Add retry mechanism for failed scans

**Cost Considerations:**

- **Queue Storage**: ~$0.05/GB/month + operations
- **Azure Functions**: Consumption plan (~$0.20/1M executions)
- **Blob Storage**: Standard tier for temporary scanning
- **Estimated cost**: <$1/month for typical usage (100 uploads/day)

**Security Considerations:**

- **Managed Identity**: Functions use Azure AD managed identity for blob access
- **SAS Tokens**: Short-lived tokens for blob access (not permanent URLs)
- **Network Security**: Functions in VNet with private endpoints to storage
- **Encryption**: All data encrypted at rest and in transit

**Output Artifact**: [contracts/background-processing.md](contracts/background-processing.md)

---

#### 3. Blazor Component Architecture

**Upload Modal Component** (`DocumentUploadModal.razor`)
- InputFile component with async upload
- Real-time progress bar (via SignalR if available, or polling)
- Cancel button to stop upload
- Error display for validation failures

**My Documents Page** (`Documents.razor`)
- Document list with sortable columns (Title, Date, Category, Size)
- Filter dropdown (Category, Project, Date Range)
- Search input (full-text search)
- Action buttons (Download, Share, Edit, Delete)

**Document Details Modal** (`DocumentDetailsModal.razor`)
- Display file metadata
- Edit title/description/category
- Replace file button
- Share button (select users/teams)

**Project Documents Tab** (in `ProjectDetails.razor`)
- Integration tab showing project documents
- Upload button (for project managers)
- List of documents associated with project

**Output Artifact**: [contracts/blazor-components.md](contracts/blazor-components.md)

---

#### 4. Database Initialization & Seeding

Addition to `ApplicationDbContext.OnModelCreating()`:

```csharp
// Document entity configuration
modelBuilder.Entity<Document>()
    .HasOne(d => d.User)
    .WithMany()
    .HasForeignKey(d => d.UserId)
    .OnDelete(DeleteBehavior.Restrict);

modelBuilder.Entity<Document>()
    .HasOne(d => d.Project)
    .WithMany()
    .HasForeignKey(d => d.ProjectId)
    .OnDelete(DeleteBehavior.Restrict);

modelBuilder.Entity<Document>()
    .HasIndex(d => d.UserId);

modelBuilder.Entity<Document>()
    .HasIndex(d => new { d.UserId, d.IsDeleted });

// Document status configuration for background processing
modelBuilder.Entity<Document>()
    .Property(d => d.Status)
    .HasDefaultValue("Scanning")
    .HasConversion<string>();

// DocumentShare configuration with cascade delete
modelBuilder.Entity<DocumentShare>()
    .HasOne(s => s.Document)
    .WithMany()
    .HasForeignKey(s => s.DocumentId)
    .OnDelete(DeleteBehavior.Cascade);  // CASCADE DELETE

modelBuilder.Entity<DocumentShare>()
    .HasIndex(s => s.DocumentId);

modelBuilder.Entity<DocumentShare>()
    .HasIndex(s => s.SharedWithUserId);
```

**Seed Data**: Sample documents for demo (training scenarios)

---

#### 5. Security Model

**Authorization Enforcement:**

```
Service Method          | Who Can Access?
-------------------------------------------
UploadAsync            | Authenticated users (any role)
DownloadAsync          | Document owner OR project member (if project doc) OR share recipient
DeleteAsync            | Document owner OR project manager (if project doc) OR admin
ShareAsync             | Document owner only
UpdateMetadataAsync    | Document owner only
SearchAsync            | Authenticated users (filters return only accessible documents)
GetUserDocumentsAsync   | Own documents only
```

**API Endpoint Protection**: All document endpoints secured with `[Authorize]` attribute + service-level checks.

**IDOR Prevention**: Service methods validate user's project membership before granting access to project documents.

---

### Quickstart Guide

**For Developers/Trainers**: [quickstart.md](quickstart.md) - Contains:
- Feature overview and user workflows
- Database schema visualization
- Service layer architecture diagram
- Setup instructions (entity configuration, migrations)
- Testing scenarios and test data
- Common implementation patterns (async uploads, IDOR protection)
- Troubleshooting guide

---

## Phase 2: Implementation Tasks

**Note**: Task breakdown generated by `/speckit.tasks` command (not created here).

**Expected Task Categories** (from specification):
1. **Data Layer**: Document/DocumentShare entity definitions, relationships, indexes
2. **Infrastructure Services**: LocalFileStorageService, MockVirusScanService, DocumentService
3. **Business Logic**: Authorization checks, upload validation, cascade delete logic
4. **UI Components**: Upload modal, document list view, search/filter UI
5. **Integration**: Project document tab, dashboard widget, notification system
6. **Testing**: Unit tests (services), integration tests (EF Core), UI tests (upload flow)
7. **Documentation**: API docs, architecture guide, troubleshooting

---

## Deliverables

### Phase 0 Artifacts
- ✅ [research.md](research.md) - Clarification resolutions

### Phase 1 Artifacts (This Document's Output)
- ✅ [data-model.md](data-model.md) - Entity definitions and relationships
- ✅ [contracts/](contracts/) - Service interfaces and Blazor component contracts
- ✅ [quickstart.md](quickstart.md) - Developer guide

### Phase 2 Artifacts (Created by `speckit.tasks`)
- _[tasks.md](tasks.md)_ - Expanded task list with acceptance criteria

---

## Success Validation

| Criterion | Measurement |
|-----------|-------------|
| Constitution alignment | All 7 principles satisfied (security, simplicity, offline-first, architecture) ✅ |
| Scope management | 6 user stories across P1-P3 (P1: upload + search; P2: sharing; P3: dashboard) ✅ |
| Technical readiness | Service abstractions ready for training → production migration ✅ |
| Design clarity | Data model, service contracts, UI components all documented ✅ |
| Risk mitigation | Security risks (IDOR, path traversal), performance risks (indexing, pagination) addressed ✅ |

---

## Notes for Implementation Team

1. **File Upload Sequence** (critical for data consistency):
   - Generate unique file path (GUID)
   - Save physical file to disk via IFileStorageService
   - Create DatabaseContext transaction
   - Insert Document record with FilePath
   - Log DocumentActivity("Upload")
   - Commit transaction
   - Return success
   
   ⚠️ Do NOT insert DB record first, then save file (can cause orphaned records).

2. **Blazor Upload Component Pattern**:
   ```csharp
   // Extract metadata BEFORE opening stream
   var fileName = SelectedFile.Name;
   var fileSize = SelectedFile.Size;
   var contentType = SelectedFile.ContentType;
   
   using var memoryStream = new MemoryStream();
   using (var fileStream = SelectedFile.OpenReadStream(maxFileSize: 26214400))  // 25MB
   {
       await fileStream.CopyToAsync(memoryStream);
   }
   memoryStream.Position = 0;
   SelectedFile = null;  // Clear reference to prevent reuse
   StateHasChanged();
   
   // Progress modal displays during upload
   ```

3. **SQLite Specific Considerations**:
   - PRAGMA journal_mode = 'wal' for better concurrency (auto-configured)
   - Text field for Category (even though fixed options - easier to query)
   - Cascade deletes must be explicitly enabled: `OnDelete(DeleteBehavior.Cascade)`
   - Pagination recommended for 500+ documents (LIMIT/OFFSET)

4. **Testing Strategy**:
   - Unit: DocumentService authorization, upload validation, cascade delete
   - Integration: EF Core Document relationships, transaction commit/rollback
   - UI: Upload modal cancel, progress update, error display
   - Security: Attempt unauthorized document access, verify IDOR protection

---

## Next Steps

1. **`/speckit.tasks`**: Break Phase 1 design into concrete implementation tasks with acceptance criteria
2. **Create feature branch**: `git checkout -b 001-document-management`
3. **Implement Phase 2**: Follow task list from tasks.md
4. **Testing**: Unit + integration + UI tests per testing strategy
5. **Review**: Constitution compliance checklist
6. **Deploy**: Merge to main branch after approval

---

**Plan Status**: ✅ READY FOR IMPLEMENTATION
**Last Updated**: 2026-03-23
**Author**: Spec-Driven Development Workflow