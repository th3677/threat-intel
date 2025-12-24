# 1. Case Study Title
UNC3944 — Identity-Based Cloud Intrusion (Credential Abuse + MFA Fatigue + SaaS Data Access)

---

# 2. Purpose
## 2.1 Objective
Produce a discrete, evidence-driven analysis of an identity-based cloud intrusion pattern attributed to **UNC3944** (Mandiant tracking). This case study focuses on what defenders can concretely observe in logs and how to translate those observations into:
1) a threat model,  
2) a risk assessment,  
3) detection requirements, and  
4) mapped controls.

## 2.2 Scope
- Identity Provider authentication and MFA events  
- SaaS audit logs (email + file storage access)  
- Cloud control-plane access (IAM actions only if observed)  
- Response decisions and investigation questions

---

# 3. Actor Profile (Who)
## 3.1 Threat Actor
**UNC3944 (Scattered Spider)** — financially motivated actor known for:
1) social engineering to obtain credentials,  
2) MFA fatigue/push abuse, and  
3) operating through legitimate identity paths (low malware reliance early).

## 3.2 Typical Goal
1) Acquire access quickly with valid credentials  
2) Move to higher-value services (email, storage, admin portals)  
3) Monetize: data theft, extortion, fraud, or resale

---

# 4. Incident Narrative (Discrete Sequence)
## 4.1 Short Story Version (What happened)
A single employee account is compromised. The attacker generates repeated MFA prompts until the user approves one. The attacker logs in successfully from an unfamiliar device and location. Within minutes, the attacker accesses SaaS resources (mailbox + files) and performs abnormal high-volume reads/downloads. The activity is detected via identity and SaaS audit logs.

## 4.2 Timeline (T0–T8)
**T0 (08:58)** — Repeated MFA challenges begin for user `j.smith@company.com`  
**T1 (09:01)** — MFA challenge is approved once after multiple prompts  
**T2 (09:02)** — Successful IdP login from a *new device* and *new IP*  
**T3 (09:04)** — First access to email service from the same new session  
**T4 (09:06)** — First access to file storage service; new “bulk download” pattern begins  
**T5 (09:10)** — Access to sensitive folder path `/Finance/Payroll/` observed  
**T6 (09:12)** — New OAuth token/session created for the account (persistence signal)  
**T7 (09:15)** — Attempted access to admin console or privileged actions (only if observed)  
**T8 (09:18)** — SOC containment: session revoked, password reset, user contacted

> NOTE: Use this timeline even if you don’t have exact timestamps yet. Later you can replace with real times from your lab logs.

---

# 5. Assets at Risk (What is being attacked)
## 5.1 Primary Assets
1) **Identity:** user credentials, MFA trust, session tokens  
2) **Data:** email contents, file storage contents (documents, spreadsheets)  
3) **Access Tokens:** OAuth/session tokens enabling long-lived access

## 5.2 High-Value Targets in This Case
1) Mailbox (sensitive conversations, password reset links, invoices)  
2) File storage (finance docs, HR records, customer data)  
3) Admin portals (if the user has elevated access)

---

# 6. Evidence / Observables (What we can prove)
## 6.1 Identity Provider Observables
### 6.1.1 MFA Fatigue Pattern (Concrete)
- **15 MFA push challenges** for `j.smith@company.com` within 3 minutes  
- **14 denied/ignored**, **1 approved**  
- Approval occurs at **09:01** (T1)

**Why it matters:** This pattern is not normal user behavior and strongly indicates MFA fatigue abuse.

### 6.1.2 New Context Login (Concrete)
- Successful login at **09:02**  
- Device: `Chrome on Windows` not previously seen for user  
- IP / Geo: new city/region compared to historical baseline  
- Session created immediately after MFA approval

**Why it matters:** New device + new geo immediately following MFA fatigue suggests compromise.

## 6.2 SaaS Audit Observables (Email + Files)
### 6.2.1 Mailbox Access (Concrete)
- Mailbox opened at **09:04** from the new session  
- Search activity increases (e.g., “invoice”, “password reset”, “wire”, “payroll”)

**Why it matters:** Attackers often search email for financial terms, reset links, or internal workflows.

### 6.2.2 File Storage Access (Concrete)
- First file storage access at **09:06**  
- Unusual download pattern: dozens/hundreds of reads/downloads in minutes  
- Access to sensitive folder `/Finance/Payroll/` at **09:10**

**Why it matters:** High-volume access and sensitive path targeting indicates data theft.

## 6.3 What We Do NOT Claim Without Evidence
- We do not claim malware execution  
- We do not claim privilege escalation unless logs show role changes/admin actions  
- We do not claim data exfiltration unless downloads/API export events confirm it

---

# 7. Assumptions (Explicit)
1) Credential acquisition occurred via social engineering/phishing (common actor behavior)  
2) MFA fatigue was the bypass method (supported by MFA burst pattern)  
3) The attacker’s objective is data theft/extortion (supported by sensitive data access patterns)

---

# 8. Threat Model (How the attack works)
## 8.1 Entry Point
1) Compromised credentials  
2) MFA fatigue leading to approval  

## 8.2 Trust Boundaries
1) User device → Identity Provider  
2) Identity Provider token → SaaS services  
3) User permissions → sensitive data access

## 8.3 Abuse Path (Step-by-step)
1) Attacker initiates MFA spam to trigger approval  
2) User approves one prompt  
3) Attacker authenticates successfully and receives session token  
4) Attacker pivots to email to gather internal context  
5) Attacker pivots to file storage for sensitive documents  
6) Attacker attempts to create persistence via additional sessions/tokens  
7) Optional: attempt privilege actions (if permissions allow)

---

# 9. MITRE ATT&CK Mapping (IDs + why)
## 9.1 Techniques Observed/Modeled
| Tactic | Technique | ID | Evidence in this case |
|---|---|---|---|
| Initial Access | Valid Accounts | T1078 | Successful login using user account |
| Credential Access | MFA Request Generation | T1621 | Burst of MFA prompts then approval |
| Discovery | Account Discovery | T1087 | (If logs show enumeration) |
| Collection | Email Collection | T1114 | Mailbox access + searches |
| Collection | Data from Cloud Storage | T1530 | Abnormal file access/download |

> Only mark a technique “observed” if telemetry supports it.

---

# 10. Risk Assessment (Specific)
## 10.1 CIA Impact
1) **Confidentiality: High**  
   - Email and file storage access → sensitive data exposure likely  
2) **Integrity: Medium**  
   - If attacker modifies docs or settings, integrity risk rises  
3) **Availability: Medium**  
   - Account lockouts, service disruption, and incident response downtime

## 10.2 Risk Score
**8/10 (High)**  
Justification: low effort entry (credentials), stealthy access, high-value data targeted quickly.

---

# 11. Detection Requirements (What to build)
## 11.1 Required Log Sources
1) IdP authentication logs (success/failure, device, location)  
2) MFA logs (challenge frequency + approvals)  
3) SaaS audit logs (email + file storage reads/downloads)  
4) Token/session lifecycle logs (new sessions, token grants)

## 11.2 Detection Use Cases (Discrete)
1) **MFA Fatigue Detector**
   - Trigger when MFA challenges > N in M minutes AND an approval occurs
2) **New Device + New Geo After MFA Burst**
   - Success login from unseen device/geo within X minutes of MFA burst
3) **Post-Login Data Access Spike**
   - High-volume file reads/downloads within Y minutes of new session
4) **Sensitive Folder/Label Access**
   - Access to finance/HR folders shortly after abnormal login context

## 11.3 False Positive Handling
- Travel: allow known travel windows or VPN tags  
- New device onboarding: allow managed device enrollment events  
- Bulk downloads: allow known backup/export service accounts

---

# 12. Response Plan (What to do)
## 12.1 Immediate Containment
1) Revoke active sessions/tokens for the user  
2) Reset password and force MFA re-enrollment  
3) Temporarily disable the account if activity continues  
4) Confirm with user whether MFA prompts were unexpected

## 12.2 Investigation Questions (Answer with logs)
1) What exact resources were accessed after login?  
2) Were files downloaded/exported or only viewed?  
3) Were forwarding rules or mailbox settings modified?  
4) Were new OAuth grants or app authorizations added?  
5) Did any privileged actions occur?

## 12.3 Recovery
1) Review account permissions; reduce if excessive  
2) Apply conditional access policies (device compliance, geo/risk-based)  
3) Add stronger MFA methods if available  
4) Implement user education + helpdesk verification improvements

---

# 13. Framework Mapping (Numbered and Practical)
## 13.1 NIST CSF 2.0 Functions
1) Identify — inventory identities, critical SaaS apps, sensitive data locations  
2) Protect — strengthen MFA, conditional access, least privilege  
3) Detect — build identity anomaly + SaaS data access detections  
4) Respond — session revocation playbooks and triage workflow  
5) Recover — restore accounts, validate integrity, improve controls

## 13.2 NIST 800-53 Families (High-Level)
1) AC — access control and privileged access management  
2) IA — identification, authentication, MFA policy  
3) AU — logging, audit trails, monitoring coverage  
4) IR — incident response procedures  
5) CA — continuous assessment and control validation

## 13.3 CIS Controls v8 (Numbers)
1) CIS 5 — Account Management  
2) CIS 6 — Access Control Management  
3) CIS 8 — Audit Log Management  
4) CIS 12 — Network Infrastructure Management (supporting telemetry)

## 13.4 ISO/IEC 27001 (Control Themes)
1) Identity & access governance  
2) Security monitoring and logging  
3) Incident management process  
4) Access to information and data classification

---

# 14. Key Takeaways (Clear)
1) MFA fatigue is visible as a burst pattern — defenders must alert on it.  
2) A successful login becomes suspicious when paired with new device/geo and immediate sensitive data access.  
3) Identity and SaaS audit logs are the highest-value telemetry for this intrusion type.  
4) Response must focus on sessions/tokens, not just password resets.

---

# 15. What I Would Build Next (Practical Next Step)
1) Implement the MFA fatigue detector first (highest signal)  
2) Add the “new device + new geo after MFA burst” correlation  
3) Add post-login data access spike detection for sensitive folders  
4) Document a response playbook with step-by-step actions and investigation questions
