# OneLake Catalog-Linked Database Skill Development Diary

## Project Overview
- **Goal:** Build a skill for creating catalog-linked databases from OneLake Iceberg data
- **Approach:** Run-through first, then capture as skill using skill-development workflow

## Environment
- **Fabric Workspace:** SnowflakeOneLakeLakehouse
- **Lakehouse Object:** TB_101
- **Schema:** RAW_POS

## Snowflake Object Names
| Object | Name |
|--------|------|
| External Volume | `TB_101_ONELAKE_EXTVOL` |
| Catalog Integration | `TB_101_ONELAKE_CATALOG_INT` |
| Catalog-Linked Database | `TB_101_ONELAKE_CLD` |

---

## Run-through Log

### Task 1: Azure App Registration and OneLake Permissions
**Status:** COMPLETED
**Started:** 2026-03-04
**Completed:** 2026-03-04

#### Steps Completed:
1. [x] Create application registration in Microsoft Entra ID
2. [x] Add user_impersonation permission (Azure Storage - Microsoft APIs)
3. [x] Create client secret
4. [x] Record: Display name, App ID, Tenant ID
5. [x] Grant app Contributor access to Fabric workspace
6. [x] Record: Workspace ID, Data Item ID

#### Values Collected:
- App Name: TB_101_Snowflake_OneLake
- Application (Client) ID: 4756a977-d9ed-4f6f-90f0-0cdcecf43fc3
- Directory (Tenant) ID: 9a2d78cb-73e9-40ee-a558-fc1ac5ef57a7
- Client Secret: (stored securely)
- Workspace ID: 0a2348b9-5a0f-4dcb-ac44-e84fcaf2ed14
- Data Item ID: 7953799c-9dc8-4d80-a375-c645be81f93d

#### Notes/Issues:
- Permission must be from "Microsoft APIs" tab, NOT "APIs my organization uses"
- Must select Azure Storage > user_impersonation
- Client secret cannot be retrieved after creation - must copy immediately

---

### Task 2: Create External Volume
**Status:** COMPLETED
**Started:** 2026-03-04
**Completed:** 2026-03-04

#### Steps Completed:
1. [x] CREATE EXTERNAL VOLUME SQL
2. [x] DESC EXTERNAL VOLUME to get consent URL
3. [x] Navigate to consent URL and Accept
4. [x] Add Snowflake multi-tenant app to Fabric workspace
5. [x] Verify (note: verification returns UNVERIFIED for read-only volumes - expected)

#### SQL Used:
```sql
CREATE OR REPLACE EXTERNAL VOLUME TB_101_ONELAKE_EXTVOL
  STORAGE_LOCATIONS = (
    (
      NAME = 'tb101_onelake'
      STORAGE_PROVIDER = 'AZURE'
      STORAGE_BASE_URL = 'azure://onelake.dfs.fabric.microsoft.com/0a2348b9-5a0f-4dcb-ac44-e84fcaf2ed14/7953799c-9dc8-4d80-a375-c645be81f93d'
      AZURE_TENANT_ID = '9a2d78cb-73e9-40ee-a558-fc1ac5ef57a7'
    )
  )
  ALLOW_WRITES = FALSE;
```

#### Notes/Issues:
- STORAGE_BASE_URL format: `azure://onelake.dfs.fabric.microsoft.com/<workspaceID>/<dataItemID>`
- ALLOW_WRITES = FALSE for OneLake (read-only supported)
- Consent flow has TWO separate grants:
  1. User's app registration -> Fabric workspace (done in Task 1)
  2. Snowflake's multi-tenant app -> Fabric workspace (done after DESC EXTERNAL VOLUME)
- Multi-tenant app name from DESC output: `7uzzjmsnowflakepacint_1713998620631`
- Search for part BEFORE underscore when adding to Fabric workspace

---

### Task 3: Create Catalog Integration
**Status:** COMPLETED
**Started:** 2026-03-04
**Completed:** 2026-03-04

#### Steps Completed:
1. [x] CREATE CATALOG INTEGRATION SQL
2. [x] SYSTEM$VERIFY_CATALOG_INTEGRATION - SUCCESS
3. [x] SYSTEM$LIST_NAMESPACES_FROM_CATALOG - Found: RAW_POS, dbo

#### SQL Used:
```sql
CREATE OR REPLACE CATALOG INTEGRATION TB_101_ONELAKE_CATALOG_INT
  CATALOG_SOURCE = ICEBERG_REST
  TABLE_FORMAT = ICEBERG
  REST_CONFIG = (
    CATALOG_URI = 'https://onelake.table.fabric.microsoft.com/iceberg'
    CATALOG_NAME = '0a2348b9-5a0f-4dcb-ac44-e84fcaf2ed14/7953799c-9dc8-4d80-a375-c645be81f93d'
  )
  REST_AUTHENTICATION = (
    TYPE = OAUTH
    OAUTH_TOKEN_URI = 'https://login.microsoftonline.com/9a2d78cb-73e9-40ee-a558-fc1ac5ef57a7/oauth2/v2.0/token'
    OAUTH_CLIENT_ID = '4756a977-d9ed-4f6f-90f0-0cdcecf43fc3'
    OAUTH_CLIENT_SECRET = '<secret>'
    OAUTH_ALLOWED_SCOPES = ('https://storage.azure.com/.default')
  )
  ENABLED = TRUE;
```

#### Notes/Issues:
- CATALOG_URI is always: `https://onelake.table.fabric.microsoft.com/iceberg`
- CATALOG_NAME format: `<workspaceID>/<dataItemID>`
- OAUTH_TOKEN_URI format: `https://login.microsoftonline.com/<tenantID>/oauth2/v2.0/token`
- OAUTH_ALLOWED_SCOPES must be: `('https://storage.azure.com/.default')`

---

### Task 4: Create Catalog-Linked Database
**Status:** COMPLETED
**Started:** 2026-03-04
**Completed:** 2026-03-04

#### Steps Completed:
1. [x] CREATE DATABASE SQL
2. [x] Check sync status - RUNNING
3. [x] SHOW SCHEMAS - Found: RAW_POS, dbo
4. [x] SHOW ICEBERG TABLES - 6 tables discovered
5. [x] Query sample data - SUCCESS

#### SQL Used:
```sql
CREATE OR REPLACE DATABASE TB_101_ONELAKE_CLD
  LINKED_CATALOG = (
    CATALOG = 'TB_101_ONELAKE_CATALOG_INT'
  )
  EXTERNAL_VOLUME = 'TB_101_ONELAKE_EXTVOL';
```

#### Tables Discovered:
- RAW_POS.country (30 rows)
- RAW_POS.franchise (335 rows)
- RAW_POS.menu (1,593 rows)
- RAW_POS.order_detail (8,213,365 rows)
- RAW_POS.order_header (3,146,593 rows)
- RAW_POS.truck (450 rows)

#### Notes/Issues:
- External volume is REQUIRED for OneLake (no vended credentials support)
- Tables appear within seconds of CLD creation
- Data queryable immediately

---

## Learnings for Skill

### Key Workflow Sequence
1. Azure: Create app registration with user_impersonation permission
2. Fabric: Grant app registration Contributor access to workspace
3. Snowflake: Create external volume
4. Azure: Accept consent URL from external volume
5. Fabric: Grant Snowflake multi-tenant app Contributor access
6. Snowflake: Create catalog integration
7. Snowflake: Create catalog-linked database

### Timing Considerations
- No significant delays observed
- Tables appear within seconds of CLD creation
- Consent propagation was immediate in this case

### Common Errors Encountered
| Error | Cause | Solution |
|-------|-------|----------|
| None encountered | - | - |

### Permission Requirements
- Azure: Application registration create permission
- Fabric: Workspace admin (to grant Contributor access)
- Snowflake: ACCOUNTADMIN (for external volume and catalog integration)

### Portal Workflow Notes
- Fabric workspace ID from URL: `/groups/<workspaceID>/`
- Lakehouse data item ID from URL: `/lakehouses/<dataItemID>`
- Azure Storage permission in "Microsoft APIs" tab, not "APIs my organization uses"
- Snowflake multi-tenant app name: search for part BEFORE underscore

### OneLake-Specific Details
- Read-only only (ALLOW_WRITES = FALSE required)
- CATALOG_URI is fixed: `https://onelake.table.fabric.microsoft.com/iceberg`
- OAUTH_ALLOWED_SCOPES fixed: `https://storage.azure.com/.default`
- External volume always required (no vended credentials)

---

## Phase 2: Skill Creation
**Status:** Ready to begin

Will use `skill-development` skill with "summarize session" workflow to capture this run-through as a reusable skill.
