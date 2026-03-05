---
name: onelake-catalog-integration-setup
description: "Setup catalog integration for Microsoft OneLake/Fabric Iceberg tables. Triggers: create onelake integration, connect snowflake to fabric, setup onelake catalog, configure fabric iceberg, microsoft fabric snowflake, onelake iceberg, onelake lakehouse snowflake, fabric lakehouse integration, troubleshoot onelake integration."
---

# OneLake Catalog Integration Setup

Setup, verify, or troubleshoot a Snowflake catalog integration for Microsoft OneLake (Fabric) Iceberg tables.

## Intent Routing (FIRST)

**Ask the user**:
```
What would you like to do?

A: Create a new catalog integration for OneLake
   → Setup Snowflake to connect to Microsoft Fabric/OneLake

B: Verify an existing catalog integration
   → Test connection and list namespaces/tables

C: Troubleshoot a catalog integration
   → Diagnose and fix connection issues
```

**Route based on response**:
- **A (Create)** → **Load** `setup/SKILL.md` then follow [Create Workflow](#create-workflow)
- **B (Verify)** → **Load** `verify/SKILL.md` then follow [Verify Workflow](#verify-workflow)
- **C (Troubleshoot)** → **Load** `references/troubleshooting.md` then follow [Troubleshoot Workflow](#troubleshoot-workflow)

---

## Create Workflow

> **⚠️ REQUIRED**: Load `setup/SKILL.md` FIRST before proceeding.

Create external volume, catalog integration, and catalog-linked database for OneLake.

### Step 1: Prerequisites (Azure & Fabric)

Follow `setup/SKILL.md` to collect:
1. Azure app registration (Client ID, Tenant ID, Secret)
2. Fabric workspace access granted to app
3. Workspace ID and Data Item ID from Fabric URL

**⚠️ STOP**: Confirm Azure/Fabric setup before proceeding

### Step 2: Create External Volume

**Load** `create/SKILL.md` → External Volume section

1. Generate CREATE EXTERNAL VOLUME SQL
2. Execute and get consent URL
3. Guide user through consent flow
4. Grant Snowflake multi-tenant app Fabric access

**⚠️ STOP**: Verify external volume before proceeding

### Step 3: Create Catalog Integration

**Load** `create/SKILL.md` → Catalog Integration section

1. Generate CREATE CATALOG INTEGRATION SQL
2. Verify with SYSTEM$VERIFY_CATALOG_INTEGRATION
3. List namespaces to confirm connectivity

**⚠️ STOP**: Verify integration before proceeding

### Step 4: Create Catalog-Linked Database

**Load** `create/SKILL.md` → CLD section

1. Generate CREATE DATABASE SQL
2. Execute and check sync status
3. Verify tables discovered

### Step 5: Verify

→ Continue to [Verify Workflow](#verify-workflow)

---

## Verify Workflow

> **⚠️ REQUIRED**: Load `verify/SKILL.md` FIRST.

### Step V1: Get Object Names

**Ask**: "What are the names of your OneLake objects?"
- External Volume: `___`
- Catalog Integration: `___`
- Database (CLD): `___`

If unknown:
```sql
SHOW EXTERNAL VOLUMES;
SHOW CATALOG INTEGRATIONS;
SHOW DATABASES;
```

### Step V2: Run Verification

Follow `verify/SKILL.md` verification checks.

### Step V3: Report Results

**If all pass**:
```
✅ OneLake integration verified
- External Volume: Working
- Catalog Integration: Connected
- CLD: Syncing
- Tables: Queryable
```

**If any fail** → Continue to [Troubleshoot Workflow](#troubleshoot-workflow)

---

## Troubleshoot Workflow

> **⚠️ REQUIRED**: Load `references/troubleshooting.md` for error patterns.

### Step T1: Identify Component

Which component has the issue?
- External Volume → Check consent/permissions
- Catalog Integration → Check OAuth credentials
- CLD → Check sync status

### Step T2: Match Error Pattern

Use `references/troubleshooting.md` to diagnose.

**⚠️ STOP**: Present diagnosis before applying fixes.

---

## OneLake-Specific Requirements

| Requirement | Value |
|-------------|-------|
| External Volume | **REQUIRED** (no vended credentials) |
| ALLOW_WRITES | `FALSE` (read-only only) |
| CATALOG_URI | `https://onelake.table.fabric.microsoft.com/iceberg` |
| OAUTH_ALLOWED_SCOPES | `https://storage.azure.com/.default` |
| Storage URL Format | `azure://onelake.dfs.fabric.microsoft.com/<workspaceID>/<dataItemID>` |

---

## Quick Reference

**External Volume**:
```sql
CREATE OR REPLACE EXTERNAL VOLUME <name>
  STORAGE_LOCATIONS = ((
    NAME = '<location_name>'
    STORAGE_PROVIDER = 'AZURE'
    STORAGE_BASE_URL = 'azure://onelake.dfs.fabric.microsoft.com/<workspaceID>/<dataItemID>'
    AZURE_TENANT_ID = '<tenantID>'
  ))
  ALLOW_WRITES = FALSE;
```

**Catalog Integration**:
```sql
CREATE OR REPLACE CATALOG INTEGRATION <name>
  CATALOG_SOURCE = ICEBERG_REST
  TABLE_FORMAT = ICEBERG
  REST_CONFIG = (
    CATALOG_URI = 'https://onelake.table.fabric.microsoft.com/iceberg'
    CATALOG_NAME = '<workspaceID>/<dataItemID>'
  )
  REST_AUTHENTICATION = (
    TYPE = OAUTH
    OAUTH_TOKEN_URI = 'https://login.microsoftonline.com/<tenantID>/oauth2/v2.0/token'
    OAUTH_CLIENT_ID = '<clientID>'
    OAUTH_CLIENT_SECRET = '<clientSecret>'
    OAUTH_ALLOWED_SCOPES = ('https://storage.azure.com/.default')
  )
  ENABLED = TRUE;
```

**Catalog-Linked Database**:
```sql
CREATE OR REPLACE DATABASE <name>
  LINKED_CATALOG = (
    CATALOG = '<catalog_integration>'
  )
  EXTERNAL_VOLUME = '<external_volume>';
```

**Diagnostic Commands**:
```sql
DESC EXTERNAL VOLUME <name>;
SELECT SYSTEM$VERIFY_EXTERNAL_VOLUME('<name>');
SELECT SYSTEM$VERIFY_CATALOG_INTEGRATION('<name>');
SELECT SYSTEM$LIST_NAMESPACES_FROM_CATALOG('<name>');
SELECT SYSTEM$LIST_ICEBERG_TABLES_FROM_CATALOG('<name>', '<namespace>');
SELECT SYSTEM$CATALOG_LINK_STATUS('<database>');
```

---

## Stopping Points

- ✋ Intent routing: Wait for user selection
- ✋ Step 1: Azure/Fabric setup confirmation
- ✋ Step 2: External volume consent flow
- ✋ Step 3: Catalog integration verification
- ✋ Troubleshoot: Before applying fixes

---

## Output

- External volume connected to OneLake
- Catalog integration with OAuth authentication
- Catalog-linked database with auto-discovered tables
- Queryable Iceberg tables from Fabric

---

## Documentation

- [Configure Catalog Integration for OneLake REST](https://docs.snowflake.com/en/user-guide/tables-iceberg-configure-catalog-integration-rest-onelake)
- [OneLake Table APIs for Iceberg](https://learn.microsoft.com/en-us/fabric/onelake/table-apis/iceberg-table-apis-get-started)
- [Snowflake Iceberg Tables](https://docs.snowflake.com/user-guide/tables-iceberg)
