# Configuration & Externalized Settings Inventory

The repository uses a small set of configuration sources centered on `appsettings`, launch profiles, Azure deployment templates, and a user-secrets identifier. It has one meaningful runtime profile in day-to-day development (`Development`) plus a production-oriented Azure Container Apps configuration path.

## Configuration Sources

| Source | Type | Path/Location | Notes |
|---|---|---|---|
| `appsettings.json` | Runtime application settings | `PhotoAlbum\appsettings.json` | Base settings for connection string, file upload limits, logging, and allowed hosts |
| `appsettings.Development.json` | Runtime environment override | `PhotoAlbum\appsettings.Development.json` | Development-only logging and detailed error settings |
| `launchSettings.json` | Local launch profile config | `PhotoAlbum\Properties\launchSettings.json` | Defines local URLs and `ASPNETCORE_ENVIRONMENT=Development` |
| Project user secrets identifier | Secret indirection | `PhotoAlbum\PhotoAlbum.csproj` | Declares `UserSecretsId`; no secrets file content is checked in |
| Dockerfile | Container runtime config | `Dockerfile` | Sets container base images and exposes port `8080` |
| Azure Developer CLI manifest | Deployment config source | `azure.yaml` | Declares the `web` service, container app host, infra module, and deploy hooks |
| Azure infra template | Externalized deployment settings | `infra\main.json` | Defines Azure SQL, storage, container app env vars, scaling, and identity-based registry access |
| Azure infra parameters | Deployment parameter defaults | `infra\main.parameters.json` | Supplies environment/location placeholders and image name default |
| Infra hooks | Deployment-time scripting | `infra\hooks\predeploy.ps1`, `infra\hooks\postprovision.ps1` | Configure managed identity registry access and wait for role propagation |
| Example environment file | Operator input template | `.env.example` | Placeholder for Azure deployment environment values such as `RESOURCE_GROUP` |

## Build Profiles

| Profile | Activation | Purpose | Key Dependencies/Plugins |
|---|---|---|---|
| `Debug` | Manual or default local build configuration | Developer-oriented compilation and local debugging | Standard SDK tooling; no profile-specific package additions found |
| `Release` | Manual `-c Release` selection and Docker publish flow | Optimized build and publish output | Used in `Dockerfile` build and publish stages |

## Runtime Profiles

| Profile | Activation Method | Config Files | Key Overrides |
|---|---|---|---|
| Default | Implicit when no environment-specific override is supplied | `appsettings.json` | Base connection string, upload limits, upload path, log levels, allowed hosts |
| Development | `ASPNETCORE_ENVIRONMENT=Development` from launch profiles | `appsettings.json` plus `appsettings.Development.json` | Enables `DetailedErrors`, raises default and EF Core logging verbosity, uses local HTTP and HTTPS URLs |
| Production | `ASPNETCORE_ENVIRONMENT=Production` injected by Azure infra | `appsettings.json` plus environment variables from `infra\main.json` | Replaces database connection with `ConnectionStrings__DefaultConnection`; adds Azure blob-related environment variables |
| Test | `IsTestEnvironment` checked in code and in-memory config created by tests | In-memory configuration inside `PhotoServiceTests` | Uses in-memory database provider semantics and custom upload path for tests |

## Properties Inventory

### `PhotoAlbum`

| Property Key | Default | Profiles | Source |
|---|---|---|---|
| `ConnectionStrings:DefaultConnection` | `[MASKED]` | Default, Development; overridden in Production | `appsettings.json`; `ConnectionStrings__DefaultConnection` from `infra\main.json` |
| `FileUpload:MaxFileSizeBytes` | `10485760` | Default, Development, Test | `appsettings.json`; duplicated in `PhotoServiceTests` in-memory config |
| `FileUpload:AllowedMimeTypes` | `image/jpeg`, `image/png`, `image/gif`, `image/webp` | Default, Development, Test | `appsettings.json`; duplicated in `PhotoServiceTests` in-memory config |
| `FileUpload:MaxFilesPerUpload` | `10` | Default, Development | `appsettings.json` |
| `FileUpload:UploadPath` | `wwwroot/uploads` | Default, Development; overridden in Test | `appsettings.json`; in-memory override in `PhotoServiceTests` |
| `Logging:LogLevel:Default` | `Information` | Default; `Debug` in Development | `appsettings.json`; overridden by `appsettings.Development.json` |
| `Logging:LogLevel:Microsoft.AspNetCore` | `Warning` | Default; `Information` in Development | `appsettings.json`; overridden by `appsettings.Development.json` |
| `Logging:LogLevel:Microsoft.EntityFrameworkCore` | not set in base | Development | `appsettings.Development.json` |
| `DetailedErrors` | `true` | Development | `appsettings.Development.json` |
| `AllowedHosts` | `*` | Default, Development, Production | `appsettings.json` |
| `ASPNETCORE_ENVIRONMENT` | not set in app config | Development, Production | `launchSettings.json`; `infra\main.json` |
| `IsTestEnvironment` | not set in checked-in files | Test when supplied externally | Read by `Program.cs`; no file-backed default found |
| `AzureStorageBlob__Endpoint` | not set in app config | Production deployment path | `infra\main.json` container app env var |
| `AzureStorageBlob__ContainerName` | `photos` in Azure infra | Production deployment path | `infra\main.json` container app env var |

## Startup Parameters & Resource Requirements

| Service | JVM/Runtime Options | Memory | Instance Count |
|---|---|---|---|
| `PhotoAlbum` local launch profile `http` | `applicationUrl=http://localhost:5134`; `ASPNETCORE_ENVIRONMENT=Development` | Not specified | 1 local process |
| `PhotoAlbum` local launch profile `https` | `applicationUrl=https://localhost:7055;http://localhost:5134`; `ASPNETCORE_ENVIRONMENT=Development` | Not specified | 1 local process |
| `PhotoAlbum` Azure Container App | `targetPort=8080`; `ASPNETCORE_ENVIRONMENT=Production`; connection string and blob settings injected as env vars | `1Gi` memory and `0.5` CPU | `minReplicas=1`, `maxReplicas=3` |

## Startup Dependency Chain

1. Azure provisioning creates supporting infrastructure declared in `infra\main.json`, including SQL, storage, registry, and Container Apps resources.
2. `infra\hooks\predeploy.ps1` configures the Container App to pull from the registry using its system-assigned managed identity.
3. `infra\hooks\postprovision.ps1` waits for the `AcrPull` role assignment to propagate before deployment proceeds.
4. On application startup, `Program.cs` creates `wwwroot/uploads` if needed.
5. Still during startup, `Program.cs` runs EF Core migrations unless `IsTestEnvironment` is true.
6. The app starts serving requests only after startup completes; no dedicated health-check endpoint or readiness probe is defined in the application code.

## Secrets & Sensitive Configuration

| Secret Reference | Type | Storage (masked) |
|---|---|---|
| `ConnectionStrings:DefaultConnection` | Database connection string | `appsettings.json` for local development; env var override in Azure deployment path; shown as `[MASKED]` |
| `UserSecretsId` | Developer secret store binding | `28fdd5b1-4b72-4763-98cc-ac5ebb3f280d` in project file; secret values not present in repo |
| `RESOURCE_GROUP` | Deployment environment input | Placeholder only in `.env.example` |

### Secrets Provisioning Workflow

The checked-in application uses a non-secret LocalDB connection string for local execution and declares a `UserSecretsId` so developers can externalize additional secrets without committing them. In the Azure deployment path, `infra\main.json` injects `ConnectionStrings__DefaultConnection` and blob-related settings as Container App environment variables, while `predeploy.ps1` and `postprovision.ps1` configure and validate managed-identity access to the container registry. No Key Vault, Vault, sealed secrets, or other dedicated secret store references were found in the repository.

## Feature Flags

| Flag Name | Default | Controlled By |
|---|---|---|
| None detected | n/a | n/a |

## Framework & Runtime Versions

| Component | Version | Source |
|---|---:|---|
| .NET target framework | `net9.0` | `PhotoAlbum\PhotoAlbum.csproj` |
| ASP.NET Core Web SDK | 9.0 family implied by target framework and SDK | `PhotoAlbum\PhotoAlbum.csproj` |
| Entity Framework Core SQL Server provider | `9.0.9` | `PhotoAlbum\PhotoAlbum.csproj` |
| Entity Framework Core Design | `9.0.9` | `PhotoAlbum\PhotoAlbum.csproj` |
| SixLabors.ImageSharp | `3.1.11` | `PhotoAlbum\PhotoAlbum.csproj` |
| Test SDK | `17.12.0` | `PhotoAlbum.Tests\PhotoAlbum.Tests.csproj` |
| xUnit | `2.9.2` | `PhotoAlbum.Tests\PhotoAlbum.Tests.csproj` |
| ASP.NET Core MVC Testing | `9.0.9` | `PhotoAlbum.Tests\PhotoAlbum.Tests.csproj` |
| EF Core InMemory | `9.0.9` | `PhotoAlbum.Tests\PhotoAlbum.Tests.csproj` |
| Container base image | `mcr.microsoft.com/dotnet/aspnet:9.0` | `Dockerfile` |
| Build image | `mcr.microsoft.com/dotnet/sdk:9.0` | `Dockerfile` |
