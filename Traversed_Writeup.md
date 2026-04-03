///////










Lab Writeup: Traversed
Platform: HackerDNA


https://hackerdna.com/labs/traversed

Difficulty: Medium

Vulnerabilities: Exposed .git Repository, Credential Leakage, Python Library Hijacking

Points: 40

Executive Summary
The Traversed lab demonstrates the critical danger of exposing version control metadata in a production environment. By extracting a hidden .git directory, historical credentials were recovered from deleted files. These credentials provided SSH access, which led to a Privilege Escalation vector involving a misconfigured sudo entry and a Python library hijacking vulnerability.

1. Information Gathering
The target was a landing page on an Nginx server stating the site was "under construction." A reconnaissance probe for common hidden directories revealed a live Git repository.

Vulnerability: Information Disclosure

Proof of Concept: A request to http://[IP]/.git/HEAD returned ref: refs/heads/master, confirming the exposure.

2. Git Forensics
Using git-dumper, the entire repository was reconstructed locally. An analysis of the commit history (git log) revealed a series of commits where a developer attempted to "redact" and "remove" a credentials file.

The Mistake: Deleting a file in a new commit does not remove it from the Git history.

The Recovery: By using git show [COMMIT_HASH], the original contents of credentials.txt were recovered, revealing a cleartext username and password: hackerdna:Password@1.

3. Initial Foothold
The recovered credentials were used to gain a remote shell via SSH.

Bash

ssh hackerdna@[IP]
Upon landing, the first flag was located through system-wide enumeration:

User Flag Path: /home/flag-user.txt

4. Privilege Escalation (Python Library Hijacking)
Local enumeration of sudo privileges (sudo -l) showed that the hackerdna user could run a specific Python script as root without a password:

(root) NOPASSWD: /usr/bin/python3 /home/hackerdna/test.py

Vulnerability Analysis
The test.py script contained the following code:

Python

import webbrowser
webbrowser.open("https://google.com")
The script was owned by root and was not writable by the current user. However, because the script was executed from /home/hackerdna/, and Python's module search order prioritizes the current working directory, a Library Hijacking attack was possible.

The Attack
A malicious file named webbrowser.py was created in the home directory.

The file contained code to spawn a system shell:
import os; os.system("/bin/sh")

The root-level script was triggered:
sudo /usr/bin/python3 /home/hackerdna/test.py

When the script attempted to import webbrowser, it loaded the malicious local file instead of the system library, granting an immediate root shell.

Root Flag Path: /root/flag-root.txt

5. Remediation Recommendations
Block Metadata Access: Configure web servers (Nginx/Apache) to deny all requests to hidden directories (starting with a dot .).

Sanitize Git History: If a secret is committed, rotating the secret is the only secure fix. Tools like BFG Repo-Cleaner can be used to purge history if rotation is impossible.

Principle of Least Privilege: Avoid NOPASSWD sudo entries for scripts that reside in user-writable directories.

Secure Python Execution: Use the -I (isolated) flag for Python scripts to ignore the current directory in the module search path.
