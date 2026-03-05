# OneLake Catalog Integration Setup Skill

A Cortex Code skill for connecting Snowflake to Microsoft OneLake/Fabric Iceberg tables using catalog-linked databases.

## Overview

This skill guides you through the complete setup process to query Iceberg tables stored in Microsoft OneLake directly from Snowflake, including:

- **External Volume** - Storage access to OneLake
- **Catalog Integration** - REST API connection with OAuth authentication  
- **Catalog-Linked Database** - Automatic table discovery and sync

## Prerequisites

- Azure Portal access (to create app registrations)
- Microsoft Fabric workspace with Iceberg tables
- Snowflake ACCOUNTADMIN role
- (Optional) Azure CLI for automated setup

## Usage

### In Cortex Code

Simply ask to set up OneLake integration. Trigger phrases include:

- "Create OneLake integration"
- "Connect Snowflake to Fabric"
- "Setup OneLake catalog"
- "Microsoft Fabric Snowflake integration"

### Manual Invocation

If the skill is installed locally, reference it by path:
```
/skill /Users/etolotti/CocoTest/CLD_Skill/onelake-catalog-integration-setup
```

## Workflow

1. **Intent Selection** - Create, Verify, or Troubleshoot
2. **Azure Setup** - App registration with user_impersonation permission (CLI or Portal)
3. **Fabric Setup** - Grant app access to workspace
4. **External Volume** - Create and complete consent flow
5. **Catalog Integration** - Create with OAuth credentials
6. **Catalog-Linked Database** - Create and verify table sync

## Key Information

| Item | Value |
|------|-------|
| OneLake CATALOG_URI | `https://onelake.table.fabric.microsoft.com/iceberg` |
| OAuth Scope | `https://storage.azure.com/.default` |
| Storage URL Format | `azure://onelake.dfs.fabric.microsoft.com/<workspaceID>/<dataItemID>` |
| Write Support | Read-only (`ALLOW_WRITES = FALSE`) |

## Skill Structure

```
onelake-catalog-integration-setup/
├── SKILL.md                 # Entry point with intent routing
├── setup/SKILL.md           # Azure/Fabric prerequisites
├── create/SKILL.md          # SQL generation & execution
├── verify/SKILL.md          # Verification workflow
└── references/
    └── troubleshooting.md   # Error patterns & solutions
```

## Documentation

- **Quickstart**: [Use Snowflake with Iceberg tables in OneLake](https://learn.microsoft.com/en-us/fabric/onelake/onelake-iceberg-snowflake)
- **Snowflake Docs**: [Configure catalog integration for OneLake REST](https://docs.snowflake.com/en/user-guide/tables-iceberg-configure-catalog-integration-rest-onelake)
- **OneLake Table APIs**: [Getting started with OneLake table APIs for Iceberg](https://learn.microsoft.com/en-us/fabric/onelake/table-apis/iceberg-table-apis-get-started)

## Development Notes

This skill was developed through a hands-on run-through documented in `DIARY.md`. Key learnings:

- Azure Storage permission must be from "Microsoft APIs" tab, not "APIs my organization uses"
- External volume always required (OneLake doesn't support vended credentials)
- Snowflake multi-tenant app name: search for part BEFORE underscore when adding to Fabric
- Read-only external volumes show UNVERIFIED in verification - this is expected

## License

Internal use - Snowflake
