# Azure Infrastructure Lab

A hands-on Azure infrastructure lab built to develop practical skills across compute, networking, identity, storage, and governance, aligned to the AZ-104 Microsoft Azure Administrator exam objectives. Built to demonstrate real-world infrastructure engineering capability in hybrid cloud environments relevant to enterprise and UK government deployments.

---

## Lab Architecture

```
                               Internet
                                  │
                    ┌─────────────┴──────────────┐
                    │                            │
         Application Gateway              NAT Gateway
         (51.142.231.76)                 (outbound only)
         uksouth                                 │
         appgw-vnet (10.1.0.0/16)                │
                    │                            │
         VNet Peering (cross-region)             │
                    │                            │
        ┌───────────┴────────────────────────────┤
        │                   │                    │
   Public Subnet       Management Subnet     SOC Subnet
   (10.0.1.0/24)       (10.0.3.0/24)       (10.0.5.0/24)
   public-nsg          management-nsg        soc-nsg
        │                   │                    │
 public-web-container  management-vm       soc-target-vm
 (Nginx - web layer)   (Jump box)       (Windows Server 2022)
 ◄── App Gateway            │                    │
     Backend Pool      Azure Bastion      Microsoft Sentinel
                            │          (Log ingestion via AMA)
                    ┌───────┘                    |
                    │                     infra-alert-group
              Private Subnet            (Email notifications)
              (10.0.2.0/24)
              private-nsg
                    │                       infralaborg
           infra-lab-container           (Storage Account)
           (App layer - isolated)       hot | cool | archive
```

---

## AZ-104 Exam Domains Covered

| Domain | Topics | Status |
|---|---|---|
| Manage Azure Identities & Governance | Entra ID, RBAC, Azure Policy, resource tags, role assignments at resource scope | ✅ Complete |
| Deploy & Manage Azure Compute | Container Instances, Virtual Machines, three-tier architecture | ✅ Complete |
| Implement & Manage Virtual Networking | VNet, subnets, NSGs, Bastion, NAT Gateway, Application Gateway, VNet Peering, Service Endpoints | ✅ Complete |
| Monitor & Maintain Azure Resources | Azure Monitor, metric alerts, log search alerts, action groups, Sentinel log ingestion, Azure Monitor Agent | ✅ Complete |
| Implement & Manage Storage | Storage accounts, blob containers, access tiers, lifecycle management, SAS tokens, storage firewall, service endpoints, RBAC | ✅ Complete |

---

## Components

| Component | Purpose | Subnet | Status |
|---|---|---|---|
| `azure-infra-vnet` | Core virtual network (10.0.0.0/16) — northeurope | — | ✅ Deployed |
| `public-subnet` | Internet-facing tier (10.0.1.0/24) | azure-infra-vnet | ✅ Deployed |
| `private-subnet` | Isolated app tier (10.0.2.0/24) | azure-infra-vnet | ✅ Deployed |
| `management-subnet` | Admin access tier (10.0.3.0/24) | azure-infra-vnet | ✅ Deployed |
| `AzureBastionSubnet` | Bastion dedicated subnet (10.0.4.0/26) | azure-infra-vnet | ✅ Deployed |
| `soc-subnet` | Security monitoring tier (10.0.5.0/24) | azure-infra-vnet | ✅ Deployed |
| `appgw-vnet` | Application Gateway VNet (10.1.0.0/16) — uksouth | — | ✅ Deployed |
| `appgw-subnet` | Application Gateway dedicated subnet (10.1.0.0/24) | appgw-vnet | ✅ Deployed |
| `public-nsg` | Traffic control — public subnet | public-subnet | ✅ Deployed |
| `private-nsg` | Traffic control — private subnet | private-subnet | ✅ Deployed |
| `management-nsg` | Traffic control — management subnet | management-subnet | ✅ Deployed |
| `soc-nsg` | Traffic control — SOC subnet | soc-subnet | ✅ Deployed |
| `infra-nat-gateway` | Outbound internet for public, management, SOC subnets | — | ✅ Deployed |
| `azure-infra-bastion` | Secure browser-based VM access | AzureBastionSubnet | ✅ Deployed |
| `appgw-to-infra` | VNet Peering — appgw-vnet to azure-infra-vnet (cross-region) | — | ✅ Deployed |
| `infra-to-appgw` | VNet Peering — azure-infra-vnet to appgw-vnet (cross-region) | — | ✅ Deployed |
| `infra-app-gateway` | Application Gateway Standard_v2 — public traffic ingress | appgw-subnet | ✅ Deployed |
| `appgw-public-ip` | Static public IP for Application Gateway (51.142.231.76) | — | ✅ Deployed |
| `management-vm` | Jump box — no public IP, Bastion access only | management-subnet | ✅ Deployed |
| `public-web-container` | Nginx web server — public facing tier, App Gateway backend | public-subnet | ✅ Deployed |
| `infra-lab-container` | App layer — isolated, no internet access | private-subnet | ✅ Deployed |
| `soc-target-vm` | Windows Server 2022 — security monitoring target | soc-subnet | ✅ Deployed |
| `security-lab-workspace` | Log Analytics workspace — Sentinel and Azure Monitor | — | ✅ Deployed |
| `infra-alert-group` | Action group — email alerts to admin on trigger | — | ✅ Deployed |
| `soc-vm-cpu-alert` | Metric alert — SOC VM CPU above 80% for 5 minutes | — | ✅ Deployed |
| `management-vm-cpu-alert` | Metric alert — management VM CPU above 80% for 5 minutes | — | ✅ Deployed |
| `sentinel-failed-login-alert` | Log search alert — more than 5 failed logins via KQL (EventID 4625) | — | ✅ Deployed |
| `infralaborg` | Storage account — StorageV2, Standard LRS, TLS 1.2, HTTPS only | — | ✅ Deployed |
| `hot-data` | Blob container — active data, Hot tier | infralaborg | ✅ Deployed |
| `cool-data` | Blob container — infrequent access, Cool tier | infralaborg | ✅ Deployed |
| `archive-data` | Blob container — long term retention, Archive tier | infralaborg | ✅ Deployed |

---

## 1. Virtual Network Architecture

### VNet Design
Deployed a hub-and-spoke style Virtual Network (`azure-infra-vnet`) with a `10.0.0.0/16` address space, providing 65,536 addresses across five segmented subnets. Each subnet serves a distinct architectural purpose mirroring enterprise and UK government Azure deployments.

### Subnet Segmentation
| Subnet | CIDR | Purpose |
|---|---|---|
| public-subnet | 10.0.1.0/24 | Internet-facing resources |
| private-subnet | 10.0.2.0/24 | Isolated backend workloads |
| management-subnet | 10.0.3.0/24 | Administrative jump box |
| AzureBastionSubnet | 10.0.4.0/26 | Azure Bastion (mandatory /26 minimum) |
| soc-subnet | 10.0.5.0/24 | Security monitoring workloads |

### Network Security Groups
Configured NSGs on each subnet implementing least privilege traffic control:

**public-nsg — Allow inbound HTTP/HTTPS from internet:**
```
Priority 100 — Allow TCP 80, 443 inbound from Any
```

**private-nsg — Allow traffic from public subnet only:**
```
Priority 100 — Allow Any inbound from 10.0.1.0/24 only
```

**management-nsg — Allow RDP/SSH from admin IP only:**
```
Priority 100 — Allow TCP 22, 3389 inbound from [admin IP] only
```

**soc-nsg — Internal access only:**
```
No inbound internet rules — internal VNet routing only
```

### NAT Gateway
Deployed `infra-nat-gateway` to provide outbound internet connectivity to public, management, and SOC subnets while maintaining inbound internet isolation. Private subnet remains fully isolated — no inbound or outbound internet access by design.

**Outbound internet enabled:**
- public-subnet ✅
- management-subnet ✅
- soc-subnet ✅

**Fully isolated:**
- private-subnet ✅ — no internet access in either direction

---

## 2. Secure Remote Access — Azure Bastion

Deployed Azure Bastion into the dedicated `AzureBastionSubnet` to provide secure browser-based RDP/SSH access to virtual machines without exposing public IPs or RDP/SSH ports to the internet.

**Access pattern:**
```
Engineer (Mac) → Azure Bastion (browser) → management-vm → soc-target-vm
```

**Security benefits:**
- No public IP required on any VM
- RDP/SSH ports never exposed to internet
- All sessions fully logged and audited
- MFA enforced through Entra ID
- Implements CIS Control 4 — secure configuration of enterprise assets

---

## 3. Compute

### Three-Tier Container Architecture
Deployed a three-tier application architecture using Azure Container Instances — cost efficient alternative to full VMs for workload tiers:

**Public tier — Nginx web server:**
- Container: `public-web-container`
- Image: `mcr.microsoft.com/oss/nginx/nginx:1.9.15-alpine`
- Subnet: `public-subnet` (10.0.1.x)
- Purpose: Internet-facing web layer

**Application tier — isolated backend:**
- Container: `infra-lab-container`
- Image: `mcr.microsoft.com/azuredocs/aci-helloworld`
- Subnet: `private-subnet` (10.0.2.x)
- Purpose: Backend application layer, no internet access

**Management tier — jump box:**
- VM: `management-vm`
- Image: Windows Server 2022 Datacenter Gen 2
- Subnet: `management-subnet` (10.0.3.x)
- No public IP — Bastion access only

### SOC Target VM
- VM: `soc-target-vm`
- Image: Windows Server 2022 Datacenter Gen 2
- Subnet: `soc-subnet` (10.0.5.x)
- No public IP — accessible via management VM jump box through Bastion
- Connected to Microsoft Sentinel via Azure Monitor Agent

---

## 4. Application Gateway & Cross-Region VNet Peering

### Application Gateway
Deployed Azure Application Gateway Standard_v2 in uksouth as a Layer 7 load balancer and traffic ingress point for the lab architecture. The gateway provides a single public endpoint routing HTTP traffic to the Nginx web container in the northeurope VNet backend pool.

**Configuration:**
- SKU: Standard_v2 (Generation 2)
- Frontend: Static public IP 51.142.231.76 on port 80
- Backend pool: public-web-container (10.0.1.5)
- Routing rule: Basic — all traffic forwarded to backend pool
- Protocol: HTTP

**Deployed via Azure CLI:**
```bash
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
```

### Cross-Region VNet Peering
Configured bidirectional VNet peering between `appgw-vnet` (uksouth) and `azure-infra-vnet` (northeurope) to enable the Application Gateway to route traffic to the backend container across regions.

**Peering configuration:**
- `appgw-to-infra`: appgw-vnet → azure-infra-vnet | Connected ✅
- `infra-to-appgw`: azure-infra-vnet → appgw-vnet | Connected ✅
- Both peerings FullyInSync with AllowVirtualNetworkAccess enabled

**Traffic flow:**
- Internet → Application Gateway (51.142.231.76) → VNet Peering (cross-region) → public-web-container Nginx (10.0.1.5) → HTTP 200

---

## 5. Security Monitoring Integration

### Windows Audit Policy Hardening
Configured comprehensive audit logging on `soc-target-vm` via PowerShell:

```powershell
auditpol /set /category:"Logon/Logoff" /success:enable /failure:enable
auditpol /set /category:"Account Management" /success:enable /failure:enable
auditpol /set /category:"Privilege Use" /success:enable /failure:enable
auditpol /set /category:"Policy Change" /success:enable /failure:enable
auditpol /set /category:"System" /success:enable /failure:enable
```

### Command Line Logging
Enabled full command line capture in process creation events via PowerShell registry configuration:

```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" -Name "ProcessCreationIncludeCmdLine_Enabled" -Value 1 -PropertyType DWORD -Force
```

### Sentinel Integration
Connected `soc-target-vm` to Microsoft Sentinel via Windows Security Events via AMA data connector and Data Collection Rule — security events ingested into Log Analytics workspace and queryable via KQL.

---

## 6. Azure Monitor & Alerting

### Action Group
Created `infra-alert-group` to deliver email notifications to the admin on any alert trigger.

```bash
az monitor action-group create \
  --resource-group security-lab-rg \
  --name infra-alert-group \
  --short-name infraalert \
  --action email admin <admin-email>
```

### Metric Alerts
Configured CPU threshold alerts on both VMs — fires when average CPU exceeds 80% over a 5 minute window, evaluated every minute:

| Alert | Target | Threshold | Severity |
|---|---|---|---|
| `soc-vm-cpu-alert` | soc-target-vm | CPU > 80% | 2 - Warning |
| `management-vm-cpu-alert` | management-vm | CPU > 80% | 2 - Warning |

### Log Search Alert — Failed Logins
Configured a KQL-based log search alert against the Sentinel workspace — fires when more than 5 failed login events (EventID 4625) are detected within a 5 minute window:

```kql
SecurityEvent
| where EventID == 4625
| summarize FailedLogins=count() by Computer
```

| Alert | Query | Threshold | Severity |
|---|---|---|---|
| `sentinel-failed-login-alert` | EventID 4625 failed logins | Count > 5 in 5 minutes | 2 - Warning |

---

## 7. Identity & Governance

### RBAC — Role Based Access Control
Configured role assignments demonstrating least privilege at multiple scopes:

| Role | Scope | Purpose |
|---|---|---|
| Reader | Resource Group | Read-only access to all resources |
| Network Contributor | VNet resource | Network management only |
| Storage Blob Data Contributor | Storage account | Blob data access at resource scope |

**Key principle demonstrated:** RBAC is additive and scope-based — permissions assigned at resource group level are inherited by all resources within it. Resource-level assignments apply only to that specific resource.

### Azure Policy
Assigned built-in policy to enforce governance across the subscription:

- **Policy:** Require a tag on resource groups
- **Tag:** Environment
- **Effect:** Deny resource group creation without Environment tag
- **Remediation:** Applied `Environment: Lab` tag to `security-lab-rg`

This mirrors government and enterprise compliance requirements where all resources must be tagged for cost management, ownership, and audit purposes.

---

## 8. Storage Accounts

### Storage Account Configuration
Deployed `infralaborg` StorageV2 storage account with enterprise security configuration:

| Setting | Value | Purpose |
|---|---|---|
| SKU | Standard_LRS | 3 copies within single datacenter |
| Kind | StorageV2 | Latest feature support |
| Access Tier | Hot | Default for active data |
| HTTPS Only | Enabled | Encrypts data in transit |
| Minimum TLS | TLS 1.2 | Enforces modern encryption |
| Public Blob Access | Disabled | No anonymous access |
| Default Network Action | Deny | Firewall blocks all by default |

### Blob Containers & Access Tiers
Created three containers representing the three Azure blob access tiers:

| Container | Tier | Use Case | Cost |
|---|---|---|---|
| `hot-data` | Hot | Frequently accessed data | Higher storage, lower access cost |
| `cool-data` | Cool | Infrequently accessed data | Lower storage, higher access cost |
| `archive-data` | Archive | Long term retention | Lowest storage, offline retrieval |

### Lifecycle Management Policy
Configured automated tier management to move blobs between tiers based on age — reducing storage costs automatically:

```json
{
  "rules": [{
    "name": "move-to-cool",
    "definition": {
      "actions": {
        "baseBlob": {
          "tierToCool": { "daysAfterModificationGreaterThan": 30 },
          "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
          "delete": { "daysAfterModificationGreaterThan": 365 }
        }
      }
    }
  }]
}
```

| Action | Trigger | Purpose |
|---|---|---|
| Move to Cool | 30 days since last modification | Reduce cost for ageing data |
| Move to Archive | 90 days since last modification | Minimum cost long term storage |
| Delete | 365 days since last modification | Automatic cleanup |

### Shared Access Signatures (SAS)
Generated time-limited SAS token providing read-only access to a specific blob — expires automatically after 24 hours. Demonstrates secure external sharing without exposing storage account keys:

```bash
az storage blob generate-sas \
  --account-name infralaborg \
  --container-name hot-data \
  --name testfile.txt \
  --permissions r \
  --expiry 2026-05-26T00:00:00Z \
  --auth-mode login \
  --as-user \
  --full-uri
```

### Storage Firewall & Service Endpoints
Locked storage account to VNet-only access — all public internet access denied by default. Added `Microsoft.Storage` service endpoint to `public-subnet` enabling private routing to storage over the Azure backbone rather than the public internet:

- Default Action: Deny
- Bypass: AzureServices
- Allowed: public-subnet via service endpoint

---

## 9. Network Verification

Verified network architecture via management VM jump box:

```cmd
# Verify private subnet reachability from management subnet
ping 10.0.2.4 — 0% packet loss ✅

# Verify outbound internet via NAT Gateway
curl http://www.google.com — successful response ✅

# Verify Application Gateway
curl http://51.142.231.76 - Nginx HTTP 200 ✅
```

---

## Security Frameworks Referenced

| Framework | Application |
|---|---|
| NIST CSF | Protect function — network segmentation, access control, secure configuration |
| CIS Control 4 | Secure configuration — Bastion replaces exposed RDP, no public IPs on VMs |
| CIS Control 5 | Account management — RBAC least privilege across resource and subscription scopes |
| CIS Control 6 | Access control management — SAS tokens, storage firewall, service endpoints |
| CIS Control 8 | Audit log management — comprehensive audit policy on SOC VM, Sentinel ingestion |
| CIS Control 12 | Network infrastructure management — VNet segmentation, NSG rules, cross-region peering |
| CIS Control 13 | Network monitoring and defence — Azure Monitor metric and log alerts, action groups |
| Zero Trust | Verify explicitly, least privilege, assume breach — applied throughout architecture |

---

## In Progress

- [ ] Azure Key Vault — secrets, certificates, key management
- [ ] Azure Firewall — centralised network security policy
- [ ] Azure Load Balancer — internal load balancing between tiers
- [ ] Terraform — rebuild entire lab as Infrastructure as Code
- [ ] Defender for Cloud — secure score and recommendation remediation
- [ ] Active Directory — on-premise identity management
- [ ] Entra ID Connect — hybrid identity synchronisation
- [ ] VNet Peering — additional VNet connections
- [ ] Azure Monitor Workbooks — custom dashboards and visualisations

---

## Environment

- **Cloud:** Microsoft Azure
- **Regions:** North Europe (primary), UK South (Application Gateway)
- **Certification Target:** AZ-104 Microsoft Azure Administrator → AZ-500 Azure Security Engineer
- **Prerequisites:** See [prerequisites.md](prerequisites.md)
- **Related Project:** [Azure SOC Lab](https://github.com/samgarfoot/azure-soc-lab)
