# Data Model: Document Upload and Management

**Feature**: 001-document-management  
**Date**: 2026-03-23  
**Status**: Ready for Implementation

---

## Entity Relationship Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  ┌─────────────┐                ┌──────────────────┐                      │
│  │   User      │──────────────┬─│    Document      │                      │
│  │─────────────│              │ │──────────────────│                      │
│  │ UserId (PK) │              │ │ DocumentId (PK)  │                      │
│  │ Email       │              │ │ UserId (FK)      │──┐ Upload by         │
│  │ DisplayName │              │ │ ProjectId (FK?)  │  │                  │
│  │ ...         │              │ │ Title            │  │                  │
│  └─────────────┘              │ │ Description      │  │                  │
│         │                      │ │ Category         │  │                  │
│         │ 1:N                  │ │ FilePath         │  │                  │
│         │                      │ │ FileSize         │  │                  │
│  ┌──────┴──────┬───────────────┤ │ FileType         │  │                  │
│  │             │               │ │ UploadDate       │  │                  │
│  │      ┌──────┴─────┐         │ │ UpdatedDate      │  │                  │
│  │      │ 1:N        │         │ │ IsDeleted        │  │                  │
│  │      │ ShareBy    │         │ └──────────────────┘  │                  │
│  │      │            │         │          △            │                  │
│  │      └─────┬──────┘         │          │ 1:N        │                  │
│  │            │                │          │            │                  │
│  │            │                │   ┌──────┴────────┐   │                  │
│  │            │                └──→│ DocumentShare │   │                  │
│  │            │                   │───────────────│   │                  │
│  │            │                   │ ShareId (PK)  │   │                  │
│  │            │                   │ DocumentId(FK)├───┘ CASCADE DELETE     │
│  │            │                   │ SharedWithUserId(FK)                   │
│  │            │                   │ SharedByUserId(FK)                     │
│  │            │                   │ ShareDate      │                      │
│  │            └──────────────────→├───────────────┤                      │
│  │                   ShareBy       │ (M:N via User)│                      │
│  │                                 └───────────────┘                      │
│  │                                       △                               │
│  │                                       │                               │
│  │                         ┌─────────────┴───────────┐                  │
│  │                         │                         │                  │
│  │                         │ (Share recipients)      │                  │
│  └─────────────────────────┼─────────────────────────┘                  │
│                            │                                             │
│  ┌─────────────┐           │  ┌───────────────────────┐                 │
│  │   Project   │───────────┼─→│ DocumentActivity      │                 │
│  │─────────────│  1:N      │  │───────────────────────│                 │
│  │ ProjectId   │           │  │ ActivityId (PK)       │                 │
│  │ Name        │           │  │ DocumentId (FK?)      │                 │
│  │ ...         │           │  │ UserId (FK)           │                 │
│  └─────────────┘           │  │ Action                │ Audit trail     │
│         △                  │  │ Details               │ (Upload,        │
│         │ 1:N              │  │ Timestamp             │  Delete,        │
│         │                  │  └───────────────────────┘  Replace, Share)│
│  ┌──────┴───────┐          │                                             │
│  │ Referenced   │          │                                             │
│  │ by Document  │          │                                             │
│  │ (Optional)   │          │                                             │
│  └──────────────┘          │                                             │
│                            │                                             │
└────────────────────────────┼─────────────────────────────────────────────┘
                             │
                             └─ For associated project documents
```

---

## Detailed Entity Definitions

### 1. Document Entity

```csharp
public class Document
{
    public int DocumentId { get; set; }
    
    // Required foreign keys
    public int UserId { get; set; }  // Who uploaded the document
    public User User { get; set; }   // Navigation property
    
    public int? ProjectId { get; set; }  // Optional - if associated with project
    public Project? Project { get; set; }  // Navigation property
    
    // Document metadata (required)
    public string Title { get; set; } = string.Empty;
    public string? Description { get; set; }
    
    // Category from predefined list (stored as string for query simplicity)
    // Values: "Project Documents", "Team Resources", "Personal Files", "Reports", "Presentations", "Other"
    public string Category { get; set; } = string.Empty;
    
    // File storage information
    public string FilePath { get; set; } = string.Empty;  // Relative path with GUID: {userId}/{projectId or "personal"}/{guid}.{ext}
    public long FileSize { get; set; }  // In bytes
    public string FileType { get; set; } = string.Empty;  // MIME type (255 chars for Office docs)
    
    // Background processing status for async virus scanning
    public string Status { get; set; } = "Scanning";  // "Uploading", "Scanning", "Clean", "Infected", "Deleted"
    
    // Timestamps
    public DateTime UploadDate { get; set; }
    public DateTime UpdatedDate { get; set; }
    
    // Soft delete flag (hard delete when true, but we use hard DELETE, not soft delete)
    // Note: Actually uses hard delete (row removed), not soft delete
    [Obsolete("Use hard delete - remove row entirely")]
    public bool IsDeleted { get; set; } = false;
    
    // Navigation properties
    public ICollection<DocumentShare> Shares { get; set; } = new List<DocumentShare>();
    public ICollection<DocumentActivity> Activities { get; set; } = new List<DocumentActivity>();
}
```

**Database Configuration (EF Core)**:

```csharp
modelBuilder.Entity<Document>()
    .HasKey(d => d.DocumentId);

modelBuilder.Entity<Document>()
    .Property(d => d.Title)
    .IsRequired()
    .HasMaxLength(255);

modelBuilder.Entity<Document>()
    .Property(d => d.Category)
    .IsRequired()
    .HasMaxLength(50);  // "Project Documents" is longest

modelBuilder.Entity<Document>()
    .Property(d => d.FilePath)
    .IsRequired()
    .HasMaxLength(500);  // Accommodate long paths with GUID

modelBuilder.Entity<Document>()
    .Property(d => d.FileType)
    .IsRequired()
    .HasMaxLength(255);  // "application/vnd.openxmlformats-officedocument" is long

modelBuilder.Entity<Document>()
    .Property(d => d.Status)
    .HasDefaultValue("Scanning")
    .HasConversion<string>();

// Foreign key relationships
modelBuilder.Entity<Document>()
    .HasOne(d => d.User)
    .WithMany()
    .HasForeignKey(d => d.UserId)
    .OnDelete(DeleteBehavior.Restrict);  // Prevent deletion of user with documents

modelBuilder.Entity<Document>()
    .HasOne(d => d.Project)
    .WithMany()
    .HasForeignKey(d => d.ProjectId)
    .OnDelete(DeleteBehavior.Restrict);  // Prevent deletion of project with documents

// Indexes for performance
modelBuilder.Entity<Document>()
    .HasIndex(d => d.UserId)
    .HasName("IX_Document_UserId");

modelBuilder.Entity<Document>()
    .HasIndex(d => d.ProjectId)
    .HasName("IX_Document_ProjectId");

modelBuilder.Entity<Document>()
    .HasIndex(d => new { d.UserId, d.IsDeleted })
    .HasName("IX_Document_UserId_IsDeleted");

modelBuilder.Entity<Document>()
    .HasIndex(d => d.Category)
    .HasName("IX_Document_Category");

modelBuilder.Entity<Document>()
    .HasIndex(d => d.UploadDate)
    .HasName("IX_Document_UploadDate");
```

---

### 2. DocumentShare Entity

```csharp
public class DocumentShare
{
    public int ShareId { get; set; }
    
    // Foreign keys
    public int DocumentId { get; set; }
    public Document Document { get; set; } = null!;
    
    public int SharedWithUserId { get; set; }  // User who receives the document
    public User SharedWithUser { get; set; } = null!;
    
    public int SharedByUserId { get; set; }  // User who shared the document
    public User SharedByUser { get; set; } = null!;
    
    // Metadata
    public DateTime ShareDate { get; set; }
    
    // Note: No explicit share end date or expiration - shares persist until document is deleted
}
```

**Database Configuration (EF Core)**:

```csharp
modelBuilder.Entity<DocumentShare>()
    .HasKey(s => s.ShareId);

modelBuilder.Entity<DocumentShare>()
    .Property(s => s.ShareDate)
    .HasDefaultValueSql("CURRENT_TIMESTAMP");  // SQLite syntax

// Cascade delete: When Document is deleted, all DocumentShare records are deleted
modelBuilder.Entity<DocumentShare>()
    .HasOne(s => s.Document)
    .WithMany(d => d.Shares)
    .HasForeignKey(s => s.DocumentId)
    .OnDelete(DeleteBehavior.Cascade);  // CRITICAL: CASCADE DELETE

modelBuilder.Entity<DocumentShare>()
    .HasOne(s => s.SharedWithUser)
    .WithMany()
    .HasForeignKey(s => s.SharedWithUserId)
    .OnDelete(DeleteBehavior.Restrict);  // Prevent deletion of user with shares

modelBuilder.Entity<DocumentShare>()
    .HasOne(s => s.SharedByUser)
    .WithMany()
    .HasForeignKey(s => s.SharedByUserId)
    .OnDelete(DeleteBehavior.Restrict);  // Prevent deletion of user who shared

// Indexes for performance
modelBuilder.Entity<DocumentShare>()
    .HasIndex(s => s.DocumentId)
    .HasName("IX_DocumentShare_DocumentId");

modelBuilder.Entity<DocumentShare>()
    .HasIndex(s => s.SharedWithUserId)
    .HasName("IX_DocumentShare_SharedWithUserId");

modelBuilder.Entity<DocumentShare>()
    .HasIndex(s => new { s.DocumentId, s.SharedWithUserId })
    .HasName("IX_DocumentShare_Document_User")
    .IsUnique();  // Prevent duplicate shares
```

---

### 3. DocumentActivity Entity (Audit Logging)

```csharp
public class DocumentActivity
{
    public int ActivityId { get; set; }
    
    // Reference to document (nullable for activities like system scans)
    public int? DocumentId { get; set; }
    public Document? Document { get; set; }
    
    // User performing the action
    public int UserId { get; set; }
    public User User { get; set; } = null!;
    
    // Action type
    public string Action { get; set; } = string.Empty;  // "Upload", "Download", "Delete", "Replace", "Share"
    
    // Additional details (JSON or free text)
    public string? Details { get; set; }
    
    // Timestamp
    public DateTime Timestamp { get; set; }
}
```

**Database Configuration (EF Core)**:

```csharp
modelBuilder.Entity<DocumentActivity>()
    .HasKey(a => a.ActivityId);

modelBuilder.Entity<DocumentActivity>()
    .Property(a => a.Action)
    .IsRequired()
    .HasMaxLength(50);

modelBuilder.Entity<DocumentActivity>()
    .Property(a => a.Details)
    .HasMaxLength(500);

modelBuilder.Entity<DocumentActivity>()
    .Property(a => a.Timestamp)
    .HasDefaultValueSql("CURRENT_TIMESTAMP");

// Foreign key relationships
modelBuilder.Entity<DocumentActivity>()
    .HasOne(a => a.Document)
    .WithMany(d => d.Activities)
    .HasForeignKey(a => a.DocumentId)
    .OnDelete(DeleteBehavior.SetNull);  // Keep activity logs even after document deletion

modelBuilder.Entity<DocumentActivity>()
    .HasOne(a => a.User)
    .WithMany()
    .HasForeignKey(a => a.UserId)
    .OnDelete(DeleteBehavior.Restrict);

// Indexes for audit queries
modelBuilder.Entity<DocumentActivity>()
    .HasIndex(a => a.DocumentId)
    .HasName("IX_DocumentActivity_DocumentId");

modelBuilder.Entity<DocumentActivity>()
    .HasIndex(a => a.UserId)
    .HasName("IX_DocumentActivity_UserId");

modelBuilder.Entity<DocumentActivity>()
    .HasIndex(a => a.Action)
    .HasName("IX_DocumentActivity_Action");

modelBuilder.Entity<DocumentActivity>()
    .HasIndex(a => a.Timestamp)
    .HasName("IX_DocumentActivity_Timestamp");
```

---

## Database Migration Guide

### Entity Framework Core Configuration

Add to `ApplicationDbContext.cs`:

```csharp
using ContosoDashboard.Models;

public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options) { }
    
    // Existing DbSets...
    public DbSet<Document> Documents { get; set; } = null!;
    public DbSet<DocumentShare> DocumentShares { get; set; } = null!;
    public DbSet<DocumentActivity> DocumentActivities { get; set; } = null!;
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        // Configure Document entity (see above)
        // Configure DocumentShare entity (see above)
        // Configure DocumentActivity entity (see above)
        
        // Seed initial categories if desired
        // modelBuilder.Entity<Document>().HasData(...);
    }
}
```

### Creating & Applying Migrations

```powershell
# Create initial migration
dotnet ef migrations add AddDocumentManagement -o Data/Migrations

# Check migration file in Data/Migrations/[timestamp]_AddDocumentManagement.cs
# Verify cascade delete is present: OnDelete(DeleteBehavior.Cascade)

# Apply to database
dotnet ef database update
```

### Verifying SQLite Schema

```sql
-- Check tables created
.tables

-- Verify Document table
PRAGMA table_info(Documents);

-- Verify DocumentShare table includes CASCADE DELETE on DocumentId
PRAGMA foreign_key_list(DocumentShares);

-- Expected output for DocumentShare foreign keys:
-- "from" column       foreign_table  to_column  on_delete
-- ShardWithUserId     Users          UserId     RESTRICT
-- SharedByUserId      Users          UserId     RESTRICT
-- DocumentId          Documents      DocumentId CASCADE ← CRITICAL
```

---

## Query Patterns & Performance

### Common Queries

**Get all documents for user (excluding deleted)**:
```csharp
var userDocs = await dbContext.Documents
    .Where(d => d.UserId == userId && !d.IsDeleted)
    .OrderByDescending(d => d.UploadDate)
    .ToListAsync();

// Uses index: IX_Document_UserId_IsDeleted
```

**Get all project documents**:
```csharp
var projectDocs = await dbContext.Documents
    .Where(d => d.ProjectId == projectId && !d.IsDeleted)
    .OrderByDescending(d => d.UploadDate)
    .ToListAsync();

// Uses index: IX_Document_ProjectId
```

**Get shared documents for user**:
```csharp
var sharedDocs = await dbContext.DocumentShares
    .Where(s => s.SharedWithUserId == userId && !s.Document.IsDeleted)
    .Include(s => s.Document)
    .OrderByDescending(s => s.ShareDate)
    .ToListAsync();

// Uses index: IX_DocumentShare_SharedWithUserId
```

**Search by document title/description**:
```csharp
var searchResults = await dbContext.Documents
    .Where(d => d.UserId == userId && !d.IsDeleted &&
           (d.Title.Contains(query) || d.Description.Contains(query)))
    .OrderByDescending(d => d.UploadDate)
    .ToListAsync();

// Note: Full-text search on SQLite limited; consider adding search index for production
```

**Get audit trail for document**:
```csharp
var history = await dbContext.DocumentActivities
    .Where(a => a.DocumentId == documentId)
    .OrderByDescending(a => a.Timestamp)
    .ToListAsync();

// Uses index: IX_DocumentActivity_DocumentId
```

---

## Data Consistency & Transactions

### Upload Transaction (Must Succeed Atomically)

```csharp
using (var transaction = await dbContext.Database.BeginTransactionAsync())
{
    try
    {
        // 1. Upload file FIRST (not in transaction - if this fails, no orphaned DB record)
        string filePath = await _fileStorageService.UploadAsync(fileStream, fileName);
        
        // 2. Create database record
        var document = new Document
        {
            UserId = userId,
            ProjectId = projectId,
            Title = title,
            Category = category,
            FilePath = filePath,
            FileSize = fileSize,
            FileType = contentType,
            UploadDate = DateTime.UtcNow,
            UpdatedDate = DateTime.UtcNow
        };
        dbContext.Documents.Add(document);
        
        // 3. Log activity
        var activity = new DocumentActivity
        {
            DocumentId = document.DocumentId,
            UserId = userId,
            Action = "Upload",
            Details = $"File: {fileName}, Size: {fileSize} bytes",
            Timestamp = DateTime.UtcNow
        };
        dbContext.DocumentActivities.Add(activity);
        
        // 4. Save all changes
        await dbContext.SaveChangesAsync();
        
        // 5. Commit transaction
        await transaction.CommitAsync();
        
        return document;
    }
    catch (Exception ex)
    {
        await transaction.RollbackAsync();
        // Clean up partially uploaded file if necessary
        await _fileStorageService.DeleteAsync(filePath);
        throw;
    }
}
```

### Delete Transaction with Cascade

```csharp
using (var transaction = await dbContext.Database.BeginTransactionAsync())
{
    try
    {
        // 1. Find document and verify authorization
        var document = await dbContext.Documents.FindAsync(documentId);
        if (document == null || !CanDelete(document, userId))
            throw new UnauthorizedAccessException();
        
        // 2. Delete physical file
        await _fileStorageService.DeleteAsync(document.FilePath);
        
        // 3. Remove from database (cascade delete on DocumentShare)
        dbContext.Documents.Remove(document);
        
        // 4. Log activity
        var activity = new DocumentActivity
        {
            UserId = userId,
            Action = "Delete",
            Details = $"Document: {document.Title}",
            Timestamp = DateTime.UtcNow
        };
        dbContext.DocumentActivities.Add(activity);
        
        // 5. Save changes (cascade delete happens here)
        await dbContext.SaveChangesAsync();
        
        // 6. Commit transaction
        await transaction.CommitAsync();
    }
    catch (Exception ex)
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

---

## Validation Rules

### Document Entity

| Field | Rules |
|-------|-------|
| Title | Required, 1-255 characters, trimmed |
| Description | Optional, 0-1000 characters |
| Category | Required, must be in predefined list |
| FilePath | Required, 1-500 characters, GUID format |
| FileSize | Required, > 0 bytes, ≤ 26,214,400 bytes (25MB) |
| FileType | Required, must be in allowed MIME types list |
| UserId | Required, must reference existing user |
| ProjectId | Optional, must reference existing project if present |

### DocumentShare Entity

| Field | Rules |
|-------|-------|
| DocumentId | Required, must reference existing document |
| SharedWithUserId | Required, must reference existing user, ≠ owner |
| SharedByUserId | Required, must reference existing user (owner) |
| ShareDate | Auto-set to current UTC time |

### DocumentActivity Entity

| Field | Rules |
|-------|-------|
| Action | Required, must be in: Upload, Download, Delete, Replace, Share |
| Details | Optional, 0-500 characters |
| Timestamp | Auto-set to current UTC time |

---

## Summary

✅ **Entity Design**:
- Document: Core entity for file metadata
- DocumentShare: M:N relationship for sharing (individual users + project teams)
- DocumentActivity: Audit trail for compliance

✅ **Relationships**:
- Document → User (who uploaded)
- Document → Project (optional association)
- DocumentShare → Document (cascade delete)
- DocumentActivity → Document (SetNull if document deleted, preserves audit trail)

✅ **Performance**:
- 5 indexes on Document (UserId, ProjectId, combined, Category, UploadDate)
- 3 indexes on DocumentShare (DocumentId, SharedWithUserId, combined unique)
- 4 indexes on DocumentActivity (DocumentId, UserId, Action, Timestamp)

✅ **Data Consistency**:
- Atomic upload transactions (file first, then DB)
- Cascade delete prevents orphaned shares
- Activity logging persists after document deletion

✅ **Offline-Ready**:
- SQLite compatible schema
- No external service dependencies
- Azure SQL migration path (same schema, different connection string)

---

**Status**: ✅ Ready for Implementation
**Last Updated**: 2026-03-23