# Azure Infrastructure Lab

A hands-on Azure infrastructure lab built to develop practical skills across compute, networking, identity, storage, and governance — aligned to the AZ-104 Microsoft Azure Administrator exam objectives. Built to demonstrate real-world infrastructure engineering capability in hybrid cloud environments relevant to enterprise and UK government deployments.

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
 (Nginx - web layer)   (Jump box)         (Windows Server 2022)
 ◄── App Gateway            │                    │
     Backend Pool      Azure Bastion        Microsoft Sentinel
                            │              (Log ingestion via AMA)
                    ┌───────┘
                    │
              Private Subnet
              (10.0.2.0/24)
              private-nsg
                    │
           infra-lab-container
           (App layer - isolated)
```

---

## AZ-104 Exam Domains Covered

| Domain | Topics | Status |
|---|---|---|
| Manage Azure Identities & Governance | Entra ID, RBAC, Azure Policy, resource tags | ✅ Complete |
| Deploy & Manage Azure Compute | Container Instances, Virtual Machines | ✅ Complete |
| Implement & Manage Virtual Networking | VNet, subnets, NSGs, Bastion, NAT Gateway | ✅ Complete |
| Monitor & Maintain Azure Resources | Sentinel log ingestion, Azure Monitor Agent | ✅ Complete |
| Implement & Manage Storage | Storage accounts, blob storage | ⏳ Planned |

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

## 4. Security Monitoring Integration

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

## 5. Identity & Governance

### RBAC — Role Based Access Control
Configured role assignments demonstrating least privilege at multiple scopes:

| Role | Scope | Purpose |
|---|---|---|
| Reader | Resource Group | Read-only access to all resources |
| Network Contributor | VNet resource | Network management only |

**Key principle demonstrated:** RBAC is additive and scope-based — permissions assigned at resource group level are inherited by all resources within it. Resource-level assignments apply only to that specific resource.

### Azure Policy
Assigned built-in policy to enforce governance across the subscription:

- **Policy:** Require a tag on resource groups
- **Tag:** Environment
- **Effect:** Deny resource group creation without Environment tag
- **Remediation:** Applied `Environment: Lab` tag to `security-lab-rg`

This mirrors government and enterprise compliance requirements where all resources must be tagged for cost management, ownership, and audit purposes.

---

## 6. Network Verification

Verified network architecture via management VM jump box:

```cmd
# Verify private subnet reachability from management subnet
ping 10.0.2.4 — 0% packet loss ✅

# Verify outbound internet via NAT Gateway
curl http://www.google.com — successful response ✅
```

Confirmed ICMP (ping) blocked to internet by Azure default — HTTP/HTTPS outbound working correctly via NAT Gateway.

---

## Security Frameworks Referenced

| Framework | Application |
|---|---|
| NIST CSF | Protect function — network segmentation, access control, secure configuration |
| CIS Control 4 | Secure configuration — Bastion replaces exposed RDP, no public IPs on VMs |
| CIS Control 5 | Account management — RBAC least privilege across resource scopes |
| CIS Control 8 | Audit log management — comprehensive audit policy on SOC VM |
| CIS Control 12 | Network infrastructure management — VNet segmentation, NSG rules |
| Zero Trust | Verify explicitly, least privilege, assume breach — applied throughout architecture |

---

## In Progress

- [ ] Azure Load Balancer — expose public container to internet properly
- [ ] Azure Monitor — infrastructure alerting and dashboards
- [ ] Storage Accounts — blob storage, access tiers, lifecycle management
- [ ] Terraform — rebuild entire lab as Infrastructure as Code
- [ ] Defender for Cloud — secure score and recommendation remediation
- [ ] Active Directory — on-premise identity management
- [ ] Entra ID Connect — hybrid identity synchronisation
- [ ] VNet Peering — connect multiple VNets
- [ ] Azure Firewall — centralised network security

---

## Environment

- **Cloud:** Microsoft Azure
- **Region:** North Europe
- **Certification Target:** AZ-104 Microsoft Azure Administrator → AZ-500 Azure Security Engineer
- **Related Project:** [Azure SOC Lab](https://github.com/samgarfoot/azure-soc-lab) — Microsoft Sentinel, KQL detection engineering, SOAR automation
