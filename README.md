# Cloud security 
Azure security baseline built using Azure's student account. This includes a hub and spoke topology, Entra ID RBAC and Microsoft Senteniel. I've built this to be mapped to NIST 800-53 standards.

## Status: In-progress - Phases 1-3 complete. Phases 4 partial, and 5 pending
  - No P2 License, student account

## Motive
I'm a current cybersecurity professional and grad student. I begin building this lab to help get more hands on experience in the cloud and a deeper understanding of all the capabilities Azure has to offer. I want to build my governance background into cloud security, engineering to be specifically. I want to grow and maintain a hybrid approach, being able to securly build cloud environments while simultaneously governing them.

## Architecture

### Design Principles
- Deny by default across all security layers
- Least privilege applied to identity and network
- Defense in depth — multiple enforcement layers
- Eliminate attack surface rather than restrict it
- Document why before how — every control has a rationale

### Lab Constraints and How They Were Handled

| Constraint | Impact | Resolution |
|-----------|--------|-----------|
| Azure for Students — no P2 license | No Privileged Identity Management | Manual JIT process documented — same security principle, higher operational overhead |
| $100 credit limit | Cannot run expensive resources continuously | Azure Firewall deployed briefly, configured, screenshotted, deleted. UDR pre-staged. |
| Azure Policy student permissions | Deny effect uncertain | Audit effect used — production migration path documented |
| Free data connectors only | Limited Sentinel telemetry | Three zero-cost sources used, coverage gaps explicitly documented |



## What Was Built

### ✅ Phase 1 — Governance Foundation
Management group hierarchy, resource group structure, mandatory tag enforcement via Azure Policy initiative, cost anomaly alerting.

### ✅ Phase 2 — Identity and Access Management
Four test users with scope-limited RBAC, three Conditional Access policies, manual JIT for Global Admin, Security Defaults replaced with CA, external collaboration locked down.

### ✅ Phase 3 — Network Security
Hub and spoke VNet topology, deny-by-default NSGs, Azure Firewall Standard with Management NIC deployed and configured, UDR enforcing hub inspection.

### Phase 4 — Security Monitoring *(Partially Complete)*
Log Analytics workspace deployed. Sentinel deployed. Entra ID connector confirmed flowing. Azure Activity connector — policy remediation task pending verification. Custom KQL rules written and saved, not yet deployed in Sentinel. Workbook pending.

### Phase 5 — Data Protection *(Not Started)*
Key Vault, storage account, private endpoints, diagnostic logging — TBD

### Phase 6 — Documentation and GitHub *(In Progress)*
Architecture diagram current through Phase 3. DECISIONS.md maintained throughout. This README.

## Walkthrough

### Phase 1 — Governance 



Management Group Hierarchy:
Tenant Root Group
└── Production
└──      └── Student: SecLab-Production Subscription
└── Security

Four resource groups with mandatory tags:

rg-seclab-identity     — Identity and access resources
rg-seclab-networking   — All network components
rg-seclab-monitoring   — Sentinel, Defender, Log Analytics
rg-seclab-storage      — Key Vault, Storage Account

Tags applied to all:
  Environment:        Lab
  Owner:              Trivaun George
  DataClassification: Internal
  Project:            AzureSecBaseline

Azure Policy initiative — four tag requirements consolidated under single assignment. Audit effect used rather than Deny.

Budget alert at $50 (80% threshold) configured as cost anomaly detection control.

**Key decisions:**

*Management groups over flat subscription structure:* Policy and RBAC inheritance flows from the top automatically. Security baselines enforced once at the management group level apply to everything underneath without touching individual resources. Similar to inheriting controls from a parent package.

*Audit effect:* Production practice. Enable Audit first to establish compliance baseline visibility, remediate existing non-compliant resources, then switch to Deny. Enabling Deny on an environment with existing untagged resources breaks deployments immediately.

*Cost alerting is a security control:*Unauthorized VM deployment, and data exfiltration all generate cost anomalies before they generate security alerts in many configurations. A $50 alert costs nothing and catches an activity category dedicated tooling frequently misses early.

**NIST coverage:** CM-2, CM-8, AC-6, CA-7, SI-4, IR-5


### Phase 2 — Identity and Access Management

**What was configured:**

Four test users representing distinct job functions with differentiated RBAC:

| Identity | Role | Scope |
|----------|------|-------|
| seclab-globaladmin | Owner | Subscription (temporary — documented removal) |
| seclab-developer | Contributor | rg-seclab-networking ONLY |
| seclab-readonly | Reader | Subscription |
| seclab-security | Security Reader | Subscription |

Security Defaults enabled
MFA is required. Conditional access policies requires Entra ID P1 or P2 for full functionality. Security Defaults provide solid baseline protection appropriate for this lab environment without licensing requirements. In a production environment with P1/P2 available, custom CA policies would replace Defaults to enable differentiated requirements. 


**Key decisions:**

*Developer scoped to resource group, not subscription:* If the developer account is compromised the attacker has Contributor on the networking resource group only — not Key Vault, monitoring resources, or identity configuration. Blast radius is contained before the incident happens.

*Manual JIT (just in time) for Global Admin (no PIM):* Without Entra ID P2, PIM is unavailable. Manual equivalent: assign the role for the specific task, complete the task, remove the role, screenshot the audit log. Operationally heavier than PIM but the security principle is identical — no standing privilege.

**NIST coverage:** AC-2, AC-3, AC-6, AC-17, AC-20, IA-2, IA-5


### Phase 3 — Network Security

**Configured:**

Hub VNet (10.0.0.0/16)
├── AzureFirewallSubnet          (10.0.1.0/26)
├── AzureFirewallManagementSubnet (10.0.4.0/26)
├── GatewaySubnet                (10.0.2.0/27)
└── ManagementSubnet             (10.0.3.0/24)

Spoke 1 VNet (10.1.0.0/16) — Workloads
├── WebTierSubnet  (10.1.1.0/24)
└── AppTierSubnet  (10.1.2.0/24)

Spoke 2 VNet (10.2.0.0/16) — Security Tooling
└── SecurityToolsSubnet (10.2.1.0/24)


Four peering connections all showing Connected. Four NSGs with deny-by-default rules associated to all workload subnets. Azure Firewall Standard with Management NIC — deployed, configured, screenshotted, deleted. Two route tables pre-staged with 10.0.1.4 next hop.

**Key decisions:**

*Hub and spoke over flat network:* Centralizes security inspection at the firewall. All inter-spoke traffic transits hub — single enforcement point. Spoke isolation means a compromised workload in Spoke 1 cannot reach security tooling in Spoke 2 without transiting hub controls. This is authorization boundary architecture — each spoke is an isolated workload boundary with cross-boundary traffic inspected at a defined enforcement point.

*Dedicated Spoke 2 for security tooling:* Monitoring resources should be isolated from the workloads they monitor. A compromised workload cannot tamper with the logs recording its compromise if those logs live in a separate isolated spoke.

*NSG deny-by-default:* Allowlist approach — any traffic not explicitly permitted is blocked. Every allow rule has a documented justification. This is the same logical structure as STIG findings for network devices — deny all, permit only what is required, document the rationale for every exception.

*Management NIC enabled on firewall:* Separates Azure platform management traffic from data plane traffic. Two benefits: management traffic does not pollute data plane logs, and the firewall remains manageable even if a data plane rule misconfiguration blocks everything. Cloud equivalent of out-of-band management on a physical firewall.

*Firewall deployed then deleted:* Azure Firewall Standard costs ~$1.25/hour. Deployed for 2-3 hours to configure and screenshot everything, then deleted. The screenshots and documentation are the portfolio artifact — not the running resource. UDR routes pre-staged with 10.0.1.4 so traffic routes correctly the moment a firewall is redeployed.

*UDR : Without User Defined Routes, Azure peering system routes allow direct spoke-to-spoke traffic bypassing the hub firewall entirely. The diagram looks correct. The security is not. UDR overrides system routes and forces all traffic through the firewall. This is the most common Azure network misconfiguration in security assessments — hub and spoke deployed without UDR, firewall in the diagram, no traffic ever inspected.

**NIST coverage:** AC-4, CM-6, CM-7, SC-3, SC-7, SC-39, SI-3, AU-2

### Phase 4 — Security Monitoring *(Partial)*

**Completed:**

Log Analytics workspace deployed in rg-seclab-monitoring. Microsoft Defender for Cloud foundational CSPM active. NIST SP 800-53 Rev 5 compliance dashboard enabled. Microsoft Sentinel deployed and connected to workspace. Microsoft Entra ID data connector connected — SigninLogs and AuditLogs confirmed flowing via test query.

<details>
<summary>Phase 4 screenshots</summary>

\Userstriva\OneDrive\Pictures\Screenshots 1\log workspace.png)
![Defender secure score](screenshots/phase4/defender-secure-score.png)
![NIST 800-53 compliance dashboard](screenshots/phase4/nist-80053-compliance-dashboard.png)
![Sentinel data connectors](screenshots/phase4/sentinel-data-connectors.png)

</details>

**Pending:**

Azure Activity connector — Policy assignment wizard completed but remediation task status unverified at stopping point. Data flow unconfirmed. Microsoft Defender for Cloud connector — not verified. Built-in analytics rules — not enabled. Workbook — not built.

**Where to resume:**

1. Verify Azure Activity connector — run `AzureActivity | take 10` in Sentinel Logs
2. If empty — rerun Policy Assignment wizard with Remediation task checkbox checked
3. Connect Defender for Cloud connector
4. Enable 2 built-in analytics rules from Rule Templates
5. Deploy 4 custom KQL rules from queries/ folder
6. Build workbook with 4 tiles

**Key decisions:**

*Log Analytics before everything else:* Every Defender and Sentinel event should be captured from the earliest point. Lesson learned in this lab — workspace deployed in Phase 4 after Phase 1-3 work was complete. Deployment activity and configuration events from earlier phases are not in the logs. In a repeat of this lab the workspace gets deployed first, before resource groups, VNets, or anything else.

*NIST 800-53 compliance dashboard specifically:* This is the framework governing DoD cloud authorization. Connecting it to the environment I am building makes Defender for Cloud recommendations directly relevant to my day job — automated cloud STIGs against the same control baseline. Important distinction: a high compliance score is not an authorization. A completed STIG checklist is not an ATO. Technical controls and authorization decisions are different things.

*Three free connectors only:* Azure Activity, Entra ID, and Defender for Cloud cover identity, platform, and security alert data — the highest-value sources for detecting cloud-specific attack patterns. Coverage gaps are documented rather than obscured.

**Custom KQL rules — threat model first:**

Every rule was designed by identifying the MITRE ATT&CK technique before writing the query. Starting with available data produces detection volume. Starting with the threat model produces detection coverage.

| Rule | MITRE Technique | Data Source |
|------|----------------|------------|
| Admin role assignment after hours | T1078 — Valid Accounts | AuditLogs |
| Admin account brute force | T1110.001 — Password Guessing | SigninLogs |
| Bulk resource deletion | T1485 — Data Destruction | AzureActivity |
| Key Vault unexpected secret access | T1555 — Password Stores | AzureDiagnostics |

**NIST coverage (when complete):** AU-2, AU-6, AU-9, CA-2, CA-7, IR-4, SI-4


### Phase 5 — Data Protection *(Not Started)*

Planned configuration:
- Key Vault with RBAC permission model and purge protection
- Four differentiated access levels — Administrator, Secrets User, Reader, no access
- Secrets and keys with explicit expiration dates
- Diagnostic logging to law-seclab-monitoring (activates Rule 4)
- Private endpoint eliminating public internet exposure
- Storage account with disabled account keys, TLS 1.2 minimum
- Blob soft delete and versioning as ransomware recovery controls
- Private endpoint on storage

Resuming after academic break 

---

## Issues and Troubleshooting 

Problems I encountered

### Issue 1 — Azure Activity Connector Not Flowing

**What happened:** Launched the Azure Policy Assignment wizard for the Azure Activity connector. Connector showed as connected but `AzureActivity | take 10` returned no results.

**Root cause identified:** The Policy Assignment wizard completes but does not automatically remediate existing resources. The **Create a remediation task** checkbox on the Remediation tab must be checked. Without it the policy exists but does not apply to current subscription resources.

**Status at stopping point:** Wizard rerun with remediation task. Data flow unverified — lab paused before confirmation.

**Resolution when resuming:** Run `AzureActivity | take 10`. If still empty go to Azure Policy → Remediation → verify a remediation task exists and completed successfully for the activity log policy.

---

### Issue 2 — Entra ID Connector Showing Disconnected Despite Checked Boxes

**What happened:** Checked all three log type boxes in the Entra ID connector configuration. Connector still showed disconnected status.

**Root cause:** Checking the boxes without clicking Apply Changes does nothing. Additionally the connector method has permission limitations on student accounts.

**Resolution:** Bypassed the connector method entirely. Created a direct Diagnostic Setting on Entra ID routing SigninLogs and AuditLogs directly to the Log Analytics workspace. Data confirmed flowing via `SigninLogs | take 10`.

**Lesson learned:** When a Sentinel connector method fails on a student account, direct Diagnostic Settings on the source resource achieve the same result. The data ends up in identical tables — the path is just different.

### Issue 3 — Microsoft 365 Insider Risk Connector Connected Unexpectedly

**What happened:** Microsoft 365 Insider Risk Management connector appeared as connected without intentional configuration.

**Impact:** None to the lab objectives. This connector routes Microsoft 365 compliance signals — different data source from what the KQL rules target.

**Resolution:** Left connected, noted as outside lab scope. Does not interfere with Azure Activity, Entra ID, or Defender for Cloud data.

---

### Issue 4 — Azure Policy Permissions on Student Account

**What happened:** Attempted to assign tag enforcement policies with Deny effect. Student subscription permissions were uncertain for custom Deny policies.

**Resolution:** Used Audit effect instead. Documents compliance visibility without blocking. Production migration path — Audit first, remediate, then switch to Deny — is documented in DECISIONS.md.

**Lesson learned:** Student accounts have subscription-level permission limitations that production accounts do not. Audit effect works for initial rollout regardless of account type. Not a workaround — a correct practice.

### Issue 5 — Firewall Cost Management

**What happened:** Azure Firewall Standard at ~$1.25/hour would consume a significant portion of the $100 student credit if left running.

**Resolution:** Deployed for 2-3 hours. Configured all rules. Took screenshots of every configuration screen. Deleted the firewall to stop billing. Retained the public IP resources (negligible cost). Pre-staged UDR routes with 10.0.1.4 as the expected firewall private IP based on subnet allocation.

**Why this is legitimate:** Route tables configurations are independent of the appliance they route traffic to. When the firewall is redeployed traffic routes correctly immediately. The documentation and screenshots are the portfolio artifact — not the running resource.


## NIST 800-53 Rev 5 Coverage

### Completed Controls

| Control | Family | Phase | Implementation |
|---------|--------|-------|---------------|
| AC-2 | Account Management | 2 | Entra ID lifecycle, RBAC, JIT documentation |
| AC-3 | Access Enforcement | 2 | RBAC, CA grant controls |
| AC-4 | Info Flow Enforcement | 3 | NSG rules, UDR, firewall rules |
| AC-6 | Least Privilege | 1,2 | Scope-limited RBAC, subnet separation |
| AC-17 | Remote Access | 2 | CA MFA, platform restriction |
| AC-20 | External Systems | 2 | Guest access restricted |
| CM-2 | Baseline Configuration | 1 | Management groups, policy inheritance |
| CM-6 | Configuration Settings | 3 | NSG rules, firewall policy, UDR |
| CM-7 | Least Functionality | 3 | Deny-by-default, FQDN allowlists |
| CM-8 | Component Inventory | 1 | Mandatory tagging |
| IA-2 | Authentication | 2 | MFA via Conditional Access |
| IA-5 | Authenticator Management | 2 | MFA registration, break-glass |
| SC-3 | Security Isolation | 3 | Dedicated security spoke, Management NIC |
| SC-7 | Boundary Protection | 3 | NSGs, Firewall, UDR |
| SC-39 | Process Isolation | 3 | Firewall Management NIC |
| SI-3 | Malicious Code Protection | 3 | Threat intel Alert and Deny |

### Pending Controls (Phases 4-5)

| Control | Phase | Planned Implementation |
|---------|-------|----------------------|
| AU-2 | 4 | Diagnostic settings, Sentinel connectors |
| AU-6 | 4 | Sentinel analytics rules, workbook |
| AU-9 | 4 | Dedicated monitoring group, workspace isolation |
| CA-2 | 4 | Defender for Cloud continuous assessment |
| CA-7 | 4 | NIST 800-53 compliance dashboard |
| CP-9 | 5 | Blob soft delete, versioning |
| IR-4 | 4 | Sentinel incident creation |
| SC-28 | 5 | Storage encryption, Key Vault |
| SI-4 | 4 | Sentinel, Defender, budget alerts |
| SI-12 | 5 | Purge protection, soft delete |

---

## MITRE ATT&CK Detection Coverage

*Rules written — deployment pending Phase 4 completion*

| Technique | ID | Rule | Status |
|-----------|-----|------|--------|
| Valid Accounts — Privilege Escalation | T1078 | Detect-AdminRoleAssignment-AfterHours | Written, not deployed |
| Brute Force — Password Guessing | T1110.001 | Detect-AdminAccount-BruteForce | Written, not deployed |
| Data Destruction | T1485 | Detect-BulkResourceDeletion | Written, not deployed |
| Service Stop | T1489 | Detect-BulkResourceDeletion | Written, not deployed |
| Credentials from Password Stores | T1555 | Detect-KeyVault-UnexpectedSecretAccess | Written, not deployed |

## Repository Contents

```
azure-security-baseline/
├── README.md                          ← This file
├── DECISIONS.md                       ← Full architectural decision log
├── LAB_LOG.md                         ← Step-by-step build documentation
├── architecture/
│   └── seclab-architecture-v1.png    ← Phase 3 complete diagram
├── screenshots/
│   ├── phase1/                        ← Governance screenshots
│   ├── phase2/                        ← IAM screenshots
│   ├── phase3/                        ← Network screenshots
│   └── phase4/                        ← Partial monitoring screenshots
└── queries/
    ├── admin-role-assignment-afterhours.kql
    ├── admin-bruteforce-detection.kql
    ├── bulk-resource-deletion.kql
    └── keyvault-unexpected-access.kql
```

*Active project — Phases 4 and 5 resuming after academic break. Last updated May 2026.*
