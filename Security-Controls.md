# Security Controls

A detailed mapping of every security control implemented in the Azure Infrastructure Lab to industry frameworks — CIS Controls v8, NIST Cybersecurity Framework, and Microsoft Zero Trust principles.

This document demonstrates the security reasoning behind architectural decisions, not just the technical implementation.

---

## Framework Overview

| Framework | Version | Application |
|---|---|---|
| CIS Controls | v8 | Implementation group IG1/IG2 controls applied throughout |
| NIST CSF | 1.1 | Identify, Protect, Detect, Respond, Recover functions |
| Microsoft Zero Trust | 2023 | Verify explicitly, least privilege, assume breach |
| NCSC Cloud Security Principles | 2023 | UK government cloud security alignment |

---

## CIS Controls Mapping

### CIS Control 1 — Inventory and Control of Enterprise Assets

| Control | Implementation | Evidence |
|---|---|---|
| 1.1 Establish and maintain asset inventory | All resources deployed with consistent naming convention and resource group tagging | `security-lab-rg` contains all assets, Environment tag applied |
| 1.2 Address unauthorised assets | NSG default deny rules prevent unregistered devices connecting to management or SOC subnets | management-nsg, soc-nsg inbound rules |

---

### CIS Control 2 — Inventory and Control of Software Assets

| Control | Implementation | Evidence |
|---|---|---|
| 2.1 Establish software inventory | Container images pinned to specific versions | `nginx:1.9.15-alpine`, `aci-helloworld:latest` |
| 2.2 Ensure authorised software only | Container instances use Microsoft Container Registry (MCR) images only | No third party or unverified images |

---

### CIS Control 3 — Data Protection

| Control | Implementation | Evidence |
|---|---|---|
| 3.1 Establish data management process | Blob containers segmented by access tier and data classification | hot-data, cool-data, archive-data |
| 3.2 Establish data retention policy | Lifecycle management policy automates retention and deletion | Delete after 365 days |
| 3.3 Configure data access control | Storage account RBAC — Storage Blob Data Contributor at resource scope | Role assignment on infralaborg |
| 3.4 Enforce data encryption at rest | Azure Storage Service Encryption enabled by default — AES-256 | Enabled on infralaborg |
| 3.5 Enforce data encryption in transit | HTTPS only enforced, TLS 1.2 minimum | infralaborg configuration |
| 3.6 Encrypt data on removable media | Not applicable — cloud-only environment | — |
| 3.10 Encrypt sensitive data in transit | All storage traffic encrypted via TLS 1.2, service endpoints bypass public internet | Service endpoint on public-subnet |

---

### CIS Control 4 — Secure Configuration of Enterprise Assets

| Control | Implementation | Evidence |
|---|---|---|
| 4.1 Establish secure configuration process | All resources deployed via Azure CLI with explicit security parameters | COMMANDS.md |
| 4.2 Establish secure configuration for network infrastructure | NSGs configured on every subnet with least privilege rules | public-nsg, private-nsg, management-nsg, soc-nsg |
| 4.3 Configure automatic session locking | Azure Bastion sessions require re-authentication via Entra ID MFA | Bastion + Entra ID |
| 4.6 Securely manage enterprise assets | No VM has a public IP — all access via Bastion only | management-vm, soc-target-vm |

---

### CIS Control 5 — Account Management

| Control | Implementation | Evidence |
|---|---|---|
| 5.1 Establish account management process | All access managed via Entra ID — no local accounts with internet access | Entra ID tenant |
| 5.2 Use unique passwords | MFA enforced via Entra ID for all Bastion sessions | Entra ID MFA policy |
| 5.4 Restrict administrator privileges | RBAC role assignments at minimum required scope — resource level not subscription | Storage Blob Data Contributor at storage scope |
| 5.5 Establish access revoking process | Role assignments removable via CLI or portal instantly | az role assignment delete |
| 5.6 Centralise account management | All identities managed in single Entra ID tenant | Default Directory |

---

### CIS Control 6 — Access Control Management

| Control | Implementation | Evidence |
|---|---|---|
| 6.1 Establish access control policy | RBAC assignments follow least privilege — Reader, Network Contributor, Storage Blob Data Contributor | Role assignment table in README |
| 6.2 Establish an allow list for authorised remote access | Bastion is the only authorised remote access path — no public IPs, no exposed RDP/SSH | azure-infra-bastion |
| 6.3 Require MFA for externally-exposed applications | Bastion access requires Entra ID MFA | Entra ID MFA |
| 6.4 Require MFA for remote access | All VM access via Bastion requires Entra ID authentication | azure-infra-bastion |
| 6.7 Centralise access control | All network access controlled at subnet level via NSGs | Four NSGs across five subnets |
| 6.8 Define and maintain role-based access | Three distinct RBAC roles assigned at appropriate scopes | Reader, Network Contributor, Storage Blob Data Contributor |

---

### CIS Control 8 — Audit Log Management

| Control | Implementation | Evidence |
|---|---|---|
| 8.1 Establish audit log management process | Windows audit policy configured on SOC VM covering all critical categories | auditpol configuration |
| 8.2 Collect audit logs | Azure Monitor Agent forwards security events to Log Analytics workspace | AMA + Data Collection Rule |
| 8.5 Collect detailed audit logs | Command line logging enabled — full process creation with command line arguments captured | ProcessCreationIncludeCmdLine registry key |
| 8.7 Collect URL request audit logs | Application Gateway access logs record all HTTP requests | infra-app-gateway |
| 8.9 Centralise audit logs | All logs centralised in security-lab-workspace Log Analytics | Microsoft Sentinel |
| 8.11 Conduct audit log reviews | KQL-based log search alert fires on anomalous login patterns | sentinel-failed-login-alert |

---

### CIS Control 12 — Network Infrastructure Management

| Control | Implementation | Evidence |
|---|---|---|
| 12.1 Ensure network infrastructure is up to date | Azure-managed infrastructure — Microsoft handles patching of Bastion, NAT Gateway, Application Gateway | Managed services |
| 12.2 Establish and maintain secure network architecture | Hub-and-spoke VNet with five isolated subnets — each with dedicated NSG | azure-infra-vnet |
| 12.3 Securely manage network infrastructure | All network changes via Azure CLI with explicit parameters — no implicit defaults | COMMANDS.md |
| 12.4 Establish and maintain architecture diagram | Full architecture diagram maintained in README | README.md |
| 12.6 Use of secure network management protocols | All management traffic via Bastion HTTPS — no Telnet, no unencrypted RDP | azure-infra-bastion |
| 12.7 Ensure remote devices use VPN or zero trust | Bastion implements zero trust remote access — verify identity before session | azure-infra-bastion |
| 12.8 Establish and maintain dedicated computing resources | SOC subnet isolated from workload subnets — security tooling on separate network tier | soc-subnet, soc-nsg |

---

### CIS Control 13 — Network Monitoring and Defence

| Control | Implementation | Evidence |
|---|---|---|
| 13.1 Centralise security event alerting | Azure Monitor action group delivers all alerts to single admin email | infra-alert-group |
| 13.2 Deploy a host-based intrusion detection solution | Microsoft Sentinel with AMA collecting Windows Security Events on SOC VM | Sentinel + AMA |
| 13.3 Deploy a network intrusion detection solution | Application Gateway provides Layer 7 inspection of all inbound HTTP traffic | infra-app-gateway |
| 13.6 Collect network traffic flow logs | NAT Gateway flow logs available for outbound traffic analysis | infra-nat-gateway |
| 13.7 Deploy a host-based intrusion prevention solution | Windows audit policy captures privilege escalation, account changes, policy modifications | auditpol configuration |
| 13.8 Deploy a network intrusion prevention solution | NSG rules deny all traffic not explicitly permitted — acts as network IPS | Four NSGs |

---

## NIST CSF Mapping

### Identify (ID)

| Subcategory | Implementation |
|---|---|
| ID.AM-1 Physical devices inventoried | All Azure resources tagged and inventoried in security-lab-rg |
| ID.AM-3 Organisational communication mapped | Architecture diagram documents all data flows and network paths |
| ID.GV-1 Organisational policy established | Azure Policy enforces Environment tag on all resource groups |
| ID.RA-1 Asset vulnerabilities identified | Defender for Cloud (planned) — secure score and recommendations |

### Protect (PR)

| Subcategory | Implementation |
|---|---|
| PR.AC-1 Identities managed | Entra ID manages all identities — no local accounts with internet exposure |
| PR.AC-3 Remote access managed | Azure Bastion — zero trust remote access, MFA enforced |
| PR.AC-4 Access permissions managed | RBAC at minimum required scope — least privilege throughout |
| PR.AC-5 Network integrity protected | Five isolated subnets with NSG least privilege rules |
| PR.DS-1 Data at rest protected | AES-256 encryption on all storage — enabled by default |
| PR.DS-2 Data in transit protected | TLS 1.2 minimum, HTTPS only, service endpoints bypass public internet |
| PR.DS-5 Protections against data leaks | Storage firewall default deny, no public blob access, SAS for controlled sharing |
| PR.IP-1 Baseline configuration established | All resources deployed with explicit security parameters via CLI |
| PR.IP-3 Configuration change control | All changes documented in COMMANDS.md — full audit trail |
| PR.PT-3 Least functionality principle | Containers run minimal images — Alpine-based Nginx, no unnecessary packages |
| PR.PT-4 Communications and networks protected | NAT Gateway outbound only, private subnet fully isolated, Bastion for management |

### Detect (DE)

| Subcategory | Implementation |
|---|---|
| DE.AE-1 Baseline network operations established | NSG rules define expected traffic patterns — deviations blocked |
| DE.AE-3 Event data aggregated | Log Analytics workspace centralises all security events |
| DE.CM-1 Networks monitored | Azure Monitor metric alerts on both VMs — CPU threshold monitoring |
| DE.CM-3 Personnel activity monitored | Windows audit policy captures all user activity on SOC VM |
| DE.CM-7 Monitoring for unauthorised activity | KQL log search alert fires on failed login threshold breach |
| DE.DP-4 Event detection validated | Alert pipeline tested — action group configured and verified |

### Respond (RS)

| Subcategory | Implementation |
|---|---|
| RS.CO-2 Incidents reported | Azure Monitor action group emails admin on alert trigger |
| RS.AN-1 Notifications investigated | Microsoft Sentinel incident management for security events |
| RS.MI-1 Incidents contained | NSG rules provide network-level containment — subnet isolation limits blast radius |

### Recover (RC)

| Subcategory | Implementation |
|---|---|
| RC.RP-1 Recovery plan executed | VMs can be restarted via CLI — start commands documented in COMMANDS.md |
| RC.IM-1 Recovery plans incorporate lessons learned | TROUBLESHOOTING.md documents issues encountered and resolutions |

---

## Zero Trust Principles

### Verify Explicitly

| Principle | Implementation |
|---|---|
| Always authenticate and authorise | Bastion enforces Entra ID MFA before any VM session |
| Use multiple data points | Entra ID evaluates identity, device, location before granting access |
| Real-time risk assessment | Sentinel monitors for anomalous behaviour post-authentication |

### Use Least Privilege Access

| Principle | Implementation |
|---|---|
| Limit user access with just-in-time | RBAC assignments at minimum required scope |
| Limit lateral movement | Subnet isolation — compromise of one tier does not grant access to others |
| Segment access by sensitivity | Five subnets with decreasing trust levels from public to private |

### Assume Breach

| Principle | Implementation |
|---|---|
| Minimise blast radius | Subnet segmentation contains compromise to individual tiers |
| Encrypt all sessions | TLS 1.2 minimum, HTTPS only, Bastion HTTPS sessions |
| Use analytics to detect threats | Sentinel KQL alerts on failed logins, Azure Monitor on infrastructure anomalies |
| Improve defences continuously | In Progress section tracks planned security improvements |

---

## NCSC Cloud Security Principles

Aligned to the NCSC 14 Cloud Security Principles — particularly relevant for UK government deployments:

| Principle | Implementation |
|---|---|
| 1. Data in transit protection | TLS 1.2 minimum, HTTPS only, service endpoints |
| 2. Asset protection and resilience | Standard LRS — 3 copies of all storage data |
| 3. Separation between customers | Single tenant Azure subscription — no shared infrastructure |
| 4. Governance framework | Azure Policy, RBAC, resource tagging |
| 5. Operational security | Audit logging, Azure Monitor alerts, Sentinel SIEM |
| 6. Personnel security | Entra ID MFA, RBAC least privilege |
| 7. Secure development | CLI-driven deployment — documented, repeatable, auditable |
| 8. Supply chain security | Microsoft Container Registry images only |
| 9. Secure user management | Entra ID centralised identity, no shared accounts |
| 10. Identity and authentication | Entra ID MFA enforced for all Bastion sessions |
| 11. External interface protection | No public IPs on VMs, Application Gateway as single ingress point |
| 12. Secure service administration | Bastion replaces exposed RDP/SSH |
| 13. Audit information | Log Analytics workspace, Windows audit policy, Azure Monitor |
| 14. Secure use of the service | NSG least privilege rules, storage firewall, SAS tokens |

---

## Security Control Summary

| Category | Controls Implemented | Framework |
|---|---|---|
| Identity & Access | Entra ID MFA, RBAC least privilege, centralised identity | CIS 5, 6 / NIST PR.AC |
| Network Security | Five isolated subnets, NSG least privilege, NAT Gateway, Bastion | CIS 12 / NIST PR.PT |
| Data Protection | AES-256 encryption, TLS 1.2, storage firewall, SAS tokens | CIS 3 / NIST PR.DS |
| Secure Configuration | CLI deployment, explicit parameters, no implicit defaults | CIS 4 / NIST PR.IP |
| Monitoring & Detection | Sentinel, Azure Monitor, Windows audit policy, KQL alerts | CIS 8, 13 / NIST DE |
| Incident Response | Action groups, email alerts, subnet containment | NIST RS |
| Governance | Azure Policy, resource tagging, Environment tag enforcement | CIS 1 / NIST ID.GV |
