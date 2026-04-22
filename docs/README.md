# Dropbox SQL Sync Suite

Dropbox SQL Sync Suite is an MVP Windows 10/11 solution for centrally validating users, pulling the latest SQL Server `.bak` file from Dropbox, and restoring a local SQL Server database with one click.

## Solution layout

```text
DropboxSqlSyncSuite.sln
database/
  01_Create_MasterDb.sql
  02_Seed_MasterDb.sql
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

## Database setup

1. Create the master database by running [`database/01_Create_MasterDb.sql`](../database/01_Create_MasterDb.sql).
2. Seed baseline roles and bootstrap settings by running [`database/02_Seed_MasterDb.sql`](../database/02_Seed_MasterDb.sql).
3. The seed does not create an admin user. Create the first administrator through the bootstrap API flow described below.

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
   - `Dropbox:AccessToken`
   - `LocalSql` values if the API host also performs server-side restore tasks later
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
   - `Dropbox:AccessToken`
   - `Dropbox:BackupFolderPath`
   - `LocalSql:ServerName`
   - `LocalSql:DatabaseName`
   - `LocalSql:DataFilePath`
   - `LocalSql:LogFilePath`
2. Run the WPF app:

```powershell
dotnet run --project src/DropboxSqlSyncSuite.Desktop/DropboxSqlSyncSuite.Desktop.csproj
```

## Dropbox app configuration

- Create a Dropbox app with access scoped to the backup folder.
- Generate a long-lived access token for MVP deployment or wire OAuth later.
- Store the token in secure deployment configuration, not in source control.
- Place `.bak` files in the configured folder, for example `/sql-backups`.

## How the sync flow works

1. Desktop user signs in or registers through the API.
2. The desktop checks `/api/sync/my-status` before each sync.
3. If the account is `Deleted` or `Disabled`, the app blocks access and removes local DB artifacts where possible.
4. If allowed, the desktop detects the newest Dropbox `.bak`, downloads it, creates a safety backup, restores via SMO, and writes the sync audit back to the API.
5. `LastSuccessfulSyncAt` is only updated on full success.

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
- Local SQL data and working files default to `C:\ProgramData\DropboxSqlSyncSuite\...`.
- The cleanup logic assumes an attached-file or explicit MDF/LDF deployment. For service-managed database storage, detach-only behavior is the safe fallback.
