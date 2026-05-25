# Prerequisites

Everything required to deploy and reproduce my Azure Infrastructure Lab on a Macbook.

---

## Azure Account

- Microsoft Azure subscription (free tier sufficient for most components)
- Contributor role at subscription scope
- Entra ID access to manage RBAC assignments

---

## Local Tools

| Tool | Purpose | Install |
|---|---|---|
| Azure CLI | Deploy and manage all resources via terminal | `brew install azure-cli` |
| PowerShell | SOC VM configuration via Bastion | Built into Windows / `brew install powershell` on Mac |
| Git | Clone and manage this repository | `brew install git` |

---

## Azure CLI Setup

```bash
# Install
brew install azure-cli

# Login
az login

# Verify correct subscription
az account show --output table

# Set subscription if multiple exist
az account set --subscription "Azure subscription 1"
```

---

## Estimated Costs

Most components fall within the Azure free tier. The following may incur small charges:

| Component | Cost | Notes |
|---|---|---|
| Application Gateway Standard_v2 | ~£0.20/hour | Stop when not in use |
| Virtual Machines (t4g.micro equivalent) | ~£0.01/hour each | Stop when not in use |
| Azure Bastion | ~£0.19/hour | Stop when not in use |
| Storage Account | Pennies/month | Based on data volume |
| Log Search Alerts | Variable | Based on query frequency |

**Recommendation:** Stop VMs and Application Gateway when not actively using the lab to minimise costs.

---

## Stopping Resources When Not In Use

```bash
# Stop VMs
az vm stop --resource-group security-lab-rg --name soc-target-vm
az vm stop --resource-group security-lab-rg --name management-vm

# Stop Application Gateway
az network application-gateway stop \
  --resource-group security-lab-rg \
  --name infra-app-gateway
```

---

## Related Projects

- [Azure SOC Lab](https://github.com/samgarfoot/azure-soc-lab) — Microsoft Sentinel, KQL detection engineering, SOAR automation
- [AWS Infrastructure Lab](https://github.com/samgarfoot/aws-hybrid-infrastructure-lab) - AWS infrastructure project
- [AWS Security Auditor](https://github.com/samgarfoot/aws-security-auditor) — Automated AWS security scanning aligned to CIS and NIST
