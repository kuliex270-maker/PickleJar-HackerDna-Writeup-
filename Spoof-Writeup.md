This writeup covers the Spoof! lab on HackerDNA. This challenge is a classic study in Host-Based Authentication vulnerabilities, specifically how older services like rlogin and rsh can be tricked if they trust an IP address without secondary verification.

HackerDNA Lab Writeup: Spoof!
Difficulty: Easy
[
](https://hackerdna.com/labs/spoof)
Core Concepts: Service Enumeration, .rhosts Misconfiguration, IP Spoofing/Identity Theft, R-services Exploitation.

Phase 1: Reconnaissance & Enumeration
As always, we start by scanning the target to see what services are exposed to the network.

1. Port Scanning
Run a standard version detection scan:

Bash

nmap -sV <TARGET_IP>
Expected Output:
The scan will likely reveal ports 512 (exec), 513 (login), or 514 (shell). These are the "R-services" (Remote Services).

2. Understanding the Vulnerability
The R-services (like rlogin) were designed for convenience in trusted environments. They often rely on a file called .rhosts located in a user's home directory. If this file contains an IP address or a hostname, the server will allow that specific source to log in without a password.

Phase 3: Identifying the Trusted Host
To "spoof" the identity, we first need to know who the server actually trusts.

1. Finding Clues
Check the web server (if active on Port 80) or other public services for a developer note, a /backup directory, or a README file. In this lab, you might find a note stating:

"Internal migrations complete. Maintenance only allowed from the admin workstation at 10.10.10.50."

This tells us that the .rhosts file likely contains the entry 10.10.10.50.

Phase 4: Exploitation (The "Spoof")
Since the server trusts 10.10.10.50, we need to make the server think our requests are coming from that IP. In a local lab environment, we can do this by adding a secondary IP to our own network interface.

1. Assigning the Trusted IP
On your attacker machine (Kali/Parrot), assign the trusted IP to your interface (e.g., eth0 or tun0):

Bash

sudo ip addr add 10.10.10.50/24 dev eth0
2. Attempting the Passwordless Login
Now, use the rlogin or rsh command to connect as a specific user (often developer, admin, or user) discovered during enumeration:

Bash

rlogin -l user -bind 10.10.10.50 <TARGET_IP>
If the .rhosts file is configured correctly on the server, you will be dropped directly into a shell without being prompted for a password.

Phase 5: Capturing the Flags
1. Capture the User Flag
Locate the user flag in the current directory or the home folder:

Bash

cat /home/user/flag-user.txt
Result: HDNA{REDACTED_USER_FLAG}

2. Privilege Escalation to Root
Check for common misconfigurations like SUID binaries or sudo permissions:

Bash

sudo -l
If the user has (ALL) NOPASSWD: ALL permissions, simply run:

Bash

sudo su
cat /root/flag-root.txt
Result: HDNA{REDACTED_ROOT_FLAG}

Defensive Remediation
The exploitation of R-services is a "relic" of older networking, but the lessons remain relevant:

Decommission Legacy Services: Services like rlogin, rsh, and rexec are inherently insecure because they transmit data in cleartext and rely on weak authentication. They should be replaced with SSH.

Avoid IP-Based Trust: Never trust a user based solely on their source IP address. IP addresses can be spoofed or hijacked (as seen in this lab).

Use Multi-Factor Authentication (MFA): Even if an IP is trusted, a second factor (like a physical token or TOTP) prevents unauthorized access.

Firewalling: Ensure that administrative services are only reachable via a VPN or a tightly controlled jump box, rather than being exposed to the wider network.
