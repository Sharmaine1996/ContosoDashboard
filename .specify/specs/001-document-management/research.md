# Research & Clarifications: Document Upload and Management

**Feature**: 001-document-management  
**Date**: 2026-03-23  
**Status**: Complete

## Clarification Decisions (Session 2026-03-23)

All ambiguities identified during specification were systematically clarified in a 5-question workflow. This document records decisions and rationales.

---

## Q1: Document Sharing Scope

**Question**: When users share documents, who can be recipients?

**Options Considered**:
- A: Individual users only (simpler but limits collaboration)
- B: Individual users AND teams (recommended)
- C: Individual users, teams, AND departments (most flexible but complex)
- D: Skip sharing for P2, implement in P3

**Decision**: **Option B - Individual users AND teams**

**Rationale**:
- Balance between flexibility and implementation simplicity
- Teams (projects) already exist in ContosoDashboard data model
- Enables project managers to quickly share documents with entire team
- Training appropriate: teaches both 1:1 and group sharing patterns
- Avoids department-level complexity not needed for MVP

**Impact on Specification**:
- FR-018 updated: "System MUST implement document sharing with individual users and teams (project-based groups)"
- User Story 4 acceptance scenarios updated to cover team sharing
- DocumentShare entity: foreign key to both individual users and projects (separate relationships)

**Implementation Note**: When sharing with a team/project, create individual DocumentShare records for each project member (simpler than team->document relationship).

---

## Q2: Virus/Malware Scanning

**Question**: For a training-only, offline-first application, how should virus scanning be implemented?

**Options Considered**:
- A: Mock implementation (simulated, always passes)
- B: Real scanning via local library (ClamAV)
- C: Interface abstraction (mock for training, real for production)
- D: Remove requirement, document as future feature

**Decision**: **Option C - Interface abstraction (IVirusScanService)**

**Rationale**:
- Aligns with Constitution principle: "Offline-First Architecture with cloud migration path"
- Training implementation: `MockVirusScanService` (always returns safe with logging)
- Production implementation: Real scanning via ClamAV or cloud service
- Teaches students about dependency injection and abstraction patterns
- Zero external dependencies for training environment
- Maintains compliance with offline-first requirement
- No knowledge of Windows Defender, ClamAV, or cloud scanning services required for training users

**Design**:
```csharp
public interface IVirusScanService
{
    Task<(bool IsSafe, string Message)> ScanAsync(Stream fileStream, string fileName);
}

// Training
public class MockVirusScanService : IVirusScanService
{
    public async Task<(bool IsSafe, string Message)> ScanAsync(Stream fileStream, string fileName)
    {
        _logger.LogInformation($"Mock scan: {fileName} - PASSED");
        return (true, "Scan passed (mock implementation)");
    }
}

// Production (future)
public class ClamAvVirusScanService : IVirusScanService { ... }
```

**Impact on Specification**:
- FR-007 updated: "System MUST implement virus/malware scanning via abstracted `IVirusScanService` interface"
- Assumptions updated: "Training uses mock virus scanning (always returns safe)"
- Out of Scope updated: Clarifies no real malware detection in training

---

## Q3: Document Deletion & Cascade Behavior

**Question**: When someone deletes a document they own, what happens to document shares and recipient access?

**Options Considered**:
- A: Hard delete - document immediately removed, all shares deleted, recipients lose access instantly
- B: Soft delete - marked as deleted in DB, shares become inaccessible (data preserved)
- C: Recipient retention - recipients keep access for 30 days, then automatic removal
- D: Notify before delete - send notifications, allow 7-day download window before permanent removal

**Decision**: **Option A - Hard delete with cascade**

**Rationale**:
- Constitution principle: "Simplicity and Clarity" - no soft delete complexity
- Aligns with document ownership model (owner has full control, immediate effect)
- Prevents confusion about deleted document state
- Database cascade delete automatically removes DocumentShare records
- No orphaned data or cleanup jobs needed
- Training appropriate: clear causality (delete = gone)
- Production can add soft-delete/recovery features later

**Database Design**:
```csharp
modelBuilder.Entity<DocumentShare>()
    .HasOne(s => s.Document)
    .WithMany()
    .HasForeignKey(s => s.DocumentId)
    .OnDelete(DeleteBehavior.Cascade);  // CASCADE DELETE
```

**Impact on Specification**:
- FR-017 updated: "System MUST allow document owners and project managers to delete documents with hard delete (immediate removal, cascading share deletion)"
- Key Entities updated: DocumentShare cascade behavior documented
- Edge case clarified: deletion is irreversible; recipients immediately lose access

---

## Q4: File Replacement & Version Tracking

**Question**: When a document file is replaced with a new version, what happens to the old file and access history?

**Options Considered**:
- A: Simple replacement - new file overwrites old (physical), metadata unchanged, no history
- B: Version history - old files kept with versioning (guid_v1, guid_v2), users can rollback
- C: Audit trail only - file replaced, DocumentActivity logs replacement event (compliance), no user-facing versions
- D: Renamed old - old file moved to archive directory, users cannot access old versions

**Decision**: **Option C - Audit trail only**

**Rationale**:
- Constitution principle: "Simplicity and Clarity" - no complex version management
- Supports audit requirements (FR-025 activity logging covers compliance needs)
- Storage-efficient (single physical file per document)
- Training appropriate: teaches audit pattern without versioning complexity
- Corporate needs met via DocumentActivity event logging
- Version history can be future enhancement if business demands

**Design**:
```csharp
// DocumentService.ReplaceFileAsync()
1. Validate: new file type, size, virus scan
2. Delete old physical file
3. Save new physical file
4. Update Document.FilePath, UpdatedDate
5. Log DocumentActivity("Replace", oldFileSize, newFileSize)
6. Commit transaction
```

**Impact on Specification**:
- FR-016 updated: "System MUST allow document owners to replace files (overwrite, audit logged, no version history)"
- Out of Scope clarified: "Version history and rollback capabilities" explicitly excluded (audit trail only)
- Assumptions updated: "File replacement uses simple overwrite, audit trail only"

---

## Q5: Upload UI/UX & Async Behavior

**Question**: For Blazor Server file upload, should uploads be synchronous (blocking UI) or asynchronous (background)?

**Options Considered**:
- A: Blocking upload - UI freezes during upload, simplest to implement, poor UX for large files
- B: Background with modal - upload in background, progress modal user can cancel, UI partially responsive
- C: Fully async/responsive - upload completely in background after dialog closes, notification on completion
- D: User choice - small files (<5MB) block, large files background

**Decision**: **Option B - Background with modal & cancel**

**Rationale**:
- Blazor Server can handle async file uploads with SignalR
- Modal prevents accidental navigation away (guard against incomplete uploads)
- Cancel button allows users to stop uploads if network issues occur
- Progress indicator matches FR-001 requirement ("Users should see a progress indicator")
- Better UX than blocking without over-engineering
- Training appropriate: teaches async Blazor patterns and SignalR communication
- Works well with 30-second performance target for 25MB files
- Simple to implement relative to fully-async (Option C)

**Blazor Pattern**:
```csharp
// DocumentUploadModal.razor
<div class="modal" @if(IsUploading) Shown>
    <p>Uploading: <strong>@FileName</strong></p>
    <div class="progress">
        <div class="progress-bar" style="width: @($"{UploadProgress}%")">
            @UploadProgress%
        </div>
    </div>
    <button onclick="CancelUpload">Cancel</button>
</div>

// When user cancels:
// 1. StopUpload signal sent to server
// 2. IFileStorageService.UploadAsync() can monitor cancellation token
// 3. Partial file cleaned up
// 4. Modal closes, user returned to document list
```

**Impact on Specification**:
- User Story 1 acceptance scenarios updated: Modal pattern documented (progress bar, cancel button)
- FR-001 updated: "System MUST allow authenticated users to upload files...with progress indicator in modal dialog that runs asynchronously"
- Assumptions updated: "Upload UI implemented as modal dialog with async background processing"

---

## Implementation Guidance Based on Clarifications

### File Upload Workflow (All 5 Clarifications Applied)

1. **User selects file** (Input validation: type, size, name)
2. **Virus scan** (via MockVirusScanService - always returns safe for training)
3. **Generate unique GUID path** (e.g., `{userId}/personal/{guid}.pdf`) before DB access
4. **Show upload modal** (Progress bar, async background upload)
5. **User can cancel** (CancellationToken propagated to service)
6. **Save file** (IFileStorageService.UploadAsync → disk)
7. **Create DB record** (Document + DocumentActivity log)
8. **Log activity** ("Upload", fileName, fileSize)
9. **Notify recipients** (if document will be shared later)

### Document Deletion With Cascade (Q3 Applied)

1. Service checks authorization (document owner or project manager)
2. Mark file for deletion in transaction
3. Delete physical file via IFileStorageService.DeleteAsync()
4. Delete Document record (cascade deletes all DocumentShare records)
5. Log DocumentActivity("Delete")
6. Commit transaction
7. → All recipients immediately lose access (no grace period)

### File Replacement Audit Trail (Q4 Applied)

1. Upload new version (same workflow as initial upload)
2. EF Core update Document.FilePath to new GUID path
3. Delete old physical file
4. Log DocumentActivity("Replace", oldPath, newPath, oldSize, newSize)
5. Commit transaction
6. → No user-facing version history, but audit log shows replacement history

### Team Sharing (Q1 Applied)

1. Document owner selects recipients (individual users + project teams)
2. For individual users: create DocumentShare(DocumentId, UserId, SharedByUserId)
3. For team/project: create DocumentShare for each project member individually
4. Send notifications to all recipients
5. Log DocumentActivity("Share")
6. Recipients see document in "Shared with Me" section + project documents tab

---

## Risks Mitigated

| Clarification | Risk Mitigated |
|---------------|----------------|
| Q1: Team Sharing (not just individual) | Risk: Feature too limited for practical use; Mitigation: Teams enable collaboration at project scope |
| Q2: Interface abstraction (not mock-only or real-only) | Risk: Training environment bloated with virus scanning; Mitigation: Mock implementation enables offline training |
| Q3: Hard delete (not soft delete) | Risk: Data bloat, complexity; Mitigation: Simple cascade delete, clear semantics |
| Q4: Audit trail (not version history) | Risk: Over-engineering; Mitigation: Logging satisfies compliance without complex versioning |
| Q5: Modal with cancel (not blocking or fully async) | Risk: Poor UX or over-engineering; Mitigation: Modal strikes balance—responsive UI, clear upload feedback, cancel capability |

---

## Dependencies Resolved

✅ All specification ambiguities resolved
✅ No "NEEDS CLARIFICATION" markers remain in spec.md
✅ Implementation team has clear guidance on each design decision
✅ Rationale documented for training/learning purposes

---

## Next Steps

→ Proceed to Phase 1 Design Artifacts:
- data-model.md (entity definitions)
- contracts/ (service interfaces)
- quickstart.md (developer guide)

→ Then to Phase 2 (speckit.tasks):
- Task breakdown with acceptance criteria
- Detailed implementation guidance per task

---

**Status**: ✅ Research Complete
**Last Updated**: 2026-03-23