//////









## **HackerDna ClearDesk Lab Overview**
https://hackerdna/labs/cleardesk
 
 * **Target:** ClearDesk v2.1 (Internal Ticket & Asset Management)
 * **Initial Entry:** Low-privilege guest credentials.
 * **Core Vulnerabilities:** * **IDOR** (Insecure Direct Object Reference) on the API ticket endpoint.
   * **Credential Leakage** via internal documentation.
   * **LFI** (Local File Inclusion) / Path Traversal in the Administrative Log Viewer.
## **Phase 1: Initial Enumeration**
Upon logging in with provided guest credentials (guest:guest2026), the dashboard appeared to be restricted to a few basic tickets. However, the application utilized a RESTful API at /api/ticket/[ID].
By intercepting requests and testing sequential IDs, it was discovered that the backend failed to implement **Object-Level Authorization**. This allowed the guest user to view tickets belonging to other users, including high-level executives.
## **Phase 2: Data Harvesting & Credential Theft**
Using a simple loop to enumerate ticket IDs, two critical pieces of information were exfiltrated:
 1. **Ticket #1:** Belongs to the IT Director (Sarah Chen). It confirmed the existence of "Annual Security Audit findings" and provided the first user-level flag.
 2. **Ticket #8:** A "Password Reset" ticket. This was the "Golden Ticket." It contained hardcoded credentials for the **Admin Portal** (admin:Cl3arD3sk_Adm1n_2026) and explicitly mentioned a "nightly backup job" failing to read a sensitive file located at /root/flag-root.txt.
## **Phase 3: The Administrative Pivot**
With the leaked credentials, we transitioned from the guest account to the admin account. Accessing the /dashboard as an administrator revealed a new **Admin Panel** containing a **Log Viewer** tool.
The Log Viewer functioned by requesting files through the following endpoint:
GET /admin/logs?file=[filename]
## **Phase 4: Local File Inclusion (LFI)**
The file parameter in the Log Viewer was found to be vulnerable to **Path Traversal**. Initial attempts to use raw ../ sequences were intermittently blocked or caused connection "hangs" due to Nginx normalization/filtering.
To bypass these filters, **Double URL Encoding** was employed (%252e%252e%252f), allowing the traversal to reach the root directory.
### **The Root Execution**
Leveraging the intelligence gathered from Ticket #8 (which provided the exact, non-standard filename of the root flag), the final payload was constructed:
```bash
curl -b admin_cookie.txt "http://[TARGET_IP]/admin/logs?file=../../../../../../root/flag-root.txt"

```
The server responded with a JSON object containing the content of the root flag file, completing the objective.
## **Remediation Recommendations**
| Vulnerability | Mitigation Strategy |
|---|---|
| **IDOR** | Implement **Attribute-Based Access Control (ABAC)** to ensure users can only access resources tied to their specific owner_id. |
| **Credential Leakage** | Enforcement of **Secret Management Policies**. Passwords should never be stored in plaintext within ticket descriptions or logs. |
| **LFI / Path Traversal** | Use a **Whitelist** for file access. The application should only allow specific filenames from a pre-defined directory and sanitize all input to remove traversal sequences (e.g., ../). |
| **Broken Authentication** | Force a **Password Reset** on first login for temporary administrative accounts and implement Multi-Factor Authentication (MFA). |
**Summary:** This lab highlights that security is only as strong as its weakest link. A simple IDOR exposed the credentials that bypassed the authentication layer, while a poorly sanitized log viewer provided the filesystem access necessary to claim root.
