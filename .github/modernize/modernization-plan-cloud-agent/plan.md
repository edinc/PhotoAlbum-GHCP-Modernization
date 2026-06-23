# Modernization Plan: PhotoAlbum Azure modernization

**Project**: PhotoAlbum

---

## Technical Framework

- **Language**: C# / .NET 9.0
- **Framework**: ASP.NET Core Razor Pages
- **Build Tool**: dotnet / MSBuild
- **Database**: SQL Server LocalDB for photo metadata
- **Key Dependencies**: EF Core, ImageSharp, xUnit

---

## Overview

> This migration modernizes PhotoAlbum for Azure based on assessment
> `report-20260623132400`. The application currently stores uploaded photo files
> on the local file system, keeps photo metadata in SQL Server LocalDB, and
> relies on repository-local configuration files for runtime settings.
>
> The new architecture will:
>
> - Move photo file storage to Azure Blob Storage to remove local disk coupling
> - Move metadata persistence to Azure SQL Database for cloud-hosted storage
> - Externalize application settings and address runtime/security upgrade work
>
> The migration follows a staged approach: upgrade the runtime baseline first,
> modernize storage and configuration dependencies next, and complete security
> remediation before any later deployment work.

---

## Migration Impact Summary

| App | Original | Azure Service | Authentication | Comments |
|-----|----------|---------------|----------------|----------|
| PhotoAlbum | Local uploads | Azure Blob Storage | Managed identity | Replaces local photo files |
| PhotoAlbum | SQL Server LocalDB | Azure SQL Database | Managed identity | Replaces local metadata DB |
| PhotoAlbum | appsettings.json | Azure App Configuration | Managed identity | Moves non-secret settings |

---

## Open Questions & Questionnaire

- [x] Q: Which assessment should drive this plan? → A: Use
  `report-20260623132400`, the existing assessment under
  `.github/modernize/assessment/reports/`.
- [x] Q: Should the plan include environment/infrastructure provisioning? → A:
  No — the repository already contains Azure setup and deployment assets, so
  this plan focuses on code migration only.
- [x] Q: Should the plan include integration testing to verify migrated
  services? → A: No additional integration-test task is included because the
  request did not explicitly ask for generated integration tests.
- [x] Q: Should the plan include a security scan and CVE remediation task? →
  A: Yes — include the required security/CVE remediation task.
- [x] Q: Which Azure deployment target should the plan use? → A: No deployment
  task is included; this plan is limited to modernization planning scope.
- [x] Q: Should the plan include containerization? → A: No separate
  containerization task is included because deployment is out of scope.
