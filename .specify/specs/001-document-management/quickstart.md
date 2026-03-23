# Quickstart Guide: Document Upload and Management Feature

**Feature**: 001-document-management  
**Date**: 2026-03-23  
**Target Audience**: Developers, Technical Leads, QA Engineers, Trainers

---

## Overview

The Document Upload and Management feature enables Contoso employees to upload, organize, search, and share work documents (PDF, Office, images) within the ContosoDashboard application. This guide provides implementation context, architectural patterns, and hands-on walkthroughs.

### Core Workflows

1. **Upload Document** (P1) - User uploads file with metadata, progress shown in modal
2. **Browse & Search Documents** (P1) - User finds documents by title, category, date range
3. **Manage Project Documents** (P2) - Project managers control project-wide documents
4. **Share with Team** (P2) - Document owner shares with individuals and project teams
5. **Edit & Delete** (P3) - Update metadata, replace files, delete from system
6. **Dashboard Integration** (P3) - Recent documents widget, summary statistics

---

## Architecture Overview

### Layer Stack

```
┌─────────────────────────────────────────────────┐
│  Blazor Server Pages (UI Layer)                │
│  - DocumentUploadModal.razor                   │
│  - Documents.razor (My Documents view)         │
│  - ProjectDetails.razor (project tab)          │
└──────────────────┬──────────────────────────────┘
                   │ Dependency Injection
                   ▼
┌─────────────────────────────────────────────────┐
│  Service Layer (Business Logic)                 │
│  ├─ IDocumentService                           │
│  ├─ IFileStorageService                        │
│  ├─ IVirusScanService                          │
│  └─ INotificationService (existing)            │
└──────────────────┬──────────────────────────────┘
                   │ DbContext
                   ▼
┌─────────────────────────────────────────────────┐
│  EF Core Data Access                           │
│  └─ ApplicationDbContext                       │
│     ├─ Document DbSet                          │
│     ├─ DocumentShare DbSet                     │
│     └─ DocumentActivity DbSet                  │
└──────────────────┬──────────────────────────────┘
                   │
         ┌─────────┴──────────┐
         ▼                    ▼
    ┌─────────────┐  ┌──────────────────┐
    │  SQLite DB  │  │ File Storage     │
    │ (local)     │  │ (IFileStorage)   │
    └─────────────┘  └──────────────────┘
```

### Key Design Patterns

#### 1. Infrastructure Abstraction (Cloud Migration Ready)

```csharp
// IFileStorageService - enables switching between local and cloud
public interface IFileStorageService
{
    Task<string> UploadAsync(Stream fileStream, string fileName, string contentType);
    Task<Stream> DownloadAsync(string filePath);
    Task DeleteAsync(string filePath);
    Task<string> GetUrlAsync(string filePath, TimeSpan expiration);
}

// Training implementation
public class LocalFileStorageService : IFileStorageService
{
    private readonly IHostEnvironment _hostEnvironment;
    
    public async Task<string> UploadAsync(Stream fileStream, string fileName, string contentType)
    {
        // Generate GUID-based path to prevent conflicts
        var uploadDir = Path.Combine(_hostEnvironment.ContentRootPath, "AppData", "uploads");
        var filePath = Path.Combine(uploadDir, Guid.NewGuid().ToString() + Path.GetExtension(fileName));
        
        using (var file = System.IO.File.Create(filePath))
        {
            await fileStream.CopyToAsync(file);
        }
        
        // Return relative path for storage in database
        var relativePath = Path.GetRelativePath(_hostEnvironment.ContentRootPath, filePath);
        return relativePath;
    }
    
    // ... other methods
}

// Production implementation (future)
public class AzureBlobStorageService : IFileStorageService { ... }
```

#### 2. Async Upload with Progress Modal

```csharp
// DocumentUploadModal.razor
@component
<div class="modal @(IsUploading ? "d-block" : "d-none")">
    <div class="modal-dialog">
        <div class="modal-content">
            <div class="modal-header">
                <h5>Upload Document</h5>
                <button type="button" class="btn-close" @onclick="Cancel" disabled="@IsUploading"></button>
            </div>
            
            <div class="modal-body">
                @if (!IsUploading)
                {
                    <EditForm Model="@uploadModel" OnValidSubmit="@HandleUpload">
                        <DataAnnotationsValidator />
                        <ValidationSummary />
                        
                        <div class="mb-3">
                            <label>Document Title *</label>
                            <InputText class="form-control" @bind-Value="uploadModel.Title" />
                        </div>
                        
                        <div class="mb-3">
                            <label>Category *</label>
                            <InputSelect class="form-select" @bind-Value="uploadModel.Category">
                                <option>-- Select --</option>
                                <option>Project Documents</option>
                                <option>Personal Files</option>
                                <option>Team Resources</option>
                                <option>Reports</option>
                                <option>Presentations</option>
                                <option>Other</option>
                            </InputSelect>
                        </div>
                        
                        <div class="mb-3">
                            <label>File *</label>
                            <InputFile OnChange="@OnFileSelected" accept=".pdf,.docx,.xlsx,.pptx,.txt,.jpg,.jpeg,.png"
                                       @ref="fileInput" />
                        </div>
                        
                        <button type="submit" class="btn btn-primary">Upload</button>
                    </EditForm>
                }
                else
                {
                    <div class="text-center">
                        <p><strong>Uploading: @SelectedFile?.Name</strong></p>
                        <div class="progress">
                            <div class="progress-bar" style="width: @($"{UploadProgress}%")">
                                @UploadProgress%
                            </div>
                        </div>
                        <p class="mt-2">@FormatBytes(UploadedBytes) / @FormatBytes(TotalBytes)</p>
                        <button class="btn btn-danger btn-sm" @onclick="CancelUpload">Cancel Upload</button>
                    </div>
                }
            </div>
        </div>
    </div>
</div>

@code {
    private IBrowserFile? SelectedFile;
    private InputFile? fileInput;
    private bool IsUploading = false;
    private int UploadProgress = 0;
    private long UploadedBytes = 0;
    private long TotalBytes = 0;
    private UploadModel uploadModel = new();
    private CancellationTokenSource? cancellationTokenSource;
    
    private async Task HandleUpload()
    {
        IsUploading = true;
        cancellationTokenSource = new CancellationTokenSource();
        UploadProgress = 0;
        
        try
        {
            // Extract metadata BEFORE opening stream (critical for Blazor)
            var fileName = SelectedFile!.Name;
            var fileSize = SelectedFile.Size;
            var contentType = SelectedFile.ContentType;
            
            using var memoryStream = new MemoryStream();
            using (var fileStream = SelectedFile.OpenReadStream(maxFileSize: 26214400))  // 25MB
            {
                // Report progress
                var buffer = new byte[81920];  // 80KB chunks
                int bytesRead;
                
                while ((bytesRead = await fileStream.ReadAsync(buffer, 0, buffer.Length, cancellationTokenSource.Token)) > 0)
                {
                    await memoryStream.WriteAsync(buffer, 0, bytesRead);
                    UploadedBytes += bytesRead;
                    TotalBytes = fileSize;
                    UploadProgress = (int)(UploadedBytes * 100 / TotalBytes);
                    StateHasChanged();
                }
            }
            
            memoryStream.Position = 0;
            SelectedFile = null;  // Clear reference to prevent reuse
            
            // Call DocumentService to upload
            var document = await DocumentService.UploadAsync(memoryStream, fileName, contentType, 
                uploadModel.Title, uploadModel.Category);
            
            IsUploading = false;
            await OnUploadComplete.InvokeAsync(document);
        }
        catch (OperationCanceledException)
        {
            // User cancelled upload
            IsUploading = false;
            await JSRuntime.InvokeVoidAsync("alert", "Upload cancelled");
        }
        catch (Exception ex)
        {
            IsUploading = false;
            await JSRuntime.InvokeVoidAsync("alert", $"Upload failed: {ex.Message}");
        }
    }
    
    private async Task CancelUpload()
    {
        cancellationTokenSource?.Cancel();
    }
    
    private string FormatBytes(long bytes)
    {
        if (bytes < 1024) return $"{bytes} B";
        if (bytes < 1024 * 1024) return $"{bytes / 1024.0:F1} KB";
        return $"{bytes / (1024.0 * 1024):F1} MB";
    }
}
```

---

## Implementation Patterns

### Pattern 1: Secure File Upload (CRITICAL)

**Sequence MUST be**: Save file → Save DB record ONLY

```csharp
public class DocumentService : IDocumentService
{
    private readonly IFileStorageService _fileStorageService;
    private readonly IVirusScanService _virusScanService;
    private readonly ApplicationDbContext _dbContext;
    private readonly ILogger<DocumentService> _logger;
    private readonly ICurrentUserService _currentUserService;
    
    public async Task<Document> UploadAsync(
        Stream fileStream,
        string fileName,
        string contentType,
        string title,
        string category,
        int? projectId = null)
    {
        var userId = _currentUserService.GetCurrentUserId();
        
        // STEP 1: Validate
        ValidateUpload(fileName, fileStream.Length, contentType, projectId);
        
        // STEP 2: Virus scan
        var (isSafe, scanMessage) = await _virusScanService.ScanAsync(fileStream, fileName);
        if (!isSafe)
            throw new SecurityException($"File rejected: {scanMessage}");
        
        fileStream.Position = 0;  // Reset stream position after scan
        
        // STEP 3: Generate UNIQUE file path BEFORE saving anything
        var fileExtension = Path.GetExtension(fileName);
        var uniqueFileName = $"{Guid.NewGuid()}{fileExtension}";
        var projectDir = projectId.HasValue ? projectId.ToString() : "personal";
        var filePath = $"{userId}/{projectDir}/{uniqueFileName}";
        
        // STEP 4: Save physical file FIRST
        var storagePath = await _fileStorageService.UploadAsync(
            fileStream, fileName, contentType);
        
        try
        {
            // STEP 5: Create database record in transaction
            using (var transaction = await _dbContext.Database.BeginTransactionAsync())
            {
                try
                {
                    var document = new Document
                    {
                        UserId = userId,
                        ProjectId = projectId,
                        Title = title,
                        Description = string.Empty,
                        Category = category,
                        FilePath = storagePath,
                        FileSize = fileStream.Length,
                        FileType = contentType,
                        UploadDate = DateTime.UtcNow,
                        UpdatedDate = DateTime.UtcNow
                    };
                    
                    _dbContext.Documents.Add(document);
                    
                    // Log activity
                    var activity = new DocumentActivity
                    {
                        UserId = userId,
                        DocumentId = document.DocumentId,
                        Action = "Upload",
                        Details = $"File: {fileName}, Size: {fileStream.Length} bytes",
                        Timestamp = DateTime.UtcNow
                    };
                    _dbContext.DocumentActivities.Add(activity);
                    
                    await _dbContext.SaveChangesAsync();
                    await transaction.CommitAsync();
                    
                    _logger.LogInformation($"Document uploaded: {document.DocumentId} by user {userId}");
                    return document;
                }
                catch (Exception dbEx)
                {
                    await transaction.RollbackAsync();
                    
                    // CRITICAL: Clean up physical file if DB insert failed
                    await _fileStorageService.DeleteAsync(storagePath);
                    
                    _logger.LogError($"Database insert failed, physical file cleaned up: {dbEx.Message}");
                    throw;
                }
            }
        }
        catch (Exception ex)
        {
            _logger.LogError($"Upload failed: {ex.Message}");
            throw;
        }
    }
    
    private void ValidateUpload(string fileName, long fileSize, string contentType, int? projectId)
    {
        // Size validation
        if (fileSize == 0 || fileSize > 26214400)  // 25MB
            throw new ArgumentException($"File size must be between 1 byte and 25MB (got {fileSize} bytes)");
        
        // Extension validation (whitelist only)
        var allowedExtensions = new[] { ".pdf", ".docx", ".xlsx", ".pptx", ".txt", ".jpg", ".jpeg", ".png" };
        var extension = Path.GetExtension(fileName).ToLowerInvariant();
        if (!allowedExtensions.Contains(extension))
            throw new ArgumentException($"File type not allowed: {extension}");
        
        // MIME type validation
        var allowedMimeTypes = new[] {
            "application/pdf",
            "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
            "application/vnd.openxmlformats-officedocument.presentationml.presentation",
            "text/plain",
            "image/jpeg",
            "image/png"
        };
        if (!allowedMimeTypes.Contains(contentType))
            throw new ArgumentException($"MIME type not allowed: {contentType}");
    }
}
```

### Pattern 2: Authorization, Not Just Authentication

```csharp
public async Task<Stream> DownloadAsync(int userId, int documentId)
{
    // CRITICAL: Verify user has access to document
    var document = await _dbContext.Documents
        .Include(d => d.Shares)
        .FirstOrDefaultAsync(d => d.DocumentId == documentId);
    
    if (document == null)
        throw new KeyNotFoundException($"Document {documentId} not found");
    
    // Check authorization
    bool hasAccess = false;
    
    // Owner can download
    if (document.UserId == userId)
        hasAccess = true;
    
    // Project members can download project documents
    if (document.ProjectId.HasValue)
    {
        var isProjectMember = await _dbContext.ProjectMembers
            .AnyAsync(pm => pm.ProjectId == document.ProjectId && pm.UserId == userId);
        if (isProjectMember)
            hasAccess = true;
    }
    
    // Shared recipients can download
    if (document.Shares.Any(s => s.SharedWithUserId == userId))
        hasAccess = true;
    
    if (!hasAccess)
        throw new UnauthorizedAccessException($"User {userId} cannot access document {documentId}");
    
    // Log download
    var activity = new DocumentActivity
    {
        DocumentId = documentId,
        UserId = userId,
        Action = "Download",
        Timestamp = DateTime.UtcNow
    };
    _dbContext.DocumentActivities.Add(activity);
    await _dbContext.SaveChangesAsync();
    
    // Return file stream
    return await _fileStorageService.DownloadAsync(document.FilePath);
}
```

### Pattern 3: Cascade Delete (Hard Delete)

```csharp
public async Task DeleteAsync(int userId, int documentId)
{
    // Get document
    var document = await _dbContext.Documents
        .Include(d => d.Shares)
        .FirstOrDefaultAsync(d => d.DocumentId == documentId);
    
    if (document == null)
        throw new KeyNotFoundException($"Document {documentId} not found");
    
    // Verify authorization
    bool canDelete = (document.UserId == userId);  // Owner
    
    if (document.ProjectId.HasValue)
    {
        // Project manager can delete
        var isProjectManager = await _dbContext.ProjectMembers
            .Where(pm => pm.ProjectId == document.ProjectId && pm.UserId == userId)
            .AnyAsync();
        if (isProjectManager)
            canDelete = true;
    }
    
    if (!canDelete)
        throw new UnauthorizedAccessException($"User {userId} cannot delete document {documentId}");
    
    // Delete physical file
    await _fileStorageService.DeleteAsync(document.FilePath);
    
    // Delete from database (CASCADE will delete DocumentShare records automatically)
    _dbContext.Documents.Remove(document);
    
    // Log deletion (happens before actual deletion)
    var activity = new DocumentActivity
    {
        DocumentId = documentId,  // May be null after cascade if we log after delete
        UserId = userId,
        Action = "Delete",
        Details = $"Deleted: {document.Title}",
        Timestamp = DateTime.UtcNow
    };
    _dbContext.DocumentActivities.Add(activity);
    
    await _dbContext.SaveChangesAsync();
    
    _logger.LogInformation($"Document deleted: {documentId} ({document.Title}) by user {userId}");
}
```

---

## Common Testing Scenarios

### Test 1: Upload Document & Verify in My Documents

**Steps**:
1. Login as "Ni Kang" (Employee role)
2. Navigate to Documents page
3. Click "Upload" button
4. Select `sample.pdf` file (< 25MB)
5. Enter title: "Quarterly Report Q1"
6. Select category: "Reports"
7. Click "Upload"
8. Verify progress modal appears
9. Verify document appears in "My Documents" list

**Expected Result**: ✅ Document visible in My Documents with correct metadata

### Test 2: Share Document with Project Team

**Steps**:
1. As project manager, upload document to project
2. Click "Share" button on document
3. Select project team members
4. Send share notifications
5. As team member, verify document in "Shared with Me" section

**Expected Result**: ✅ Document appears in shared view with share() coming from project manager

### Test 3: Delete Document & Verify Cascade

**Steps**:
1. Upload document to projectDocument is in project view & shared recipients have access
2. Delete document as owner
3. Verify document removed from My Documents
4. **As shared recipient**: Verify document no longer accessible

**Expected Result**: ✅ Hard delete + cascade → all recipients lose access immediately

### Test 4: Search Documents

**Steps**:
1. Upload 5 documents with different titles, categories, dates
2. Search for title keyword
3. Filter by category
4. Filter by date range
5. Combine search + filters

**Expected Result**: ✅ Search returns matching documents within 2 seconds

---

## Troubleshooting Guide

### Issue: "File not found" after upload

**Cause**: Physical file saved but database record wasn't persisted (transaction rolled back)

**Solution**: 
- Check application logs for transaction errors
- Verify file storage directory has write permissions
- Ensure database migration was applied (Document table exists)
- Check if IFileStorageService is registered in DI

### Issue: Unauthorized access to project documents

**Cause**: Authorization check failed in DocumentService

**Solution**:
- Verify user is member of project (ProjectMembers table)
- Check if project membership includes user (IDOR protection working correctly)
- Verify Document.ProjectId is set correctly
- Check user's claims include UserId (from CustomAuthenticationStateProvider)

### Issue: Upload modal doesn't show progress

**Cause**: Progress events not firing or StateHasChanged() not called

**Solution**:
- Ensure @bind-Value is used for model properties
- Call StateHasChanged() after updating UploadProgress
- Check browser console for JavaScript errors
- Verify MemoryStream is being written to during upload

### Issue: Cascade delete didn't remove DocumentShare records

**Cause**: FK constraint not configured with Cascade delete

**Solution**:
- Check migration file for OnDelete(DeleteBehavior.Cascade)
- If missing, create new migration:
  ```powershell
  dotnet ef migrations add FixCascadeDelete
  dotnet ef database update
  ```
- Verify SQLite pragma shows CASCADE:
  ```sql
  PRAGMA foreign_key_list(DocumentShares);
  ```

---

## Performance Optimization Tips

1. **Index Strategy**: Create indexes on frequently-queried columns
   - UserId, ProjectId on Documents
   - SharedWithUserId on DocumentShare
   - DocumentId, Action on DocumentActivity

2. **Pagination**: Implement for document lists > 100 items
   ```csharp
   const int pageSize = 25;
   var page = await documents
       .Skip((pageNumber - 1) * pageSize)
       .Take(pageSize)
       .ToListAsync();
   ```

3. **Search Optimization**: For large datasets, consider full-text search
   ```csharp
   // SQLite supports LIKE but not full-text matching by default
   // For training: LIKE is sufficient
   // For production: Implement FTS5 or external search service
   ```

4. **Lazy Loading**: Use .Include() to prevent N+1 queries
   ```csharp
   var docs = await _dbContext.Documents
       .Where(d => d.UserId == userId)
       .Include(d => d.Shares)  // Load shares in single query
       .ToListAsync();
   ```

---

## References

- [Data Model](data-model.md) - Entity definitions, relationships, indexes
- [Research](research.md) - Clarifications and design decisions
- [Plan](plan.md) - Full implementation plan with gates and risks
- [Feature Spec](spec.md) - User stories, requirements, success criteria

---

**Status**: ✅ Quickstart Ready
**Last Updated**: 2026-03-23
**Version**: 1.0