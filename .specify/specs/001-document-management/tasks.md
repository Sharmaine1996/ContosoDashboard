# Tasks: Document Upload and Management

**Input**: Design documents from `.specify/specs/001-document-management/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: Tests are OPTIONAL - not explicitly requested in specification. Tasks below include test examples but they can be skipped for faster implementation.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Blazor Server app**: `ContosoDashboard/` project root
- **Models**: `ContosoDashboard/Data/Models/`
- **Services**: `ContosoDashboard/Services/`
- **Pages**: `ContosoDashboard/Pages/`
- **Tests**: `ContosoDashboard.Tests/` (if created)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Create feature branch `001-document-management` and verify .specify directory structure
- [ ] T002 Verify .NET 10.0 SDK and all NuGet packages are compatible
- [ ] T003 [P] Create ContosoDashboard.Tests project for unit/integration tests (optional)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T004 Create Document entity in ContosoDashboard/Data/Models/Document.cs with all properties and validation
- [ ] T005 Create DocumentShare entity in ContosoDashboard/Data/Models/DocumentShare.cs with relationships
- [ ] T006 Create DocumentActivity entity in ContosoDashboard/Data/Models/DocumentActivity.cs for audit logging
- [ ] T007 Update ApplicationDbContext.cs to include new DbSets and EF Core configuration (indexes, relationships, cascade deletes)
- [ ] T008 Create EF Core migration: `dotnet ef migrations add AddDocumentManagement`
- [ ] T009 Apply migration and verify database schema: `dotnet ef database update`
- [ ] T010 Create IFileStorageService interface in ContosoDashboard/Services/IFileStorageService.cs
- [ ] T011 Implement LocalFileStorageService in ContosoDashboard/Services/LocalFileStorageService.cs (GUID-based paths)
- [ ] T012 Create IVirusScanService interface in ContosoDashboard/Services/IVirusScanService.cs
- [ ] T013 Implement MockVirusScanService in ContosoDashboard/Services/MockVirusScanService.cs (training implementation)
- [ ] T014 Create IDocumentService interface in ContosoDashboard/Services/IDocumentService.cs with all business methods
- [ ] T015 Implement DocumentService in ContosoDashboard/Services/DocumentService.cs with authorization checks
- [ ] T016 Implement DocumentController POST /api/documents endpoint in ContosoDashboard/Controllers/DocumentController.cs
  - Depends on: T015 (DocumentService)
  - Note: Include comprehensive error handling for file size limits and unsupported types
- [ ] T017 Register all services in Program.cs dependency injection container
- [ ] T018 Test application startup and database seeding to verify foundational setup

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - Upload Personal Documents (Priority: P1) 🎯 MVP

**Goal**: Enable employees to upload work documents with progress tracking and validation

**Independent Test**: Upload a single document, verify it appears in database and can be downloaded

### Tests for User Story 1 (OPTIONAL) ⚠️

- [ ] T019 [P] [US1] Unit test for DocumentService.UploadAsync in ContosoDashboard.Tests/Services/DocumentServiceTests.cs
- [ ] T020 [P] [US1] Integration test for file upload workflow in ContosoDashboard.Tests/Integration/DocumentUploadTests.cs

### Implementation for User Story 1

- [ ] T021 [P] [US1] Create DocumentUploadModal.razor component in ContosoDashboard/Pages/DocumentUploadModal.razor with async upload and progress
- [ ] T022 [P] [US1] Add file validation logic (size < 25MB, supported types) to DocumentService.UploadAsync
- [ ] T023 [P] [US1] Implement upload progress tracking with CancellationToken support in DocumentUploadModal.razor
- [ ] T024 [US1] Add DocumentUploadModal to Index.razor or create dedicated upload page
- [ ] T025 [US1] Test upload functionality: select file, enter metadata, upload with progress, verify database record created
- [ ] T026 [US1] Add error handling for upload failures (network issues, validation errors) in modal

**Checkpoint**: At this point, User Story 1 should be fully functional - users can upload documents with progress feedback

---

## Phase 4: User Story 2 - Browse and Search Documents (Priority: P1)

**Goal**: Enable employees to view and search their uploaded documents

**Independent Test**: Upload multiple documents, search and filter to verify correct results

### Tests for User Story 2 (OPTIONAL) ⚠️

- [ ] T027 [P] [US2] Unit test for DocumentService.SearchAsync in ContosoDashboard.Tests/Services/DocumentServiceTests.cs
- [ ] T028 [P] [US2] Integration test for search and filtering in ContosoDashboard.Tests/Integration/DocumentSearchTests.cs

### Implementation for User Story 2

- [ ] T029 [P] [US2] Create Documents.razor page in ContosoDashboard/Pages/Documents.razor for "My Documents" view
- [ ] T030 [P] [US2] Implement document list display with sorting (title, date, category, size) in Documents.razor
- [ ] T031 [P] [US2] Add category filter dropdown to Documents.razor
- [ ] T032 [P] [US2] Add date range filter to Documents.razor
- [ ] T033 [P] [US2] Implement search input with full-text search in DocumentService.SearchAsync
- [ ] T034 [US2] Add download functionality to document list items
- [ ] T035 [US2] Implement pagination for large document lists (50+ documents)
- [ ] T036 [US2] Test search performance: verify results return within 2 seconds for 500 documents

**Checkpoint**: At this point, User Stories 1 AND 2 should both work - users can upload and find documents

---

## Phase 5: User Story 3 - Manage Project Documents (Priority: P2)

**Goal**: Enable project managers and team members to upload and access project-related documents

**Independent Test**: Project manager uploads document to project, team members can view and download it

### Tests for User Story 3 (OPTIONAL) ⚠️

- [ ] T037 [P] [US3] Unit test for DocumentService.GetProjectDocumentsAsync in ContosoDashboard.Tests/Services/DocumentServiceTests.cs
- [ ] T038 [P] [US3] Integration test for project document access control in ContosoDashboard.Tests/Integration/ProjectDocumentTests.cs

### Implementation for User Story 3

- [ ] T039 [P] [US3] Add "Project Documents" section to ProjectDetails.razor page
- [ ] T040 [P] [US3] Implement project document upload in ProjectDetails.razor (project managers only)
- [ ] T041 [P] [US3] Add project document list display in ProjectDetails.razor
- [ ] T042 [US3] Implement DocumentService.GetProjectDocumentsAsync with project membership authorization
- [ ] T043 [US3] Allow team members to upload to projects (if permitted by business rules)
- [ ] T044 [US3] Test project document access: verify only project members can see project documents

**Checkpoint**: User Stories 1, 2, AND 3 should work - project collaboration enabled

---

## Phase 6: User Story 4 - Share Documents with Team Members (Priority: P2)

**Goal**: Enable document owners to share documents with individual users and project teams

**Independent Test**: User shares document with another user, recipient can access it

### Tests for User Story 4 (OPTIONAL) ⚠️

- [ ] T045 [P] [US4] Unit test for DocumentService.ShareAsync in ContosoDashboard.Tests/Services/DocumentServiceTests.cs
- [ ] T046 [P] [US4] Integration test for document sharing workflow in ContosoDashboard.Tests/Integration/DocumentSharingTests.cs

### Implementation for User Story 4

- [ ] T047 [P] [US4] Create DocumentShareModal.razor component for selecting users/teams to share with
- [ ] T048 [P] [US4] Implement DocumentService.ShareAsync for individual users and project teams
- [ ] T049 [P] [US4] Add share button to document list items in Documents.razor and ProjectDetails.razor
- [ ] T050 [P] [US4] Create "Shared with Me" section in Documents.razor
- [ ] T051 [US4] Implement DocumentService.GetSharedDocumentsAsync for recipient access
- [ ] T052 [US4] Add notification system integration (existing NotificationService)
- [ ] T053 [US4] Test sharing: share document, verify recipient can access, check notifications sent

**Checkpoint**: User Stories 1-4 should work - full document sharing and collaboration enabled

---

## Phase 7: User Story 5 - Edit and Delete Documents (Priority: P3)

**Goal**: Enable document owners to maintain document metadata and remove unwanted documents

**Independent Test**: Edit document metadata, then delete document and verify cascade behavior

### Tests for User Story 5 (OPTIONAL) ⚠️

- [ ] T054 [P] [US5] Unit test for DocumentService.UpdateMetadataAsync and DeleteAsync in ContosoDashboard.Tests/Services/DocumentServiceTests.cs
- [ ] T055 [P] [US5] Integration test for cascade delete behavior in ContosoDashboard.Tests/Integration/DocumentLifecycleTests.cs

### Implementation for User Story 5

- [ ] T056 [P] [US5] Create DocumentDetailsModal.razor for editing metadata (title, description, category)
- [ ] T057 [P] [US5] Implement DocumentService.UpdateMetadataAsync with owner authorization
- [ ] T058 [P] [US5] Add edit button to document list items and implement modal integration
- [ ] T059 [P] [US5] Implement DocumentService.DeleteAsync with hard delete and cascade to DocumentShare
- [ ] T060 [P] [US5] Add delete button with confirmation dialog to document actions
- [ ] T061 [P] [US5] Implement file replacement functionality (overwrite existing file, log activity)
- [ ] T062 [US5] Test delete cascade: delete document, verify shares removed, activity logged

**Checkpoint**: User Stories 1-5 should work - full document lifecycle management complete

---

## Phase 8: User Story 6 - Dashboard Integration (Priority: P3)

**Goal**: Show recent document activity and statistics on the main dashboard

**Independent Test**: Upload documents and verify they appear in dashboard widgets

### Tests for User Story 6 (OPTIONAL) ⚠️

- [ ] T063 [P] [US6] Unit test for dashboard document queries in ContosoDashboard.Tests/Services/DashboardServiceTests.cs
- [ ] T064 [P] [US6] UI test for dashboard widgets in ContosoDashboard.Tests/UI/DashboardTests.cs

### Implementation for User Story 6

- [ ] T065 [P] [US6] Add "Recent Documents" widget to Index.razor dashboard
- [ ] T066 [P] [US6] Implement DocumentService.GetRecentDocumentsAsync (last 5 uploaded)
- [ ] T067 [P] [US6] Add document count to dashboard statistics cards
- [ ] T068 [US6] Integrate with existing dashboard layout and styling
- [ ] T069 [US6] Test dashboard integration: verify recent documents appear, counts update

**Checkpoint**: All user stories should now be independently functional

---

## Phase 9: Background Processing (Production Enhancement)

**Purpose**: Async virus scanning using Azure Functions (production only)

- [ ] T070 Create Azure Functions project for background processing
- [ ] T071 Implement ProcessVirusScan function with Queue Storage trigger
- [ ] T072 Add Document.Status field support to UI (scanning/clean/infected states)
- [ ] T073 Update DocumentService to queue scan requests for production
- [ ] T074 Configure Azure Queue Storage and Function App settings
- [ ] T075 Test background processing: upload triggers queue, function updates status

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] T076 [P] Add comprehensive error handling and user-friendly error messages across all components
- [ ] T077 [P] Implement audit logging for all document operations (DocumentActivity table)
- [ ] T078 [P] Add loading states and skeleton screens for better UX
- [ ] T079 [P] Performance optimization: add database indexes, implement caching where appropriate
- [ ] T080 [P] Security hardening: input validation, XSS protection, CSRF protection
- [ ] T081 [P] Accessibility improvements: ARIA labels, keyboard navigation, screen reader support
- [ ] T082 [P] Documentation updates: update README.md, add API documentation
- [ ] T083 Run end-to-end testing across all user stories
- [ ] T084 Code cleanup and refactoring for maintainability

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3-8)**: All depend on Foundational phase completion
  - User stories can proceed in parallel (if staffed) or sequentially in priority order (P1 → P2 → P3)
- **Polish (Phase 10)**: Depends on all desired user stories being complete

### User Story Dependencies

- **US1 (P1)**: Can start after Foundational - No dependencies on other stories
- **US2 (P1)**: Can start after Foundational - Independent but works with US1
- **US3 (P2)**: Can start after Foundational - May reference US1 patterns
- **US4 (P2)**: Can start after Foundational - May reference US1/US2 patterns
- **US5 (P3)**: Can start after Foundational - May reference US1-US4 patterns
- **US6 (P3)**: Can start after Foundational - May reference US1-US5 patterns

### Within Each User Story

- Models before services before UI components
- Core functionality before advanced features
- Error handling and validation throughout

### Parallel Opportunities

- All Setup tasks marked [P] can run in parallel
- All Foundational tasks marked [P] can run in parallel
- Once Foundational completes, all user stories can start in parallel
- Models, services, and UI components within stories can be parallelized
- Different user stories can be worked on by different team members

---

## Parallel Example: User Story 1

```bash
# Launch all foundational service implementations together:
Task: "Create IFileStorageService interface in ContosoDashboard/Services/IFileStorageService.cs"
Task: "Implement LocalFileStorageService in ContosoDashboard/Services/LocalFileStorageService.cs"
Task: "Create IVirusScanService interface in ContosoDashboard/Services/IVirusScanService.cs"
Task: "Implement MockVirusScanService in ContosoDashboard/Services/MockVirusScanService.cs"

# Launch User Story 1 components together:
Task: "Create DocumentUploadModal.razor component in ContosoDashboard/Pages/DocumentUploadModal.razor"
Task: "Add file validation logic to DocumentService.UploadAsync"
Task: "Implement upload progress tracking in DocumentUploadModal.razor"
```

---

## Implementation Strategy

### MVP First (User Stories 1-2 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL)
3. Complete Phase 3: User Story 1 (Upload)
4. Complete Phase 4: User Story 2 (Browse/Search)
5. **STOP and VALIDATE**: Test core upload/search functionality
6. Deploy/demo MVP if ready

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add US1 + US2 → Test independently → Deploy/Demo (MVP!)
3. Add US3 + US4 → Test independently → Deploy/Demo (Collaboration)
4. Add US5 + US6 → Test independently → Deploy/Demo (Full feature)
5. Each increment adds value without breaking previous functionality

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: US1 (Upload) + US2 (Browse)
   - Developer B: US3 (Projects) + US4 (Sharing)
   - Developer C: US5 (Edit/Delete) + US6 (Dashboard)
3. Stories complete and integrate independently

---

## Notes

- [P] tasks = different files, no dependencies - can run in parallel
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each task or logical group of related tasks
- Stop at any checkpoint to validate story independently
- Background processing (Phase 9) is production-only enhancement
- Tests are optional - skip if not requested for faster delivery</content>
<parameter name="filePath">/Users/sharmaine.miranda/Documents/training-projects/ContosoDashboard/.specify/specs/001-document-management/tasks.md