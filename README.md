# Zero Trust Azure Landing Zone — CloudMed Solutions Inc.

> **Course Lab | Azure Cloud Governance & Security Architecture**  
> Designed by: [Your Name]  
> Date: March 2026

---

## Table of Contents

1. [Company Overview](#1-company-overview)
2. [Governance and Identity](#2-governance-and-identity)
3. [Network Architecture](#3-network-architecture)
4. [Zero Trust Controls](#4-zero-trust-controls)
5. [Monitoring, Compliance, and Cost](#5-monitoring-compliance-and-cost)
6. [Conceptual Architecture Diagram](#6-conceptual-architecture-diagram)
7. [Summary and Recommendations](#7-summary-and-recommendations)

---

## 1. Company Overview

### About CloudMed Solutions Inc.

CloudMed Solutions Inc. is a Canadian healthcare technology company that delivers cloud-based telemedicine and patient management platforms to hospitals and clinics across **Canada**, the **United States**, and **Europe**. Their flagship product, **MedConnect**, enables:

- **Secure telehealth sessions** between physicians and patients via real-time video and messaging
- **Electronic Medical Record (EMR) management** for clinical documentation and patient history
- **AI-based health analytics** for predictive diagnostics and operational intelligence

CloudMed processes and stores **Protected Health Information (PHI)** for millions of patients across multiple jurisdictions, making data security and regulatory compliance a business-critical concern — not an afterthought.

### Why Zero Trust?

Traditional perimeter-based ("castle-and-moat") security models assume that everything inside the network can be trusted. This assumption is fundamentally unsafe for CloudMed because:

| Risk Factor | Impact on CloudMed |
|---|---|
| Multi-region operations (Canada, EU) | Broader attack surface across geo-distributed infrastructure |
| Third-party integrations (hospitals, clinics) | External identities accessing sensitive workloads |
| Sensitive PHI and EMR data | High-value target for ransomware, insider threats, and exfiltration |
| Remote clinical workforce | Users connecting from unmanaged or personal devices |
| Regulatory scrutiny | Breaches trigger mandatory reporting, fines, and reputational damage |

A **Zero Trust architecture** eliminates implicit trust. Every user, device, and service must continuously authenticate and prove authorization — regardless of network location. This is the only model robust enough for a healthcare provider operating under multiple international compliance regimes.

### Key Compliance and Operational Drivers

| Regulation | Jurisdiction | Key Requirement |
|---|---|---|
| **HIPAA** | United States | Protection of PHI, audit controls, breach notification |
| **GDPR** | European Union | Data minimization, right to erasure, cross-border transfer restrictions |
| **PIPEDA** | Canada | Consent for data collection, accountability, safeguards for personal information |

Operationally, CloudMed requires:
- Centralized governance and policy enforcement across all subscriptions
- Full visibility into resource configurations and access patterns
- Cost accountability and departmental chargeback
- Scalable architecture that can onboard new regions or business units without redesign

---

## 2. Governance and Identity

### 2.1 Management Group Hierarchy

The management group hierarchy enforces policy and RBAC at scale — policies applied at a higher level cascade automatically to all child subscriptions.

```
Tenant Root Group
└── CloudMed (Root Management Group)
    ├── Platform
    │   ├── Connectivity Subscription      ← Hub VNet, Firewall, DNS, Bastion
    │   └── Management Subscription        ← Log Analytics, Defender, Automation
    ├── Production
    │   ├── App-Prod Subscription          ← Web / Mobile App Tier
    │   ├── API-Prod Subscription          ← Backend API Tier
    │   └── Data-Prod Subscription         ← Azure SQL, Storage, Key Vault
    └── Development
        ├── App-Dev Subscription
        ├── API-Dev Subscription
        └── Sandbox Subscription           ← Experimentation, no PHI
```

**Design Rationale:**
- **Platform** subscriptions contain shared infrastructure (networking, monitoring) — centrally managed by the Platform team with no business workloads present.
- **Production** subscriptions are isolated by tier (App, API, Data) to enforce least privilege and minimize blast radius.
- **Development** subscriptions use the same topology as Production but are explicitly policy-blocked from accessing real PHI data and from exposing public IPs.
- The **Sandbox** subscription allows developers to experiment freely without governance overhead — and is locked away from the production network.

---

### 2.2 Role-Based Access Control (RBAC)

RBAC assignments are made at the **management group or subscription level** — not individual resources — to ensure consistent enforcement.

| Role | Scope | Permissions |
|---|---|---|
| **Cloud Platform Admin** | CloudMed Root MG | Full access to Platform subscriptions; read-only to Production |
| **Security Admin** | CloudMed Root MG | Read access everywhere; write to Defender and Policy |
| **DevOps Engineer** | Production & Dev Subscriptions | Contributor on compute/networking; no access to Key Vault secrets |
| **Data Engineer** | Data-Prod Subscription | Read/write to approved storage accounts and SQL databases via Managed Identity only |
| **Finance / FinOps** | CloudMed Root MG | Read-only; Cost Management + Billing Contributor |
| **Developer** | Dev Subscriptions | Contributor in Dev; no Production access |
| **Clinician / End User** | Application-level (AAD groups) | Access only to MedConnect app via Entra ID — never to Azure infrastructure |

All admin roles use **Azure Privileged Identity Management (PIM)** for Just-in-Time (JIT) elevation — no one holds standing Contributor or Owner rights in Production.

---

### 2.3 Azure Policy Assignments

Policies are inherited through the management group hierarchy. Key policies include:

| Policy | Scope | Effect |
|---|---|---|
| **Allowed Regions** | CloudMed Root MG | Only `canadacentral` and `westeurope` permitted |
| **Require Resource Tags** | CloudMed Root MG | Enforce `env`, `owner`, `costcenter`, `dataclass` tags on all resources |
| **No Public IP Addresses** | Production MG | Deny creation of public IP resources in production |
| **Require Encryption at Rest** | Production & Platform MG | Audit/deny storage accounts without encryption enabled |
| **Require Private Endpoints** | Data-Prod Subscription | Deny creation of SQL/Storage without Private Endpoint |
| **Allowed VM SKUs** | Dev Subscriptions | Restrict to cost-appropriate SKUs in non-production |
| **Require Diagnostic Settings** | All Subscriptions | Audit resources that do not forward logs to Log Analytics |
| **Deny Classic Resources** | CloudMed Root MG | Block legacy (non-ARM) resource types |

Policies are defined and managed using **Azure Policy Initiatives (policy sets)** grouped by compliance framework (HIPAA, GDPR, CIS Benchmark), enabling one-click compliance reporting.

---

### 2.4 Identity and Access — Azure Entra ID

CloudMed uses **Microsoft Entra ID (formerly Azure AD)** as the single identity plane for all cloud access.

**Key Identity Controls:**

- **Multi-Factor Authentication (MFA)** — Required for all users. Enforced via Conditional Access policy, with no exceptions for admin accounts.
- **Conditional Access Policies:**
  - Require MFA + compliant device for access to Production workloads
  - Block sign-ins from countries outside Canada, USA, and EU
  - Require MFA step-up for any Privileged Identity Management elevation
  - Block legacy authentication protocols (basic auth, NTLM)
- **Managed Identities** — All Azure services authenticate to other services (Key Vault, SQL, Storage) using system-assigned or user-assigned managed identities. No service accounts with passwords.
- **Workload Identity Federation** — CI/CD pipelines authenticate to Azure without storing secrets.
- **Entra ID Groups** — RBAC is assigned to AAD security groups, never individual users, making access reviews and offboarding auditable.

---

## 3. Network Architecture

### 3.1 Hub-and-Spoke Design

CloudMed's network follows a **Hub-and-Spoke topology** deployed across two Azure regions: **Canada Central** (primary) and **West Europe** (secondary). This architecture centralizes security controls in a shared hub while isolating workloads in dedicated spokes.

```
CANADA CENTRAL REGION
┌──────────────────────────────────────────────────────────────┐
│  HUB VNet (10.0.0.0/16) — Connectivity Subscription         │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │ Azure Firewall│  │ Azure Bastion│  │  Private DNS Zone │  │
│  │ Premium       │  │  (Admin Only)│  │  (internal names) │  │
│  └──────────────┘  └──────────────┘  └───────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Gateway Subnet — ExpressRoute / VPN Gateway          │    │
│  └──────────────────────────────────────────────────────┘    │
└───────────┬──────────────────────────────────────────────────┘
            │ VNet Peering (hub-spoke, gateway transit enabled)
   ┌─────────┴──────────────────────────────────────┐
   │                                                 │
   ▼                                                 ▼
SPOKE 1: App VNet (10.1.0.0/16)       SPOKE 2: API VNet (10.2.0.0/16)
App-Prod Subscription                 API-Prod Subscription
┌────────────────────────┐            ┌────────────────────────┐
│ Subnet: App-Frontend   │            │ Subnet: API-Services   │
│ (App Service, APIM)    │            │ (AKS / App Service)    │
│                        │            │                        │
│ Subnet: App-Internal   │            │ Subnet: API-Internal   │
│ (Private Link, PE)     │            │ (Private Endpoints)    │
└────────────────────────┘            └────────────────────────┘
            │                                       │
            └─────────────┬─────────────────────────┘
                          │
                          ▼
            SPOKE 3: Data VNet (10.3.0.0/16)
            Data-Prod Subscription
            ┌──────────────────────────────┐
            │ Subnet: DataServices         │
            │ (Azure SQL MI, Key Vault PE) │
            │                              │
            │ Subnet: Storage              │
            │ (Storage Accounts, ADLS)     │
            └──────────────────────────────┘
```

**Regional Pairing:** An equivalent hub-and-spoke topology exists in **West Europe**, connected to Canada Central via **Global VNet Peering** through the respective hubs. Traffic between regions flows through both regional Azure Firewalls for inspection.

---

### 3.2 Hub Components

| Component | Purpose | Zero Trust Alignment |
|---|---|---|
| **Azure Firewall Premium** | Inspect and filter all north-south (internet) and east-west (inter-spoke) traffic using IDPS and TLS inspection | Assume Breach — no unfiltered traffic between zones |
| **Azure Bastion (Standard)** | Browser-based, RDP/SSH access to VMs without public IPs | Least Privilege — JIT access via PIM; no persistent connectivity |
| **Azure Private DNS Zones** | Internal name resolution for private endpoints across all spokes | Verify Explicitly — services resolve to private IPs only |
| **ExpressRoute / VPN Gateway** | Encrypted connectivity to on-premises hospital networks | Assume Breach — encrypted channels; monitored via NSG flow logs |
| **Log Analytics Workspace** | Central sink for all diagnostic logs, NSG flow logs, firewall logs | Visibility layer |
| **Azure DDoS Protection Standard** | Layer 3/4 protection for public-facing resources | Perimeter hardening |

---

### 3.3 Spoke Isolation and East-West Traffic Control

**Workloads are isolated across three production spokes:**

- **App Spoke** — Hosts the MedConnect web frontend (Azure App Service) and API Management Gateway (APIM). The only spoke with any internet-facing exposure, fronted by Azure Application Gateway with Web Application Firewall (WAF) enabled.
- **API Spoke** — Contains backend microservices running on Azure Kubernetes Service (AKS). No public endpoints. All traffic enters through the App spoke via APIM or through the Firewall.
- **Data Spoke** — Contains Azure SQL Managed Instance, Azure Data Lake Storage Gen2, and Azure Key Vault. **Completely isolated** — no direct spoke-to-internet path exists.

**East-West Traffic Control (inter-spoke):**
- All inter-spoke traffic is routed through **User-Defined Routes (UDRs)** that force traffic to the Azure Firewall in the hub before it can reach another spoke.
- Azure Firewall Premium inspects this traffic with **IDPS (Intrusion Detection and Prevention System)** signatures enabled in alert-and-deny mode.
- **Network Security Groups (NSGs)** are applied at the **subnet level** in every spoke with explicit allow rules — default-deny for all other traffic.

**Private Endpoints:**
- Azure SQL, Azure Storage, Azure Key Vault, and Azure Container Registry all use **Private Endpoints** within their respective subnet. Their public network access is disabled at the resource level.
- Private DNS zones in the hub resolve `*.database.windows.net`, `*.blob.core.windows.net`, etc. to the private IP of the endpoint — ensuring no traffic ever leaves the virtual network.

---

## 4. Zero Trust Controls

### 4.1 Principle 1 — Verify Explicitly

*Authenticate and authorize every access attempt using all available data points: identity, location, device health, service, and data classification.*

**Implementation:**

- **Entra ID with Conditional Access** — Every user session is evaluated against policies that check: MFA completion, device compliance state (via Intune), sign-in risk score (Identity Protection), and user location. A clinician signing in from an unmanaged device or from an unexpected country will be blocked or challenged.
- **Azure API Management (APIM)** — All API calls to MedConnect backend services pass through APIM, which enforces OAuth 2.0 / JWT validation and rate limiting. Services that fail token validation receive a 401 — no fallback to anonymous access.
- **Managed Identity for Service-to-Service Auth** — No service uses a username/password or a stored secret to connect to Azure resources. Every workload authenticates with a Managed Identity token that is scoped precisely to what it needs.
- **Certificate-Based mTLS** — AKS microservices use mutual TLS with a service mesh (Azure Service Mesh / Istio) to verify both the client and the server in every east-west service call.

---

### 4.2 Principle 2 — Use Least Privilege Access

*Limit access to only what is explicitly required. Eliminate standing privileges.*

**Implementation:**

- **Just-in-Time (JIT) Access via PIM** — Platform Admins and Security Admins have no standing Contributor or Owner assignments. They request elevation through PIM, which requires MFA re-authentication, business justification, and an approver. Elevation is time-limited (max 4 hours) and fully logged.
- **Minimal RBAC Scopes** — Role assignments are made at the most specific scope possible. A DevOps engineer deploying to the API-Prod subscription has no visibility into the Data-Prod subscription.
- **Key Vault Access Policies** — Secrets, keys, and certificates in Azure Key Vault are accessible only to the specific Managed Identity of the workload that needs them. Humans never retrieve production secrets directly.
- **Azure SQL Contained Database Users** — Application services connect to Azure SQL using AAD authentication (Managed Identity). No SQL logins with passwords are provisioned in production databases.
- **Network Least Privilege via NSG Rules** — NSG rules are written as explicit allows with source/destination IP, port, and protocol. No wildcard rules (0.0.0.0/0) exist in production subnets.

---

### 4.3 Principle 3 — Assume Breach

*Design as if the attacker is already inside. Segment, monitor, encrypt, and prepare to respond.*

**Implementation:**

- **Network Segmentation** — Three production spokes with no direct peering between them. All lateral traffic is filtered by Azure Firewall, making lateral movement from the App tier to the Data tier require traversing a firewall with IDPS.
- **Encryption Everywhere** — Data at rest is encrypted using Azure-managed keys (upgraded to Customer-Managed Keys via Key Vault for PHI workloads). Data in transit uses TLS 1.2+ enforced via Azure Policy. Azure SQL Transparent Data Encryption (TDE) is enabled on all databases.
- **Microsoft Defender for Cloud** — Continuously evaluates resource configurations against HIPAA and CIS benchmarks. Defender for Servers, Defender for SQL, Defender for Containers, and Defender for Storage are all enabled to generate real-time threat alerts.
- **Incident Response Readiness** — Azure Sentinel (Microsoft Sentinel) is deployed in the Management subscription as the SIEM/SOAR platform. Playbooks automate response to common incidents (e.g., auto-revoke session tokens on high-risk sign-in alerts, auto-block IP on Firewall threat intel match).
- **Immutable Audit Logs** — All activity logs, diagnostic logs, and security logs are written to a Storage Account with **immutable blob storage (WORM)** enabled — ensuring logs cannot be deleted or modified even by administrators.

---

### 4.4 Specific Design Examples

| # | Design Example | Zero Trust Principle |
|---|---|---|
| 1 | **Azure Bastion for Admin Access** — All VM administrative access goes through Azure Bastion in the hub. No public IPs on VMs. JIT VM access via Defender for Cloud further restricts when RDP/SSH ports are even open (15-minute windows). | Least Privilege + Assume Breach |
| 2 | **Private Link for Azure SQL** — Azure SQL Managed Instance in the Data spoke is accessible only via a Private Endpoint. Public access is disabled at the resource level. DNS resolves to the private IP. Apps in the API spoke connect over the private endpoint through the firewall. | Verify Explicitly + Assume Breach |
| 3 | **Azure Policy: Deny Public IPs in Production** — A built-in deny policy applied at the Production management group prevents any team from deploying a public IP resource. This eliminates accidental internet exposure as a human-error category entirely. | Assume Breach |
| 4 | **Conditional Access: Compliant Device Requirement** — Clinical staff must use Intune-enrolled, compliant devices to access MedConnect or any Azure management portal. Non-compliant devices (missing patches, disabled disk encryption) are automatically blocked. | Verify Explicitly |
| 5 | **Customer-Managed Keys for PHI Data** — Azure Storage accounts and Azure SQL databases holding PHI are encrypted with Customer-Managed Keys stored in Azure Key Vault. CloudMed controls key rotation and can revoke access to data instantly by disabling the key — useful in breach or legal hold scenarios. | Assume Breach + Least Privilege |

---

## 5. Monitoring, Compliance, and Cost

### 5.1 Monitoring Architecture

All CloudMed subscriptions forward logs and metrics to a **centralized Log Analytics Workspace** in the Management subscription. This creates a single pane of glass for security and operations.

| Source | Log Type | Destination |
|---|---|---|
| Azure Firewall | Network rules, IDPS alerts, DNS queries | Log Analytics Workspace |
| NSG Flow Logs | Traffic flow records per subnet | Log Analytics + Traffic Analytics |
| Azure Bastion | Admin session logs | Log Analytics Workspace |
| Azure SQL | Query logs, login events | Log Analytics Workspace |
| Azure Entra ID | Sign-in logs, audit logs, risk events | Log Analytics + Microsoft Sentinel |
| Azure Key Vault | Secret access, policy changes | Log Analytics Workspace |
| App Service / AKS | Application logs, container stdout | Log Analytics (Container Insights) |
| Activity Logs | All ARM operations (create, delete, modify) | Log Analytics Workspace |

**Microsoft Sentinel** ingests all logs from Log Analytics and applies analytics rules to detect:
- Brute-force attempts on Bastion
- Unusual data export from Storage accounts
- Privilege escalation attempts (PIM elevation followed by unusual activity)
- Impossible travel (sign-ins from two geographically distant locations within minutes)

**Microsoft Defender for Cloud** provides:
- Secure Score tracked over time (target: ≥80%)
- Regulatory compliance dashboard mapped to HIPAA, GDPR, and CIS Azure Benchmark
- Workload protection alerts for servers, containers, SQL, and storage

---

### 5.2 Compliance Enforcement

| Mechanism | Purpose |
|---|---|
| **Azure Policy Initiatives** | Enforce HIPAA controls (encryption, private access, tagging) with Audit and Deny effects |
| **Defender for Cloud Compliance Dashboard** | Real-time view of control pass/fail status mapped to HIPAA, GDPR, PCI-DSS |
| **Azure Policy Compliance Reports** | Exported monthly to storage and shared with CloudMed's compliance team and external auditors |
| **Immutable Log Retention** | Logs retained for **90 days hot** in Log Analytics + **7 years cold** in Storage (WORM) to satisfy HIPAA audit trail requirements |
| **Access Reviews (Entra ID)** | Quarterly automated reviews of all RBAC assignments and PIM-eligible roles — inactive or unnecessary assignments are auto-revoked |

---

### 5.3 Cost Management

| Control | Detail |
|---|---|
| **Mandatory Resource Tags** | `env`, `owner`, `costcenter`, `dataclass` tags enforced by Azure Policy. Resources missing tags are flagged in compliance reports. |
| **Subscription Budgets** | Each production subscription has a monthly budget with alerts at 80% and 100% thresholds, notifying the FinOps team and subscription owner. |
| **Cost Center Tagging** | Tags drive chargeback reporting in Azure Cost Management — each business unit (Clinical Platform, AI Analytics, Infrastructure) sees its own spend. |
| **Dev Subscription SKU Limits** | Azure Policy restricts VM sizes and regions in Development subscriptions to prevent over-spending on non-production workloads. |
| **Advisor Recommendations** | Azure Advisor cost recommendations (right-sizing, reserved instances) reviewed monthly by the FinOps team. |
| **Reserved Instances** | Stable production workloads (AKS node pools, Azure SQL MI) are covered by 1-year Reserved Instances to achieve up to 40% cost savings vs. pay-as-you-go. |

---

## 6. Conceptual Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════╗
║              CLOUDMED SOLUTIONS — ZERO TRUST AZURE LANDING ZONE             ║
║                     Canada Central + West Europe Regions                     ║
╚══════════════════════════════════════════════════════════════════════════════╝

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                    TENANT ROOT / MANAGEMENT GROUPS                       │
  │  [Root] → [CloudMed MG] → [Platform] [Production] [Development]         │
  │  Azure Policy: Allowed Regions | No Public IPs | Tags | Encryption       │
  │  PIM: JIT Admin Access | Conditional Access: MFA + Compliant Device      │
  └────────────────────────────┬────────────────────────────────────────────┘
                               │ RBAC + Policy Inheritance
  ╔════════════════════════════╪════════════════════════════════════════════╗
  ║          CANADA CENTRAL REGION         ║       WEST EUROPE REGION       ║
  ║                                        ║    (Mirror Hub-Spoke Topology) ║
  ║  ┌─────────────────────────────────┐   ║   ┌──────────────────────┐    ║
  ║  │   PLATFORM SUBSCRIPTIONS        │   ║   │  Platform (EU)       │    ║
  ║  │                                 │   ║   └──────────────────────┘    ║
  ║  │  ┌──────────────────────────┐   │   ║   ┌──────────────────────┐    ║
  ║  │  │  MANAGEMENT SUBSCRIPTION │   │   ║   │  Production (EU)     │◄───╫──── GDPR Boundary
  ║  │  │  • Log Analytics         │   │   ║   └──────────────────────┘    ║
  ║  │  │  • Microsoft Sentinel    │   │   ║                                ║
  ║  │  │  • Defender for Cloud    │   │   ║   Global VNet Peering          ║
  ║  │  │  • Automation Account    │   │   ║   (Hub ↔ Hub, Firewall-chained)║
  ║  │  └──────────────────────────┘   │   ╚════════════════════════════════╝
  ║  │                                 │
  ║  │  ┌──────────────────────────┐   │
  ║  │  │ CONNECTIVITY SUBSCRIPTION│   │
  ║  │  │                          │   │
  ║  │  │   HUB VNet 10.0.0.0/16  │   │
  ║  │  │  ┌────────────────────┐  │   │
  ║  │  │  │ Azure Firewall     │  │   │
  ║  │  │  │ Premium + IDPS     │◄─┼───┼───── All spoke traffic inspected
  ║  │  │  └────────────────────┘  │   │       (east-west + north-south)
  ║  │  │  ┌────────────────────┐  │   │
  ║  │  │  │ Azure Bastion      │  │   │
  ║  │  │  │ Standard (JIT PIM) │◄─┼───┼───── Admin SSH/RDP only via Bastion
  ║  │  │  └────────────────────┘  │   │       No public IPs on any VM
  ║  │  │  ┌────────────────────┐  │   │
  ║  │  │  │ Private DNS Zones  │  │   │
  ║  │  │  │ (internal names)   │  │   │
  ║  │  │  └────────────────────┘  │   │
  ║  │  │  ┌────────────────────┐  │   │
  ║  │  │  │ VPN/ER Gateway     │◄─┼───┼───── Encrypted link to on-prem
  ║  │  │  └────────────────────┘  │   │       hospitals/clinics
  ║  │  └──────────────────────────┘   │
  ║  └──────────────┬──────────────────┘
  ║                 │ VNet Peering + UDRs (force traffic via Firewall)
  ║   ┌─────────────┼──────────────────────────────────────┐
  ║   │             │                                       │
  ║   ▼             ▼                                       ▼
  ║  ┌─────────────────────┐  ┌──────────────────────┐  ┌────────────────────┐
  ║  │ SPOKE 1 (App)        │  │ SPOKE 2 (API)         │  │ SPOKE 3 (Data)     │
  ║  │ 10.1.0.0/16          │  │ 10.2.0.0/16           │  │ 10.3.0.0/16        │
  ║  │ App-Prod Subscription│  │ API-Prod Subscription │  │ Data-Prod Sub.     │
  ║  │                      │  │                        │  │                    │
  ║  │ ┌──────────────────┐ │  │ ┌────────────────────┐│  │ ┌────────────────┐ │
  ║  │ │ App Gateway + WAF│ │  │ │ AKS (microservices)││  │ │ Azure SQL MI   │ │
  ║  │ │ (Public Ingress) │ │  │ │ + mTLS service mesh││  │ │ (Private EP)   │ │
  ║  │ └──────────────────┘ │  │ └────────────────────┘│  │ + TDE + CMK     │ │
  ║  │ ┌──────────────────┐ │  │ ┌────────────────────┐│  │ └────────────────┘ │
  ║  │ │ APIM (OAuth2/JWT)│ │  │ │ Private Endpoints  ││  │ ┌────────────────┐ │
  ║  │ │ App Service (TLS)│ │  │ │ (no public access) ││  │ │ ADLS Gen2      │ │
  ║  │ └──────────────────┘ │  │ └────────────────────┘│  │ │ (Private EP)   │ │
  ║  │                      │  │                        │  │ + Immutable WORM│ │
  ║  │ NSG: Deny all except │  │ NSG: Only from App     │  │ └────────────────┘ │
  ║  │ HTTPS 443 inbound    │  │ spoke via Firewall     │  │ ┌────────────────┐ │
  ║  └──────────────────────┘  └────────────────────────┘  │ │ Azure Key Vault│ │
  ║                                                         │ │ (CMK, Secrets) │ │
  ║                                                         │ + Private EP     │ │
  ║                                                         │ └────────────────┘ │
  ║                                                         │                    │
  ║                                                         │ NSG: Only from API │
  ║                                                         │ spoke via Firewall │
  ║                                                         └────────────────────┘
  ║
  ║  ZERO TRUST LAYER SUMMARY
  ║  ┌────────────────────────────────────────────────────────────────────────┐
  ║  │ [Verify Explicitly]  Entra ID + MFA + Conditional Access + APIM JWT    │
  ║  │ [Least Privilege]    PIM JIT + Managed Identity + NSG Deny-Default     │
  ║  │ [Assume Breach]      Firewall IDPS + Segmentation + CMK + Sentinel     │
  ║  └────────────────────────────────────────────────────────────────────────┘
  ╚══════════════════════════════════════════════════════════════════════════════╝
```

### Diagram Layer Legend

| Layer | Components | Zero Trust Mapping |
|---|---|---|
| **Governance** | Management Groups, Azure Policy, RBAC, PIM | Verify Explicitly, Least Privilege |
| **Identity** | Entra ID, Conditional Access, MFA, Managed Identities | Verify Explicitly |
| **Network — Hub** | Azure Firewall, Bastion, DNS, Gateway | Assume Breach, Least Privilege |
| **Network — Spokes** | App/API/Data VNets, NSGs, UDRs, Private Endpoints | Assume Breach, Least Privilege |
| **Workload** | App Service, AKS, Azure SQL, ADLS, Key Vault | Verify Explicitly, Assume Breach |
| **Monitoring** | Log Analytics, Sentinel, Defender for Cloud | Assume Breach (visibility) |

---

## 7. Summary and Recommendations

### Design Rationale

The CloudMed Solutions Zero Trust Landing Zone is built on three foundational commitments:

**Security by Design** — Security is not layered on top of architecture; it is embedded into every component. Private Endpoints, NSG deny-defaults, Firewall IDPS, and Conditional Access policies mean that the environment is secure even in the absence of correct human behavior. There is no configuration an administrator can "forget" that would leave PHI exposed — Azure Policy enforces it.

**Compliance as Code** — HIPAA, GDPR, and PIPEDA controls are expressed as Azure Policy definitions that are evaluated continuously and automatically. Compliance reports are not prepared manually before audits — they are generated in real time from Defender for Cloud. This reduces audit overhead and eliminates the risk of configuration drift going undetected.

**Scalability without Complexity** — The management group hierarchy, hub-and-spoke network, and subscription-per-workload-tier pattern are designed to onboard new regions, new products, or new regulatory boundaries without redesigning the architecture. A new spoke can be provisioned, peered to the hub, and automatically governed by inherited policies within hours.

This design directly addresses CloudMed's key risks: an external attacker who breaches the App tier cannot reach PHI in the Data tier without traversing an Azure Firewall with IDPS. A compromised employee account cannot escalate privileges without triggering a PIM alert and approval workflow. A misconfiguration that exposes a resource publicly is blocked by policy before it can be saved.

---

### Recommendation 1 — Infrastructure as Code and GitOps Automation

Currently, the architecture is designed to be deployed manually through the Azure Portal or CLI. The immediate next step is to codify the entire landing zone using **Azure Bicep** or **Terraform**, stored in a GitHub repository with branch protection rules.

A GitOps pipeline (GitHub Actions or Azure DevOps) would:
- Automatically deploy policy definitions and RBAC assignments when merged to main
- Run `terraform plan` / `az deployment what-if` as a pull request check — showing any proposed changes before they reach production
- Enforce that **no infrastructure change reaches production without a code review and an approved PR**

This eliminates configuration drift, creates an auditable change history for every resource, and dramatically reduces the time to recover from or re-create environments (disaster recovery, new region onboarding).

---

### Recommendation 2 — Zero Trust Network Access (ZTNA) for Remote Clinical Workforce

The current design secures the cloud infrastructure, but clinical staff access MedConnect through public internet via Application Gateway + WAF. A mature evolution of the Zero Trust posture would replace or supplement this with **Microsoft Entra Private Access** (formerly Azure AD Application Proxy / Global Secure Access).

With ZTNA in place:
- Clinical users are issued a **lightweight agent** on their managed device
- Access to MedConnect is brokered through Entra — the application is never exposed directly on the internet
- Every session carries full identity context (user, device, risk score, location), enabling **continuous access evaluation** — a session that becomes risky mid-use (device becomes non-compliant, anomalous behavior detected) is terminated automatically without requiring the user to sign in again
- This removes the Application Gateway as a public-internet-facing attack surface entirely

This is the path from "Zero Trust for infrastructure" to "Zero Trust for every workload access, everywhere."

---

*This document was prepared as part of an Azure cloud architecture lab exercise. All company names, IP ranges, and configurations are fictional and intended for educational purposes only.*
