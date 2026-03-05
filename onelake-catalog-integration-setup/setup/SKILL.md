---
name: onelake-setup-prerequisites
description: "Gather Azure and Fabric prerequisites for OneLake integration"
parent_skill: onelake-catalog-integration-setup
---

# OneLake Prerequisites Setup

Collect Azure app registration and Microsoft Fabric configuration for OneLake integration.

## When to Load

From main skill Create Workflow → Step 1: Before creating any Snowflake objects.

---

## Workflow

### Step 1.0: Choose Setup Method

**Ask**: "How would you like to set up the Azure app registration?"

```
A: Azure CLI (automated)
   → Faster, scriptable, requires az CLI installed

B: Azure Portal (manual)
   → Step-by-step UI guidance
```

**If A** → Continue to [CLI Setup](#cli-setup)
**If B** → Skip to [Portal Setup](#portal-setup)

---

## CLI Setup

### Step 1.1-CLI: Check Azure CLI

```bash
az --version
az account show
```

If not logged in:
```bash
az login
```

### Step 1.2-CLI: Get App Name

**Ask**: "What name would you like for the app registration?"

Suggested: `Snowflake_OneLake_<LAKEHOUSE>`

### Step 1.3-CLI: Create App Registration (Automated)

**Execute the following commands**:

```bash
# Set app name
APP_NAME="<APP_REGISTRATION_NAME>"

# Create the app registration
az ad app create --display-name "$APP_NAME" --sign-in-audience AzureADMyOrg

# Get the App ID
APP_ID=$(az ad app list --display-name "$APP_NAME" --query "[0].appId" -o tsv)
echo "Application (Client) ID: $APP_ID"

# Get Tenant ID
TENANT_ID=$(az account show --query tenantId -o tsv)
echo "Directory (Tenant) ID: $TENANT_ID"

# Add Azure Storage user_impersonation permission
# Azure Storage API ID: e406a681-f3d4-42a8-90b6-c2b029497af1
# user_impersonation scope ID: 03e0da56-190b-40ad-a80c-ea378c433f7f
az ad app permission add \
  --id $APP_ID \
  --api e406a681-f3d4-42a8-90b6-c2b029497af1 \
  --api-permissions 03e0da56-190b-40ad-a80c-ea378c433f7f=Scope

# Create service principal (required for Fabric access)
az ad sp create --id $APP_ID

# Create client secret (valid for 1 year)
az ad app credential reset \
  --id $APP_ID \
  --append \
  --years 1 \
  --query "{clientId: appId, clientSecret: password, tenantId: tenant}" \
  -o json
```

**Output will contain**:
- `clientId`: Application (Client) ID
- `clientSecret`: Client Secret (save this!)
- `tenantId`: Directory (Tenant) ID

**⚠️ STOP**: Record the output values, especially the client secret.

→ Skip to [Step 1.4: Grant Fabric Workspace Access](#step-14-grant-fabric-workspace-access)

---

## Portal Setup

### Step 1.1-Portal: Azure App Registration

**Ask**: "Do you already have an Azure app registration configured for OneLake?"

**If No** → Guide through creation:

```
Azure Portal Setup:
═══════════════════════════════════════════════════════════

1. Go to Azure Portal → Microsoft Entra ID → App registrations
2. Click "+ New registration"
3. Enter name (e.g., "Snowflake_OneLake_Integration")
4. Click "Register"

From the Overview page, record:
- Application (Client) ID: _______________
- Directory (Tenant) ID: _______________
```

**⚠️ STOP**: Wait for user to provide Client ID and Tenant ID.

---

### Step 1.2-Portal: Add API Permission

```
Add Permission:
═══════════════════════════════════════════════════════════

1. In app registration → API permissions
2. Click "+ Add a permission"
3. Select "Microsoft APIs" tab (NOT "APIs my organization uses")
4. Select "Azure Storage"
5. Check "user_impersonation"
6. Click "Add permissions"
```

**⚠️ STOP**: Confirm permission added.

---

### Step 1.3-Portal: Create Client Secret

```
Create Secret:
═══════════════════════════════════════════════════════════

1. Go to Certificates & secrets
2. Click "+ New client secret"
3. Add description, select expiration
4. Click "Add"
5. IMMEDIATELY copy the secret Value (cannot retrieve later!)

Client Secret: _______________
```

**⚠️ STOP**: Wait for user to provide client secret.

---

## Common Steps (Both Methods)

### Step 1.4: Grant Fabric Workspace Access

**Note**: This step requires the Fabric portal (no CLI available for Fabric RBAC).

```
Fabric Workspace Access:
═══════════════════════════════════════════════════════════

1. Go to Microsoft Fabric (app.fabric.microsoft.com)
2. Open your workspace
3. Click "Manage access" (top right)
4. Click "+ Add people or groups"
5. Search for your app registration name
6. Select "Contributor" role
7. Click "Add"
```

**⚠️ STOP**: Confirm access granted.

---

### Step 1.5: Get Workspace and Data Item IDs

```
From Fabric URL:
═══════════════════════════════════════════════════════════

Open your lakehouse in Fabric. The URL format is:
https://app.fabric.microsoft.com/groups/<workspaceID>/lakehouses/<dataItemID>

Workspace ID: _______________
Data Item ID: _______________
```

**⚠️ STOP**: Wait for user to provide IDs.

---

### Step 1.6: Get Snowflake Object Names

**Ask**: "What names would you like for the Snowflake objects?"

Suggested naming convention: `<LAKEHOUSE>_ONELAKE_<TYPE>`

| Object | Suggested Name |
|--------|----------------|
| External Volume | `<LAKEHOUSE>_ONELAKE_EXTVOL` |
| Catalog Integration | `<LAKEHOUSE>_ONELAKE_CATALOG_INT` |
| Database (CLD) | `<LAKEHOUSE>_ONELAKE_CLD` |

---

### Step 1.7: Prerequisites Summary

**Present checklist**:

```
OneLake Prerequisites Summary
═══════════════════════════════════════════════════════════

Azure App Registration:
✓ Client ID: <CLIENT_ID>
✓ Tenant ID: <TENANT_ID>
✓ Client Secret: (provided)
✓ Permission: user_impersonation for Azure Storage

Microsoft Fabric:
✓ App granted Contributor access to workspace
✓ Workspace ID: <WORKSPACE_ID>
✓ Data Item ID: <DATA_ITEM_ID>

Snowflake Object Names:
✓ External Volume: <EXTERNAL_VOLUME_NAME>
✓ Catalog Integration: <CATALOG_INTEGRATION_NAME>
✓ Database: <DATABASE_NAME>

═══════════════════════════════════════════════════════════
```

**⚠️ MANDATORY STOPPING POINT**: "Does everything look correct? Ready to create the external volume?"

- If yes → **Continue** to `../SKILL.md` Step 2 → **Load** `create/SKILL.md`
- If changes needed → Ask what to update

---

## CLI Quick Reference

**Full automated setup script**:
```bash
#!/bin/bash
APP_NAME="${1:-Snowflake_OneLake_Integration}"

# Create app
az ad app create --display-name "$APP_NAME" --sign-in-audience AzureADMyOrg

# Get IDs
APP_ID=$(az ad app list --display-name "$APP_NAME" --query "[0].appId" -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)

# Add permission
az ad app permission add \
  --id $APP_ID \
  --api e406a681-f3d4-42a8-90b6-c2b029497af1 \
  --api-permissions 03e0da56-190b-40ad-a80c-ea378c433f7f=Scope

# Create service principal
az ad sp create --id $APP_ID

# Create secret and output all values
echo "=== OneLake App Registration ==="
echo "App Name: $APP_NAME"
echo "Client ID: $APP_ID"
echo "Tenant ID: $TENANT_ID"
az ad app credential reset --id $APP_ID --append --years 1 --query "password" -o tsv | xargs -I {} echo "Client Secret: {}"
```

---

## Output

Validated configuration ready for Snowflake object creation:
- Azure credentials
- Fabric IDs
- Snowflake object names

## Next Steps

After user confirms:
→ **Continue** to `../SKILL.md` Step 2: Create External Volume
→ **Load** `create/SKILL.md`
