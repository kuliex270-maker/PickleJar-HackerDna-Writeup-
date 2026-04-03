///////









This writeup covers the Include me lab on HackerDNA. This challenge is a quintessential introduction to Local File Inclusion (LFI), a vulnerability that occurs when an application uses untrusted input to build a path to a file that is then "included" or read by the server.

https://hackerdna.com/labs/include-me

HackerDNA Lab Writeup: Include me

Difficulty: Easy

Core Concepts: URL Parameter Manipulation, Directory Traversal, Local File Inclusion (LFI), Sensitive File Disclosure.

Phase 1: Reconnaissance
We start by navigating to the provided target IP in a web browser.

1. Initial Observation
The homepage appears to be a simple web application. Upon clicking through the navigation menu (e.g., "Home", "About", "Contact"), notice how the URL changes:

http://<TARGET_IP>/index.php?page=home.php

http://<TARGET_IP>/index.php?page=about.php

2. Identifying the Vulnerability
The ?page= parameter is a classic indicator of a potential LFI vulnerability. It suggests the PHP code is likely using a function like include(), require(), or file_get_contents() to load the content of the file specified in the URL.

Phase 2: Exploitation (LFI)
Our goal is to break out of the intended directory (likely a folder containing website pages) and read sensitive system files.

1. Testing for Directory Traversal
We use the ../ (dot-dot-slash) sequence to move up the directory tree. Since we don't know exactly how deep the webroot is, we use several sequences to ensure we reach the root (/) directory.

Target File: /etc/passwd (This file contains a list of system users and is readable by all users, making it the perfect "proof of concept" for LFI).

The Payload:
Modify the URL in your browser to:

Plaintext

http://<TARGET_IP>/index.php?page=../../../../../../etc/passwd
Result: If the page displays a list of users (e.g., root:x:0:0:root:/root:/bin/bash), the LFI is confirmed.

Phase 3: Finding the Flags
In HackerDNA labs, flags are typically placed in standard locations or user home directories.

1. Capture the User Flag
Based on the /etc/passwd file, identify the non-root username (e.g., ctf-user or hacker). We can now use the LFI to read the flag from that user's home directory.

URL Payload:

Plaintext

http://<TARGET_IP>/index.php?page=../../../../../../home/user/flag-user.txt
(Note: Replace user with the actual username found in Phase 2).

Result: HDNA{REDACTED_USER_FLAG}

2. Capture the Root Flag
The final objective is usually in the /root directory. Because the web server process (like www-data) often doesn't have permissions to read the /root folder, a basic LFI might fail here unless the server is misconfigured or running as root.

URL Payload:

Plaintext

http://<TARGET_IP>/index.php?page=../../../../../../root/flag-root.txt
Note: If direct access to /root is denied, look for other interesting files via LFI, such as log files or configuration files (config.php), which might contain passwords for SSH access.

Alternative Method: PHP Wrappers
If the server has filters in place (e.g., it appends .php to your input), you can use PHP wrappers to bypass them or read the source code of the PHP files themselves.

Reading Source Code (Base64 Encode):

Plaintext

index.php?page=php://filter/convert.base64-encode/resource=index.php
You can then decode the resulting string to see the raw PHP logic.

Defensive Remediation
To prevent LFI vulnerabilities, developers should:

Use an Allowlist: Only allow specific, pre-defined filenames to be passed to the include function.

Sanitize Input: Strip out ../ and null bytes from user-supplied parameters.

Filesystem Permissions: Ensure the web server user has the minimum permissions necessary and cannot access sensitive directories like /etc or /root.

Use Database IDs: Instead of passing filenames (e.g., page=contact.php), use database IDs (e.g., page=2) to map to the desired content.
