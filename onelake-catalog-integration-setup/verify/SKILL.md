---
name: onelake-verify
description: "Verify OneLake integration components"
parent_skill: onelake-catalog-integration-setup
---

# Verify OneLake Integration

Verify all components of the OneLake integration are working correctly.

## When to Load

From main skill:
- Step 5: After creating all objects
- Intent B: Direct verification request

---

## Workflow

### Step V1: Get Object Names

**Ask** for object names if not already known:

```
What are the names of your OneLake objects?

- External Volume: _______________
- Catalog Integration: _______________
- Database (CLD): _______________
```

If user doesn't know:
```sql
SHOW EXTERNAL VOLUMES LIKE '%ONELAKE%';
SHOW CATALOG INTEGRATIONS LIKE '%ONELAKE%';
SHOW DATABASES LIKE '%ONELAKE%';
```

---

### Step V2: Verify External Volume

```sql
DESC EXTERNAL VOLUME <EXTERNAL_VOLUME_NAME>;
```

**Check**:
- STORAGE_PROVIDER = AZURE
- STORAGE_BASE_URL contains `onelake.dfs.fabric.microsoft.com`
- AZURE_TENANT_ID is set

**Optional** (may show UNVERIFIED for read-only):
```sql
SELECT SYSTEM$VERIFY_EXTERNAL_VOLUME('<EXTERNAL_VOLUME_NAME>');
```

---

### Step V3: Verify Catalog Integration

```sql
-- Check integration exists and is enabled
SHOW CATALOG INTEGRATIONS LIKE '<CATALOG_INTEGRATION_NAME>';

-- Test connection
SELECT SYSTEM$VERIFY_CATALOG_INTEGRATION('<CATALOG_INTEGRATION_NAME>');
```

**Expected**: `{"success": true, ...}`

---

### Step V4: List Available Data

```sql
-- List namespaces (schemas in Fabric)
SELECT SYSTEM$LIST_NAMESPACES_FROM_CATALOG('<CATALOG_INTEGRATION_NAME>');

-- List tables in a namespace
SELECT SYSTEM$LIST_ICEBERG_TABLES_FROM_CATALOG('<CATALOG_INTEGRATION_NAME>', '<NAMESPACE>');
```

---

### Step V5: Verify CLD

```sql
-- Check sync status
SELECT SYSTEM$CATALOG_LINK_STATUS('<DATABASE_NAME>');

-- List schemas
SHOW SCHEMAS IN DATABASE <DATABASE_NAME>;

-- List tables
SHOW ICEBERG TABLES IN DATABASE <DATABASE_NAME>;
```

---

### Step V6: Test Data Access

```sql
SELECT * FROM <DATABASE_NAME>.<SCHEMA>.<TABLE> LIMIT 5;
```

---

### Step V7: Report Results

**All checks pass**:
```
OneLake Integration Verification Report
═══════════════════════════════════════════════════════════

✅ External Volume: <EXTERNAL_VOLUME_NAME>
   └─ Storage: azure://onelake.dfs.fabric.microsoft.com/...
   └─ Status: Created

✅ Catalog Integration: <CATALOG_INTEGRATION_NAME>
   └─ Source: ICEBERG_REST
   └─ Verification: SUCCESS
   └─ Namespaces: <count> found

✅ Catalog-Linked Database: <DATABASE_NAME>
   └─ Sync Status: RUNNING
   └─ Schemas: <count>
   └─ Tables: <count>
   └─ Data Access: VERIFIED

═══════════════════════════════════════════════════════════
```

**If any check fails** → Load `references/troubleshooting.md`

---

## Quick Diagnostic Commands

```sql
-- All-in-one verification
DESC EXTERNAL VOLUME <vol>;
SELECT SYSTEM$VERIFY_CATALOG_INTEGRATION('<int>');
SELECT SYSTEM$CATALOG_LINK_STATUS('<db>');
SHOW ICEBERG TABLES IN DATABASE <db>;
```

---

## Output

Verification report with status of all OneLake integration components.
