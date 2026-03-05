# OneLake Integration Troubleshooting

Error patterns and solutions for OneLake catalog integration issues.

---

## Error Quick Reference

| Error Contains | Section |
|----------------|---------|
| `Consent not granted`, `AADSTS` | [Consent Issues](#consent-issues) |
| `Invalid client`, `AADSTS7000215` | [OAuth Credentials](#oauth-credential-issues) |
| `Access Denied`, `403` | [Permission Issues](#permission-issues) |
| `Catalog not found`, `404` | [Configuration Issues](#configuration-issues) |
| Sync stuck, tables not appearing | [CLD Sync Issues](#cld-sync-issues) |

---

## Consent Issues

### "Consent not granted" or "AADSTS65001"

**Cause**: Snowflake's service principal hasn't been granted consent.

**Solution**:
1. Run `DESC EXTERNAL VOLUME <name>` to get `AZURE_CONSENT_URL`
2. Navigate to the consent URL in browser
3. Click "Accept"
4. Wait 1-2 minutes for propagation

### "Snowflake app not found in Fabric"

**Cause**: Multi-tenant app name search issue.

**Solution**:
1. Get `AZURE_MULTI_TENANT_APP_NAME` from `DESC EXTERNAL VOLUME`
2. Search for the part **BEFORE the underscore** only
3. Example: `7uzzjmsnowflakepacint_1713998620631` → search for `7uzzjmsnowflakepacint`

---

## OAuth Credential Issues

### "Invalid client" or "AADSTS7000215"

**Cause**: Client ID or secret is incorrect.

**Solution**:
1. Verify Client ID matches Azure app registration
2. Check if client secret has expired
3. Create new secret if needed
4. Recreate catalog integration with correct credentials

### "Invalid scope" or "AADSTS70011"

**Cause**: Wrong OAuth scope specified.

**Solution**:
OneLake requires exactly:
```sql
OAUTH_ALLOWED_SCOPES = ('https://storage.azure.com/.default')
```

### "Invalid token URI"

**Cause**: Incorrect token endpoint format.

**Solution**:
Format must be:
```
https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token
```

---

## Permission Issues

### "Access Denied" on External Volume

**Cause**: App registration doesn't have Fabric workspace access.

**Solution**:
1. Go to Microsoft Fabric → Your workspace
2. Manage access → Add people or groups
3. Add your app registration with **Contributor** role

### "Access Denied" on Catalog Operations

**Cause**: Missing `user_impersonation` permission.

**Solution**:
1. Azure Portal → App registration → API permissions
2. Add permission → Microsoft APIs → Azure Storage
3. Select `user_impersonation`

---

## Configuration Issues

### "Catalog not found" (404)

**Cause**: Invalid workspace or data item ID.

**Solution**:
1. Verify Workspace ID from Fabric URL: `/groups/<workspaceID>/`
2. Verify Data Item ID from Fabric URL: `/lakehouses/<dataItemID>`
3. CATALOG_NAME format must be: `<workspaceID>/<dataItemID>`

### "Invalid CATALOG_URI"

**Cause**: Wrong OneLake endpoint.

**Solution**:
OneLake requires exactly:
```sql
CATALOG_URI = 'https://onelake.table.fabric.microsoft.com/iceberg'
```

### "External volume required"

**Cause**: Attempting to use vended credentials.

**Solution**:
OneLake does not support vended credentials. You MUST:
1. Create an external volume
2. Specify it when creating the CLD:
```sql
CREATE DATABASE <name>
  LINKED_CATALOG = (CATALOG = '<integration>')
  EXTERNAL_VOLUME = '<volume>';
```

---

## CLD Sync Issues

### Tables not appearing

**Cause**: Initial sync in progress or namespace filtering.

**Solution**:
1. Check sync status:
```sql
SELECT SYSTEM$CATALOG_LINK_STATUS('<database>');
```
2. Wait for `executionState` to show `RUNNING` or `SUCCEEDED`
3. Check if namespace filtering is blocking:
```sql
SHOW PARAMETERS LIKE 'ALLOWED_NAMESPACES' IN DATABASE <database>;
```

### Sync stuck in "RUNNING" state

**Cause**: Usually normal - sync runs continuously.

**Solution**:
`RUNNING` is the expected state for a healthy CLD. Check if tables exist:
```sql
SHOW ICEBERG TABLES IN DATABASE <database>;
```

### "Table not initialized"

**Cause**: Table discovered but metadata not yet loaded.

**Solution**:
1. Wait a few minutes for auto-refresh
2. Check table status:
```sql
SELECT SYSTEM$AUTO_REFRESH_STATUS('<db>.<schema>.<table>');
```

---

## Diagnostic Commands

```sql
-- External Volume
DESC EXTERNAL VOLUME <name>;
SELECT SYSTEM$VERIFY_EXTERNAL_VOLUME('<name>');

-- Catalog Integration
DESC CATALOG INTEGRATION <name>;
SELECT SYSTEM$VERIFY_CATALOG_INTEGRATION('<name>');
SELECT SYSTEM$LIST_NAMESPACES_FROM_CATALOG('<name>');
SELECT SYSTEM$LIST_ICEBERG_TABLES_FROM_CATALOG('<name>', '<namespace>');

-- CLD
SELECT SYSTEM$CATALOG_LINK_STATUS('<database>');
SHOW SCHEMAS IN DATABASE <database>;
SHOW ICEBERG TABLES IN DATABASE <database>;

-- Table-level
SELECT SYSTEM$AUTO_REFRESH_STATUS('<db>.<schema>.<table>');
```

---

## Common Fixes Checklist

- [ ] Consent URL accepted in browser
- [ ] Snowflake multi-tenant app added to Fabric workspace
- [ ] App registration has user_impersonation permission
- [ ] App registration has Contributor access to workspace
- [ ] Client secret is not expired
- [ ] CATALOG_URI is `https://onelake.table.fabric.microsoft.com/iceberg`
- [ ] OAUTH_ALLOWED_SCOPES is `https://storage.azure.com/.default`
- [ ] Workspace ID and Data Item ID are correct (from URL)
