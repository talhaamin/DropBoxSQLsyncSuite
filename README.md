# Dropbox SQL Sync Suite

Dropbox SQL Sync Suite is a Windows 10/11 solution for centrally validating users, discovering the latest SQL Server `.bak` in Dropbox through the API, staging that backup securely, restoring a local SQL Server database, and creating/uploading local SQL backups back to Dropbox from the desktop app.

It combines a WPF desktop client, an ASP.NET Core API, a shared master database, in-product administration tools, centralized Dropbox connectivity, sync/audit reporting, and guided restore workflows for distributed desktop environments.

## Highlights

- Central user validation with role-based access for `Admin` and `User`
- API-staged Dropbox restore flow so desktops do not hold Dropbox credentials
- Backup and upload flow for creating local SQL Server `.bak` files and sending them to Dropbox through the API
- Local safety backup before overwrite/restore
- Sync audit trail with success, failure, blocked, and rollback visibility
- Admin tools for user lifecycle management and force-resync workflows
- Admin-only Dropbox connect/disconnect flow with centrally stored refresh token
- In-product settings management for central, desktop, API advanced, and protected deployment settings
- Configuration-audit reporting for settings changes without exposing secrets
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

## Architecture Summary

- Desktop users authenticate against the API and never talk to Dropbox directly.
- The API owns Dropbox OAuth credentials and the centrally stored Dropbox refresh token.
- During sync, the desktop:
  1. checks `/api/sync/my-status`
  2. asks the API to stage the newest Dropbox backup
  3. downloads the staged file from the API
  4. restores the local SQL database
  5. writes sync audit history back to the API
- During backup upload, the desktop:
  1. checks `/api/sync/my-status`
  2. creates a compressed local SQL Server `.bak`
  3. posts it to `/api/sync/upload-backup`
  4. lets the API upload it to Dropbox with the central refresh token
  5. writes the same style of console log and audit history
- Admins manage users, Dropbox connectivity, settings, and configuration audit from the desktop UI.

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
- `Dropbox:AppKey`
- `Dropbox:AppSecret`
- `Dropbox:OAuthRedirectUri`
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

## Dropbox Connection Setup

Dropbox is now connected centrally through the API instead of by storing an access token in the desktop app.

Before connecting Dropbox:

- configure `Dropbox:AppKey`
- configure `Dropbox:AppSecret`
- configure `Dropbox:OAuthRedirectUri`
- register that exact redirect URI in the Dropbox App Console
- enable at least:
  - `files.metadata.read`
  - `files.content.read`
  - `files.content.write`

Admin flow:

1. Start the API.
2. Sign into the desktop app as an admin.
3. Open `Settings`.
4. Click `Connect Dropbox`.
5. Complete Dropbox consent in the browser.
6. Return to the app and refresh settings.

Normal behavior:

- this is usually a one-time setup per environment
- users do not reconnect Dropbox per sync
- desktops do not need Dropbox secrets or refresh tokens

## How Sync Works

1. The desktop user signs in or registers through the API.
2. Before each restore, the desktop checks the user's sync status from the master service.
3. If the account is disabled or deleted, the app blocks access and attempts local cleanup.
4. If allowed, the desktop asks the API to stage the newest Dropbox `.bak`, downloads that staged backup, creates a safety backup, and restores the local SQL Server database.
5. The desktop writes the sync result back to the API audit trail.
6. `LastSuccessfulSyncAt` is updated only after a full successful run.

### Normal Sync vs Forced Resync

Normal sync is what happens when a user clicks `Sync Latest Backup`. The app checks account status, downloads the newest valid Dropbox `.bak` through the API, creates a safety backup, restores the local SQL Server database, and records the run as a normal user-triggered sync.

Forced resync is an admin instruction for a specific user. It does not secretly restore the database in the background. Instead, it marks that user so the next time they run sync, the same restore workflow is treated as a required refresh from the master Dropbox backup source. After a successful forced resync, the pending forced-resync flag is cleared and the audit trail records the run with `TriggeredBy = ForcedResync`.

Use Forced Resync when an admin needs a user to refresh from the master backup after a corrected backup was uploaded, the user's local database is suspected to be stale or damaged, or the admin needs an audit-visible proof that the user refreshed from the central source.

## Database Target and Source Settings

Use this section when you want to restore a Dropbox `.bak` into a specific local database name, or when Backup and Upload should create a `.bak` from a specific SQL database.

### Restore a Dropbox `.bak` to a desired database name

Restore sync chooses the destination database in this priority order:

1. `Admin User Management -> Assigned destination database`
   - Use this for a user-specific restore target.
   - Example: `SalesLocalDb`.
2. `Settings -> Central application settings -> Default local database name (fallback)`
   - Used when the user has no assigned destination database.
   - Example: `DropboxSqlSyncSuiteLocal`.
3. `Settings -> Desktop environment settings -> Local SQL database name`
   - Used only as a final technical fallback if both the user assignment and central fallback are unavailable.

Restore also uses these desktop settings:

- `Local SQL server name [Backup/Restore]`
- `Local SQL logical data file name [Restore]`
- `Local SQL MDF file path [Restore/cleanup]`
- `Local SQL LDF file path [Restore/cleanup]`
- `Local SQL working directory [Backup/Restore]`
- `Create local safety backup before restore [Restore]`

Scenario: one user restores to `SalesLocalDb`:

```text
Admin User Management -> Assigned destination database: SalesLocalDb
Settings -> Desktop environment settings -> Local SQL server name: .\SQLEXPRESS
Settings -> Desktop environment settings -> Local SQL logical data file name: SalesLocalDb
Settings -> Desktop environment settings -> Local SQL MDF file path: C:\ProgramData\DropboxSqlSyncSuite\Data\SalesLocalDb.mdf
Settings -> Desktop environment settings -> Local SQL LDF file path: C:\ProgramData\DropboxSqlSyncSuite\Data\SalesLocalDb_log.ldf
```

Scenario: all users without an assigned DB restore to `CompanyDefaultDb`:

```text
Settings -> Central application settings -> Default local database name (fallback): CompanyDefaultDb
Admin User Management -> Assigned destination database: blank
Settings -> Desktop environment settings -> Local SQL server name: .\SQLEXPRESS
Settings -> Desktop environment settings -> Local SQL MDF file path: C:\ProgramData\DropboxSqlSyncSuite\Data\CompanyDefaultDb.mdf
Settings -> Desktop environment settings -> Local SQL LDF file path: C:\ProgramData\DropboxSqlSyncSuite\Data\CompanyDefaultDb_log.ldf
```

MDF/LDF note for restore: if the target database name changes, keep the MDF and LDF paths in the same folder and use matching file names unless you intentionally want SQL Server files in another location. When a user-specific or central fallback database name is used, the desktop uses the configured MDF/LDF folders and derives file names from the effective database name.

### Take a backup of a specific database

Backup and Upload chooses the source database from the current desktop only:

- `Settings -> Desktop environment settings -> Local SQL database name [Backup source / restore last fallback]`
- `Settings -> Desktop environment settings -> Local SQL server name [Backup/Restore]`
- `Settings -> Desktop environment settings -> Local SQL working directory [Backup/Restore]`

Backup upload naming and destination use central settings:

- `Settings -> Central application settings -> Backup upload file prefix [Backup upload]`
- `Settings -> Central application settings -> Backup upload Dropbox folder [Backup upload]`
- `Settings -> Central application settings -> Backup upload local retention count [Backup upload]`

Scenario: back up `SalesLocalDb` and upload it to `/sql-backups/sales`:

```text
Settings -> Desktop environment settings -> Local SQL server name: .\SQLEXPRESS
Settings -> Desktop environment settings -> Local SQL database name: SalesLocalDb
Settings -> Desktop environment settings -> Local SQL working directory: C:\ProgramData\DropboxSqlSyncSuite\Working
Settings -> Central application settings -> Backup upload file prefix: SalesLocalDb
Settings -> Central application settings -> Backup upload Dropbox folder: /sql-backups/sales
```

MDF/LDF note for backup: Backup and Upload backs up an existing SQL database by server name and database name, so MDF/LDF paths are not used to create the `.bak`. Still keep MDF/LDF settings aligned with the same database if this desktop will also restore, roll back, or run deleted-account cleanup.

Scenario: same desktop backs up and restores `SalesLocalDb`:

```text
Admin User Management -> Assigned destination database: SalesLocalDb
Settings -> Desktop environment settings -> Local SQL database name: SalesLocalDb
Settings -> Desktop environment settings -> Local SQL logical data file name: SalesLocalDb
Settings -> Desktop environment settings -> Local SQL MDF file path: C:\ProgramData\DropboxSqlSyncSuite\Data\SalesLocalDb.mdf
Settings -> Desktop environment settings -> Local SQL LDF file path: C:\ProgramData\DropboxSqlSyncSuite\Data\SalesLocalDb_log.ldf
Settings -> Central application settings -> Backup upload file prefix: SalesLocalDb
```

Restart the desktop app after saving Desktop environment settings.

## How Backup Upload Works

1. The desktop validates account/device status with the same sync-status endpoint.
2. If allowed, it loads backup-upload settings from the master API.
3. It creates a compressed local SQL Server `.bak` in the configured working directory.
4. It uploads the `.bak` to the API, which writes it to the configured Dropbox folder.
5. The run is captured in Sync Reports with `TriggeredBy = BackupUpload` and the saved console log.

To back up a different database, change `Settings -> Desktop environment settings -> Local SQL database name [Backup source / restore last fallback]` on that desktop. If the database is on a different SQL Server instance, also change `Local SQL server name [Backup/Restore]`. To make uploaded file names match the new database, update `Settings -> Central application settings -> Backup upload file prefix [Backup upload]`.

## Settings Workspace

The app now separates settings into clear admin-facing groups:

- `Central operational settings`
  - shared defaults stored in `core.AppSettings`, including restore sync and backup-upload folders, file prefix, and retention
- `Desktop environment settings`
  - per-user desktop config under `%LocalAppData%\DropboxSqlSyncSuite\Config\appsettings.json`; this is where `LocalSql:DatabaseName` controls the Backup and Upload source database and acts only as the final restore fallback
- `API advanced settings`
  - API `appsettings.json` values such as token lifetimes, logging levels, allowed hosts, and staging paths; API-side Local SQL keys are hidden from the screen because restore targets and backup sources are configured through Desktop environment settings and Admin User Management
- `Protected deployment settings`
  - connection string, JWT secret, Dropbox app key/secret, redirect URI, and refresh token

Protected values are masked by default, require stronger confirmation, and should generally be followed by an API restart.

## Reporting and Audit

- `Sync Reports` lets users and admins review sync outcomes, filter history, and open saved console logs for newer runs.
- `Configuration Audit` lets admins review sanitized configuration changes by group, actor, and timestamp without exposing secret values.
- Live sync runs show a console log on the progress screen and also write a per-run log file under `%LocalAppData%\DropboxSqlSyncSuite\SyncLogs`.

## IIS Warm-Start Recommendation

If you host the API in IIS, use these settings to avoid cold-start delays on the first desktop login:

- Application Pool `Start Mode` = `AlwaysRunning`
- Application Pool `Idle Time-out (minutes)` = `0`
- IIS app/site `Preload Enabled` = `True`
- Install IIS `Application Initialization`

## Dropbox App Configuration

- Create a Dropbox app scoped to the backup folder only
- Use the API-side OAuth connect flow instead of storing a manual long-lived access token in the desktop app
- Keep Dropbox credentials out of source control
- Store `.bak` files in the configured folder, for example `/sql-backups`

## Notes

- Protected desktop tokens are stored under `%LocalAppData%\DropboxSqlSyncSuite\SecureStore`
- Desktop logs are written under `%LocalAppData%\DropboxSqlSyncSuite\Logs`
- Per-sync console logs are written under `%LocalAppData%\DropboxSqlSyncSuite\SyncLogs`
- API logs are written under `%LocalAppData%\DropboxSqlSyncSuite\Api\Logs`
- Local SQL working files default to `C:\ProgramData\DropboxSqlSyncSuite\...`
- The cleanup flow assumes attached-file or explicit MDF/LDF deployments

## Additional Documentation

Detailed setup, database table purposes, Dropbox OAuth flow, advanced settings guidance, and first-run deployment notes are available in [`docs/README.md`](docs/README.md).
