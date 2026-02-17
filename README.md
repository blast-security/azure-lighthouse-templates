# Azure Onboarding Templates

ARM templates for granting Blast read-only access to your Azure environment.

## RBAC Role Assignment

These templates assign the **Reader** role to the Blast service principal, enabling read-only resource inventory collection.

### Option 1: Management Group Scope (Recommended)

Assigns Reader at the management group level. The role **inherits to all child subscriptions** — one deployment covers everything.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fblast-security%2Fazure-lighthouse-templates%2Fmain%2Frbac-role-assignment%2Fmanagement-group.json)

**Steps:**
1. Click the button above
2. Select the **management group** to grant access on (select root management group for all subscriptions)
3. Select a **region** for deployment metadata (does not affect where the role applies)
4. Fill in `servicePrincipalObjectId` — find it in: Azure Portal > Entra ID > Enterprise Applications > search for "blast-activity-collector" > copy the **Object ID**
5. Click **Review + create** > **Create**

**Via Azure CLI:**
```bash
az deployment mg create \
  --management-group-id <MANAGEMENT_GROUP_ID> \
  --location eastus2 \
  --template-uri https://raw.githubusercontent.com/blast-security/azure-lighthouse-templates/main/rbac-role-assignment/management-group.json \
  --parameters servicePrincipalObjectId=<SP_OBJECT_ID>
```

### Option 2: Single Subscription Scope

Assigns Reader on a single subscription. Use this if you want granular control over which subscriptions to grant access on, or if you don't have management group access.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fblast-security%2Fazure-lighthouse-templates%2Fmain%2Frbac-role-assignment%2Fsubscription.json)

**Steps:**
1. Click the button above
2. Select the **subscription** to grant access on
3. Fill in `servicePrincipalObjectId` — find it in: Azure Portal > Entra ID > Enterprise Applications > search for "blast-activity-collector" > copy the **Object ID**
4. Click **Review + create** > **Create**
5. Repeat for each additional subscription

**Via Azure CLI:**
```bash
az deployment sub create \
  --location eastus2 \
  --template-uri https://raw.githubusercontent.com/blast-security/azure-lighthouse-templates/main/rbac-role-assignment/subscription.json \
  --parameters servicePrincipalObjectId=<SP_OBJECT_ID>
```

## Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `servicePrincipalObjectId` | Yes | — | The Object ID of the Blast service principal in your tenant (Enterprise Applications > Object ID) |
| `roleDefinitionId` | No | `acdd72a7-3385-48ef-bd42-f606fba81ae7` (Reader) | The RBAC role to assign |

## What This Grants

- **Reader** role (read-only) — Blast can view resources but cannot modify, create, or delete anything
- Used for resource inventory collection via the Azure Resource Manager API

## How to Find the Service Principal Object ID

1. Go to **Azure Portal** > **Entra ID** > **Enterprise Applications**
2. Search for **"blast-activity-collector"**
3. Copy the **Object ID** (not the Application ID)

> **Note:** The service principal only exists in your tenant after you complete Step 1 of onboarding (admin consent for Graph API permissions). If you can't find it, contact your Blast representative.

## Revoking Access

- **Management group scope:** Azure Portal > Management groups > select the management group > Access control (IAM) > find "blast-activity-collector" > Remove
- **Subscription scope:** Azure Portal > Subscriptions > select subscription > Access control (IAM) > find "blast-activity-collector" > Remove
