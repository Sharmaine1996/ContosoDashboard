<!--
Sync Impact Report (2026-03-23)
- Version change: 0.0.0 → 1.0.0
- List of modified principles: All principles added (7 core principles defined)
- Added sections: Additional Constraints, Development Workflow
- Removed sections: None
- Templates requiring updates: None - templates are generic and don't reference specific principles
- Follow-up TODOs: None - constitution is complete and aligned
-->
# ContosoDashboard Constitution

## Core Principles

### I. Training-First Development
Every feature in ContosoDashboard is designed primarily for educational purposes in Spec-Driven Development training. Features must demonstrate best practices, include comprehensive documentation, and serve as examples for students learning ASP.NET Core, Blazor, and Entity Framework.

### II. Mock Authentication (NON-NEGOTIABLE)
ContosoDashboard uses cookie-based mock authentication exclusively for training environments. No external identity providers or password hashing required. Authentication must support role-based access control with hierarchical permissions (Employee → TeamLead → ProjectManager → Administrator).

### III. Offline-First Architecture
The application must work completely offline without cloud dependencies. All infrastructure (database, file storage, authentication) uses local implementations with interfaces for future cloud migration. SQLite database with Entity Framework Core ensures portability across platforms.

### IV. Clean Architecture with Service Layer
Business logic is strictly separated into service classes with dependency injection. Controllers/Pages contain only UI logic. Services implement interfaces for testability and maintainability. No direct database access from UI components.

### V. Security by Design
IDOR protection, service-level authorization checks, and defense in depth are mandatory. All endpoints require authentication; role-based policies enforce access control. Security headers (CSP, X-Frame-Options, etc.) are configured by default.

### VI. Comprehensive Testing
Unit tests for services, integration tests for database operations, and UI tests for critical workflows. Test data seeding ensures consistent test environments. Code coverage targets established for all new features.

### VII. Simplicity and Clarity
Start with minimal viable features. Avoid over-engineering. Code must be readable, well-commented, and follow C# best practices. YAGNI principles applied - implement only what's needed for training scenarios.

## Additional Constraints

### Technology Stack Requirements
- Framework: ASP.NET Core 8.0+ with Blazor Server
- Database: SQLite for development, Azure SQL ready for production
- UI: Bootstrap 5.3 with responsive design
- Authentication: Cookie-based mock system (Azure AD interface ready)
- Architecture: Clean separation (Models, Services, Pages, Shared)

### Performance Standards
- Page load times under 2 seconds
- Database queries optimized with proper indexing
- Memory usage monitored in development
- No blocking operations in UI threads

### Documentation Requirements
- README with setup instructions and feature overview
- Code comments for complex business logic
- API documentation for service methods
- Training scenarios documented in StakeholderDocs

## Development Workflow

### Code Review Process
- All PRs require review by at least one team member
- Constitution compliance must be verified
- Security review for authentication/authorization changes
- Performance impact assessment for database changes

### Quality Gates
- Build must pass with no warnings
- Unit test coverage >80% for new code
- Integration tests pass in CI/CD
- Manual testing of critical user workflows

### Deployment Approval
- Development deployments automatic
- Staging requires code review approval
- Production requires additional testing sign-off
- Rollback plan documented for all deployments

## Governance

Constitution supersedes all other practices. Amendments require:
1. Clear rationale documented
2. Impact assessment on existing code
3. Migration plan for implementation
4. Approval from project maintainers

All development activities must verify compliance with these principles. Use StakeholderDocs for runtime development guidance and training scenarios.

**Version**: 1.0.0 | **Ratified**: 2026-03-23 | **Last Amended**: 2026-03-23
