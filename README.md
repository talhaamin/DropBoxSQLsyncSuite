# Dropbox SQL Sync Suite

Dropbox SQL Sync Suite is a Windows 10/11 MVP for centrally validating users, pulling the latest SQL Server `.bak` file from Dropbox, and restoring a local SQL Server database with one click.

It combines a WPF desktop client, an ASP.NET Core API, a shared application database, and an admin workflow for managing users, sync permissions, and restore settings.

## Highlights

- Central user validation with role-based access for `Admin` and `User`
- One-click restore flow for the latest Dropbox SQL backup
- Local safety backup before overwrite/restore
- Sync audit trail with success, failure, blocked, and rollback visibility
- Admin tools for user lifecycle management and force-resync workflows
- Bootstrap-first admin creation flow for clean initial deployment

## Screens

### API

![Swagger API](screens/1.PNG)

Swagger UI for authentication, sync, settings, and user-management endpoints.

### Desktop App

| Login | Dashboard |
| --- | --- |
| ![Login screen](screens/2.PNG) | ![Dashboard](screens/3.PNG) |
| Sign in or register with a centrally managed account. | Launch the latest sync, view current status, and access admin tools. |

| Settings | User Management |
| --- | --- |
| ![Settings screen](screens/4.PNG) | ![User management](screens/5.PNG) |
| Manage central app settings such as backup folder, default local DB name, and retention. | Create users, review status, and manage operator/admin access. |

| Sync Reports | Sync Progress |
| --- | --- |
| ![Sync reports](screens/6.PNG) | ![Sync progress](screens/7.PNG) |
| Review outcomes, filter sync history, and inspect execution details. | Follow the restore session with live console output and progress feedback. |

### Restore Log Detail

![Restore log detail](screens/8.PNG)

Detailed restore-stage output for overwrite progress, completion, and audit confirmation.

## Solution Layout

```text
DropboxSqlSyncSuite.sln
database/
  01_Create_MasterDb.sql
  02_Seed_MasterDb.sql
  03_Diagnose_Invalid_IdentityHashes.sql
  04_Reset_Invalid_Bootstrap_Admin.sql
docs/
  README.md
screens/
src/
  DropboxSqlSyncSuite.Api
  DropboxSqlSyncSuite.Core
  DropboxSqlSyncSuite.Desktop
  DropboxSqlSyncSuite.Infrastructure
  DropboxSqlSyncSuite.Shared
```

## Tech Stack

- `.NET 9`
- `WPF` desktop client
- `ASP.NET Core Web API`
- `SQL Server`
- `ASP.NET Core Identity`
- `Dropbox API`
- `Serilog`
- `Swagger / OpenAPI`

## Prerequisites

- Windows 10 or Windows 11
- .NET SDK `9.0.x`
- SQL Server Express, Standard, or Developer Edition
- Dropbox app with API access to a dedicated backup folder
- Visual Studio 2022 or newer with `.NET desktop development` and `ASP.NET and web development`

## Getting Started

### 1. Create the master database

Run these scripts in order:

1. `database/01_Create_MasterDb.sql`
2. `database/02_Seed_MasterDb.sql`

The seed prepares roles and bootstrap settings, but it does not create the first administrator account.

### 2. Configure the API

Update `src/DropboxSqlSyncSuite.Api/appsettings.json`:

- `ConnectionStrings:MasterDb`
- `Jwt:SecretKey`
- `Dropbox:AccessToken`
- `LocalSql` values if you plan to use restore-related settings on the API host

Generate a JWT signing key in PowerShell:

```powershell
[Convert]::ToBase64String((1..64 | ForEach-Object { [byte](Get-Random -Minimum 0 -Maximum 256) }))
```

Build and run:

```powershell
dotnet restore DropboxSqlSyncSuite.sln
dotnet build DropboxSqlSyncSuite.sln
dotnet run --project src/DropboxSqlSyncSuite.Api/DropboxSqlSyncSuite.Api.csproj
```

### 3. Configure the desktop app

Update `src/DropboxSqlSyncSuite.Desktop/appsettings.json`:

- `Api:BaseUrl`
- `Dropbox:AccessToken`
- `Dropbox:BackupFolderPath`
- `LocalSql:ServerName`
- `LocalSql:DatabaseName`
- `LocalSql:DataFilePath`
- `LocalSql:LogFilePath`

Run the desktop client:

```powershell
dotnet run --project src/DropboxSqlSyncSuite.Desktop/DropboxSqlSyncSuite.Desktop.csproj
```

## Bootstrap Admin Setup

The first administrator is created through a one-time bootstrap endpoint.

- Reserved bootstrap email: `admin@dropboxsqlsyncsuite.local`
- Flag: `Bootstrap:AdminRequired = true`
- Endpoint: `POST /api/auth/bootstrap-admin`

Example request:

```json
{
  "email": "admin@dropboxsqlsyncsuite.local",
  "password": "StrongPass123!",
  "fullName": "System Administrator",
  "localDbName": "DropboxSqlSyncSuiteLocal",
  "dropboxFolderPath": "/sql-backups",
  "deviceIdentifier": "HQ-DESKTOP-01",
  "deviceName": "HQ Front Desk"
}
```

On success, the API creates the first admin account, stores a real ASP.NET Identity password hash, and disables the bootstrap requirement.

## How Sync Works

1. The desktop user signs in or registers through the API.
2. Before each restore, the desktop checks the user's sync status from the master service.
3. If the account is disabled or deleted, the app blocks access and attempts local cleanup.
4. If allowed, the desktop locates the newest Dropbox `.bak`, downloads it, creates a safety backup, and restores the local SQL Server database.
5. The desktop writes the sync result back to the API audit trail.
6. `LastSuccessfulSyncAt` is updated only after a full successful run.

## IIS Warm-Start Recommendation

If you host the API in IIS, use these settings to avoid cold-start delays on the first desktop login:

- Application Pool `Start Mode` = `AlwaysRunning`
- Application Pool `Idle Time-out (minutes)` = `0`
- IIS app/site `Preload Enabled` = `True`
- Install IIS `Application Initialization`

## Dropbox App Configuration

- Create a Dropbox app scoped to the backup folder only
- Generate an access token for MVP deployment, or replace with OAuth later
- Keep Dropbox credentials out of source control
- Store `.bak` files in the configured folder, for example `/sql-backups`

## Notes

- Protected desktop tokens are stored under `%LocalAppData%\DropboxSqlSyncSuite\SecureStore`
- Desktop logs are written under `%LocalAppData%\DropboxSqlSyncSuite\Logs`
- Local SQL working files default to `C:\ProgramData\DropboxSqlSyncSuite\...`
- The cleanup flow assumes attached-file or explicit MDF/LDF deployments

## Additional Documentation

The detailed project setup and database notes are available in [`docs/README.md`](docs/README.md).
