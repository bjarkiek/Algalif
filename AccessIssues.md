# Access Database (.mdb) Integration Challenges

## Overview
Integrating on-premises Microsoft Access .mdb files with Azure Data Factory via ODBC proved significantly more complex than anticipated due to driver compatibility, security restrictions, and ADF architectural constraints.

## Timeline of Issues & Resolutions

### 1. ODBC Driver Not Installed
**Error:** `General error Unable to open registry key Temporary (volatile) Ace DSN`
**Cause:** The Microsoft Access Database Engine was not installed on the self-hosted Integration Runtime (IR) machine.
**Resolution:** Installed the 64-bit Microsoft Access Database Engine on the Assa machine (where the IR runs).

### 2. Driver Bitness Mismatch
**Error:** Same volatile Ace DSN registry error persisted after driver install.
**Cause:** Multiple 32-bit Access ODBC drivers were present alongside the 64-bit one. Needed to confirm the 64-bit driver `Microsoft Access Driver (*.mdb, *.accdb)` was available, matching the 64-bit IR.
**Resolution:** Verified correct 64-bit driver was registered via `Get-OdbcDriver`.

### 3. IR Service Account Cannot Use ODBC (Volatile Registry Key)
**Error:** `Unable to open registry key Temporary (volatile) Ace DSN for process ... Jet`
**Cause:** The IR service (DIAHostService) was running as LocalSystem, which lacks a HKEY_CURRENT_USER registry hive. The Access ODBC driver requires a user profile to create temporary volatile DSN entries.
**Resolution:** Changed the IR service to run under the Administrator account via `services.msc` → Log On tab. The Administrator account has a proper user profile and HKCU hive.

### 4. ADF Blocks Loopback/Self-Referencing Connections
**Error:** `Access to localhost is denied` / `Access to 127.0.0.1 is denied` / `Access to assa is denied`
**Cause:** We needed to stage .mdb files locally on the IR machine (Assa) for ODBC to read them. ADF's security validation blocks the IR from connecting back to itself via FileServer linked services — whether using `localhost`, `127.0.0.1`, or the machine's own hostname. This is by design.
**Resolution:** Abandoned the download-to-local approach entirely. Instead, the pipeline reads .mdb files directly from their original source location on the Embla file server (`\\embla\c$\Alerton\Compass\2.0\ALGAC\ALGAC\TrendlogData`) via UNC path through the ODBC connection. The Assa IR can already reach Embla over the network.

### 5. MSysObjects Permission Denied
**Error:** `Record(s) cannot be read; no read permission on 'MSysObjects'`
**Cause:** The pipeline used a Lookup activity querying `MSysObjects` (Access system table) to discover table names starting with `tblTrendlog`. Access restricts read permissions on system tables by default.
**Resolution:** Eliminated the MSysObjects lookup entirely. Since the table name follows a predictable pattern — filename `Trendlog_0002000_0000000128.mdb` always contains table `tblTrendlog_0002000_0000000128` — the pipeline now derives the table name from the filename using the expression `@concat('tbl', replace(item().name, '.mdb', ''))`.

### 6. Publish Failures
**Error:** `At least one resource deployment operation failed` and 404 on `arm-template-parameters-definition.json`
**Cause:** ADF publish requires an `arm-template-parameters-definition.json` file in the Git repo. This file was missing.
**Resolution:** Created the file with a minimal valid ARM parameters schema.

## Key Learnings
- ADF's ODBC connector with Microsoft Access is not a well-supported path — most documentation assumes SQL Server, Oracle, or other enterprise databases.
- The self-hosted IR's service account needs a full user profile for Access ODBC to work.
- ADF intentionally prevents the IR from connecting to itself, making local file staging impossible without workarounds.
- Where possible, avoid querying Access system tables (MSysObjects) — derive metadata from naming conventions instead.
