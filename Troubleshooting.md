# Troubleshooting

Real issues encountered during the deployment of the Azure Infrastructure Lab — 
including the error, the cause, and the resolution. This document reflects the 
actual build process rather than an idealised step-by-step guide.

---

## 1. Public IP Address Limit Reached

**When:** Deploying Application Gateway public IP in North Europe

**Error:**
(PublicIPCountLimitReached) Cannot create more than 3 public IP addresses
for this subscription in this region.
Code: PublicIPCountLimitReached

**Cause:**
Azure free tier subscriptions are limited to 3 public IP addresses per region. 
North Europe already had three allocated:
- `bastion-public-ip` — Azure Bastion
- `nat-gateway-pip` — NAT Gateway
- `windows-target-vm-ip` — SOC target VM

**What was tried first:**
Attempted to request a quota increase via Azure Portal → Subscriptions → 
Usage + Quotas. The option did not appear in the portal for this subscription type.

**Resolution:**
Deployed the Application Gateway public IP in UK South instead of North Europe — 
a different region has a separate public IP quota:

```bash
az network public-ip create \
  --resource-group security-lab-rg \
  --name appgw-public-ip \
  --sku Standard \
  --allocation-method Static \
  --location uksouth
```

**Side effect:**
This meant the Application Gateway itself had to be deployed in UK South, 
creating a cross-region architecture. The Application Gateway subnet was 
moved to a new VNet (`appgw-vnet`) in UK South, requiring cross-region 
VNet peering to reach the backend container in North Europe.

**Lesson learned:**
Plan IP address allocation before starting deployment in free tier subscriptions. 
In production, request quota increases proactively before hitting limits. The 
cross-region architecture that resulted is actually more realistic and 
demonstrates additional skills — VNet peering and multi-region design.

---

## 2. Application Gateway Subnet in Wrong Region

**When:** Attempting to create Application Gateway after deploying public IP in UK South

**Error:**
The Application Gateway could not be created using `appgw-subnet` inside 
`azure-infra-vnet` in North Europe because the public IP was in UK South. 
Azure requires the Application Gateway and its public IP to be in the same region.

**Cause:**
`appgw-subnet` was initially created inside `azure-infra-vnet` (North Europe). 
After the public IP was moved to UK South the subnet was in the wrong region 
for the gateway.

**Resolution:**
Deleted the subnet from `azure-infra-vnet` and created a new dedicated VNet 
in UK South with its own subnet:

```bash
# Delete subnet from wrong region VNet
az network vnet subnet delete \
  --resource-group security-lab-rg \
  --vnet-name azure-infra-vnet \
  --name appgw-subnet

# Create new VNet in correct region
az network vnet create \
  --resource-group security-lab-rg \
  --name appgw-vnet \
  --location uksouth \
  --address-prefix 10.1.0.0/16 \
  --subnet-name appgw-subnet \
  --subnet-prefix 10.1.0.0/24
```

**Lesson learned:**
Application Gateway, its subnet, and its public IP must all be in the same 
region. When deploying cross-region architectures plan the region for each 
component before creating any resources.

---

## 3. VNet Peering — Backend Health Empty After Peering

**When:** Checking Application Gateway backend health after deployment

**Error:**
```bash
az network application-gateway show-backend-health \
  --resource-group security-lab-rg \
  --name infra-app-gateway \
  --output table
# Returns empty — no output
```

**Cause:**
The Application Gateway in UK South could not reach the backend container 
at `10.0.1.5` in North Europe because the two VNets had no network path 
between them. Without peering, VNets are completely isolated even within 
the same subscription.

**Resolution:**
Configured bidirectional VNet peering between `appgw-vnet` and 
`azure-infra-vnet`. Both directions must be created explicitly — Azure 
peering is not automatic in both directions:

```bash
# Peer from Application Gateway VNet to infrastructure VNet
az network vnet peering create \
  --resource-group security-lab-rg \
  --name appgw-to-infra \
  --vnet-name appgw-vnet \
  --remote-vnet azure-infra-vnet \
  --allow-vnet-access

# Peer from infrastructure VNet back to Application Gateway VNet
az network vnet peering create \
  --resource-group security-lab-rg \
  --name infra-to-appgw \
  --vnet-name azure-infra-vnet \
  --remote-vnet appgw-vnet \
  --allow-vnet-access
```

**Lesson learned:**
VNet peering in Azure is non-transitive and requires explicit configuration 
in both directions. Creating a peering from VNet A to VNet B does not 
automatically create the return path. Always configure both sides and verify 
`peeringState: Connected` on both before testing connectivity.

---

## 4. Storage Network Rule — Service Endpoint Not Configured

**When:** Adding VNet rule to storage account firewall

**Error:**
(NetworkAclsValidationFailure) Validation of network acls failure:
SubnetsHaveNoServiceEndpointsConfigured: Subnets public-subnet of virtual
network azure-infra-vnet do not have ServiceEndpoints for Microsoft.Storage
resources configured. Add Microsoft.Storage to subnet's ServiceEndpoints
collection before trying to ACL Microsoft.Storage resources to these subnets.
Code: NetworkAclsValidationFailure

**Cause:**
Azure requires a `Microsoft.Storage` service endpoint to be enabled on a 
subnet before that subnet can be added to a storage account firewall rule. 
The service endpoint creates a direct network path from the subnet to the 
storage service over the Azure backbone — without it the firewall rule 
cannot be applied.

**Resolution:**
Added the `Microsoft.Storage` service endpoint to `public-subnet` first, 
then retried the network rule:

```bash
# Add service endpoint to subnet
az network vnet subnet update \
  --resource-group security-lab-rg \
  --vnet-name azure-infra-vnet \
  --name public-subnet \
  --service-endpoints Microsoft.Storage

# Then add network rule to storage account
az storage account network-rule add \
  --resource-group security-lab-rg \
  --account-name infralaborg \
  --vnet-name azure-infra-vnet \
  --subnet public-subnet
```

**Lesson learned:**
Service endpoints must be configured on the subnet before storage firewall 
rules referencing that subnet can be applied. Service endpoints also improve 
security by routing traffic over the Azure backbone rather than the public 
internet — they are worth enabling regardless of firewall requirements.

---

## 5. Storage Blob Upload — Insufficient Permissions

**When:** Uploading test blob to storage container

**Error:**
You do not have the required permissions needed to perform this operation.
Depending on your operation, you may need to be assigned one of the following roles:
"Storage Blob Data Owner"
"Storage Blob Data Contributor"
"Storage Blob Data Reader"

**Cause:**
Even as the subscription owner, Azure Storage requires explicit data plane 
role assignments to read and write blob data when using `--auth-mode login`. 
Management plane permissions (creating the storage account) do not 
automatically grant data plane permissions (reading and writing blobs). 
This is a deliberate separation of concerns in Azure RBAC.

**Resolution:**
Assigned `Storage Blob Data Contributor` role at the storage account scope:

```bash
# Get current user ID
az ad signed-in-user show --query id --output tsv

# Assign role at storage account scope
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee <user-id> \
  --scope /subscriptions/<subscription-id>/resourceGroups/security-lab-rg/providers/Microsoft.Storage/storageAccounts/infralaborg
```

**Note:** Role assignments can take up to 5 minutes to propagate. If the 
upload still fails immediately after assignment wait 60 seconds and retry.

**Lesson learned:**
Azure RBAC separates management plane and data plane permissions. Being an 
Owner or Contributor at subscription level does not grant access to read or 
write blob data. Data plane roles (`Storage Blob Data Contributor`, 
`Storage Blob Data Reader`) must be assigned explicitly. This is the correct 
security posture — least privilege applied at the data layer.

---

## 6. Scheduled Query Alert — Condition Syntax Error

**When:** Creating log search alert via Azure CLI

**Error:**
line 1:6 extraneous input '>' expecting {'/', '.', '_', '', ':', '%', '-',
',', '|', QUOTE, WHITESPACE, WORD}
line 1:8 mismatched input '5' expecting WHITESPACE
usage error: --condition {avg,min,max,total,count} ["METRIC COLUMN" from]
"QUERY_PLACEHOLDER" {=,!=,>,>=,<,<=} THRESHOLD

**Cause:**
The Azure CLI `scheduled-query` extension has a strict condition syntax 
that differs from the portal. The condition string must reference a named 
column from the query result using exact syntax — `count ColumnName >= 5` 
not `count > 5`.

**What was tried:**
```bash
# First attempt - failed
--condition "count > 5"

# Second attempt - failed  
--condition "count FailedLogins > 5"
```

**Resolution:**
Created the log search alert via Azure Portal instead — the portal provides 
a guided interface for condition configuration that avoids the CLI syntax 
issues:

Azure Portal → Monitor → Alerts → Create Alert Rule

- → Scope: security-lab-workspace
- → Signal: Custom log search
- → Query: SecurityEvent | where EventID == 4625 | summarize FailedLogins=count() by Computer
- → Threshold: Greater than 5
- → Aggregation granularity: 5 minutes
- → Frequency: 5 minutes
- → Action group: infra-alert-group

**Lesson learned:**
Not all Azure CLI extensions are at feature parity with the portal. For 
complex alert conditions the portal provides better validation and error 
feedback. The CLI is preferable for most resources but knowing when to use 
the portal is a practical skill in itself.

---

## 7. Azure CLI Not Installed

**When:** First attempt to run `az` commands in Mac terminal

**Error:**
`zsh: command not found: az`

**Cause:**
Azure CLI is not installed by default on macOS. Unlike AWS CLI which may 
already be present, Azure CLI requires explicit installation.

**Resolution:**
Installed via Homebrew:

```bash
brew install azure-cli
```

Then authenticated:

```bash
az login
az account show --output table
```

**Lesson learned:**
Always verify CLI tool installation before starting a deployment session. 
For a repeatable lab environment consider documenting all required tools 
in `PREREQUISITES.md` — which this project now includes.

---

## Common Commands for Diagnosing Issues

These commands are useful when something is not working as expected:

```bash
# Check what is deployed in the resource group
az resource list --resource-group security-lab-rg --output table

# Check VNet peering status
az network vnet peering list \
  --resource-group security-lab-rg \
  --vnet-name appgw-vnet \
  --output table

# Check Application Gateway backend health
az network application-gateway show-backend-health \
  --resource-group security-lab-rg \
  --name infra-app-gateway \
  --query 'backendAddressPools[].backendHttpSettingsCollection[].servers[].[address,health]' \
  --output table

# Check storage account network rules
az storage account show \
  --resource-group security-lab-rg \
  --name infralaborg \
  --query networkRuleSet \
  --output json

# Check VM power state
az vm list \
  --resource-group security-lab-rg \
  --show-details \
  --query '[*].[name,powerState]' \
  --output table

# Check role assignments on storage account
az role assignment list \
  --scope /subscriptions/<subscription-id>/resourceGroups/security-lab-rg/providers/Microsoft.Storage/storageAccounts/infralaborg \
  --output table

# Check public IP usage
az network public-ip list \
  --resource-group security-lab-rg \
  --output table
```
