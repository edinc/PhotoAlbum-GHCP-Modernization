# Configuration & Externalized Settings Inventory

This document inventories configuration files, runtime profiles, and sensitive settings used by PhotoAlbum.

## Configuration Sources

| Source | Type | Path/Location | Notes |
|---|---|---|---|
| appsettings.json | Application config | `PhotoAlbum/appsettings.json` | Connection string, upload limits, logging, host settings |
| appsettings.Development.json | Environment override | `PhotoAlbum/appsettings.Development.json` | Development logging override |
| launchSettings.json | Local run profile config | `PhotoAlbum/Properties/launchSettings.json` | Local URLs and ASPNETCORE_ENVIRONMENT values |
| User Secrets | Secret store reference | `UserSecretsId` in `PhotoAlbum.csproj` | Development secret source for local machine |

## Build Profiles

| Profile | Activation | Purpose | Key Dependencies/Plugins |
|---|---|---|---|
| Debug | `dotnet build` default local configuration | Development compile/debug symbols | Standard SDK build pipeline |
| Release | `dotnet build -c Release` | Optimized production build output | Standard SDK build pipeline |

## Runtime Profiles

| Profile | Activation Method | Config Files | Key Overrides |
|---|---|---|---|
| Development | `ASPNETCORE_ENVIRONMENT=Development` (launch settings) | `appsettings.json` + `appsettings.Development.json` | Local ports, development logging |
| Production/Default | Environment default when Development not set | `appsettings.json` | Baseline connection and upload settings |
| Test | `IsTestEnvironment=true` config value during tests | Test host configuration | Skips startup migration execution |

## Properties Inventory

| Property Key | Default | Profiles | Source |
|---|---|---|---|
| ConnectionStrings:DefaultConnection | LocalDB connection string | Default/Development | appsettings.json |
| FileUpload:MaxFileSizeBytes | 10485760 | Default/Development | appsettings.json |
| FileUpload:AllowedMimeTypes | jpeg,png,gif,webp | Default/Development | appsettings.json |
| FileUpload:MaxFilesPerUpload | 10 | Default/Development | appsettings.json |
| FileUpload:UploadPath | `wwwroot/uploads` | Default/Development | appsettings.json |
| Logging:LogLevel:Default | Information | Default/Development | appsettings.json |
| Logging:LogLevel:Microsoft.AspNetCore | Warning | Default/Development | appsettings.json |
| AllowedHosts | `*` | Default/Development | appsettings.json |
| ASPNETCORE_ENVIRONMENT | Development (launch profiles) | Development | launchSettings.json |
| IsTestEnvironment | false (unless overridden in tests) | Test override | Runtime/test config |

## Startup Parameters & Resource Requirements

| Service | JVM/Runtime Options | Memory | Instance Count |
|---|---|---|---|
| PhotoAlbum | .NET runtime defaults, no explicit startup flags in repo | Not explicitly configured | 1 process (local app instance) |

## Startup Dependency Chain

1. PhotoAlbum process starts and loads configuration.
2. Upload directory is created if missing.
3. Database migration runs unless test mode is enabled.
4. HTTP pipeline starts accepting requests.

## Secrets & Sensitive Configuration

| Secret Reference | Type | Storage (masked) |
|---|---|---|
| `ConnectionStrings:DefaultConnection` | Database connection string | appsettings/User Secrets (credentials masked if present) |
| `UserSecretsId` reference | Secret source mapping | Local development secret store |

### Secrets Provisioning Workflow

Primary configuration values are loaded from JSON files, with sensitive local overrides expected through .NET User Secrets during development. There is no Key Vault, managed identity, or centralized secret manager configuration in the repository.

## Feature Flags

| Flag Name | Default | Controlled By |
|---|---|---|
| IsTestEnvironment | false | Runtime configuration value used to skip migrations in test context |

## Framework & Runtime Versions

| Component | Version | Source |
|---|---|---|
| .NET target framework | net9.0 | `PhotoAlbum.csproj`, `PhotoAlbum.Tests.csproj` |
| ASP.NET Core | 9.0 (via SDK) | `Microsoft.NET.Sdk.Web` |
| EF Core SQL Server | 9.0.9 | `PhotoAlbum.csproj` |
| EF Core InMemory | 9.0.9 | `PhotoAlbum.Tests.csproj` |
| ImageSharp | 3.1.11 | `PhotoAlbum.csproj` |
| xUnit | 2.9.2 | `PhotoAlbum.Tests.csproj` |
