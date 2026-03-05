---
name: onelake-create-objects
description: "Create Snowflake objects for OneLake integration"
parent_skill: onelake-catalog-integration-setup
---

# Create OneLake Snowflake Objects

Generate and execute SQL to create external volume, catalog integration, and catalog-linked database.

## When to Load

From main skill Step 2/3/4: After prerequisites collected in `setup/SKILL.md`.

## Prerequisites

Must have from setup phase:
- Client ID, Tenant ID, Client Secret
- Workspace ID, Data Item ID
- Object names (external volume, catalog integration, database)

---

## External Volume

### Step 2.1: Generate External Volume SQL

```sql
CREATE OR REPLACE EXTERNAL VOLUME <EXTERNAL_VOLUME_NAME>
  STORAGE_LOCATIONS = (
    (
      NAME = '<location_name>'
      STORAGE_PROVIDER = 'AZURE'
      STORAGE_BASE_URL = 'azure://onelake.dfs.fabric.microsoft.com/<WORKSPACE_ID>/<DATA_ITEM_ID>'
      AZURE_TENANT_ID = '<TENANT_ID>'
    )
  )
  ALLOW_WRITES = FALSE;
```

**Present generated SQL** with actual values substituted.

**⚠️ STOP**: "Ready to execute this SQL?"

### Step 2.2: Execute and Get Consent URL

After creation, run:
```sql
DESC EXTERNAL VOLUME <EXTERNAL_VOLUME_NAME>;
```

Extract from output:
- `AZURE_CONSENT_URL`
- `AZURE_MULTI_TENANT_APP_NAME`

### Step 2.3: Consent Flow

**Present to user**:
```
External Volume Consent Flow
═══════════════════════════════════════════════════════════

Step A: Accept Consent
1. Open this URL in your browser:
   <AZURE_CONSENT_URL>
2. Click "Accept"

Step B: Grant Snowflake App Access
1. Go to Microsoft Fabric → Your workspace
2. Click "Manage access"
3. Click "+ Add people or groups"
4. Search for: <part_before_underscore_from_AZURE_MULTI_TENANT_APP_NAME>
5. Select "Contributor" role
6. Click "Add"

═══════════════════════════════════════════════════════════
```

**⚠️ STOP**: "Let me know when you've completed both steps."

### Step 2.4: Verify External Volume (Optional)

```sql
SELECT SYSTEM$VERIFY_EXTERNAL_VOLUME('<EXTERNAL_VOLUME_NAME>');
```

**Note**: For read-only volumes (ALLOW_WRITES=FALSE), verification returns UNVERIFIED. This is expected - the volume will work when used.

---

## Catalog Integration

### Step 3.1: Generate Catalog Integration SQL

```sql
CREATE OR REPLACE CATALOG INTEGRATION <CATALOG_INTEGRATION_NAME>
  CATALOG_SOURCE = ICEBERG_REST
  TABLE_FORMAT = ICEBERG
  REST_CONFIG = (
    CATALOG_URI = 'https://onelake.table.fabric.microsoft.com/iceberg'
    CATALOG_NAME = '<WORKSPACE_ID>/<DATA_ITEM_ID>'
  )
  REST_AUTHENTICATION = (
    TYPE = OAUTH
    OAUTH_TOKEN_URI = 'https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token'
    OAUTH_CLIENT_ID = '<CLIENT_ID>'
    OAUTH_CLIENT_SECRET = '<CLIENT_SECRET>'
    OAUTH_ALLOWED_SCOPES = ('https://storage.azure.com/.default')
  )
  ENABLED = TRUE;
```

**Present generated SQL** with actual values substituted.

**⚠️ STOP**: "Ready to execute this SQL?"

### Step 3.2: Verify Catalog Integration

```sql
SELECT SYSTEM$VERIFY_CATALOG_INTEGRATION('<CATALOG_INTEGRATION_NAME>');
```

**Expected**: `{"success": true, ...}`

**If error** → Load `references/troubleshooting.md`

### Step 3.3: List Namespaces

```sql
SELECT SYSTEM$LIST_NAMESPACES_FROM_CATALOG('<CATALOG_INTEGRATION_NAME>');
```

**Present namespaces found**:
```
✅ Catalog Integration verified
- Status: SUCCESS
- Namespaces found: <list>
```

**⚠️ STOP**: "Namespaces look correct? Ready to create the catalog-linked database?"

---

## Catalog-Linked Database

### Step 4.1: Generate CLD SQL

```sql
CREATE OR REPLACE DATABASE <DATABASE_NAME>
  LINKED_CATALOG = (
    CATALOG = '<CATALOG_INTEGRATION_NAME>'
  )
  EXTERNAL_VOLUME = '<EXTERNAL_VOLUME_NAME>';
```

**Present generated SQL**.

**⚠️ STOP**: "Ready to execute this SQL?"

### Step 4.2: Execute and Check Sync

After creation:
```sql
SELECT SYSTEM$CATALOG_LINK_STATUS('<DATABASE_NAME>');
```

### Step 4.3: Verify Tables

```sql
SHOW SCHEMAS IN DATABASE <DATABASE_NAME>;
SHOW ICEBERG TABLES IN DATABASE <DATABASE_NAME>;
```

### Step 4.4: Test Query

```sql
SELECT * FROM <DATABASE_NAME>.<SCHEMA>.<TABLE> LIMIT 5;
```

---

## Output

Successfully created:
- External volume connected to OneLake
- Catalog integration with OAuth authentication
- Catalog-linked database with synced tables

## Next Steps

→ **Continue** to `../SKILL.md` Step 5: Verify
→ **Load** `verify/SKILL.md`
