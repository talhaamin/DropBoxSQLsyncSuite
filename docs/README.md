# Dropbox SQL Sync Suite

Dropbox SQL Sync Suite is an MVP Windows 10/11 solution for centrally validating users, pulling the latest SQL Server `.bak` file from Dropbox, and restoring a local SQL Server database with one click.

## Solution layout

```text
DropboxSqlSyncSuite.sln
database/
  01_Create_MasterDb.sql
  02_Seed_MasterDb.sql
  03_Diagnose_Invalid_IdentityHashes.sql
  04_Reset_Invalid_Bootstrap_Admin.sql
docs/
  README.md
src/
  DropboxSqlSyncSuite.Api
  DropboxSqlSyncSuite.Core
  DropboxSqlSyncSuite.Desktop
  DropboxSqlSyncSuite.Infrastructure
  DropboxSqlSyncSuite.Shared
```

## Prerequisites

- Windows 10 or Windows 11
- .NET SDK 9.0.x
- SQL Server Express, Standard, or Developer Edition
- Dropbox app with API access and a dedicated backup folder
- Visual Studio 2022 or newer with `.NET desktop development` and `ASP.NET and web development`

## Current architecture

The current product flow is:

1. The desktop app authenticates against the API.
2. The API owns Dropbox OAuth credentials and the centrally stored Dropbox refresh token.
3. During sync, the desktop asks the API to stage the newest Dropbox backup.
4. The desktop downloads that staged file from the API.
5. The desktop restores the local SQL Server database and writes sync audit history back to the API.

This means Dropbox credentials are not required on user desktops.

## Database setup

1. Create the master database by running [`database/01_Create_MasterDb.sql`](../database/01_Create_MasterDb.sql).
2. Seed baseline roles and bootstrap settings by running [`database/02_Seed_MasterDb.sql`](../database/02_Seed_MasterDb.sql).
3. Existing deployments can add backup-upload settings by running [`database/05_Add_BackupUpload_Settings.sql`](../database/05_Add_BackupUpload_Settings.sql).
4. The seed does not create an admin user. Create the first administrator through the bootstrap API flow described below.

Helpful recovery scripts:

- [`database/03_Diagnose_Invalid_IdentityHashes.sql`](../database/03_Diagnose_Invalid_IdentityHashes.sql)
  - identifies suspicious or invalid password hashes in `core.Users`
- [`database/04_Reset_Invalid_Bootstrap_Admin.sql`](../database/04_Reset_Invalid_Bootstrap_Admin.sql)
  - safely removes only the invalid bootstrap admin row and resets bootstrap flags
- [`database/05_Add_BackupUpload_Settings.sql`](../database/05_Add_BackupUpload_Settings.sql)
  - adds the central settings used by the backup and Dropbox upload workflow

## Database tables reference

The master database uses the `core` schema and combines ASP.NET Identity tables with application-specific sync and audit tables.

### Identity and user lifecycle tables

- `core.Roles`
  - Purpose: stores application roles such as `Admin` and `User`.
  - Usage: used by ASP.NET Identity and API authorization attributes to decide which endpoints and desktop features a signed-in user may access.
  - Relationships: referenced by `core.UserRoles` and `core.RoleClaims`.

- `core.Users`
  - Purpose: stores central user accounts, status, profile fields, local DB metadata, and key timestamps.
  - Usage: this is the master user record used for login, sync authorization, deleted/disabled enforcement, last login tracking, and last successful sync tracking.
  - Relationships: parent table for `core.UserRoles`, `core.UserClaims`, `core.UserLogins`, `core.UserTokens`, `core.RefreshTokens`, `core.UserDevices`, `core.SyncHistory`, `core.UserStatusHistory`, and `core.ForceResyncRequests`. It also self-references through `CreatedBy` and `ModifiedBy`.

- `core.UserRoles`
  - Purpose: joins users to roles.
  - Usage: determines whether a user is a normal operator or administrator.
  - Relationships: many-to-many bridge between `core.Users` and `core.Roles`.

- `core.UserClaims`
  - Purpose: stores per-user claims when needed by Identity.
  - Usage: included for Identity compatibility and future extension.
  - Relationships: child of `core.Users`.

- `core.UserLogins`
  - Purpose: stores external-login mappings for Identity.
  - Usage: included for compatibility even though the MVP mainly uses email/password login.
  - Relationships: child of `core.Users`.

- `core.UserTokens`
  - Purpose: stores token values managed by Identity providers.
  - Usage: reserved for Identity token workflows.
  - Relationships: child of `core.Users`.

- `core.RoleClaims`
  - Purpose: stores claims attached to roles.
  - Usage: available for richer authorization rules if the product grows beyond simple role checks.
  - Relationships: child of `core.Roles`.

### Session, device, and sync execution tables

- `core.RefreshTokens`
  - Purpose: stores long-lived refresh tokens issued after login.
  - Usage: supports token refresh, token rotation, revocation, and session continuation between desktop runs.
  - Relationships: child of `core.Users`.

- `core.UserDevices`
  - Purpose: stores known desktop installations/devices for each user.
  - Usage: tracks machine identity, local DB mapping, last-seen time, and last sync time so the API can understand where each user is operating.
  - Relationships: child of `core.Users`; optionally referenced by `core.SyncHistory`.

- `core.SyncHistory`
  - Purpose: stores every sync attempt and its outcome.
  - Usage: records start/completion timestamps, backup file metadata, local target info, trigger source, error details, and final state such as `Succeeded`, `Failed`, `BlockedDeleted`, or rollback results.
  - Relationships: child of `core.Users`; optional child of `core.UserDevices`.

- `core.ForceResyncRequests`
  - Purpose: stores explicit force-resync requests created by admins.
  - Usage: lets the system tell a user’s next sync to refresh from the master backup source even if the user would otherwise continue normally.
  - Relationships: child of `core.Users` through `UserId`; also linked to the requesting admin through `RequestedBy`.

### Audit and application configuration tables

- `core.AdminActionsLog`
  - Purpose: stores administrator actions performed through the API and desktop.
  - Usage: audits events such as creating users, disabling users, deleting users, forcing resync, and updating operational settings.
  - Relationships: linked to `core.Users` twice: once for the acting admin (`AdminUserId`) and optionally once for the target user (`TargetUserId`).

- `core.AppSettings`
  - Purpose: stores centrally managed operational settings.
  - Usage: holds values such as Dropbox backup folder, local backup retention count, overwrite behavior, default local DB name, and bootstrap-admin flags.
  - Relationships: optionally linked to `core.Users` through `ModifiedBy` so settings changes can be attributed to a user.

- `core.UserStatusHistory`
  - Purpose: stores status transitions for users.
  - Usage: keeps a historical trail when an account changes between `Active`, `Disabled`, and `Deleted`.
  - Relationships: child of `core.Users` through `UserId`; optionally linked to the user/admin who made the change through `ChangedBy`.

### Relationship summary

- One user can have many refresh tokens, devices, sync-history records, status-history records, and force-resync requests.
- One role can be assigned to many users, and one user can have many roles through `core.UserRoles`.
- One admin user can create many audit records in `core.AdminActionsLog`.
- One user device can appear in many sync-history records.
- `core.AppSettings` acts as the central operational configuration store used by the API and admin settings UI.

## API setup

1. Update [`src/DropboxSqlSyncSuite.Api/appsettings.json`](../src/DropboxSqlSyncSuite.Api/appsettings.json):
   - `ConnectionStrings:MasterDb`
   - `Jwt:SecretKey`
   - `Dropbox:AppKey`
   - `Dropbox:AppSecret`
   - `Dropbox:OAuthRedirectUri`
2. For development, you can generate a JWT signing key in PowerShell:

```powershell
[Convert]::ToBase64String((1..64 | ForEach-Object { [byte](Get-Random -Minimum 0 -Maximum 256) }))
```

3. Paste the generated value into:

```json
"Jwt": {
  "SecretKey": "PASTE_THE_GENERATED_VALUE_HERE"
}
```
4. Restore and build:

```powershell
dotnet restore DropboxSqlSyncSuite.sln
dotnet build DropboxSqlSyncSuite.sln
```

5. Run the API:

```powershell
dotnet run --project src/DropboxSqlSyncSuite.Api/DropboxSqlSyncSuite.Api.csproj
```

## IIS deployment warm-start settings

If you host `DropboxSqlSyncSuite.Api` in IIS, enable the following settings so the API stays warm and the first desktop login does not pay the full cold-start cost.

- Application Pool:
  - `Start Mode` = `AlwaysRunning`
  - `Idle Time-out (minutes)` = `0`
- IIS application / site:
  - `Preload Enabled` = `True`
- Windows feature:
  - install IIS `Application Initialization`

Recommended IIS Manager path:

1. Open `Application Pools`.
2. Select the pool used by `DBSS-API`.
3. Open `Advanced Settings...`.
4. Set `Start Mode` to `AlwaysRunning`.
5. Set `Idle Time-out (minutes)` to `0`.
6. Open the `DBSS-API` application under the target site.
7. Open `Advanced Settings...`.
8. Set `Preload Enabled` to `True`.

These settings keep the ASP.NET Core process warm so login, `/api/users/me`, and other first-hit API calls are much faster after idle periods.

## Desktop setup

1. Update [`src/DropboxSqlSyncSuite.Desktop/appsettings.json`](../src/DropboxSqlSyncSuite.Desktop/appsettings.json):
   - `Api:BaseUrl`
   - `Dropbox:BackupFolderPath`
   - `LocalSql:ServerName`
   - `LocalSql:DatabaseName`
   - `LocalSql:DataFilePath`
   - `LocalSql:LogFilePath`
2. Run the WPF app:

```powershell
dotnet run --project src/DropboxSqlSyncSuite.Desktop/DropboxSqlSyncSuite.Desktop.csproj
```

## Desktop user experience summary

Normal users can:

- sign in
- view dashboard status
- trigger sync
- review sync reports and console logs
- open the built-in Help guide

Administrators can additionally:

- create users
- disable or delete users
- issue force-resync requests
- connect or disconnect Dropbox centrally
- manage settings across central, desktop, API, and protected sections
- review configuration audit history

## In-product configuration management plan

The recommended future design is to let administrators manage all settings in-product, but not as raw JSON editing. Instead, expose every field from both `appsettings.json` files through a structured admin configuration workspace with validation, restart guidance, and secret protection.

### Section split

Use three top-level sections in the admin settings experience:

1. `Operational Settings`
2. `Desktop Environment Settings`
3. `Protected Deployment Settings`

Each field should show:

- source: `API config`, `Desktop config`, or `Central DB`
- whether the value is secret
- whether the value is editable
- whether it applies immediately
- whether it requires desktop restart
- whether it requires API restart
- validation status

### 1. Operational Settings

These are the safest fields for day-to-day administration and are the best candidates for normal in-product editing.

API `appsettings.json` fields:

- `Jwt:AccessTokenMinutes`
- `Jwt:RefreshTokenDays`
- `Dropbox:BackupFolderPath`
- `Dropbox:DownloadDirectory`
- `Dropbox:StagingDirectory`
- `Dropbox:StagedFileRetentionMinutes`
- `LocalSql:ServerName`
- `LocalSql:DatabaseName`
- `LocalSql:LogicalDataFileName`
- `LocalSql:DataFilePath`
- `LocalSql:LogFilePath`
- `LocalSql:WorkingDirectory`
- `LocalSql:CreateSafetyBackup`

Desktop `appsettings.json` fields:

- `Api:TimeoutSeconds`
- `Dropbox:BackupFolderPath`
- `Dropbox:DownloadDirectory`
- `LocalSql:ServerName`
- `LocalSql:DatabaseName`
- `LocalSql:LogicalDataFileName`
- `LocalSql:DataFilePath`
- `LocalSql:LogFilePath`
- `LocalSql:WorkingDirectory`
- `LocalSql:CreateSafetyBackup`

Recommended behavior:

- editable from the main admin settings UI
- strongly typed controls instead of raw JSON
- path validation before save
- range validation for numeric fields

Backup upload settings:

- `BackupUpload:DropboxFolder`: Dropbox folder where locally-created SQL Server backups are uploaded.
- `BackupUpload:FilePrefix`: filename prefix used for generated upload backups.
- `BackupUpload:LocalRetentionCount`: number of local upload-backup `.bak` files retained in the working directory.
- API-side Local SQL keys should stay out of the normal admin settings screen because they do not control desktop restore targets or backup sources.

### 2. Desktop Environment Settings

These values are still admin-manageable, but they are more deployment-sensitive and should sit in an advanced section with warnings.

API `appsettings.json` fields:

- `AllowedHosts`
- `Serilog:MinimumLevel:Default`
- `Serilog:MinimumLevel:Override:Microsoft`
- `Serilog:MinimumLevel:Override:System`

Desktop `appsettings.json` fields:

- `Api:BaseUrl`
- `Serilog:MinimumLevel:Default`

Recommended behavior:

- show in an `Advanced Configuration` area
- warn before save when a value can interrupt connectivity or logging behavior
- mark `Api:BaseUrl` as desktop restart or reconnect sensitive
- mark logging-level changes as requiring app restart for predictable behavior
- mark `AllowedHosts` as API restart sensitive

### 3. Protected Deployment Settings

These fields should still be visible to privileged admins if you want complete in-product coverage, but they need stronger safeguards than the other sections.

API `appsettings.json` fields:

- `ConnectionStrings:MasterDb`
- `Jwt:Issuer`
- `Jwt:Audience`
- `Jwt:SecretKey`
- `Dropbox:AccessToken`
- `Dropbox:RefreshToken`
- `Dropbox:AppKey`
- `Dropbox:AppSecret`
- `Dropbox:OAuthRedirectUri`

Desktop `appsettings.json` fields:

- none currently fall into the same secret category as the API-side values

Recommended behavior:

- visible only to high-privilege administrators
- secrets masked by default with explicit reveal
- save action protected by extra confirmation
- every change written to audit log
- API restart required after save
- connection string and JWT secret changes should warn that current sessions or API startup may be affected immediately after restart

### Complete field inventory

The following list covers every field currently present in both `appsettings.json` files.

API `src/DropboxSqlSyncSuite.Api/appsettings.json`:

- `Serilog:MinimumLevel:Default`
- `Serilog:MinimumLevel:Override:Microsoft`
- `Serilog:MinimumLevel:Override:System`
- `ConnectionStrings:MasterDb`
- `Jwt:Issuer`
- `Jwt:Audience`
- `Jwt:SecretKey`
- `Jwt:AccessTokenMinutes`
- `Jwt:RefreshTokenDays`
- `Dropbox:AccessToken`
- `Dropbox:RefreshToken`
- `Dropbox:AppKey`
- `Dropbox:AppSecret`
- `Dropbox:OAuthRedirectUri`
- `Dropbox:BackupFolderPath`
- `Dropbox:DownloadDirectory`
- `Dropbox:StagingDirectory`
- `Dropbox:StagedFileRetentionMinutes`
- `LocalSql:ServerName`
- `LocalSql:DatabaseName`
- `LocalSql:LogicalDataFileName`
- `LocalSql:DataFilePath`
- `LocalSql:LogFilePath`
- `LocalSql:WorkingDirectory`
- `LocalSql:CreateSafetyBackup`
- `AllowedHosts`

Desktop `src/DropboxSqlSyncSuite.Desktop/appsettings.json`:

- `Serilog:MinimumLevel:Default`
- `Api:BaseUrl`
- `Api:TimeoutSeconds`
- `Dropbox:BackupFolderPath`
- `Dropbox:DownloadDirectory`
- `LocalSql:ServerName`
- `LocalSql:DatabaseName`
- `LocalSql:LogicalDataFileName`
- `LocalSql:DataFilePath`
- `LocalSql:LogFilePath`
- `LocalSql:WorkingDirectory`
- `LocalSql:CreateSafetyBackup`

### Recommended save model

Do not save raw JSON from the UI by default. Instead:

- read both config files into typed DTOs
- present them as grouped forms
- validate each field before save
- write a timestamped backup of the original file
- save only the changed section
- record who changed what in `AdminActionsLog`
- clearly report whether the change affects:
  - current session only
  - next sync run
  - desktop restart
  - API restart

### Recommended role and safety model

If you implement full in-product editing for all fields, use two admin safety levels:

- `Admin`
  - may edit operational settings and desktop environment settings
- `Deployment Admin` or equivalent elevated permission
  - may edit protected deployment settings such as connection strings, JWT secrets, Dropbox app credentials, and redirect URIs

### Recommended implementation phases

Phase 1:

- keep the current central operational settings
- add desktop environment editing for:
  - `Api:BaseUrl`
  - `Api:TimeoutSeconds`
  - local SQL fields
  - desktop Dropbox folder and download directory

Phase 2:

- add API operational and advanced fields:
  - token durations
  - logging levels
  - `AllowedHosts`
  - API-side Dropbox staging directories
- keep API-side Local SQL fields hidden unless a real server-side SQL workflow starts using them

Phase 3:

- add protected deployment settings with masking, confirmations, restart warnings, and full auditing:
  - `ConnectionStrings:MasterDb`
  - `Jwt:Issuer`
  - `Jwt:Audience`
  - `Jwt:SecretKey`
  - `Dropbox:AppKey`
  - `Dropbox:AppSecret`
  - `Dropbox:OAuthRedirectUri`
  - optionally legacy `Dropbox:AccessToken`
  - centrally managed `Dropbox:RefreshToken`

## Dropbox app configuration

1. Create a Dropbox app with access scoped to the backup folder.
2. In the Dropbox App Console, enable at least:
   - `files.metadata.read`
   - `files.content.read`
   - `files.content.write`
3. Register the exact redirect URI used by the API, for example:
   - `https://localhost:7280/api/settings/dropbox/oauth/callback`
   - or your deployed IIS/API URL equivalent
4. Put these values in [`src/DropboxSqlSyncSuite.Api/appsettings.json`](../src/DropboxSqlSyncSuite.Api/appsettings.json) or secure deployment configuration:
   - `Dropbox:AppKey`
   - `Dropbox:AppSecret`
   - `Dropbox:OAuthRedirectUri`
5. Do not put Dropbox tokens in desktop config. The desktop app now uses the API-staging flow instead of calling Dropbox directly.
6. After the API is running, sign in as an admin and complete `Settings -> Connect Dropbox` once.

### Local IIS example values

If your API is hosted locally at:

```text
http://localhost/DBSS-API
```

use these values before connecting Dropbox through the admin settings flow.

Desktop config in [`src/DropboxSqlSyncSuite.Desktop/appsettings.json`](../src/DropboxSqlSyncSuite.Desktop/appsettings.json):

- `Api:BaseUrl = http://localhost/DBSS-API`
- `Dropbox:BackupFolderPath = /sql-backups`
- `LocalSql:ServerName = DESKTOP-1PDQRLS`
- `LocalSql:DatabaseName = DropboxSqlSyncSuiteLocal`

API config in [`src/DropboxSqlSyncSuite.Api/appsettings.json`](../src/DropboxSqlSyncSuite.Api/appsettings.json):

- `Dropbox:AppKey = <your Dropbox app key>`
- `Dropbox:AppSecret = <your Dropbox app secret>`
- `Dropbox:OAuthRedirectUri = http://localhost/DBSS-API/api/settings/dropbox/oauth/callback`
- `Dropbox:BackupFolderPath = /sql-backups`

Register this exact redirect URI in the Dropbox App Console:

```text
http://localhost/DBSS-API/api/settings/dropbox/oauth/callback
```

The refresh token does not need to be pasted manually when using the in-product admin connect flow.

### Dropbox OAuth connection flow

The product now supports an admin-only OAuth flow so the API can store a Dropbox refresh token centrally.

1. Start the API.
2. Sign into the desktop app as an administrator.
3. Open `Settings`.
4. Click `Connect Dropbox`.
5. Complete the Dropbox sign-in/consent flow in the browser.
6. After Dropbox redirects back to the API callback page, return to the desktop app.
7. Click `Refresh Settings`.
8. Confirm that the settings screen shows Dropbox as connected.

The dashboard also shows an admin-only Dropbox health card with:

- `Connected`
- `Needs reconnect`
- latest status summary
- last connected time when available

### Is Dropbox connect one-time?

Yes. In normal deployment this is a one-time admin setup per environment, Dropbox app, and Dropbox account.

You do not need to reconnect:

- for each desktop user
- for each sync
- every time the app starts

You only need to reconnect if:

- an admin clicks `Disconnect Dropbox`
- Dropbox authorization is revoked
- Dropbox scopes or app credentials change
- the deployment is moved to a different Dropbox app or account

What is stored centrally:

- `Dropbox:RefreshToken` in `core.AppSettings`
- `Dropbox:AccountId` in `core.AppSettings`
- `Dropbox:ConnectedAtUtc` in `core.AppSettings`

What remains in API deployment configuration:

- `Dropbox:AppKey`
- `Dropbox:AppSecret`
- `Dropbox:OAuthRedirectUri`

### Disconnecting Dropbox

Administrators can disconnect Dropbox from the same `Settings` screen.

- Click `Disconnect Dropbox`
- Confirm the action
- The API clears the centrally stored refresh token and Dropbox connection metadata
- New sync runs will stop until Dropbox is connected again

## How the sync flow works

1. Desktop user signs in or registers through the API.
2. The desktop checks `/api/sync/my-status` before each sync.
3. If the account is `Deleted` or `Disabled`, the app blocks access and removes local DB artifacts where possible.
4. If allowed, the desktop asks the API to stage the newest Dropbox `.bak`, downloads the staged backup from the API, creates a safety backup, restores via SMO, and writes the sync audit back to the API.
5. `LastSuccessfulSyncAt` is only updated on full success.

During sync the desktop also:

- shows a live sync console under the progress bar
- writes a per-run sync log file under `%LocalAppData%\DropboxSqlSyncSuite\SyncLogs`
- keeps the sync screen open after success or failure until the user returns manually
- records console/audit details so newer report entries can open the full console log later

### Normal sync vs forced resync

Normal sync means the user clicks `Sync Latest Backup` and the app performs the regular restore path: status check, newest Dropbox `.bak` selection, download through the API, safety backup, local SQL restore, console log, and sync-history audit.

Forced resync means an admin has marked that user for a required refresh. It does not run automatically in the background. The next time that user clicks sync, the desktop still follows the same restore path, but the run is treated as admin-required and is audited with `TriggeredBy = ForcedResync`. When that forced run succeeds, the pending forced-resync request is cleared.

Real scenarios for Forced Resync:

- an admin uploaded or selected a corrected master `.bak` and wants a specific user to refresh from it
- a user's local SQL database is suspected to be stale, corrupt, or out of step with the master source
- the admin needs a clear audit trail showing that the user's next sync was required by administration

## Database target and source settings

Use this section when an admin needs to control either side of the database movement:

- restore direction: Dropbox `.bak` -> desired local SQL database
- backup direction: specific local SQL database -> new `.bak` uploaded to Dropbox

### Restore a Dropbox `.bak` to a desired database name

Restore sync chooses the destination database in this priority order:

1. `Admin User Management -> Assigned destination database`
   - User-specific restore target.
2. `Settings -> Central application settings -> Default local database name (fallback)`
   - Shared restore fallback when the user has no assigned destination database.
3. `Settings -> Desktop environment settings -> Local SQL database name`
   - Final technical fallback only.

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

## How the backup and upload flow works

The dashboard also includes `Backup and Upload`, which mirrors the restore sync experience for the outbound direction.

1. The desktop checks `/api/sync/my-status` before any backup work starts.
2. If the account is `Deleted` or `Disabled`, the workflow is blocked and an audit entry is recorded.
3. If allowed, the desktop loads the central backup-upload settings from `/api/settings`.
4. The desktop creates a compressed SQL Server `.bak` from the configured local database through SMO.
5. The desktop posts that `.bak` to `/api/sync/upload-backup`.
6. The API uploads the file to Dropbox with the centrally stored refresh token.
7. The desktop records the same style of console log and sync-history audit entry. Backup-upload entries use `TriggeredBy = BackupUpload`.

During backup upload the desktop also:

- shows the same live operation console used by restore sync
- writes a per-run log file under `%LocalAppData%\DropboxSqlSyncSuite\SyncLogs`
- keeps the progress screen open after success or failure
- records Dropbox path, revision, local backup path, file size, source database, and error details in `DetailsJson`

### Changing which database is backed up

Backup and Upload uses the desktop-side local SQL settings from the current machine.

- Change `Settings -> Desktop environment settings -> Local SQL database name [Backup source / restore last fallback]` to back up a different database.
- That field maps to `LocalSql:DatabaseName` in the desktop configuration saved under `%LocalAppData%\DropboxSqlSyncSuite\Config\appsettings.json`.
- Change `Local SQL server name [Backup/Restore]` only if the database is on another SQL Server instance.
- Update `Settings -> Central application settings -> Backup upload file prefix [Backup upload]` if the uploaded `.bak` filename should match the new database name.
- Keep MDF/LDF and logical-file settings aligned with the local deployment because restore and cleanup use those values.

## Bootstrap admin setup

The database seed reserves the first admin email in `core.AppSettings` and enables a one-time bootstrap path.

- Reserved bootstrap email: `admin@dropboxsqlsyncsuite.local`
- Bootstrap flag: `Bootstrap:AdminRequired = true`
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

Bootstrap behavior:

- The call only succeeds when `Bootstrap:AdminRequired` is `true`.
- The call fails if any existing user already has the `Admin` role.
- The email must match `Bootstrap:AdminEmail`.
- On success, ASP.NET Identity generates the real password hash and the API flips:
  - `Bootstrap:AdminRequired` to `false`
  - `Bootstrap:SeededAdminUser` to `true`

## First-run sequence

1. Run [`database/01_Create_MasterDb.sql`](../database/01_Create_MasterDb.sql).
2. Run [`database/02_Seed_MasterDb.sql`](../database/02_Seed_MasterDb.sql).
3. Configure [`src/DropboxSqlSyncSuite.Api/appsettings.json`](../src/DropboxSqlSyncSuite.Api/appsettings.json) with the real SQL, JWT, and Dropbox values.
4. Start the API with `dotnet run --project src/DropboxSqlSyncSuite.Api/DropboxSqlSyncSuite.Api.csproj`.
5. Call `POST /api/auth/bootstrap-admin` once to create the first administrator.
6. Sign into the desktop app using that admin account.
7. Create additional users from the admin UI or API.

## Installer readiness assumptions

- The desktop app stores protected tokens under `%LocalAppData%\DropboxSqlSyncSuite\SecureStore`.
- Logs are written under `%LocalAppData%\DropboxSqlSyncSuite\Logs`.
- Per-sync logs are written under `%LocalAppData%\DropboxSqlSyncSuite\SyncLogs`.
- API logs are written under `%LocalAppData%\DropboxSqlSyncSuite\Api\Logs`.
- Local SQL data and working files default to `C:\ProgramData\DropboxSqlSyncSuite\...`.
- The cleanup logic assumes an attached-file or explicit MDF/LDF deployment. For service-managed database storage, detach-only behavior is the safe fallback.
