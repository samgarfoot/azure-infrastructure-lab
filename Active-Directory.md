# Active Directory — corp.infralab.local

Hands-on Active Directory deployment simulating an on-premises enterprise identity environment, connected to the Azure infrastructure lab via cross-region VNet peering. Built as the foundation for Entra ID Connect hybrid identity synchronisation.

---

## Domain Controller

| Setting | Value |
|---|---|
| VM | ad-01 |
| OS | Windows Server 2022 Datacenter |
| Domain | corp.infralab.local |
| NetBIOS Name | CORP |
| Hostname | ad-01.corp.infralab.local |
| IP Address | 10.2.1.4 |
| Domain Mode | Windows Server 2016 |
| Global Catalog | Enabled |
| DNS | Integrated with AD DS |
| FSMO Roles | All roles held by ad-01 |
| Subnet | ad-subnet (10.2.1.0/24) |
| VNet | ad-vnet (10.2.0.0/16) — Sweden Central |

---

## Network — Cross-Region VNet Peering

Bidirectional peering between `ad-vnet` (Sweden Central) and `azure-infra-vnet` (North Europe) enables Bastion access to `ad-01` without exposing a public IP, and provides the network path for future Entra ID Connect synchronisation.

| Peering Link | Direction | Status |
|---|---|---|
| `ad-to-infra` | ad-vnet → azure-infra-vnet | ✅ Connected |
| `infra-to-ad` | azure-infra-vnet → ad-vnet | ✅ Connected |

---

## Organisational Unit Structure

```
corp.infralab.local
├── Employees
│   ├── IT
│   ├── Finance
│   └── HR
├── Groups
├── Service Accounts
└── Corp Computers
```

---

## Users

| Name | SAM Account | OU | Department |
|---|---|---|---|
| John Smith | jsmith | OU=IT,OU=Employees | IT |
| Sarah Connor | sconnor | OU=IT,OU=Employees | IT |
| James Brown | jbrown | OU=Finance,OU=Employees | Finance |
| Emma Wilson | ewilson | OU=Finance,OU=Employees | Finance |
| Claire Davies | cdavies | OU=HR,OU=Employees | HR |
| Mike Taylor | mtaylor | OU=HR,OU=Employees | HR |

---

## Security Groups

| Group | Scope | OU | Members |
|---|---|---|---|
| IT-Staff | Global Security | OU=Groups | jsmith, sconnor |
| Finance-Staff | Global Security | OU=Groups | jbrown, ewilson |
| HR-Staff | Global Security | OU=Groups | cdavies, mtaylor |

---

## Password & Lockout Policy

Configured via `Set-ADDefaultDomainPasswordPolicy` — applied domain-wide.

| Setting | Value | Purpose |
|---|---|---|
| Minimum Password Length | 12 characters | Reduces brute force risk |
| Password History | 10 passwords | Prevents password reuse |
| Maximum Password Age | 90 days | Forces regular rotation |
| Minimum Password Age | 1 day | Prevents immediate reuse |
| Complexity | Enabled | Requires mixed character types |
| Lockout Threshold | 5 attempts | Limits brute force attempts |
| Lockout Duration | 30 minutes | Automatic unlock after cooldown |
| Observation Window | 30 minutes | Resets failed attempt counter |

---

## Security Frameworks

| Framework | Application |
|---|---|
| CIS Control 5 | Account management — departmental OU structure, security groups, least privilege group membership |
| CIS Control 6 | Access control — domain password policy, account lockout, complexity enforcement |
| Zero Trust | All domain accounts require strong passwords, lockout enforced, no standing admin access |

---

## In Progress

- [ ] Entra ID Connect — sync corp.infralab.local users to Entra ID tenant
- [ ] Group Policy Objects — password, audit, and security baseline GPOs
- [ ] Domain join soc-target-vm to corp.infralab.local
- [ ] Administrative Units in Entra ID — scoped delegated administration
- [ ] Conditional Access — MFA enforcement for synced hybrid accounts
