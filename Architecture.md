# Architecture Design Decisions

A detailed explanation of the design decisions behind the Azure Infrastructure Lab — covering why each component was chosen, how they interact, and how the architecture aligns to enterprise and UK government deployment patterns.

---

## Design Philosophy

The lab is built around three core principles drawn from Zero Trust and CIS security frameworks:

- **Least privilege** — every component has only the access it needs, nothing more
- **Defence in depth** — multiple security layers so no single failure exposes the environment
- **Explicit verification** — no implicit trust between network tiers, all traffic controlled at subnet level

---

## Network Architecture — Why Hub and Spoke

A hub-and-spoke VNet design was chosen over a flat single-subnet architecture for three reasons:

**1. Blast radius containment**
If the public-facing Nginx container is compromised, the attacker is contained within `public-subnet`. NSG rules prevent lateral movement to `private-subnet` or `management-subnet` — they would need to break through additional network controls to progress further.

**2. Mirrors enterprise deployments**
UK government and enterprise Azure environments almost universally use subnet segmentation. A flat network is a red flag in any architecture review. This design demonstrates familiarity with production patterns.

**3. Granular traffic control**
Each subnet has its own NSG with tailored rules. Public subnet allows HTTP/HTTPS inbound. Private subnet allows only traffic from public subnet. Management subnet allows only admin access. SOC subnet is internal only. This is impossible to achieve cleanly in a flat network.

**Why five subnets:**

| Subnet | Reason for isolation |
|---|---|
| public-subnet | Internet-facing — highest risk, must be isolated |
| private-subnet | Backend data — never directly internet accessible |
| management-subnet | Admin access — separate from workload traffic |
| AzureBastionSubnet | Mandatory dedicated subnet — Azure requirement for Bastion |
| soc-subnet | Security tooling — isolated from workload to prevent tampering |

---

## Application Gateway — Why Not a Basic Load Balancer

A Standard_v2 Application Gateway was chosen over a basic Azure Load Balancer for several reasons:

**Layer 7 vs Layer 4**
A basic Load Balancer operates at Layer 4 — it distributes TCP/UDP connections without understanding HTTP. Application Gateway operates at Layer 7 — it understands HTTP/HTTPS, can route based on URL paths, hostnames, and headers, and can inspect request content.

**WAF capability**
Application Gateway Standard_v2 supports the Web Application Firewall (WAF) add-on — providing OWASP ruleset protection against SQL injection, XSS, and other web attacks. A basic load balancer has no application-layer inspection capability.

**Cross-region architecture**
The lab deliberately deploys the Application Gateway in UK South while the backend runs in North Europe. This demonstrates cross-region VNet peering and mirrors real enterprise patterns where ingress and compute are in different regions for compliance or latency reasons.

**AZ-104 and AZ-500 relevance**
Application Gateway is a core topic in both AZ-104 and AZ-500. Understanding SKUs, backend pools, routing rules, and health probes is directly exam-relevant. A basic load balancer covers far less of the exam syllabus.

---

## Azure Bastion — Why Not a Jump Box with Public IP

The traditional approach to VM access in a lab is to assign a public IP and open RDP/SSH to the internet. This was deliberately avoided:

**Exposed RDP is one of the most exploited attack vectors in cloud environments.** RDP brute force and credential stuffing against publicly exposed VMs is automated and constant. Any VM with port 3389 open to the internet will receive attack traffic within minutes of deployment.

Azure Bastion eliminates this by:
- Removing the need for any public IP on VMs
- Proxying RDP/SSH through the Azure portal over HTTPS/443 only
- Enforcing Entra ID authentication and MFA before any session begins
- Logging all session activity in Azure Monitor

**The tradeoff:** Bastion costs approximately £0.19/hour. For a lab this is worth it for the security posture and the learning value — Bastion is a core AZ-104 exam topic and a standard enterprise control.

---

## NAT Gateway — Why Outbound-Only

The NAT Gateway provides outbound internet access to public, management, and SOC subnets while keeping private-subnet fully isolated. This is a deliberate security decision:

**Without NAT Gateway**, VMs and containers would need public IPs to reach the internet for updates, package downloads, and API calls — creating unnecessary attack surface.

**With NAT Gateway**, all outbound traffic exits through a single managed IP address. Inbound connections from the internet are not possible via NAT — only traffic initiated from inside the VNet can traverse it.

**Private subnet has no NAT Gateway by design.** The application tier should never initiate outbound internet connections. If it does, that is a potential indicator of compromise — data exfiltration, C2 communication, or misconfiguration.

---

## Storage Account Security Design

The storage account was configured with multiple overlapping security controls — demonstrating defence in depth applied to data storage:

**Why TLS 1.2 minimum:**
TLS 1.0 and 1.1 have known vulnerabilities. Enforcing TLS 1.2 minimum ensures all data in transit is protected by a modern cipher suite. This aligns to NCSC and NIST guidance on transport security.

**Why public blob access disabled:**
Anonymous public access to blob containers is a leading cause of cloud data breaches. Disabling it at the account level means no container can ever be made publicly accessible, regardless of individual container settings.

**Why default action Deny on storage firewall:**
A default Allow posture means any new network path to the storage account works automatically — including unexpected ones. A default Deny posture means every access path must be explicitly permitted. This is the Zero Trust model applied to storage.

**Why service endpoints over public internet routing:**
Without service endpoints, traffic from `public-subnet` to the storage account travels over the public internet, even though both are Azure resources. Service endpoints route traffic over the Azure backbone network — faster, more secure, and never leaving the Microsoft network.

**Why lifecycle management:**
In production environments, storing all data in Hot tier regardless of access patterns is expensive and unnecessary. Lifecycle management automates cost optimisation — data automatically moves to cheaper tiers as it ages, without manual intervention.

---

## Cross-Region VNet Peering — Design Considerations

Deploying the Application Gateway in UK South while the backend runs in North Europe introduced a cross-region dependency that required careful consideration:

**Address space planning:**
`appgw-vnet` uses `10.1.0.0/16` and `azure-infra-vnet` uses `10.0.0.0/16`. These address spaces must not overlap — overlapping address spaces cannot be peered. In larger environments this requires careful IP address management (IPAM) planning.

**Bidirectional peering requirement:**
VNet peering in Azure is not transitive and not automatic in both directions. Creating a peering from VNet A to VNet B does not automatically create the return peering. Both sides must be explicitly configured — a common source of connectivity issues in enterprise environments.

**Latency consideration:**
Cross-region traffic introduces latency compared to same-region deployments. For a production workload this would need to be measured and justified. For this lab it demonstrates the capability and is acceptable for a showcase architecture.

---

## Security Monitoring Architecture

The security monitoring stack was designed to provide visibility at multiple levels:

**Host level — Windows audit policy:**
Configured directly on `soc-target-vm` via PowerShell. Captures logon events, privilege use, account management, and policy changes. This is the raw event source — everything starts here.

**Collection level — Azure Monitor Agent:**
AMA replaces the legacy Log Analytics Agent and collects events from the host based on Data Collection Rules. DCRs define exactly which event categories and IDs are forwarded to the workspace — avoiding log noise from irrelevant events.

**Analysis level — Microsoft Sentinel:**
Sentinel receives the collected events into the Log Analytics workspace and makes them queryable via KQL. This is where detection rules, hunting queries, and incident management live.

**Alerting level — Azure Monitor:**
Sits alongside Sentinel and covers infrastructure metrics (CPU, memory) that Sentinel doesn't monitor. The two systems are complementary — Sentinel for security events, Azure Monitor for infrastructure health.

**Why this separation matters:**
A single alerting system covering both security events and infrastructure metrics becomes noisy and hard to manage. Security analysts should not be receiving CPU alerts. Infrastructure teams should not be triaging failed login alerts. Separating concerns keeps each system focused and actionable.

---

## What This Architecture Demonstrates

| Capability | Evidence |
|---|---|
| Network segmentation | Five isolated subnets with granular NSG rules |
| Zero Trust networking | Default deny, explicit allow, no implicit trust between tiers |
| Secure remote access | Bastion replacing exposed RDP/SSH |
| Layer 7 traffic management | Application Gateway with backend pool and routing rules |
| Cross-region architecture | VNet peering between UK South and North Europe |
| Defence in depth storage | Firewall, service endpoints, SAS, RBAC, lifecycle management |
| Security monitoring | Host audit policy → AMA → Sentinel → Azure Monitor alerts |
| Least privilege RBAC | Role assignments at resource scope, not subscription scope |
| Infrastructure as CLI | All resources deployed and documented via Azure CLI |
