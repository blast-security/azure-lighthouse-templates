# Blast Azure Integration Templates

Templates for granting Blast read-only access to your Azure environment. Blast uses this access to collect resource inventory, audit logs, and security data — entirely read-only, no changes are ever made to your environment.

---

## How It Works

Blast connects to your Azure tenant using a multi-tenant Azure AD application. Access is controlled through two independent systems that each cover a different API:

| Access | Granted via | Covers |
|--------|------------|--------|
| **Graph API** | Admin consent (Step 1 — done with your Blast representative) | Audit logs, sign-in logs, directory data |
| **Azure Resource Manager** | RBAC role assignment (this repo) | Resource inventory — subscriptions, VMs, storage, networks, etc. |

**This repository handles the RBAC role assignment (Step 2).** Step 1 (admin consent) is completed separately with your Blast representative before using the templates here.

---

## Prerequisites

Before deploying, confirm:

- **Admin consent is complete** — the `blast-activity-collector` enterprise application must already exist in your tenant. If you can't find it in **Entra ID > Enterprise Applications**, complete Step 1 with your Blast representative first.
- **You have the right Azure role** — deploying at management group scope requires **Owner** or **User Access Administrator** on the management group. Subscription scope requires the same on the subscription.

---

## Integration Options

| | Standard Integration | Self-Managed Integration |
|--|---------------------|------------------------|
| **App registration** | Blast's (no credentials to share) | Your own (you share credentials with Blast) |
| **Setup effort** | Deploy one ARM template | Run one script |
| **Credential ownership** | Blast manages | You manage and rotate |
| **Use when** | Default — works for most organizations | Your policy blocks third-party app consent |

---

## Standard Integration

Assigns the **Reader** role to the Blast service principal in your tenant. Blast never receives your credentials.

### Step 1 — Find Your Service Principal Object ID

You need this for both deployment options below.

1. Go to **Azure Portal** → **Entra ID** → **Enterprise Applications**
2. Search for **"blast-activity-collector"**
3. Open the result and copy the **Object ID**

> Use the **Object ID** from the Enterprise Applications page, not the Application ID. They are different values.

If you cannot find "blast-activity-collector", admin consent has not been completed. Contact your Blast representative.

### Step 2 — Deploy the RBAC Template

Choose the scope that matches your environment:

---

#### Option A: Management Group Scope (Recommended)

Assigns Reader at the management group level. The role **inherits to all child subscriptions automatically** — one deployment covers your entire Azure environment.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fblast-security%2Fazure-lighthouse-templates%2Fmain%2Frbac-role-assignment%2Fmanagement-group.json)

**Portal steps:**
1. Click **Deploy to Azure** above
2. Select the **management group** to deploy to (select the root management group to cover all subscriptions)
3. Select any **region** for deployment metadata — this does not affect where the role applies
4. Enter the **Service Principal Object ID** from Step 1
5. Configure any [optional roles](#optional-roles) if applicable
6. Click **Review + create** → **Create**

**Azure CLI:**
```bash
az deployment mg create \
  --management-group-id <MANAGEMENT_GROUP_ID> \
  --location eastus2 \
  --template-uri https://raw.githubusercontent.com/blast-security/azure-lighthouse-templates/main/rbac-role-assignment/management-group.json \
  --parameters servicePrincipalObjectId=<SP_OBJECT_ID>
```

---

#### Option B: Subscription Scope

Assigns Reader on a single subscription. Use this if you don't have management group access, or if you want to grant access to specific subscriptions only. Run once per subscription.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fblast-security%2Fazure-lighthouse-templates%2Fmain%2Frbac-role-assignment%2Fsubscription.json/createUIDefinitionUri/https%3A%2F%2Fraw.githubusercontent.com%2Fblast-security%2Fazure-lighthouse-templates%2Fmain%2Frbac-role-assignment%2Fsubscription-ui.json)

**Portal steps:**
1. Click **Deploy to Azure** above
2. Select the **subscription** to grant access on
3. Enter the **Service Principal Object ID** from Step 1
4. Configure any [optional roles](#optional-roles) if applicable
5. Click **Review + create** → **Create**

**Azure CLI — single subscription:**
```bash
az deployment sub create \
  --subscription <SUB_ID> \
  --location eastus2 \
  --template-uri https://raw.githubusercontent.com/blast-security/azure-lighthouse-templates/main/rbac-role-assignment/subscription.json \
  --parameters servicePrincipalObjectId=<SP_OBJECT_ID>
```

**Azure CLI — multiple subscriptions:**
```bash
for SUB_ID in "<SUB_ID_1>" "<SUB_ID_2>" "<SUB_ID_3>"; do
  az deployment sub create \
    --subscription "$SUB_ID" \
    --location eastus2 \
    --template-uri https://raw.githubusercontent.com/blast-security/azure-lighthouse-templates/main/rbac-role-assignment/subscription.json \
    --parameters servicePrincipalObjectId=<SP_OBJECT_ID>
done
```

---

### Optional Roles

The base Reader role covers resource inventory. The following roles are optional and enable specific data collection features. Enable only what applies to your environment.

| Role | Scope | When to enable |
|------|-------|----------------|
| **Monitoring Reader** | Subscription or management group | You want Blast to collect monitoring metrics and activity logs |
| **Storage Blob Data Reader** | Specific storage account | You forward audit logs to a storage account and want Blast to read them |
| **Azure Event Hubs Data Receiver** | Specific Event Hub namespace | You stream events to Event Hub and want Blast to receive them |
| **Log Analytics Reader** | Specific Log Analytics workspace | You send logs to Log Analytics and want Blast to query them |

In the portal deployment, these are selectable via dropdown — no manual resource ID entry required. Via CLI, provide the full resource ID:

```bash
az deployment sub create \
  --subscription <SUB_ID> \
  --location eastus2 \
  --template-uri https://raw.githubusercontent.com/blast-security/azure-lighthouse-templates/main/rbac-role-assignment/subscription.json \
  --parameters servicePrincipalObjectId=<SP_OBJECT_ID> \
               enableMonitoringReader=true \
               storageAccountResourceId=/subscriptions/<SUB_ID>/resourceGroups/<RG>/providers/Microsoft.Storage/storageAccounts/<ACCOUNT_NAME>
```

---

### What Access Is Granted

All roles assigned are read-only. Blast cannot create, modify, or delete any resources in your environment.

| Role | Built-in Azure Role | Read-only? |
|------|---------------------|-----------|
| Base | [Reader](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#reader) | Yes |
| Optional | [Monitoring Reader](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#monitoring-reader) | Yes |
| Optional | [Storage Blob Data Reader](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#storage-blob-data-reader) | Yes |
| Optional | [Azure Event Hubs Data Receiver](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#azure-event-hubs-data-receiver) | Yes |
| Optional | [Log Analytics Reader](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#log-analytics-reader) | Yes |

You can review the exact permissions of any built-in role in the [Azure portal](https://portal.azure.com/#view/Microsoft_Azure_AD/RoleListBlade) or the Azure RBAC documentation linked above.

---

### Revoking Access

To remove Blast's access at any time:

**Management group scope:**
Azure Portal → Management groups → select the group → **Access control (IAM)** → find "blast-activity-collector" → **Remove**

**Subscription scope:**
Azure Portal → Subscriptions → select the subscription → **Access control (IAM)** → find "blast-activity-collector" → **Remove**

**Revoke everything (Graph API + RBAC):**
Azure Portal → Entra ID → Enterprise Applications → find "blast-activity-collector" → **Delete**

Deleting the enterprise application removes admin consent and all role assignments associated with it.

---

## Self-Managed Integration

For organizations that cannot consent to a third-party application. You create your own app registration in your tenant and share the credentials with Blast.

**Use this if:**
- Your security policy blocks third-party application consent
- You require full ownership and control of all credentials
- You prefer to manage your own secret rotation schedule

### Setup

Run in [Azure Cloud Shell](https://shell.azure.com) or any terminal with Azure CLI:

```bash
curl -sL https://raw.githubusercontent.com/blast-security/azure-lighthouse-templates/main/self-managed/setup.sh | bash
```

Or download and inspect before running:

```bash
curl -sLO https://raw.githubusercontent.com/blast-security/azure-lighthouse-templates/main/self-managed/setup.sh
# review setup.sh
chmod +x setup.sh && ./setup.sh
```

### What the Script Does

1. Creates an app registration (`blast-collector`) in your Entra ID tenant
2. Creates a client secret with a 24-month expiration
3. Grants Graph API permissions — `AuditLog.Read.All` + `Directory.Read.All`
4. Grants admin consent for those permissions
5. Assigns the **Reader** role on your subscription(s) or management group
6. Outputs the credentials to share with Blast

### Required Permissions

| Permission | Needed for |
|------------|-----------|
| **Global Administrator** or **Privileged Role Administrator** | Granting admin consent for Graph API permissions |
| **Owner** or **User Access Administrator** on the target scope | Assigning the Reader RBAC role |

### What You'll Share with Blast

| Value | Description |
|-------|-------------|
| Tenant ID | Your Azure AD tenant ID |
| Client ID | The app registration's Application (client) ID |
| Client Secret | The credential Blast uses to authenticate |

Share these values securely with your Blast representative — not via plain email or chat.

### Revoking Access (Self-Managed)

```bash
# Remove everything — deletes the app registration and all associated permissions
az ad app delete --id <CLIENT_ID>

# Remove RBAC only — keeps Graph API access, removes resource inventory access
az role assignment delete --assignee <CLIENT_ID> --role Reader
```
