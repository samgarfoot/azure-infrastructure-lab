# Azure CLI Commands Reference

Every Azure CLI command used to deploy the Azure Infrastructure Lab, organised by section. All commands were executed from a Mac terminal using Azure CLI 2.86.0.

---

## Prerequisites

```bash
# Install Azure CLI
brew install azure-cli

# Login
az login

# Verify subscription
az account show --output table

# List all resources in resource group
az resource list --resource-group security-lab-rg --output table
```

---

## 1. Virtual Network & Subnets

```bash
# List all VNets
az network vnet list --output table

# List all subnets in VNet
az network vnet subnet list \
  --resource-group security-lab-rg \
  --vnet-name azure-infra-vnet \
  --output table

# Add Microsoft.Storage service endpoint to public-subnet
az network vnet subnet update \
  --resource-group security-lab-rg \
  --vnet-name azure-infra-vnet \
  --name public-subnet \
  --service-endpoints Microsoft.Storage

# Create Application Gateway VNet in uksouth
az network vnet create \
  --resource-group security-lab-rg \
  --name appgw-vnet \
  --location uksouth \
  --address-prefix 10.1.0.0/16 \
  --subnet-name appgw-subnet \
  --subnet-prefix 10.1.0.0/24
```

---

## 2. Public IP Addresses

```bash
# List all public IPs
az network public-ip list --output table

# Create public IP for Application Gateway
az network public-ip create \
  --resource-group security-lab-rg \
  --name appgw-public-ip \
  --sku Standard \
  --allocation-method Static \
  --location uksouth
```

---

## 3. VNet Peering

```bash
# Create peering from appgw-vnet to azure-infra-vnet
az network vnet peering create \
  --resource-group security-lab-rg \
  --name appgw-to-infra \
  --vnet-name appgw-vnet \
  --remote-vnet azure-infra-vnet \
  --allow-vnet-access

# Create peering from azure-infra-vnet to appgw-vnet
az network vnet peering create \
  --resource-group security-lab-rg \
  --name infra-to-appgw \
  --vnet-name azure-infra-vnet \
  --remote-vnet appgw-vnet \
  --allow-vnet-access

# Verify peering status
az network vnet peering list \
  --resource-group security-lab-rg \
  --vnet-name appgw-vnet \
  --output table
```

---

## 4. Application Gateway

```bash
# Create Application Gateway
az network application-gateway create \
  --resource-group security-lab-rg \
  --name infra-app-gateway \
  --location uksouth \
  --vnet-name appgw-vnet \
  --subnet appgw-subnet \
  --public-ip-address appgw-public-ip \
  --sku Standard_v2 \
  --capacity 1 \
  --frontend-port 80 \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --routing-rule-type Basic \
  --servers 10.0.1.5 \
  --priority 100

# Check backend health
az network application-gateway show-backend-health \
  --resource-group security-lab-rg \
  --name infra-app-gateway \
  --query 'backendAddressPools[].backendHttpSettingsCollection[].servers[].[address,health]' \
  --output table

# Stop Application Gateway (saves cost when not in use)
az network application-gateway stop \
  --resource-group security-lab-rg \
  --name infra-app-gateway

# Start Application Gateway
az network application-gateway start \
  --resource-group security-lab-rg \
  --name infra-app-gateway
```

---

## 5. Virtual Machines

```bash
# List all VMs
az vm list --output table

# Get VM resource ID
az vm show \
  --resource-group security-lab-rg \
  --name soc-target-vm \
  --query id \
  --output tsv

# Stop VMs (saves cost when not in use)
az vm stop \
  --resource-group security-lab-rg \
  --name soc-target-vm

az vm stop \
  --resource-group security-lab-rg \
  --name management-vm

# Start VMs
az vm start \
  --resource-group security-lab-rg \
  --name soc-target-vm

az vm start \
  --resource-group security-lab-rg \
  --name management-vm

# Check VM power state
az vm list \
  --resource-group security-lab-rg \
  --show-details \
  --query '[*].[name,powerState]' \
  --output table
```

---

## 6. Container Instances

```bash
# List all containers
az container list --output table

# Show container details including IP
az container show \
  --resource-group security-lab-rg \
  --name public-web-container \
  --query '{name:name, ip:ipAddress.ip, status:provisioningState}' \
  --output table
```

---

## 7. Azure Monitor & Alerting

```bash
# List Log Analytics workspaces
az monitor log-analytics workspace list --output table

# Create action group
az monitor action-group create \
  --resource-group security-lab-rg \
  --name infra-alert-group \
  --short-name infraalert \
  --action email admin <your-email>

# Create CPU metric alert — SOC VM
az monitor metrics alert create \
  --resource-group security-lab-rg \
  --name soc-vm-cpu-alert \
  --scopes /subscriptions/<subscription-id>/resourceGroups/security-lab-rg/providers/Microsoft.Compute/virtualMachines/soc-target-vm \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action infra-alert-group \
  --description "SOC VM CPU above 80 percent" \
  --severity 2

# Create CPU metric alert — management VM
az monitor metrics alert create \
  --resource-group security-lab-rg \
  --name management-vm-cpu-alert \
  --scopes /subscriptions/<subscription-id>/resourceGroups/security-lab-rg/providers/Microsoft.Compute/virtualMachines/management-vm \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action infra-alert-group \
  --description "Management VM CPU above 80 percent" \
  --severity 2

# List all metric alerts
az monitor metrics alert list \
  --resource-group security-lab-rg \
  --output table

# List all action groups
az monitor action-group list \
  --resource-group security-lab-rg \
  --output table
```

---

## 8. RBAC & Identity

```bash
# Get current signed-in user ID
az ad signed-in-user show --query id --output tsv

# Assign Storage Blob Data Contributor at storage scope
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee <user-id> \
  --scope /subscriptions/<subscription-id>/resourceGroups/security-lab-rg/providers/Microsoft.Storage/storageAccounts/infralaborg

# List role assignments on storage account
az role assignment list \
  --scope /subscriptions/<subscription-id>/resourceGroups/security-lab-rg/providers/Microsoft.Storage/storageAccounts/infralaborg \
  --output table

# List all role assignments in resource group
az role assignment list \
  --resource-group security-lab-rg \
  --output table
```

---

## 9. Storage Accounts

```bash
# Create storage account
az storage account create \
  --resource-group security-lab-rg \
  --name infralaborg \
  --location northeurope \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot \
  --allow-blob-public-access false \
  --min-tls-version TLS1_2

# List storage accounts
az storage account list --output table

# Create blob containers
az storage container create \
  --account-name infralaborg \
  --name hot-data \
  --auth-mode login

az storage container create \
  --account-name infralaborg \
  --name cool-data \
  --auth-mode login

az storage container create \
  --account-name infralaborg \
  --name archive-data \
  --auth-mode login

# List containers
az storage container list \
  --account-name infralaborg \
  --auth-mode login \
  --output table

# Upload blob
az storage blob upload \
  --account-name infralaborg \
  --container-name hot-data \
  --name testfile.txt \
  --file testfile.txt \
  --auth-mode login

# Change blob access tier
az storage blob set-tier \
  --account-name infralaborg \
  --container-name hot-data \
  --name testfile.txt \
  --tier Cool \
  --auth-mode login

# Verify blob tier
az storage blob show \
  --account-name infralaborg \
  --container-name hot-data \
  --name testfile.txt \
  --auth-mode login \
  --query "{name:name, tier:properties.blobTier, size:properties.contentLength}" \
  --output table

# Generate SAS token (read-only, 24 hour expiry)
az storage blob generate-sas \
  --account-name infralaborg \
  --container-name hot-data \
  --name testfile.txt \
  --permissions r \
  --expiry 2026-05-26T00:00:00Z \
  --auth-mode login \
  --as-user \
  --full-uri \
  --output tsv

# Apply lifecycle management policy
az storage account management-policy create \
  --account-name infralaborg \
  --resource-group security-lab-rg \
  --policy @lifecycle-policy.json

# Set storage firewall default action to Deny
az storage account update \
  --resource-group security-lab-rg \
  --name infralaborg \
  --default-action Deny \
  --bypass AzureServices

# Add VNet rule to storage firewall
az storage account network-rule add \
  --resource-group security-lab-rg \
  --account-name infralaborg \
  --vnet-name azure-infra-vnet \
  --subnet public-subnet

# List storage firewall rules
az storage account network-rule list \
  --resource-group security-lab-rg \
  --account-name infralaborg \
  --output table
```

---

## 10. Cost Management

```bash
# Stop all VMs
az vm stop --resource-group security-lab-rg --name soc-target-vm
az vm stop --resource-group security-lab-rg --name management-vm

# Stop Application Gateway
az network application-gateway stop \
  --resource-group security-lab-rg \
  --name infra-app-gateway

# Check what is currently running
az vm list \
  --resource-group security-lab-rg \
  --show-details \
  --query '[*].[name,powerState]' \
  --output table

az network application-gateway show \
  --resource-group security-lab-rg \
  --name infra-app-gateway \
  --query '{name:name, state:operationalState}' \
  --output table

# Check public IPs in use
az network public-ip list \
  --resource-group security-lab-rg \
  --output table
```

---

## 11. Useful Diagnostic Commands

```bash
# Show full resource group overview
az resource list \
  --resource-group security-lab-rg \
  --output table

# Check VNet peering status
az network vnet peering list \
  --resource-group security-lab-rg \
  --vnet-name azure-infra-vnet \
  --output table

# Show NSG rules for a subnet
az network nsg rule list \
  --resource-group security-lab-rg \
  --nsg-name public-nsg \
  --output table

# Show storage account network rules
az storage account show \
  --resource-group security-lab-rg \
  --name infralaborg \
  --query networkRuleSet \
  --output json

# Check Application Gateway backend health
az network application-gateway show-backend-health \
  --resource-group security-lab-rg \
  --name infra-app-gateway \
  --query 'backendAddressPools[].backendHttpSettingsCollection[].servers[].[address,health]' \
  --output table
```
